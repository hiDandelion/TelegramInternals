---
title: "Real-Time Updates: From Push Channel to UI"
description: "How Telegram iOS receives, groups, validates, and applies real-time updates — the UpdateMessageService push receiver, UpdateGroup classification, AccountStateManager operation queue, gap detection with PTS/QTS/SEQ, and difference polling."
published: 2025-07-10
tags:
  - Networking
toc: true
lang: en
abbrlink: 13-realtime-updates
pin: 13
---

Telegram feels instant. When someone sends you a message, it appears within milliseconds — not because of polling, but because a persistent MTProto connection pushes updates in real time. The update system is the most complex piece of the networking layer, handling ordering, deduplication, gap detection, and atomic state application.

Post 9 covered how Postbox state sync works at the storage level. This post focuses on the network side — how updates arrive, how they're classified and validated, and how the system recovers when things go wrong.

## The Push Channel

The primary MTProto connection (to the user's home datacenter) serves dual purposes: it sends API requests and receives **server-initiated updates**. These updates arrive as `Api.Updates` messages — unsolicited messages pushed by the server whenever something changes.

`UpdateMessageService` is the MTMessageService plugin that intercepts these:

```swift
// UpdateMessageService.swift, line 7
class UpdateMessageService: NSObject, MTMessageService {
    var peerId: PeerId!
    let pipe: ValuePipe<[UpdateGroup]> = ValuePipe()
    var mtProto: MTProto?

    func mtProto(_ mtProto: MTProto!,
                 receivedMessage message: MTIncomingMessage!,
                 authInfoSelector: MTDatacenterAuthInfoSelector,
                 networkType: Int32) {
        if let updates = (message.body as? BoxedMessage)?.body as? Api.Updates {
            self.addUpdates(updates)
        }
    }
}
```

It's registered with MTProto alongside `MTRequestMessageService` during account initialization. Every incoming MTProto message that decodes as `Api.Updates` is routed here.

## Api.Updates: The Seven Delivery Formats

The server sends updates in seven different formats, optimized for different scenarios:

```swift
public enum Updates {
    case updates(Cons_updates)              // Full batch: updates[] + users[] + chats[] + date + seq
    case updatesCombined(Cons_updatesCombined)  // Merged batch: seqStart..seq range
    case updateShort(Cons_updateShort)       // Single update + date, no entities
    case updateShortMessage(...)             // Optimized: new PM, inlined fields
    case updateShortChatMessage(...)         // Optimized: new group message, inlined fields
    case updateShortSentMessage(...)         // Response to sendMessage: pts only
    case updatesTooLong                      // "You're behind, poll getDifference"
}
```

The "short" variants exist for performance. When someone sends a simple text message, the server doesn't need to pack the full `updates` wrapper with empty arrays — it sends `updateShortMessage` with all fields inlined.

`UpdateMessageService.addUpdates` normalizes all seven formats into a uniform representation. The short message variants are expanded into full `Api.Message` + `Api.Update.updateNewMessage` objects:

```swift
// UpdateMessageService.swift, line 64
case let .updateShortChatMessage(data):
    let generatedMessage = Api.Message.message(.init(
        flags: data.flags, flags2: 0, id: data.id,
        fromId: .peerUser(.init(userId: data.fromId)),
        peerId: Api.Peer.peerChat(.init(chatId: data.chatId)),
        date: data.date, message: data.message,
        media: Api.MessageMedia.messageMediaEmpty,
        // ... all other fields from the short payload
    ))
    let update = Api.Update.updateNewMessage(.init(
        message: generatedMessage,
        pts: data.pts, ptsCount: data.ptsCount
    ))
    let groups = groupUpdates([update], users: [], chats: [],
                              date: data.date, seqRange: nil)
    self.putNext(groups)
```

`updatesTooLong` triggers a full reset — the server is saying "you've been disconnected too long, I can't send individual updates anymore":

```swift
case .updatesTooLong:
    self.pipe.putNext([.reset])
```

Session changes also trigger resets:

```swift
func mtProtoDidChangeSession(_ mtProto: MTProto!) {
    self.pipe.putNext([.reset])
}
```

## UpdateGroup: Classification by Ordering Mechanism

After normalization, updates are classified into `UpdateGroup` values based on how they're ordered:

```swift
// UpdateGroup.swift, line 5
enum UpdateGroup {
    case withPts(updates: [Api.Update], users: [Api.User], chats: [Api.Chat])
    case withQts(updates: [Api.Update], users: [Api.User], chats: [Api.Chat])
    case withSeq(updates: [Api.Update], seqRange: (Int32, Int32), date: Int32,
                 users: [Api.User], chats: [Api.Chat])
    case withDate(updates: [Api.Update], date: Int32, users: [Api.User], chats: [Api.Chat])
    case reset
    case updatePts(pts: Int32, ptsCount: Int32)
    case updateChannelPts(channelId: Int64, pts: Int32, ptsCount: Int32)
    case ensurePeerHasLocalState(id: PeerId)
}
```

The classification happens in `groupUpdates`:

```swift
// UpdateGroup.swift, line 206
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
    if updatesWithPts.count != 0 {
        groups.append(.withPts(updates: updatesWithPts, users: users, chats: chats))
    }
    if updatesWithQts.count != 0 {
        groups.append(.withQts(updates: updatesWithQts, users: users, chats: chats))
    }
    if let seqRange = seqRange {
        groups.append(.withSeq(updates: otherUpdates, seqRange: seqRange,
                               date: date, users: users, chats: chats))
    } else {
        groups.append(.withDate(updates: otherUpdates, date: date,
                                users: users, chats: chats))
    }
    return groups
}
```

A single server push can produce multiple groups — the PTS updates are separated from QTS updates because they use different counters.

### Which Updates Carry PTS?

`apiUpdatePtsRange` extracts the PTS range from update types that have it:

```swift
func apiUpdatePtsRange(_ update: Api.Update) -> (Int32, Int32)? {
    switch update {
    case let .updateNewMessage(data):       return (data.pts, data.ptsCount)
    case let .updateDeleteMessages(data):   return (data.pts, data.ptsCount)
    case let .updateEditMessage(data):      return (data.pts, data.ptsCount)
    case let .updateReadHistoryInbox(data):  return (data.pts, data.ptsCount)
    case let .updateReadHistoryOutbox(data): return (data.pts, data.ptsCount)
    case let .updateReadMessagesContents(data): return (data.pts, data.ptsCount)
    case let .updateWebPage(data):          return (data.pts, data.ptsCount)
    case let .updateFolderPeers(data):      return (data.pts, data.ptsCount)
    case let .updatePinnedMessages(data):   return (data.pts, data.ptsCount)
    default: return nil
    }
}
```

Only `updateNewEncryptedMessage` carries QTS:

```swift
func apiUpdateQtsRange(_ update: Api.Update) -> (Int32, Int32)? {
    switch update {
    case let .updateNewEncryptedMessage(data): return (data.qts, 1)
    default: return nil
    }
}
```

Everything else (typing indicators, user status, chat settings changes) goes through the SEQ or date path.

### Sorting

Before processing, updates within each group are sorted by their counter values:

```swift
func ptsUpdates(_ groups: [UpdateGroup]) -> [PtsUpdate] {
    var result: [PtsUpdate] = []
    // ... collect all PTS updates
    result.sort(by: { $0.ptsRange.0 < $1.ptsRange.0 })
    return result
}
```

This ensures updates are applied in the correct order even if the server delivers them slightly out of sequence.

## AccountStateManager: The Operation Queue

`AccountStateManager` orchestrates all update processing through a serial operation queue:

```swift
// AccountStateManager.swift
private enum AccountStateManagerOperationContent {
    case pollDifference(Int32, AccountFinalStateEvents)
    case collectUpdateGroups([UpdateGroup], Double)
    case processUpdateGroups([UpdateGroup])
    case custom(Int32, Signal<Void, NoError>)
    case pollCompletion(Int32, [MessageId], [(Int32, ([MessageId]) -> Void)])
    case processEvents(Int32, AccountFinalStateEvents)
    case replayAsynchronouslyBuiltFinalState(AccountFinalState, () -> Void)
}
```

The queue ensures that only one operation runs at a time. Updates can't be applied while a difference poll is in progress, and difference results can't be applied while updates are being processed.

### Initialization

On account login, the manager registers the update service with MTProto:

```swift
public func reset() {
    self.queue.async {
        if self.updateService == nil {
            self.updateService = UpdateMessageService(peerId: self.accountPeerId)
            self.updateServiceDisposable.set(
                self.updateService!.pipe.signal().start(next: { [weak self] groups in
                    self?.addUpdateGroups(groups)
                })
            )
            self.network.mtProto.add(self.updateService)
        }
        // Start with getDifference to catch up from last known state
        self.replaceOperations(with: .pollDifference(self.getNextId(), AccountFinalStateEvents()))
        self.startFirstOperation()
    }
}
```

The first operation is always `pollDifference` — even if the app was only backgrounded for a second, there might be updates that arrived while the push channel was disconnected.

### Update Batching

When real-time updates arrive, they're batched to avoid excessive database transactions:

```swift
func addUpdateGroups(_ groups: [UpdateGroup]) {
    self.queue.async {
        if let last = self.operations.last {
            switch last.content {
            case .pollDifference, .processUpdateGroups, .custom, .pollCompletion,
                 .processEvents, .replayAsynchronouslyBuiltFinalState:
                // Other operation running — queue for later
                self.addOperation(.collectUpdateGroups(groups, 0.0), position: .last)

            case let .collectUpdateGroups(currentGroups, timeout):
                // Already collecting — merge into existing batch
                let merged = AccountStateManagerOperation(
                    content: .collectUpdateGroups(currentGroups + groups, timeout)
                )
                merged.isRunning = last.isRunning
                self.operations[self.operations.count - 1] = merged
                self.startFirstOperation()
            }
        }
    }
}
```

The `collectUpdateGroups` operation waits up to 2 seconds for more updates to arrive before processing the batch. This reduces the number of Postbox transactions when many updates arrive in quick succession (e.g., catching up after reconnection).

## Gap Detection: The PTS Validation Algorithm

The core of update reliability is gap detection. When processing PTS updates, the system checks whether each update follows sequentially from the last known state:

```swift
// AccountStateManagementUtils.swift — simplified
for update in currentPtsUpdates {
    if updatedState.state.pts >= update.ptsRange.0 {
        // Already seen this update — skip (deduplication)
        continue
    }

    if ptsUpdatesAfterHole.isEmpty &&
       updatedState.state.pts == update.ptsRange.0 - update.ptsRange.1 {
        // Sequential: currentPts + ptsCount == newPts
        // Apply the update
        updatedState.mergeChats(update.chats)
        updatedState.mergeUsers(update.users)
        collectedUpdates.append(update)
        updatedState.updateState(pts: update.ptsRange.0, ...)
    } else {
        // Gap detected!
        Logger.shared.log("State",
            "update pts hole: \(update.ptsRange.0) != " +
            "\(updatedState.state.pts) + \(update.ptsRange.1)")
        ptsUpdatesAfterHole.append(update)
    }
}
```

The math: if the client's current PTS is 100, and an update arrives with `pts = 102, ptsCount = 2`, that's sequential (100 + 2 = 102). But if the update has `pts = 105, ptsCount = 2`, there's a gap (100 + 2 ≠ 105) — updates with PTS 101-103 are missing.

When a gap is detected, the system falls back to `getDifference` to fetch the missing updates.

### Channel PTS Independence

Channels (supergroups) have their own PTS counters, independent of the user's personal PTS:

```swift
case let .updateNewChannelMessage(data):
    if let previousState = updatedState.channelStates[peerId] {
        if previousState.pts >= pts {
            // Old update — skip
        } else if previousState.pts + ptsCount == pts {
            // Sequential — apply
            updatedState.addMessages([message], location: .UpperHistoryBlock)
            updatedState.updateChannelState(peerId, pts: pts)
        } else {
            // Channel-specific gap
            missingUpdatesFromChannels.insert(peerId)
        }
    }
```

Channel gaps trigger `updates.getChannelDifference` for just that channel, without disrupting the processing of personal updates.

## Difference Polling: Recovery from Gaps

When gaps are detected (or on fresh start), `AccountStateManager` calls `updates.getDifference`:

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
- **newMessages** — messages the client missed
- **newEncryptedMessages** — secret chat messages
- **otherUpdates** — status changes, read receipts, etc.
- **chats** and **users** — entities referenced by the updates
- **state** — new PTS/QTS/SEQ/date values

The response type is `Api.updates.Difference`:

```swift
case .difference(data):
    // Full difference — apply all and update state
case .differenceSlice(data):
    // Partial difference — there's more to fetch
    // Apply what we have, then poll again with updated state
case .differenceTooLong(data):
    // Too far behind — reset PTS to server's value
    // Chat histories may have gaps that need hole-filling
case .differenceEmpty(data):
    // All caught up
```

`differenceSlice` is important — when there are thousands of missed updates, the server sends them in pages. The client applies each page, updates its state cursor, and polls again until it gets `differenceEmpty`.

## The Three-Phase Processing Pipeline

Updates (whether from real-time push or getDifference) go through three phases:

### Phase 1: Build Mutable State

```swift
initialStateWithUpdateGroups(postbox: postbox, groups: groups)
    -> Signal<AccountMutableState, NoError>
```

Reads the current state from Postbox and creates a mutable copy that accumulates changes.

### Phase 2: Apply Updates

```swift
finalStateWithUpdateGroups(
    accountPeerId: peerId, postbox: postbox, network: network,
    state: mutableState, groups: groups, asyncResetChannels: channels
) -> Signal<AccountFinalState, NoError>
```

Each update group is processed in counter order. PTS, QTS, and SEQ counters are validated. Gaps trigger fallback to getDifference. The result is a `AccountFinalState` — an immutable snapshot of all changes to apply.

### Phase 3: Replay to Postbox

```swift
replayFinalState(
    accountManager: accountManager, postbox: postbox,
    accountPeerId: peerId, mediaBox: mediaBox,
    transaction: transaction, finalState: finalState
) -> AccountReplayedFinalState?
```

The final state is replayed inside a single Postbox transaction — all messages, state changes, peer updates, and read states are applied atomically. This ensures the database never contains a partial update.

## Event Dispatch

After the Postbox transaction commits, the `AccountReplayedFinalState` contains events that need to reach the UI:

```swift
case let .processEvents(operationId, events):
    // Typing indicators
    if !events.updatedTypingActivities.isEmpty {
        strongSelf.peerInputActivityManager?.transaction { manager in
            for (chatPeerId, activities) in events.updatedTypingActivities {
                // Update typing UI per-chat
            }
        }
    }

    // New message notifications
    let messageList = events.addedIncomingMessageIds.compactMap { id in
        messagesForNotification(transaction: transaction, id: id)
    }
    strongSelf.notificationMessagesPipe.putNext(messageList)

    // Group call participant updates
    if !events.updatedGroupCallParticipants.isEmpty {
        strongSelf.groupCallParticipantUpdatesPipe.putNext(
            events.updatedGroupCallParticipants
        )
    }

    // And many more: reactions, stories, online status, etc.
```

These events flow through dedicated `ValuePipe` signals to the UI layer. The typing indicator pipe drives the "typing..." animation. The notification pipe triggers local notifications for background messages. Postbox view updates (triggered by the committed transaction) handle the rest — chat list reordering, new message display, unread badge updates.

## Connection Lifecycle

The update system's behavior depends on connection state:

**App launch:**
1. MTProto connects to home datacenter
2. `AccountStateManager.reset()` registers `UpdateMessageService`
3. `pollDifference` fetches everything missed since last `state.pts`
4. Postbox is updated, views refresh, UI shows current state
5. Real-time push channel is now active

**Background/foreground transition:**
1. App enters background → MTProto may disconnect (iOS kills the socket)
2. App enters foreground → MTProto reconnects
3. `mtProtoDidChangeSession` fires → `reset` event
4. `pollDifference` catches up on missed updates
5. Push channel resumes

**Network switch (Wi-Fi ↔ cellular):**
1. TCP connection drops
2. `MTTcpConnectionBehaviour` detects failure
3. Exponential backoff before reconnection
4. `MTNetworkAvailability` monitors system connectivity
5. On reconnection, session may or may not have changed
6. If session changed → `reset` → `getDifference`
7. If session survived → real-time updates continue seamlessly

**Long disconnection (hours/days):**
1. `getDifference` may return `differenceSlice` (paginated)
2. Multiple rounds of polling until caught up
3. Or `differenceTooLong` if server has compacted the delta
4. In `differenceTooLong`, PTS jumps forward and chat histories may have holes
5. Holes are filled on-demand when the user opens a chat (via `ManagedMessageHistoryHoles`)

## Tracing a Real-Time Message

Following a message from sender to receiver's screen:

1. **Sender** sends message → server acknowledges with `updateShortSentMessage(pts: N, ptsCount: 1)`
2. **Server** pushes to receiver's MTProto connection: `updateShortMessage(id, userId, message, pts: M, ptsCount: 1)`
3. **MTProto** decrypts and parses → `MTIncomingMessage` with body = `Api.Updates.updateShortMessage`
4. **UpdateMessageService.mtProto(_:receivedMessage:)** extracts the update
5. **addUpdates** expands `updateShortMessage` into full `Api.Message` + `Api.Update.updateNewMessage`
6. **groupUpdates** classifies it as `.withPts` (because updateNewMessage has pts)
7. **pipe.putNext** sends `[UpdateGroup.withPts(...)]` to AccountStateManager
8. **addUpdateGroups** merges into the current `collectUpdateGroups` batch
9. After timeout (or immediately if no pending batch), **processUpdateGroups** runs
10. **Gap check:** `currentPts + 1 == M` → no gap, apply the update
11. **AccountMutableState** adds the message and advances PTS to M
12. **replayFinalState** commits to Postbox in a single transaction
13. **ViewTracker.updateViews** propagates the new message to active views
14. **MessageHistoryView** and **ChatListView** emit updates through their pipes
15. **ChatController** receives the new message → animates it into the list
16. **ChatListController** receives the updated chat → reorders it to the top
17. **NotificationMessagesPipe** triggers local notification if app is backgrounded

Total time from server push to UI update: typically **10-50ms** on a warmed connection. The persistent MTProto connection avoids the TCP handshake overhead of HTTP, and the update processing pipeline is optimized for the common case of sequential updates with no gaps.

## Key Takeaways

1. **Seven delivery formats, one processing path.** The server optimizes wire format (short messages for common cases), but `UpdateMessageService` normalizes everything into `UpdateGroup` before processing. Downstream code never deals with format variants.

2. **Classification by ordering mechanism.** PTS, QTS, and SEQ updates are fundamentally different — they use different counters and different gap-detection logic. Separating them into groups at classification time simplifies downstream processing.

3. **Serial operation queue.** `AccountStateManager` never processes updates concurrently. The operation queue ensures getDifference and real-time updates don't race, and that Postbox transactions are applied in the correct order.

4. **Batching amortizes cost.** The 2-second collection window during burst updates reduces the number of Postbox transactions. During normal operation (one message at a time), the timeout is effectively zero.

5. **Channels are independent.** Each channel has its own PTS counter and can be synced independently. A gap in one channel doesn't block processing of personal messages or other channels.

6. **Atomic replay.** All changes from a batch of updates are applied in a single Postbox transaction. The database is never in a state where some updates from a batch are applied and others aren't.
