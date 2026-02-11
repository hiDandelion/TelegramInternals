---
title: "ComponentFlow: Telegram's Declarative UI Framework"
description: "How Telegram built a SwiftUI-like component system on top of UIKit — the Component protocol with value-type diffing, ComponentState for imperative re-renders, the Environment dependency injection system, CombinedComponent for composite layouts, ComponentHostView as the UIKit bridge, and built-in primitives like VStack, Text, and Button."
published: 2025-07-16
tags:
  - UI Framework
toc: true
lang: en
abbrlink: 15-component-flow
pin: 11
---

Telegram's UI framework has two layers. The previous post covered the low-level layer — AsyncDisplayKit's background rendering and node abstraction. This post covers the high-level layer: **ComponentFlow**, a custom declarative UI framework that feels remarkably like SwiftUI but runs on real UIViews.

Why build a SwiftUI-like framework instead of just using SwiftUI? Three reasons:

1. **Deployment target** — Telegram supports iOS versions where SwiftUI either didn't exist or was too buggy
2. **UIKit integration** — SwiftUI's `UIViewRepresentable` has overhead and limitations; ComponentFlow components *are* UIViews
3. **Performance control** — SwiftUI's layout engine is opaque; ComponentFlow gives explicit control over when and how views update

## The Component Protocol

The entire framework revolves around one protocol:

```swift
// Component.swift (lines 123-131)
public protocol Component: _TypeErasedComponent, Equatable {
    associatedtype EnvironmentType = Empty
    associatedtype View: UIView = UIView
    associatedtype State: ComponentState = EmptyComponentState

    func makeView() -> View
    func makeState() -> State
    func update(view: View, availableSize: CGSize, state: State,
                environment: Environment<EnvironmentType>,
                transition: ComponentTransition) -> CGSize
}
```

Three associated types, three lifecycle methods:

- **`EnvironmentType`** — The dependency injection context this component expects (defaults to `Empty`)
- **`View`** — The concrete UIView subclass this component renders into
- **`State`** — Mutable state that persists across re-renders

- **`makeView()`** — Creates the UIView once, like SwiftUI's `body` creating views
- **`makeState()`** — Creates the state once, like `@State` initialization
- **`update()`** — Called on every re-render. Returns the component's calculated size.

The key insight: components conform to `Equatable`. This means the framework can diff the previous component value against the new one and skip the `update()` call entirely if nothing changed. Unlike SwiftUI which uses view identity and state comparison, ComponentFlow directly compares the component structs.

## A Complete Component: Text

The `Text` component demonstrates every concept in 110 lines:

```swift
// Text.swift
public final class Text: Component {
    // Immutable props — this is the "description" of what to render
    public let text: String
    public let font: UIFont
    public let color: UIColor
    public let tintColor: UIColor?

    // Equatable implementation — enables diff-based updates
    public static func ==(lhs: Text, rhs: Text) -> Bool {
        if lhs.text != rhs.text { return false }
        if !lhs.font.isEqual(rhs.font) { return false }
        if !lhs.color.isEqual(rhs.color) { return false }
        if lhs.tintColor != rhs.tintColor { return false }
        return true
    }
```

