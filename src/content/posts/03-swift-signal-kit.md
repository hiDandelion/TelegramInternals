---
title: "SwiftSignalKit: Telegram's Reactive Framework from Scratch"
description: "A complete deep dive into SwiftSignalKit — the custom reactive framework that predates Combine by years and powers every async operation in Telegram iOS."
published: 2026-02-07
tags:
  - Reactive Foundation
toc: true
lang: en
abbrlink: 03-swift-signal-kit
pin: 23
---

Every asynchronous operation in Telegram iOS — network requests, database queries, UI state updates, file downloads, WebSocket messages — flows through **SwiftSignalKit**, a custom reactive framework that Telegram built years before Apple shipped Combine. Understanding it is a prerequisite to reading any other module in the codebase.

This post covers every core type in the framework: `Signal`, `Subscriber`, `Disposable`, `Promise`, `ValuePromise`, `ValuePipe`, `Queue`, `Bag`, `Atomic`, and the `|>` pipe operator. We'll look at the actual implementation code — these are small, elegant files that are worth reading in full.

## Why Not RxSwift or Combine?

SwiftSignalKit was created around 2015-2016, before Combine existed (2019) and when RxSwift was still maturing. But even today, Telegram hasn't migrated. Here's why:

1. **Performance**: SwiftSignalKit uses `pthread_mutex_t` directly for synchronization — not GCD queues, not `NSLock`, not `os_unfair_lock`. This gives the absolute minimum overhead per lock/unlock cycle.

2. **Simplicity**: The entire framework is 26 files totaling ~2,000 lines. Compare that to RxSwift (~50,000 lines) or Combine (closed-source, but massive). Fewer abstractions means fewer surprises.

3. **No scheduler abstraction**: RxSwift has `Scheduler` with many implementations. Combine has `Scheduler` protocol. SwiftSignalKit has `Queue` — a thin wrapper around `DispatchQueue`. That's it. Queue dispatch is explicit and always visible.

4. **The `|>` pipe operator**: Instead of method chaining (`signal.map{}.filter{}`), SwiftSignalKit uses a free-function + pipe pattern (`signal |> map {} |> filter {}`). This enables operators to be defined as standalone functions without extending `Signal`, which avoids polluting the type with hundreds of methods.

## The Core: Signal\<T, E\>

The entire framework rests on one type. Here it is, in full:

```swift
// SwiftSignalKit/Source/Signal.swift

public enum NoValue { }
public enum NoError { }

precedencegroup PipeRight {
    associativity: left
    higherThan: DefaultPrecedence
}

infix operator |> : PipeRight

public func |> <T, U>(value: T, function: ((T) -> U)) -> U {
    return function(value)
}

public final class Signal<T, E> {
    private let generator: (Subscriber<T, E>) -> Disposable

    public init(_ generator: @escaping(Subscriber<T, E>) -> Disposable) {
        self.generator = generator
    }

    public func start(next: ((T) -> Void)! = nil, error: ((E) -> Void)! = nil,
                      completed: (() -> Void)! = nil) -> Disposable {
        let subscriber = Subscriber<T, E>(next: next, error: error, completed: completed)
        let disposable = self.generator(subscriber)
        let wrappedDisposable = subscriber.assignDisposable(disposable)
        return SubscriberDisposable(subscriber: subscriber, disposable: wrappedDisposable)
    }

    public static func single(_ value: T) -> Signal<T, E> {
        return Signal<T, E> { subscriber in
            subscriber.putNext(value)
            subscriber.putCompletion()
            return EmptyDisposable
        }
    }

    public static func complete() -> Signal<T, E> {
        return Signal<T, E> { subscriber in
            subscriber.putCompletion()
            return EmptyDisposable
        }
    }

    public static func fail(_ error: E) -> Signal<T, E> {
        return Signal<T, E> { subscriber in
            subscriber.putError(error)
            return EmptyDisposable
        }
    }

    public static func never() -> Signal<T, E> {
        return Signal<T, E> { _ in
            return EmptyDisposable
        }
    }
}
```

