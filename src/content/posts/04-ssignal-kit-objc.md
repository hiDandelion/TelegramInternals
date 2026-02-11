---
title: "SSignalKit: The Objective-C Reactive Layer"
description: "A deep dive into the Objective-C twin of SwiftSignalKit — the reactive framework powering LegacyComponents, media pipelines, and the bridge between ObjC and Swift in Telegram iOS."
published: 2026-02-08
tags:
  - Reactive Foundation
toc: true
lang: en
abbrlink: 04-ssignal-kit-objc
pin: 22
---

In the [previous post](/posts/03-swift-signal-kit/), we dissected SwiftSignalKit — the Swift reactive framework that powers every async operation in Telegram iOS. But Telegram iOS predates Swift. Many core modules — the camera, media editor, legacy gallery, and dozens of UI components — are written in Objective-C. These modules need the same reactive primitives.

That's where **SSignalKit** comes in: a parallel reactive framework written in pure Objective-C, living in the same repository under `submodules/SSignalKit/SSignalKit/Source/SSignalKit/`. It's not a wrapper around the Swift version — it's a standalone implementation with its own design choices, optimized for ObjC's memory model and block-based APIs.

## Why Two Frameworks?

This is the most natural question. Why maintain two implementations of the same concept?

1. **Historical**: Telegram iOS was Objective-C first. SSignalKit existed before the Swift rewrite began. The `LegacyComponents` module alone has 200+ ObjC files that rely on `SSignal`.

2. **Module boundaries**: Bazel enforces strict module boundaries. A pure ObjC target like `LegacyComponents` cannot import SwiftSignalKit (a Swift module). It needs an ObjC-native reactive library.

3. **Performance**: Bridging between ObjC and Swift at the signal level would add overhead on every `putNext` call — boxing, unboxing, retain/release at the boundary. Two native implementations avoid this entirely.

4. **API fit**: ObjC blocks have different ergonomics than Swift closures. SSignalKit's API is designed for ObjC patterns — `id`-typed values, category-based operator extension, no generics.

The two frameworks are not interoperable — you don't bridge an `SSignal` into a `Signal<T, E>`. Instead, the bridge layer (`LegacyUI`, `LegacyMediaPickerUI`) subscribes to one and feeds values into the other at the boundary.

## SSignal: The Core Signal Class

Here's the complete header:

```objc
// SSignalKit/Source/SSignalKit/SSignal.h

@interface SSignal : NSObject
{
@public
    id<SDisposable> _Nullable (^ _Nonnull _generator)(SSubscriber * _Nonnull);
}

- (instancetype _Nonnull)initWithGenerator:
    (id<SDisposable> _Nullable (^ _Nonnull)(SSubscriber * _Nonnull))generator;

- (id<SDisposable> _Nullable)startWithNext:(void (^ _Nullable)(id _Nullable next))next
                                     error:(void (^ _Nullable)(id _Nullable error))error
                                 completed:(void (^ _Nullable)())completed;

- (id<SDisposable> _Nullable)startStrictWithNext:(void (^ _Nullable)(id _Nullable next))next
                                           error:(void (^ _Nullable)(id _Nullable error))error
                                       completed:(void (^ _Nullable)())completed
                                            file:(const char * _Nonnull)file
                                            line:(int)line;
@end
```

Compare this to the Swift version:

| | SwiftSignalKit | SSignalKit |
|---|---|---|
| **Value type** | Generic `T` | `id` (any object) |
| **Error type** | Generic `E` | `id` (any object) |
| **Generator** | `(Subscriber<T, E>) -> Disposable` | `(SSubscriber *) -> id<SDisposable>` |
| **Type safety** | Compile-time via generics | Runtime only |
| **Operator style** | Free functions + `\|>` pipe | Categories (method chaining) |

The fundamental architecture is identical: a signal is a generator block that receives a subscriber and returns a disposable. The critical difference is that ObjC has no generics for this purpose, so every value is `id`. This means you lose compile-time type checking but gain zero-cost ObjC block interop.

### The Implementation

```objc
// SSignal.m

@implementation SSignal

- (instancetype)initWithGenerator:(id<SDisposable> (^)(SSubscriber *))generator {
    self = [super init];
    if (self != nil) {
        _generator = [generator copy];
    }
    return self;
}

- (id<SDisposable>)startWithNext:(void (^)(id next))next
                           error:(void (^)(id error))error
                       completed:(void (^)())completed {
    SSubscriber *subscriber = [[SSubscriber alloc] initWithNext:next error:error completed:completed];
    id<SDisposable> disposable = _generator(subscriber);
    [subscriber _assignDisposable:disposable];
    return [[SSubscriberDisposable alloc] initWithSubscriber:subscriber disposable:disposable];
}
```

The `start` flow is identical to the Swift version:
1. Create a subscriber with the three handler blocks
2. Run the generator, passing the subscriber
3. Assign the returned disposable to the subscriber (so terminal events can clean up)
4. Wrap everything in `SSubscriberDisposable` for external cancellation

### SSubscriberDisposable: Breaking Retain Cycles

Notice the return type isn't the subscriber or the generator's disposable directly. It's `SSubscriberDisposable`, a private class that serves a critical role:

