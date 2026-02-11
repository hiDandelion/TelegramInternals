---
title: "App Lifecycle: From main.m to Your Chat List"
description: "Tracing the complete launch path of Telegram iOS — from the Objective-C entry point through the massive AppDelegate to the moment your chat list appears."
published: 2026-02-06
tags:
  - Orientation
toc: true
lang: en
abbrlink: 02-app-lifecycle
pin: 24
---

In the [previous post](/posts/01-architecture-map/), we mapped the 274-module architecture of Telegram iOS. Now we'll trace what happens when you tap the Telegram icon — the exact sequence of code that runs from process launch to your chat list appearing on screen.

This is a surprisingly complex journey that involves: an Objective-C entry point, a custom UIApplication subclass, a 3,300-line AppDelegate, two different application contexts (authorized vs. unauthorized), a multi-account system, and a custom root controller that assembles four tab-bar screens. Understanding this flow is essential because it establishes the patterns used throughout the entire codebase.

## The Entry Point: main.m

Every iOS app begins in `main()`. Telegram's is in Objective-C:

```objc
// Telegram/Telegram-iOS/main.m
#import <UIKit/UIKit.h>

int main(int argc, char *argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, @"Application", @"AppDelegate");
    }
}
```

Two things to notice:

1. **The third argument is `@"Application"`**, not `nil`. This tells UIKit to use a custom `UIApplication` subclass named `Application` instead of the default.
2. **The fourth argument is `@"AppDelegate"`**. This is the delegate class — but it doesn't live in the main app target. It's in the `TelegramUI` module.

Why Objective-C for the entry point? Because `UIApplicationMain` is a C function that takes class name strings. Using an `.m` file avoids any Swift bridging complexity at the very first moment of process launch.

## Application.swift: The Custom UIApplication

```swift
// Telegram/Telegram-iOS/Application.swift
import UIKit

@objc(Application) class Application: UIApplication {
    override func sendEvent(_ event: UIEvent) {
        super.sendEvent(event)
    }
}
```

This is intentionally minimal. The `@objc(Application)` attribute ensures the Objective-C runtime can find this class by the name `"Application"` (matching what `main.m` passes to `UIApplicationMain`).

The `sendEvent` override is a hook point. Right now it just calls `super`, but having a custom `UIApplication` subclass means Telegram *can* intercept every touch, motion, and remote-control event before any view or gesture recognizer sees it. This is useful for:
- Detecting user activity (for auto-lock timers)
- Global keyboard shortcut handling
- Debug gesture detection

This is a pattern worth knowing: if you ever need to globally intercept events before they reach any view controller, subclassing `UIApplication` with a custom `sendEvent` is the way.

## AppDelegate: The 3,300-Line Monster

The `AppDelegate` lives in `submodules/TelegramUI/Sources/AppDelegate.swift` — not in the main app target. This is a deliberate architectural choice: by putting the delegate in a library module, it can import all the heavy frameworks (TelegramCore, Postbox, SwiftSignalKit, etc.) without polluting the thin app target.

```swift
// submodules/TelegramUI/Sources/AppDelegate.swift
@objc(AppDelegate) class AppDelegate: UIResponder, UIApplicationDelegate,
    PKPushRegistryDelegate, UNUserNotificationCenterDelegate,
    URLSessionDelegate, URLSessionTaskDelegate {
    // ...
}
```

The `@objc(AppDelegate)` attribute is critical — it's what makes `main.m`'s `@"AppDelegate"` string find this class at runtime.

Let's trace through `didFinishLaunchingWithOptions` — the method that bootstraps the entire app.

### Phase 1: Window and Metal Engine Setup (Lines 323-420)

The very first thing the AppDelegate does is set up the window system:

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
    precondition(!testIsLaunched)
    testIsLaunched = true

    let launchStartTime = CFAbsoluteTimeGetCurrent()

    // Set up global factory closures for context menus, navigation bars, etc.
    defaultNavigationBarImpl = { presentationData in
        return NavigationBarImpl(presentationData: presentationData)
    }
    makeContextControllerImpl = { context, presentationData, configuration, ... in
        return ContextControllerImpl(...)
    }
    // ... several more factory closures
