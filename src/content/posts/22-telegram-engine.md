---
title: "TelegramEngine: The API Facade"
description: "How TelegramEngine wraps the complexity of TelegramCore into a clean, modular API surface. Covers the lazy subsystem pattern with 19 domain modules, the _internal_ delegation convention, Engine* wrapper types (EnginePeer, EngineMessage) that simplify raw Postbox models, the EngineData reactive query system with variadic generic subscriptions, PostboxViewDataItem protocol for type-safe database access, EngineDataMap and EngineDataList collection patterns, and how 200+ public methods organize a quarter-million lines of networking and persistence code."
published: 2025-07-31
tags:
  - Feature Deep Dives
toc: true
lang: en
abbrlink: 22-telegram-engine
pin: 4
---

`TelegramCore` is a massive module — hundreds of files dealing with the MTProto network layer, Postbox database, state synchronization, and dozens of API endpoints. If the UI layer had to interact with all of this directly, every view controller would need to understand signal composition, Postbox transactions, API serialization, and internal model types. The result would be brittle, tightly-coupled code that breaks whenever the internal implementation changes.

`TelegramEngine` solves this by providing a **facade** — a clean, domain-organized API surface that hides the complexity of TelegramCore behind lazy-loaded subsystem modules, type-safe wrapper types, and reactive data subscriptions.

## The Main Class

`TelegramEngine` itself is remarkably simple — 123 lines:

```swift
public final class TelegramEngine {
    public let account: Account

    public init(account: Account) {
        self.account = account
    }

    public lazy var peers: Peers = Peers(account: self.account)
    public lazy var messages: Messages = Messages(account: self.account)
    public lazy var data: EngineData =
        EngineData(accountPeerId: self.account.peerId, postbox: self.account.postbox)
    public lazy var privacy: Privacy = Privacy(account: self.account)
    public lazy var auth: Auth = Auth(account: self.account)
    public lazy var accountData: AccountData = AccountData(account: self.account)
    public lazy var stickers: Stickers = Stickers(account: self.account)
    public lazy var calls: Calls = Calls(account: self.account)
    public lazy var contacts: Contacts = Contacts(account: self.account)
    public lazy var payments: Payments = Payments(account: self.account)
    public lazy var localization: Localization = Localization(account: self.account)
    public lazy var themes: Themes = Themes(account: self.account)
    public lazy var resolve: Resolve = Resolve(account: self.account)
    public lazy var itemCache: ItemCache = ItemCache(account: self.account)
    public lazy var orderedLists: OrderedLists = OrderedLists(account: self.account)
    public lazy var notices: Notices = Notices(account: self.account)
    public lazy var preferences: Preferences = Preferences(account: self.account)
    public lazy var resources: Resources = Resources(account: self.account)
    public lazy var historyImport: HistoryImport = HistoryImport(account: self.account)
}
```

Each property is `lazy` — the subsystem class is only instantiated when first accessed. If a screen never touches stickers, the `Stickers` subsystem is never created. This keeps memory usage proportional to what's actually used.

## The Subsystem Pattern

Each subsystem follows the same structure. Here's a simplified view of `TelegramEngine.Peers`:

```swift
public extension TelegramEngine {
    final class Peers {
        private let account: Account

        init(account: Account) {
            self.account = account
        }

        public func addressNameAvailability(domain: AddressNameDomain, name: String)
            -> Signal<AddressNameAvailability, NoError> {
            return _internal_addressNameAvailability(
                account: self.account, domain: domain, name: name)
        }

        public func findChannelById(channelId: Int64) -> Signal<EnginePeer?, NoError> {
            return _internal_findChannelById(
                accountPeerId: self.account.peerId,
                postbox: self.account.postbox,
                network: self.account.network,
                channelId: channelId
            )
            |> map { peer in
                return peer.flatMap(EnginePeer.init)
            }
        }

        public func channelsForStories() -> Signal<[EnginePeer], NoError> {
            return _internal_channelsForStories(account: self.account)
            |> map { peers in
                return peers.map(EnginePeer.init)
            }
        }
    }
}
```

Three consistent patterns appear:

### 1. The `_internal_` Convention

Every public engine method delegates to a free function prefixed with `_internal_`:

```swift
public func updateAddressName(domain: AddressNameDomain, name: String?)
    -> Signal<Void, UpdateAddressNameError> {
    return _internal_updateAddressName(
        account: self.account, domain: domain, name: name)
}
```

