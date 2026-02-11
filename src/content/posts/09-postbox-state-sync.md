---
title: "Postbox State Sync: How Local and Remote State Converge"
description: "The complete state synchronization pipeline — from MTProto getDifference to Postbox transactions, hole filling, the AccountStateManager operation queue, and SeedConfiguration bootstrap."
published: 2025-07-05
tags:
  - Persistence
toc: true
lang: en
abbrlink: 09-postbox-state-sync
pin: 17
---

In the previous three posts we covered Postbox's storage layer (ValueBox + Tables), its data organization (70+ table classes), and its reactive view system. But we haven't addressed the hardest question: **how does data get into Postbox in the first place, and how does it stay in sync with the server?**

Telegram doesn't poll. It doesn't do full refreshes. Instead, it maintains a **state cursor** — a set of sequence numbers that tell the server "I have everything up to here, give me what's new." When the app launches, or when the WebSocket delivers an update it can't process in sequence, the app asks the server for a *difference* — the delta between its state and the server's. That delta is then decomposed into mutation operations, applied atomically to Postbox, and propagated to views.

This post traces the entire pipeline.

## The State Cursor: pts, qts, date, seq

Telegram's update protocol is built around four sequence numbers, stored as `AuthorizedAccountState.State`:

| Field | What it tracks |
|-------|---------------|
| `pts` | **Message-related events** — new messages, reads, edits, deletes for private chats and groups. Incremented by the server on each relevant event. |
| `qts` | **Secret chat events** — encrypted messages use a separate sequence. |
| `date` | **Timestamp** — the server's Unix timestamp at the time of the last processed update. |
| `seq` | **Update container sequence** — the sequence number for "combined" update containers, ensuring they're processed in order. |

These four numbers are Postbox's "position in the event stream." When the app calls `updates.getDifference(pts: ..., date: ..., qts: ...)`, the server responds with everything that happened after that position.

Channels add a fifth dimension: each channel has its own `pts` tracked separately, because channel updates are independent of the user's personal update stream.

### Where the State Lives

The state cursor is stored in Postbox's `MetadataTable` — a simple key-value table using integer keys:

```swift
final class MetadataTable: Table {
    private enum MetadataKey: Int32 {
        case UserVersion = 1
        case State = 2
        case TransactionStateVersion = 3
        case MasterClientId = 4
        case RemoteContactCount = 6
    }

    private var cachedState: PostboxCoding?

    func state() -> PostboxCoding? {
        if let cachedState = self.cachedState {
            return cachedState
        }
        if let value = self.valueBox.get(self.table, key: self.key(.State)) {
            if let state = PostboxDecoder(buffer: value).decodeRootObject() {
                self.cachedState = state
                return state
            }
        }
        return nil
    }

    func setState(_ state: PostboxCoding) {
        self.cachedState = state
        let encoder = PostboxEncoder()
        encoder.encodeRootObject(state)
        self.valueBox.set(self.table, key: self.key(.State), value: encoder.readBufferNoCopy())
    }
}
```

The state is cached in memory and serialized with Postbox's custom binary encoder. The `Transaction` class exposes this to the rest of the app:

```swift
// Transaction API
public func getState() -> PostboxCoding? {
    return self.postbox?.getState()
}

public func setState(_ state: PostboxCoding) {
    self.postbox?.setState(state)
}
```

The `TransactionStateVersion` key serves a different purpose: it's a monotonically increasing counter that increments on every non-empty transaction. Other processes (like the Notification Service Extension) compare this version to detect when the main app has modified the database.

## The Three-Phase State Pipeline

When the app needs to sync with the server (at launch, after reconnection, or when updates arrive out of order), the state flows through three phases:

```
[Server API Response]
        ↓
Phase 1: initialStateWithDifference() → AccountMutableState
        ↓
Phase 2: finalStateWithDifference()   → AccountFinalState
        ↓
Phase 3: replayFinalState()           → AccountReplayedFinalState
        ↓
[Postbox Transaction Applied]
```

### Phase 1: Initial State Snapshot

Before processing a difference, the system captures the current state from Postbox:

```swift
final class AccountInitialState {
    let state: AuthorizedAccountState.State   // pts, qts, date, seq
    let peerIds: Set<PeerId>                  // All known peer IDs
    let channelStates: [PeerId: AccountStateChannelState]  // Per-channel pts
    let peerChatInfos: [PeerId: PeerChatInfo]
    let locallyGeneratedMessageTimestamps: [PeerId: [(MessageId.Namespace, Int32)]]
    let cloudReadStates: [PeerId: PeerReadState]
    let channelsToPollExplicitely: Set<PeerId>
}
```

This snapshot is the "before" picture. It captures everything needed to determine what the difference actually changes — peer IDs that might need resolving, channel states that might need updating, read states that might need merging.

### Phase 2: Building the Final State

`finalStateWithDifference()` parses the API response and builds an `AccountMutableState` — an intermediate representation that accumulates all changes as an ordered list of operations:

```swift
struct AccountMutableState {
    let initialState: AccountInitialState
    var operations: [AccountStateMutationOperation] = []  // Ordered list of changes
    var state: AuthorizedAccountState.State                // Updated cursor
    var peers: [PeerId: Peer]                              // Accumulated peer data
    var channelStates: [PeerId: AccountStateChannelState]
    var namespacesWithHolesFromPreviousState: [PeerId: [MessageId.Namespace: HoleFromPreviousState]]
    // ... 20+ more fields
}
```

The key insight is the `operations` array. Instead of directly modifying Postbox, the difference is decomposed into atomic operations:

```swift
enum AccountStateMutationOperation {
    // Message operations
    case AddMessages([StoreMessage], AddMessagesLocation)
    case DeleteMessages([MessageId])
    case EditMessage(MessageId, StoreMessage)
    case DeleteMessagesWithGlobalIds([Int32])

    // Read state
    case ReadInbox(MessageId)
    case ReadOutbox(MessageId, Int32?)
    case ResetReadState(peerId:, namespace:, maxIncomingReadId:, ...)

    // Peer updates
    case MergeApiChats([Api.Chat])
    case MergeApiUsers([Api.User])
    case MergePeerPresences([PeerId: Api.UserStatus], Bool)
    case UpdatePeer(PeerId, (Peer?) -> Peer?)
    case UpdateCachedPeerData(PeerId, (CachedPeerData?) -> CachedPeerData?)

    // State updates
    case UpdateState(AuthorizedAccountState.State)
    case UpdateChannelState(PeerId, Int32)
    case UpdateChannelInvalidationPts(PeerId, Int32)

    // Notification settings
    case UpdateNotificationSettings(AccountStateNotificationSettingsSubject, TelegramPeerNotificationSettings)
    case UpdateGlobalNotificationSettings(AccountStateGlobalNotificationSettingsSubject, MessageNotificationSettings)

    // And 80+ more operation types covering:
    // Stories, stickers, calls, secret chats, pins, threads,
    // group calls, wallpapers, star balances, reactions, polls...
}
```

Over **80 operation types** cover every possible state change in the Telegram protocol. Each operation is a self-contained instruction that can be applied to Postbox without external context.

The result wraps into `AccountFinalState`:

```swift
struct AccountFinalState {
    var state: AccountMutableState
    var shouldPoll: Bool           // Whether to getDifference again
    var incomplete: Bool           // Whether we're still processing
    var missingUpdatesFromChannels: Set<PeerId>  // Channels needing resync
}
```

The `shouldPoll` flag is important: when the server responds with `differenceSlice` (meaning "here's a chunk, but there's more"), the system needs to poll again. This repeats until the server responds with `differenceEmpty`.

### Phase 3: Replay into Postbox

`replayFinalState()` is where the mutation operations actually hit the database. It runs inside a Postbox transaction and processes every operation in order:

```swift
func replayFinalState(
    accountManager: AccountManager,
    postbox: Postbox,
    accountPeerId: PeerId,
    mediaBox: MediaBox,
    transaction: Transaction,
    finalState: AccountFinalState,
    ...
) -> AccountReplayedFinalState? {
    // Process each operation
    for operation in finalState.state.operations {
        switch operation {
        case let .AddMessages(messages, location):
            let _ = transaction.addMessages(messages, location: location)

        case let .DeleteMessages(ids):
            transaction.deleteMessages(ids, forEachMedia: { ... })

        case let .EditMessage(id, message):
            transaction.updateMessage(id, update: { ... })

        case let .ReadInbox(messageId):
            transaction.applyIncomingReadMaxId(messageId)

        case let .MergeApiChats(chats):
            for chat in chats {
                if let peer = parseTelegramGroupOrChannel(chat: chat) {
                    transaction.updatePeersInternal([peer], update: { ... })
                }
            }

        case let .UpdateState(state):
            let currentState = transaction.getState() as! AuthorizedAccountState
            transaction.setState(currentState.changedState(state))

        case let .UpdateChannelState(peerId, pts):
            transaction.setPeerChatState(peerId,
                state: ChannelState(pts: pts, ...))

        // ... 80+ more cases
        }
    }

    // Return the replayed state with derived events
    return AccountReplayedFinalState(
        state: finalState.state,
        addedIncomingMessageIds: addedIncomingMessageIds,
        updatedWebpages: updatedWebpages,
        // ... derived events for notification, UI updates, etc.
    )
}
```

Because this runs inside a single Postbox transaction, all changes are atomic. Either the entire difference is applied, or nothing is. The transaction triggers `ViewTracker.updateViews()` at commit time, so all active views update simultaneously.

### Branch and Merge: Parallel Channel Processing

`AccountMutableState` supports **branching** for parallel channel state processing:

```swift
struct AccountMutableState {
    let branchOperationIndex: Int

    func branch() -> AccountMutableState {
        // Creates a copy with branchOperationIndex set to current operations.count
        return AccountMutableState(
            initialState: self.initialState,
            operations: self.operations,
            state: self.state,
            // ... all other fields copied
            branchOperationIndex: self.operations.count
        )
    }

    mutating func merge(_ other: AccountMutableState) {
        // Merge only operations added AFTER the branch point
        for i in other.branchOperationIndex ..< other.operations.count {
            self.addOperation(other.operations[i])
        }
        // Merge accumulated peers, stories, etc.
        for (_, peer) in other.insertedPeers {
            self.peers[peer.id] = peer
        }
        self.preCachedResources.append(contentsOf: other.preCachedResources)
        self.externallyUpdatedPeerId.formUnion(other.externallyUpdatedPeerId)
    }
}
```

This is used when processing a difference that contains updates for multiple channels. Each channel's updates can be processed independently (branched), then merged back into the main state. The `branchOperationIndex` ensures that only operations added after the branch point are merged — avoiding duplicate application of operations that existed before the branch.

## The AccountStateManager: Operation Queue

The `AccountStateManager` is the orchestrator that decides when to poll for differences, how to process incoming updates, and how to handle concurrent state operations. It uses a **serial operation queue** — only one operation runs at a time:

```swift
private enum AccountStateManagerOperationContent {
    case pollDifference(Int32, AccountFinalStateEvents)
    case collectUpdateGroups([UpdateGroup], Double)
    case processUpdateGroups([UpdateGroup])
    case custom(Int32, Signal<Void, NoError>)
    case pollCompletion(Int32, [MessageId], [(Int32, ([MessageId]) -> Void)])
    case processEvents(Int32, AccountFinalStateEvents)
    case replayAsynchronouslyBuiltFinalState(AccountFinalState, () -> Void)
}

private final class AccountStateManagerOperation {
    var isRunning: Bool = false
    let content: AccountStateManagerOperationContent
}
```

The operation types form a state machine:

1. **`pollDifference`** — Calls `updates.getDifference()` on the MTProto network layer, processes the response through the three-phase pipeline, and applies the result to Postbox.

2. **`collectUpdateGroups`** — When WebSocket updates arrive in rapid succession, they're collected into a batch with a small timeout (to allow more updates to arrive) before processing.

3. **`processUpdateGroups`** — Takes collected update groups and processes them through the state pipeline. If an update's `seq` or `pts` doesn't match the expected sequence, triggers a `pollDifference` instead.

4. **`custom`** — Arbitrary one-shot operations that need to be serialized with state processing.

5. **`pollCompletion`** — Post-processing after a difference is applied (e.g., sending delivery receipts for new messages).

6. **`processEvents`** — Processes derived events from a replayed state (e.g., triggering notifications, updating call state).