The component is a value type (semantically — it's a `final class` for performance, but treated as immutable). Every property participates in equality checking.

The `View` subclass owns the rendering state:

```swift
    // Measurement cache lives on the view, not the component
    private final class MeasureState: Equatable {
        let attributedText: NSAttributedString
        let availableSize: CGSize
        let size: CGSize
    }

    public final class View: UIView {
        private var measureState: MeasureState?

        public func update(component: Text, availableSize: CGSize,
                          transition: ComponentTransition) -> CGSize {
            let attributedText = NSAttributedString(
                string: component.text,
                attributes: [.font: component.font,
                             .foregroundColor: component.color])

            // Diff check — skip re-render if nothing changed
            if let measureState = self.measureState {
                if measureState.attributedText.isEqual(to: attributedText)
                   && measureState.availableSize == availableSize {
                    return measureState.size
                }
            }

            // Measure text
            var boundingRect = attributedText.boundingRect(
                with: availableSize, options: .usesLineFragmentOrigin,
                context: nil)
            boundingRect.size.width = ceil(boundingRect.size.width)
            boundingRect.size.height = ceil(boundingRect.size.height)

            // Render to image and set on layer directly
            let renderer = UIGraphicsImageRenderer(
                bounds: CGRect(origin: .zero, size: boundingRect.size))
            let image = renderer.image { context in
                UIGraphicsPushContext(context.cgContext)
                attributedText.draw(at: CGPoint())
                UIGraphicsPopContext()
            }
            self.layer.contents = image.cgImage

            self.measureState = MeasureState(
                attributedText: attributedText,
                availableSize: availableSize,
                size: boundingRect.size)

            return boundingRect.size
        }
    }
```

Notice the pattern: the view caches its previous measurement. If the attributed text and available size haven't changed, it returns the cached size without touching the layer. Text is rendered directly into `layer.contents` as a `CGImage` — the same technique AsyncDisplayKit uses for background rendering, but here it's explicit.

The component's `update()` method simply delegates to the view:

```swift
    public func makeView() -> View {
        return View()
    }

    public func update(view: View, availableSize: CGSize,
                      state: EmptyComponentState,
                      environment: Environment<Empty>,
                      transition: ComponentTransition) -> CGSize {
        return view.update(component: self, availableSize: availableSize,
                          transition: transition)
    }
}
```

`Text` uses the default `EmptyComponentState` and `Empty` environment — it's a pure function of its props.

## ComponentState: Imperative Re-renders

For components that need mutable state, `ComponentState` provides a trigger mechanism:

```swift
// Component.swift (lines 91-105)
open class ComponentState {
    open var _updated: ((ComponentTransition, Bool) -> Void)?
    var isUpdated: Bool = false

    public init() { }

    public final func updated(
        transition: ComponentTransition = .immediate,
        isLocal: Bool = false
    ) {
        self.isUpdated = true
        self._updated?(transition, isLocal)
    }
}
```

The pattern is like React's `setState` — calling `state.updated()` triggers a re-render of the component. The `transition` parameter controls whether the re-render animates (unlike SwiftUI where you'd wrap in `withAnimation`).

Here's how it looks in practice (from `TranslateScreen.swift` in the agent research):

```swift
final class State: ComponentState {
    var translatedText: String?
    var fromLanguage: String?
    private var translationDisposable = MetaDisposable()

    init(context: AccountContext, text: String) {
        super.init()

        // Start async work, trigger re-render when done
        self.translationDisposable.set(
            translate(text: text).start(next: { [weak self] result in
                guard let self else { return }
                self.translatedText = result
                self.updated(transition: .immediate) // Re-render!
            })
        )
    }

    func changeLanguage(from: String, to: String) {
        self.fromLanguage = from
        self.updated(transition: .immediate)
    }
}
```

This is deliberately imperative. SwiftUI uses property wrappers (`@State`, `@Published`) to automatically detect changes. ComponentFlow requires you to explicitly call `state.updated()`. This gives more control — you can batch multiple state changes and trigger a single re-render, or choose different animation transitions for different updates.

## Context: The View-Component Bridge

How does the framework associate a component's state with its view across re-renders? Through `ComponentContext`, stored on the UIView using Objective-C associated objects:

```swift
// Component.swift (lines 45-67)
class ComponentContext<ComponentType: Component>:
    AnyComponentContext<ComponentType.EnvironmentType>
{
    var component: ComponentType
    let state: ComponentType.State

    init(component: ComponentType,
         environment: Environment<ComponentType.EnvironmentType>,
         state: ComponentType.State) {
        self.component = component
        self.state = state
        super.init(environment: environment)
    }
}
```

The context stores the current component, its state, and the environment. It's attached to the view:

```swift
// Component.swift (lines 69-89)
extension UIView {
    func context(typeErasedComponent component: _TypeErasedComponent)
        -> _TypeErasedComponentContext
    {
        if let context = objc_getAssociatedObject(
            self, &UIView_TypeErasedComponentContextKey
        ) as? _TypeErasedComponentContext {
            return context
        } else {
            let context = component._makeContext()
            objc_setAssociatedObject(
                self, &UIView_TypeErasedComponentContextKey,
                context, .OBJC_ASSOCIATION_RETAIN_NONATOMIC)
            return context
        }
    }
}
```