The `_internal_` functions live alongside the engine code but contain all the actual implementation — network requests, Postbox transactions, state management. This separation means:

- The public API is a thin wrapper with zero logic
- Internal functions can be refactored without changing the public surface
- Internal functions receive explicit parameters (`account`, `postbox`, `network`) rather than accessing instance state, making them easier to test and reason about

### 2. Type Wrapping at Boundaries

Engine methods return `Engine*` wrapper types instead of raw Postbox types:

```swift
// Internal function returns raw Peer
_internal_findChannelById(...) // → Signal<Peer?, NoError>

// Engine wraps it
|> map { peer in peer.flatMap(EnginePeer.init) }
```

This wrapping happens at the **boundary** — internal code works with `Peer`, `Message`, `TelegramUser`, etc., and the engine maps them to `EnginePeer`, `EngineMessage`, etc. at the last moment. This means wrapping overhead only occurs once per API call, not throughout the internal implementation.

### 3. Signal-Based Returns

Every method returns a `Signal<T, E>` from SwiftSignalKit. For one-shot operations (like checking username availability), the signal completes after delivering one value. For live-updating data (like peer presence), the signal stays active and pushes updates.

## The 19 Domain Modules

| Module | Examples | Files |
|--------|----------|-------|
| `peers` | Find peer, update username, set profile photo, ban member | 66 |
| `messages` | Send, edit, delete, search, forward, pin, translate | 54 |
| `data` | Reactive subscriptions to any Postbox data | 11 |
| `privacy` | Block users, active sessions, two-step verification | 11 |
| `auth` | Login, logout, 2FA, password recovery | 9 |
| `accountData` | Account settings, notification preferences | 7 |
| `stickers` | Sticker packs, emoji, saved GIFs | 14 |
| `calls` | Voice/video calls, group calls | 5 |
| `contacts` | Import, sync, add, delete contacts | 10 |
| `payments` | Stars, invoices, receipts | 12 |
| `localization` | Language packs, translations | 9 |
| `themes` | Chat themes, wallpapers | 4 |
| `resolve` | Deep links, username resolution | 4 |
| `itemCache` | Generic key-value cache | 3 |
| `orderedLists` | Ordered list management | 3 |
| `notices` | Dismissible notices, tips | 3 |
| `preferences` | User preferences storage | 3 |
| `resources` | Resource management | 4 |
| `historyImport` | Chat history import from other apps | 3 |
| `secureId` | Passport/ID verification | 31 |

Together these modules expose **200+ public methods** — the complete Telegram API as seen by the UI layer.

## Engine* Wrapper Types

The engine defines simplified wrapper types that hide Postbox implementation details.

### EnginePeer

```swift
public enum EnginePeer: Equatable {
    public typealias Id = PeerId

    case user(TelegramUser)
    case legacyGroup(TelegramGroup)
    case channel(TelegramChannel)
    case secretChat(TelegramSecretChat)

    public var id: Id { return self._asPeer().id }
    public var addressName: String? { return self._asPeer().addressName }
    public var displayLetters: [String] { return self._asPeer().displayLetters }

    public func _asPeer() -> Peer { /* exhaustive enum conversion */ }
}
```

`EnginePeer` is an enum that unifies the four peer types (`TelegramUser`, `TelegramGroup`, `TelegramChannel`, `TelegramSecretChat`) into a single type. UI code can pattern-match on it when it needs type-specific behavior, or use the common properties (`id`, `addressName`) without caring about the concrete type.

It also defines nested types for common peer data:

```swift
public extension EnginePeer {
    struct Presence: Equatable {
        public enum Status: Comparable {
            case present(until: Int32)
            case recently(isHidden: Bool)
            case lastWeek(isHidden: Bool)
            case lastMonth(isHidden: Bool)
            case longTimeAgo
        }
        public var status: Status
        public var lastActivity: Int32
    }

    struct NotificationSettings: Equatable {
        public enum MuteState: Equatable {
            case `default`
            case unmuted
            case muted(until: Int32)
        }
        public var muteState: MuteState
        public var messageSound: MessageSound
        public var displayPreviews: DisplayPreviews
    }
}
```

### EngineMessage

