---
title: "Postbox Views: Live Queries That Power Every Screen"
description: "How Telegram's view system turns database transactions into real-time UI updates — the Mutable/Immutable pattern, ViewTracker, CombinedView, hole tracking, and the type-safe EngineData layer."
published: 2025-07-04
tags:
  - Persistence
toc: true
lang: en
abbrlink: 08-postbox-views
pin: 18
---

In the previous two posts we explored Postbox's storage layer — the `ValueBox` key-value abstraction and the 70+ `Table` classes built on top of it. But storage alone doesn't explain how Telegram keeps every screen in the app perfectly in sync with the database. When a new message arrives via WebSocket, the chat list badge updates, the message list scrolls, the peer's "last message" preview changes, and any open profile screens refresh — all simultaneously, with no manual invalidation.

The mechanism that makes this possible is the **view system**. This post dissects it completely.

## The Core Problem: Reactive Database Reads

Most iOS apps treat persistence as request-response: you ask for data, you get a snapshot, you render it. If the data changes later, you either poll, use `NSFetchedResultsController`, or wire up manual notifications. Each approach has well-known problems — polling wastes CPU, `NSFetchedResultsController` only works with Core Data, and manual notifications are error-prone and tend to miss edge cases.

Telegram's approach is different. Every database read is a **live query** — a signal that emits an initial snapshot, then automatically re-emits whenever the underlying data changes. You subscribe once and receive updates forever. The update mechanism is precise: views only re-emit when data they actually depend on changes, not on every transaction.

Here's what it looks like from the consumer side:

```swift
// Subscribe to a peer's data — this signal never completes
let peerView: Signal<PeerView, NoError> = postbox.combinedView(
    keys: [.peer(peerId: peerId, components: .all)]
)
|> map { views -> PeerView in
    return views.views[.peer(peerId: peerId, components: .all)] as! PeerView
}

// The signal emits:
// 1. Immediately: current snapshot from disk
// 2. Later: every time this peer's data changes in any transaction
disposable = peerView.start(next: { view in
    // view.peers, view.cachedData, view.notificationSettings, etc.
    // All updated atomically, consistently
    updateUI(with: view)
})
```

No polling. No notification names. No manual cache invalidation. The view system handles everything.

## The Two-Layer View Protocol

Every view in Postbox follows a strict two-class pattern: a **mutable internal view** that tracks changes, and an **immutable snapshot** that gets delivered to subscribers.

```swift
// Protocol for internal mutable views — lives on the Postbox queue
protocol MutablePostboxView: AnyObject {
    func replay(postbox: PostboxImpl, transaction: PostboxTransaction) -> Bool
    func refreshDueToExternalTransaction(postbox: PostboxImpl) -> Bool
    func immutableView() -> PostboxView
}

// Protocol for external immutable views — safe to use on any thread
public protocol PostboxView: AnyObject {
}
```

Three methods, three responsibilities:

1. **`replay(postbox:transaction:)`** — Called after every transaction. The view examines the `PostboxTransaction` change record to determine if anything relevant changed. Returns `true` if the view was modified, `false` to suppress the update. This is the **selectivity mechanism** — each view type implements its own logic for determining relevance.

2. **`refreshDueToExternalTransaction(postbox:)`** — Called when another process (like a notification extension) modified the database file externally. The view must reload its entire state from disk because there's no `PostboxTransaction` change record available.

3. **`immutableView()`** — Creates a frozen snapshot by copying all mutable state into an immutable `PostboxView` subclass. This snapshot is what subscribers actually receive, ensuring thread safety without locks.

### Why Two Classes Instead of One?

This pattern exists because of the threading model. `MutablePostboxView` instances live on the Postbox serial queue and are mutated in-place during `replay()`. But subscribers receive updates on their own queues (often the main queue). If the mutable view were passed directly, you'd need locks on every property access, or risk data races.

Instead, `immutableView()` creates a value-type-like copy at the moment of emission:

```swift
// PeerView.swift — the immutable snapshot
public final class PeerView: PostboxView {
    public let peerId: PeerId                          // let, not var
    public let cachedData: CachedPeerData?             // let, not var
    public let notificationSettings: PeerNotificationSettings?
    public let peers: [PeerId: Peer]
    public let peerPresences: [PeerId: PeerPresence]
    public let messages: [MessageId: Message]
    public let media: [MediaId: Media]
    public let peerIsContact: Bool
    public let groupId: PeerGroupId?
    public let storyStats: PeerStoryStats?
    public let memberStoryStats: [PeerId: PeerStoryStats]
    public let associatedCachedData: [PeerId: CachedPeerData]

    init(_ mutableView: MutablePeerView) {
        self.peerId = mutableView.peerId
        self.cachedData = mutableView.cachedData
        self.notificationSettings = mutableView.notificationSettings
        self.peers = mutableView.peers              // Dictionary copy
        self.peerPresences = mutableView.peerPresences
        self.messages = mutableView.messages
        self.media = mutableView.media
        self.peerIsContact = mutableView.peerIsContact
        self.groupId = mutableView.groupId
        self.storyStats = mutableView.storyStats
        self.memberStoryStats = mutableView.memberStoryStats
        self.associatedCachedData = mutableView.associatedCachedData
    }
}
```