```objc
// SSignal.m (private class)

@interface SSubscriberDisposable : NSObject <SDisposable>
{
    __weak SSubscriber *_subscriber;  // WEAK reference
    id<SDisposable> _disposable;
    pthread_mutex_t _lock;
}

- (void)dispose {
    SSubscriber *subscriber = nil;
    id<SDisposable> disposeItem = nil;
    pthread_mutex_lock(&_lock);
    disposeItem = _disposable;
    _disposable = nil;
    subscriber = _subscriber;
    _subscriber = nil;
    pthread_mutex_unlock(&_lock);

    [disposeItem dispose];
    [subscriber _markTerminatedWithoutDisposal];
}
```

The `__weak` reference to the subscriber is deliberate. When you dispose externally:
1. The disposable is nilled (breaking the retain from generator → disposable → resources)
2. The subscriber is marked terminated (so no more events flow through)
3. The subscriber's blocks are released (breaking the retain from subscriber → closure → captured objects)

Without this weak reference, you'd have a retain cycle: `signal` → `generator block` → `subscriber` → `SSubscriberDisposable` → back to subscriber.

### SStrictDisposable: Leak Detection in DEBUG

The `startStrict` variant wraps the disposable in an `SStrictDisposable` that crashes in DEBUG if you forget to dispose:

```objc
// SSignal.m (private class)

@implementation SStrictDisposable

- (void)dealloc {
#if DEBUG
    pthread_mutex_lock(&_lock);
    if (!_isDisposed) {
        NSLog(@"Leaked disposable from %s:%d", _file, _line);
        assert(false);
    }
    pthread_mutex_unlock(&_lock);
    pthread_mutex_destroy(&_lock);
#endif
}

- (void)dispose {
#if DEBUG
    pthread_mutex_lock(&_lock);
    _isDisposed = true;
    pthread_mutex_unlock(&_lock);
#endif
    [_disposable dispose];
}
```

In production builds, `SStrictDisposable` is a zero-cost passthrough (the `#if DEBUG` blocks compile away). In debug, it records the file and line where the subscription was created and asserts on dealloc if `dispose` was never called. This catches the most common reactive programming bug: subscribing to a signal and forgetting to manage the disposable.

## SSubscriber: Thread-Safe Event Delivery

```objc
// SSubscriber.m

@interface SSubscriber ()
{
    @protected
    os_unfair_lock _lock;        // Fast spinlock for state
    bool _terminated;            // Terminal event flag
    id<SDisposable> _disposable; // Upstream cleanup
    SSubscriberBlocks *_blocks;  // Handler blocks (next/error/completed)
}
@end
```

A key design difference from the Swift version: SSubscriber uses `os_unfair_lock` while SwiftSignalKit's Subscriber uses `pthread_mutex_t`. Both are valid — `os_unfair_lock` is slightly faster (no fairness guarantee means less overhead) but can't be used recursively. Since SSubscriber never needs recursive locking (the lock protects a simple state check), the unfair lock is the better choice.

### putNext: The Hot Path

```objc
- (void)putNext:(id)next {
    SSubscriberBlocks *blocks = nil;

    os_unfair_lock_lock(&_lock);
    if (!_terminated) {
        blocks = _blocks;
    }
    os_unfair_lock_unlock(&_lock);

    if (blocks && blocks->_next) {
        blocks->_next(next);
    }
}
```

The pattern is: lock, check terminated, grab blocks reference, unlock, call block outside the lock. This is critical — the block callback happens outside the lock, so downstream operators can't deadlock against the subscriber's lock.

### putError and putCompletion: Terminal Events

```objc
- (void)putError:(id)error {
    bool shouldDispose = false;
    SSubscriberBlocks *blocks = nil;

    os_unfair_lock_lock(&_lock);
    if (!_terminated) {
        blocks = _blocks;
        _blocks = nil;          // Release blocks immediately
        shouldDispose = true;
        _terminated = true;
    }
    os_unfair_lock_unlock(&_lock);

    if (blocks && blocks->_error) {
        blocks->_error(error);
    }

    if (shouldDispose) {
        [self->_disposable dispose];
        self->_disposable = nil;
    }
}
```

Terminal events (`putError`, `putCompletion`) do three things atomically:
1. Set `_terminated = true` so no future events pass through
2. Nil out `_blocks` to release all closure references immediately
3. Dispose the upstream disposable to cancel any pending work

The pattern of nilling `_blocks` under the lock but calling `blocks->_error(error)` after releasing the lock is essential. If the error handler itself triggers a re-entrant `putNext`, the `_terminated` flag ensures it's dropped.

### The SSubscriberBlocks Separation

Why are the blocks stored in a separate `SSubscriberBlocks` object instead of directly on the subscriber?

```objc
@interface SSubscriberBlocks : NSObject {
    @public
    void (^_next)(id);
    void (^_error)(id);
    void (^_completed)();
}
@end
```

This is a memory management optimization. By grouping all three blocks into one object, the subscriber can release all of them in a single atomic operation (`_blocks = nil`). Without this, you'd need to nil three separate ivars under the lock, or risk one block being called after another was already released. The `@public` ivars avoid method dispatch overhead on the hot path.

## The Disposable Hierarchy

### SDisposable Protocol