That's the core. Let's break down every design decision:

### The Generator Pattern

`Signal` is **cold** — it doesn't do anything until `start()` is called. The `generator` closure receives a `Subscriber` and returns a `Disposable`. This is the same pattern as RxSwift's `Observable.create` or Combine's custom `Publisher`, but with less ceremony.

When you call `signal.start(next: { ... })`, the framework:
1. Creates a `Subscriber` wrapping your callbacks
2. Calls the `generator`, which begins producing values by calling `subscriber.putNext()`
3. The generator returns a `Disposable` that cleans up resources when the subscription is cancelled
4. Wraps everything in a `SubscriberDisposable` for safe disposal

### Generic Error Type

`Signal<T, E>` is generic over both value type `T` and error type `E`. When there can be no error, the `NoError` enum is used: `Signal<String, NoError>`. When there's no meaningful value, `NoValue` is used. These are empty enums — they can never be instantiated, providing compile-time guarantees.

### The |> Pipe Operator

This is the most distinctive syntactic choice. Instead of:

```swift
// Method chaining (how RxSwift/Combine do it)
signal
    .map { $0 + 1 }
    .filter { $0 > 0 }
    .deliverOnMainQueue()
```

SwiftSignalKit does:

```swift
// Pipe operator (how Telegram does it)
signal
|> map { $0 + 1 }
|> filter { $0 > 0 }
|> deliverOnMainQueue
```

Why? Because `map`, `filter`, and `deliverOnMainQueue` are **free functions**, not methods on `Signal`. Here's the actual `map` implementation:

```swift
// SwiftSignalKit/Source/Signal_Mapping.swift
public func map<T, E, R>(_ f: @escaping(T) -> R) -> (Signal<T, E>) -> Signal<R, E> {
    return { signal in
        return Signal<R, E> { subscriber in
            return signal.start(next: { next in
                subscriber.putNext(f(next))
            }, error: { error in
                subscriber.putError(error)
            }, completed: {
                subscriber.putCompletion()
            })
        }
    }
}
```

`map` is a function that takes a transform closure and returns *another function* that takes a `Signal` and returns a new `Signal`. The `|>` operator threads the left-hand signal into the right-hand function. This curried design means:

1. **Operators don't pollute `Signal`'s interface** — you can define operators in any file without extending `Signal`.
2. **Operators compose naturally** — `signal |> map { } |> filter { }` reads left-to-right.
3. **Type inference works beautifully** — the compiler infers all generic parameters from the pipeline.

### Factory Methods

Four static factory methods cover the common cases:

| Method | Behavior |
|--------|----------|
| `.single(value)` | Emits one value, then completes |
| `.complete()` | Completes immediately with no values |
| `.fail(error)` | Fails immediately with an error |
| `.never()` | Never emits, never completes (useful as a placeholder) |

### Async/Await Bridge

There's also a bridge to Swift concurrency:

```swift
@available(iOS 13.0, macOS 10.15, *)
public extension Signal where E == NoError {
    func get() async -> T {
        let disposable = MetaDisposable()
        return await withTaskCancellationHandler(operation: {
            return await withCheckedContinuation { continuation in
                disposable.set((self |> take(1)).startStandalone(next: { value in
                    continuation.resume(returning: value)
                }))
            }
        }, onCancel: {
            disposable.dispose()
        })
    }
}
```

This lets you `await` any `Signal<T, NoError>` to get its first value. It's only available for `NoError` signals because async/await doesn't have a built-in error channel matching Signal's generic error type.

### startStrict: Leak Detection

There are three `start` variants:

```swift
// Normal start — no leak tracking
signal.start(next: { ... })

// Standalone start — no leak tracking, different internal path
signal.startStandalone(next: { ... })

// Strict start — asserts in DEBUG if the disposable is leaked
signal.startStrict(next: { ... })
```

`startStrict` wraps the returned disposable in `StrictDisposable`, which asserts in the `deinit` if `dispose()` was never called. This catches the common bug of forgetting to store a disposable, which silently cancels the subscription.

