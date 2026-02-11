---
title: "ItemListUI: Building Settings Screens Declaratively"
description: "How Telegram iOS composes its settings screens using ItemListUI — the ItemListController reactive state machine, ItemListNodeEntry for declarative list definitions, automatic diffing with MergeLists, 16 built-in item types (switches, disclosures, inputs, actions), neighbor-aware insets for section styling, and the complete pattern for building an iOS Settings-style screen."
published: 2025-07-25
tags:
  - UI Framework
toc: true
lang: en
abbrlink: 18-item-list-ui
pin: 8
---

Telegram has dozens of settings screens — account settings, privacy, notifications, data usage, appearance, language, proxy, and more. Each follows the familiar iOS Settings pattern: grouped sections with disclosure rows, toggle switches, text inputs, and action buttons. Building these screens manually with `UITableView` would mean thousands of lines of boilerplate for data sources, cell registration, and section management.

Instead, Telegram uses **ItemListUI** — a declarative framework where you define entries as an enum, and the framework handles diffing, animation, section grouping, and styling automatically. It's built on top of `ListView` (from Post 14) and the reactive system from `SwiftSignalKit` (Posts 3-4).

## The Core Architecture

An ItemListUI screen has three layers:

1. **Entries** — An enum defining every row, implementing `ItemListNodeEntry`
2. **State signal** — A `Signal` that emits `(ControllerState, (NodeState, Arguments))` whenever data changes
3. **ItemListController** — The `ViewController` subclass that wires everything together

## ItemListNodeEntry: The Row Definition Protocol

Every row in a settings screen is defined by an entry that conforms to `ItemListNodeEntry`:

```swift
// ItemListControllerNode.swift (lines 16-26)
public protocol ItemListNodeAnyEntry {
    var anyId: AnyHashable { get }
    var tag: ItemListItemTag? { get }
    func isLessThan(_ rhs: ItemListNodeAnyEntry) -> Bool
    func isEqual(_ rhs: ItemListNodeAnyEntry) -> Bool
    func item(presentationData: ItemListPresentationData,
              arguments: Any) -> ListViewItem
}

public protocol ItemListNodeEntry: Comparable, Identifiable,
    ItemListNodeAnyEntry {
    var section: ItemListSectionId { get }
}
```

Each entry must:
- Be **`Identifiable`** — Provide a stable ID for diffing (insertions/deletions)
- Be **`Comparable`** — Define ordering for sorting
- Be **`Equatable`** — Support equality checking for update detection
- Implement **`item()`** — Convert itself into a concrete `ListViewItem` for rendering

The `section` property groups entries into visual sections. Items with the same `sectionId` appear in the same rounded group (in `.blocks` style) or with the same header (in `.plain` style).

## Automatic Diffing with MergeLists

When the state signal emits new entries, the framework diffs them against the previous list:

```swift
// ItemListControllerNode.swift (lines 52-66)
private func preparedItemListNodeEntryTransition(
    from fromEntries: [ItemListNodeAnyEntry],
    to toEntries: [ItemListNodeAnyEntry],
    presentationData: ItemListPresentationData,
    arguments: Any,
    presentationDataUpdated: Bool
) -> ItemListNodeEntryTransition {
    let (deleteIndices, indicesAndItems, updateIndices) =
        mergeListsStableWithUpdates(
            leftList: fromEntries,
            rightList: toEntries,
            isLess: { lhs, rhs in lhs.isLessThan(rhs) },
            isEqual: { lhs, rhs in lhs.isEqual(rhs) },
            getId: { value in value.anyId },
            allUpdated: presentationDataUpdated)

    let deletions = deleteIndices.map {
        ListViewDeleteItem(index: $0, directionHint: nil)
    }
    let insertions = indicesAndItems.map {
        ListViewInsertItem(
            index: $0.0, previousIndex: $0.2,
            item: $0.1.item(presentationData: presentationData,
                           arguments: arguments),
            directionHint: nil)
    }
    let updates = updateIndices.map {
        ListViewUpdateItem(
            index: $0.0, previousIndex: $0.2,
            item: $0.1.item(presentationData: presentationData,
                           arguments: arguments),
            directionHint: nil)
    }

    return ItemListNodeEntryTransition(
        deletions: deletions, insertions: insertions, updates: updates)
}
```