```objc
// SDisposable.h
@protocol SDisposable <NSObject>
- (void)dispose;
@end
```

One method. That's the entire protocol. The Swift version is identical.

### SBlockDisposable: Closure-Based Cleanup

```objc
// SBlockDisposable.m

@interface SBlockDisposable () {
    void (^_action)();
    pthread_mutex_t _lock;
}

- (void)dispose {
    void (^disposeAction)() = nil;

    pthread_mutex_lock(&_lock);
    disposeAction = _action;
    _action = nil;                  // Ensure one-shot execution
    pthread_mutex_unlock(&_lock);

    if (disposeAction) {
        disposeAction();
    }
}
```

The nil-then-execute pattern guarantees the cleanup block runs exactly once, even if `dispose` is called from multiple threads. Note the lock is `pthread_mutex_t` here, not `os_unfair_lock` — this is intentional because `SBlockDisposable` is used everywhere, and the mutex's slightly more predictable behavior under high contention makes it a better default.

An interesting detail: the `dealloc` grabs the lock and nils the action but **doesn't execute it**:

```objc
- (void)dealloc {
    void (^freeAction)() = nil;
    pthread_mutex_lock(&_lock);
    freeAction = _action;
    _action = nil;
    pthread_mutex_unlock(&_lock);

    if (freeAction) {
        // Intentionally empty — just release the block
    }

    pthread_mutex_destroy(&_lock);
}
```

If the disposable is deallocated without being disposed, the action block is silently released. This is different from `SStrictDisposable` which would assert. The philosophy: `SBlockDisposable` is often used internally where the framework guarantees cleanup, so crashing on missed disposal would be noise.

### SMetaDisposable: The Swappable Container

```objc
// SMetaDisposable.m

- (void)setDisposable:(id<SDisposable>)disposable {
    id<SDisposable> previousDisposable = nil;
    bool disposeImmediately = false;

    pthread_mutex_lock(&_lock);
    disposeImmediately = _disposed;
    if (!disposeImmediately) {
        previousDisposable = _disposable;
        _disposable = disposable;
    }
    pthread_mutex_unlock(&_lock);

    if (previousDisposable) {
        [previousDisposable dispose];
    }

    if (disposeImmediately) {
        [disposable dispose];
    }
}
```

`SMetaDisposable` holds one disposable at a time. When you set a new one:
- If the container is already disposed → the new disposable is disposed immediately
- Otherwise → the previous disposable is disposed, the new one takes its place

This is the cornerstone pattern for subscriptions that need to be "replaced." For example, when a user types in a search field, each keystroke creates a new search signal. A `SMetaDisposable` holds the current search subscription and automatically cancels the previous one.

### SDisposableSet: Parallel Cleanup

```objc
// SDisposableSet.m

- (void)add:(id<SDisposable>)disposable {
    bool disposeImmediately = false;

    pthread_mutex_lock(&_lock);
    if (_disposed) {
        disposeImmediately = true;
    } else {
        [_disposables addObject:disposable];
    }
    pthread_mutex_unlock(&_lock);

    if (disposeImmediately) {
        [disposable dispose];
    }
}

- (void)dispose {
    NSArray<id<SDisposable>> *disposables = nil;
    pthread_mutex_lock(&_lock);
    if (!_disposed) {
        _disposed = true;
        disposables = _disposables;
        _disposables = nil;
    }
    pthread_mutex_unlock(&_lock);

    if (disposables) {
        for (id<SDisposable> disposable in disposables) {
            [disposable dispose];
        }
    }
}
```

`SDisposableSet` collects multiple disposables and disposes them all at once. If you add a disposable after the set is already disposed, it's disposed immediately — no leaks. The `remove:` method uses identity comparison (`==`) not equality, since disposables don't conform to `isEqual:`.

## SQueue: Queue Identity Optimization

```objc
// SQueue.m

static const void *SQueueSpecificKey = &SQueueSpecificKey;

- (instancetype)init {
    dispatch_queue_t queue = dispatch_queue_create(NULL, NULL);
    dispatch_queue_set_specific(queue, SQueueSpecificKey, (__bridge void *)self, NULL);
    return [self initWithNativeQueue:queue queueSpecific:(__bridge void *)self];
}

- (void)dispatch:(dispatch_block_t)block {
    if (_queueSpecific != NULL &&
        dispatch_get_specific(SQueueSpecificKey) == _queueSpecific)
        block();              // Already on this queue — run synchronously
    else if (_specialIsMainQueue && [NSThread isMainThread])
        block();              // Main queue shortcut
    else
        dispatch_async(_queue, block);
}
```

This is the same optimization as the Swift `Queue` class, but the implementation reveals the GCD mechanism more clearly:

1. **`dispatch_queue_set_specific`** stores a pointer (the `SQueue` instance itself) as queue-local data associated with `SQueueSpecificKey`.
2. **`dispatch_get_specific`** reads that pointer from the currently executing queue.
3. If they match, we're already on this queue — execute synchronously to avoid deadlock and reduce overhead.

The `_specialIsMainQueue` flag adds a second fast path. The main queue can't use `dispatch_get_specific` reliably (it might be running on a thread that also has a custom queue specific set), so it falls back to `[NSThread isMainThread]`.

Factory methods provide convenient singletons:

```objc
+ (SQueue *)mainQueue {
    static SQueue *queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        queue = [[SQueue alloc] initWithNativeQueue:dispatch_get_main_queue()
                                      queueSpecific:NULL];
        queue->_specialIsMainQueue = true;
    });
    return queue;
}
```

## SAtomic: Thread-Safe Value Container

```objc
// SAtomic.m

- (id)modify:(id (^)(id))f {
    id newValue = nil;
    pthread_mutex_lock(&_lock);
    newValue = f(_value);
    _value = newValue;
    pthread_mutex_unlock(&_lock);
    return newValue;
}

- (id)with:(id (^)(id))f {
    id result = nil;
    pthread_mutex_lock(&_lock);
    result = f(_value);
    pthread_mutex_unlock(&_lock);
    return result;
}
```

`SAtomic` provides two key operations:
- **`modify:`** — transform the value and store the result (like a compare-and-swap)
- **`with:`** — read the value through a closure without modifying it (like a synchronized read)

Both execute the closure under the lock, which means the closure sees a consistent snapshot. This is used extensively in operators that need to track state across emissions.

The ObjC version adds a feature the Swift version lacks: **recursive mutex support**.

```objc
- (instancetype)initWithValue:(id)value recursive:(bool)recursive {
    self = [super init];
    if (self != nil) {
        _isRecursive = recursive;
        if (recursive) {
            pthread_mutexattr_init(&_attr);
            pthread_mutexattr_settype(&_attr, PTHREAD_MUTEX_RECURSIVE);
            pthread_mutex_init(&_lock, &_attr);
        } else {
            pthread_mutex_init(&_lock, NULL);
        }
        _value = value;
    }
    return self;
}
```

A recursive mutex allows the same thread to lock it multiple times. This is needed when an `SAtomic`'s `modify` callback triggers code that itself calls `with` on the same `SAtomic` — a pattern that can occur in complex ObjC code with delegate callbacks.

## SBag: Unordered Collection with Stable Keys

```objc
// SBag.m

@interface SBag ()
{
    NSInteger _nextKey;
    NSMutableArray *_items;
    NSMutableArray *_itemKeys;
}

- (NSInteger)addItem:(id)item {
    if (item == nil) return -1;

    NSInteger key = _nextKey;
    [_items addObject:item];
    [_itemKeys addObject:@(key)];
    _nextKey++;
    return key;
}

- (void)removeItem:(NSInteger)key {
    NSUInteger index = 0;
    for (NSNumber *itemKey in _itemKeys) {
        if ([itemKey integerValue] == key) {
            [_items removeObjectAtIndex:index];
            [_itemKeys removeObjectAtIndex:index];
            break;
        }
        index++;
    }
}
```

The ObjC `SBag` is simpler than its Swift counterpart — just two parallel arrays instead of a dictionary. This is because the ObjC version doesn't need to handle `SparseBag` or `CounterBag` variants. The key guarantee is the same: `addItem` returns a monotonically increasing integer key that uniquely identifies the item for later removal. Items are iterated in insertion order.

`SBag` is **not thread-safe** by itself. It's always used inside an `SAtomic` or protected by an `os_unfair_lock`, which is why thread safety is delegated to the container.

## SVariable: The Reactive State Holder

`SVariable` is the ObjC equivalent of Swift's `Promise` — a reactive variable that caches the latest value and replays it to new subscribers:

```objc
// SVariable.m

@interface SVariable () {
    os_unfair_lock _lock;
    id _value;
    bool _hasValue;
    SBag *_subscribers;
    SMetaDisposable *_disposable;
}

- (SSignal *)signal {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        os_unfair_lock_lock(&self->_lock);
        id currentValue = _value;
        bool hasValue = _hasValue;
        NSInteger index = [self->_subscribers addItem:[^(id value) {
            [subscriber putNext:value];
        } copy]];
        os_unfair_lock_unlock(&self->_lock);

        if (hasValue) {
            [subscriber putNext:currentValue];    // Replay cached value
        }

        return [[SBlockDisposable alloc] initWithBlock:^{
            os_unfair_lock_lock(&self->_lock);
            [self->_subscribers removeItem:index];
            os_unfair_lock_unlock(&self->_lock);
        }];
    }];
}

- (void)set:(SSignal *)signal {
    os_unfair_lock_lock(&_lock);
    _hasValue = false;                    // Reset cache
    os_unfair_lock_unlock(&_lock);

    __weak SVariable *weakSelf = self;
    [_disposable setDisposable:[signal startWithNext:^(id next) {
        __strong SVariable *strongSelf = weakSelf;
        if (strongSelf != nil) {
            NSArray *subscribers = nil;
            os_unfair_lock_lock(&strongSelf->_lock);
            strongSelf->_value = next;
            strongSelf->_hasValue = true;
            subscribers = [strongSelf->_subscribers copyItems];
            os_unfair_lock_unlock(&strongSelf->_lock);

            for (void (^subscriber)(id) in subscribers) {
                subscriber(next);
            }
        }
    }]];
}
```

The flow:
1. **Subscribe**: Lock, snapshot the current value, register a callback block in the `SBag`, unlock. If there was a cached value, emit it immediately.
2. **Set**: Provide a new source signal. The `SMetaDisposable` automatically cancels the previous source. Each value from the new source is cached and broadcast to all current subscribers.
3. **Unsubscribe**: The disposal block removes the callback from the `SBag`.

