---
title: "Postbox: Telegram's Custom SQLite Persistence Layer"
description: "Why Telegram built its own database framework instead of using Core Data or Realm, and how Postbox turns raw SQLite into a reactive, type-safe persistence engine."
published: 2026-02-10
tags:
  - Persistence
toc: true
lang: en
abbrlink: 06-postbox-overview
pin: 20
---

Every message you see in Telegram, every contact, every sticker pack, every read state, every notification setting — it all lives in **Postbox**, a custom persistence layer that wraps SQLite into a reactive, type-safe database framework. Understanding Postbox is essential because it's the data foundation that every other module depends on.

This post covers the architecture, design motivations, and core abstractions. The [next three posts](/posts/07-postbox-tables/) cover tables, views, and synchronization in detail.

## Why Not Core Data, Realm, or GRDB?

Telegram's data requirements are unusual for an iOS app:

1. **Scale**: A power user might have 50,000+ chats, millions of messages, thousands of media items. Core Data's in-memory coordinator model buckles under this scale.

2. **Reactivity**: The UI must update instantly when data changes — a new message arrives, a read state updates, a contact comes online. This means the database needs to push changes to active views, not wait for the UI to poll.

3. **Custom serialization**: Telegram uses a binary serialization format (not JSON, not Codable) with type-hash-based polymorphic decoding. This predates Swift's `Codable` by years and is optimized for incremental reading.

4. **Encryption**: The database must be encrypted at rest using SQLCipher. This needs to be transparent to the application layer.

5. **Single-writer, multi-reader**: All writes happen on one dedicated queue. Views can read from any queue but only observe snapshots.

6. **Hole management**: Message history isn't contiguous — there are gaps ("holes") that get filled lazily as the user scrolls. No existing database framework models this.

Postbox addresses all of these by providing three layers of abstraction above raw SQLite:

```
┌─────────────────────────────────────────────┐
│  Views (Live Reactive Queries)              │
│  PeerView, MessageHistoryView, ChatListView │
├─────────────────────────────────────────────┤
│  Tables (Business Logic)                    │
│  PeerTable, MessageHistoryTable, etc.       │
├─────────────────────────────────────────────┤
│  ValueBox (Key-Value Storage)               │
│  SqliteValueBox wrapping SQLCipher          │
└─────────────────────────────────────────────┘
```

## Layer 1: ValueBox — The Key-Value Foundation

At the bottom is `ValueBox`, a protocol that abstracts key-value storage:

```swift
// Postbox/Sources/ValueBox.swift

public enum ValueBoxKeyType: Int32 {
    case binary
    case int64
}

public struct ValueBoxTable {
    let id: Int32
    let keyType: ValueBoxKeyType
    let compactValuesOnCreation: Bool
}

public protocol ValueBox {
    func begin()
    func commit()
    func checkpoint()

    func get(_ table: ValueBoxTable, key: ValueBoxKey) -> ReadBuffer?
    func set(_ table: ValueBoxTable, key: ValueBoxKey, value: MemoryBuffer)
    func remove(_ table: ValueBoxTable, key: ValueBoxKey, secure: Bool)
    func exists(_ table: ValueBoxTable, key: ValueBoxKey) -> Bool

    func range(_ table: ValueBoxTable, start: ValueBoxKey, end: ValueBoxKey,
               values: (ValueBoxKey, ReadBuffer) -> Bool, limit: Int)
    func filteredRange(_ table: ValueBoxTable, start: ValueBoxKey, end: ValueBoxKey,
                       values: (ValueBoxKey, ReadBuffer) -> ValueBoxFilterResult, limit: Int)
    func scan(_ table: ValueBoxTable, values: (ValueBoxKey, ReadBuffer) -> Bool)

    func count(_ table: ValueBoxTable, start: ValueBoxKey, end: ValueBoxKey) -> Int
    func removeRange(_ table: ValueBoxTable, start: ValueBoxKey, end: ValueBoxKey)

    // Full-text search
    func fullTextSet(_ table: ValueBoxFullTextTable, collectionId: String,
                     itemId: String, contents: String, tags: String)
    func fullTextMatch(_ table: ValueBoxFullTextTable, collectionId: String?,
                       query: String, tags: String?,
                       values: (String, String) -> Bool)
    // ...
}
```