```

These factory closures are a form of **dependency injection at the module boundary**. The `Display` module defines protocols and function types for navigation bars, context menus, etc. But the *implementations* live in separate modules (`NavigationBarImpl`, `ContextControllerImpl`). By setting these globals at launch, the `Display` module can create implementations without importing them directly. This breaks circular dependencies.

Next, the custom window:

```swift
    let (window, hostView) = nativeWindowHostView()
    let statusBarHost = ApplicationStatusBarHost(scene: window.windowScene)
    self.mainWindow = Window1(hostView: hostView, statusBarHost: statusBarHost)
    // ...
    self.window = window
    self.nativeWindow = window

    hostView.containerView.layer.addSublayer(MetalEngine.shared.rootLayer)
```

Telegram doesn't use a plain `UIWindow`. It uses `Window1` — a custom window class from the `Display` module that provides:
- Custom status bar management (`StatusBarHost`)
- A host view with overlay layering for modals, alerts, and global overlays
- Integration with `MetalEngine.shared.rootLayer` for GPU-accelerated animation rendering

The Metal engine's root layer is added at the very bottom of the view hierarchy, allowing Lottie stickers and other animations to render with Metal shaders without disrupting the UIKit layer tree.

### Phase 2: Build Configuration and Network Arguments (Lines 527-627)

```swift
    let baseAppBundleId = Bundle.main.bundleIdentifier!
    let appGroupName = "group.\(baseAppBundleId)"

    let buildConfig = BuildConfig(baseAppBundleId: baseAppBundleId)
    self.buildConfig = buildConfig

    let apiId: Int32 = buildConfig.apiId
    let apiHash: String = buildConfig.apiHash
```

`BuildConfig` reads configuration injected at build time by the Bazel build system — the API ID, API hash, bundle IDs, feature flags. This is where the separation between "build input" and "source code" happens: sensitive credentials live in `build-input/configuration-repository/` and never touch the Git repository.

Then comes `NetworkInitializationArguments` — a massive struct that configures the entire networking stack:

```swift
    let networkArguments = NetworkInitializationArguments(
        apiId: apiId,
        apiHash: apiHash,
        languagesCategory: languagesCategory,
        appVersion: appVersion,
        voipMaxLayer: PresentationCallManagerImpl.voipMaxLayer,
        voipVersions: PresentationCallManagerImpl.voipVersions(...),
        appData: self.regularDeviceToken.get() |> map { token in
            // Encode APNS token + build signatures into app data
            return buildConfig.bundleData(withAppToken: token, ...)
        },
        externalRequestVerificationStream: self.firebaseRequestVerificationSecretStream.get(),
        encryptionProvider: OpenSSLEncryptionProvider(),
        // ...
    )
```

Notice how signals (`self.regularDeviceToken.get()`) are threaded into the network arguments — the APNS device token isn't available at launch, so it's provided as a reactive stream that the network layer subscribes to. When the token arrives later, the network layer automatically picks it up. This is SwiftSignalKit in action.

### Phase 3: Storage and File System (Lines 629-689)

```swift
    guard let appGroupUrl = maybeAppGroupUrl else {
        self.mainWindow?.presentNative(UIAlertController(title: nil, message: "Error 2", preferredStyle: .alert))
        return true
    }

    let rootPath = rootPathForBasePath(appGroupUrl.path)
    performAppGroupUpgrades(appGroupPath: appGroupUrl.path, rootPath: rootPath)

    let deviceSpecificEncryptionParameters = BuildConfig.deviceSpecificEncryptionParameters(rootPath, baseAppBundleId: baseAppBundleId)
    let encryptionParameters = ValueBoxEncryptionParameters(
        forceEncryptionIfNoSet: false,
        key: ValueBoxEncryptionParameters.Key(data: deviceSpecificEncryptionParameters.key)!,
        salt: ValueBoxEncryptionParameters.Salt(data: deviceSpecificEncryptionParameters.salt)!
    )

    TempBox.initializeShared(basePath: rootPath, processType: "app", launchSpecificId: Int64.random(in: Int64.min ... Int64.max))
```

Everything lives in the **App Group container** — this is shared storage that both the main app and extensions (Share, Notifications, Watch, Widgets) can access. The encryption parameters are device-specific, derived from the Keychain, ensuring the SQLite database (Postbox) is encrypted at rest.

There's even a **disk write test** — Telegram writes 1MB of random data to verify the device has enough storage:

```swift
    let writeAbilityTestFile = TempBox.shared.tempFile(fileName: "test.bin")
    var writeAbilityTestSuccess = true
    if let testFile = ManagedFile(queue: nil, path: writeAbilityTestFile.path, mode: .readwrite) {
        let bufferSize = 128 * 1024
        let randomBuffer = malloc(bufferSize)!
        // Write 1MB in 128KB chunks
        while writtenBytes < 1024 * 1024 {
            let actualBytes = testFile.write(randomBuffer, count: bufferSize)
            // ...
        }
    }

    if !writeAbilityTestSuccess {
        // Show "insufficient space" alert and crash
        self.mainWindow?.presentNative(UIAlertController(title: nil, message: "The device does not have sufficient free space.", ...))
        return true
    }