7. **`replayAsynchronouslyBuiltFinalState`** — For states built asynchronously (like channel resets), replays them when the queue is ready.

The serial queue ensures that state processing is never concurrent — you never have two `getDifference` calls in flight, or an update being processed while a difference is being applied.

### Operations Are Positioned

Operations can be added at the front or back of the queue:

```swift
private enum AccountStateManagerAddOperationPosition {
    case first
    case last
}
```

A `pollDifference` is typically added at `.first` because it supersedes any pending update processing — if we need to resync, there's no point processing stale updates. Regular update groups are added at `.last` for FIFO processing.

## Hole Filling: Demand-Driven Data Fetching

We covered hole tracking in the previous post from the view side. Now let's see the server side — how holes are actually filled.

### The Managed Holes System

`ManagedMessageHistoryHolesContext` is the bridge between the Postbox view system and the network layer:

```swift
private final class ManagedMessageHistoryHolesContext {
    private var pendingEntries: [PendingEntry] = []      // Active fetch requests
    private var completedEntries: [MessageHistoryHolesViewEntry: Double] = [:]  // Cooldown tracker

    init(queue: Queue, accountPeerId: PeerId, postbox: Postbox, network: Network,
         entries: Signal<Set<MessageHistoryHolesViewEntry>, NoError>) {
        // Subscribe to the postbox's holes view
        self.currentEntriesDisposable = (entries |> deliverOn(self.queue)).start(
            next: { [weak self] entries in
                self?.update(entries: entries)
            }
        )
    }
}
```

The `update` method is the core logic:

```swift
func update(entries: Set<MessageHistoryHolesViewEntry>) {
    for entry in entries {
        // Skip recently completed holes (20-second cooldown)
        if let previousTimestamp = self.completedEntries[entry] {
            if previousTimestamp >= CFAbsoluteTimeGetCurrent() - 20.0 {
                continue
            }
        }

        switch entry.hole {
        case let .peer(peerHole):
            let key = LocationKey(peerId: peerHole.peerId,
                                  threadId: peerHole.threadId,
                                  space: entry.space)
            // Only one fetch per (peerId, threadId, space) at a time
            if !self.pendingEntries.contains(where: { $0.key == key }) {
                let disposable = MetaDisposable()
                let pendingEntry = PendingEntry(id: nextEntryId, key: key,
                                                entry: entry, disposable: disposable)
                self.pendingEntries.append(pendingEntry)

                // Start the actual network fetch
                pendingEntry.disposable.set(
                    (fetchMessageHistoryHole(
                        accountPeerId: self.accountPeerId,
                        source: .network(self.network),
                        postbox: self.postbox,
                        peerInput: .direct(peerId: peerHole.peerId,
                                          threadId: peerHole.threadId),
                        namespace: peerHole.namespace,
                        direction: entry.direction,
                        space: entry.space,
                        count: entry.count
                    ) |> deliverOn(self.queue)).start(completed: { [weak self] in
                        guard let self = self else { return }
                        self.pendingEntries.removeAll(where: { $0.id == id })
                        self.completedEntries[entry] = CFAbsoluteTimeGetCurrent()
                        // Re-evaluate — more holes may have appeared
                        self.update(entries: self.currentEntries)
                    })
                )
            }
        }
    }
}
```

Key design decisions:

1. **Deduplication by location key** — Only one fetch per `(peerId, threadId, space)` tuple runs at a time. If a view reports a hole for peer X in space `.everywhere`, and another view reports a different hole for the same peer and space, only one fetch runs.

2. **20-second cooldown** — After a hole is filled, the same hole entry is suppressed for 20 seconds. This prevents rapid re-fetching if the server returns fewer messages than expected or if the hole isn't fully resolved.

3. **Self-re-evaluation** — When a fetch completes, `update()` is called again with the current entries. This handles cascading holes: filling one hole might reveal another hole further back in the conversation.

4. **Discarded entry recycling** — When a hole disappears from the view (because the user navigated away), the pending fetch isn't immediately cancelled. It's moved to a "discarded" list for 0.5 seconds, allowing it to be reused if the user quickly navigates back.

### The fetchMessageHistoryHole Function

