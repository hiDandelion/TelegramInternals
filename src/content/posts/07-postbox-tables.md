---
title: "Postbox Tables: How Telegram Stores Messages, Peers, and Chats"
description: "A deep dive into the 70+ table classes that make up Postbox — from the 3,500-line MessageHistoryTable to the 115-line PeerTable, with key encoding strategies, hole management, and cross-table dependencies."
published: 2026-02-11
tags:
  - Persistence
toc: true
lang: en
abbrlink: 07-postbox-tables
pin: 19
---

In the [previous post](/posts/06-postbox-overview/), we saw Postbox's three-layer architecture: ValueBox → Tables → Views. This post dives into the table layer — the 70+ specialized classes that encode Telegram's data model into key-value storage.

## Table Categories

Postbox's tables fall into clear functional groups:

| Category | Tables | Purpose |
|---|---|---|
| **Messages** | 20+ | Message storage, indexing, tags, threads, holes, full-text search |
| **Peers** | 10+ | User/group/channel storage, presence, cached data, relationships |
| **Chat List** | 4 | Chat ordering, pinning, folder assignment |
| **Read State** | 3 | Read/unread tracking and sync |
| **Collections** | 5 | Sticker packs, saved GIFs, ordered/unordered lists |
| **Operations** | 4 | Pending server operations queue |
| **Stories** | 6 | Story posts, states, subscriptions |
| **Preferences** | 4 | Settings, notices, keychain, metadata |

Let's examine the most important ones.

## MessageHistoryTable: The Core (3,500 Lines)

This is the most complex table, responsible for storing every message in every chat. It depends on 18 other tables:

```swift
// Postbox/Sources/MessageHistoryTable.swift:68

final class MessageHistoryTable: Table {
    static func tableSpec(_ id: Int32) -> ValueBoxTable {
        return ValueBoxTable(id: id, keyType: .binary, compactValuesOnCreation: false)
    }

    let messageHistoryIndexTable: MessageHistoryIndexTable
    let messageHistoryHoleIndexTable: MessageHistoryHoleIndexTable
    let messageMediaTable: MessageMediaTable
    let historyMetadataTable: MessageHistoryMetadataTable
    let globallyUniqueMessageIdsTable: MessageGloballyUniqueIdTable
    let unsentTable: MessageHistoryUnsentTable
    let failedTable: MessageHistoryFailedTable
    let tagsTable: MessageHistoryTagsTable
    let threadsTable: MessageHistoryThreadsTable
    let threadTagsTable: MessageHistoryThreadTagsTable
    let customTagTable: MessageCustomTagTable
    let customTagWithTagTable: MessageCustomTagWithTagTable
    let globalTagsTable: GlobalMessageHistoryTagsTable
    let localTagsTable: LocalMessageHistoryTagsTable
    let timeBasedAttributesTable: TimestampBasedMessageAttributesTable
    let readStateTable: MessageHistoryReadStateTable
    let synchronizeReadStateTable: MessageHistorySynchronizeReadStateTable
    let textIndexTable: MessageHistoryTextIndexTable
    let summaryTable: MessageHistoryTagsSummaryTable
    let pendingActionsTable: PendingMessageActionsTable
}
```

### Key Design

The message key is 20 bytes:

```
┌──────────┬───────────┬───────────┬──────────┐
│ PeerId   │ Namespace │ Timestamp │ Id       │
│ 8 bytes  │ 4 bytes   │ 4 bytes   │ 4 bytes  │
└──────────┴───────────┴───────────┴──────────┘
```

```swift
private func extractKey(_ key: ValueBoxKey) -> MessageIndex {
    return MessageIndex(
        id: MessageId(
            peerId: PeerId(key.getInt64(0)),
            namespace: key.getInt32(8),
            id: key.getInt32(8 + 4 + 4)
        ),
        timestamp: key.getInt32(8 + 4)
    )
}
```

This key layout has crucial implications:

1. **Range queries by peer** are efficient — all messages for a peer share the same 8-byte prefix, so `range(start: [peerId, 0, 0, 0], end: [peerId+1, 0, 0, 0])` scans exactly one peer's messages.

2. **Within a peer, messages are ordered by timestamp** — SQLite's B-tree index keeps them sorted by the third field, giving chronological ordering naturally.