Every property is `let`. The class is a snapshot frozen in time. Swift's copy-on-write semantics for Dictionary and Array make the copying efficient — the actual data is shared until mutation, and the immutable class never mutates.

## MutablePeerView: Anatomy of a Complete View

Let's trace `MutablePeerView` as a representative example, since it shows every aspect of the pattern.

### Initialization — Loading from Disk

When a subscriber first connects, the view loads its complete state from the Postbox tables:

```swift
final class MutablePeerView: MutablePostboxView {
    let peerId: PeerId
    let contactPeerId: PeerId
    let components: PeerViewComponents
    var notificationSettings: PeerNotificationSettings?
    var cachedData: CachedPeerData?
    var peers: [PeerId: Peer] = [:]
    var peerPresences: [PeerId: PeerPresence] = [:]
    var messages: [MessageId: Message] = [:]
    var media: [MediaId: Media] = [:]
    var peerIsContact: Bool
    var groupId: PeerGroupId?

    init(postbox: PostboxImpl, peerId: PeerId, components: PeerViewComponents) {
        self.peerId = peerId

        // Resolve the peer and its associations
        var peerIds = Set<PeerId>()
        peerIds.insert(peerId)

        if let peer = postbox.peerTable.get(peerId) {
            if let associatedPeerId = peer.associatedPeerId {
                peerIds.insert(associatedPeerId)
            }
            if let additionalAssociatedPeerId = peer.additionalAssociatedPeerId {
                peerIds.insert(additionalAssociatedPeerId)
            }
        }

        // Load cached data and expand the peer set
        self.cachedData = postbox.cachedPeerDataTable.get(contactPeerId)
        if let cachedData = self.cachedData {
            peerIds.formUnion(cachedData.peerIds)
        }

        // Bulk-load all relevant peers and presences
        for id in peerIds {
            if let peer = postbox.peerTable.get(id) {
                self.peers[id] = peer
            }
            if let presence = postbox.peerPresenceTable.get(id) {
                self.peerPresences[id] = presence
            }
        }

        // Load notification settings, contact status, etc.
        self.notificationSettings = postbox.peerNotificationSettingsTable.getEffective(peerId)
        self.peerIsContact = postbox.contactsTable.isContact(peerId: contactPeerId)
        self.groupId = postbox.chatListIndexTable.get(peerId: peerId).inclusion.groupId
    }
}
```

This initial load reads from **six different tables** (`peerTable`, `cachedPeerDataTable`, `peerPresenceTable`, `peerNotificationSettingsTable`, `contactsTable`, `chatListIndexTable`) and assembles a complete view of the peer. The subscriber gets all this data atomically in a single snapshot.

### Replay — Selective Update Logic

The `replay` method is where the performance magic happens. Instead of reloading everything, it examines the `PostboxTransaction` change record to find what changed:

```swift
func replay(postbox: PostboxImpl, transaction: PostboxTransaction) -> Bool {
    let updatedPeers = transaction.currentUpdatedPeers
    let updatedNotificationSettings = transaction.currentUpdatedPeerNotificationSettings
    let updatedCachedPeerData = transaction.currentUpdatedCachedPeerData
    let updatedPeerPresences = transaction.currentUpdatedPeerPresences

    var updated = false

    // Check if cached data for our peer changed
    if let cachedData = updatedCachedPeerData[self.contactPeerId]?.updated {
        self.cachedData = cachedData
        updated = true
        // Re-expand the peer set since cachedData.peerIds may have changed
    }

    // Check if any peer in our set was updated
    for id in relevantPeerIds {
        if let peer = updatedPeers[id] {
            self.peers[id] = peer
            updated = true
        }
        if let presence = updatedPeerPresences[id] {
            self.peerPresences[id] = presence
            updated = true
        }
    }

    // Check notification settings
    if let (_, settings) = updatedNotificationSettings[self.peerId] {
        self.notificationSettings = settings
        updated = true
    }

    // Check contact list changes
    if let replaceContactPeerIds = transaction.replaceContactPeerIds {
        let isNowContact = replaceContactPeerIds.contains(self.contactPeerId)
        if self.peerIsContact != isNowContact {
            self.peerIsContact = isNowContact
            updated = true
        }
    }

    return updated  // Only true if something this view cares about changed
}
```

The key insight: when a message is sent, `MutablePeerView.replay` returns `false` unless that message affects cached data or peer state. The view doesn't waste cycles on irrelevant changes. This is why Telegram can have dozens of active views and still process transactions quickly — each view does O(1) work checking whether it's affected, not O(n) scanning.

## The ViewTracker: Managing Active Subscriptions

The `ViewTracker` is the central registry for all active view subscriptions. It uses the `Bag` collection (from SwiftSignalKit) to store pairs of `(MutableView, ValuePipe)`:

```swift
final class ViewTracker {
    private let queue: Queue

    // Typed views — each with their own Bag
    private var chatListViews = Bag<(MutableChatListView, ValuePipe<(ChatListView, ViewUpdateType)>)>()
    private var messageHistoryViews = Bag<(MutableMessageHistoryView, ValuePipe<(MessageHistoryView, ViewUpdateType)>)>()
    private var peerViews = Bag<(MutablePeerView, ValuePipe<PeerView>)>()
    private var messageViews = Bag<(MutableMessageView, ValuePipe<MessageView>)>()
    private var preferencesViews = Bag<(MutablePreferencesView, ValuePipe<PreferencesView>)>()
    private var multiplePeersViews = Bag<(MutableMultiplePeersView, ValuePipe<MultiplePeersView>)>()
    private var itemCollectionsViews = Bag<(MutableItemCollectionsView, ValuePipe<ItemCollectionsView>)>()
    private var failedMessageIdsViews = Bag<(MutableFailedMessageIdsView, ValuePipe<FailedMessageIdsView>)>()
    private var unreadMessageCountsViews = Bag<(MutableUnreadMessageCountsView, ValuePipe<UnreadMessageCountsView>)>()
    private var postboxStateViews = Bag<(MutablePostboxStateView, ValuePipe<PostboxStateView>)>()
    private var contactPeerIdsViews = Bag<(MutableContactPeerIdsView, ValuePipe<ContactPeerIdsView>)>()
    private var combinedViews = Bag<(CombinedMutableView, ValuePipe<CombinedView>)>()

    // Singleton views (only one instance shared across all subscribers)
    private let messageHistoryHolesView = MutableMessageHistoryHolesView()
    private let messageHistoryHolesViewSubscribers = Bag<ValuePipe<MessageHistoryHolesView>>()
    private let chatListHolesView = MutableChatListHolesView()
    private let chatListHolesViewSubscribers = Bag<ValuePipe<ChatListHolesView>>()

    private var unsentMessageView: UnsentMessageHistoryView
    private var synchronizeReadStatesView: MutableSynchronizePeerReadStatesView
    // ...15+ Bag properties total
}
```

Each `Bag` entry pairs a mutable view with its `ValuePipe` — the pipe is the connection back to the subscriber's signal. When a view is updated, the pipe pushes the new immutable snapshot downstream.

### The Add/Remove Pattern

Every view type follows the same registration pattern:

```swift
func addPeerView(_ view: MutablePeerView) -> (
    Bag<(MutablePeerView, ValuePipe<PeerView>)>.Index,
    Signal<PeerView, NoError>
) {
    let record = (view, ValuePipe<PeerView>())
    let index = self.peerViews.add(record)
    return (index, record.1.signal())
}

func removePeerView(_ index: Bag<...>.Index) {
    self.peerViews.remove(index)
}
```

The returned `index` is stored by the caller and used to unsubscribe later. The returned `Signal` is the live stream of immutable snapshots. `Bag` provides O(1) add and O(1) remove, which matters because view registration happens frequently (every screen transition creates/destroys views).

### The UpdateViews Pipeline

After every transaction commits, `PostboxImpl` calls `viewTracker.updateViews()`. This is the most important method in the entire view system — it replays the transaction against every active view:

```swift
func updateViews(postbox: PostboxImpl, currentTransaction: Transaction,
                 transaction: PostboxTransaction) {
    var updateTrackedHoles = false

    // 1. State views — simple, just check if state changed
    if let currentUpdatedState = transaction.currentUpdatedState {
        for (mutableView, pipe) in self.postboxStateViews.copyItems() {
            if mutableView.replay(updatedState: currentUpdatedState) {
                pipe.putNext(PostboxStateView(mutableView))
            }
        }
    }

    // 2. Message history views — complex, with hole tracking
    for (mutableView, pipe) in self.messageHistoryViews.copyItems() {
        var updated = false
        let previousPeerIds = mutableView.peerIds

        if mutableView.replay(postbox: postbox, transaction: transaction) {
            updated = true
        }

        // Determine update type for hole management
        var updateType: ViewUpdateType = .Generic
        // ... check if hole operations affect this view's peer ...

        mutableView.updatePeerIds(transaction: transaction)
        if mutableView.peerIds != previousPeerIds {
            updateType = .UpdateVisible
            let _ = mutableView.refreshDueToExternalTransaction(postbox: postbox)
            updated = true
        }

        if updated {
            updateTrackedHoles = true
            pipe.putNext((MessageHistoryView(mutableView), updateType))
        }
    }

    // 3. Message views
    for (mutableView, pipe) in self.messageViews.copyItems() {
        let operations = transaction.currentOperationsByPeerId[mutableView.messageId.peerId]
        if operations != nil || !transaction.updatedMedia.isEmpty {
            if mutableView.replay(postbox: postbox, operations: operations ?? [],
                                  updatedMedia: transaction.updatedMedia) {
                pipe.putNext(MessageView(mutableView))
            }
        }
    }

    // 4. Chat list views — with render step
    for (mutableView, pipe) in self.chatListViews.copyItems() {
        let context = MutableChatListViewReplayContext()
        if mutableView.replay(postbox: postbox, ..., transaction: transaction, context: context) {
            mutableView.complete(postbox: postbox, context: context)
            mutableView.render(postbox: postbox)
            pipe.putNext((ChatListView(mutableView), .Generic))
        }
    }

    // 5-15. Every other view type follows the same pattern...
    // peer views, unread counts, preferences, item collections, etc.

    // 16. Combined views
    for (mutableView, pipe) in self.combinedViews.copyItems() {
        let result = mutableView.replay(postbox: postbox, transaction: transaction)
        if result.updated {
            pipe.putNext(mutableView.immutableView())
        }
        if result.updateTrackedHoles {
            updateTrackedHoles = true
        }
    }

    // 17. Update hole tracking for fetch-on-demand
    if updateTrackedHoles {
        self.updateTrackedHoles()
    }
}
```