## Subscriber\<T, E\>: Thread-Safe Event Delivery

The `Subscriber` is the bridge between the signal's generator and your callbacks:

```swift
// SwiftSignalKit/Source/Subscriber.swift
public final class Subscriber<T, E> {
    private var next: ((T) -> Void)!
    private var error: ((E) -> Void)!
    private var completed: (() -> Void)!

    private var lock = pthread_mutex_t()
    private var terminated = false
    internal var disposable: Disposable?

    public func putNext(_ next: T) {
        var action: ((T) -> Void)! = nil
        pthread_mutex_lock(&self.lock)
        if !self.terminated {
            action = self.next
        }
        pthread_mutex_unlock(&self.lock)

        if action != nil {
            action(next)
        }
    }

    public func putError(_ error: E) {
        var action: ((E) -> Void)! = nil
        var disposeDisposable: Disposable?

        pthread_mutex_lock(&self.lock)
        if !self.terminated {
            action = self.error
            self.next = nil
            self.error = nil
            self.completed = nil
            self.terminated = true
            disposeDisposable = self.disposable
            self.disposable = nil
        }
        pthread_mutex_unlock(&self.lock)

        if action != nil { action(error) }
        if let d = disposeDisposable { d.dispose() }
    }

    public func putCompletion() {
        // Same pattern as putError — terminates and disposes
    }
}
```

Key guarantees:

1. **Thread safety**: Every method acquires a `pthread_mutex_t` before checking state. Values can be pushed from any thread.
2. **Terminal state**: Once `putError` or `putCompletion` is called, the subscriber is `terminated`. Further calls to `putNext` are silently dropped.
3. **Automatic cleanup**: On termination, all callback references are nilled out (preventing retain cycles) and the inner disposable is disposed.
4. **One terminal event**: You can't get both an error and a completion — whichever arrives first wins.

The use of `pthread_mutex_t` instead of `os_unfair_lock` or GCD is deliberate: it's the most portable and predictable mutex on Apple platforms, with minimal overhead.

## The Disposable Hierarchy

Resource cleanup is managed through five disposable types:

### EmptyDisposable

```swift
final class _EmptyDisposable: Disposable {
    func dispose() { }
}

public let EmptyDisposable: Disposable = _EmptyDisposable()
```

A singleton that does nothing. Used when a signal has no resources to clean up (like `.single()` or `.complete()`).

### ActionDisposable

```swift
public final class ActionDisposable: Disposable {
    private var lock = pthread_mutex_t()
    private var action: (() -> Void)?

    public init(action: @escaping() -> Void) {
        self.action = action
        pthread_mutex_init(&self.lock, nil)
    }

    public func dispose() {
        let disposeAction: (() -> Void)?
        pthread_mutex_lock(&self.lock)
        disposeAction = self.action
        self.action = nil
        pthread_mutex_unlock(&self.lock)

        disposeAction?()
    }
}
```

Runs a closure exactly once on dispose. The action is nilled out after execution, so calling `dispose()` multiple times is safe. This is the most commonly used disposable — whenever a signal needs custom cleanup, it returns an `ActionDisposable`.

### MetaDisposable — The Workhorse

```swift
public final class MetaDisposable: Disposable {
    private var lock = pthread_mutex_t()
    private var disposed = false
    private var disposable: Disposable! = nil

    public func set(_ disposable: Disposable?) {
        var previousDisposable: Disposable! = nil
        var disposeImmediately = false

        pthread_mutex_lock(&self.lock)
        disposeImmediately = self.disposed
        if !disposeImmediately {
            previousDisposable = self.disposable
            self.disposable = disposable
        }
        pthread_mutex_unlock(&self.lock)

        if previousDisposable != nil {
            previousDisposable.dispose()
        }
        if disposeImmediately {
            disposable?.dispose()
        }
    }

    public func dispose() {
        var disposable: Disposable! = nil
        pthread_mutex_lock(&self.lock)
        if !self.disposed {
            self.disposed = true
            disposable = self.disposable
            self.disposable = nil
        }
        pthread_mutex_unlock(&self.lock)

        if disposable != nil { disposable.dispose() }
    }
}
```