On first render, the framework calls `_makeContext()`, which creates the state and context, and attaches it to the view. On subsequent renders, it retrieves the existing context — the state persists because the view persists.

## The Update Decision Pipeline

When a component is re-rendered, the framework runs a four-way diff before deciding to call `update()`:

```swift
// CombinedComponent.swift (lines 4-70)
private func updateChildAnyComponent<EnvironmentType>(
    id: _AnyChildComponent.Id,
    component: AnyComponent<EnvironmentType>,
    view: UIView,
    availableSize: CGSize,
    transition: ComponentTransition
) -> _UpdatedChildComponent {
    let context = view.context(component: component)

    var isEnvironmentUpdated = false
    var isStateUpdated = false
    var isComponentUpdated = false
    var availableSizeUpdated = false

    if context.environment.calculateIsUpdated() {
        isEnvironmentUpdated = true
    }

    if context.erasedState.isUpdated {
        context.erasedState.isUpdated = false
        isStateUpdated = true
    }

    if context.erasedComponent != component {
        isComponentUpdated = true
    }

    if context.layoutResult.availableSize != availableSize {
        availableSizeUpdated = true
    }

    let isUpdated = isEnvironmentUpdated || isStateUpdated
                    || isComponentUpdated || availableSizeUpdated

    if !isUpdated, let size = context.layoutResult.size {
        return _UpdatedChildComponent(/* cached size */)
    } else {
        let size = component._update(
            view: view, availableSize: availableSize,
            environment: context.environment, transition: transition)
        context.layoutResult.size = size
        return _UpdatedChildComponent(/* new size */)
    }
}
```

Four checks, any of which triggers an update:

1. **Environment changed** — Theme, strings, or other injected values updated
2. **State changed** — `state.updated()` was called
3. **Component changed** — Props differ (via `Equatable`)
4. **Available size changed** — Parent is offering different dimensions

If *none* changed, the framework returns the cached size from the previous layout — no `update()` call, no view mutation. This is the performance optimization that makes ComponentFlow fast: most components in a complex hierarchy skip updates on most render passes.

## Environment: Type-Safe Dependency Injection

The `Environment` system lets parent components inject values that children can read without prop-drilling:

```swift
// Environment.swift (lines 4-10)
public final class Empty: Equatable {
    static let shared: Empty = Empty()
    public static func ==(lhs: Empty, rhs: Empty) -> Bool { return true }
}

// Environment.swift (lines 12-39)
public class _Environment {
    fileprivate var data: [Int: _EnvironmentValue] = [:]
    var _isUpdated: Bool = false

    func calculateIsUpdated() -> Bool {
        if self._isUpdated { return true }
        for (_, item) in self.data {
            if let parentEnvironment = item.parentEnvironment,
               parentEnvironment.calculateIsUpdated() {
                return true
            }
        }
        return false
    }
}
```

Environment values are stored by index in a dictionary. The `calculateIsUpdated()` method walks the environment chain to detect changes.