The `__weak`/`__strong` dance in `set:` prevents a retain cycle: the signal's `next` block captures `self` weakly, then promotes to strong only for the duration of the callback.

## Operators via Categories

In Swift, operators are free functions composed with `|>`. In Objective-C, operators are **categories** — extensions to the `SSignal` class.

### Mapping: map, filter, ignoreRepeated

```objc
// SSignal+Mapping.m

- (SSignal *)map:(id (^)(id))f {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        return [self startWithNext:^(id next) {
            [subscriber putNext:f(next)];
        } error:^(id error) {
            [subscriber putError:error];
        } completed:^{
            [subscriber putCompletion];
        }];
    }];
}

- (SSignal *)filter:(bool (^)(id))f {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        return [self startWithNext:^(id next) {
            if (f(next))
                [subscriber putNext:next];
        } error:^(id error) {
            [subscriber putError:error];
        } completed:^{
            [subscriber putCompletion];
        }];
    }];
}
```

The pattern is universal: create a new signal whose generator subscribes to `self`, transforms the values, and forwards them to the downstream subscriber. The returned disposable is the upstream subscription itself.

`ignoreRepeated` is more interesting — it needs state:

```objc
- (SSignal *)ignoreRepeated {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        SAtomic *state = [[SAtomic alloc] initWithValue:
            [[SSignalIgnoreRepeatedState alloc] init]];

        return [self startWithNext:^(id next) {
            bool shouldPassthrough = [[state with:^id(SSignalIgnoreRepeatedState *s) {
                if (!s.hasValue) {
                    s.hasValue = true;
                    s.value = next;
                    return @true;
                } else if ((s.value == nil && next == nil) ||
                           [(id<NSObject>)s.value isEqual:next]) {
                    return @false;
                }
                s.value = next;
                return @true;
            }] boolValue];

            if (shouldPassthrough) {
                [subscriber putNext:next];
            }
        } error:^(id error) {
            [subscriber putError:error];
        } completed:^{
            [subscriber putCompletion];
        }];
    }];
}
```

The `SAtomic` holds state across emissions. Each value is compared with `isEqual:` — since there are no generics, the implementation relies on the ObjC `isEqual:` protocol. Note the special handling for double-nil (both previous and current are `nil`).

### Meta Operators: switchToLatest, mapToSignal, queue, throttled

The meta operators handle signals of signals. The key abstraction is `SSignalQueueState`:

```objc
// SSignal+Meta.m

@interface SSignalQueueState : NSObject <SDisposable>
{
    os_unfair_lock _lock;
    bool _executingSignal;
    bool _terminated;

    id<SDisposable> _disposable;
    SMetaDisposable *_currentDisposable;
    SSubscriber *_subscriber;
    NSMutableArray *_queuedSignals;
    bool _queueMode;
    bool _throttleMode;
}
```

This single state class powers four different operators through two boolean flags:

| Operator | `_queueMode` | `_throttleMode` | Behavior |
|---|---|---|---|
| `switchToLatest` | `false` | `false` | Cancel previous, start latest |
| `queue` | `true` | `false` | Queue all, execute sequentially |
| `throttled` | `true` | `true` | Queue only the latest, drop older |

When `_queueMode` is false (switchToLatest), there's no queue — each new inner signal simply replaces the current one via the `SMetaDisposable`:

```objc
- (SSignal *)switchToLatest {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        SSignalQueueState *state = [[SSignalQueueState alloc]
            initWithSubscriber:subscriber queueMode:false throttleMode:false];

        [state beginWithDisposable:[self startWithNext:^(id next) {
            [state enqueueSignal:next];
        } error:^(id error) {
            [subscriber putError:error];
        } completed:^{
            [state beginCompletion];
        }]];

        return state;
    }];
}
```

And `mapToSignal` is simply map + switchToLatest:

```objc
- (SSignal *)mapToSignal:(SSignal *(^)(id))f {
    return [[self map:f] switchToLatest];
}
```

When `_queueMode` is true, signals are queued and executed sequentially. Each time an inner signal completes, `headCompleted` is called:

```objc
- (void)headCompleted {
    SSignal *nextSignal = nil;
    bool terminated = false;

    os_unfair_lock_lock(&_lock);
    _executingSignal = false;

    if (_queueMode) {
        if (_queuedSignals.count != 0) {
            nextSignal = _queuedSignals[0];
            [_queuedSignals removeObjectAtIndex:0];
            _executingSignal = true;
        } else {
            terminated = _terminated;
        }
    } else {
        terminated = _terminated;
    }
    os_unfair_lock_unlock(&_lock);

    if (terminated)
        [_subscriber putCompletion];
    else if (nextSignal != nil)
        // Start the next queued signal...
}
```

When `_throttleMode` is also true, queuing clears the array first — only the latest pending signal survives:

```objc
- (void)enqueueSignal:(SSignal *)signal {
    bool startSignal = false;
    os_unfair_lock_lock(&_lock);
    if (_queueMode && _executingSignal) {
        if (_throttleMode) {
            [_queuedSignals removeAllObjects];  // Drop all but latest
        }
        [_queuedSignals addObject:signal];
    } else {
        _executingSignal = true;
        startSignal = true;
    }
    os_unfair_lock_unlock(&_lock);
    // ...
}
```