`MetaDisposable` is a **swappable container** for a single disposable. When you call `set()`:
- The *previous* disposable is immediately disposed
- The new one takes its place
- If the MetaDisposable is already disposed, the new one is disposed immediately

This is the standard pattern for "latest subscription only" — when you only care about the most recent result. You'll see it *everywhere* in the codebase:

```swift
private let fetchDisposable = MetaDisposable()

func loadData(for id: String) {
    // Automatically cancels the previous fetch
    fetchDisposable.set(
        api.fetchData(id: id).start(next: { data in
            self.update(data)
        })
    )
}

deinit {
    fetchDisposable.dispose()
}
```

### DisposableSet

```swift
public final class DisposableSet: Disposable {
    private var disposables: [Disposable] = []
    // ...
    public func add(_ disposable: Disposable) { /* ... */ }
    public func dispose() { /* disposes all */ }
}
```

Collects multiple disposables and disposes them all at once. Used when a signal creates multiple child subscriptions that all need to be cleaned up together.

### DisposableDict\<T: Hashable\>

```swift
public final class DisposableDict<T: Hashable>: Disposable {
    private var disposables: [T: Disposable] = [:]
    // ...
    public func set(_ disposable: Disposable?, forKey key: T) { /* ... */ }
}
```

A keyed version of `DisposableSet`. Each key maps to one disposable, and setting a new disposable for an existing key disposes the old one. This is used for managing subscriptions indexed by ID — for example, tracking download progress for multiple files simultaneously.

## Promise\<T\> and ValuePromise\<T\>: Mutable Signal Sources

### Promise\<T\>

`Promise` is a **mutable signal source** that multicasts a signal to multiple subscribers:

```swift
public final class Promise<T> {
    private var value: T?
    private let disposable = MetaDisposable()
    private let subscribers = Bag<(T) -> Void>()

    public func set(_ signal: Signal<T, NoError>) {
        self.disposable.set(signal.start(next: { [weak self] next in
            if let strongSelf = self {
                strongSelf.value = next
                let subscribers = strongSelf.subscribers.copyItems()
                for subscriber in subscribers {
                    subscriber(next)
                }
            }
        }))
    }

    public func get() -> Signal<T, NoError> {
        return Signal { subscriber in
            let currentValue = self.value
            let index = self.subscribers.add({ next in
                subscriber.putNext(next)
            })

            if let currentValue = currentValue {
                subscriber.putNext(currentValue)
            }

            return ActionDisposable {
                self.subscribers.remove(index)
            }
        }
    }
}
```

How it works:
1. You call `promise.set(someSignal)` — this subscribes to the signal and caches each value it produces
2. Subscribers via `promise.get()` receive the current cached value immediately (if any), then all future values
3. Calling `set()` again replaces the source signal — the `MetaDisposable` automatically cancels the previous one

`Promise` is used throughout the codebase for deferred values. The `AppDelegate` uses it for contexts that aren't available at launch:

```swift
private let sharedContextPromise = Promise<SharedApplicationContext>()
private let context = Promise<AuthorizedApplicationContext?>()

// Later, when the context is ready:
sharedContextPromise.set(.single(sharedContext))
```

### ValuePromise\<T: Equatable\>

`ValuePromise` is a simpler variant for **imperative value updates** with optional deduplication:

```swift
public final class ValuePromise<T: Equatable> {
    private var value: T?
    private let subscribers = Bag<(T) -> Void>()
    public let ignoreRepeated: Bool

    public func set(_ value: T) {
        let subscribers: [(T) -> Void]
        if !self.ignoreRepeated || self.value != value {
            self.value = value
            subscribers = self.subscribers.copyItems()
        } else {
            subscribers = []
        }
        for subscriber in subscribers {
            subscriber(value)
        }
    }

    public func get() -> Signal<T, NoError> {
        return Signal { subscriber in
            let currentValue = self.value
            let index = self.subscribers.add({ next in
                subscriber.putNext(next)
            })
            if let currentValue = currentValue {
                subscriber.putNext(currentValue)
            }
            return ActionDisposable {
                self.subscribers.remove(index)
            }
        }
    }
}
```