```

This is defensive programming at its finest. Rather than discovering a full disk mid-operation (which could corrupt the database), Telegram checks upfront and fails gracefully.

### Phase 4: TelegramApplicationBindings (Lines 771-984)

This is one of the most architecturally interesting parts. `TelegramApplicationBindings` is a struct of **closures** that injects platform capabilities into the business logic:

```swift
    let applicationBindings = TelegramApplicationBindings(
        isMainApp: true,
        appBundleId: baseAppBundleId,
        appBuildType: buildConfig.isAppStoreBuild ? .public : .internal,
        containerPath: appGroupUrl.path,
        appSpecificScheme: buildConfig.appSpecificUrlScheme,
        openUrl: { url in
            UIApplication.shared.open(parsedUrl, options: [:], completionHandler: nil)
        },
        openUniversalUrl: { url, completion in
            UIApplication.shared.open(parsedUrl, options: [.universalLinksOnly: true], completionHandler: { value in
                completion.completion(value)
            })
        },
        canOpenUrl: { url in
            return UIApplication.shared.canOpenURL(parsedUrl)
        },
        applicationInForeground: self.isInForegroundPromise.get(),
        applicationIsActive: self.isActivePromise.get(),
        clearMessageNotifications: { ids in
            for id in ids { self.clearNotificationsManager?.append(id) }
        },
        pushIdleTimerExtension: { /* disable screen timeout */ },
        openSettings: { UIApplication.shared.open(URL(string: UIApplication.openSettingsURLString)!) },
        registerForNotifications: { completion in /* ... */ },
        requestSiriAuthorization: { completion in /* ... */ },
        forceOrientation: { orientation in /* ... */ },
        // ... 20+ more closures
    )
```

This is the **dependency injection strategy** for the entire app. `TelegramCore` and `AccountContext` are platform-agnostic modules — they don't import UIKit. But they need to open URLs, check notification permissions, manage the idle timer, etc. Instead of importing UIKit, they accept these capabilities as closures bundled in `TelegramApplicationBindings`.

This means:
- The Share extension can provide different bindings (it can't open URLs the same way).
- The Watch extension provides its own bindings.
- Testing could provide mock bindings.

Key closures to understand:

| Closure | What it does |
|---------|-------------|
| `applicationInForeground` | `Signal<Bool, NoError>` — reactive stream of foreground state |
| `applicationIsActive` | `Signal<Bool, NoError>` — reactive stream of active state |
| `openUrl` | Opens external URLs via `UIApplication.shared.open` |
| `registerForNotifications` | Triggers APNS registration |
| `pushIdleTimerExtension` | Returns a `Disposable` that keeps the screen on while held |
| `getAvailableAlternateIcons` | Returns the list of app icons |
| `forceOrientation` | Forces device orientation (for media viewing) |

### Phase 5: AccountManager and SharedAccountContext (Lines 986-1050)

Now the core initialization:

```swift
    let accountManager = AccountManager<TelegramAccountManagerTypes>(
        basePath: rootPath + "/accounts-metadata",
        isTemporary: false,
        isReadOnly: false,
        useCaches: true,
        removeDatabaseOnError: true
    )
    self.accountManager = accountManager

    telegramUIDeclareEncodables()
    initializeAccountManagement()
```

`AccountManager` is a Postbox-backed database that manages **multiple accounts**. It stores account records with their IDs, attributes (notification keys, backup data), and ordering. This is the multi-account system — Telegram supports logging into multiple accounts simultaneously.

Then comes the `SharedAccountContextImpl` — the central coordinator:

```swift
    let sharedContext = SharedAccountContextImpl(
        mainWindow: self.mainWindow,
        sharedContainerPath: legacyBasePath,
        basePath: rootPath,
        encryptionParameters: encryptionParameters,
        accountManager: accountManager,
        appLockContext: appLockContext,
        applicationBindings: applicationBindings,
        initialPresentationDataAndSettings: initialPresentationDataAndSettings,
        networkArguments: networkArguments,
        rootPath: rootPath,
        legacyBasePath: legacyBasePath,
        apsNotificationToken: self.notificationTokenPromise.get() |> map(Optional.init),
        voipNotificationToken: self.voipTokenPromise.get() |> map(Optional.init),
        // ...
    )
