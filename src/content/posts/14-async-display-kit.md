---
title: "AsyncDisplayKit: Telegram's Custom Rendering Engine"
description: "How Telegram iOS achieves 60fps scrolling using a forked version of Facebook/Pinterest's Texture framework — ASDisplayNode as the abstraction over UIView/CALayer, background rendering pipelines, the interface state machine, display sentinel cancellation, subtree rasterization, and the Display module's Swift wrapper layer."
published: 2025-07-13
tags:
  - UI Framework
toc: true
lang: en
abbrlink: 14-async-display-kit
pin: 12
---

Telegram is one of the smoothest-scrolling apps on iOS. Chat lists with thousands of conversations, message threads with mixed media, sticker pickers with hundreds of animated images — all feel buttery at 60fps. This isn't because Apple's UIKit is secretly fast enough. It's because Telegram **bypasses UIKit's rendering model entirely** and does most of its work on background threads.

The engine behind this is **AsyncDisplayKit** — a framework originally created at Facebook for the Paper app, later maintained by Pinterest as "Texture." Telegram ships a forked version with significant modifications. On top of it sits the **Display** module — 126 Swift files providing Telegram-specific node types, reactive integrations, and a custom navigation stack. Together, they form the foundation of every pixel on screen.

## Why Not Just Use UIKit?

UIKit's rendering model has a fundamental constraint: almost everything happens on the main thread. When a `UITableViewCell` appears, the main thread must:

1. Create/dequeue the cell
2. Layout subviews (Auto Layout constraint solving)
3. Render text (CoreText layout + glyph rasterization)
4. Decode images (JPEG/PNG decompression)
5. Composite layers (shadow rendering, corner rounding)

At 60fps, you have **16.67ms per frame**. A single complex cell with rich text, multiple images, and rounded corners can easily take 30-50ms to lay out and render — causing visible frame drops during scrolling.

AsyncDisplayKit solves this by moving steps 2-5 to background threads. The main thread's only job during scrolling is positioning pre-rendered content.

## ASDisplayNode: The Core Abstraction

The central class is `ASDisplayNode` — an abstraction that wraps `UIView` and `CALayer` but can exist without either:

```objc
// ASDisplayNode.h
@interface ASDisplayNode : NSObject <ASLocking> {
@public
    void *_displayNodeContext;
}
```

Note what's *not* here: `ASDisplayNode` does **not** inherit from `UIView`. It's a plain `NSObject`. This is the key insight — nodes are lightweight objects that can be created, configured, and laid out on any thread. Views and layers are created lazily, only when the node actually needs to appear on screen.

### Lazy View Creation

When you access the `view` property for the first time, the node creates its backing view:

```objc
// ASDisplayNode.h (lines 219-244)
@property (readonly) UIView *view;              // Lazy, main-thread only
@property (readonly) CALayer *layer;            // Lazy, main-thread only
@property (readonly, getter=isNodeLoaded) BOOL nodeLoaded;
@property (getter=isLayerBacked) BOOL layerBacked;  // Use CALayer instead of UIView
```

The `layerBacked` property is a critical optimization. If a node doesn't need touch handling (it's not tappable), it can be backed by a raw `CALayer` instead of a `UIView`, saving the overhead of the UIResponder chain and gesture recognizer system.

### The Pending State Pattern

What happens when you set properties on a node before its view exists? The node accumulates them in a `_ASPendingState` object:

```objc
// ASDisplayNode.mm (lines 95-104)
_ASPendingState *ASDisplayNodeGetPendingState(ASDisplayNode *node)
{
    ASLockScope(node);
    _ASPendingState *result = node->_pendingViewState;
    if (result == nil) {
        result = [[_ASPendingState alloc] init];
        node->_pendingViewState = result;
    }
    return result;
}
```

When you call `node.backgroundColor = .red` before the view loads, the color is stored in `_pendingViewState`. When the view is eventually created, all accumulated properties are applied at once. This means you can fully configure a node's appearance on a background thread, and the main thread only needs to apply the batch when the view materializes.

### Synchronous vs. Asynchronous Nodes

Not all nodes are created equal. The framework distinguishes between two modes:

```objc
// ASDisplayNode.mm (lines 317-319)
BOOL isSynchronous = ![_viewClass isSubclassOfClass:[_ASDisplayView class]]
                    || ![_layerClass isSubclassOfClass:[_ASDisplayLayer class]];
setFlag(Synchronous, isSynchronous);
```