The key difference from `Promise`:
- **`set()` takes a plain value**, not a signal — you imperatively push values
- **`ignoreRepeated`**: When `true`, setting the same value twice (by `Equatable` comparison) doesn't notify subscribers. This is huge for avoiding redundant UI updates.

The `AppDelegate` uses this for foreground state:

```swift
private let isInForegroundPromise = ValuePromise<Bool>(false, ignoreRepeated: true)

// In applicationDidBecomeActive:
isInForegroundPromise.set(true)

// In applicationDidEnterBackground:
isInForegroundPromise.set(false)
```

Because `ignoreRepeated` is `true`, rapidly toggling between the same state (which iOS can do) doesn't trigger redundant updates.

## ValuePipe\<T\>: Fire-and-Forget Event Bus

`ValuePipe` is the simplest signal source — a broadcast event bus with no caching:

```swift
public final class ValuePipe<T> {
    private let subscribers = Atomic(value: Bag<(T) -> Void>())

    public func signal() -> Signal<T, NoError> {
        return Signal { [weak self] subscriber in
            if let strongSelf = self {
                let index = strongSelf.subscribers.with { value -> Bag<T>.Index in
                    return value.add { next in
                        subscriber.putNext(next)
                    }
                }
                return ActionDisposable { [weak strongSelf] in
                    strongSelf?.subscribers.with { value -> Void in
                        value.remove(index)
                    }
                }
            } else {
                return EmptyDisposable
            }
        }
    }

    public func putNext(_ next: T) {
        let items = self.subscribers.with { $0.copyItems() }
        for f in items { f(next) }
    }
}
```

Unlike `Promise` or `ValuePromise`, there's **no cached value** — if you subscribe after a value was pushed, you don't get it. This is perfect for events that are only meaningful in real-time: button taps, incoming WebSocket messages, notification events.

## Bag\<T\>: The Subscriber Collection

`Bag` is a custom collection that `Promise`, `ValuePromise`, and `ValuePipe` all use internally to store their subscriber callbacks:

```swift
public final class Bag<T> {
    public typealias Index = Int
    private var nextIndex: Index = 0
    private var items: [T] = []
    private var itemKeys: [Index] = []

    public func add(_ item: T) -> Index {
        let key = self.nextIndex
        self.nextIndex += 1
        self.items.append(item)
        self.itemKeys.append(key)
        return key
    }

    public func remove(_ index: Index) {
        for (i, key) in self.itemKeys.enumerated() {
            if key == index {
                self.items.remove(at: i)
                self.itemKeys.remove(at: i)
                break
            }
        }
    }

    public func copyItems() -> [T] {
        return self.items
    }
}
```

Why not just use an `Array` or `Dictionary`? Because `Bag` provides **stable indices** — adding an item returns an `Index` that remains valid even after other items are removed. This is critical for signal subscribers: when you subscribe, you get an index; when you unsubscribe (in the `ActionDisposable`), you remove by that index. Other subscribers' indices aren't affected.

## Queue: Thread Management

```swift
public final class Queue {
    private let nativeQueue: DispatchQueue
    private var specific = NSObject()
    private let specialIsMainQueue: Bool

    public func isCurrent() -> Bool {
        if DispatchQueue.getSpecific(key: QueueSpecificKey) === self.specific {
            return true
        } else if self.specialIsMainQueue && Thread.isMainThread {
            return true
        }
        return false
    }

    public func async(_ f: @escaping () -> Void) {
        if self.isCurrent() {
            f()  // Execute synchronously if already on this queue
        } else {
            self.nativeQueue.async(execute: f)
        }
    }
}
```

The key behavior: **`async()` is synchronous if you're already on the target queue**. This avoids unnecessary dispatch hops and is why Telegram's reactive pipelines feel fast — when you `deliverOnMainQueue` and you're already on the main queue, there's zero overhead.

