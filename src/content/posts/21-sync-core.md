---
title: "SyncCore: The State Synchronization Engine"
description: "How Telegram iOS keeps its local database perfectly synchronized with the server. Covers AccountStateManager's operation queue, the pts/qts/seq counter system for gap-free update tracking, UpdateGroup classification, the difference-fetching protocol, AccountMutableState with 50+ mutation operations, channel-specific pts tracking, hole detection and asynchronous filling, the replayFinalState atomic database writer, and the complete update pipeline from WebSocket to Postbox."
published: 2025-07-30
tags:
  - Feature Deep Dives
toc: true
lang: en
abbrlink: 21-sync-core
pin: 5
---

Telegram's messaging protocol faces a fundamental challenge: the server is the source of truth, but the client must work offline, handle network interruptions gracefully, and never lose or duplicate a message. The solution is a sophisticated state synchronization engine built around three counters — `pts`, `qts`, and `seq` — that together ensure every update from the server is applied exactly once, in order, with no gaps.

This post dissects the sync engine that lives in `TelegramCore/Sources/State/`.

## The Three Counters

The client maintains three counters that mirror server-side state:

```swift
public class AuthorizedAccountState: AccountState {
    public final class State: PostboxCoding {
        public let pts: Int32   // "Points" — messages in private chats and groups
        public let qts: Int32   // "Quoted timestamp" — secret chat messages
        public let date: Int32  // Last sync timestamp
        public let seq: Int32   // "Sequence" — metadata and settings updates
    }
}
```

| Counter | Scope | Incremented By |
|---------|-------|----------------|
| **pts** | Personal messages | New message, edit, delete, read, pin, etc. |
| **qts** | Secret chats | New encrypted message, encrypted file |
| **seq** | Global metadata | Profile changes, contact updates, settings |

Each update from the server carries its counter's new value and a count:

```
Update: { type: newMessage, pts: 5001, pts_count: 1 }
```

The client checks: `current_pts + pts_count == new_pts`. If yes, the update is contiguous and can be applied. If not, there's a **gap** — some updates were missed, and the client must fetch them.

## AccountStateManager: The Orchestrator

`AccountStateManager` (858+ lines) is the central coordinator. It runs on a dedicated queue and processes operations sequentially:

```swift
class AccountStateManager {
    private var operations: [AccountStateManagerOperation] = []

    private func startFirstOperation() {
        guard let operation = self.operations.first else { return }
        guard !operation.isRunning else { return }
        operation.isRunning = true

        switch operation.content {
        case let .pollDifference(_, currentEvents):
            // Fetch updates from server via getDifference
        case let .collectUpdateGroups(groups, timeout):
            // Batch WebSocket updates with timeout
        case let .processUpdateGroups(groups):
            // Apply batched WebSocket updates
        case let .processEvents(_, events):
            // Emit events to UI subscribers
        }
    }
}
```

Operations run one at a time. New WebSocket updates are batched in a `collectUpdateGroups` operation with a short timeout, allowing multiple rapid updates to be processed together. This batching is critical for performance — when you open a busy group chat, you might receive dozens of updates per second.

## The Update Pipeline

### Step 1: WebSocket → UpdateMessageService

Raw MTProto messages arrive via the WebSocket connection and are caught by `UpdateMessageService`:

```swift
class UpdateMessageService: NSObject, MTMessageService {
    let pipe: ValuePipe<[UpdateGroup]> = ValuePipe()

    func mtProto(_ mtProto: MTProto!, receivedMessage message: MTIncomingMessage!,
                 authInfoSelector: MTDatacenterAuthInfoSelector, networkType: Int32) {
        if let updates = (message.body as? BoxedMessage)?.body as? Api.Updates {
            self.addUpdates(updates)
        }
    }
}
```

The service converts `Api.Updates` (the raw Telegram API type) into `UpdateGroup` values. There are several `Api.Updates` variants for efficiency:

- **`updateShortMessage`** — a compact format for simple text messages (no media, no entities), containing just the user ID, text, and counters. The service reconstructs a full `Api.Message` from it.
- **`updates`** — the standard format with a list of `Api.Update` objects, users, chats, and counters.
- **`updatesTooLong`** — the server is telling the client "you've been offline too long, do a full getDifference."

### Step 2: Update Grouping

Updates are classified by their counter type:

```swift
enum UpdateGroup {
    case withPts(updates: [Api.Update], users: [Api.User], chats: [Api.Chat])
    case withQts(updates: [Api.Update], users: [Api.User], chats: [Api.Chat])
    case withSeq(updates: [Api.Update], seqRange: (Int32, Int32), date: Int32,
                 users: [Api.User], chats: [Api.Chat])
    case withDate(updates: [Api.Update], date: Int32, users: [Api.User], chats: [Api.Chat])
    case reset
    case updatePts(pts: Int32, ptsCount: Int32)
    case updateChannelPts(channelId: Int64, pts: Int32, ptsCount: Int32)
}
```

The grouping function sorts incoming updates by counter type:

```swift
func groupUpdates(_ updates: [Api.Update], users: [Api.User], chats: [Api.Chat],
                  date: Int32, seqRange: (Int32, Int32)?) -> [UpdateGroup] {
    var updatesWithPts: [Api.Update] = []
    var updatesWithQts: [Api.Update] = []
    var otherUpdates: [Api.Update] = []

    for update in updates {
        if let _ = apiUpdatePtsRange(update) {
            updatesWithPts.append(update)
        } else if let _ = apiUpdateQtsRange(update) {
            updatesWithQts.append(update)
        } else {
            otherUpdates.append(update)
        }
    }

    var groups: [UpdateGroup] = []
    if !updatesWithPts.isEmpty {
        groups.append(.withPts(updates: updatesWithPts, users: users, chats: chats))
    }
    if !updatesWithQts.isEmpty {
        groups.append(.withQts(updates: updatesWithQts, users: users, chats: chats))
    }
    if let seqRange = seqRange {
        groups.append(.withSeq(updates: otherUpdates, seqRange: seqRange,
                               date: date, users: users, chats: chats))
    }
    return groups
}
```

This separation matters because each counter type has its own gap detection logic. A gap in pts (missing a message) requires different recovery than a gap in seq (missing a settings update).

### Step 3: Difference Fetching

When the client starts up (or detects a gap), it calls `getDifference`:

```swift
let request = network.request(
    Api.functions.updates.getDifference(
        flags: flags,
        pts: authorizedState.pts,
        ptsLimit: nil,
        ptsTotalLimit: ptsTotalLimit,
        date: authorizedState.date,
        qts: authorizedState.qts,
        qtsLimit: nil
    )
)
```

The server responds with everything that changed since the client's last known state:

- **`.difference`** — here are all the new messages, chats, users, and other updates. Apply them and you're caught up.
- **`.differenceSlice`** — here's a partial batch. Apply these and call getDifference again for more.
- **`.differenceTooLong`** — you've been offline so long that the server can't provide a diff. Reset and re-sync.

The `differenceSlice` response is particularly important for clients that have been offline for days — the server sends updates in manageable chunks rather than one massive response.

### Step 4: AccountMutableState — The Accumulator

Updates are not applied to the database one-by-one. Instead, they're accumulated in an `AccountMutableState` — an in-memory representation of pending changes:

```swift
struct AccountMutableState {
    let initialState: AccountInitialState
    let branchOperationIndex: Int

    var state: AuthorizedAccountState.State  // Current pts, qts, seq, date
    var peers: [PeerId: Peer]
    var channelStates: [PeerId: AccountStateChannelState]
    var storedMessages: Set<MessageId>
    var readInboxMaxIds: [PeerId: MessageId]

    var operations: [AccountStateMutationOperation] = []
}
```