3. **Namespace isolation** — Cloud messages (namespace 0), local messages (namespace 5), and secret messages have different namespaces, so they don't interleave in the index.

### Value Encoding

Each message value contains:

```swift
private struct MessageDataFlags: OptionSet {
    static let hasGloballyUniqueId = MessageDataFlags(rawValue: 1 << 0)
    static let hasGlobalTags       = MessageDataFlags(rawValue: 1 << 1)
    static let hasGroupingKey      = MessageDataFlags(rawValue: 1 << 2)
    static let hasGroupInfo        = MessageDataFlags(rawValue: 1 << 3)
    static let hasLocalTags        = MessageDataFlags(rawValue: 1 << 4)
    static let hasThreadId         = MessageDataFlags(rawValue: 1 << 5)
}
```

The flags byte at the start tells the decoder which optional fields are present. This is a compact variable-length encoding — messages without threads or groups skip those fields entirely. A simple text message might be 50 bytes; a forwarded message in a thread with global tags might be 200 bytes.

### What Happens When a Message Is Added

Adding a message touches many tables:

1. **MessageHistoryTable**: Store the message data
2. **MessageHistoryIndexTable**: Add to the ID→timestamp index
3. **MessageHistoryTagsTable**: Add entries for each tag (photo, file, link, etc.)
4. **MessageHistoryThreadsTable**: If threaded, add to thread index
5. **MessageHistoryUnsentTable**: If outgoing and not yet sent, add to unsent queue
6. **GlobalMessageHistoryTagsTable**: If globally tagged, add to global index
7. **MessageMediaTable**: Store media objects (photos, documents)
8. **MessageHistoryTextIndexTable**: Index message text for full-text search
9. **MessageHistoryTagsSummaryTable**: Update tag counts
10. **ReadStateTable**: Potentially update unread count

This is why `MessageHistoryTable` depends on 18 tables — every message operation cascades.

## MessageHistoryIndexTable: Fast ID Lookups

The main `MessageHistoryTable` key includes the timestamp, which means looking up a message by ID alone requires knowing its timestamp. The `MessageHistoryIndexTable` solves this:

```
Key: PeerId(8) + Namespace(4) + Id(4) = 16 bytes
Value: Timestamp(4) + flags
```

Given a `MessageId`, you can look up the timestamp in the index table, construct the full 20-byte key, and fetch the message from the history table. The index also stores flags like incoming/outgoing direction.

Key operations:
- `exists(id)` — Check if a message ID is used
- `top(peerId, namespace)` — Get the newest message index for a peer
- `fillHole(peerId, namespace, range, messages)` — Bulk-replace a hole with actual messages

## MessageHistoryHoleIndexTable: Tracking Missing Data

This table is unique to Telegram and has no equivalent in standard databases. It tracks ranges of message IDs that haven't been fetched from the server yet:

```swift
// MessageHistoryHoleIndexTable key:
// PeerId(8) + Namespace(4) + Tag(4) + MinId(4) + MaxId(4) = 24 bytes

enum MessageHistoryIndexHoleOperation {
    case insert(ClosedRange<MessageId.Id>)
    case remove(ClosedRange<MessageId.Id>)
}

public enum MessageHistoryHoleSpace: Equatable, Hashable {
    case everywhere           // Applies to all messages
    case tag(MessageTags)     // Applies to tagged subset
}
```

When you open Telegram for the first time, the entire message history for each chat is a single hole: `1...Int32.max`. As you scroll, the server returns pages of messages, and the hole is split:

```
Before:  [1 ──────────────────────── MAX]
         (one big hole)

Fetch messages 500-600:
After:   [1 ────── 499] [601 ──── MAX]
         (two smaller holes)

Fetch messages 200-400:
After:   [1 ── 199] [401 ── 499] [601 ── MAX]
         (three holes)
```

Holes can be per-space: the "everywhere" space tracks all messages, while tag-specific spaces (like `.tag(.photo)`) track which photos have been fetched. This allows the "shared media" gallery to fetch only photos without affecting the main message history holes.

Key operations:
- `addHole(peerId, namespace, space, range)` — Mark a range as missing
- `remove(peerId, namespace, space, range)` — Fill part of a hole (server returned data)
- `closest(peerId, namespace, space, range)` — Find the nearest hole to a given position (used to determine what to fetch next)

