---
title: "Reactive Patterns in Practice: The Telegram iOS Cookbook"
description: "Real-world patterns from the Telegram iOS codebase — Promise vs ValuePromise, disposable lifecycle management, the cached-then-remote strategy, signal composition chains, and more."
published: 2026-02-09
tags:
  - Reactive Foundation
toc: true
lang: en
abbrlink: 05-reactive-patterns
pin: 21
---

Posts [3](/posts/03-swift-signal-kit/) and [4](/posts/04-ssignal-kit-objc/) explained every type in SwiftSignalKit and SSignalKit. Now it's time to see how these primitives combine into patterns that appear throughout the Telegram iOS codebase. These aren't theoretical — they're extracted from production code that handles millions of concurrent users.

This post is organized as a cookbook: each section presents a pattern, explains why it exists, and shows real code from the Telegram source.

## Pattern 1: Promise vs ValuePromise — Choosing the Right Reactive Variable

Both `Promise<T>` and `ValuePromise<T>` cache a value and replay it to new subscribers. The difference is subtle but critical:

| | `Promise<T>` | `ValuePromise<T>` |
|---|---|---|
| **Set with** | `set(Signal<T, NoError>)` | `set(T)` |
| **Source** | An entire signal chain | A single value |
| **Replays** | Latest value from the signal | The value itself |
| **Dedup** | No | Optional `ignoreRepeated` |

### When to Use ValuePromise

Use `ValuePromise` when the state is simple — a boolean flag, an enum, a counter — and you set it synchronously:

```swift
// TelegramCore/Sources/Account/Account.swift:1187

private let _loggedOut = ValuePromise<Bool>(false, ignoreRepeated: true)
public var loggedOut: Signal<Bool, NoError> {
    return self._loggedOut.get()
}

private let _importantTasksRunning = ValuePromise<AccountRunningImportantTasks>(
    [], ignoreRepeated: true
)
public var importantTasksRunning: Signal<AccountRunningImportantTasks, NoError> {
    return self._importantTasksRunning.get()
}
```

Why `ValuePromise` here:
- The `loggedOut` state changes based on a specific event (server response or user action) — you know the exact value at the moment you set it
- `ignoreRepeated: true` prevents redundant UI updates when the same value is set multiple times
- No need for signal composition — just `self._loggedOut.set(true)` when logout happens

Another common use — UI readiness flags:

```swift
// TelegramUI/Sources/ChatController.swift:265

let contentDataReady = ValuePromise<Bool>(false, ignoreRepeated: true)
```

And voice call state:

```swift
// TelegramCallsUI/Sources/PresentationGroupCall.swift:599

private let isMutedPromise = ValuePromise<PresentationGroupCallMuteAction>(
    .muted(isPushToTalkActive: false)
)
public var isMuted: Signal<Bool, NoError> {
    return self.isMutedPromise.get()
    |> map { value -> Bool in
        switch value {
        case let .muted(isPushToTalkActive):
            return !isPushToTalkActive
        case .unmuted:
            return false
        }
    }
}
```

Note how `isMutedPromise` stores the raw enum, but the public `isMuted` signal maps it to a simple bool. The `ValuePromise` holds the internal state; the `Signal` is the public interface.

### When to Use Promise

Use `Promise` when the value comes from an asynchronous source — a network request, a database query, or a signal chain:

```swift
// AccountContext/Sources/UniversalVideoNode.swift:123

private let _status = Promise<MediaPlayerStatus?>()
public var status: Signal<MediaPlayerStatus?, NoError> {
    return self._status.get()
}

private let _bufferingStatus = Promise<(RangeSet<Int64>, Int64)?>()
public var bufferingStatus: Signal<(RangeSet<Int64>, Int64)?, NoError> {
    return self._bufferingStatus.get()
}

private let _ready = Promise<Void>()
public var ready: Signal<Void, NoError> {
    return self._ready.get()
}
```