Key design decisions:

**Two key types**: `binary` keys are variable-length byte buffers constructed manually. `int64` keys are 8-byte integers used for simple lookups (like PeerId → Peer). Binary keys enable composite keys: a 20-byte key for messages encodes `PeerId(8) + Namespace(4) + Timestamp(4) + Id(4)`, which means range queries over this key automatically filter by peer, then namespace, then time order.

**Closure-based iteration**: Range queries use callbacks (`(ValueBoxKey, ReadBuffer) -> Bool`) instead of returning arrays. The `Bool` return value controls continuation — return `false` to stop early. This avoids allocating arrays for large scans.

**No ORM overhead**: Values are raw `MemoryBuffer`s (byte buffers). There's no object-relational mapping, no property list serialization. Tables write and read bytes directly using `PostboxEncoder`/`PostboxDecoder`.

The concrete implementation is `SqliteValueBox`, which:
- Creates one SQLite table per `ValueBoxTable` (named `t0`, `t1`, `t2`, etc.)
- Uses WAL (Write-Ahead Logging) mode for concurrent reads
- Supports encryption via SQLCipher parameters
- Caches 15+ prepared statement types for performance
- Handles checkpointing and database integrity

## Layer 2: Table — The Business Logic Layer

Each domain concept gets its own `Table` subclass:

```swift
// Postbox/Sources/Table.swift

open class Table {
    public final let valueBox: ValueBox
    public final let table: ValueBoxTable
    public final let useCaches: Bool

    public init(valueBox: ValueBox, table: ValueBoxTable, useCaches: Bool) {
        self.valueBox = valueBox
        self.table = table
        self.useCaches = useCaches
    }

    open func clearMemoryCache() {
    }

    open func beforeCommit() {
    }
}
```

Just 19 lines. The base class provides:
- Access to the `ValueBox` for storage operations
- A `ValueBoxTable` descriptor (ID + key type)
- A `useCaches` flag for memory caching
- A `beforeCommit()` hook for batching writes
- A `clearMemoryCache()` hook for memory pressure

Here's how `PeerTable` extends it:

```swift
// Postbox/Sources/PeerTable.swift

final class PeerTable: Table {
    static func tableSpec(_ id: Int32) -> ValueBoxTable {
        return ValueBoxTable(id: id, keyType: .int64, compactValuesOnCreation: false)
    }

    private let sharedEncoder = PostboxEncoder()
    private let sharedKey = ValueBoxKey(length: 8)
    private var cachedPeers: [PeerId: Peer] = [:]
    private var updatedInitialPeers: [PeerId: Peer?] = [:]

    private func key(_ id: PeerId) -> ValueBoxKey {
        self.sharedKey.setInt64(0, value: id.toInt64())
        return self.sharedKey
    }

    func set(_ peer: Peer) {
        let previous = self.get(peer.id)
        self.cachedPeers[peer.id] = peer
        if self.updatedInitialPeers[peer.id] == nil {
            self.updatedInitialPeers[peer.id] = previous
        }
    }

    func get(_ id: PeerId) -> Peer? {
        if let peer = self.cachedPeers[id] {
            return peer
        }
        if let value = self.valueBox.get(self.table, key: self.key(id)) {
            if let peer = PostboxDecoder(buffer: value).decodeRootObject() as? Peer {
                self.cachedPeers[id] = peer
                return peer
            }
        }
        return nil
    }

    override func beforeCommit() {
        if !self.updatedInitialPeers.isEmpty {
            for (peerId, previousPeer) in self.updatedInitialPeers {
                if let peer = self.cachedPeers[peerId] {
                    self.sharedEncoder.reset()
                    self.sharedEncoder.encodeRootObject(peer)
                    self.valueBox.set(self.table, key: self.key(peerId),
                                      value: self.sharedEncoder.readBufferNoCopy())
                    // Update dependent tables...
                }
            }
            self.updatedInitialPeers.removeAll()
        }
    }
}
```