```swift
public final class EngineMessage: Equatable {
    public typealias Id = MessageId
    public typealias Index = MessageIndex
    public typealias Tags = MessageTags

    private let impl: Message

    public var id: Id { return self.impl.id }
    public var text: String { return self.impl.text }
    public var timestamp: Int32 { return self.impl.timestamp }
    public var author: EnginePeer? {
        return self.impl.author.flatMap(EnginePeer.init)
    }
    public var media: [Media] { return self.impl.media }
    public var attributes: [MessageAttribute] { return self.impl.attributes }

    public init(_ impl: Message) { self.impl = impl }
    public func _asMessage() -> Message { return self.impl }
}
```

`EngineMessage` wraps the internal `Message` class with a public-friendly interface. Notice that `author` returns `EnginePeer?` instead of `Peer?` — the wrapping cascades through the type system.

The `_asMessage()` escape hatch allows code that needs the raw type (like message rendering nodes) to unwrap it. The underscore prefix signals "you're crossing the abstraction boundary."

## EngineData: Reactive Database Queries

The `data` subsystem is architecturally distinct from the other modules. Instead of imperative methods, it provides a **declarative query system** for subscribing to database changes:

```swift
public extension TelegramEngine {
    final class EngineData {
        let accountPeerId: PeerId
        let postbox: Postbox

        public func subscribe<T0: TelegramEngineDataItem>(_ t0: T0)
            -> Signal<T0.Result, NoError>

        public func subscribe<T0: TelegramEngineDataItem,
                              T1: TelegramEngineDataItem>(_ t0: T0, _ t1: T1)
            -> Signal<(T0.Result, T1.Result), NoError>

        // ... up to 10 items

        public func get<T0: TelegramEngineDataItem>(_ t0: T0)
            -> Signal<T0.Result, NoError> {
            return self.subscribe(t0) |> take(1)
        }
    }
}
```

`subscribe` returns a live signal that emits whenever the underlying data changes. `get` returns a one-shot signal (takes only the first value). The variadic overloads accept up to 10 items, returning a tuple:

```swift
// Subscribe to peer info and notification settings simultaneously
engine.data.subscribe(
    TelegramEngine.EngineData.Item.Peer.Peer(id: userId),
    TelegramEngine.EngineData.Item.Peer.NotificationSettings(id: userId)
)
|> map { (peer, notificationSettings) in
    // Both values update atomically
}
```

### Data Items: The Protocol Stack

Each data item conforms to a protocol chain:

```swift
public protocol TelegramEngineDataItem {
    associatedtype Result
}

public protocol TelegramEngineMapKeyDataItem {
    associatedtype Key: Hashable
    var mapKey: Key { get }
}

protocol PostboxViewDataItem: TelegramEngineDataItem {
    var key: PostboxViewKey { get }
    func extract(view: PostboxView) -> Result
}
```

A data item defines:
1. What **result type** it produces
2. What **Postbox view key** it needs (which database table/query)
3. How to **extract** the result from the raw Postbox view

Here's a concrete example:

```swift
public extension TelegramEngine.EngineData.Item {
    enum Peer {
        public struct Peer: TelegramEngineDataItem, TelegramEngineMapKeyDataItem,
                           PostboxViewDataItem {
            public typealias Result = Optional<EnginePeer>

            fileprivate var id: EnginePeer.Id
            public var mapKey: EnginePeer.Id { return self.id }

            var key: PostboxViewKey { return .basicPeer(self.id) }

            func extract(view: PostboxView) -> Result {
                guard let view = view as? BasicPeerView else {
                    preconditionFailure()
                }
                guard let peer = view.peer else { return nil }
                return EnginePeer(peer)
            }
        }
    }
}
```

The system works by:
1. Collecting all `PostboxViewKey` values from the items
2. Creating a `combinedView(keys:)` on the Postbox — a single subscription that watches all keys
3. When any key's view updates, extracting results from all views
4. Returning the results as a typed tuple

This batching is crucial for performance. Instead of each UI component creating its own Postbox subscription (which would require separate database queries), `EngineData.subscribe` combines them into a single multi-key observation.

### Collection Types: EngineDataMap and EngineDataList

For bulk queries, two collection wrappers are available:

```swift
// Get notification settings for multiple peers at once
engine.data.subscribe(
    EngineDataMap(
        peers.map { TelegramEngine.EngineData.Item.Peer.NotificationSettings(id: $0.id) }
    )
)
// Result: Signal<[EnginePeer.Id: EnginePeer.NotificationSettings], NoError>
```