The `operations` array accumulates database mutations as they're computed. There are **50+ operation types**:

```swift
enum AccountStateMutationOperation {
    case AddMessages([StoreMessage], AddMessagesLocation)
    case DeleteMessages([MessageId])
    case EditMessage(MessageId, StoreMessage)
    case UpdateMedia(MediaId, Media?)
    case ReadInbox(MessageId)
    case ReadOutbox(MessageId, Int32?)
    case ResetReadState(peerId: PeerId, namespace: MessageId.Namespace, ...)
    case UpdateChannelState(PeerId, Int32)
    case UpdateNotificationSettings(...)
    case MergeApiChats([Api.Chat])
    case MergeApiUsers([Api.User])
    case UpdatePinnedItemIds(groupId: PeerGroupId, ...)
    case UpdateChatListInclusion(peerId: PeerId, ...)
    // ... 35+ more
}
```

### Step 5: Update Application

The function `finalStateWithUpdatesAndServerTime` (6,045 lines in `AccountStateManagementUtils.swift`) is the heart of the sync engine. It processes each update type:

```swift
for update in sortedUpdates(updates) {
    switch update {
    case let .updateNewMessage(data):
        if let message = StoreMessage(apiMessage: data.message,
                                       accountPeerId: accountPeerId) {
            updatedState.addMessages([message], location: .UpperHistoryBlock)
        }

    case let .updateDeleteMessages(data):
        updatedState.deleteMessagesWithGlobalIds(data.messages)

    case let .updateEditMessage(data):
        if let message = StoreMessage(apiMessage: data.message,
                                       accountPeerId: accountPeerId) {
            updatedState.editMessage(messageId, message: message)
        }

    case let .updateReadHistoryInbox(data):
        updatedState.readInbox(MessageId(
            peerId: data.peer.peerId,
            namespace: Namespaces.Message.Cloud,
            id: data.maxId
        ))

    // ... 100+ more cases
    }
}
```

This is a **100+ case switch statement** — one case for every type of update the Telegram API can send. Each case translates the raw API update into one or more `AccountStateMutationOperation` values.

## Channel-Specific State Tracking

Regular chats share a global `pts` counter, but channels have their own **per-channel pts**. This is necessary because channels can have millions of subscribers, and the server can't include every channel update in the global difference.

```swift
var channelStates: [PeerId: AccountStateChannelState]

case let .updateDeleteChannelMessages(data):
    let peerId = PeerId(namespace: Namespaces.Peer.CloudChannel, id: data.channelId)
    if let previousState = updatedState.channelStates[peerId] {
        if previousState.pts >= data.pts {
            // Old update — already applied, skip
        } else if previousState.pts + data.ptsCount == data.pts {
            // Contiguous — apply it
            updatedState.deleteMessages(data.messages.map { id in
                MessageId(peerId: peerId, namespace: Namespaces.Message.Cloud, id: id)
            })
            updatedState.updateChannelState(peerId, pts: data.pts)
        } else {
            // GAP! Need to fetch channel difference
            Logger.shared.log("State",
                "channel \(peerId) delete pts hole \(previousState.pts) + \(ptsCount) != \(pts)")
            missingUpdatesFromChannels.insert(peerId)
        }
    }
```

When a channel gap is detected, the channel is added to `missingUpdatesFromChannels`. After the main update processing completes, the state manager fetches channel-specific differences for each affected channel.

## Hole Management

Holes are gaps in the message history — ranges where the client knows messages exist but hasn't downloaded them yet. The system in `Holes.swift` (1,281 lines) manages filling these gaps:

### When Holes Appear

1. **pts gap** — the expected pts doesn't match, some message updates were missed
2. **Channel invalidation** — the server says the channel state is too old
3. **User scrolling** — the user scrolls to a region of the history that hasn't been loaded yet
4. **Initial load** — only the most recent messages are fetched at startup