This reveals several important patterns:

### Write-Behind Caching

`PeerTable.set()` doesn't write to SQLite immediately. It stores the new value in `cachedPeers` and records the previous value in `updatedInitialPeers`. The actual SQLite write happens in `beforeCommit()`, which runs once at the end of the transaction.

Benefits:
- Multiple `set()` calls to the same peer in one transaction result in a single SQLite write
- The previous value is tracked for change diffing (needed by the view system)
- Read-your-writes: `get()` checks the cache first, so subsequent reads in the same transaction see the latest value

### Shared Encoder and Key Objects

The `sharedEncoder` and `sharedKey` are reused across calls to avoid allocating new objects per operation. In a transaction that updates 1,000 peers, this saves 1,000 `PostboxEncoder` and 1,000 `ValueBoxKey` allocations.

### Dependent Table Updates

`PeerTable.beforeCommit()` also updates related tables (reverse associations, timeout properties). This ensures cross-table consistency within a single transaction.

## Layer 3: PostboxImpl — The Orchestrator

`PostboxImpl` is the core class that ties everything together. It's approximately 5,000 lines and holds references to **all 70+ table instances**:

```swift
// Postbox/Sources/Postbox.swift:1590

final class PostboxImpl {
    let queue: Queue
    let valueBox: SqliteValueBox

    // 70+ table instances
    let metadataTable: MetadataTable               // table ID: 0
    let keychainTable: KeychainTable               // table ID: 1
    let peerTable: PeerTable                       // table ID: 2
    let globalMessageIdsTable: GlobalMessageIdsTable // table ID: 3
    let messageHistoryIndexTable: MessageHistoryIndexTable // table ID: 4
    let mediaTable: MessageMediaTable              // table ID: 6
    let messageHistoryTable: MessageHistoryTable   // table ID: 7
    let messageHistoryMetadataTable: MessageHistoryMetadataTable // table ID: 10
    let messageHistoryUnsentTable: MessageHistoryUnsentTable // table ID: 11
    let messageHistoryTagsTable: MessageHistoryTagsTable // table ID: 12
    let peerChatStateTable: PeerChatStateTable     // table ID: 13
    let readStateTable: MessageHistoryReadStateTable // table ID: 14
    let contactsTable: ContactTable                // table ID: 16
    let chatListIndexTable: ChatListIndexTable     // table ID: ...
    let chatListTable: ChatListTable
    // ... 50+ more tables
    let storyTable: StoryTable                     // table ID: recent additions
    // ...

    let tables: [Table]  // All tables in one array for batch operations
}
```

Each table gets a static integer ID (0, 1, 2, ..., 85). These IDs correspond to SQLite table names (`t0`, `t1`, `t2`). The IDs are stable across app versions — the table-to-ID mapping is part of the database schema contract.

### Table Initialization Order

Tables are initialized in dependency order. Some tables depend on others:

```swift
// PeerTable needs ReverseAssociatedPeerTable and PeerTimeoutPropertiesTable
self.reverseAssociatedPeerTable = ReverseAssociatedPeerTable(
    valueBox: self.valueBox, table: ..., useCaches: useCaches)
self.peerTimeoutPropertiesTable = PeerTimeoutPropertiesTable(
    valueBox: self.valueBox, table: ..., useCaches: useCaches)
self.peerTable = PeerTable(
    valueBox: self.valueBox, table: ..., useCaches: useCaches,
    reverseAssociatedTable: self.reverseAssociatedPeerTable,
    peerTimeoutPropertiesTable: self.peerTimeoutPropertiesTable)

// MessageHistoryTable has the most dependencies — 18 other tables
self.messageHistoryTable = MessageHistoryTable(
    valueBox: self.valueBox, table: ..., useCaches: useCaches,
    seedConfiguration: seedConfiguration,
    messageHistoryIndexTable: self.messageHistoryIndexTable,
    messageHistoryHoleIndexTable: self.messageHistoryHoleIndexTable,
    messageMediaTable: self.mediaTable,
    historyMetadataTable: self.messageHistoryMetadataTable,
    globallyUniqueMessageIdsTable: self.globallyUniqueMessageIdsTable,
    unsentTable: self.messageHistoryUnsentTable,
    failedTable: self.messageHistoryFailedTable,
    tagsTable: self.messageHistoryTagsTable,
    threadsTable: self.messageHistoryThreadsTable,
    // ... 9 more
)
```