The actual network request is built in `Holes.swift`:

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
) -> Signal<FetchMessageHistoryHoleResult?, NoError> {
    let count = min(100, rawCount)  // API limits to 100 messages per request

    return postbox.stateView()
    |> mapToSignal { view -> Signal<AuthorizedAccountState, NoError> in
        // Wait for authorized state
        if let state = view.state as? AuthorizedAccountState {
            return .single(state)
        } else {
            return .complete()
        }
    }
    |> take(1)
    |> mapToSignal { _ -> Signal<FetchMessageHistoryHoleResult?, NoError> in
        return postbox.transaction { transaction -> (Peer?, ...) in
            // Look up the peer to construct the API request
            let peer = transaction.getPeer(peerId)
            return (peer, ...)
        }
        |> mapToSignal { (peer, ...) -> Signal<...> in
            guard let peer = peer else { return .single(emptyResult) }
            guard let inputPeer = forceApiInputPeer(peer) else { return .single(emptyResult) }

            // Build the API request based on direction and space
            let request: Signal<Api.messages.Messages, MTRpcError>

            switch space {
            case .everywhere:
                // Regular history fetch
                switch direction {
                case let .range(start, end):
                    // messages.getHistory with appropriate offsets
                    request = source.request(Api.functions.messages.getHistory(
                        peer: inputPeer,
                        offsetId: ..., offsetDate: 0, addOffset: ...,
                        limit: Int32(count), maxId: ..., minId: ..., hash: 0
                    ))
                case let .aroundId(id):
                    // Centered fetch around a specific message
                    request = source.request(Api.functions.messages.getHistory(
                        peer: inputPeer,
                        offsetId: id.id,
                        addOffset: Int32(-count / 2), ...
                    ))
                }

            case let .tag(tag):
                // Tagged search (photos, videos, documents, etc.)
                request = source.request(Api.functions.messages.search(
                    peer: inputPeer,
                    q: "",
                    filter: messageFilterForTagMask(tag)!, ...
                ))

            case let .customTag(tag, _):
                // Custom tag search (reactions, saved message tags)
                request = source.request(Api.functions.messages.search(
                    peer: inputPeer, ...
                ))
            }

            // Process the response
            return request
            |> mapToSignal { result in
                return postbox.transaction { transaction in
                    // Parse messages from API response
                    let messages = apiMessageParsing(result)
                    let peers = apiPeerParsing(result)

                    // Add peers to Postbox
                    updatePeers(transaction: transaction, peers: peers)

                    // Remove the hole range that was fetched
                    transaction.removeHole(
                        peerId: peerId, threadId: threadId,
                        namespace: namespace, space: space,
                        range: filledRange
                    )

                    // Add the fetched messages
                    let _ = transaction.addMessages(storeMessages, location: .Random)

                    // If fewer messages than expected, the hole might be partially filled
                    // The remaining hole will be re-detected by the view system
                }
            }
        }
    }
}
```

The direction parameter is critical:

```swift
public enum MessageHistoryViewRelativeHoleDirection: Equatable, Hashable {
    case range(start: MessageId, end: MessageId)  // Fill a specific range
    case aroundId(MessageId)                       // Center on a message ID
}
```

A `.range` direction fills a known gap between two message IDs. An `.aroundId` direction is used when scrolling to a specific message (e.g., "jump to reply") — the system fetches messages centered around that ID.

### The Space Dimension

Holes exist per **space**, not just per peer:

```swift
public enum MessageHistoryHoleSpace: Equatable, Hashable {
    case everywhere    // All messages
    case tag(MessageTags)  // Only messages with specific tags (photos, videos, etc.)
}
```

This means a peer can have complete general history but still have holes in its tagged views. When the user opens the "Media" tab in a chat, a tagged hole is detected, and `fetchMessageHistoryHole` fetches using `messages.search` with the appropriate filter instead of `messages.getHistory`.

## SeedConfiguration: Bootstrap and Schema

When a new account is created (or when the app is installed fresh), the database starts empty. `SeedConfiguration` defines how the initial state is bootstrapped:

```swift
public final class SeedConfiguration {
    // Initial chat list hole — triggers loading the first page of chats
    public let initializeChatListWithHole: (
        topLevel: ChatListHole?,
        groups: ChatListHole?
    )

