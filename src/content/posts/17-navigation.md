---
title: "Navigation Architecture: Controllers, Containers, and the Custom Navigation Bar"
description: "How Telegram iOS manages screen hierarchy using a custom NavigationController built on AsyncDisplayKit nodes — the ViewController base class with display nodes, eight navigation presentation modes, flat vs. split containers for iPad, modal and overlay stacking, the custom NavigationBar with SVG back arrows, and layout propagation."
published: 2025-07-22
tags:
  - UI Framework
toc: true
lang: en
abbrlink: 17-navigation
pin: 9
---

Telegram doesn't use `UINavigationController` the way Apple intends. It subclasses it, replaces its navigation bar, manages view controllers manually, and adds layers that UIKit never anticipated — modal containers, overlay containers, minimized containers, and global overlay containers. The result is a navigation system that supports phone layouts, iPad split views, picture-in-picture minimization, and overlay presentation, all managed through a single coordinator.

## ViewController: The Base Class

Every screen in Telegram inherits from `ViewController`, which extends `UIViewController` with AsyncDisplayKit integration:

```swift
// ViewController.swift (lines 85-213)
@objc open class ViewController: UIViewController, ContainableController {
    public let presentationContext: PresentationContext
    public let statusBar: StatusBar
    public let navigationBar: NavigationBar?
    public private(set) var toolbar: Toolbar?

    public final var supportedOrientations: ViewControllerSupportedOrientations
    public final var lockedOrientation: UIInterfaceOrientationMask?
    public final var lockOrientation: Bool = false

    open var navigationPresentation: ViewControllerNavigationPresentation = .default
    open var previousItem: NavigationPreviousAction?

    private var _displayNode: ASDisplayNode?
    public final var displayNode: ASDisplayNode {
        get {
            if let value = self._displayNode {
                return value
            } else {
                self.loadDisplayNode()
                return self._displayNode!
            }
        }
        set(value) {
            self._displayNode = value
        }
    }
}
```

Key differences from a stock `UIViewController`:

**Display node instead of view hierarchy.** The controller's content lives in a `displayNode` — an `ASDisplayNode` that manages its own view. This is loaded lazily via `loadDisplayNode()`, which subclasses override to create their node tree. The pattern mirrors AsyncDisplayKit's philosophy: create the node hierarchy before any views exist, allowing layout calculations on background threads.

**Custom status bar and navigation bar.** Telegram doesn't use `UINavigationBar`. Each controller has its own `StatusBar` and `NavigationBar` instances (both ASDisplayNode subclasses), managed directly rather than through UIKit's navigation system.

**Orientation management.** The `supportedOrientations` struct separates regular and compact size class orientations:

```swift
// ViewController.swift (lines 37-49)
public struct ViewControllerSupportedOrientations: Equatable {
    public var regularSize: UIInterfaceOrientationMask  // iPad
    public var compactSize: UIInterfaceOrientationMask  // iPhone

    public func intersection(_ other: ViewControllerSupportedOrientations)
        -> ViewControllerSupportedOrientations {
        return ViewControllerSupportedOrientations(
            regularSize: self.regularSize.intersection(other.regularSize),
            compactSize: self.compactSize.intersection(other.compactSize))
    }
}
```

The `NavigationController` intersects all pushed controllers' orientations to determine what the app supports at any moment. This is why Telegram properly locks rotation when a controller requires it.

### Eight Navigation Presentation Modes

How a controller is presented depends on its `navigationPresentation`:

```swift
// ViewController.swift (lines 61-70)
public enum ViewControllerNavigationPresentation {
    case `default`              // Standard push
    case master                 // Master column in split view
    case modal                  // Modal sheet
    case flatModal              // Modal without sheet animation
    case standaloneModal        // Standalone modal window
    case standaloneFlatModal    // Standalone without animation
    case modalInLargeLayout     // Modal only on iPad
    case modalInCompactLayout   // Modal only on iPhone
}
```

The `modalInLargeLayout` and `modalInCompactLayout` variants are adaptive — the same controller pushes on one device class and presents modally on the other. This is how Telegram handles screens like "Add Contact" that push on iPhone but sheet on iPad.