`MessageHistoryTable` depends on 18 other tables because adding a message affects the index, tags, threads, unsent queue, read state, text search, and more.

## The Transaction Model

All Postbox operations happen inside transactions. The `Transaction` class wraps table operations with safety checks:

```swift
// Postbox/Sources/Postbox.swift:22

public final class Transaction {
    private let queue: Queue
    private weak var postbox: PostboxImpl?
    var disposed = false

    public func addMessages(_ messages: [StoreMessage], location: AddMessagesLocation)
        -> [Int64: MessageId] {
        assert(!self.disposed)
        if let postbox = self.postbox {
            return postbox.addMessages(transaction: self, messages: messages, location: location)
        }
        return [:]
    }

    public func addHole(peerId: PeerId, threadId: Int64?, namespace: MessageId.Namespace,
                        space: MessageHistoryHoleOperationSpace,
                        range: ClosedRange<MessageId.Id>) {
        assert(!self.disposed)
        self.postbox?.addHole(peerId: peerId, threadId: threadId, ...)
    }

    public func getHoles(peerId: PeerId, namespace: MessageId.Namespace) -> IndexSet {
        assert(!self.disposed)
        return self.postbox?.messageHistoryHoleIndexTable.closest(...) ?? IndexSet()
    }
    // ... 100+ more methods
}
```

Every method asserts `!self.disposed` — if you try to use a transaction after it's committed, you get a crash in debug. The `postbox` reference is weak, so the transaction can't keep PostboxImpl alive after it should be deallocated.

### Transaction Lifecycle

```
1. Postbox.transaction { tx in ... }
   ↓
2. queue.async {
      valueBox.begin()           // SQLite: BEGIN TRANSACTION
   ↓
3.    let tx = Transaction(queue, postbox)
      f(tx)                      // User code runs
   ↓
4.    beforeCommit()             // Tables flush caches to SQLite
   ↓
5.    Build PostboxTransaction   // Capture all changes
   ↓
6.    viewTracker.updateViews()  // Replay changes to active views
   ↓
7.    valueBox.commit()          // SQLite: COMMIT
   ↓
8.    tx.disposed = true         // Prevent further use
   }
```

### PostboxTransaction: The Change Record

At step 5, all changes from the transaction are captured in an immutable `PostboxTransaction` object:

```swift
// Postbox/Sources/PostboxTransaction.swift

final class PostboxTransaction {
    let currentOperationsByPeerId: [PeerId: [MessageHistoryOperation]]
    let chatListOperations: [PeerGroupId: [ChatListOperation]]
    let currentUpdatedPeers: [PeerId: Peer]
    let currentUpdatedPeerNotificationSettings: [PeerId: (PeerNotificationSettings?, PeerNotificationSettings)]
    let currentUpdatedCachedPeerData: [PeerId: (previous: CachedPeerData?, updated: CachedPeerData)]
    let currentUpdatedPeerPresences: [PeerId: PeerPresence]
    let currentPeerHoleOperations: [MessageHistoryIndexHoleOperationKey: [MessageHistoryIndexHoleOperation]]
    let currentPreferencesOperations: [PreferencesOperation]
    let updatedMedia: [MediaId: Media?]
    let replaceContactPeerIds: Set<PeerId>?
    let storyEvents: [StoryTable.Event]
    // ... 50+ more fields
}
```