**Asynchronous nodes** use the custom `_ASDisplayView` and `_ASDisplayLayer`, which support background rendering. **Synchronous nodes** wrap existing UIKit views (like `UITextField` or `UISwitch`) and render on the main thread. Telegram uses asynchronous nodes for everything it can — text, images, backgrounds, separators — and falls back to synchronous only for UIKit controls that can't be replicated.

## The Background Rendering Pipeline

The real magic is in `ASDisplayNode+AsyncDisplay.mm`, which implements the asynchronous rendering system.

### Step 1: Capture Parameters on Main Thread

When a node needs to display, it first captures immutable drawing parameters on the main thread:

```objc
// ASDisplayNode+AsyncDisplay.mm (lines 39-50)
- (NSObject *)drawParameters
{
    __instanceLock__.lock();
    BOOL implementsDrawParameters = _flags.implementsDrawParameters;
    __instanceLock__.unlock();

    if (implementsDrawParameters) {
        return [self drawParametersForAsyncLayer:self.asyncLayer];
    } else {
        return nil;
    }
}
```

This is a snapshot of everything needed for rendering — colors, fonts, text, layout metrics. Once captured, the background thread can render without touching the node or any UIKit state.

### Step 2: Build the Display Block

The `_displayBlockWithAsynchronous:` method (lines 156-276) constructs a block that will execute on a background thread:

```objc
- (asyncdisplaykit_async_transaction_operation_block_t)
    _displayBlockWithAsynchronous:(BOOL)asynchronous
                  isCancelledBlock:(asdisplaynode_iscancelled_block_t)isCancelledBlock
                       rasterizing:(BOOL)rasterizing
{
    ASDisplayNodeAssertMainThread();

    // Capture properties while holding the lock
    __instanceLock__.lock();
    flags = _flags;
    BOOL usesImageDisplay = flags.implementsImageDisplay;
    BOOL usesDrawRect = flags.implementsDrawRect;
    __instanceLock__.unlock();

    BOOL opaque = self.opaque;
    CGRect bounds = self.bounds;
    CGFloat contentsScaleForDisplay = _contentsScaleForDisplay;
    id drawParameters = [self drawParameters];

    displayBlock = ^id{
        CHECK_CANCELLED_AND_RETURN_NIL();

        if (shouldCreateGraphicsContext) {
            return ASGraphicsCreateImage(
                self.primitiveTraitCollection, bounds.size, opaque,
                contentsScaleForDisplay, nil, isCancelledBlock, ^{
                    if (usesImageDisplay) {
                        image = [self.class displayWithParameters:drawParameters
                                                     isCancelled:isCancelledBlock];
                    } else if (usesDrawRect) {
                        [self.class drawRect:bounds
                              withParameters:drawParameters
                                 isCancelled:isCancelledBlock
                               isRasterizing:rasterizing];
                    }
                });
        }
    };
    return displayBlock;
}
```

Two rendering paths exist:

- **`+displayWithParameters:isCancelled:`** — Returns a `UIImage` directly. Used for nodes that generate images (like image nodes).
- **`+drawRect:withParameters:isCancelled:isRasterizing:`** — Draws into a `CGContext`. Used for nodes that use CoreGraphics drawing (like text nodes).

Both methods are **class methods**, not instance methods. This is deliberate — the rendering block should not access instance state. Everything it needs was captured in `drawParameters`.

### Step 3: Execute and Commit

The `displayAsyncLayer:asynchronously:` method (lines 317-389) orchestrates the actual execution:

```objc
- (void)displayAsyncLayer:(_ASDisplayLayer *)asyncLayer
            asynchronously:(BOOL)asynchronously
{
    // Build cancellation check using atomic sentinel
    asdisplaynode_iscancelled_block_t isCancelledBlock = nil;
    if (asynchronously) {
        uint displaySentinelValue = ++_displaySentinel;
        __weak ASDisplayNode *weakSelf = self;
        isCancelledBlock = ^BOOL{
            __strong ASDisplayNode *self = weakSelf;
            return self == nil ||
                   (displaySentinelValue != self->_displaySentinel.load());
        };
    }

    // Build the display block
    asyncdisplaykit_async_transaction_operation_block_t displayBlock =
        [self _displayBlockWithAsynchronous:asynchronously
                            isCancelledBlock:isCancelledBlock
                                 rasterizing:NO];

    // Completion runs on main thread
    completionBlock = ^(id<NSObject> value, BOOL canceled) {
        if (!canceled && !isCancelledBlock()) {
            UIImage *image = (UIImage *)value;
            layer.contentsScale = self.contentsScale;
            layer.contents = (id)image.CGImage;
            [self didDisplayAsyncLayer:self.asyncLayer];
        }
    };
}
```

