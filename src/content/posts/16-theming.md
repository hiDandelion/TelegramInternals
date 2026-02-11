---
title: "Theming and Presentation Data: The Visual Identity System"
description: "How Telegram iOS defines its entire visual appearance through PresentationTheme — a 1,669-line class hierarchy covering every UI surface, with four built-in themes, cloud theme sync, automatic night mode switching, reactive propagation via PresentationData signals, and a resource cache for generated images."
published: 2025-07-19
tags:
  - UI Framework
toc: true
lang: en
abbrlink: 16-theming
pin: 10
---

Every color in Telegram — from the tab bar icon tint to the chat bubble background to the destructive action red — comes from a single source of truth: `PresentationTheme`. This isn't just a color palette. It's a deep object graph with 200+ color properties organized by UI surface, supporting four built-in themes, unlimited custom themes, cloud-synced themes, per-chat themes, and automatic night mode switching.

## PresentationTheme: The Root Object

The theme is defined in `PresentationTheme.swift` (1,669 lines):

```swift
// PresentationTheme.swift (lines 1576-1618)
public final class PresentationTheme: Equatable {
    public let name: PresentationThemeName
    public let index: Int64
    public let referenceTheme: PresentationBuiltinThemeReference
    public let overallDarkAppearance: Bool
    public let intro: PresentationThemeIntro
    public let passcode: PresentationThemePasscode
    public let rootController: PresentationThemeRootController
    public let list: PresentationThemeList
    public let chatList: PresentationThemeChatList
    public let chat: PresentationThemeChat
    public let actionSheet: PresentationThemeActionSheet
    public let contextMenu: PresentationThemeContextMenu
    public let inAppNotification: PresentationThemeInAppNotification
    public let chart: PresentationThemeChart
    public let preview: Bool
    public let resourceCache: PresentationsResourceCache

    public static func ==(lhs: PresentationTheme, rhs: PresentationTheme) -> Bool {
        return lhs === rhs  // Identity equality — themes are reference types
    }
}
```

Each property is itself a class with dozens of color properties. The hierarchy mirrors the app's UI structure:

```
PresentationTheme
├── intro                    // Onboarding screens
├── passcode                 // Lock screen gradient + button color
├── rootController
│   ├── statusBarStyle       // .black or .white
│   ├── tabBar               // 9 colors (bg, separator, icon, selected, badge...)
│   ├── navigationBar        // 12 colors (button, text, bg, separator, badge...)
│   └── navigationSearchBar  // 6 colors
├── list                     // Settings & list screens (30+ colors)
│   ├── blocksBackgroundColor
│   ├── itemPrimaryTextColor
│   ├── itemAccentColor
│   ├── itemDestructiveColor
│   ├── itemSwitchColors
│   ├── itemDisclosureActions
│   └── ...
├── chatList                 // Chat list (25+ colors)
│   ├── titleColor
│   ├── secretTitleColor     // Green for secret chats
│   ├── unreadBadgeActiveBackgroundColor
│   └── ...
├── chat                     // Message view (50+ colors)
│   ├── message (incoming/outgoing bubble colors)
│   ├── inputPanel
│   ├── historyNavigation
│   └── ...
├── actionSheet              // Bottom sheets
├── contextMenu              // Long-press menus
├── inAppNotification        // Banner notifications
└── chart                    // Statistics charts
```

### The withUpdated Pattern

Every theme sub-class follows the same immutable update pattern:

```swift
// PresentationTheme.swift (lines 65-90)
public final class PresentationThemeRootTabBar {
    public let backgroundColor: UIColor
    public let separatorColor: UIColor
    public let iconColor: UIColor
    public let selectedIconColor: UIColor
    public let textColor: UIColor
    public let selectedTextColor: UIColor
    public let badgeBackgroundColor: UIColor
    public let badgeStrokeColor: UIColor
    public let badgeTextColor: UIColor

    public func withUpdated(
        backgroundColor: UIColor? = nil,
        separatorColor: UIColor? = nil,
        iconColor: UIColor? = nil,
        selectedIconColor: UIColor? = nil,
        // ... all optional
    ) -> PresentationThemeRootTabBar {
        return PresentationThemeRootTabBar(
            backgroundColor: backgroundColor ?? self.backgroundColor,
            separatorColor: separatorColor ?? self.separatorColor,
            // ...
        )
    }
}
```