    // Initial message holes per peer namespace
    // When a peer is first seen, these holes trigger message fetching
    public let messageHoles: [PeerId.Namespace: [MessageId.Namespace: Set<MessageTags>]]

    // Holes added during schema upgrades (for existing peers that need re-indexing)
    public let upgradedMessageHoles: [PeerId.Namespace: [MessageId.Namespace: Set<MessageTags>]]

    // Thread-specific holes (for forum topics)
    public let messageThreadHoles: (PeerId.Namespace, Int64?) -> [MessageId.Namespace]?

    // Default read states for new peers
    public let defaultMessageNamespaceReadStates: [MessageId.Namespace: PeerReadState]

    // Which message namespaces count as "chat messages" (vs service messages)
    public let chatMessagesNamespaces: Set<MessageId.Namespace>

    // How to calculate unread badges per peer type
    public let peerSummaryCounterTags: (Peer, Bool) -> PeerSummaryCounterTags

    // Whether a peer uses thread-based summary (forums)
    public let peerSummaryIsThreadBased: (Peer, Peer?) -> (value: Bool, threadsArePeers: Bool)

    // Peer namespace-level config for full-text search
    public let peerNamespacesRequiringMessageTextIndex: [PeerId.Namespace]

    // Global message tags that exist in the system (calls, etc.)
    public let existingGlobalMessageTags: GlobalMessageTags
}
```

The `initializeChatListWithHole` is particularly important: it creates a single `ChatListHole` in the empty chat list. The managed chat list holes system detects this hole and fetches the first page of dialogs from the server. As dialogs arrive, the hole shrinks. If there are more dialogs than fit in one page, a new hole is placed at the boundary, triggering another fetch when the user scrolls down.

The `messageHoles` configuration works similarly for individual chats: when a peer is first encountered, holes are created for the specified namespace/tag combinations. These holes drive the initial message loading for that chat.

### Schema Upgrades via upgradedMessageHoles

When a new version of the app needs to index messages differently (e.g., adding a new tag type for "GIF" messages), `upgradedMessageHoles` specifies which holes to insert for existing peers. This triggers re-fetching of messages that might now match the new tag criteria — without losing existing local data.

## The PeerOperationLog: Persistent Task Queue

While `AccountStateManager` handles the inbound direction (server → app), the **`PeerOperationLogTable`** handles the outbound direction (app → server). It's a persistent queue of operations that need to be sent to the server:

```swift
public struct PeerOperationLogEntry {
    public let peerId: PeerId
    public let tag: PeerOperationLogTag           // Operation type
    public let tagLocalIndex: Int32               // Per-peer ordering
    public let mergedIndex: Int32?                // Global ordering across all peers
    public let contents: PostboxCoding            // Serialized operation payload
}
```

Examples of queued operations:
- Sync read states to the server
- Upload pending media
- Sync notification settings
- Sync chat list folder ordering
- Sync pinned messages

The `mergedIndex` provides a global ordering across all peers, allowing the system to process operations in the order they were created regardless of which peer they belong to. Operations are consumed by "managed operations" — long-running signal subscribers in TelegramCore that watch the operation log and execute API calls:

```swift
// Conceptual pattern (simplified)
postbox.mergedOperationLogView(tag: .SynchronizeReadState)
|> mapToSignal { entries in
    for entry in entries {
        // Execute API call
        return network.request(Api.functions.messages.readHistory(...))
        |> mapToSignal { _ in
            // Remove the operation on success
            return postbox.transaction { transaction in
                transaction.operationLogRemoveEntry(
                    peerId: entry.peerId,
                    tag: entry.tag,
                    tagLocalIndex: entry.tagLocalIndex
                )
            }
        }
    }
}
```

The operation log survives app termination. If the app is killed while uploading media, the operation remains in the log and is retried on next launch. The `mergedIndex` system ensures ordering — you don't sync a read state for message 100 before message 50 has been synced.

## Multi-Process Coordination

Telegram iOS has multiple processes that access the database:
- **Main app** — the primary user interface
- **Notification Service Extension** — processes push notifications, may insert messages
- **Share Extension** — sends messages directly

The `MetadataTable.TransactionStateVersion` coordinates between them:

```swift
func incrementTransactionStateVersion() -> Int64 {
    var version = self.transactionStateVersion() + 1
    self.valueBox.set(self.table, key: self.key(.TransactionStateVersion), value: ...)
    return version
}
```

The main app periodically checks this version. If it doesn't match the expected value, another process has modified the database. The ViewTracker's `refreshViewsDueToExternalTransaction()` then reloads all active views from disk.

The `MasterClientId` provides another coordination mechanism — it identifies which process is the "master" (primary UI). Other processes can check this to decide whether to defer certain operations.

## The Complete Sync Flow

Let's trace what happens when the app launches after being in the background:

```
1. App launches, Account initializes
   ↓