The rendered `UIImage` arrives as a `CGImage` set directly on the `CALayer.contents`. This is the fastest possible way to update layer content — no view hierarchy traversal, no Auto Layout, no drawing.

### The Display Sentinel: Fast Cancellation

The `_displaySentinel` is an `std::atomic_uint` that increments every time the node is marked for re-display:

```objc
// ASDisplayNode.mm
std::atomic_uint _displaySentinel;
```

When the user scrolls fast, a node that was about to appear might scroll back off-screen before its background render completes. The sentinel check `displaySentinelValue != self->_displaySentinel.load()` catches this — if the sentinel has been bumped since the render started, the render is abandoned immediately. This prevents wasted CPU cycles and keeps the display queue responsive.

The cancellation block is checked multiple times during rendering via the `CHECK_CANCELLED_AND_RETURN_NIL` macro:

```objc
#define CHECK_CANCELLED_AND_RETURN_NIL(expr)  \
    if (isCancelledBlock()) {                 \
        expr;                                 \
        return nil;                           \
    }
```

### The Display Queue

Background rendering runs on a high-priority concurrent queue:

```objc
// _ASDisplayLayer.mm (lines 124-135)
+ (dispatch_queue_t)displayQueue
{
    static dispatch_queue_t displayQueue = NULL;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        displayQueue = dispatch_queue_create(
            "org.AsyncDisplayKit.ASDisplayLayer.displayQueue",
            DISPATCH_QUEUE_CONCURRENT);
        dispatch_set_target_queue(displayQueue,
            dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0));
    });
    return displayQueue;
}
```

`DISPATCH_QUEUE_CONCURRENT` allows multiple nodes to render simultaneously on different CPU cores. The `DISPATCH_QUEUE_PRIORITY_HIGH` target ensures rendering takes priority over other background work like network I/O.

## Subtree Rasterization

For complex node hierarchies — like a chat bubble containing text, a reply preview, forward header, and timestamp — rendering each sub-node individually would mean multiple texture uploads to the GPU. Subtree rasterization draws the entire tree into a single image:

```objc
// ASDisplayNode+AsyncDisplay.mm (lines 52-153)
- (void)_recursivelyRasterizeSelfAndSublayersWithIsCancelledBlock:
    (asdisplaynode_iscancelled_block_t)isCancelledBlock
    displayBlocks:(NSMutableArray *)displayBlocks
{
    // Skip hidden or invisible nodes
    if (self.isHidden || self.alpha <= 0.0) {
        return;
    }

    // Layout if needed (for rasterized subnodes without layers)
    if (rasterizingFromAscendent) {
        [self __layout];
    }

    // Capture drawing state
    UIColor *backgroundColor = self.backgroundColor;
    CGRect bounds = self.bounds;
    CGFloat cornerRadius = self.cornerRadius;

    // Add this node's drawing to the block list
    [displayBlocks addObject:^{
        CGContextRef context = UIGraphicsGetCurrentContext();
        CGContextSaveGState(context);
        CGContextTranslateCTM(context, frame.origin.x, frame.origin.y);

        if (backgroundColor && CGColorGetAlpha(backgroundColor.CGColor) > 0) {
            CGContextSetFillColorWithColor(context, backgroundColor.CGColor);
            CGContextFillRect(context, bounds);
        }
        // ... draw content with proper transforms
    }];

    // Recurse into subnodes
    for (ASDisplayNode *subnode in self.subnodes) {
        [subnode _recursivelyRasterizeSelfAndSublayersWithIsCancelledBlock:
            isCancelledBlock displayBlocks:displayBlocks];
    }
}
```

The rasterized subnodes don't even need their own layers or views — they exist purely as node objects whose content is drawn into the parent's backing store. This dramatically reduces GPU compositing work and memory usage.