`isCurrent()` uses `DispatchQueue.setSpecific` for custom serial queues and `Thread.isMainThread` for the main queue — the most reliable way to detect the current queue on Apple platforms.

Three global singletons are pre-created:

```swift
private let globalMainQueue = Queue(queue: DispatchQueue.main, specialIsMainQueue: true)
private let globalDefaultQueue = Queue(queue: DispatchQueue.global(qos: .default))
private let globalBackgroundQueue = Queue(queue: DispatchQueue.global(qos: .background))
```

## QueueLocalObject\<T\>: Thread-Confined State

```swift
public final class QueueLocalObject<T: AnyObject> {
    public let queue: Queue
    private var valueRef: Unmanaged<T>?

    public init(queue: Queue, generate: @escaping () -> T) {
        self.queue = queue
        self.queue.async {
            let value = generate()
            self.valueRef = Unmanaged.passRetained(value)
        }
    }

    deinit {
        let valueRef = self.valueRef
        self.queue.async {
            valueRef?.release()
        }
    }

    public func with(_ f: @escaping (T) -> Void) {
        self.queue.async {
            if let valueRef = self.valueRef {
                f(valueRef.takeUnretainedValue())
            }
        }
    }
}
```

`QueueLocalObject` ensures an object is **created, accessed, and destroyed on a specific queue**. It uses `Unmanaged` for manual reference counting to guarantee the release happens on the correct queue. This is used for thread-confined state in TelegramCore — for example, the `AccountStateManager` keeps its mutable state in a `QueueLocalObject` to prevent data races.

## Atomic\<T\>: Thread-Safe Value Box

```swift
public final class Atomic<T> {
    private var lock: pthread_mutex_t
    private var value: T

    public func with<R>(_ f: (T) -> R) -> R {
        pthread_mutex_lock(&self.lock)
        let result = f(self.value)
        pthread_mutex_unlock(&self.lock)
        return result
    }

    public func modify(_ f: (T) -> T) -> T {
        pthread_mutex_lock(&self.lock)
        let result = f(self.value)
        self.value = result
        pthread_mutex_unlock(&self.lock)
        return result
    }

    public func swap(_ value: T) -> T {
        pthread_mutex_lock(&self.lock)
        let previous = self.value
        self.value = value
        pthread_mutex_unlock(&self.lock)
        return previous
    }
}
```

A simple mutex-protected box. `with` reads the value, `modify` transforms it in-place, `swap` replaces it and returns the old value. All three are atomic operations.

## Key Operators

### Mapping Operators

```swift
// map: transform values
public func map<T, E, R>(_ f: @escaping(T) -> R) -> (Signal<T, E>) -> Signal<R, E>

// filter: pass only matching values
public func filter<T, E>(_ f: @escaping(T) -> Bool) -> (Signal<T, E>) -> Signal<T, E>

// flatMap: map optional values (nil passes through as nil)
public func flatMap<T, E, R>(_ f: @escaping (T) -> R) -> (Signal<T?, E>) -> Signal<R?, E>

// distinctUntilChanged: suppress consecutive duplicates
public func distinctUntilChanged<T: Equatable, E>(_ signal: Signal<T, E>) -> Signal<T, E>
```

### Meta Operators (Signal of Signals)

The most powerful operators handle `Signal<Signal<T, E>, E>`:

```swift
// switchToLatest: subscribe to the latest inner signal, cancel the previous
public func switchToLatest<T, E>(_ signal: Signal<Signal<T, E>, E>) -> Signal<T, E>

// mapToSignal: map each value to a signal, then switchToLatest
public func mapToSignal<T, R, E>(_ f: @escaping(T) -> Signal<R, E>) -> (Signal<T, E>) -> Signal<R, E>

// queue: subscribe to inner signals sequentially (wait for each to complete)
public func queue<T, E>(_ signal: Signal<Signal<T, E>, E>) -> Signal<T, E>

// throttled: like queue but drops intermediate signals, only keeps the latest
public func throttled<T, E>(_ signal: Signal<Signal<T, E>, E>) -> Signal<T, E>

// then: concatenate two signals (wait for first to complete, then subscribe to second)
public func then<T, E>(_ nextSignal: Signal<T, E>) -> (Signal<T, E>) -> Signal<T, E>
```