`EngineDataMap` collects results into a dictionary indexed by `mapKey`:

```swift
public final class EngineDataMap<Item: TelegramEngineDataItem &
    TelegramEngineMapKeyDataItem>: TelegramEngineDataItem
{
    public typealias Result = [Item.Key: Item.Result]
    // ...
}
```

`EngineDataList` collects results into an ordered array:

```swift
public final class EngineDataList<Item: TelegramEngineDataItem &
    TelegramEngineMapKeyDataItem>: TelegramEngineDataItem
{
    public typealias Result = [Item.Result]
    // ...
}
```

`EngineDataOptional` wraps a potentially-absent item:

```swift
public final class EngineDataOptional<Item: TelegramEngineDataItem>:
    TelegramEngineDataItem
{
    public typealias Result = Item.Result?
}
```

## Unauthorized Engine

For the login flow (before the user has authenticated), a separate `TelegramEngineUnauthorized` provides a limited API:

```swift
public final class TelegramEngineUnauthorized {
    public let account: UnauthorizedAccount

    public lazy var auth: Auth = Auth(account: self.account)
    public lazy var localization: Localization = Localization(account: self.account)
    public lazy var resolve: Resolve = Resolve(account: self.account)
}
```

Only three subsystems are available — authentication, localization (so the login screen can be translated), and URL resolution (for handling deep links during login). Everything else requires an authenticated account.

The `SomeTelegramEngine` enum provides a type-safe union:

```swift
public enum SomeTelegramEngine {
    case authorized(TelegramEngine)
    case unauthorized(TelegramEngineUnauthorized)
}
```

## The Account Object

TelegramEngine wraps `Account`, which holds the core infrastructure:

```swift
public class Account {
    public let id: AccountRecordId
    public let basePath: String
    public let postbox: Postbox                              // SQLite database
    public let network: Network                              // MTProto network
    public let peerId: PeerId                                // Current user's ID
    public private(set) var stateManager: AccountStateManager!  // Sync engine
    public private(set) var viewTracker: AccountViewTracker!    // Postbox views
    public private(set) var pendingMessageManager: PendingMessageManager!
    public private(set) var callSessionManager: CallSessionManager!
}
```

Engine subsystems access `account.postbox` for database operations, `account.network` for API calls, and `account.stateManager` for state synchronization. The `Account` object is the nexus that connects all layers.

## How UI Code Uses the Engine

Here's a typical usage pattern in a view model:

```swift
// Fetch peer info (one-shot)
let peer = engine.data.get(
    TelegramEngine.EngineData.Item.Peer.Peer(id: peerId)
)

// Subscribe to live presence updates
let presence = engine.data.subscribe(
    TelegramEngine.EngineData.Item.Peer.Presence(id: peerId)
)

// Send a message
engine.messages.sendMessage(
    peerId: peerId,
    text: "Hello!",
    // ...
)

// Search messages
engine.messages.searchMessages(
    location: .general(tags: nil, minDate: nil, maxDate: nil),
    query: "search term",
    state: nil,
    limit: 50
)

// Update privacy settings
engine.privacy.requestUpdatePeerIsBlocked(peerId: spammerId, isBlocked: true)
```

The UI code never touches `Postbox`, `Network`, or `Api` types directly. Everything flows through `TelegramEngine`.

## Architectural Summary

TelegramEngine is a textbook application of the **Facade pattern** at scale:

1. **Lazy subsystems** — 19 modules loaded on demand, organized by domain
2. **Delegation convention** — public methods delegate to `_internal_` functions for clean separation
3. **Type wrapping** — `EnginePeer`, `EngineMessage`, etc. simplify raw Postbox models for consumers
4. **Reactive queries** — `EngineData.subscribe()` with variadic generics enables type-safe, batched database observations
5. **Collection types** — `EngineDataMap` and `EngineDataList` handle bulk queries efficiently
6. **Authorization scoping** — `TelegramEngine` vs `TelegramEngineUnauthorized` ensures only valid operations are available

The result is that a 274-module iOS app with 250,000+ lines of networking and persistence code presents itself to the UI layer as a set of clean, well-typed, reactive method calls. Any developer can call `engine.messages.sendMessage(...)` without understanding MTProto serialization, Postbox transactions, or the pts counter system. The complexity is contained; the interface is simple.