Important details in this flow:

- **`copyItems()`** — The `Bag` method creates a snapshot of all entries. This is crucial because a view's `replay()` or its subscriber's callback could add or remove views during iteration.
- **Selective pipe delivery** — `pipe.putNext()` is only called when `replay()` returns `true`. Most views return `false` for most transactions, so the pipe stays silent.
- **Chat list render phase** — Chat list views have a three-step update: `replay()` detects changes, `complete()` resolves additional data, and `render()` converts intermediate entries into fully resolved entries with peers and messages. This layered approach avoids repeatedly resolving the same data.

## CombinedView: Multi-Key Subscriptions

While specific view types like `MutablePeerView` and `MutableChatListView` have dedicated Bag slots in the ViewTracker, the **CombinedView** pattern is the general-purpose mechanism. It wraps multiple `MutablePostboxView` instances under different `PostboxViewKey` keys:

```swift
final class CombinedMutableView {
    let views: [PostboxViewKey: MutablePostboxView]

    func replay(postbox: PostboxImpl, transaction: PostboxTransaction)
        -> (updated: Bool, updateTrackedHoles: Bool) {
        var anyUpdated = false
        for (_, view) in self.views {
            if view.replay(postbox: postbox, transaction: transaction) {
                anyUpdated = true
            }
        }
        return (anyUpdated, updateTrackedHoles)
    }

    func immutableView() -> CombinedView {
        var result: [PostboxViewKey: PostboxView] = [:]
        for (key, view) in self.views {
            result[key] = view.immutableView()
        }
        return CombinedView(views: result)
    }
}
```

And the public immutable counterpart:

```swift
public final class CombinedView {
    public let views: [PostboxViewKey: PostboxView]
}
```

This lets consumers subscribe to multiple data items with a single signal:

```swift
let signal = postbox.combinedView(keys: [
    .peer(peerId: peerId, components: .all),
    .cachedPeerData(peerId: peerId),
    .unreadCounts(items: [.peer(id: peerId, handleThreads: false)])
])
|> map { combinedView -> (PeerView, CachedPeerDataView, UnreadMessageCountsView) in
    let peer = combinedView.views[.peer(...)] as! PeerView
    let cached = combinedView.views[.cachedPeerData(...)] as! CachedPeerDataView
    let unread = combinedView.views[.unreadCounts(...)] as! UnreadMessageCountsView
    return (peer, cached, unread)
}
```

The CombinedView emits whenever **any** of its constituent views changes. This is exactly what a UI screen needs: one signal that fires when any visible data changes, delivering a consistent snapshot of everything the screen displays.

## PostboxViewKey: The View Factory

The `PostboxViewKey` enum is the registry of all possible view types. With **50+ cases**, it covers every data query pattern in the app:

```swift
public enum PostboxViewKey: Hashable {
    // Messages and chat history
    case historyView(HistoryView)
    case messages(Set<MessageId>)
    case messageGroup(id: MessageId)
    case topChatMessage(peerIds: [PeerId])
    case deletedMessages(peerId: PeerId)
    case globalMessageTags(globalTag: GlobalMessageTags, position: MessageIndex, count: Int, ...)

    // Peers
    case peer(peerId: PeerId, components: PeerViewComponents)
    case basicPeer(PeerId)
    case peerPresences(peerIds: Set<PeerId>)
    case contacts(accountPeerId: PeerId?, includePresences: Bool)
    case isContact(id: PeerId)

    // Chat list
    case chatListIndex(id: PeerId)
    case peerChatInclusion(PeerId)
    case allChatListHoles(PeerGroupId)
    case additionalChatListItems

    // Notification settings
    case peerNotificationSettings(peerIds: Set<PeerId>)
    case pendingPeerNotificationSettings
    case peerNotificationSettingsBehaviorTimestampView

    // State and data
    case peerChatState(peerId: PeerId)
    case preferences(keys: Set<ValueBoxKey>)
    case preferencesPrefix(keyPrefix: ValueBoxKey)
    case orderedItemList(id: Int32)
    case cachedPeerData(peerId: PeerId)
    case cachedItem(ItemCacheEntryId)
    case notice(key: NoticeEntryKey)

    // Unread counts
    case unreadCounts(items: [UnreadMessageCountsItem])
    case combinedReadState(peerId: PeerId, handleThreads: Bool)
    case historyTagSummaryView(tag: MessageTags, peerId: PeerId, ...)
    case historyCustomTagSummariesView(peerId: PeerId, ...)
    case historyTagInfo(peerId: PeerId, tag: MessageTags)

    // Actions and sync
    case pendingMessageActions(type: PendingMessageActionType)
    case pendingMessageActionsSummary(type: ..., peerId: ..., namespace: ...)
    case invalidatedMessageHistoryTagSummaries(peerId: ..., threadId: ..., ...)
    case synchronizeGroupMessageStats
    case localMessageTag(LocalMessageTags)

    // Items and collections (stickers, etc.)
    case itemCollectionInfos(namespaces: [ItemCollectionId.Namespace])
    case itemCollectionIds(namespaces: [ItemCollectionId.Namespace])
    case itemCollectionInfo(id: ItemCollectionId)

    // Stories
    case storySubscriptions(key: PostboxStorySubscriptionsKey)
    case storiesState(key: PostboxStoryStatesKey)
    case storyItems(peerId: PeerId)
    case storyExpirationTimeItems
    case peerStoryStats(peerIds: Set<PeerId>)
    case story(id: StoryId)

    // Thread and forum
    case messageHistoryThreadIndex(id: PeerId, summaryComponents: ...)
    case messageHistoryThreadInfo(peerId: PeerId, threadId: Int64)
    case peerTimeoutAttributes

    // Saved messages
    case savedMessagesIndex(peerId: PeerId)
    case savedMessagesStats(peerId: PeerId)

    // Interface state
    case chatInterfaceState(peerId: PeerId)
}
```