The `mergeListsStableWithUpdates` function (from the `MergeLists` module) computes the minimal set of insertions, deletions, and updates needed to transform the old list into the new one. This produces smooth animations — when you toggle a setting that reveals new rows, they animate in; when rows disappear, they animate out.

When `presentationDataUpdated` is true (theme or language changed), all items are marked as updated, triggering a full re-render with new colors.

## ItemListControllerState: Navigation Configuration

The controller state defines everything above the list — title, navigation buttons, and tab bar appearance:

```swift
// ItemListController.swift (lines 86-106)
public struct ItemListControllerState {
    let presentationData: ItemListPresentationData
    let title: ItemListControllerTitle
    let leftNavigationButton: ItemListNavigationButton?
    let rightNavigationButton: ItemListNavigationButton?
    let secondaryRightNavigationButton: ItemListNavigationButton?
    let backNavigationButton: ItemListBackButton?
    let tabBarItem: ItemListControllerTabBarItem?
    let animateChanges: Bool
}
```

### Title Types

The title adapts to different screen types:

```swift
// ItemListController.swift (lines 59-64)
public enum ItemListControllerTitle: Equatable {
    case text(String)                    // Simple title
    case textWithSubtitle(String, String) // Title + subtitle
    case sectionControl([String], Int)   // Segmented control (tabs)
    case textWithTabs(String, [String], Int) // Title + tabs below nav bar
}
```

The `.sectionControl` variant replaces the title with a `UISegmentedControl`, used in screens like "Privacy" where you switch between categories. `.textWithTabs` shows a title with tab buttons below the navigation bar.

### Navigation Buttons

```swift
// ItemListController.swift (lines 30-49)
public enum ItemListNavigationButtonContent: Equatable {
    case none
    case text(String)
    case icon(ItemListNavigationButtonContentIcon)  // .search, .add, .action
    case node(ASDisplayNode)  // Custom node content
}

public struct ItemListNavigationButton {
    public let content: ItemListNavigationButtonContent
    public let style: ItemListNavigationButtonStyle  // .regular, .bold, .activity
    public let enabled: Bool
    public let action: () -> Void
}
```

The `.activity` style shows a loading spinner instead of text — used when "Done" triggers a network request.

## ItemListNodeState: The List Configuration

```swift
// From agent research
public final class ItemListNodeState {
    let presentationData: ItemListPresentationData
    let entries: [ItemListNodeAnyEntry]
    let style: ItemListStyle          // .plain or .blocks
    let emptyStateItem: ItemListControllerEmptyStateItem?
    let searchItem: ItemListControllerSearch?
    let toolbarItem: ItemListToolbarItem?
    let headerItem: ItemListControllerHeaderItem?
    let footerItem: ItemListControllerFooterItem?
    let animateChanges: Bool
    let crossfadeState: Bool
    let scrollEnabled: Bool
    let focusItemTag: ItemListItemTag?
    let ensureVisibleItemTag: ItemListItemTag?
    let initialScrollToItem: ListViewScrollToItem?
}
```

Key options:

- **`style`** — `.blocks` creates grouped sections (like iOS Settings); `.plain` creates flat lists
- **`emptyStateItem`** — Placeholder shown when the list is empty
- **`searchItem`** — Search bar configuration
- **`toolbarItem`** — Bottom toolbar with up to 3 action buttons
- **`focusItemTag`** — Automatically focus a specific input field after update
- **`ensureVisibleItemTag`** — Scroll to make a specific item visible

### Two List Styles

```swift
// ItemListControllerNode.swift (lines 68-71)
public enum ItemListStyle {
    case plain   // Flat list (like Contacts)
    case blocks  // Grouped sections (like Settings)
}
```

The `.blocks` style adds section backgrounds, rounded corners on first/last items, and inter-section spacing. The `.plain` style shows items edge-to-edge with simple separators.

## The 16 Built-in Item Types

ItemListUI ships with 16 item types covering every common settings pattern:

### ItemListSwitchItem — Toggle Switch

```swift
// ItemListSwitchItem.swift (lines 16-60)
public class ItemListSwitchItem: ListViewItem, ItemListItem {
    let icon: UIImage?
    let title: String
    let text: String?               // Optional subtitle
    let textColor: TextColor        // .primary or .accent
    let value: Bool
    let type: ItemListSwitchItemNodeType  // .regular or .icon
    let enableInteractiveChanges: Bool
    let enabled: Bool
    let displayLocked: Bool         // Show lock icon (premium)
    let maximumNumberOfLines: Int
    let updated: (Bool) -> Void     // Callback on toggle
    let activatedWhileDisabled: () -> Void  // Tap when locked
}
```