### Filling Holes

```swift
func fetchMessageHistoryHole(
    accountPeerId: PeerId,
    source: FetchMessageHistoryHoleSource,
    postbox: Postbox,
    peerInput: FetchMessageHistoryHoleThreadInput,
    namespace: MessageId.Namespace,
    direction: MessageHistoryViewRelativeHoleDirection,
    space: MessageHistoryHoleOperationSpace,
    count rawCount: Int
) -> Signal<FetchMessageHistoryHoleResult?, NoError>
```

The hole filler makes different API calls depending on the context:

- **Regular chats**: `messages.getHistory` with offset/limit parameters
- **Channels**: `channels.getMessages` with specific message IDs
- **Forum topics**: `messages.getReplies` with the topic's thread ID

Holes are tracked per-peer and per-namespace in the Postbox. When a hole is filled, the new messages are inserted and the hole entry is removed. If the response indicates more messages exist, the hole is shrunk rather than removed.

## State Replay: Writing to the Database

Once all updates are processed, the accumulated operations are applied to Postbox in a single atomic transaction:

```swift
func replayFinalState(
    accountManager: AccountManager<TelegramAccountManagerTypes>,
    postbox: Postbox,
    accountPeerId: PeerId,
    mediaBox: MediaBox,
    transaction: Transaction,
    finalState: AccountFinalState,
    // ...
) -> AccountReplayedFinalState? {
    // 1. Verify state consistency
    if !skipVerification {
        let verified = verifyTransaction(transaction, finalState: finalState.state)
        if !verified {
            Logger.shared.log("State", "failed to verify final state")
            return nil
        }
    }

    // 2. Apply all queued operations
    for operation in optimizedOperations(finalState.state.operations) {
        switch operation {
        case let .AddMessages(messages, location):
            transaction.addMessages(messages, location: location)

        case let .DeleteMessages(messageIds):
            transaction.removeMessages(messageIds)

        case let .EditMessage(messageId, message):
            transaction.updateMessage(messageId, update: { _ in message })

        case let .ReadInbox(messageId):
            transaction.updatePeerChatInternal(peerId: messageId.peerId) { current in
                current.withUpdatedReadState(
                    current.readState.withUpdatedIncomingReadId(messageId.id)
                )
            }

        case let .UpdateChannelState(peerId, pts):
            transaction.updateCachedPeerData(peerId: peerId) { current in
                var state = current as? ChannelState
                state?.pts = pts
                return state
            }
        // ... 50+ more operation handlers
        }
    }

    // 3. Update global state counters
    transaction.setState(
        currentState.changedState(
            AuthorizedAccountState.State(
                pts: finalState.state.state.pts,
                qts: finalState.state.state.qts,
                date: finalState.state.state.date,
                seq: finalState.state.state.seq
            )
        )
    )

    // 4. Return events for UI layer
    return AccountReplayedFinalState(
        state: finalState,
        addedIncomingMessageIds: addedIncomingMessageIds,
        deletedMessageIds: deletedMessageIds,
        updatedTypingActivities: updatedTypingActivities,
        // ... 20+ event types
    )
}
```

The `optimizedOperations` call merges redundant operations — for example, if a message is added and then immediately edited, the two operations are combined into a single add with the edited content.

The atomic transaction is essential. If the app crashes mid-sync, no partial state is written. The next launch will re-fetch from the last committed state.

## Event Emission

After replay, events are emitted to UI subscribers through signal pipes:

```swift
public var notificationMessages: Signal<[([Message], PeerGroupId, Bool,
    MessageHistoryThreadData?)]>, NoError> {
    return self.notificationMessagesPipe.signal()
}

public var deletedMessages: Signal<[DeletedMessageId], NoError> {
    return self.deletedMessagesPipe.signal()
}

public var storyUpdates: Signal<[InternalStoryUpdate], NoError> {
    return self.storyUpdatesPipe.signal()
}
```