The factory function `postboxViewForKey()` maps each key to its concrete `MutablePostboxView`:

```swift
func postboxViewForKey(postbox: PostboxImpl, key: PostboxViewKey) -> MutablePostboxView {
    switch key {
    case let .peer(peerId, components):
        return MutablePeerView(postbox: postbox, peerId: peerId, components: components)
    case let .basicPeer(peerId):
        return MutableBasicPeerView(postbox: postbox, peerId: peerId)
    case let .preferences(keys):
        return MutablePreferencesView(postbox: postbox, keys: keys)
    case let .cachedPeerData(peerId):
        return MutableCachedPeerDataView(postbox: postbox, peerId: peerId,
                                         trackAssociatedMessages: false)
    case let .orderedItemList(id):
        return MutableOrderedItemListView(postbox: postbox, collectionId: id)
    case let .storyItems(peerId):
        return MutableStoryItemsView(postbox: postbox, peerId: peerId)
    // ... 50+ more cases
    }
}
```

This is a classic factory pattern that keeps the view creation logic centralized while allowing the `CombinedView` to work with any combination of view types.

### PeerViewComponents: Opt-In Data Loading

Notice the `PeerViewComponents` option set in the `.peer` key:

```swift
public struct PeerViewComponents: OptionSet {
    public static let cachedData = PeerViewComponents(rawValue: 1 << 0)
    public static let subPeers = PeerViewComponents(rawValue: 1 << 1)
    public static let messages = PeerViewComponents(rawValue: 1 << 2)
    public static let groupId = PeerViewComponents(rawValue: 1 << 3)
    public static let storyStats = PeerViewComponents(rawValue: 1 << 4)

    public static let all: PeerViewComponents = [.cachedData, .subPeers, .messages, .groupId, .storyStats]
}
```

This is a performance optimization: callers can request only the data they need. A simple peer name lookup uses `.basicPeer` (no cached data, no messages, no story stats). A full profile screen uses `.peer(peerId:, components: .all)`. This avoids loading expensive data (like story stats or associated messages) when it's not needed.

## Connecting Views to SwiftSignalKit

The `PostboxImpl` class exposes views through methods that return `Signal`:

```swift
// PostboxImpl
public func combinedView(keys: [PostboxViewKey]) -> Signal<CombinedView, NoError> {
    return self.transactionSignal { subscriber, transaction in
        // 1. Create mutable views for each key
        var views: [PostboxViewKey: MutablePostboxView] = [:]
        for key in keys {
            views[key] = postboxViewForKey(postbox: self, key: key)
        }
        let view = CombinedMutableView(views: views)

        // 2. Register with ViewTracker (returns signal from ValuePipe)
        let (index, signal) = self.viewTracker.addCombinedView(view)

        // 3. Emit initial snapshot immediately
        subscriber.putNext(view.immutableView())

        // 4. Forward all future updates from the pipe
        let disposable = signal.start(next: { next in
            subscriber.putNext(next)
        })

        // 5. Cleanup on disposal
        return ActionDisposable { [weak self] in
            disposable.dispose()
            if let strongSelf = self {
                strongSelf.queue.async {
                    strongSelf.viewTracker.removeCombinedView(index)
                }
            }
        }
    }
}
```

The `transactionSignal` wrapper ensures everything runs on the Postbox serial queue. This is critical: view creation, initial data loading, and registration all happen atomically within a transaction, so no updates can be missed between the initial load and the registration.

The lifecycle:

1. **Subscribe** → `transactionSignal` dispatches onto the Postbox queue
2. **Create** → Mutable views load their initial state from tables
3. **Register** → Views are added to the ViewTracker's Bag
4. **Initial emit** → `subscriber.putNext(view.immutableView())` delivers the first snapshot
5. **Live updates** → After each future transaction, `ViewTracker.updateViews()` calls `replay()`, and if updated, pushes through the `ValuePipe`
6. **Dispose** → The view is removed from the Bag, the pipe stops, and the mutable view is deallocated

## Hole Tracking: Demand-Driven Data Fetching

One of the most sophisticated features of the view system is **automatic hole tracking**. When a `MutableMessageHistoryView` is registered, the ViewTracker inspects it to find any "holes" — ranges of message IDs that haven't been fetched from the server yet.

```swift
private func updateTrackedHoles() {
    var firstHolesAndTags = Set<MessageHistoryHolesViewEntry>()

    // Check all registered message history views
    for (view, _) in self.messageHistoryViews.copyItems() {
        if let (hole, direction, count, userId) = view.firstHole() {
            let space: MessageHistoryHoleOperationSpace
            if let tag = view.tag {
                space = .tag(tag)  // or .customTag
            } else {
                space = .everywhere
            }
            firstHolesAndTags.insert(MessageHistoryHolesViewEntry(
                hole: hole, direction: direction, space: space,
                count: count, userId: userId
            ))
        }
    }

    // Also check message history views inside combined views
    for (view, _) in self.combinedViews.copyItems() {
        for (_, subview) in view.views {
            if let subview = subview as? MutableMessageHistoryView {
                if let (hole, direction, count, userId) = subview.firstHole() {
                    // Same logic...
                }
            }
        }
    }

    // Publish updated holes — this drives network fetches
    if self.messageHistoryHolesView.update(firstHolesAndTags) {
        for subscriber in self.messageHistoryHolesViewSubscribers.copyItems() {
            subscriber.putNext(MessageHistoryHolesView(self.messageHistoryHolesView))
        }
    }
}
```

The `MessageHistoryHolesView` is a **singleton view** — there's only one instance shared across all subscribers, unlike per-screen views. This is because hole tracking is a global concern: TelegramCore subscribes to it and decides which holes to fill from the network.

The flow:

1. User opens a chat → `MutableMessageHistoryView` is created
2. The view finds messages 1-50 in the database, but a hole at 51-100
3. `updateTrackedHoles()` publishes this hole
4. TelegramCore's hole filler subscribes, sees the hole, makes an API call
5. The API response inserts messages 51-100 into Postbox
6. The insertion transaction triggers `replay()` on the message history view
7. The view updates, hole shrinks or disappears
8. `updateTrackedHoles()` runs again, reflects the new state

This is **demand-driven data fetching** — data is only requested from the server when a view is actively displaying a hole. If no one is looking at chat X, its holes are never filled. If the user scrolls to an old part of the conversation, the hole appears in the view, gets tracked, and gets filled — automatically.

### ViewUpdateType: Informing the UI

The view system doesn't just say "data changed." It tells the UI **how** it changed:

```swift
public enum ViewUpdateType: Equatable {
    case Initial              // First emission after subscription
    case InitialUnread(MessageIndex)  // Initial with unread anchor
    case Generic              // Normal data update
    case FillHole             // A hole was filled (don't animate)
    case UpdateVisible        // Peer associations changed (reload)
}
```

This matters for animations. When a new message arrives (`.Generic`), the UI can animate the insertion. When a hole is filled (`.FillHole`), the UI should reload without animation — dozens of messages appeared at once and animating each would be chaotic.

## Singleton Views: Global Operational Signals

While most views are per-subscriber (one `MutablePeerView` per open profile screen), some views are **singletons** — one shared mutable instance with multiple pipe subscribers:

```swift
// ViewTracker — singleton views
private let messageHistoryHolesView = MutableMessageHistoryHolesView()
private let messageHistoryHolesViewSubscribers = Bag<ValuePipe<MessageHistoryHolesView>>()

private let chatListHolesView = MutableChatListHolesView()
private let chatListHolesViewSubscribers = Bag<ValuePipe<ChatListHolesView>>()

private var unsentMessageView: UnsentMessageHistoryView
private let unsendMessageIdsViewSubscribers = Bag<ValuePipe<UnsentMessageIdsView>>()

private var synchronizeReadStatesView: MutableSynchronizePeerReadStatesView
private let synchronizePeerReadStatesViewSubscribers = Bag<ValuePipe<SynchronizePeerReadStatesView>>()
```

These singletons track operational concerns:
- **`messageHistoryHolesView`** — Which message ranges need fetching (drives `fetchMessageHistoryHole`)
- **`chatListHolesView`** — Which chat list pages need loading
- **`unsentMessageView`** — Which messages are queued for sending (drives the send queue)
- **`synchronizeReadStatesView`** — Which read states need syncing to the server

The signal creation for these is slightly different — the subscriber gets the initial value immediately and shares the singleton's pipe:

```swift
func messageHistoryHolesViewSignal() -> Signal<MessageHistoryHolesView, NoError> {
    return Signal { subscriber in
        let disposable = MetaDisposable()
        self.queue.async {
            // Emit current state immediately
            subscriber.putNext(MessageHistoryHolesView(self.messageHistoryHolesView))

            // Create a pipe for future updates
            let pipe = ValuePipe<MessageHistoryHolesView>()
            let index = self.messageHistoryHolesViewSubscribers.add(pipe)

            // Forward pipe emissions to subscriber
            let pipeDisposable = pipe.signal().start(next: { view in
                subscriber.putNext(view)
            })

            // Cleanup
            disposable.set(ActionDisposable {
                self.queue.async {
                    pipeDisposable.dispose()
                    self.messageHistoryHolesViewSubscribers.remove(index)
                }
            })
        }
        return disposable
    }
}
```

## External Transaction Recovery

When another process (like the Notification Service Extension) modifies the SQLite database, the main app process can't rely on `PostboxTransaction` change records — those only exist within the process. The `refreshViewsDueToExternalTransaction` method handles this:

```swift
func refreshViewsDueToExternalTransaction(
    postbox: PostboxImpl,
    currentTransaction: Transaction,
    fetchUnsentMessageIds: () -> [MessageId],
    fetchSynchronizePeerReadStateOperations: () -> [PeerId: PeerReadStateSynchronizationOperation]
) {
    var updateTrackedHoles = false

    // Message history views reload entirely from disk
    for (mutableView, pipe) in self.messageHistoryViews.copyItems() {
        if mutableView.refreshDueToExternalTransaction(postbox: postbox) {
            pipe.putNext((MessageHistoryView(mutableView), .Generic))
            updateTrackedHoles = true
        }
    }

    // Chat list views reload
    for (mutableView, pipe) in self.chatListViews.copyItems() {
        if mutableView.refreshDueToExternalTransaction(postbox: postbox,
                                                        currentTransaction: currentTransaction) {
            mutableView.render(postbox: postbox)
            pipe.putNext((ChatListView(mutableView), .Generic))
        }
    }

    // Peer views reset
    for (mutableView, pipe) in self.peerViews.copyItems() {
        if mutableView.reset(postbox: postbox) {
            pipe.putNext(PeerView(mutableView))
        }
    }

    // Operational views re-fetch their data
    if self.unsentMessageView.refreshDueToExternalTransaction(
        fetchUnsentMessageIds: fetchUnsentMessageIds) {
        self.unsentViewUpdated()
    }
}
```

This is more expensive than a normal `replay()` because each view must reload from disk without knowing what changed. But it's necessary for multi-process correctness — the Notification Service Extension might have inserted new messages, updated read states, or modified peer data while the main app was in the background.

## TelegramEngine.EngineData: The Type-Safe Layer

The raw `PostboxViewKey` + `CombinedView` API requires casting — you get back `PostboxView` and must cast to the concrete type. This is error-prone. Telegram adds a type-safe layer on top through `TelegramEngine.EngineData`:

```swift
// The protocol chain:
protocol PostboxViewDataItem: TelegramEngineDataItem, AnyPostboxViewDataItem {
    var key: PostboxViewKey { get }
    func extract(view: PostboxView) -> Result
}

public protocol TelegramEngineDataItem {
    associatedtype Result
}
```

A concrete item looks like this:

```swift
extension TelegramEngine.EngineData.Item {
    enum Peer {
        // Type-safe peer lookup
        public struct Peer: TelegramEngineDataItem, PostboxViewDataItem {
            public typealias Result = Optional<EnginePeer>

            fileprivate var id: EnginePeer.Id

            public init(id: EnginePeer.Id) {
                self.id = id
            }

            var key: PostboxViewKey {
                return .basicPeer(self.id)
            }

            func extract(view: PostboxView) -> Result {
                guard let view = view as? BasicPeerView else {
                    preconditionFailure()
                }
                return view.peer.map(EnginePeer.init)
            }
        }
    }
}
```

The usage is fully type-safe with no casting:

```swift
// Single item — returns Signal<EnginePeer?, NoError>
let peer = engine.data.subscribe(
    TelegramEngine.EngineData.Item.Peer.Peer(id: peerId)
)

// Multiple items — returns Signal<(EnginePeer?, PeerPresence?), NoError>
let combined = engine.data.subscribe(
    TelegramEngine.EngineData.Item.Peer.Peer(id: peerId),
    TelegramEngine.EngineData.Item.Peer.Presence(id: peerId)
)

// One-shot read — same as subscribe |> take(1)
let snapshot = engine.data.get(
    TelegramEngine.EngineData.Item.Peer.Peer(id: peerId)
)
```

Under the hood, `EngineData._subscribe` collects all keys from the items, creates a single `combinedView`, and uses each item's `extract()` to pull typed results from the views:

```swift
private func _subscribe(items: [AnyPostboxViewDataItem]) -> Signal<[Any], NoError> {
    var keys = Set<PostboxViewKey>()
    for item in items {
        for key in item.keys(data: self) {
            keys.insert(key)
        }
    }
    return self.postbox.combinedView(keys: Array(keys))
    |> map { views -> [Any] in
        var results: [Any] = []
        for item in items {
            results.append(item._extract(data: self, views: views.views))
        }
        return results
    }
}
```