The switch item handles the common "toggle with explanation" pattern. The `.icon` type replaces the standard UISwitch with a custom animated toggle. `displayLocked` shows a lock overlay for premium-gated features, with `activatedWhileDisabled` presenting the upgrade screen.

### ItemListDisclosureItem — Tappable Row with Chevron

```swift
// ItemListDisclosureItem.swift (lines 50-76)
public class ItemListDisclosureItem: ListViewItem, ItemListItem {
    let icon: UIImage?
    let iconPeer: EnginePeer?       // Peer avatar as icon
    let title: String
    let attributedTitle: NSAttributedString?
    let titleColor: ItemListDisclosureItemTitleColor
    let titleFont: ItemListDisclosureItemTitleFont
    let titleBadge: String?         // Badge next to title
    let label: String               // Right-side text
    let labelStyle: ItemListDisclosureLabelStyle
    let disclosureStyle: ItemListDisclosureStyle  // .arrow, .optionArrows, .none
    let action: (() -> Void)?
    let shimmeringIndex: Int?       // Loading skeleton
}
```

The disclosure item is the most versatile row type. `labelStyle` supports 8 variants:

```swift
public enum ItemListDisclosureLabelStyle {
    case text                       // Plain text
    case detailText                 // Smaller, gray text
    case coloredText(UIColor)       // Custom color text
    case textWithIcon(UIImage)      // Text with trailing icon
    case multilineDetailText        // Multi-line detail
    case badge(UIColor)             // Colored badge
    case color(UIColor)             // Color swatch
    case semitransparentBadge(UIColor) // Semi-transparent badge
    case image(image: UIImage, size: CGSize) // Custom image
}
```

### ItemListActionItem — Action Button

```swift
// ItemListActionItem.swift (lines 20-45)
public class ItemListActionItem: ListViewItem, ItemListItem {
    let title: String
    let kind: ItemListActionKind     // .generic, .destructive, .neutral, .disabled
    let alignment: ItemListActionAlignment  // .natural, .center
    let action: () -> Void
    let longTapAction: (() -> Void)?
}
```

The `.destructive` kind renders in red — used for "Log Out", "Delete Account", etc. The `.center` alignment centers the text — used for standalone action buttons like "Add Account".

### Other Item Types

| Item | Purpose |
|------|---------|
| `ItemListSectionHeaderItem` | Section header text |
| `ItemListTextItem` | Static text paragraphs (footers, explanations) |
| `ItemListTextWithLabelItem` | Text with a leading label |
| `ItemListSingleLineInputItem` | Single-line text input |
| `ItemListMultilineInputItem` | Multi-line text input |
| `ItemListMultilineTextItem` | Multi-line display text |
| `ItemListCheckboxItem` | Checkmark selection |
| `ItemListEditableItem` | Row with reorder/delete |
| `ItemListExpandableSwitchItem` | Switch that reveals more options |
| `ItemListPlaceholderItem` | Loading placeholder |
| `ItemListInfoItem` | Information message |
| `ItemListActivityTextItem` | Text with activity indicator |
| `ItemListTextEmptyStateItem` | Empty state message |

## The Neighbor System: Section-Aware Styling

Items need to know their position within a section to render correct corners and separators. The `itemListNeighbors` function determines each item's context:

```swift
// ItemListItem.swift (lines 50-60)
public enum ItemListNeighbor {
    case none                                    // Edge of list
    case otherSection(ItemListInsetWithOtherSection) // Different section
    case sameSection(alwaysPlain: Bool)          // Same section
}

public struct ItemListNeighbors {
    public var top: ItemListNeighbor
    public var bottom: ItemListNeighbor
}
```