These signals drive the UI layer. When `ChatHistoryListNodeImpl` observes a change in the `MessageHistoryView`, it creates new list items and triggers an animated update. When the notification service observes `notificationMessages`, it shows system notifications for incoming messages.

## Branching and Merging

For parallel operations (multiple channels needing updates simultaneously), the state supports branching:

```swift
func branch() -> AccountMutableState {
    return AccountMutableState(..., branchOperationIndex: self.operations.count)
}

mutating func merge(_ other: AccountMutableState) {
    for i in other.branchOperationIndex ..< other.operations.count {
        self.addOperation(other.operations[i])
    }
}
```

A branch captures the current operation count as a watermark. After the branch completes its work, its operations (from the watermark onward) are merged back into the main state. This allows channel difference fetches to run in parallel without interfering with each other.

## A Concrete Example: Processing a New Message

To make this concrete, let's trace what happens when your friend sends you "Hello":

1. **Server pushes `updateShortMessage`** via WebSocket:
   ```
   { id: 12345, userId: 987, message: "Hello",
     pts: 5001, pts_count: 1, date: 1704067200 }
   ```

2. **`UpdateMessageService` receives it**, reconstructs a full `Api.Message`, wraps it in `Api.Update.updateNewMessage`, groups it as `.withPts(...)`.

3. **`AccountStateManager` collects it** into a `collectUpdateGroups` operation. After a brief timeout (to batch with any other rapid updates), it promotes to `processUpdateGroups`.

4. **`initialStateWithUpdateGroups`** reads the current state from Postbox: `{ pts: 5000, qts: 42, seq: 100, date: 1704067100 }`.

5. **`finalStateWithUpdatesAndServerTime`** processes the update:
   - Checks: `5000 + 1 == 5001` ✓ (contiguous)
   - Creates `StoreMessage` from the `Api.Message`
   - Adds `.AddMessages([message], .UpperHistoryBlock)` to operations
   - Advances state to `pts: 5001, date: 1704067200`

6. **`replayFinalState`** writes to Postbox:
   - `transaction.addMessages([storeMessage], location: .UpperHistoryBlock)`
   - `transaction.setState(newState with pts: 5001)`
   - Returns event: `addedIncomingMessageIds: [MessageId(12345)]`

7. **Event emission**: `notificationMessagesPipe.putNext([message])` — the UI and notification systems are notified.

8. **ChatHistoryListNodeImpl** observes the Postbox change via its `MessageHistoryView` subscription, creates a new `ChatMessageItemImpl`, and inserts it into the list with an animation.

Total time from WebSocket to screen: typically under 50ms.

## Error Recovery

The sync engine handles several failure modes:

### Gap Recovery
When `pts + pts_count != expected_pts`, the client falls back to `getDifference`. This fetches all updates from the server between the client's last known state and the current state, filling the gap.

### Channel Too Long
When the server responds with `updateChannelTooLong`, the client's channel state is too far behind. It resets the channel's local state and re-fetches from scratch.

### Difference Too Long
When `getDifference` returns `.differenceTooLong`, the client has been offline so long that the server can't provide a diff. The client resets its entire state and starts fresh — a full re-sync.

### AUTH_KEY_DUPLICATED
If the auth key is detected as a duplicate (the user logged in from another device), the state manager triggers a session termination.

## Why This Architecture Works

The pts/qts/seq counter system is elegant because it provides three guarantees:

1. **Completeness** — every update is accounted for. Gaps are detected immediately via counter arithmetic.
2. **Ordering** — updates within a counter domain are applied in order. The sorted processing ensures deterministic state.
3. **Idempotency** — receiving the same update twice is harmless. If `current_pts >= update_pts`, the update is a no-op.

Combined with atomic Postbox transactions and the operation-queue architecture, this ensures the client's local database is always a consistent, complete snapshot of the server's state — even across network interruptions, app suspensions, and device switches.