```

`SharedAccountContextImpl` is arguably the **most important object in the entire app**. It coordinates:
- All active accounts and their lifecycle
- Presentation data (themes, strings, locale)
- The call manager
- The media manager
- The contact data manager
- Navigation and presentation
- App lock and passcode
- Account switching

This is the object that UI code receives (via the `SharedAccountContext` protocol) to access cross-cutting concerns.

## The Context Split: Authorized vs. Unauthorized

This is the most distinctive architectural pattern in the Telegram app. The application doesn't just have "a state" — it has two fundamentally different contexts:

### UnauthorizedApplicationContext

When no account is logged in (or during the login flow):

```swift
// submodules/TelegramUI/Sources/ApplicationContext.swift
final class UnauthorizedApplicationContext {
    let sharedContext: SharedAccountContextImpl
    let account: UnauthorizedAccount
    let rootController: AuthorizationSequenceController
    let isReady = Promise<Bool>()

    init(apiId: Int32, apiHash: String, sharedContext: SharedAccountContextImpl,
         account: UnauthorizedAccount,
         otherAccountPhoneNumbers: ((String, AccountRecordId, Bool)?, [(String, AccountRecordId, Bool)])) {

        self.rootController = AuthorizationSequenceController(
            sharedContext: sharedContext,
            account: account,
            otherAccountPhoneNumbers: otherAccountPhoneNumbers,
            presentationData: presentationData,
            openUrl: sharedContext.applicationBindings.openUrl,
            apiId: apiId,
            apiHash: apiHash,
            authorizationCompleted: {
                authorizationCompleted?()
            }
        )

        // The unauthorized account's network activity is tied to foreground state
        account.shouldBeServiceTaskMaster.set(
            sharedContext.applicationBindings.applicationInForeground
            |> map { value -> AccountServiceTaskMasterMode in
                if value { return .always } else { return .never }
            }
        )
    }
}
```

The root controller is `AuthorizationSequenceController` — a custom navigation controller that manages the phone number entry, code verification, 2FA, and registration steps. The key insight is that this context wraps an `UnauthorizedAccount` (from TelegramCore), which has limited network capabilities — it can only make auth-related API calls.

### AuthorizedApplicationContext

When a user is logged in:

```swift
final class AuthorizedApplicationContext {
    let sharedApplicationContext: SharedApplicationContext
    let mainWindow: Window1
    let lockedCoveringView: LockedWindowCoveringView
    let context: AccountContextImpl
    let rootController: TelegramRootController
    let notificationController: NotificationContainerController

    // Disposables for reactive subscriptions
    private let passcodeStatusDisposable = MetaDisposable()
    private let loggedOutDisposable = MetaDisposable()
    private let inAppNotificationSettingsDisposable = MetaDisposable()
    private let notificationMessagesDisposable = MetaDisposable()
    private let termsOfServiceUpdatesDisposable = MetaDisposable()
    private let permissionsDisposable = MetaDisposable()
    // ... more disposables