### Dispatch: deliverOn, startOn

```objc
// SSignal+Dispatch.m

- (SSignal *)deliverOn:(SQueue *)queue {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        return [self startWithNext:^(id next) {
            [queue dispatch:^{
                [subscriber putNext:next];
            }];
        } error:^(id error) {
            [queue dispatch:^{
                [subscriber putError:error];
            }];
        } completed:^{
            [queue dispatch:^{
                [subscriber putCompletion];
            }];
        }];
    }];
}
```

`deliverOn` wraps every event emission in a queue dispatch. If the subscriber is already on the target queue, `SQueue`'s identity check makes this a synchronous call — no overhead.

`startOn` is different: it delays the subscription itself to a different queue:

```objc
- (SSignal *)startOn:(SQueue *)queue {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        __block bool isCancelled = false;
        SMetaDisposable *disposable = [[SMetaDisposable alloc] init];
        [disposable setDisposable:[[SBlockDisposable alloc] initWithBlock:^{
            isCancelled = true;
        }]];

        [queue dispatch:^{
            if (!isCancelled) {
                [disposable setDisposable:[self startWithNext:^(id next) {
                    [subscriber putNext:next];
                } error:^(id error) {
                    [subscriber putError:error];
                } completed:^{
                    [subscriber putCompletion];
                }]];
            }
        }];

        return disposable;
    }];
}
```

The `isCancelled` flag prevents a race: if the subscriber disposes before the queued block runs, we skip the subscription entirely.

### Combine: combineSignals, mergeSignals

```objc
// SSignal+Combine.m

+ (SSignal *)combineSignals:(NSArray *)signals withInitialStates:(NSArray *)initialStates {
    return [[SSignal alloc] initWithGenerator:^(SSubscriber *subscriber) {
        SAtomic *combineState = [[SAtomic alloc] initWithValue:
            [[SSignalCombineState alloc] initWithLatestValues:initialLatestValues
                                           completedStatuses:completedStatuses
                                                       error:false]];
        SDisposableSet *compositeDisposable = [[SDisposableSet alloc] init];

        NSUInteger index = 0;
        for (SSignal *signal in signals) {
            id<SDisposable> disposable = [signal startWithNext:^(id next) {
                SSignalCombineState *currentState = [combineState modify:^id(SSignalCombineState *state) {
                    NSMutableDictionary *latestValues = [state.latestValues mutableCopy];
                    latestValues[@(index)] = next;
                    return [[SSignalCombineState alloc] initWithLatestValues:latestValues ...];
                }];
                // Only emit when ALL signals have produced at least one value
                NSMutableArray *latestValues = [[NSMutableArray alloc] init];
                for (NSUInteger i = 0; i < count; i++) {
                    id value = currentState.latestValues[@(i)];
                    if (value == nil) {
                        latestValues = nil;
                        break;
                    }
                    latestValues[i] = value;
                }
                if (latestValues != nil)
                    [subscriber putNext:latestValues];
            } ...];
            [compositeDisposable add:disposable];
            index++;
        }
        return compositeDisposable;
    }];
}
```

The combine logic:
1. Subscribe to all signals in parallel
2. Track the latest value from each in a dictionary (keyed by index)
3. Only emit a combined array when every signal has produced at least one value
4. On each new value from any signal, emit the latest values from all signals

`mergeSignals` is simpler — it just forwards all values from all signals without combining:

```objc
+ (SSignal *)mergeSignals:(NSArray *)signals {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        SDisposableSet *disposables = [[SDisposableSet alloc] init];
        SAtomic *completedStates = [[SAtomic alloc] initWithValue:[[NSSet alloc] init]];

        NSInteger index = -1;
        for (SSignal *signal in signals) {
            index++;
            id<SDisposable> disposable = [signal startWithNext:^(id next) {
                [subscriber putNext:next];        // Forward directly
            } error:^(id error) {
                [subscriber putError:error];
            } completed:^{
                NSSet *set = [completedStates modify:^id(NSSet *set) {
                    return [set setByAddingObject:@(index)];
                }];
                if (set.count == count)
                    [subscriber putCompletion];   // Complete when ALL complete
            }];
            [disposables add:disposable];
        }
        return disposables;
    }];
}
```

### SPipe: The Hot Signal Subject