### Layout Propagation

Layout flows from the window down through controllers:

```swift
// ViewController.swift (lines 263-269)
open func navigationLayout(layout: ContainerViewLayout) -> NavigationLayout {
    let statusBarHeight: CGFloat = layout.statusBarHeight ?? 0.0
    var defaultNavigationBarHeight: CGFloat
    if self._presentedInModal && self._hasGlassStyle {
        defaultNavigationBarHeight = 68.0  // Taller modal nav bar
    } else {
        defaultNavigationBarHeight = 60.0  // Standard nav bar
    }
    // ...
}
```

The `ContainerViewLayout` struct carries the complete layout context — safe area insets, status bar height, input height (keyboard), and size class. Each controller computes its own navigation layout from this, then passes the remaining space to its content node.

### Modal Overlay Transition

Controllers can react to being partially covered by a modal sheet:

```swift
// ViewController.swift (lines 175-184)
public private(set) var modalStyleOverlayTransitionFactor: CGFloat = 0.0
public var modalStyleOverlayTransitionFactorUpdated:
    ((ContainedViewLayoutTransition) -> Void)?

public func updateModalStyleOverlayTransitionFactor(
    _ value: CGFloat,
    transition: ContainedViewLayoutTransition
) {
    if self.modalStyleOverlayTransitionFactor != value {
        self.modalStyleOverlayTransitionFactor = value
        self.modalStyleOverlayTransitionFactorUpdated?(transition)
    }
}
```

As a modal slides up over a controller, the `modalStyleOverlayTransitionFactor` animates from 0 to 1. The underlying controller can use this to dim its content or scale down — the same effect you see in iOS when a sheet partially covers the content behind it.

## NavigationController: The Container

`NavigationController` manages the entire navigation stack:

```swift
// NavigationController.swift (lines 148-268)
open class NavigationController: UINavigationController,
    ContainableController, UIGestureRecognizerDelegate
{
    private let mode: NavigationControllerMode
    private var theme: NavigationControllerTheme
    private let isFlat: Bool

    // Core stacks
    private var _viewControllers: [ViewController] = []
    private var rootContainer: RootContainer?
    private var modalContainers: [NavigationModalContainer] = []
    private var overlayContainers: [NavigationOverlayContainer] = []
    open var minimizedContainer: MinimizedContainer?
    private var globalOverlayContainers: [NavigationOverlayContainer] = []

    // State
    public private(set) var validLayout: ContainerViewLayout?
    private var inCallStatusBar: StatusBar?
}
```

### Five Layers of Content

The navigation controller manages five distinct layers, each with different stacking behavior:

```
┌─────────────────────────────────────────┐
│ 5. Global overlays (above keyboard)     │  ← globalOverlayContainers
├─────────────────────────────────────────┤
│ 4. Global overlays (below keyboard)     │  ← globalOverlayBelowKeyboard
├─────────────────────────────────────────┤
│ 3. Minimized container (PiP)            │  ← minimizedContainer
├─────────────────────────────────────────┤
│ 2. Modal stack (sheets)                 │  ← modalContainers
├─────────────────────────────────────────┤
│ 1. Root container (push/pop stack)      │  ← rootContainer
│    - Flat (phone) or Split (iPad)       │
└─────────────────────────────────────────┘
```

**Root container** — The primary navigation stack. On iPhone, it's a `NavigationContainer` (flat stack). On iPad, it's a `NavigationSplitContainer` (master/detail columns):

```swift
// NavigationController.swift (lines 61-64)
private enum RootContainer {
    case flat(NavigationContainer)
    case split(NavigationSplitContainer)
}
```

**Modal containers** — Sheets presented over the root. Each modal is wrapped in `NavigationModalContainer`, which handles the sheet's dismiss gesture, corner radius, and dimming overlay. Multiple modals can stack.

**Overlay containers** — Floating UI that doesn't block the underlying content. Used for toast notifications, floating action buttons, and call banners.

**Minimized container** — Picture-in-picture mode for calls and live streams. When minimized, the content shrinks to a draggable corner window:

```swift
// NavigationController.swift (lines 179-206)
open var minimizedContainer: MinimizedContainer? {
    didSet {
        self.minimizedContainer?.navigationController = self
        self.minimizedContainer?.willMaximize = { [weak self] _ in
            guard let self else { return }
            self.isMaximizing = true
            self.updateContainersNonReentrant(
                transition: .animated(duration: 0.4, curve: .spring))
        }
        self.minimizedContainer?.willDismiss = { [weak self] _ in
            guard let self else { return }
            self.minimizedContainer = nil
            self.updateContainersNonReentrant(
                transition: .animated(duration: 0.4, curve: .spring))
        }
    }
}
```

**Global overlays** — UI that floats above even the keyboard. Split into above-keyboard and below-keyboard variants for precise z-ordering.

### Two Navigation Modes

```swift
// NavigationController.swift (lines 51-54)
public enum NavigationControllerMode {
    case single              // iPhone: single column
    case automaticMasterDetail // iPad: adaptive split view
}
```

In `automaticMasterDetail` mode, the navigation controller creates a `NavigationSplitContainer` that shows master and detail columns side by side. Controllers with `navigationPresentation = .master` go in the left column; others go in the right. When the device width narrows (iPad multitasking), it collapses to single-column automatically.

The `MasterDetailLayoutBlackout` enum handles the transition between split and collapsed:

```swift
public enum MasterDetailLayoutBlackout: Equatable {
    case master   // Master column is hidden during transition
    case details  // Details column is hidden during transition
}
```

### The Controller Stack

Unlike UIKit's `UINavigationController.viewControllers`, Telegram maintains its own stack with reactive observation:

```swift
// NavigationController.swift (lines 228-244)
private var _viewControllers: [ViewController] = []
override open var viewControllers: [UIViewController] {
    get {
        return self._viewControllers.map { $0 as UIViewController }
    } set(value) {
        self.setViewControllers(value, animated: false)
    }
}

private var _viewControllersPromise = ValuePromise<[UIViewController]>()
public var viewControllersSignal: Signal<[UIViewController], NoError> {
    return _viewControllersPromise.get()
}
```

The `viewControllersSignal` lets other parts of the app react to navigation changes. For example, the tab bar badge count updates when the chat list controller appears in the stack.

### Interaction Blocking

The navigation controller node blocks touch events during transitions to prevent double-taps:

```swift
// NavigationController.swift (lines 79-117)
private final class NavigationControllerNode: ASDisplayNode {
    override func hitTest(_ point: CGPoint,
                         with event: UIEvent?) -> UIView? {
        if let controller = self.controller,
           controller.isInteractionDisabled() {
            return self.view  // Absorb all touches
        } else {
            return super.hitTest(point, with: event)
        }
    }

    override func accessibilityPerformEscape() -> Bool {
        if let controller = self.controller,
           controller.viewControllers.count > 1 {
            let _ = self.controller?.popViewController(animated: true)
            return true
        }
        return false
    }
}
```

The VoiceOver escape gesture automatically triggers a back navigation — accessibility support built into the custom navigation system.

## NavigationBar: Custom Drawing

Telegram's `NavigationBar` replaces `UINavigationBar` entirely. It's an ASDisplayNode that draws its own content:

### Theme Structure

```swift
// NavigationBar.swift (lines 7-58)
public final class NavigationBarTheme {
    public let overallDarkAppearance: Bool
    public let buttonColor: UIColor
    public let disabledButtonColor: UIColor
    public let primaryTextColor: UIColor
    public let backgroundColor: UIColor
    public let opaqueBackgroundColor: UIColor
    public let enableBackgroundBlur: Bool
    public let separatorColor: UIColor
    public let badgeBackgroundColor: UIColor
    public let badgeStrokeColor: UIColor
    public let badgeTextColor: UIColor
    public let edgeEffectColor: UIColor?
    public let style: NavigationBar.Style
    public let glassStyle: NavigationBar.GlassStyle
}
```