## The Interface State Machine

AsyncDisplayKit doesn't just render nodes — it intelligently manages when each phase of work happens. The `ASInterfaceState` is a bitmask that drives a progressive loading pipeline:

```objc
// ASDisplayNode+InterfaceState.h (lines 20-41)
typedef NS_OPTIONS(NSUInteger, ASInterfaceState) {
    ASInterfaceStateNone          = 0,
    ASInterfaceStateMeasureLayout = 1 << 0,  // Calculate layout
    ASInterfaceStatePreload       = 1 << 1,  // Fetch data
    ASInterfaceStateDisplay       = 1 << 2,  // Render content
    ASInterfaceStateVisible       = 1 << 3,  // On screen

    ASInterfaceStateInHierarchy =
        ASInterfaceStateMeasureLayout | ASInterfaceStatePreload |
        ASInterfaceStateDisplay | ASInterfaceStateVisible,
};
```

Each state represents a phase of increasing urgency:

1. **MeasureLayout** — The node *might* appear. Calculate its size on a background thread so layout is ready.
2. **Preload** — The node is *likely* to appear. Start loading data from disk or network.
3. **Display** — The node is *very likely* to appear. Begin background rendering.
4. **Visible** — The node is on screen (at least 1 pixel visible).

The delegate protocol gives nodes precise lifecycle callbacks:

```objc
// ASDisplayNode+InterfaceState.h (lines 43-99)
@protocol ASInterfaceStateDelegate <NSObject>
- (void)interfaceStateDidChange:(ASInterfaceState)newState
                      fromState:(ASInterfaceState)oldState;

- (void)didEnterVisibleState;   // Node appeared on screen
- (void)didExitVisibleState;    // Node scrolled away
- (void)didEnterDisplayState;   // Should begin rendering
- (void)didExitDisplayState;    // Can cancel rendering
- (void)didEnterPreloadState;   // Start loading data
- (void)didExitPreloadState;    // Cancel data loading
@end
```

This creates a pipeline where, as the user scrolls, nodes 2-3 screens ahead are already loading data, nodes 1 screen ahead are already rendering, and by the time content scrolls into view, it's already a pre-rendered `CGImage` that just needs to be composited.

When nodes scroll **out** of view, the pipeline runs in reverse — `didExitDisplayState` lets nodes clear their backing stores to free memory, and `didExitPreloadState` lets them cancel pending network requests.

## The _ASDisplayLayer

Telegram's fork uses a custom `_ASDisplayLayer` that intercepts standard CALayer behavior:

```objc
// _ASDisplayLayer.mm
@implementation _ASDisplayLayer
{
    BOOL _attemptedDisplayWhileZeroSized;
    struct {
        BOOL delegateDidChangeBounds:1;
    } _delegateFlags;
}

- (void)setBounds:(CGRect)bounds
{
    CGRect oldBounds = self.bounds;
    [super setBounds:bounds];
    self.asyncdisplaykit_node.threadSafeBounds = bounds;
    [(id<ASCALayerExtendedDelegate>)self.delegate
        layer:self didChangeBoundsWithOldValue:oldBounds newValue:bounds];

    // If we tried to display at zero size, retry now
    if (_attemptedDisplayWhileZeroSized &&
        CGRectIsEmpty(bounds) == NO &&
        self.needsDisplayOnBoundsChange == NO) {
        _attemptedDisplayWhileZeroSized = NO;
        [self setNeedsDisplay];
    }
}

- (void)setDisplaySuspended:(BOOL)displaySuspended
{
    if (_displaySuspended != displaySuspended) {
        _displaySuspended = displaySuspended;
        if (!displaySuspended) {
            [self setNeedsDisplay];     // Resume: trigger display
        } else {
            [self cancelAsyncDisplay];  // Suspend: cancel in-flight renders
        }
    }
}
```

The `threadSafeBounds` property is notable — it stores the bounds in a way that's safe to read from background threads during layout calculations.

In debug builds, the layer asserts that content changes happen on the main thread:

```objc
#if DEBUG
- (void)setContents:(id)contents
{
    ASDisplayNodeAssertMainThread();
    [super setContents:contents];
}
#endif
```

## Thread Safety Architecture

ASDisplayNode uses a C++ recursive mutex for thread safety:

```objc
// ASDisplayNodeInternal.h
@interface ASDisplayNode () {
@package
    AS::RecursiveMutex __instanceLock__;

    _ASPendingState *_pendingViewState;
    UIView *_view;
    CALayer *_layer;

    struct ASDisplayNodeFlags {
        unsigned layerBacked:1;
        unsigned displaysAsynchronously:1;
        unsigned isInHierarchy:1;
        // ... 20+ bit flags
    } _flags;
}
```

The bit-packed `_flags` struct minimizes memory overhead — all boolean state fits into a few bytes. The recursive mutex allows the same thread to re-enter the lock (important when property accessors call each other).

The general threading contract:

- **Any thread before view load**: Properties stored in `_pendingViewState`, protected by mutex
- **Main thread after view load**: Properties forwarded directly to `UIView`/`CALayer`
- **Background thread during display**: Only captured `drawParameters` accessed (no node state)

## The Display Module: Swift Wrappers

The `Display` module (126 Swift files) wraps AsyncDisplayKit with Telegram-specific functionality. Here are the key node types.

### ListView: Bypassing UIKit's Layout System

`ListView` is the workhorse behind chat lists, message lists, and nearly every scrollable surface. It fundamentally bypasses UIKit's layout machinery:

```swift
// ListView.swift (lines 19-38)
private final class ListViewBackingLayer: CALayer {
    override func setNeedsLayout() { }
    override func layoutSublayers() { }
    override func setNeedsDisplay() { }
    override func displayIfNeeded() { }
    override func needsDisplay() -> Bool { return false }
    override func display() { }
}

public final class ListViewBackingView: UIView {
    public fileprivate(set) weak var target: ListView?

    override public class var layerClass: AnyClass {
        return ListViewBackingLayer.self
    }

    override public func setNeedsLayout() { }
    override public func layoutSubviews() { }
    override public func setNeedsDisplay() { }
}
```

Every single layout and display method is a **no-op**. `ListView` manages node positions entirely by itself, bypassing `UIKit`'s `setNeedsLayout`/`layoutSubviews` cycle. This eliminates the overhead that `UITableView` and `UICollectionView` incur on every frame during scrolling.

The `ListViewBackingView` delegates touch handling to the `ListView` node, which processes hit testing manually with awareness of headers and item nodes:

```swift
override public func hitTest(_ point: CGPoint, with event: UIEvent?) -> UIView? {
    if !self.isHidden, let target = self.target {
        if target.bounds.contains(point) {
            // Stop deceleration animation on touch
            if target.decelerationAnimator != nil {
                target.decelerationAnimator?.isPaused = true
                target.decelerationAnimator = nil
            }
        }
        if let result = target.headerHitTest(point, with: event) {
            return result
        }
    }
    return super.hitTest(point, with: event)
}
```

The `ListView` class itself manages all the state that `UIScrollView` normally handles:

```swift
open class ListView: ASDisplayNode, ASScrollViewDelegate {
    public final let scroller: ListViewScroller
    public private(set) final var visibleSize: CGSize = CGSize()
    public private(set) final var insets = UIEdgeInsets()
    private final var lastContentOffset: CGPoint = CGPoint()

    private final var displayLink: CADisplayLink!
    private final var needsAnimations = false

    private let infiniteScrollSize: CGFloat
    public final var preloadPages: Bool = true
}
```

The `preloadPages` property controls preloading range — when enabled, nodes up to 500 points beyond the visible area enter the preload/display interface states, ensuring content is ready before it scrolls into view.

### TextNode: CoreText on Background Threads

`TextNode` handles all text rendering in Telegram — message text, usernames, timestamps, captions. It uses CoreText for layout and supports advanced features:

```swift
// TextNode.swift
private final class TextNodeStrikethrough {
    let range: NSRange
    let frame: CGRect
}

private final class TextNodeSpoiler {
    let range: NSRange
    let frame: CGRect
}

private final class TextNodeEmbeddedItem {
    let range: NSRange
    let frame: CGRect
    let item: AnyHashable
}
```

Every text feature — strikethroughs, spoiler overlays, embedded items (custom emoji), block quotes — is tracked as a separate overlay with its own range and frame. The CoreText layout happens on a background thread, producing a `TextNodeLayout` with pre-calculated line frames, link hit areas, and decoration positions.