```objc
// SSignal+Pipe.m

@implementation SPipe

- (instancetype)initWithReplay:(bool)replay {
    self = [super init];
    if (self != nil) {
        SAtomic *subscribers = [[SAtomic alloc] initWithValue:[[SBag alloc] init]];
        SAtomic *replayState = replay
            ? [[SAtomic alloc] initWithValue:
                [[SPipeReplayState alloc] initWithReceivedValue:false recentValue:nil]]
            : nil;

        _signalProducer = [^SSignal * {
            return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
                __block NSUInteger index = 0;
                [subscribers with:^id(SBag *bag) {
                    index = [bag addItem:[^(id next) {
                        [subscriber putNext:next];
                    } copy]];
                    return nil;
                }];

                if (replay) {
                    [replayState with:^id(SPipeReplayState *state) {
                        if (state.hasReceivedValue)
                            [subscriber putNext:state.recentValue];
                        return nil;
                    }];
                }

                return [[SBlockDisposable alloc] initWithBlock:^{
                    [subscribers with:^id(SBag *bag) {
                        [bag removeItem:index];
                        return nil;
                    }];
                }];
            }];
        } copy];

        _sink = [^(id next) {
            NSArray *items = [subscribers with:^id(SBag *bag) {
                return [bag copyItems];
            }];
            for (void (^item)(id) in items) {
                item(next);
            }
            if (replay) {
                [replayState modify:^id(SPipeReplayState *state) {
                    return [[SPipeReplayState alloc]
                        initWithReceivedValue:true recentValue:next];
                }];
            }
        } copy];
    }
    return self;
}
```

`SPipe` is the ObjC equivalent of a "Subject" in RxSwift terminology:
- **`signalProducer`**: A block that creates a new `SSignal` connected to the pipe. Each subscriber registers a callback in the shared `SBag`.
- **`sink`**: A block that pushes values to all current subscribers by iterating the `SBag`.
- **`replay`**: When true, the most recent value is cached and replayed to new subscribers immediately.

Usage looks like:
```objc
SPipe *pipe = [[SPipe alloc] initWithReplay:true];
id<SDisposable> sub = [pipe.signalProducer() startWithNext:^(id next) {
    NSLog(@"Received: %@", next);
}];
pipe.sink(@"hello");  // Prints "Received: hello"
```

### Error Handling: catch, restart, retryIf

```objc
// SSignal+Catch.m

- (SSignal *)catch:(SSignal *(^)(id error))f {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        SDisposableSet *disposable = [[SDisposableSet alloc] init];

        [disposable add:[self startWithNext:^(id next) {
            [subscriber putNext:next];
        } error:^(id error) {
            SSignal *signal = f(error);          // Error → recovery signal
            [disposable add:[signal startWithNext:^(id next) {
                [subscriber putNext:next];
            } error:^(id error) {
                [subscriber putError:error];     // Recovery failed
            } completed:^{
                [subscriber putCompletion];
            }]];
        } completed:^{
            [subscriber putCompletion];
        }]];

        return disposable;
    }];
}
```

The `restart` operator is particularly elegant — it uses a recursive block helper to re-subscribe on completion:

```objc
static dispatch_block_t recursiveBlock(void (^block)(dispatch_block_t recurse)) {
    return ^{
        block(recursiveBlock(block));
    };
}

- (SSignal *)restart {
    return [[SSignal alloc] initWithGenerator:^id<SDisposable>(SSubscriber *subscriber) {
        SAtomic *shouldRestart = [[SAtomic alloc] initWithValue:@true];
        SMetaDisposable *currentDisposable = [[SMetaDisposable alloc] init];

        void (^start)() = recursiveBlock(^(dispatch_block_t recurse) {
            NSNumber *current = [shouldRestart with:^id(NSNumber *c) { return c; }];
            if ([current boolValue]) {
                id<SDisposable> disposable = [self startWithNext:^(id next) {
                    [subscriber putNext:next];
                } error:^(id error) {
                    [subscriber putError:error];
                } completed:^{
                    recurse();  // Re-subscribe on completion
                }];
                [currentDisposable setDisposable:disposable];
            }
        });

        start();

        return [[SBlockDisposable alloc] initWithBlock:^{
            [currentDisposable dispose];
            [shouldRestart modify:^id(id _) { return @false; }];
        }];
    }];
}
```

The `recursiveBlock` helper creates a Y-combinator-like pattern: the block receives its own recursive call as a parameter. Disposal sets `shouldRestart` to false, preventing infinite re-subscription.

## STimer: Dispatch Source Timers

```objc
// STimer.m

- (void)start {
    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, _nativeQueue);
    dispatch_source_set_timer(_timer,
        dispatch_time(DISPATCH_TIME_NOW, (int64_t)(_timeout * NSEC_PER_SEC)),
        _repeat ? (int64_t)(_timeout * NSEC_PER_SEC) : DISPATCH_TIME_FOREVER,
        0);

    dispatch_source_set_event_handler(_timer, ^{
        if (_completion)
            _completion(self);
        if (!_repeat)
            [self invalidate];
    });
    dispatch_resume(_timer);
}

- (void)fireAndInvalidate {
    if (_completion)
        _completion(self);
    [self invalidate];
}
```

`STimer` wraps `dispatch_source_t` for timer operations. It's used by the `throttleOn:delay:` operator to implement time-based throttling. The `fireAndInvalidate` method triggers the callback immediately and cancels the timer — useful when a throttle's upstream completes and you need to flush the last buffered value.

## Thread Safety Summary

SSignalKit uses a deliberate mix of locking primitives:

| Primitive | Used By | Why |
|---|---|---|
| `os_unfair_lock` | SSubscriber, SVariable, SSignalQueueState | Hot path (putNext), needs minimal overhead |
| `pthread_mutex_t` | SBlockDisposable, SMetaDisposable, SDisposableSet, SAtomic, SStrictDisposable | Cold path (dispose, state management), predictable under contention |
| `PTHREAD_MUTEX_RECURSIVE` | SAtomic (optional) | When callbacks might re-enter the same lock |