This change record serves a critical purpose: it tells the view system **what changed** without requiring views to re-scan the database. Each view's `replay()` method only checks the fields relevant to its data. A `PeerView` only looks at `currentUpdatedPeers`, `currentUpdatedPeerPresences`, and `currentUpdatedCachedPeerData`. A `ChatListView` only looks at `chatListOperations` and `currentUpdatedChatListInclusions`.

The `isEmpty` property checks all 50+ fields and returns `true` only if nothing changed — an optimization to skip view updates entirely when a transaction is read-only.

## Integration with SwiftSignalKit

Postbox exposes two types of signals:

### One-Shot Transactions

```swift
// Returns Signal<T, NoError> that emits once and completes
public func transaction<T>(_ f: @escaping (Transaction) -> T) -> Signal<T, NoError> {
    return Signal { subscriber in
        self.queue.async {
            // ... begin transaction
            let result = f(transaction)
            // ... commit
            subscriber.putNext(result)
            subscriber.putCompletion()
        }
        return EmptyDisposable
    }
}
```

The closure runs on the Postbox queue, executes the transaction, and emits the result as a single signal value. This is how TelegramCore reads and writes data.

### Live Views

```swift
// Returns Signal<PeerView, NoError> that emits on every relevant change
public func peerView(id: PeerId) -> Signal<PeerView, NoError> {
    return Signal { subscriber in
        self.queue.async {
            let mutableView = MutablePeerView(postbox: self, peerId: id)
            let (index, signal) = self.viewTracker.addPeerView(mutableView)

            // Initial snapshot
            subscriber.putNext(PeerView(mutableView))

            // Future updates
            let disposable = signal.start(next: { view in
                subscriber.putNext(view)
            })

            return ActionDisposable {
                disposable.dispose()
                self.queue.async {
                    self.viewTracker.removePeerView(index)
                }
            }
        }
    }
}
```

Live views never complete — they emit an initial snapshot immediately, then emit a new snapshot every time a relevant transaction commits. Subscribers must dispose to stop receiving updates. This is the foundation of Telegram's reactive UI: subscribe to a view, and your UI automatically updates when the data changes.

## Identity Types

Postbox defines precise identity types for every entity:

```swift
// PeerId: 8 bytes encoding namespace + ID
public struct PeerId: Hashable, Comparable {
    let namespace: Namespace     // User, Group, Channel, SecretChat
    let id: Id
    func toInt64() -> Int64      // Packed representation for storage
}

// MessageId: 16 bytes encoding peer + namespace + ID
public struct MessageId: Hashable, Comparable {
    let peerId: PeerId           // Which chat
    let namespace: Int32         // Cloud vs Local vs SecretIncoming, etc.
    let id: Int32                // Sequential within namespace
}

// MessageIndex: MessageId + timestamp for ordering
public struct MessageIndex: Comparable {
    let id: MessageId
    let timestamp: Int32         // Unix timestamp
}

// MediaId: 12 bytes encoding namespace + ID
public struct MediaId: Hashable {
    let namespace: Int32
    let id: Int64
}
```

These types enforce correctness at compile time. You can't accidentally pass a `PeerId` where a `MediaId` is expected. The `Comparable` conformance on `MessageIndex` defines the sort order used by every message query.

## Custom Binary Serialization

Postbox uses `PostboxEncoder` and `PostboxDecoder` instead of `Codable`:

```swift
let encoder = PostboxEncoder()
encoder.encodeRootObject(peer)                    // Writes type hash + fields
let data: MemoryBuffer = encoder.readBufferNoCopy()

let decoder = PostboxDecoder(buffer: data)
let peer = decoder.decodeRootObject() as? Peer    // Reads type hash, dispatches
```

The encoding format:
1. **Type hash** (4 bytes): A MurmurHash32 of the class name, used for polymorphic dispatch during decoding
2. **Field count** (variable): Number of encoded fields
3. **Fields**: Each field is a key string + type tag + value bytes

The type hash system means Postbox can decode any `Peer` subclass (User, Group, Channel, SecretChat) without knowing the type at compile time. The decoder looks up the hash in a registry, instantiates the correct class, and calls its `init(decoder:)` initializer.