This pattern appears on every sub-class throughout the file. It enables selective customization — when a custom theme changes only the accent color, the `withUpdated` method preserves all other colors from the base theme.

### The List Theme: 35 Colors for Settings Screens

The `PresentationThemeList` class (lines 425-562) demonstrates the granularity:

```swift
public final class PresentationThemeList {
    public let blocksBackgroundColor: UIColor       // Section background
    public let modalBlocksBackgroundColor: UIColor  // Modal variant
    public let plainBackgroundColor: UIColor        // Full-width list bg
    public let itemPrimaryTextColor: UIColor        // Main text
    public let itemSecondaryTextColor: UIColor      // Subtitle text
    public let itemDisabledTextColor: UIColor       // Greyed out
    public let itemAccentColor: UIColor             // Tappable items, links
    public let itemHighlightedColor: UIColor        // Highlighted accent
    public let itemDestructiveColor: UIColor        // Delete, destructive
    public let itemPlaceholderTextColor: UIColor    // Input placeholder
    public let itemBlocksBackgroundColor: UIColor   // Individual cell bg
    public let itemHighlightedBackgroundColor: UIColor // Tap highlight
    public let itemBlocksSeparatorColor: UIColor    // Between cells
    public let disclosureArrowColor: UIColor        // Chevron >
    public let sectionHeaderTextColor: UIColor      // Section header
    public let freeTextColor: UIColor               // Footer text
    public let freeTextErrorColor: UIColor          // Error footer
    public let freeTextSuccessColor: UIColor        // Success footer
    public let itemSwitchColors: PresentationThemeSwitch
    public let itemDisclosureActions: PresentationThemeItemDisclosureActions
    public let itemCheckColors: PresentationThemeFillStrokeForeground
    public let freeInputField: PresentationInputFieldTheme
    public let paymentOption: PaymentOption
    // ... 35 total
}
```

Every visual element in a settings screen has its own color property. This is why Telegram themes can customize every detail — there are no "generic" colors shared between unrelated UI surfaces.

### Resource Cache: Generated Image Caching

Theme-derived images (like rounded corners, gradients, icons tinted to theme colors) are generated on demand and cached:

```swift
// PresentationTheme.swift (lines 1620-1634)
public func image(_ key: Int32,
                  _ generate: (PresentationTheme) -> UIImage?) -> UIImage? {
    return self.resourceCache.image(key, self, generate)
}

public func object(_ key: Int32,
                   _ generate: (PresentationTheme) -> AnyObject?) -> AnyObject? {
    return self.resourceCache.object(key, self, generate)
}
```