Block quotes support both regular quotes and code blocks, with collapsible sections:

```swift
public final class TextNodeBlockQuoteData: NSObject {
    public enum Kind: Equatable {
        case quote
        case code(language: String?)
    }

    public let kind: Kind
    public let title: NSAttributedString?
    public let color: UIColor
    public let secondaryColor: UIColor?
    public let tertiaryColor: UIColor?
    public let backgroundColor: UIColor
    public let isCollapsible: Bool
}
```

### TransformImageNode: Reactive Image Processing

`TransformImageNode` is the backbone of image display. It combines a transform signal with layout arguments using reactive composition:

```swift
// TransformImageNode.swift
open class TransformImageNode: ASDisplayNode {
    private var disposable = MetaDisposable()
    private var currentTransform: ((TransformImageArguments) -> DrawingContext?)?
    public private(set) var currentArguments: TransformImageArguments?
    private var argumentsPromise = ValuePromise<TransformImageArguments>(ignoreRepeated: true)

    public func setSignal(
        _ signal: Signal<(TransformImageArguments) -> DrawingContext?, NoError>,
        attemptSynchronously: Bool = false,
        dispatchOnDisplayLink: Bool = true
    ) {
        let argumentsPromise = self.argumentsPromise
        let data = combineLatest(signal, argumentsPromise.get())
        // ...
    }
}
```

The pattern: a **transform function** arrives via `Signal` (e.g., a JPEG decoder), and **layout arguments** arrive via `ValuePromise` (size, corners, crop rect). When either updates, they're combined and the image is re-processed. The `ignoreRepeated: true` on the arguments promise prevents re-rendering when layout hasn't actually changed.

The `TransformImageArguments` include everything needed for the image transform:

```swift
// TransformImageArguments.swift
public struct TransformImageArguments: Equatable {
    public let corners: ImageCorners
    public let imageSize: CGSize
    public let boundingSize: CGSize
    public let intrinsicInsets: UIEdgeInsets
    public let emptyColor: UIColor?
    public let custom: Any?
}
```

The `ImageCorners` struct supports per-corner control with a special "tail" variant for chat bubbles:

```swift
public enum ImageCorner: Equatable {
    case Corner(CGFloat)
    case Tail(CGFloat, UIImage)  // Chat bubble tail!
}
```

This is how Telegram renders images inside chat bubbles with the distinctive rounded-corner-plus-tail shape.

### ImageNode: DisplayLink Dispatching

`ImageNode` provides reactive image display with DisplayLink-synchronized updates:

```swift
// ImageNode.swift
public let displayLinkDispatcher = DisplayLinkDispatcher()

public class ImageNode: ASDisplayNode {
    public func setSignal(_ signal: Signal<UIImage?, NoError>) {
        self.disposable.set((signal |> deliverOnMainQueue).start(next: {
            [weak self] next in
            dispatcher.dispatch {
                if let strongSelf = self {
                    if animate, let previousContents = strongSelf.contents {
                        strongSelf.contents = image
                        let tempLayer = CALayer()
                        tempLayer.contents = previousContents
                        strongSelf.layer.addSublayer(tempLayer)
                        tempLayer.animateAlpha(from: 1.0, to: 0.0, duration: 0.2)
                    } else {
                        strongSelf.contents = image
                    }
                }
            }
        }))
    }
}
```

The `displayLinkDispatcher` synchronizes image updates to the display refresh rate. Instead of setting `layer.contents` immediately (which might happen between frames), it batches the update to coincide with the next `CADisplayLink` callback. The cross-fade animation adds a temporary layer with the old contents that fades out.

## Corner Rounding: Three Strategies

Rounded corners are everywhere in Telegram (avatars, images, bubbles). AsyncDisplayKit provides three strategies with very different performance characteristics:

```objc
// ASDisplayNode.h
typedef NS_ENUM(NSInteger, ASCornerRoundingType) {
    ASCornerRoundingTypeDefaultSlowCALayer,  // CALayer.cornerRadius
    ASCornerRoundingTypePrecomposited,       // Draw in CGContext
    ASCornerRoundingTypeClipping             // Overlay masks
};
```

**`DefaultSlowCALayer`** — Uses `CALayer.cornerRadius`. This triggers GPU offscreen rendering for every frame, which is expensive during scrolling.