Two styles exist:
- `.legacy` — Classic opaque/blurred navigation bar
- `.glass` — Modern translucent glass effect (used in Telegram's newer screens)

### SVG Back Arrow

The back arrow is drawn programmatically using SVG path data:

```swift
// NavigationBar.swift (lines 8-17)
public static func generateBackArrowImage(color: UIColor) -> UIImage? {
    return generateImage(CGSize(width: 13.0, height: 22.0),
                        rotatedContext: { size, context in
        context.clear(CGRect(origin: CGPoint(), size: size))
        context.setFillColor(color.cgColor)
        context.translateBy(x: 0.0, y: -UIScreenPixel)

        let _ = try? drawSvgPath(context, path:
            "M3.60751322,11.5 L11.5468531,3.56066017 " +
            "C12.1326395,2.97487373 12.1326395,2.02512627 " +
            "11.5468531,1.43933983 ...")
    })
}
```

This generates the arrow at the exact tint color of the current theme, cached in a static dictionary. No image assets needed — the arrow adapts to any theme color.

### Navigation Bar Presentation Data

The navigation bar is themed through a dedicated data struct:

```swift
// NavigationBar.swift (lines 60-78)
public final class NavigationBarStrings {
    public let back: String  // Localized "Back"
    public let close: String // Localized "Close"
}

public final class NavigationBarPresentationData {
    public let theme: NavigationBarTheme
    public let strings: NavigationBarStrings
}
```

### Previous Action

The back button adapts based on what's behind the current controller:

```swift
// NavigationBar.swift (lines 80-99)
public enum NavigationPreviousAction: Equatable {
    case item(UINavigationItem)  // Show previous controller's title
    case close                    // Show "Close" button (for modals)
}
```

When a controller is the root of a modal stack, it shows "Close" instead of a back arrow. When pushed onto an existing stack, it shows the previous controller's title. This is set via `previousItem` on the `ViewController`.

## NavigationControllerTheme

The navigation controller's overall theme bundles the status bar style, navigation bar theme, and empty area color:

```swift
// NavigationController.swift (lines 11-21)
public final class NavigationControllerTheme {
    public let statusBar: NavigationStatusBarStyle
    public let navigationBar: NavigationBarTheme
    public let emptyAreaColor: UIColor
}
```

The `emptyAreaColor` fills any visible background when controllers don't cover the full screen — like the area behind a narrow modal or during a split view transition.

## Presentation Patterns

### Standard Push

The simplest navigation — push a controller onto the stack:

```swift
let controller = SomeViewController(context: context)
(navigationController as? NavigationController)?
    .pushViewController(controller, animated: true)
```

The navigation controller wraps the controller in the root container, triggers the push animation, and updates the `_viewControllers` array.

### Modal Presentation

Modal sheets are presented through the window system:

```swift
let controller = SomeViewController(context: context)
controller.navigationPresentation = .modal
present(controller, in: .window(.root),
        with: ViewControllerPresentationArguments(
            presentationAnimation: .modalSheet))
```

The `ViewControllerPresentationArguments` control the animation:

```swift
// ViewController.swift (lines 51-59)
open class ViewControllerPresentationArguments {
    public let presentationAnimation: ViewControllerPresentationAnimation
    public let completion: (() -> Void)?
}

public enum ViewControllerPresentationAnimation {
    case none
    case modalSheet
}
```

### Overlay Presentation

Overlays float above the main content without blocking it:

```swift
(navigationController as? NavigationController)?.presentOverlay(
    controller: overlayController, in: .current)
```

Overlays are placed in `NavigationOverlayContainer` instances and can optionally request to be below the keyboard (for toolbars that should stay visible when the keyboard appears).

### Adaptive Presentation

Controllers with `.modalInLargeLayout` are pushed on iPhone but presented modally on iPad. The navigation controller checks the current size class and routes accordingly. This happens transparently — the presenting code doesn't need to know which device it's running on.

## ContainableController Protocol

All controllers that can be managed by the navigation system implement `ContainableController`:

```swift
public protocol ContainableController {
    var isOpaqueWhenInOverlay: Bool { get }
    var blocksBackgroundWhenInOverlay: Bool { get }
    var updateTransitionWhenPresentedAsModal:
        ((CGFloat, ContainedViewLayoutTransition) -> Void)? { get }
    var ready: Promise<Bool> { get }
    func combinedSupportedOrientations(
        currentOrientationToLock: UIInterfaceOrientationMask
    ) -> ViewControllerSupportedOrientations
}
```

The `ready` promise is important — it prevents the navigation controller from showing a controller before its content is loaded. When pushing to a screen that loads data asynchronously, the push animation waits until `ready` resolves to `true`.

## The Custom Status Bar

Telegram replaces the system status bar with its own `StatusBar` node. This allows:

- **Animated transitions** between black and white styles during navigation
- **In-call status bar** — A custom bar showing call duration, with a tap-to-return action:

```swift
// NavigationController.swift (lines 172-173)
var inCallNavigate: (() -> Void)?
private var inCallStatusBar: StatusBar?
```

- **Offset animations** — The status bar can slide up/down during scroll or navigation transitions

The `NavigationStatusBarStyle` enum is simpler than UIKit's:

```swift
public enum NavigationStatusBarStyle {
    case black  // Dark content (light backgrounds)
    case white  // Light content (dark backgrounds)
}
```

## Layout Flow

When the device rotates or a modal appears, layout propagates from the top:

1. **Window** detects size change, creates a `ContainerViewLayout`
2. **NavigationController** receives `containerLayoutUpdated(_:transition:)`
3. NavigationController updates each container layer:
   - Root container (flat or split)
   - Each modal container
   - Each overlay container
   - Minimized container
   - Global overlays
4. Each container propagates layout to its child controllers
5. Each controller calls `navigationLayout(layout:)` to compute navigation bar frame
6. Controller passes remaining space to its `displayNode`

The `ContainedViewLayoutTransition` carries the animation context:

```swift
public enum ContainedViewLayoutTransition {
    case immediate
    case animated(duration: Double, curve: ContainedViewLayoutTransitionCurve)
}
```

Every layout update carries a transition, so animations compose naturally — a rotation triggers animated layout updates at every level of the hierarchy.

## TabBarController

The tab bar is defined as a protocol (not a concrete class):

```swift
// TabBarController.swift (from Display)
public protocol TabBarController: ViewController {
    var currentController: ViewController? { get }
    var controllers: [ViewController] { get }
    var selectedIndex: Int { get set }

    func setControllers(_ controllers: [ViewController],
                       selectedIndex: Int?)
    func updateBackgroundAlpha(_ alpha: CGFloat,
                              transition: ContainedViewLayoutTransition)
    func frameForControllerTab(controller: ViewController) -> CGRect?
    func updateIsTabBarEnabled(_ value: Bool,
                              transition: ContainedViewLayoutTransition)
    func updateIsTabBarHidden(_ value: Bool,
                             transition: ContainedViewLayoutTransition)
}
```

The concrete implementation `TabBarControllerImpl` lives in the `TabBarUI` module. It manages the four main tabs (Chats, Contacts, Calls, Settings) and handles badge updates, tab bar visibility, and the tab bar's interaction state.

The `frameForControllerTab` method returns the exact frame of a tab button — used for positioning context menus and transition animations that originate from a tab icon.

## Summary

Telegram's navigation system replaces UIKit's navigation with a custom stack built on AsyncDisplayKit:

- **ViewController** — Base class with display node, custom status/navigation bars, eight presentation modes, orientation management, and layout propagation
- **NavigationController** — Five-layer container managing root (flat/split), modals, overlays, minimized (PiP), and global overlays
- **NavigationBar** — Custom ASDisplayNode with SVG back arrow, glass/legacy styles, theme integration, and programmatic drawing
- **Adaptive layout** — `automaticMasterDetail` mode handles iPad split views, collapsing to single-column on narrow widths
- **ContainableController** — Protocol with `ready` promise for async content loading, orientation intersection, and modal transition callbacks
- **Layout propagation** — `ContainerViewLayout` flows from window through containers to controllers, with animation transitions at every level

The next post covers ItemListUI — the framework that uses all of these navigation and theming primitives to build Telegram's settings screens.