The general rule: `os_unfair_lock` for the event delivery path (called thousands of times per second), `pthread_mutex_t` for lifecycle operations (called once or a few times per signal chain).

## Complete Type Comparison: SSignalKit vs SwiftSignalKit

| SSignalKit (ObjC) | SwiftSignalKit (Swift) | Notes |
|---|---|---|
| `SSignal` | `Signal<T, E>` | Untyped vs generic |
| `SSubscriber` | `Subscriber<T, E>` | `os_unfair_lock` vs `pthread_mutex_t` |
| `SBlockDisposable` | `ActionDisposable` | Same semantics |
| `SMetaDisposable` | `MetaDisposable` | Same semantics |
| `SDisposableSet` | `DisposableSet` | Same semantics |
| `SQueue` | `Queue` | Identical optimization |
| `SAtomic` | `Atomic<T>` | ObjC adds recursive mutex option |
| `SBag` | `Bag<T>` | ObjC uses parallel arrays, Swift uses dictionary |
| `SVariable` | `Promise<T>` | Same: cache + replay latest + subscribe to source |
| `SPipe` | `ValuePipe<T>` | SPipe adds replay option |
| `STimer` | *(no equivalent)* | ObjC only — uses dispatch_source |
| Categories | `\|>` free functions | Method chaining vs pipe operator |
| `SSignalQueue` | *(no equivalent)* | ObjC only — queued execution helper |
| `SThreadPool` | *(no equivalent)* | ObjC only — custom thread pool dispatch |

## Where SSignalKit Is Used

SSignalKit is imported by approximately 30 modules, primarily:

- **LegacyComponents** (~200 ObjC files): Camera, photo/video editor, media picker, gallery, all the pre-Swift UI components
- **LegacyUI**: Bridge layer that wraps ObjC components for Swift consumption
- **LegacyMediaPickerUI**: Media picker integration between ObjC gallery and Swift UI
- **WebSearchUI**: Legacy web search gallery
- **ShareItems**: Share extension data handling
- **FetchVideoMediaResource**: Video transcoding pipeline
- **RMIntro**: Animated intro screen

The pattern at the boundary is always the same: Swift code creates a `Signal`, the bridge layer subscribes to it, converts values to ObjC objects, and feeds them into ObjC code that expects `SSignal`. Or vice versa — ObjC code produces an `SSignal`, and the bridge layer wraps it in a `Signal<T, NoError>` for Swift consumption.

## Architectural Takeaways

1. **Parallel implementations are OK.** Having two reactive frameworks sounds like technical debt, but the alternative (ObjC ↔ Swift bridging on every event) would be worse for performance. Each framework is native to its language, with no bridging overhead.

2. **The designs converge.** Despite being written in different languages with different idioms, the two frameworks have nearly identical architectures. This isn't coincidence — both were designed by the same team solving the same problem. When you understand one, you understand both.

3. **Lock discipline matters.** The consistent pattern of "lock, read state, unlock, act on state" is never violated. Closures are always called outside locks. This prevents the deadlocks that plague less disciplined reactive frameworks.

4. **`id` typing is a conscious trade-off.** Losing generics in ObjC means runtime errors instead of compile-time errors. But the ObjC modules that use SSignalKit are mature, well-tested code that doesn't change frequently. The trade-off was acceptable.

5. **Operator composition scales.** The category-based operator design means new operators can be added in separate files without modifying any existing code. Each category file (`SSignal+Mapping`, `SSignal+Meta`, etc.) is independent.

In the [next post](/posts/05-reactive-patterns/), we'll see how both frameworks are used in practice — the patterns that appear everywhere in Telegram iOS, from Promise vs ValuePromise choices to the `cached |> then(remote)` loading strategy.

## Key Files Reference

| File | Path | Lines |
|---|---|---|
| SSignal | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal.h/.m` | 22 + 163 |
| SSubscriber | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSubscriber.h/.m` | 24 + 288 |
| SDisposable | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SDisposable.h` | 7 |
| SBlockDisposable | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SBlockDisposable.h/.m` | 8 + 53 |
| SMetaDisposable | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SMetaDisposable.h/.m` | 8 + 77 |
| SDisposableSet | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SDisposableSet.h/.m` | 8 + 84 |
| SQueue | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SQueue.h/.m` | 20 + 125 |
| SAtomic | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SAtomic.h/.m` | 13 + 94 |
| SBag | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SBag.h/.m` | 12 + 75 |
| SVariable | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SVariable.h/.m` | 13 + 94 |
| SSignal+Mapping | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Mapping.h/.m` | 10 + 84 |
| SSignal+Meta | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Meta.h/.m` | 23 + 326 |
| SSignal+Dispatch | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Dispatch.h/.m` | 15 + 213 |
| SSignal+Pipe | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Pipe.h/.m` | 11 + 104 |
| SSignal+Combine | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Combine.h/.m` | 11 + 178 |
| SSignal+Catch | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Catch.h/.m` | 10 + 148 |
| SSignal+Single | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/SSignal+Single.h/.m` | 10 + 42 |
| STimer | `submodules/SSignalKit/SSignalKit/Source/SSignalKit/STimer.h/.m` | 15 + 84 |