    var passcodeController: PasscodeEntryController?
    let isReady = Promise<Bool>()
```

The `init` method creates the root UI:

```swift
    init(sharedApplicationContext: SharedApplicationContext, mainWindow: Window1,
         context: AccountContextImpl, accountManager: AccountManager<TelegramAccountManagerTypes>,
         showCallsTab: Bool, reinitializedNotificationSettings: @escaping () -> Void) {

        self.context = context
        self.notificationController = NotificationContainerController(context: context)
        self.rootController = TelegramRootController(context: context)

        // Add the tab controllers immediately
        if self.rootController.rootTabController == nil {
            self.rootController.addRootControllers(showCallsTab: self.showCallsTab)
        }

        // Track readiness: wait for both the tab controller and its selected child to be ready
        if let tabsController = self.rootController.viewControllers.first as? TabBarController,
           !tabsController.controllers.isEmpty {
            let controller = tabsController.controllers[tabsController.selectedIndex]
            let combinedReady = combineLatest(tabsController.ready.get(), controller.ready.get())
            |> map { $0 && $1 }
            |> filter { $0 }
            |> take(1)
            self.isReady.set(combinedReady)
        }
```

Notice the **readiness pattern**: the context isn't considered "ready" until both the tab controller AND its selected child controller report ready via `Promise<Bool>`. This gates the transition from the launch screen to the main UI — no flash of empty content.

The `AuthorizedApplicationContext` also subscribes to many reactive streams:

- **Passcode lock state**: Shows/hides the passcode screen when the app enters background.
- **Logged-out signal**: If the server invalidates the session, this triggers automatic logout.
- **In-app notification settings**: Watches for changes to notification preferences.
- **Incoming notifications**: Displays in-app notification banners.
- **Terms of service updates**: Shows ToS acceptance screen when required.
- **Permission requests**: Manages permission request flows.

Each of these is a SwiftSignalKit `Signal` subscription stored in a `MetaDisposable` — the standard pattern for lifecycle-managed reactive subscriptions.

## SharedApplicationContext: The God Object

Between the AppDelegate and the application contexts sits `SharedApplicationContext`:

```swift
// submodules/TelegramUI/Sources/AppDelegate.swift
final class SharedApplicationContext {
    let sharedContext: SharedAccountContextImpl
    let notificationManager: SharedNotificationManager
    let wakeupManager: SharedWakeupManager
    let overlayMediaController: ViewController & OverlayMediaController
    var minimizedContainer: [AccountRecordId: MinimizedContainer] = [:]
}
```

This is a thin wrapper that groups the `SharedAccountContextImpl` with managers that need to persist across account switches:
- **SharedNotificationManager**: Handles push notification display across all accounts.
- **SharedWakeupManager**: Manages background task execution (location updates, etc.).
- **OverlayMediaController**: The floating media player (for picture-in-picture playback that persists while navigating).
- **MinimizedContainer per account**: Tracks minimized web apps/browsers for each account.

## TelegramRootController: Assembling the Tab Bar

Once `AuthorizedApplicationContext` creates `TelegramRootController`, the tab bar is assembled:

```swift
// submodules/TelegramUI/Sources/TelegramRootController.swift
public final class TelegramRootController: NavigationController, TelegramRootControllerInterface {
    private let context: AccountContext

    public var rootTabController: TabBarController?
    public var contactsController: ContactsController?
    public var callListController: CallListController?
    public var chatListController: ChatListController?
    public var accountSettingsController: PeerInfoScreen?

    public init(context: AccountContext) {
        self.context = context
        self.presentationData = context.sharedContext.currentPresentationData.with { $0 }

        // automaticMasterDetail enables iPad split view
        super.init(mode: .automaticMasterDetail, theme: NavigationControllerTheme(presentationTheme: self.presentationData.theme))

        // Subscribe to theme changes
        self.presentationDataDisposable = (context.sharedContext.presentationData
        |> deliverOnMainQueue).startStrict(next: { [weak self] presentationData in
            if let strongSelf = self {
                let previousTheme = strongSelf.presentationData.theme
                strongSelf.presentationData = presentationData
                if previousTheme !== presentationData.theme {
                    (strongSelf.rootTabController as? TabBarControllerImpl)?.updateTheme(theme: presentationData.theme)
                }
            }
        })
    }
```

The root controller extends `NavigationController` with `.automaticMasterDetail` mode — on iPad, this gives you a sidebar/detail split view automatically. On iPhone, it's a standard navigation stack.

The `addRootControllers` method creates the four tabs:

```swift
    public func addRootControllers(showCallsTab: Bool) {
        let tabBarController = TabBarControllerImpl(theme: self.presentationData.theme, strings: self.presentationData.strings)
        tabBarController.navigationPresentation = .master

        // 1. Chat List (the main tab)
        let chatListController = self.context.sharedContext.makeChatListController(
            context: self.context,
            location: .chatList(groupId: .root),
            controlsHistoryPreload: true,
            hideNetworkActivityStatus: false,
            previewing: false,
            enableDebugActions: !GlobalExperimentalSettings.isAppStoreBuild
        )

        // 2. Call List
        let callListController = CallListController(context: self.context, mode: .tab)

        // 3. Contacts
        let contactsController = ContactsController(context: self.context)
        contactsController.switchToChatsController = { [weak self] in
            self?.openChatsController(activateSearch: false)
        }

        // 4. Settings (implemented as PeerInfoScreen for the current user)
        let accountSettingsController = PeerInfoScreenImpl(
            context: self.context,
            peerId: self.context.account.peerId,
            isSettings: true
        )

        // Assemble the tabs
        var controllers: [ViewController] = []
        controllers.append(contactsController)
        if showCallsTab {
            controllers.append(callListController)
        }
        controllers.append(chatListController)
        controllers.append(accountSettingsController)

        tabBarController.setControllers(controllers, selectedIndex: controllers.count - 2) // Select chat list

        self.rootTabController = tabBarController
        self.pushViewController(tabBarController, animated: false)
    }
```

Key observations:

1. **Chat list is not a separate controller class from TelegramUI** — it's created through `self.context.sharedContext.makeChatListController(...)`. This is another factory pattern at the module boundary — `ChatListUI` module provides the implementation, `AccountContext` module defines the interface.

2. **Settings is a PeerInfoScreen** — Telegram reuses its user profile screen for settings by passing `isSettings: true` and the current user's `peerId`. This is clever code reuse — the settings page IS your own profile, with some additional sections.

3. **The calls tab is optional** — controlled by `showCallsTab`, which reflects a user preference. The `updateRootControllers` method handles toggling this at runtime.

4. **The selected index defaults to the chat list** (`controllers.count - 2`), which is the second-to-last tab.

## The Complete Launch Sequence

Putting it all together, here's the full boot timeline:

```
1. main.m              → UIApplicationMain("Application", "AppDelegate")
2. Application.swift   → Custom UIApplication created (sendEvent hook)
3. AppDelegate         → didFinishLaunchingWithOptions:
   a. Factory closures   → Set up navigation bar, context menu implementations
   b. Window1            → Custom window with Metal engine root layer
   c. BuildConfig        → Read API credentials from build configuration
   d. NetworkArguments   → Configure MTProto with API ID, tokens, encryption
   e. Storage            → App Group container, encryption keys, disk write test
   f. Logging            → Set up file-based logging
   g. Bindings           → Create TelegramApplicationBindings (20+ closures)
   h. AccountManager     → Open multi-account database
   i. PresentationData   → Load initial theme, strings, locale settings
   j. SharedAccountContext → Create the central coordinator
   k. Account loading    → Open the current account's Postbox database
4. Account switch       → Based on account state:
   ├── No account       → Create UnauthorizedApplicationContext
   │                     → Show AuthorizationSequenceController (login)
   └── Has account      → Create AuthorizedApplicationContext
                         → Create TelegramRootController
                         → addRootControllers (Contacts, Calls, ChatList, Settings)
                         → Wait for readiness signals
                         → Display main UI
```

## Architectural Takeaways

### 1. The Closure-Based DI Pattern

Instead of protocols with many methods (which require full conformance), Telegram uses structs of closures (`TelegramApplicationBindings`). Each closure is independently injectable, and the compiler enforces that all are provided at construction.

### 2. Readiness Gates

The app doesn't show UI until it's ready. `Promise<Bool>` signals gate transitions between states. This prevents flashes of empty content and ensures smooth handoffs between the launch screen and the main UI.

### 3. Authorized/Unauthorized State Machine

The app is fundamentally a state machine with two states. Each state has its own context, root controller, and available capabilities. This keeps the auth flow and main app cleanly separated.

### 4. MetaDisposable for Lifecycle Management

Every reactive subscription is stored in a `MetaDisposable`. When the context is deallocated, all disposables are automatically cleaned up. When a subscription needs to be replaced (e.g., switching accounts), `MetaDisposable.set()` atomically swaps the old subscription for the new one.

### 5. Factory Closures at Module Boundaries

Global factory closures (`defaultNavigationBarImpl`, `makeContextControllerImpl`) break circular dependencies between modules. The `Display` module defines what it needs; the `TelegramUI` module provides the implementations at launch.

## What's Next

In the [next post](/posts/03-swift-signal-kit/), we'll dive deep into SwiftSignalKit — the reactive framework that powers every async operation in the codebase. Understanding `Signal`, `Promise`, `Disposable`, and the `|>` pipe operator is essential before we can read any of the business logic or UI code.

---

*This post covers: `Telegram/Telegram-iOS/main.m`, `Telegram/Telegram-iOS/Application.swift`, `submodules/TelegramUI/Sources/AppDelegate.swift`, `submodules/TelegramUI/Sources/ApplicationContext.swift`, `submodules/TelegramUI/Sources/TelegramRootController.swift`*