## PeerTable: Simple Int64-Keyed Storage

Compared to the message tables, `PeerTable` is refreshingly simple:

```swift
// Postbox/Sources/PeerTable.swift

final class PeerTable: Table {
    static func tableSpec(_ id: Int32) -> ValueBoxTable {
        return ValueBoxTable(id: id, keyType: .int64, compactValuesOnCreation: false)
    }

    private var cachedPeers: [PeerId: Peer] = [:]
    private var updatedInitialPeers: [PeerId: Peer?] = [:]

    func get(_ id: PeerId) -> Peer? {
        if let peer = self.cachedPeers[id] {
            return peer                     // Cache hit
        }
        if let value = self.valueBox.get(self.table, key: self.key(id)) {
            if let peer = PostboxDecoder(buffer: value).decodeRootObject() as? Peer {
                self.cachedPeers[id] = peer
                return peer                 // Decode and cache
            }
        }
        return nil                          // Not found
    }

    func set(_ peer: Peer) {
        let previous = self.get(peer.id)    // Read before write
        self.cachedPeers[peer.id] = peer
        if self.updatedInitialPeers[peer.id] == nil {
            self.updatedInitialPeers[peer.id] = previous  // Track changes
        }
    }
}
```

The `int64` key type means the key is just `PeerId.toInt64()` — an 8-byte integer. No composite key construction needed.

The `updatedInitialPeers` dictionary captures the "before" state for each modified peer. This serves two purposes:
1. The view system needs to know what changed (before vs after)
2. Dependent tables (reverse associations, timeout properties) need the previous state to undo old entries

### Dependent Table Updates

When `beforeCommit()` runs, `PeerTable` updates two other tables:

```swift
override func beforeCommit() {
    for (peerId, previousPeer) in self.updatedInitialPeers {
        if let peer = self.cachedPeers[peerId] {
            // Write to SQLite
            self.sharedEncoder.reset()
            self.sharedEncoder.encodeRootObject(peer)
            self.valueBox.set(self.table, key: self.key(peerId),
                              value: self.sharedEncoder.readBufferNoCopy())

            // Update reverse associations
            let previousAssociation = previousPeer?.associatedPeerId
            if previousAssociation != peer.associatedPeerId {
                if let prev = previousAssociation {
                    self.reverseAssociatedTable.removeReverseAssociation(target: prev, from: peerId)
                }
                if let assoc = peer.associatedPeerId {
                    self.reverseAssociatedTable.addReverseAssociation(target: assoc, from: peerId)
                }
            }
        }
    }
    self.updatedInitialPeers.removeAll()
}
```

The `ReverseAssociatedPeerTable` maintains a bidirectional mapping. When a secret chat peer has `associatedPeerId` pointing to the user, the reverse table maps user → secret chat. This enables finding all secret chats for a given user.

## ChatListIndexTable: Chat List Ordering

The chat list is one of the most complex parts because it must handle:
- Pinned chats (user-defined order at the top)
- Regular chats (sorted by latest message timestamp)
- Archived chats (separate group)
- Folders (multiple groups)
- Minimum timestamp (chats that should appear even without messages)

```swift
// Postbox/Sources/ChatListIndexTable.swift

struct ChatListPeerInclusionIndex {
    let topMessageIndex: MessageIndex?
    let inclusion: PeerChatListInclusion
}

public enum PeerChatListInclusion {
    case notIncluded
    case ifHasMessagesOrOneOf(groupId: PeerGroupId, pinningIndex: UInt16?, minTimestamp: Int32?)
}
```

A peer's chat list position is determined by two factors:
1. **Inclusion**: Should this peer appear in the chat list at all? If so, which group (main, archive)?
2. **Top message index**: The timestamp of the latest message determines sort position.

The `ChatListIndex` combines these:

```swift
func includedIndex(peerId: PeerId) -> (PeerGroupId, ChatListIndex)? {
    switch inclusion {
    case .notIncluded:
        return nil
    case let .ifHasMessagesOrOneOf(groupId, pinningIndex, minTimestamp):
        if let topMessageIndex = self.topMessageIndex {
            return (groupId, ChatListIndex(pinningIndex: pinningIndex, messageIndex: topMessageIndex))
        } else if let pinningIndex = pinningIndex {
            // Pinned but no messages — use zero timestamp
            return (groupId, ChatListIndex(pinningIndex: pinningIndex,
                messageIndex: MessageIndex(id: MessageId(peerId: peerId, namespace: 0, id: 0),
                                          timestamp: 0)))
        } else {
            return nil   // No messages and not pinned — not in list
        }
    }
}
```

Pinned chats get a `pinningIndex` — the lower the index, the higher in the pinned section. The actual `ChatListTable` uses a composite key where pinned items sort before unpinned ones.

## ChatListTable: The Sorted Chat List

The `ChatListTable` key encodes the full sort order:

```
GroupId(4) + PinningKey(2) + Timestamp(4) + Namespace(1) + Id(4) + PeerId(8) + Type(1) = ~24 bytes
```

The `PinningKey` uses inverted indices — `UInt16.max - 1 - pinningIndex` — so pinned items with lower logical indices get higher byte values and sort first in SQLite's ascending B-tree. This trick avoids a separate table or secondary sort.

The table stores two entry types:

```swift
enum ChatListEntryType {
    case message    // Real chat with data
    case hole       // Gap needing server fetch
}
```

Holes in the chat list work like message history holes — they represent portions of the chat list that haven't been loaded from the server yet.

## MessageHistoryReadStateTable: Tracking Read/Unread

```swift
// Postbox/Sources/MessageHistoryReadStateTable.swift

public enum PeerReadState: Equatable {
    case idBased(maxIncomingReadId: MessageId.Id,
                 maxOutgoingReadId: MessageId.Id,
                 maxKnownId: MessageId.Id,
                 count: Int32,
                 markedUnread: Bool)
    case indexBased(maxIncomingReadIndex: MessageIndex,
                    maxOutgoingReadIndex: MessageIndex,
                    count: Int32,
                    markedUnread: Bool)
}
```

Two variants exist because different peer types use different read state models:
- **idBased**: Regular chats and groups. Read state is tracked by message ID. `count` is the number of unread messages.
- **indexBased**: Some special cases where read state is tracked by message index (timestamp-based).

The `maxIncomingReadId` is the latest message ID we've read from others. The `maxOutgoingReadId` is the latest message ID that others have read from us. The `maxKnownId` is the highest message ID we know about (used to calculate the unread count accurately).

`markedUnread` handles the Telegram feature where users can mark a chat as unread even after reading all messages — the blue dot appears but the count is 0.

### Manual Binary Serialization

Unlike `PeerTable` which uses `PostboxEncoder`, the read state table hand-writes binary data for maximum compactness:

```swift
private func get(_ id: PeerId) -> InternalPeerReadStates? {
    if let value = self.valueBox.get(self.table, key: self.key(id)) {
        var count: Int32 = 0
        value.read(&count, offset: 0, length: 4)    // Read namespace count

        for _ in 0 ..< count {
            var namespaceId: Int32 = 0
            value.read(&namespaceId, offset: 0, length: 4)

            var kind: Int8 = 0
            value.read(&kind, offset: 0, length: 1)

            if kind == 0 {  // ID-based
                var maxIncomingReadId: Int32 = 0
                value.read(&maxIncomingReadId, offset: 0, length: 4)
                // ... read remaining fields
                state = .idBased(maxIncomingReadId: maxIncomingReadId, ...)
            } else {  // Index-based
                // ... read index fields
                state = .indexBased(...)
            }
        }
    }
}
```

The `read(&var, offset: 0, length: N)` calls advance a cursor through the raw bytes. No JSON parsing, no key-value lookups, no property names — just sequential binary reads. For a table that's queried on every chat view update, this compactness matters.

## MessageHistoryTagsTable: Filtered Message Lists

When you tap "Shared Media" in a chat, Telegram shows photos, files, links, and audio separately. Each of these is a **tag**:

```swift
public struct MessageTags: OptionSet {
    public let rawValue: UInt32
    // Standard tags (configured in SeedConfiguration):
    // .photo, .file, .video, .music, .voiceOrInstantVideo, .webPage, etc.
}
```

The tags table key is:

```
PeerId(8) + Tag(4) + Namespace(4) + Timestamp(4) + Id(4) = 24 bytes
```