`mapToSignal` is the **most used operator** in the codebase — it's the equivalent of RxSwift's `flatMapLatest` or Combine's `flatMap` with `.latest` strategy. The implementation is elegant:

```swift
public func mapToSignal<T, R, E>(_ f: @escaping(T) -> Signal<R, E>) -> (Signal<T, E>) -> Signal<R, E> {
    return { signal -> Signal<R, E> in
        return Signal<Signal<R, E>, E> { subscriber in
            return signal.start(next: { next in
                subscriber.putNext(f(next))
            }, error: { error in
                subscriber.putError(error)
            }, completed: {
                subscriber.putCompletion()
            })
        } |> switchToLatest
    }
}
```

It maps each value to a signal, creating a `Signal<Signal<R, E>, E>`, then pipes that into `switchToLatest`.

### Dispatch Operators

```swift
// deliverOn: deliver events on a specific queue
public func deliverOn<T, E>(_ queue: Queue) -> (Signal<T, E>) -> Signal<T, E>

// deliverOnMainQueue: shorthand for deliverOn(Queue.mainQueue())
public func deliverOnMainQueue<T, E>(_ signal: Signal<T, E>) -> Signal<T, E>

// runOn: run the generator on a specific queue (affects subscription, not delivery)
public func runOn<T, E>(_ queue: Queue) -> (Signal<T, E>) -> Signal<T, E>
```

The critical distinction: `deliverOn` dispatches the *output* to a queue. `runOn` dispatches the *subscription* (generator execution) to a queue.

### Take Operators

```swift
public func take<T, E>(_ count: Int) -> (Signal<T, E>) -> Signal<T, E>
public func takeLast<T, E>(_ count: Int) -> (Signal<T, E>) -> Signal<T, E>
```

`take(1)` is extremely common — it turns a long-lived signal into a one-shot:

```swift
// Get the current presentation data (one value, then done)
context.sharedContext.presentationData |> take(1) |> deliverOnMainQueue
```

## The Complete Type Map

Here's every type in SwiftSignalKit and its purpose:

| Type | Purpose | Equivalent in Combine |
|------|---------|----------------------|
| `Signal<T, E>` | Cold observable stream | `Publisher` |
| `Subscriber<T, E>` | Receives events from a signal | `Subscriber` |
| `Disposable` | Cancellation token | `Cancellable` |
| `EmptyDisposable` | No-op disposal | `AnyCancellable {}` |
| `ActionDisposable` | Dispose with a closure | `AnyCancellable { ... }` |
| `MetaDisposable` | Swappable single disposable | — (no direct equivalent) |
| `DisposableSet` | Collection of disposables | `Set<AnyCancellable>` |
| `DisposableDict<T>` | Keyed disposables | — |
| `StrictDisposable` | Leak-detecting wrapper | — |
| `Promise<T>` | Mutable signal source | `CurrentValueSubject` (closest) |
| `ValuePromise<T>` | Imperative value with dedup | `CurrentValueSubject` + `removeDuplicates` |
| `ValuePipe<T>` | Event bus (no caching) | `PassthroughSubject` |
| `Queue` | Serial dispatch queue wrapper | `DispatchQueue` |
| `QueueLocalObject<T>` | Thread-confined object | — |
| `Atomic<T>` | Mutex-protected value | — |
| `Bag<T>` | Indexed subscriber collection | — |
| `Lock` | Mutex wrapper | `NSLock` |

## What's Next

In the [next post](/posts/04-ssignal-kit-objc/), we'll look at `SSignalKit` — the Objective-C counterpart that powers the MTProto networking layer, and how the two frameworks interoperate. Then in [Post 5](/posts/05-reactive-patterns/), we'll see how these primitives are used idiomatically throughout the Telegram codebase with real examples.

---

*This post covers all 26 files in `submodules/SSignalKit/SwiftSignalKit/Source/`*