2. Postbox opens SQLite database
   MetadataTable.state() returns last saved AuthorizedAccountState.State
   (pts: 12345, qts: 67, date: 1709251200, seq: 890)
   ↓
3. AccountStateManager starts
   Adds pollDifference operation to the queue
   ↓
4. getDifference(pts: 12345, date: 1709251200, qts: 67)
   Server responds with differenceSlice:
   - 50 new messages
   - 12 read state updates
   - 3 peer info updates
   - New state: (pts: 12395, qts: 67, date: 1709252000, seq: 895)
   ↓
5. Phase 1: initialStateWithDifference()
   Captures current AccountInitialState from Postbox
   ↓
6. Phase 2: finalStateWithDifference()
   Parses the API response into AccountMutableState
   Creates operations: [AddMessages(...), ReadInbox(...), MergeApiUsers(...), ...]
   Sets shouldPoll = true (it was a slice, more data available)
   ↓
7. Phase 3: replayFinalState()
   Opens a Postbox transaction
   Processes each operation:
     - Adds 50 messages → writes to messageHistoryTable
     - Updates 12 read states → writes to readStateTable
     - Updates 3 peers → writes to peerTable
     - Updates state cursor → writes to metadataTable
   Commits transaction
   ↓
8. ViewTracker.updateViews() fires
   - Chat list views update (new messages change ordering)
   - Any open message history views update
   - Unread count views update
   - All affected subscribers receive new snapshots
   ↓
9. Since shouldPoll = true, repeat from step 4
   getDifference(pts: 12395, date: 1709252000, qts: 67)
   Server responds with differenceEmpty → done
   ↓
10. Meanwhile, hole tracking kicks in
    If user opens a chat with a hole:
    - MutableMessageHistoryView detects the hole
    - ViewTracker.updateTrackedHoles() publishes it
    - ManagedMessageHistoryHolesContext calls fetchMessageHistoryHole()
    - Messages arrive, hole is removed
    - View updates automatically
```

## Channel State: Independent Cursors

Channels have their own `pts` values, tracked per-peer rather than globally. When a channel update arrives with a `pts` gap:

1. The update processing detects the gap
2. A `channelDifference` request is issued for that channel specifically
3. The channel's state is branched from the main state using `AccountMutableState.branch()`
4. The channel difference is processed independently
5. The branch is merged back using `AccountMutableState.merge()`
6. If the gap is too large (`channelDifferenceTooLong`), the channel's message history is reset with new holes

This per-channel isolation prevents a busy channel from blocking updates for other conversations.

## Summary

Postbox's state sync is a three-layer system:

1. **State cursor** (`pts`, `qts`, `date`, `seq`) — a position in the server's event stream, persisted in MetadataTable.

2. **Difference pipeline** — a three-phase transformation (initial → final → replay) that converts API responses into ordered mutation operations, applied atomically to Postbox within a transaction.

3. **Hole filling** — a demand-driven fetch system where:
   - Views detect unfetched ranges (holes)
   - ViewTracker aggregates holes from all active views
   - `ManagedMessageHistoryHolesContext` deduplicates and rate-limits fetch requests
   - `fetchMessageHistoryHole` executes API calls per space (everywhere, tag, customTag)
   - Results are applied via Postbox transactions, which automatically update views

This completes Part III of our series. We've covered Postbox from its lowest level (ValueBox key-value storage) through its table organization, its reactive view system, and now its state synchronization with the Telegram server. In Part IV, we'll move to the networking layer — MTProto, the API layer, and how data actually gets on and off the wire.