By putting `Tag` before `Timestamp`, range queries efficiently fetch "all photos for peer X in chronological order" without scanning non-photo messages.

When a message is added, `MessageHistoryTable` checks its media types and inserts corresponding tag entries. A message with both a photo and a link gets two tag entries.

The `MessageHistoryTagsSummaryTable` maintains running counts per tag per peer, so the UI can show "247 photos" without counting them.

## ItemCollectionInfoTable + ItemCollectionItemTable: Sticker Packs

Sticker packs, saved GIFs, and similar collections use a two-table design:

```
InfoTable key:  Namespace(4) + OrderIndex(4) + CollectionId(8) = 16 bytes
ItemTable key:  Namespace(4) + CollectionId(8) + ItemIndex(4) + ItemId(8) = 24 bytes
```

The `InfoTable` stores metadata about each collection (name, thumbnail, count), ordered by `OrderIndex`. The `ItemTable` stores individual items within each collection.

This separation allows:
- Listing all sticker packs (query InfoTable only — don't load every sticker)
- Loading a specific pack's stickers (query ItemTable with collection prefix)
- Reordering packs (update InfoTable order without touching items)
- Searching by keyword (items have `indexKeys` for search)

## PeerOperationLogTable: Reliable Operation Queue

Some operations must survive app restarts — sending a message, clearing history, updating notification settings. The `PeerOperationLogTable` is a persistent queue:

```
Key: Tag(1) + PeerId(8) + LocalIndex(4) = 13 bytes
```

Each operation has:
- **Tag**: Operation type (1 byte — send message, clear history, etc.)
- **PeerId**: Which peer the operation targets
- **LocalIndex**: Monotonically increasing per peer+tag, ensuring ordering
- **Contents**: Opaque encoded data for the operation

The sync engine processes operations in order, removing them after successful server confirmation. If the app crashes mid-operation, the entry survives in SQLite and is retried on next launch.

## Cross-Table Consistency Patterns

### The beforeCommit() Cascade

When a transaction commits, tables flush in a specific order:

1. **MessageHistoryTable.beforeCommit()** — Flushes messages, updates indexes, tags, threads, read states, text search, summaries, unsent queue, failed queue, global/local tags
2. **PeerTable.beforeCommit()** — Flushes peers, updates reverse associations and timeout properties
3. **ChatListIndexTable.beforeCommit()** — Recalculates chat list positions for modified peers
4. **Other tables** — Each flushes its cached writes

The order matters because later tables depend on earlier results. For example, `ChatListIndexTable` needs the updated top message index from `MessageHistoryTable`.

### The updatedInitialPeers Pattern

Every table that supports write-behind caching follows the same pattern:

```swift
private var cachedValues: [Key: Value] = [:]
private var updatedInitialValues: [Key: Value?] = [:]

func set(_ value: Value) {
    let previous = self.get(value.key)
    self.cachedValues[value.key] = value
    if self.updatedInitialValues[value.key] == nil {
        self.updatedInitialValues[value.key] = previous
    }
}
```

The `updatedInitialValues` dictionary is only written once per key per transaction. If you set the same peer three times in one transaction, `updatedInitialValues` still holds the original value from before the transaction. This gives the change tracking system accurate before/after diffs.

### Full-Text Search Integration

`MessageHistoryTextIndexTable` uses SQLite's FTS (Full-Text Search) extension:

```swift
func add(messageIndex: MessageIndex, text: String) {
    self.valueBox.fullTextSet(
        self.table,
        collectionId: "\(messageIndex.id.peerId.toInt64())",
        itemId: "\(messageIndex.id.namespace):\(messageIndex.id.id)",
        contents: text,
        tags: ""
    )
}

func search(query: String) -> [MessageIndex] {
    var result: [MessageIndex] = []
    self.valueBox.fullTextMatch(self.table, collectionId: nil, query: query, tags: nil,
        values: { collectionId, itemId in
            // Parse collectionId back to PeerId, itemId to MessageId
            result.append(messageIndex)
            return true
        })
    return result
}
```

The FTS table is separate from the main message table — it stores only the text content with message IDs as keys. This keeps the FTS index compact while allowing join-free lookups: search returns `MessageIndex` values that can directly fetch full messages from `MessageHistoryTable`.

## The Complete Table Map

Here's every table with its ID and key type, showing the full scope:

| ID | Table | Key Type | Purpose |
|---|---|---|---|
| 0 | MetadataTable | binary | Global state, versions |
| 1 | KeychainTable | binary | Encrypted credentials |
| 2 | PeerTable | int64 | Peer (user/chat) data |
| 3 | GlobalMessageIdsTable | binary | Cross-namespace message ID mapping |
| 4 | MessageHistoryIndexTable | binary | Message ID → timestamp index |
| 6 | MessageMediaTable | binary | Media objects |
| 7 | MessageHistoryTable | binary | Message data |
| 10 | MessageHistoryMetadataTable | binary | Chat init state, counters |
| 11 | MessageHistoryUnsentTable | binary | Pending send queue |
| 12 | MessageHistoryTagsTable | binary | Tagged message index |
| 13 | PeerChatStateTable | int64 | Per-peer sync state |
| 14 | MessageHistoryReadStateTable | int64 | Read/unread tracking |
| 15 | SynchronizeReadStateTable | binary | Read state sync queue |
| 16 | ContactTable | binary | Contact peer IDs |
| 17 | RatingTable | binary | Peer ratings |
| 18 | CachedPeerDataTable | int64 | Extended peer metadata |
| 26-27 | PeerNameIndex/TokenIndex | binary | Name search |
| 32 | GloballyUniqueMessageIdTable | binary | Dedup IDs |
| 33-34 | TimestampBasedAttributes | binary | Auto-delete scheduling |
| 39 | GlobalMessageHistoryTagsTable | binary | Global tag index |
| 40 | ReverseAssociatedPeerTable | binary | Bidirectional peer map |
| 41 | TextIndexTable | FTS | Message full-text search |
| 44 | TagsSummaryTable | binary | Tag count summaries |
| 45-46 | PendingMessageActions | binary | Scheduled actions |
| 47 | InvalidatedTagsSummary | binary | Summary invalidation |
| 48 | PendingNotificationSettings | binary | Notification sync |
| 49 | MessageHistoryFailedTable | binary | Failed send queue |
| 52 | LocalMessageHistoryTagsTable | binary | Client-side tags |
| 55 | AdditionalChatListItems | binary | Extra chat list entries |
| 56 | MessageHistoryHoleIndexTable | binary | Message history holes |
| 59 | InvalidatedGroupMessageStats | binary | Stats invalidation |
| 60-61 | NotificationBehavior | binary | Notification categorization |
| 62 | MessageHistoryThreadsTable | binary | Forum thread index |
| 63 | ThreadHoleIndexTable | binary | Thread-specific holes |
| 64 | PeerTimeoutProperties | binary | Auto-delete timers |
| 71-77 | Thread-related tables | binary | Thread tags, state, pins |
| 81-85 | CustomTag tables | binary | Application-defined tags |
| Recent | Story tables (6) | binary | Stories feature |

The IDs are not sequential because tables were added over years of development. Gaps in the numbering represent tables that were removed or never assigned.

## Architectural Takeaways

1. **Composite binary keys are Postbox's superpower.** By carefully constructing multi-field keys, every table gets efficient range queries without secondary indexes. The `PeerId + Timestamp` prefix on message keys means "all messages in this chat, sorted by time" is a single sequential scan.

2. **The hole model is essential for a messaging app.** Without holes, you'd either fetch entire chat histories (impossible at scale) or lose track of what's been fetched. Holes make incremental loading a first-class concept.

3. **Tag tables are inverted indexes.** Instead of querying "all messages where media contains photo," Postbox maintains a pre-built index of message IDs per tag. This trades write-time cost (updating N tag tables per message insert) for read-time speed (tag queries are instant).

4. **The operation log ensures reliability.** Messages are queued as operations before being sent. If the app crashes, the queue persists. This is the same pattern used by robust message queuing systems.

5. **Write-behind caching unifies reads and writes.** During a transaction, you read and write to the same in-memory cache. Only at commit time does the data touch SQLite. This makes transactions fast and consistent.

In the [next post](/posts/08-postbox-views/), we'll see how the view system turns these tables into live, reactive queries that automatically update the UI.