Why `Promise` here:
- Media player status comes from an ongoing signal (the player's state stream), not a single value
- You set it with `self._status.set(someSignal)` where `someSignal` is a continuous stream of status updates
- When you switch video sources, calling `set()` with a new signal automatically unsubscribes from the old one

**The rule of thumb**: If you call `.set(value)` — use `ValuePromise`. If you call `.set(signal)` — use `Promise`.

## Pattern 2: MetaDisposable — The Subscription Swapper

The most common disposable pattern in Telegram is `MetaDisposable` for "latest-wins" subscriptions:

```swift
// TelegramUI/Sources/ChatController.swift:272-280

let preloadNextChatPeerIdDisposable = MetaDisposable()
let navigationActionDisposable = MetaDisposable()
let messageIndexDisposable = MetaDisposable()
```

When a user taps a contact to open their profile, the previous profile-loading subscription is automatically cancelled:

```swift
strongSelf.navigationActionDisposable.set(
    (strongSelf.context.account.postbox.loadedPeerWithId(peerId.id)
    |> take(1)
    |> deliverOnMainQueue).startStrict(next: { [weak self] peer in
        if let strongSelf = self {
            if peer.restrictionText(platform: "ios",
                contentSettings: strongSelf.context.currentContentSettings.with { $0 }
            ) == nil {
                if let infoController = strongSelf.context.sharedContext
                    .makePeerInfoController(...) {
                    strongSelf.effectiveNavigationController?
                        .pushViewController(infoController)
                }
            }
        }
    })
)
```

The `MetaDisposable.set()` call:
1. Disposes the previous subscription (if any)
2. Stores the new one
3. If the MetaDisposable itself was already disposed, disposes the new subscription immediately

This prevents a class of bugs where rapid user interaction (tapping multiple contacts quickly) creates zombie subscriptions that deliver stale results.

## Pattern 3: DisposableDict — Keyed Subscription Management

When you need multiple independent subscriptions indexed by some key:

```swift
// TelegramCore/Sources/State/AccountViewTracker.swift:301-319

private var updatedViewCountDisposables = DisposableDict<Int32>()
private var updatedReactionsDisposables = DisposableDict<Int32>()
private var seenLiveLocationDisposables = DisposableDict<Int32>()
private var updatedExtendedMediaDisposables = DisposableDict<Int32>()
```

Each message has its own subscription for tracking view counts, reactions, etc. The `Int32` key is the message ID. When a message scrolls off screen, its disposable is removed. When the tracker is deallocated, all disposables are cleaned up at once.

Another real example — managing per-account disposables:

```swift
// TelegramUI/Sources/SharedAccountContext.swift:165

private let managedAccountDisposables = DisposableDict<AccountRecordId>()
```

Each account gets its own disposable for state management. When an account is removed, only its disposable is cancelled.

And for live location broadcasting — each message that shares a live location gets its own edit disposable:

```swift
// LiveLocationManager/Sources/LiveLocationManager.swift:46

private let editMessageDisposables = DisposableDict<EngineMessage.Id>()
```

## Pattern 4: DisposableSet — Parallel Operations with Shared Cleanup

When multiple subscriptions should live and die together:

```swift
// TelegramCore/Sources/TelegramEngine/Messages/BotWebView.swift:198

let disposableSet = DisposableSet()
disposableSet.add(pollDisposable)
disposableSet.add(dismissDisposable)
return disposableSet
```

Bot web views need both periodic polling AND dismiss-event listening. Both run concurrently. When the web view closes, one `dispose()` call cancels both.

The typical lifecycle pattern for a view controller:

```swift
class SomeController {
    private let disposables = DisposableSet()

    func setup() {
        disposables.add(dataSignal.start(next: { ... }))
        disposables.add(presenceSignal.start(next: { ... }))
        disposables.add(notificationSignal.start(next: { ... }))
    }

    deinit {
        disposables.dispose()
    }
}
```

## Pattern 5: cached |> then(remote) — The Cache-First Strategy

This is the most architecturally significant pattern in TelegramCore. Nearly every data-fetching function follows the same structure:

```swift
// TelegramCore/Sources/TelegramEngine/Localization/Localizations.swift:45-69

func _internal_availableLocalizations(
    postbox: Postbox, network: Network, allowCached: Bool
) -> Signal<[LocalizationInfo], NoError> {

    // Step 1: Try the cache
    let cached: Signal<[LocalizationInfo], NoError>
    if allowCached {
        cached = postbox.transaction { transaction -> Signal<[LocalizationInfo], NoError> in
            if let entry = transaction.retrieveItemCacheEntry(
                id: ItemCacheEntryId(
                    collectionId: Namespaces.CachedItemCollection.cachedAvailableLocalizations,
                    key: ValueBoxKey(length: 0)
                )
            )?.get(CachedLocalizationInfos.self) {
                return .single(entry.list)
            }
            return .complete()       // No cache → emit nothing, don't block
        } |> switchToLatest
    } else {
        cached = .complete()
    }

    // Step 2: Fetch from network and update cache
    let remote = network.request(Api.functions.langpack.getLanguages(langPack: ""))
    |> retryRequest
    |> mapToSignal { languages -> Signal<[LocalizationInfo], NoError> in
        let infos: [LocalizationInfo] = languages.map(LocalizationInfo.init(apiLanguage:))
        return postbox.transaction { transaction -> [LocalizationInfo] in
            if let entry = CodableEntry(CachedLocalizationInfos(list: infos)) {
                transaction.putItemCacheEntry(
                    id: ItemCacheEntryId(
                        collectionId: Namespaces.CachedItemCollection.cachedAvailableLocalizations,
                        key: ValueBoxKey(length: 0)
                    ),
                    entry: entry
                )
            }
            return infos
        }
    }

    // Step 3: Emit cache immediately, then fresh data
    return cached |> then(remote)
}
```

The data flow:

```
Subscriber receives:
  1. [cached localizations]       ← instant, from Postbox
  2. [fresh localizations]        ← after network round-trip, also saved to Postbox
```

The `then` operator emits all values from the first signal, waits for it to complete, then subscribes to the second signal and emits its values. Since the cache signal emits `.single(list)` (which completes immediately) or `.complete()` (if no cache), the remote signal always runs.

Why this matters:
- **Instant UI**: The cached data renders the screen immediately. No loading spinner for returning users.
- **Eventually consistent**: The fresh data arrives seconds later and updates the UI.
- **Offline-first**: If the network request fails, the UI still shows cached data.
- **Self-healing cache**: Every successful network response writes to the cache, so the next launch gets fresher data.

### The Transaction |> switchToLatest Pattern

Notice the `|> switchToLatest` after the Postbox transaction:

```swift
cached = postbox.transaction { transaction -> Signal<[LocalizationInfo], NoError> in
    // Returns a Signal, not a value
    if let entry = ... {
        return .single(entry.list)
    }
    return .complete()
} |> switchToLatest
```

`postbox.transaction` returns `Signal<T, NoError>` where `T` is the closure's return type. When the closure itself returns a `Signal`, the result is `Signal<Signal<[LocalizationInfo], NoError>, NoError>` — a signal of signals. The `switchToLatest` flattens this to `Signal<[LocalizationInfo], NoError>`.

This pattern appears everywhere when you need to make a decision inside a transaction that determines what happens next.

## Pattern 6: Network → Transform → Persist Chains

The localization download function shows a full multi-step chain:

```swift
// TelegramCore/Sources/TelegramEngine/Localization/Localizations.swift:111

func _internal_downloadAndApplyLocalization(
    accountManager: AccountManager<TelegramAccountManagerTypes>,
    postbox: Postbox,
    network: Network,
    languageCode: String
) -> Signal<Void, DownloadAndApplyLocalizationError> {

    return _internal_requestLocalizationPreview(network: network, identifier: languageCode)
    |> mapError { _ -> DownloadAndApplyLocalizationError in .generic }
    |> mapToSignal { preview -> Signal<Void, DownloadAndApplyLocalizationError> in

        // Download primary (and optional secondary) localization in parallel
        var downloads: [Signal<Localization, DownloadLocalizationError>] = []
        downloads.append(_internal_downloadLocalization(
            network: network, languageCode: preview.languageCode
        ))
        if let secondaryCode = preview.baseLanguageCode {
            downloads.append(_internal_downloadLocalization(
                network: network, languageCode: secondaryCode
            ))
        }

        return combineLatest(downloads)
        |> mapError { _ -> DownloadAndApplyLocalizationError in .generic }
        |> mapToSignal { components -> Signal<Void, DownloadAndApplyLocalizationError> in

            // Save to AccountManager
            return accountManager.transaction { transaction -> Void in
                transaction.updateSharedData(SharedDataKeys.localizationSettings, { _ in
                    return PreferencesEntry(LocalizationSettings(
                        primaryComponent: LocalizationComponent(...),
                        secondaryComponent: secondaryComponent
                    ))
                })
            }
            |> castError(DownloadAndApplyLocalizationError.self)
            |> mapToSignal { _ -> Signal<Void, DownloadAndApplyLocalizationError> in

                // Also save to Postbox
                return postbox.transaction { transaction -> Void in
                    updateLocalizationListStateInteractively(
                        transaction: transaction, { state in ... }
                    )
                }
                |> castError(DownloadAndApplyLocalizationError.self)
            }
        }
    }
}
```

The chain reads like a recipe:
1. **Request preview** from API → get language metadata
2. **Download localizations** in parallel with `combineLatest`
3. **Save to AccountManager** (shared settings across accounts)
4. **Save to Postbox** (per-account persistent state)
5. All error types are unified with `mapError`/`castError`

### combineLatest for Parallel Downloads

```swift
var downloads: [Signal<Localization, DownloadLocalizationError>] = []
downloads.append(downloadPrimary)
if let secondaryCode = preview.baseLanguageCode {
    downloads.append(downloadSecondary)
}
return combineLatest(downloads)
```

`combineLatest` subscribes to all signals simultaneously and waits for each to produce at least one value. When both downloads complete, it emits a single `[Localization]` array. If either fails, the error propagates immediately.

### mapError and castError: Unifying Error Types

Telegram's signal chains often cross error-type boundaries. Two operators handle this:

```swift
// mapError: Transform the error value
|> mapError { _ -> DownloadAndApplyLocalizationError in .generic }

// castError: Change the error type when the signal can't actually error
|> castError(DownloadAndApplyLocalizationError.self)
```

`mapError` transforms one error type to another (useful when a downstream chain expects a different error). `castError` is for signals with `NoError` that need to be composed with failable signals — it changes the type signature without adding actual error-handling logic.

## Pattern 7: combineLatest for UI State Composition

UI screens often merge multiple data sources into a single view model:

```swift
// AccountUtils/Sources/AccountUtils.swift:10

public func activeAccountsAndPeers(context: AccountContext, includePrimary: Bool = false)
    -> Signal<((AccountContext, EnginePeer)?,
               [(AccountContext, EnginePeer, Int32)]), NoError> {

    return context.sharedContext.activeAccountContexts
    |> mapToSignal { primary, activeAccounts, _ -> Signal<...> in

        // For each account, create a signal that combines peer data + unread count
        var accounts: [Signal<(AccountContext, EnginePeer, Int32)?, NoError>] = []

        func accountWithPeer(_ context: AccountContext) -> Signal<...> {
            return combineLatest(
                context.account.postbox.peerView(id: context.account.peerId),
                renderedTotalUnreadCount(
                    accountManager: sharedContext.accountManager,
                    engine: context.engine
                )
            )
            |> map { view, totalUnreadCount -> (EnginePeer?, Int32) in
                return (view.peers[view.peerId].flatMap(EnginePeer.init),
                        totalUnreadCount.0)
            }
            |> distinctUntilChanged { lhs, rhs in
                lhs.0 != rhs.0 || lhs.1 != rhs.1 ? false : true
            }
            |> map { peer, totalUnreadCount in
                peer.map { (context, $0, totalUnreadCount) }
            }
        }

        for (_, context, _) in activeAccounts {
            accounts.append(accountWithPeer(context))
        }

        // Outer combineLatest: merge all account signals
        return combineLatest(accounts)
        |> map { accounts -> ((AccountContext, EnginePeer)?,
                              [(AccountContext, EnginePeer, Int32)]) in
            // Extract primary and filter list
            ...
        }
    }
}
```

The structure is:
1. For each account: `combineLatest(peerView, unreadCount)` → merge two data sources
2. `distinctUntilChanged` → suppress duplicate emissions
3. Outer `combineLatest(accounts)` → combine all accounts into a single array
4. `map` → shape into the final view model tuple

This powers the account switcher. When any account's unread count changes, only that account's inner `combineLatest` re-emits, which triggers the outer `combineLatest` to produce a new combined state.

## Pattern 8: deliverOnMainQueue for UI Updates

Every signal chain that touches UIKit must deliver on the main queue. Telegram has a universal pattern:

```swift
// The standard subscription ending
(someSignal
|> deliverOnMainQueue).startStrict(next: { [weak self] value in
    guard let strongSelf = self else { return }
    strongSelf.updateUI(with: value)
})
```

Why `deliverOnMainQueue` and not `DispatchQueue.main.async`?

1. **Queue identity check**: If `deliverOnMainQueue` is called from code that's already on the main queue, the block executes synchronously. No GCD dispatch overhead, no frame delay.

2. **Signal-level, not call-level**: The operator wraps the entire subscription. Every `next`, `error`, and `completion` is dispatched. You can't accidentally forget to dispatch one callback.

3. **Composability**: It's part of the signal chain. You can add operators before or after it without restructuring your code.

### startStrict vs start

You'll see two variants throughout the codebase:

```swift
// Strict: crashes in DEBUG if you forget to dispose
signal.startStrict(next: { ... }, file: #file, line: #line)

// Lenient: silently leaks if you forget to dispose
signal.start(next: { ... })
```

The `startStrict` variant wraps the returned disposable in `StrictDisposable`, which asserts on dealloc if `dispose()` was never called. In practice:

- Use `startStrict` when the disposable is stored in a property (`MetaDisposable`, `DisposableSet`, etc.)
- Use `start` for fire-and-forget operations that naturally complete (like a `take(1)` signal)

The Telegram codebase has been gradually migrating from `start` to `startStrict` to catch disposal leaks earlier. When you see both in the same file, the `start` calls are typically older code that hasn't been updated.

## Pattern 9: take(1) — One-Shot Signals

When you only need the current value from a continuous signal:

```swift
strongSelf.navigationActionDisposable.set(
    (strongSelf.context.account.postbox.loadedPeerWithId(peerId.id)
    |> take(1)
    |> deliverOnMainQueue).startStrict(next: { [weak self] peer in
        // Use peer once
    })
)
```

`take(1)` subscribes, waits for the first value, emits it, then completes and disposes. This is essential for Postbox signals, which are **continuous** — they emit the current value and then keep emitting updates whenever the data changes. Without `take(1)`, you'd get a stream of updates every time anyone changes the peer's data.

The pattern is also used for network requests that return a single result:

```swift
return network.request(Api.functions.help.getConfig())
|> retryRequest                    // Retry on transient errors
|> mapToSignal { result -> ... }
```

`network.request` naturally completes after one response, but `retryRequest` may restart it multiple times. The consumer typically only cares about the final successful response.

## Pattern 10: The weak/strong Dance

Every closure that captures `self` uses the same pattern:

```swift
(someSignal
|> deliverOnMainQueue).startStrict(next: { [weak self] value in
    guard let strongSelf = self else { return }
    strongSelf.someProperty = value
    strongSelf.someMethod()
})
```

Why not capture `self` strongly?

Because signal subscriptions can outlive the object that created them. If a view controller subscribes to a network signal and the user navigates away before the response arrives, the view controller should be deallocated. A strong capture in the `next` block would keep the view controller alive until the signal completes — potentially forever for continuous signals.

The `[weak self]` + `guard let strongSelf` pattern:
1. Captures `self` weakly — won't prevent deallocation
2. Promotes to strong only for the duration of the callback
3. If `self` was already deallocated, the `guard` returns early

This is so universal in Telegram's codebase that you can assume every closure in a signal chain captures `self` weakly unless you see otherwise.

## Pattern 11: The SVariable / Promise Lifecycle

`SVariable` (ObjC) and `Promise` (Swift) follow the same lifecycle pattern for reactive state that needs to track an external signal:

```swift
// Swift pattern
class SomeManager {
    private let currentDataPromise = Promise<Data?>(nil)
    var currentData: Signal<Data?, NoError> {
        return currentDataPromise.get()
    }

    func switchToSource(_ signal: Signal<Data?, NoError>) {
        // Automatically unsubscribes from previous source
        currentDataPromise.set(signal)
    }
}
```

```objc
// ObjC equivalent
@interface SomeManager () {
    SVariable *_currentData;
}
@end

@implementation SomeManager
- (SSignal *)currentData {
    return [_currentData signal];
}

- (void)switchToSource:(SSignal *)signal {
    [_currentData set:signal];
}
@end
```

The key behavior:
- Calling `set()` with a new signal **automatically cancels** the subscription to the previous signal (via the internal `MetaDisposable`/`SMetaDisposable`)
- The latest value from the new signal is cached and replayed to any current and future subscribers
- Subscribers see a seamless transition — they don't know or care that the underlying source changed

## Pattern 12: retryRequest — Network Resilience

Network requests in Telegram never use raw `network.request()` without error handling:

```swift
let remote = network.request(Api.functions.langpack.getLanguages(langPack: ""))
|> retryRequest
|> mapToSignal { ... }
```

`retryRequest` is defined in TelegramCore and implements exponential backoff with jitter:
- On transient network errors, it waits and retries
- On permanent errors (4xx), it fails immediately
- The retry count and delay are tuned for Telegram's specific error semantics

This is applied to almost every API call, which is why Telegram feels resilient on flaky connections.

## Pattern 13: Signal Factories and the |> Operator

In TelegramCore, functions that return signals are called **signal factories**. They take parameters and return a configured `Signal`:

```swift
// This is a signal factory — it doesn't execute anything yet
func _internal_currentlySuggestedLocalization(
    network: Network, extractKeys: [String]
) -> Signal<SuggestedLocalizationInfo?, NoError> {
    return network.request(Api.functions.help.getConfig())
    |> retryRequest
    |> mapToSignal { result -> Signal<SuggestedLocalizationInfo?, NoError> in
        switch result {
        case let .config(configData):
            if let suggestedLangCode = configData.suggestedLangCode {
                return _internal_suggestedLocalizationInfo(
                    network: network,
                    languageCode: suggestedLangCode,
                    extractKeys: extractKeys
                )
                |> map(Optional.init)
            } else {
                return .single(nil)
            }
        }
    }
}
```

The `|>` pipe operator chains these factories together:

```swift
return network.request(...)           // Signal<Api.Config, MTRpcError>
|> retryRequest                        // Signal<Api.Config, NoError>
|> mapToSignal { config in ... }       // Signal<SuggestedLocalizationInfo?, NoError>
```

Each line transforms the signal type. Reading a `|>` chain top-to-bottom tells you the exact data flow from source to subscriber.

### Why |> Instead of Method Chaining

Method chaining (`signal.map{}.filter{}`) requires operators to be defined as methods on `Signal`. With 30+ operators, `Signal` would have a massive API surface, making autocomplete useless and documentation overwhelming.

The `|>` approach defines each operator as a free function that returns a `(Signal<A, E>) -> Signal<B, E>` closure:

```swift
// This is all that's needed to define an operator
public func map<T, E, R>(_ f: @escaping (T) -> R) -> (Signal<T, E>) -> Signal<R, E> {
    return { signal in
        return Signal { subscriber in
            return signal.start(next: { value in
                subscriber.putNext(f(value))
            }, error: { error in
                subscriber.putError(error)
            }, completed: {
                subscriber.putCompletion()
            })
        }
    }
}
```

Benefits:
- Operators are modular — each lives in its own file
- Custom operators for specific domains (e.g., `retryRequest`) don't pollute the base type
- The type signature tells you exactly what the operator does: it takes a signal of `T` and returns a signal of `R`

## Pattern 14: Postbox Live Monitoring

Postbox (Telegram's persistence layer) signals are fundamentally different from network signals. They're **continuous** — they emit the current state immediately and then keep emitting updates:

```swift
// LiveLocationManager/Sources/LiveLocationManager.swift:56

self.messagesDisposable = (
    self.engine.messages.activeLiveLocationMessages()
    |> deliverOn(self.queue)
).start(next: { [weak self] messages in
    guard let strongSelf = self else { return }

    let timestamp = Int32(CFAbsoluteTimeGetCurrent() + kCFAbsoluteTimeIntervalSince1970)
    var broadcastToMessageIds: [EngineMessage.Id: Int32] = [:]
    var stopMessageIds = Set<EngineMessage.Id>()

    for message in messages {
        if !message.flags.contains(.Incoming) {
            if message.flags.intersection([.Failed, .Unsent]).isEmpty {
                var activeLiveBroadcastingTimeout: Int32?
                for media in message.media {
                    if let telegramMap = media as? TelegramMediaMap {
                        if let timeout = telegramMap.liveBroadcastingTimeout {
                            if timeout == liveLocationIndefinitePeriod ||
                               message.timestamp + timeout > timestamp {
                                activeLiveBroadcastingTimeout = timeout
                            }
                        }
                    }
                }
                if let timeout = activeLiveBroadcastingTimeout {
                    broadcastToMessageIds[message.id] = timeout == liveLocationIndefinitePeriod
                        ? timeout
                        : message.timestamp + timeout
                } else {
                    stopMessageIds.insert(message.id)
                }
            }
        }
    }

    strongSelf.update(
        broadcastToMessageIds: broadcastToMessageIds,
        stopMessageIds: stopMessageIds
    )
})
```

The `activeLiveLocationMessages()` signal emits whenever the set of live-location-sharing messages changes — when a new share starts, when one expires, when a message is deleted. The manager doesn't poll; it reacts to database changes.

This is why Telegram's UI feels so responsive: data changes propagate automatically from Postbox through the signal chain to the UI, with no manual refresh logic.

## Pattern 15: Resource Loading with Fallback

Photo and media loading uses `combineLatest` for speculative parallel fetching:

```swift
// PhotoResources/Sources/PhotoResources.swift:704

let signal = combineLatest(
    maybePreviewSourceFullSize,
    maybeFullSize
)
|> map { maybePreviewSourceFullSize, maybeFullSize -> MediaResourceData in
    if maybePreviewSourceFullSize.complete {
        return maybePreviewSourceFullSize    // Prefer preview source
    } else {
        return maybeFullSize                  // Fall back to full size
    }
}
|> take(1)
|> mapToSignal { maybeData -> Signal<Tuple3<Data?, Tuple2<Data, String>?, Bool>, NoError> in
    if maybeData.complete && !forceThumbnail {
        let loadedData = try? Data(contentsOf: URL(fileURLWithPath: maybeData.path))
        return .single(Tuple(nil, loadedData.map { Tuple($0, maybeData.path) }, true))
    } else {
        // Fall back to thumbnail loading
        ...
    }
}
```

The strategy:
1. Start downloading from two sources simultaneously (`combineLatest`)
2. Pick whichever completes first (`map` with preference logic)
3. Take only the first successful result (`take(1)`)
4. If the data is ready, use it; otherwise fall back to a thumbnail

This is why Telegram shows images so fast — it speculatively loads from multiple sources and uses the first available one.

## Summary: Pattern Quick Reference

| Pattern | When to Use | Key Operator |
|---|---|---|
| `ValuePromise` | Simple synchronous state | `.set(value)` |
| `Promise` | Async state from a signal | `.set(signal)` |
| `MetaDisposable` | Latest-wins subscription | `.set(disposable)` |
| `DisposableDict` | Keyed parallel subscriptions | `.set(key, disposable)` |
| `DisposableSet` | Parallel operations with shared lifetime | `.add(disposable)` |
| `cached \|> then(remote)` | Cache-first loading | `then` |
| `combineLatest` | Merging multiple data sources | `combineLatest` |
| `take(1)` | One-shot from continuous signal | `take` |
| `deliverOnMainQueue` | UI updates | `deliverOn` |
| `retryRequest` | Network resilience | Custom operator |
| `\|> switchToLatest` | Unwrapping transaction results | `switchToLatest` |
| `startStrict` | Leak detection for subscriptions | Debug tool |

These patterns compose. A real Telegram signal chain might use five or six of them together:

```swift
// A typical TelegramCore function
return postbox.transaction { transaction -> [LocalizationInfo]? in
    // Read cache
}
|> switchToLatest                    // Pattern: transaction unwrap
|> mapToSignal { cached in
    if let cached = cached {
        return .single(cached)       // Pattern: cache-first
        |> then(remote)
    } else {
        return remote
    }
}
|> retryRequest                       // Pattern: network resilience
|> deliverOnMainQueue                 // Pattern: UI delivery
```

In the [next post](/posts/06-postbox-overview/), we'll dive into Postbox — the custom SQLite persistence layer that makes all these reactive patterns possible by turning database state into live signals.