**`Precomposited`** — During the async display pass, the background thread draws the rounded corners directly into the rendered `UIImage`:

```objc
// ASDisplayNode+AsyncDisplay.mm (lines 282-291)
- (void)__willDisplayNodeContentWithRenderingContext:(CGContextRef)context
                                      drawParameters:(id)drawParameters
{
    if (cornerRoundingType == ASCornerRoundingTypePrecomposited
        && cornerRadius > 0.0) {
        CGRect boundingBox = CGContextGetClipBoundingBox(context);
        [[UIBezierPath bezierPathWithRoundedRect:boundingBox
                                    cornerRadius:cornerRadius] addClip];
    }
}
```

The clipping path is applied before drawing, so the rendered image already has rounded corners baked in. The GPU just composites a rectangular texture — no offscreen rendering needed.

**`Clipping`** — Overlays pre-rendered corner mask images. Useful when the content behind the rounded area is a solid color.

Telegram predominantly uses the precomposited strategy for avatars and media, and the clipping strategy for chat bubbles where the background color is known.

## How It All Comes Together

Here's what happens when you scroll through a chat list, frame by frame:

**3 screens ahead** — `ASInterfaceStateMeasureLayout`:
- Node enters the preload range
- `calculateSizeThatFits:` runs on a background thread
- Layout is cached for instant positioning later

**2 screens ahead** — `ASInterfaceStatePreload`:
- `didEnterPreloadState` fires
- Avatar images begin loading from disk cache
- Last message text is decoded

**1 screen ahead** — `ASInterfaceStateDisplay`:
- `didEnterDisplayState` fires
- `setNeedsDisplay` triggers the async rendering pipeline
- Background thread creates `CGImage` with text, avatar, badges
- Display sentinel tracks the render

**Enters viewport** — `ASInterfaceStateVisible`:
- `completionBlock` fires on main thread
- `layer.contents = image.CGImage` — one line, takes microseconds
- Node positions already calculated, `frame` assignment only

**Main thread budget per frame**: ~2-3ms (frame positioning + content assignment)

**Background thread work** (parallel, not blocking UI):
- CoreText layout: 5-15ms per complex text node
- Image decoding: 10-30ms per photo
- Compositing: 5-10ms per rasterized subtree

Since the background work is concurrent across multiple CPU cores and the main thread only does fast operations, Telegram maintains 60fps even with hundreds of items in the list.

**Scrolling away** — Interface state reverts:
- `didExitDisplayState`: Backing store cleared (memory freed)
- `didExitPreloadState`: Network requests cancelled
- Node still exists with cached layout, ready to re-render if scrolled back

## Why Fork Instead of Using Texture Directly?

Telegram's fork diverges from upstream Texture in several ways:

1. **No ASCollectionNode/ASTableNode** — Telegram uses its own `ListView` which predates these and has different assumptions about data flow
2. **SwiftSignalKit integration** — The reactive layer is tightly coupled in ways that Texture's `IGListKit` integration never was
3. **Custom navigation** — The `ViewController` base class and `NavigationController` are built on AsyncDisplayKit nodes, not standard `UIViewController` containers
4. **Performance tuning** — The display queue priority, rasterization heuristics, and cancellation timing are tuned for messaging workloads

The framework has proven so central to Telegram's architecture that it would be impractical to update from upstream — the fork has become its own product, optimized for exactly one use case: the fastest possible messaging UI on iOS.

## Summary

AsyncDisplayKit and the Display module together form Telegram's rendering engine:

- **ASDisplayNode** abstracts UIView/CALayer, allowing creation and configuration on any thread
- **Pending state** accumulates property changes before view load, applying them in one batch
- **Background rendering** moves CoreText layout, image decoding, and compositing off the main thread
- **Display sentinel** enables instant cancellation of stale renders during fast scrolling
- **Subtree rasterization** collapses complex node hierarchies into single GPU textures
- **Interface state machine** drives a four-stage progressive loading pipeline (measure → preload → display → visible)
- **ListView** bypasses UIKit's layout system entirely for maximum scroll performance
- **TextNode, TransformImageNode, ImageNode** provide reactive, async-aware display primitives

The next post explores ComponentFlow — Telegram's declarative component framework that builds on top of these nodes, providing a SwiftUI-like development experience while maintaining full control over rendering performance.