The `subscribe` overloads (up to 4-tuples) provide compile-time type checking:

```swift
public func subscribe<T0: TelegramEngineDataItem>(_ t0: T0) -> Signal<T0.Result, NoError> {
    return self._subscribe(items: [t0 as! AnyPostboxViewDataItem])
    |> map { results -> T0.Result in
        return results[0] as! T0.Result
    }
}

public func subscribe<T0: TelegramEngineDataItem, T1: TelegramEngineDataItem>(
    _ t0: T0, _ t1: T1
) -> Signal<(T0.Result, T1.Result), NoError> {
    return self._subscribe(items: [t0 as! AnyPostboxViewDataItem, t1 as! AnyPostboxViewDataItem])
    |> map { results -> (T0.Result, T1.Result) in
        return (results[0] as! T0.Result, results[1] as! T1.Result)
    }
}
```

There's also `EngineDataMap` for batch lookups:

```swift
// Fetch multiple peers at once
let peers = engine.data.subscribe(
    EngineDataMap(peerIds.map { TelegramEngine.EngineData.Item.Peer.Peer(id: $0) })
)
// Returns Signal<[PeerId: EnginePeer?], NoError>
```

This collapses N peer lookups into a single `combinedView` subscription with N keys — one signal, one subscription, N results.

## The Complete Data Flow

Let's trace a complete cycle from WebSocket message to UI update:

```
1. WebSocket delivers a new message
   ↓
2. TelegramCore creates a Postbox transaction
   postbox.transaction { transaction in
       transaction.addMessages([message], location: .UpperHistoryBlock)
   }
   ↓
3. PostboxImpl.addMessages writes to multiple tables
   - messageHistoryTable.addMessage(...)       → writes message data
   - messageHistoryIndexTable.add(...)         → writes timestamp index
   - messageHistoryTagsTable.add(...)          → updates tag indices
   - chatListIndexTable.setTopMessageIndex()   → updates chat ordering
   - peerTable.set(...)                        → updates last message
   All changes recorded in currentOperationsByPeerId, currentUpdatedPeers, etc.
   ↓
4. Transaction commits
   - All tables call beforeCommit() (flush write-behind caches)
   - SQLite COMMIT
   - PostboxTransaction is assembled from all current* dictionaries
   ↓
5. viewTracker.updateViews(transaction:) is called
   - Iterates all Bags of active views
   - Each view's replay() checks the transaction for relevant changes
   - Views that changed: pipe.putNext(immutableView)
   - Views that didn't change: nothing happens (no wasted emissions)
   ↓
6. ValuePipe delivers to Signal subscribers
   - Signal was created via postbox.combinedView(keys:)
   - Subscriber is on the main queue (via |> deliverOnMainQueue)
   ↓
7. UI updates
   - ChatListController sees new chat order → reloads table
   - ChatController sees new message → inserts row with animation
   - Badge counter sees new unread count → updates tab bar
   All from the SAME transaction, consistently, atomically.
```

## Performance Characteristics

The view system achieves several performance goals:

**O(1) relevance checking**: Most views check 1-2 dictionary lookups in `PostboxTransaction` to determine relevance. A `MutablePeerView` for peer X checks `updatedPeers[X]`, `updatedCachedPeerData[X]`, etc. — constant time regardless of transaction size.

**Lazy rendering**: Message history views store `IntermediateMessage` entries and only fully resolve them (loading peers, media, associated messages) in the `render()` phase, which happens lazily or on demand. This avoids loading heavy data for messages that might be scrolled past.

**Copy-on-write snapshots**: The `immutableView()` call copies dictionaries and arrays, but Swift's COW semantics mean the actual data is shared until mutation. Since immutable views are never mutated, the copy is essentially free.

**Single-pass updates**: All views are updated in a single pass through the ViewTracker after each transaction. There's no cascading — a view update doesn't trigger another transaction, which would trigger more view updates. The system is strictly one-directional: writes → transaction → view updates.

**Demand-driven holes**: Network requests for missing data are only triggered when a view is actively observing a hole. No background prefetching, no wasted bandwidth.

## Summary

The Postbox view system is a custom reactive database query layer built on three key ideas:

1. **Mutable/Immutable split** — Internal mutable views are replayed on the Postbox queue; external immutable snapshots are safe for any thread.
2. **Selective replay** — Each view examines the `PostboxTransaction` change record and only re-emits when relevant data changed, keeping update costs proportional to actual changes, not total data.
3. **ViewTracker orchestration** — A central manager holds all active views in `Bag` collections, iterates them after every transaction, and pushes updates through `ValuePipe` to `Signal` subscribers.

On top of this, `CombinedView` enables multi-key subscriptions, `PostboxViewKey` provides a factory for 50+ view types, and `TelegramEngine.EngineData` adds compile-time type safety. The hole tracking system creates a demand-driven feedback loop between UI visibility and network fetching.

In the next post, we'll close out Part III by examining how Postbox synchronizes with the server — the state management protocol, hole filling strategies, and how local and remote state converge.