```swift
// ItemListItem.swift (lines 72-112)
public func itemListNeighbors(
    item: ItemListItem,
    topItem: ItemListItem?,
    bottomItem: ItemListItem?
) -> ItemListNeighbors {
    let topNeighbor: ItemListNeighbor
    if let topItem = topItem {
        if topItem.sectionId != item.sectionId {
            let topInset: ItemListInsetWithOtherSection
            if topItem.requestsNoInset {
                topInset = .none
            } else if topItem is ItemListTextItem {
                topInset = .reduced  // Less space after text
            } else {
                topInset = .full
            }
            topNeighbor = .otherSection(topInset)
        } else {
            topNeighbor = .sameSection(alwaysPlain: topItem.isAlwaysPlain)
        }
    } else {
        topNeighbor = .none
    }
    // ... same for bottomNeighbor
    return ItemListNeighbors(top: topNeighbor, bottom: bottomNeighbor)
}
```

This produces correct visual results:
- First item in a section gets top rounded corners
- Last item in a section gets bottom rounded corners
- Middle items get no rounded corners but get separators
- Space between sections adjusts based on whether a text footer is present (`.reduced` vs `.full`)

The inset calculation converts neighbors into UIEdgeInsets:

```swift
public func itemListNeighborsPlainInsets(_ neighbors: ItemListNeighbors)
    -> UIEdgeInsets {
    var insets = UIEdgeInsets()
    switch neighbors.top {
    case .otherSection:
        insets.top += 22.0  // Inter-section spacing
    case .none, .sameSection:
        break
    }
    // ...
}

public func itemListNeighborsGroupedInsets(_ neighbors: ItemListNeighbors)
    -> UIEdgeInsets {
    // Grouped style: more spacing, rounded corners
}
```

## ItemListItem Protocol

Every item type conforms to `ItemListItem`:

```swift
// ItemListItem.swift (lines 11-16)
public protocol ItemListItem {
    var sectionId: ItemListSectionId { get }
    var tag: ItemListItemTag? { get }
    var isAlwaysPlain: Bool { get }
    var requestsNoInset: Bool { get }
}
```

- **`sectionId`** — Groups items into sections
- **`tag`** — Enables focus management (scroll to input, highlight specific row)
- **`isAlwaysPlain`** — Some items (like section headers) don't get grouped styling
- **`requestsNoInset`** — Removes inter-section spacing

The `ItemListItemTag` protocol enables programmatic focus:

```swift
public protocol ItemListItemTag {
    func isEqual(to other: ItemListItemTag) -> Bool
}

public protocol ItemListItemFocusableNode {
    func focus()
    func selectAll()
}
```

When `focusItemTag` is set in the state, the framework finds the matching node and calls `focus()` — automatically scrolling to and activating a text input field.

## Async Item Layout

Each item type uses the AsyncDisplayKit async layout pattern:

```swift
// ItemListActionItem.swift (lines 47-60)
public func nodeConfiguredForParams(
    async: @escaping (@escaping () -> Void) -> Void,
    params: ListViewItemLayoutParams,
    synchronousLoads: Bool,
    previousItem: ListViewItem?,
    nextItem: ListViewItem?,
    completion: @escaping (ListViewItemNode, ...) -> Void
) {
    async {
        let node = ItemListActionItemNode()
        let (layout, apply) = node.asyncLayout()(
            self, params,
            itemListNeighbors(item: self,
                             topItem: previousItem as? ItemListItem,
                             bottomItem: nextItem as? ItemListItem))

        node.contentSize = layout.contentSize
        node.insets = layout.insets

        Queue.mainQueue().async {
            completion(node, {
                return (nil, { _ in apply(false) })
            })
        }
    }
}
```

Layout happens on a background thread (the `async` closure), using the `asyncLayout()` pattern from AsyncDisplayKit. The layout result is then applied on the main queue. This means complex settings screens with many items don't block the main thread during initial display or updates.

## ItemListPresentationData

Items receive their theme through `ItemListPresentationData`:

```swift
// ItemListItem.swift
public final class ItemListPresentationData: Equatable {
    public let theme: PresentationTheme
    public let fontSize: PresentationFontSize
    public let strings: PresentationStrings
    public let nameDisplayOrder: PresentationPersonNameOrder
    public let dateTimeFormat: PresentationDateTimeFormat
}
```

This is a subset of the full `PresentationData`, containing only what items need for rendering. Each item receives it through its `item()` method, ensuring all items use consistent theming.

## The Toolbar System

Settings screens can have a bottom toolbar with up to 3 actions:

```swift
// ItemListControllerNode.swift (lines 78-120)
open class ItemListToolbarItem {
    public struct Action {
        public let title: String
        public let isEnabled: Bool
        public let action: () -> Void
    }

    let actions: [Action]

    var toolbar: Toolbar {
        if self.actions.count == 1 {
            // Centered single action
            middleAction = ToolbarAction(title: action.title, ...)
        } else if self.actions.count == 2 {
            // Left + right actions
            leftAction = ...
            rightAction = ...
        } else if self.actions.count == 3 {
            // Left + middle + right
            leftAction = ...
            middleAction = ...
            rightAction = ...
        }
    }
}
```

This is used for screens like "Select All / Delete Selected" toolbars in edit mode.

## Complete Example: Building a Settings Screen

Here's the pattern every settings screen follows:

### 1. Define Entry Enum

```swift
private enum SettingsSection: Int32 {
    case account = 0
    case notifications = 1
    case privacy = 2
    case actions = 3
}

private enum SettingsEntry: ItemListNodeEntry {
    case accountHeader(PresentationTheme)
    case username(PresentationTheme, String, String)
    case phoneNumber(PresentationTheme, String, String)
    case notificationsToggle(PresentationTheme, String, Bool)
    case notificationsInfo(PresentationTheme, String)
    case privacyLastSeen(PresentationTheme, String, String)
    case logOut(PresentationTheme, String)

    var section: ItemListSectionId {
        switch self {
        case .accountHeader, .username, .phoneNumber:
            return SettingsSection.account.rawValue
        case .notificationsToggle, .notificationsInfo:
            return SettingsSection.notifications.rawValue
        case .privacyLastSeen:
            return SettingsSection.privacy.rawValue
        case .logOut:
            return SettingsSection.actions.rawValue
        }
    }

    // Identifiable
    var stableId: Int {
        switch self {
        case .accountHeader: return 0
        case .username: return 1
        case .phoneNumber: return 2
        case .notificationsToggle: return 3
        case .notificationsInfo: return 4
        case .privacyLastSeen: return 5
        case .logOut: return 6
        }
    }

    // Comparable — controls ordering
    static func <(lhs: SettingsEntry, rhs: SettingsEntry) -> Bool {
        return lhs.stableId < rhs.stableId
    }

    // Equatable — controls update detection
    static func ==(lhs: SettingsEntry, rhs: SettingsEntry) -> Bool {
        switch (lhs, rhs) {
        case let (.username(t1, s1, v1), .username(t2, s2, v2)):
            return t1 === t2 && s1 == s2 && v1 == v2
        // ... other cases
        }
    }

    // Convert to ListViewItem
    func item(presentationData: ItemListPresentationData,
              arguments: Any) -> ListViewItem {
        let args = arguments as! SettingsArguments
        switch self {
        case let .username(_, title, value):
            return ItemListDisclosureItem(
                presentationData: presentationData,
                title: title,
                label: value,
                sectionId: self.section,
                style: .blocks,
                action: { args.openUsername() })

        case let .notificationsToggle(_, title, value):
            return ItemListSwitchItem(
                presentationData: presentationData,
                title: title,
                value: value,
                sectionId: self.section,
                style: .blocks,
                updated: { newValue in
                    args.updateNotifications(newValue)
                })

        case let .logOut(_, title):
            return ItemListActionItem(
                presentationData: presentationData,
                title: title,
                kind: .destructive,
                alignment: .center,
                sectionId: self.section,
                style: .blocks,
                action: { args.logOut() })
        // ... other cases
        }
    }
}
```

### 2. Define Arguments Struct

```swift
private struct SettingsArguments {
    let openUsername: () -> Void
    let updateNotifications: (Bool) -> Void
    let logOut: () -> Void
}
```

Arguments provide callbacks that entries use in their `item()` implementations. This separates the row definition (data) from the handler logic (actions).

### 3. Build the State Signal