UI code calls `theme.image(ResourceKey.chatBubbleCorner) { theme in generateCornerImage(theme) }`. The first call generates the image; subsequent calls return the cached version. When the theme changes, a new `PresentationTheme` instance is created (they're identity-compared via `===`), so the old cache is discarded naturally.

## The Four Built-in Themes

Telegram ships four base themes, each defined in its own file:

```swift
// MakePresentationTheme.swift (lines 7-20)
public func makeDefaultPresentationTheme(
    reference: PresentationBuiltinThemeReference,
    extendingThemeReference: PresentationThemeReference? = nil,
    serviceBackgroundColor: UIColor?,
    preview: Bool = false
) -> PresentationTheme {
    switch reference {
    case .dayClassic:
        return makeDefaultDayPresentationTheme(day: false, ...)
    case .day:
        return makeDefaultDayPresentationTheme(day: true, ...)
    case .night:
        return makeDefaultDarkPresentationTheme(...)
    case .nightAccent:
        return makeDefaultDarkTintedPresentationTheme(...)
    }
}
```

| Theme | File | Description |
|-------|------|-------------|
| Day Classic | `DefaultDayPresentationTheme.swift` | Light theme with classic Telegram look |
| Day | Same file, `day: true` | Modern light theme with slightly different styling |
| Night | `DefaultDarkPresentationTheme.swift` | Pure dark theme (black backgrounds) |
| Night Accent | `DefaultDarkTintedPresentationTheme.swift` | Tinted dark theme (dark blue backgrounds) |

The `referenceTheme` property stores which base theme this is, enabling the framework to determine dark/light appearance:

```swift
public init(/* ... */) {
    var overallDarkAppearance = overallDarkAppearance
    if [.night, .tinted].contains(referenceTheme.baseTheme) {
        overallDarkAppearance = true  // Force dark for night themes
    }
    // ...
}
```

## Custom Themes: Three Levels of Customization

### Level 1: Accent Color Customization

The simplest customization changes the accent color and optionally bubble colors:

```swift
// MakePresentationTheme.swift (lines 22-34)
public func customizePresentationTheme(
    _ theme: PresentationTheme,
    editing: Bool,
    title: String? = nil,
    accentColor: UIColor?,
    outgoingAccentColor: UIColor?,
    backgroundColors: [UInt32],
    bubbleColors: [UInt32],
    animateBubbleColors: Bool?,
    wallpaper: TelegramWallpaper? = nil,
    baseColor: PresentationThemeBaseColor? = nil
) -> PresentationTheme {
    switch theme.referenceTheme {
    case .day, .dayClassic:
        return customizeDefaultDayTheme(theme: theme, ...)
    case .night:
        return customizeDefaultDarkPresentationTheme(theme: theme, ...)
    case .nightAccent:
        return customizeDefaultDarkTintedPresentationTheme(theme: theme, ...)
    }
}
```

The accent color propagates to dozens of derived colors — links, switches, badges, button tints — all computed from the single accent value. Bubble colors independently control outgoing message appearance.

### Level 2: Cloud Themes

Themes can be synced from the server as `TelegramTheme` objects:

```swift
// MakePresentationTheme.swift (lines 41-55)
public func makePresentationTheme(
    cloudTheme: TelegramTheme,
    dark: Bool = false
) -> PresentationTheme? {
    // Find settings matching light/dark preference
    let settings: TelegramThemeSettings?
    if let exactSettings = cloudTheme.settings?.first(where: {
        dark ? ($0.baseTheme == .night || $0.baseTheme == .tinted)
             : ($0.baseTheme == .classic || $0.baseTheme == .day)
    }) {
        settings = exactSettings
    } else if let firstSettings = cloudTheme.settings?.first {
        settings = firstSettings
    } else {
        settings = nil
    }

    // Build from base + customization
    let defaultTheme = makeDefaultPresentationTheme(
        reference: PresentationBuiltinThemeReference(baseTheme: settings.baseTheme),
        extendingThemeReference: .cloud(PresentationCloudTheme(theme: cloudTheme, ...)))
    return customizePresentationTheme(defaultTheme, ...)
}
```

Cloud themes carry a `TelegramThemeSettings` with accent color, bubble colors, and wallpaper. The framework builds on a base theme and applies the cloud customizations on top.

### Level 3: Per-Chat Themes

Individual chats can have their own theme:

```swift
public func makePresentationTheme(chatTheme: ChatTheme, dark: Bool = false)
    -> PresentationTheme?
```

This follows the same pattern — start from a base, apply customizations. The chat theme is stored as part of the chat's settings on the server.

### Theme Reference Tracking

The `PresentationThemeReference` enum tracks where a theme came from:

```swift
enum PresentationThemeReference {
    case builtin(PresentationBuiltinThemeReference)  // 4 built-in
    case local(PresentationLocalThemeInfo)            // .attheme files
    case cloud(PresentationCloudTheme)                // Server-synced
}
```

This reference is used for persistence — when the user selects a theme, the reference (not the entire theme object) is saved to `AccountManager`. On launch, the reference is resolved into a full `PresentationTheme`.

## PresentationData: The Complete Presentation Context

The theme is just one piece of the presentation puzzle. `PresentationData` bundles everything that affects how the UI looks and behaves:

```swift
// PresentationData.swift (lines 81-108)
public final class PresentationData: Equatable {
    public let strings: PresentationStrings         // Localized strings
    public let theme: PresentationTheme             // Colors
    public let autoNightModeTriggered: Bool          // Night mode active?
    public let chatWallpaper: TelegramWallpaper     // Chat background
    public let chatFontSize: PresentationFontSize   // Message text size
    public let chatBubbleCorners: PresentationChatBubbleCorners // Bubble shape
    public let listsFontSize: PresentationFontSize  // List item text size
    public let dateTimeFormat: PresentationDateTimeFormat // 12/24h, date order
    public let nameDisplayOrder: PresentationPersonNameOrder // First Last or Last First
    public let nameSortOrder: PresentationPersonNameOrder   // Contact sorting
    public let reduceMotion: Bool                   // Accessibility
    public let largeEmoji: Bool                     // Large single emoji
}
```

This is the single object that every view controller and component receives. It encapsulates:

- **Visual**: theme colors, wallpaper, font sizes, bubble corners
- **Localization**: all UI strings for the current language
- **Formatting**: date/time format, name display order
- **Accessibility**: reduced motion, large emoji preferences

### Chat Bubble Corners

The bubble corner configuration is especially interesting:

```swift
// PresentationData.swift (lines 67-79)
public struct PresentationChatBubbleCorners: Equatable, Hashable {
    public var mainRadius: CGFloat       // Primary corner radius
    public var auxiliaryRadius: CGFloat   // Secondary (smaller) radius
    public var mergeBubbleCorners: Bool   // Merge adjacent message corners
    public var hasTails: Bool             // Chat bubble tails
}
```

Telegram lets users customize bubble corner radius (from sharp rectangles to fully rounded) and toggle bubble tails — all controlled through this struct that propagates via `PresentationData`.

## Reactive Theme Propagation

Theme changes propagate through the app via SwiftSignalKit signals. The central signal is `updatedPresentationData()`:

```swift
// PresentationData.swift (lines 237-310)
public func currentPresentationDataAndSettings(
    accountManager: AccountManager<TelegramAccountManagerTypes>,
    systemUserInterfaceStyle: WindowUserInterfaceStyle
) -> Signal<InitialPresentationDataAndSettings, NoError> {
    return accountManager.transaction { transaction -> InternalData in
        let localizationSettings = transaction.getSharedData(
            SharedDataKeys.localizationSettings)
        let presentationThemeSettings = transaction.getSharedData(
            ApplicationSpecificSharedDataKeys.presentationThemeSettings)
        // ... load all settings from AccountManager
    }
    |> deliverOn(Queue(name: "PresentationData-Load", qos: .userInteractive))
    |> map { internalData -> InitialPresentationDataAndSettings in
        // Resolve theme reference to full PresentationTheme
        let themeSettings = internalData.presentationThemeSettings
            ?? PresentationThemeSettings.defaultSettings

        // Check automatic night mode
        let parameters = AutomaticThemeSwitchParameters(
            settings: themeSettings.automaticThemeSwitchSetting)
        let autoNightModeTriggered: Bool
        if automaticThemeShouldSwitchNow(parameters,
            systemUserInterfaceStyle: systemUserInterfaceStyle) {
            effectiveTheme = themeSettings.automaticThemeSwitchSetting.theme
            autoNightModeTriggered = true
        } else {
            effectiveTheme = themeSettings.theme
            autoNightModeTriggered = false
        }

        // Build PresentationData from settings
        return InitialPresentationDataAndSettings(
            presentationData: PresentationData(
                strings: strings,
                theme: resolvedTheme,
                autoNightModeTriggered: autoNightModeTriggered,
                chatWallpaper: wallpaper,
                chatFontSize: themeSettings.fontSize,
                chatBubbleCorners: chatBubbleCorners,
                // ...
            ),
            // ... other settings
        )
    }
}
```

The entire presentation state is loaded from `AccountManager` (the multi-account preference store from Post 6), including the automatic night mode check. The signal is delivered on a high-priority queue to minimize theme switch latency.

## Automatic Night Mode

Telegram supports four automatic theme switching modes:

1. **System** — Follow iOS dark mode (`UITraitCollection.userInterfaceStyle`)
2. **Time-based** — Switch at sunrise/sunset (using the Sunrise library)
3. **Brightness-based** — Switch when screen brightness drops below a threshold
4. **Manual** — User-specified time ranges

The `automaticThemeShouldSwitchNow()` function evaluates these conditions:

```swift
// PresentationData.swift (referenced at line 390)
if automaticThemeShouldSwitchNow(parameters,
    systemUserInterfaceStyle: systemUserInterfaceStyle) {
    effectiveTheme = themeSettings.automaticThemeSwitchSetting.theme
    autoNightModeTriggered = true

    if let baseTheme = themeSettings.themePreferredBaseTheme[effectiveTheme.index],
       [.night, .tinted].contains(baseTheme) {
        preferredBaseTheme = baseTheme
    } else {
        preferredBaseTheme = .night
    }
}
```

When night mode is triggered, the app switches to the user's preferred dark theme variant (pure dark or tinted). The `autoNightModeTriggered` flag lets the UI show an indicator that the switch was automatic.

## How Controllers Receive Theme Updates

The propagation chain works differently for old-style (node-based) and new-style (ComponentFlow) controllers.

### Node-Based Controllers

Traditional view controllers subscribe to the `presentationData` signal:

```swift
// Pattern used throughout TelegramUI
class SomeController: ViewController {
    private var presentationData: PresentationData
    private var presentationDataDisposable: Disposable?

    init(context: AccountContext) {
        self.presentationData = context.sharedContext.currentPresentationData.with { $0 }
        super.init()

        self.presentationDataDisposable = (
            context.sharedContext.presentationData
            |> deliverOnMainQueue
        ).start(next: { [weak self] presentationData in
            guard let self else { return }
            let previousTheme = self.presentationData.theme
            self.presentationData = presentationData

            if previousTheme !== presentationData.theme {
                self.updateTheme(presentationData.theme)
            }
        })
    }

    private func updateTheme(_ theme: PresentationTheme) {
        self.navigationBar?.updatePresentationData(...)
        self.controllerNode.updateTheme(theme)
    }
}
```

The pattern:
1. Read the current value synchronously on init
2. Subscribe to the signal for updates
3. On update, compare by identity (`!==`) and apply changes
4. Propagate to child nodes

### ComponentFlow Controllers

For ComponentFlow-based screens, the theme flows through the Environment:

```swift
// Usage in ComponentFlow screens
let environment = context.environment[
    ViewControllerComponentContainer.Environment.self
].value
let theme = environment.theme
let strings = environment.strings
```

The `ViewControllerComponentContainer` injects `PresentationData` into the component environment. When the theme changes, the environment value changes, triggering the four-way diff (Post 15) and re-rendering affected components.

## PresentationStrings: Localization

The `PresentationStrings` class provides all localized strings for the app. It's generated from `.strings` files and includes pluralization rules for different languages:

```swift
// Part of PresentationData
public let strings: PresentationStrings
```

Strings are accessed as properties: `presentationData.strings.Settings_General`, `presentationData.strings.Chat_MessageDeleted`. The generator produces type-safe accessors for every string key.

Strings travel alongside the theme in `PresentationData` because they often change together — switching the app language triggers a new `PresentationData` emission with updated strings and potentially an RTL-adjusted theme.

## The Date/Time Format System

Telegram respects locale-specific formatting:

```swift
// PresentationData.swift (lines 13-41)
public struct PresentationDateTimeFormat: Equatable {
    public let timeFormat: PresentationTimeFormat      // .regular (12h) / .military (24h)
    public let dateFormat: PresentationDateFormat       // .monthFirst / .dayFirst
    public let dateSeparator: String                    // "." or "/" etc.
    public let dateSuffix: String                       // "." in some locales
    public let requiresFullYear: Bool
    public let decimalSeparator: String
    public let groupingSeparator: String
}
```

Every timestamp in the app — message times, last seen, file dates — formats through this struct. It's loaded from the device locale settings and bundled into `PresentationData`.

## Modal Blocks Background

A subtle but important theming detail — modal screens can have different background colors:

```swift
// PresentationTheme.swift (lines 1661-1668)
public func withModalBlocksBackground() -> PresentationTheme {
    if self.list.blocksBackgroundColor.rgb ==
       self.list.plainBackgroundColor.rgb {
        let list = self.list.withUpdated(
            blocksBackgroundColor: self.list.modalBlocksBackgroundColor,
            itemBlocksBackgroundColor: self.list.itemModalBlocksBackgroundColor)
        return PresentationTheme(/* ... list: list ... */)
    } else {
        return self
    }
}
```

When a settings screen is presented modally (as a sheet), its section backgrounds change to provide visual distinction from the underlying content. This only applies when the regular and plain backgrounds are the same color — otherwise the existing contrast is sufficient.

## How Theme Colors Flow to Pixels

Tracing a single color from definition to screen:

1. **Definition**: `DefaultDayPresentationTheme.swift` defines `PresentationThemeRootTabBar(iconColor: UIColor(rgb: 0x999999), ...)`

2. **Storage**: The theme is persisted as a `PresentationThemeReference.builtin(.dayClassic)` in `AccountManager`

3. **Resolution**: On load, `currentPresentationDataAndSettings()` resolves the reference to a full `PresentationTheme`

4. **Signal**: The resolved `PresentationData` is emitted on the `presentationData` signal

5. **Delivery**: `SharedAccountContextImpl` receives the update and stores it in `currentPresentationData`

6. **Propagation**: Each controller's disposable fires with the new data

7. **Application**: The tab bar controller reads `theme.rootController.tabBar.iconColor` and applies it

8. **Custom themes**: If a custom accent color was set, step 1 changes — `customizeDefaultDayTheme` derives the icon color from the accent color instead

The entire chain executes within a single run loop iteration when the user switches themes.

## Summary

Telegram's theming system is comprehensive and deeply integrated:

- **PresentationTheme** — 1,669-line class hierarchy with 200+ color properties organized by UI surface (tab bar, navigation, lists, chat, action sheets, context menus)
- **Four built-in themes** — Day Classic, Day, Night, Night Accent — each fully defined in their own files with every color explicitly specified
- **Three customization levels** — Accent color (automatic derivation), cloud themes (server-synced settings), per-chat themes (individual conversations)
- **PresentationData** — Bundles theme + strings + formatting + accessibility into a single propagatable object
- **Reactive propagation** — SwiftSignalKit signals deliver theme changes to every controller on the main queue
- **Automatic night mode** — System, time-based, brightness-based, or manual switching
- **Resource cache** — Theme-derived images generated once and cached on the theme instance
- **withUpdated pattern** — Immutable updates enabling selective customization while preserving base colors

The next post covers navigation architecture — how `NavigationController`, `ViewController`, and the custom navigation bar work together to manage Telegram's screen hierarchy.