Why not `Codable`?
- `Codable` didn't exist when Postbox was built
- `PostboxEncoder` writes to `MemoryBuffer` (zero-copy shared memory), not `Data`
- The type hash enables polymorphic collections (an array of different `Peer` types)
- Field access is by name, enabling forward/backward compatibility (new fields are ignored by old code)

## The SeedConfiguration

When Postbox is initialized, it receives a `SeedConfiguration` that defines app-specific constants:

```swift
public struct SeedConfiguration {
    public let globalMessageIdsPeerIdNamespaces: Set<GlobalMessageIdsNamespace>
    public let initializeChatListWithHole: (groupId: PeerGroupId, id: MessageId.Id, ...)
    public let messageHoles: [PeerId.Namespace: [MessageId.Namespace: ...]]
    public let existingMessageTags: MessageTags
    public let messageTagsWithSummary: MessageTags
    public let existingGlobalMessageTags: GlobalMessageTags
    // ...
}
```

This tells Postbox:
- Which message namespaces exist (cloud messages, local messages, secret messages)
- How to initialize the chat list (with a hole covering the entire range, to be filled from the server)
- Which message tags are valid (photos, files, links, etc.)
- Which tags should maintain summary counts

This separation means Postbox itself doesn't know anything about Telegram's specific domain — it's a generic framework configured at initialization time.

## Performance Characteristics

Based on the architecture:

- **All writes are serialized** on a single queue. This eliminates write contention but means write throughput depends on transaction size.
- **Reads during a transaction** hit the memory cache first, then SQLite. Frequently accessed peers stay in `cachedPeers`.
- **Views are updated synchronously** after each write transaction. This means the UI always sees consistent data, but large transactions can block the Postbox queue.
- **SQLite WAL mode** allows concurrent reads from other threads, though Postbox routes all access through its queue.
- **Batch commits**: `beforeCommit()` flushes all cached writes in one pass, turning many small writes into a single SQLite transaction.

## Architectural Takeaways

1. **The ValueBox abstraction pays for itself.** By isolating SQLite behind a protocol, Postbox can change storage implementations, add encryption transparently, and test tables without a database.

2. **Write-behind caching is essential for batch performance.** Tables accumulate changes in memory and flush once per transaction. Without this, adding 100 messages would mean 100 separate SQLite INSERTs instead of one batch.

3. **The change record (PostboxTransaction) enables efficient reactivity.** Instead of views polling the database or diffing before/after snapshots, the transaction tells each view exactly what changed.

4. **70+ tables isn't excessive when each is focused.** Each table handles one concern: peers, messages, tags, read states, etc. The alternative — fewer tables with more complex queries — would be harder to optimize and maintain.

5. **Custom serialization was worth the investment.** The type-hash-based encoding handles polymorphism, forward compatibility, and zero-copy memory sharing — features that `Codable` still doesn't provide in combination.

In the [next post](/posts/07-postbox-tables/), we'll examine the most important tables in detail — how messages, chats, and peers are stored, keyed, and queried.

## Key Files Reference

| File | Path | Lines | Role |
|---|---|---|---|
| ValueBox | `Postbox/Sources/ValueBox.swift` | 102 | Storage protocol |
| SqliteValueBox | `Postbox/Sources/SqliteValueBox.swift` | ~1,000 | SQLite implementation |
| Table | `Postbox/Sources/Table.swift` | 19 | Base table class |
| Postbox | `Postbox/Sources/Postbox.swift` | ~5,200 | Orchestrator (PostboxImpl) |
| PostboxTransaction | `Postbox/Sources/PostboxTransaction.swift` | 347 | Change record |
| PostboxView | `Postbox/Sources/PostboxView.swift` | 112 | View protocol + CombinedView |
| ViewTracker | `Postbox/Sources/ViewTracker.swift` | 667 | View subscription manager |
| PeerTable | `Postbox/Sources/PeerTable.swift` | 115 | Example table implementation |