```swift
let stateSignal = combineLatest(
    context.account.viewTracker.peerView(peerId),
    context.sharedContext.presentationData
)
|> map { peerView, presentationData
    -> (ItemListControllerState, (ItemListNodeState, SettingsArguments)) in

    var entries: [SettingsEntry] = []

    entries.append(.accountHeader(presentationData.theme))
    entries.append(.username(presentationData.theme,
        "Username", "@" + (user.username ?? "")))
    entries.append(.phoneNumber(presentationData.theme,
        "Phone", user.phone ?? ""))
    entries.append(.notificationsToggle(presentationData.theme,
        "Notifications", notificationsEnabled))
    entries.append(.notificationsInfo(presentationData.theme,
        "You can set custom notifications for each chat."))
    entries.append(.privacyLastSeen(presentationData.theme,
        "Last Seen", lastSeenSetting))
    entries.append(.logOut(presentationData.theme, "Log Out"))

    let controllerState = ItemListControllerState(
        presentationData: ItemListPresentationData(presentationData),
        title: .text("Settings"),
        leftNavigationButton: nil,
        rightNavigationButton: ItemListNavigationButton(
            content: .text("Edit"),
            style: .regular,
            enabled: true,
            action: { /* enter edit mode */ }),
        backNavigationButton: nil)

    let listState = ItemListNodeState(
        presentationData: ItemListPresentationData(presentationData),
        entries: entries.sorted(),
        style: .blocks,
        animateChanges: true)

    let arguments = SettingsArguments(
        openUsername: { /* push username screen */ },
        updateNotifications: { newValue in
            /* update settings */ },
        logOut: { /* present confirmation */ })

    return (controllerState, (listState, arguments))
}
```

### 4. Create the Controller

```swift
let controller = ItemListController(
    presentationData: ItemListPresentationData(
        context.sharedContext.currentPresentationData.with { $0 }),
    updatedPresentationData: context.sharedContext.presentationData
        |> map { ItemListPresentationData($0) },
    state: stateSignal,
    tabBarItem: nil)
```

That's it. The framework handles:
- Initial rendering of all entries
- Diffing when the signal emits new entries
- Animated insertions/deletions/updates
- Section grouping and rounded corners
- Theme updates (via `updatedPresentationData`)
- Navigation bar title and button updates
- Scroll position management
- Focus management (for input fields)

## ItemListController: The Glue

`ItemListController` is a `ViewController` subclass that wires the state signal to the node:

```swift
// ItemListController.swift (lines 108-120)
open class ItemListController: ViewController,
    KeyShortcutResponder, PresentableController
{
    var controllerNode: ItemListControllerNode {
        return (self.displayNode as! ItemListControllerNode)
    }

    private let state: Signal<
        (ItemListControllerState, (ItemListNodeState, Any)), NoError>

    private var leftNavigationButtonTitleAndStyle:
        (ItemListNavigationButtonContent, ItemListNavigationButtonStyle)?
    private var rightNavigationButtonTitleAndStyle:
        [(ItemListNavigationButtonContent, ItemListNavigationButtonStyle)] = []

    private var segmentedTitleView: ItemListControllerSegmentedTitleView?
    private var tabsNavigationContentNode: ItemListControllerTabsContentNode?
}
```

When the state signal emits, the controller:
1. Updates navigation bar title and buttons (comparing with previous values to avoid unnecessary UI changes)
2. Passes entries to `ItemListControllerNode` for diffing
3. Handles the `animateChanges` flag to control transition animation
4. Manages the segmented control or tabs if the title type requires them

## System Styles: Legacy and Glass

Items support two visual systems:

```swift
// ItemListControllerNode.swift (lines 73-76)
public enum ItemListSystemStyle {
    case glass   // Modern translucent style
    case legacy  // Classic opaque style
}
```

The `.glass` style uses translucent materials and vibrancy effects matching iOS's newer design language. Most Telegram screens use `.legacy`, but newer features use `.glass`.

## Summary

ItemListUI is Telegram's framework for declarative settings screens:

- **`ItemListNodeEntry`** — Enum-based row definitions with `Comparable`/`Equatable`/`Identifiable` for automatic diffing
- **Automatic diffing** — `mergeListsStableWithUpdates` computes minimal insertions/deletions/updates with animation
- **16 built-in item types** — Switches, disclosures, actions, inputs, checkboxes, headers, text, and more
- **Neighbor-aware styling** — Items automatically get correct corners, separators, and spacing based on their position within sections
- **Reactive state** — A single `Signal` drives both navigation bar configuration and list content
- **Async layout** — Item node layout runs on background threads via AsyncDisplayKit
- **Arguments pattern** — Callbacks separated from data through a typed arguments struct

This concludes Part V of the series. The next part will dive into specific feature implementations — ChatController, message rendering, SyncCore, TelegramEngine, and encryption.