The clever part is how environment types are accessed. Instead of string keys (like SwiftUI's `EnvironmentKey` protocol), ComponentFlow uses **tuple-indexed subscripts**:

```swift
// Environment.swift (lines 110-149)
public extension Environment {
    // Single value
    subscript(_ t1: T.Type) -> EnvironmentValue<T> where T: Equatable {
        return EnvironmentValue(environment: self, index: 0)
    }

    // Two-value tuple
    subscript<T1, T2>(_ t1: T1.Type) -> EnvironmentValue<T1>
        where T == (T1, T2), T1: Equatable, T2: Equatable {
        return EnvironmentValue(environment: self, index: 0)
    }

    subscript<T1, T2>(_ t2: T2.Type) -> EnvironmentValue<T2>
        where T == (T1, T2), T1: Equatable, T2: Equatable {
        return EnvironmentValue(environment: self, index: 1)
    }

    // Three-value tuple, four-value tuple...
}
```

A component declares its environment as a tuple type:

```swift
// Example: component needing theme and strings
struct MyComponent: Component {
    typealias EnvironmentType = (PresentationTheme, PresentationStrings)
    // ...
}
```

Inside `update()`, you access individual values by type:

```swift
func update(view: View, availableSize: CGSize, state: State,
            environment: Environment<(PresentationTheme, PresentationStrings)>,
            transition: ComponentTransition) -> CGSize {
    let theme = environment[PresentationTheme.self].value
    let strings = environment[PresentationStrings.self].value
    // ...
}
```

The subscript constraint `where T == (T1, T2)` ensures type safety at compile time — you can't access a value that wasn't declared in the `EnvironmentType`. The tuple approach supports up to 4 values, which covers most real-world needs.

### EnvironmentValue: Change Tracking with Reference Chains

`EnvironmentValue` wraps a value with change detection:

```swift
// Environment.swift (lines 41-90)
@dynamicMemberLookup
public final class EnvironmentValue<T: Equatable>: _EnvironmentValue, Equatable {
    private var storage: EnvironmentValueStorage<T>

    public var value: T {
        switch self.storage {
        case let .direct(value):
            return value
        case let .reference(environment, index):
            return (environment.data[index] as! EnvironmentValue<T>).value
        }
    }

    public subscript<V>(dynamicMember keyPath: KeyPath<T, V>) -> V {
        return self.value[keyPath: keyPath]
    }
}
```

The `.reference` case is interesting — it allows environment values to be forwarded from parent environments without copying. When a child component's environment references a parent's value, changes propagate automatically through the reference chain.

The `@dynamicMemberLookup` conformance lets you access nested properties directly: `environment[PresentationTheme.self].rootController.tabBar.backgroundColor`.

## CombinedComponent: Composite Layouts

Simple components render a single view. For composite layouts (a row with an icon, text, and chevron), `CombinedComponent` provides a declarative body pattern similar to SwiftUI:

```swift
public protocol CombinedComponent: Component {
    typealias Body = (CombinedComponentContext<Self>) -> CGSize
    static var body: Body { get }
}
```

The body is a **static closure** that receives a context and returns a size. Here's `VStack`:

```swift
// VStack.swift
public final class VStack<ChildEnvironment: Equatable>: CombinedComponent {
    public typealias EnvironmentType = ChildEnvironment

    private let items: [AnyComponentWithIdentity<ChildEnvironment>]
    private let alignment: VStackAlignment
    private let spacing: CGFloat
    private let fillWidth: Bool

    public static var body: Body {
        let children = ChildMap(environment: ChildEnvironment.self,
                                keyedBy: AnyHashable.self)

        return { context in
            // 1. Update all children, getting their sizes
            let updatedChildren = context.component.items.map { item in
                return children[item.id].update(
                    component: item.component,
                    environment: {
                        context.environment[ChildEnvironment.self]
                    },
                    availableSize: context.availableSize,
                    transition: context.transition
                )
            }

            // 2. Calculate total size
            var size = CGSize(width: 0.0, height: 0.0)
            for child in updatedChildren {
                size.height += child.size.height
                size.width = max(size.width, child.size.width)
            }
            size.height += context.component.spacing
                * CGFloat(updatedChildren.count - 1)

            // 3. Position children with alignment
            var nextY = 0.0
            for child in updatedChildren {
                let childFrame: CGRect
                switch context.component.alignment {
                case .left:
                    childFrame = CGRect(
                        origin: CGPoint(x: 0.0, y: nextY),
                        size: child.size)
                case .center:
                    childFrame = CGRect(
                        origin: CGPoint(
                            x: floor((size.width - child.size.width) * 0.5),
                            y: nextY),
                        size: child.size)
                case .right:
                    childFrame = CGRect(
                        origin: CGPoint(
                            x: size.width - child.size.width, y: nextY),
                        size: child.size)
                }

                // 4. Add child with enter/exit animations
                context.add(child
                    .position(childFrame.center)
                    .appear(.default(scale: true, alpha: true))
                    .disappear(.default(scale: true, alpha: true))
                )

                nextY += child.size.height + context.component.spacing
            }

            return size
        }
    }
}
```

The key concepts:

**`ChildMap`** maintains stable identity for child views across re-renders. When `children[item.id]` is called, it returns the same view instance for the same ID, or creates a new one. This is equivalent to React's `key` prop.

**`context.add()`** adds a child to the parent's view hierarchy with optional modifiers. The fluent API supports:

```swift
context.add(child
    .position(CGPoint(x: 100, y: 100))
    .scale(1.2)
    .opacity(0.8)
    .cornerRadius(10.0)
    .clipsToBounds(true)
    .appear(.default(scale: true, alpha: true))
    .disappear(.default(scale: true, alpha: true))
)
```

**`.appear()` / `.disappear()`** define enter/exit animations. When a child is added or removed, these transitions run automatically — similar to SwiftUI's `.transition()` modifier.

### View Recycling

`CombinedComponent` recycles child views by identity. When the component body updates:

1. New child calls `children[item.id].update(component:...)`.
2. If a view for that ID already exists, it's reused with the new component.
3. If a child ID disappears, its view runs the `.disappear()` transition and is removed.
4. If a new child ID appears, a new view is created with the `.appear()` transition.

This is why the `update()` function on new views forces `transition.withAnimation(.none)`:

```swift
let view: ComponentType.View
if let current = parentContext.childViews[self.id] {
    view = current.view as! ComponentType.View
} else {
    view = component.makeView()
    transition = transition.withAnimation(.none)  // No animation for first render
}
```

## ComponentHostView: The UIKit Bridge

`ComponentHostView` embeds ComponentFlow into traditional UIKit hierarchies:

```swift
// ComponentHostView.swift (lines 25-106)
public final class ComponentHostView<EnvironmentType>: UIView {
    private var currentComponent: AnyComponent<EnvironmentType>?
    private var currentContainerSize: CGSize?
    private var currentSize: CGSize?
    public private(set) var componentView: UIView?
    private(set) var isUpdating: Bool = false
```

The `update()` method is the entry point:

```swift
    public func update(
        transition: ComponentTransition,
        component: AnyComponent<EnvironmentType>,
        @EnvironmentBuilder environment: () -> Environment<EnvironmentType>,
        forceUpdate: Bool = false,
        containerSize: CGSize
    ) -> CGSize {
        // Create view if needed
        let componentView: UIView
        if let current = self.componentView {
            componentView = current
        } else {
            componentView = component._makeView()
            self.componentView = componentView
            self.addSubview(componentView)
        }

        let context = componentView.context(component: component)

        // Update environment
        EnvironmentBuilder._environment = context.erasedEnvironment
        let environmentResult = environment()
        context.erasedEnvironment = environmentResult

        // Diff check — skip if nothing changed
        let isEnvironmentUpdated =
            context.erasedEnvironment.calculateIsUpdated()
        if !forceUpdate, !isEnvironmentUpdated,
           let currentComponent = self.currentComponent,
           let currentContainerSize = self.currentContainerSize,
           currentContainerSize == containerSize,
           currentComponent == component {
            return currentSize!
        }

        // Wire up state-triggered re-renders
        componentState._updated = { [weak self] transition, _ in
            guard let self else { return }
            let _ = self._update(
                transition: transition, component: component,
                /* ... */ forceUpdate: true,
                containerSize: containerSize)
        }

        // Run the component's update
        let updatedSize = component._update(
            view: componentView, availableSize: containerSize,
            environment: context.erasedEnvironment, transition: transition)

        transition.setFrame(
            view: componentView,
            frame: CGRect(origin: .zero, size: updatedSize))

        return updatedSize
    }
```

The state wiring is critical: `componentState._updated` is set to a closure that re-runs the update. When any component in the tree calls `state.updated()`, the closure fires, which calls `_update()` with `forceUpdate: true`, which re-renders the component tree.

The `hitTest` override passes through transparent areas:

```swift
    override public func hitTest(_ point: CGPoint,
                                with event: UIEvent?) -> UIView? {
        // ...
        let result = super.hitTest(point, with: event)
        if result != self {
            return result
        } else {
            return nil  // Pass through if nothing was hit
        }
    }
```

This means `ComponentHostView` is transparent to touch events — only its child components receive touches.

## ComponentTransition: Explicit Animations

Unlike SwiftUI's implicit `.animation()` modifier, ComponentFlow uses explicit transitions:

```swift
// Transition.swift (lines 140-196)
public struct ComponentTransition {
    public enum Animation {
        public enum Curve {
            case easeInOut
            case spring
            case linear
            case custom(Float, Float, Float, Float)
            case bounce(stiffness: CGFloat, damping: CGFloat)

            public static var slide: Curve {
                return .custom(0.33, 0.52, 0.25, 0.99)
            }
        }

        case none
        case curve(duration: Double, curve: Curve)
    }

    public var animation: Animation
    private var _userData: [Any] = []
}
```

The transition is passed through the entire component tree. When a component calls `transition.setFrame(view:frame:)`, the frame change is either applied immediately (for `.none`) or animated with the specified curve:

```swift
public extension ComponentTransition {
    func animateView(
        allowUserInteraction: Bool = true,
        delay: Double = 0.0,
        _ f: @escaping () -> Void,
        completion: ((Bool) -> Void)? = nil
    ) {
        switch self.animation {
        case .none:
            f()
            completion?(true)
        case let .curve(duration, curve):
            switch curve {
            case .spring, .bounce, .custom:
                // Spring animations with CALayer override
                CALayer.push(CALayerSpringParametersOverride(...))
                UIView.animate(
                    withDuration: duration,
                    delay: delay,
                    usingSpringWithDamping: dampingValue,
                    initialSpringVelocity: 0.0,
                    options: options,
                    animations: { f() },
                    completion: completion)
                CALayer.popSpringParametersOverride()
            default:
                UIView.animate(
                    withDuration: duration, delay: delay,
                    options: options,
                    animations: { f() },
                    completion: completion)
            }
        }
    }
}
```

The `CALayer.push(CALayerSpringParametersOverride(...))` is a Telegram-specific hook that overrides Core Animation's spring parameters at the CALayer level, giving more control over animation physics than standard `UIView.animate` provides.

The `_userData` array on transitions is an extensible metadata system — components can attach arbitrary data to transitions that other components read:

```swift
public func userData<T>(_ type: T.Type) -> T? {
    for item in self._userData.reversed() {
        if let item = item as? T {
            return item
        }
    }
    return nil
}
```

## ActionSlot: Callback Indirection

`ActionSlot` provides a reference-stable callback mechanism:

```swift
// ActionSlot.swift
public final class ActionSlot<Arguments>: Equatable {
    private var target: ((Arguments) -> Void)?

    public init() { }

    public static func ==(lhs: ActionSlot<Arguments>,
                          rhs: ActionSlot<Arguments>) -> Bool {
        return lhs === rhs  // Identity equality
    }

    public func connect(_ target: @escaping (Arguments) -> Void) {
        self.target = target
    }

    public func invoke(_ arguments: Arguments) {
        self.target?(arguments)
    }
}
```

Why not just pass closures? Because closures aren't `Equatable`. If a component stores a closure, it can never be equal to another instance of the same component — the diff check always triggers an update.

`ActionSlot` uses identity equality (`===`), so the same slot instance always compares equal to itself. The parent creates the slot once, passes it as a prop, and connects the actual handler later:

```swift
// Usage pattern
let highlightedAction = ActionSlot<Bool>()

// Create component with stable slot
let button = Button(
    content: AnyComponent(Text(text: "Tap me", ...)),
    action: { print("Tapped") },
    highlightedAction: highlightedAction
)

// Connect handler later (doesn't change the component's equality)
highlightedAction.connect { isHighlighted in
    print("Highlighted: \(isHighlighted)")
}
```

## Built-in Components

ComponentFlow ships with primitives that cover common UI needs:

**Layout containers:**
- `VStack` — Vertical stack with alignment (left, center, right) and spacing
- `HStack` — Horizontal stack with spacing
- `ZStack` — Overlaying children

**Visual primitives:**
- `Text` — Renders attributed text to `layer.contents`
- `Image` — Wraps `UIImageView` with tint color
- `Rectangle` — Solid color fill
- `RoundedRectangle` — Supports gradients, strokes, smooth corners

**Interactive:**
- `Button` — Full-featured button with automatic highlight, content insets, minimum size, hold action (long press with repeat), and expanded hit test areas:

```swift
// Button.swift (lines 4-63)
public final class Button: Component {
    public let content: AnyComponent<Empty>
    public let contentInsets: UIEdgeInsets
    public let minSize: CGSize?
    public let hitTestEdgeInsets: UIEdgeInsets?
    public let tag: AnyObject?
    public let automaticHighlight: Bool
    public let isEnabled: Bool
    public let isExclusive: Bool
    public let action: () -> Void
    public let holdAction: ((UIView) -> Void)?
    public let highlightedAction: ActionSlot<Bool>?

    // Builder pattern for optional configuration
    public func minSize(_ minSize: CGSize?) -> Button {
        return Button(/* copy all props with new minSize */)
    }
}
```

The `holdAction` fires repeatedly while the button is held — used for Telegram features like voice message recording and seek-by-holding on media controls.

## Type Erasure: AnyComponent

Since `Component` has associated types, it can't be used directly as a type. `AnyComponent` provides type erasure:

```swift
// Component.swift (lines 175-200)
public class AnyComponent<EnvironmentType>: _TypeErasedComponent, Equatable {
    public let wrapped: _TypeErasedComponent

    public init<ComponentType: Component>(_ component: ComponentType)
        where ComponentType.EnvironmentType == EnvironmentType {
        self.wrapped = component
    }

    public static func ==(lhs: AnyComponent, rhs: AnyComponent) -> Bool {
        return lhs.wrapped._isEqual(to: rhs.wrapped)
    }

    public func _makeView() -> UIView {
        return self.wrapped._makeView()
    }

    public func _update(view: UIView, availableSize: CGSize,
                        environment: Any,
                        transition: ComponentTransition) -> CGSize {
        return self.wrapped._update(
            view: view, availableSize: availableSize,
            environment: environment, transition: transition)
    }
}
```

`AnyComponent` preserves the `EnvironmentType` (so you can't pass a component expecting a theme into a host expecting nothing), but erases the concrete component type, `View` type, and `State` type. Equality comparison delegates to `_isEqual(to:)`, which casts and compares.

## ComponentFlow vs SwiftUI

The architectures are strikingly similar but differ in key ways:

| Aspect | ComponentFlow | SwiftUI |
|--------|--------------|---------|
| Views | Real `UIView` instances | Virtual view tree, UIKit underneath |
| Layout | Manual `CGSize` calculation in `update()` | Automatic layout engine |
| State changes | Imperative `state.updated()` | Automatic via `@State` property wrapper |
| Diffing | `Equatable` on component value | View identity + state comparison |
| Animations | Explicit `ComponentTransition` passed through tree | Implicit `.animation()` modifier |
| Environment | Tuple-indexed subscripts (up to 4 values) | `@Environment` property wrapper with `EnvironmentKey` |
| Host bridge | `ComponentHostView` embeds in UIKit | `UIHostingController` wraps SwiftUI |
| Gesture handling | `UIGestureRecognizer` on view | `.gesture()` modifier |

The most significant philosophical difference: ComponentFlow keeps you in UIKit-land. Your component's view *is* a UIView. You set `layer.contents` directly. You add `UIGestureRecognizer`s. You call `UIView.animate`. There's no abstraction boundary to cross — which means there's no abstraction boundary to fight with.

## Summary

ComponentFlow is a declarative UI framework that provides SwiftUI-like developer ergonomics while staying firmly in UIKit territory:

- **Component protocol** — Value-type descriptions with `Equatable` diffing, `update()` as the single lifecycle method, and `makeView()/makeState()` for one-time creation
- **ComponentState** — Imperative state management with explicit `updated()` triggers and transition control
- **Environment** — Type-safe dependency injection using tuple-indexed subscripts, with automatic change propagation
- **CombinedComponent** — Composite layouts with child identity tracking, view recycling, and enter/exit animations
- **ComponentHostView** — Zero-overhead bridge to UIKit, with automatic re-render wiring and transparent hit testing
- **ComponentTransition** — Explicit animation system with spring physics, custom curves, and user data extensibility

The next post covers Telegram's theming system — how `PresentationTheme` defines the entire visual appearance and propagates changes through the Environment to every component on screen.
