---
title: "Extensions: Six Processes Sharing One Database"
description: "A deep dive into Telegram iOS's six app extensions — Share, NotificationService, NotificationContent, Widget, SiriIntents, and BroadcastUpload. Covers how each extension bootstraps a minimal Telegram stack from the shared App Group container, the encryption parameter derivation pattern every extension repeats, the AccountAuxiliaryMethods stub that disables features extensions don't need, and the architectural constraints of running TelegramCore outside the main app process."
published: 2025-08-04
tags:
  - Build & Extensions
toc: true
lang: en
abbrlink: 24-extensions
pin: 2
---

iOS extensions run as separate processes. They have their own address space, their own memory limits (far lower than the main app), and no direct communication channel to the host app at runtime. Yet Telegram ships six extensions that all need to read the same encrypted database, render the same avatars, and speak the same protocol. This post explains how they pull it off.

## The App Group Contract

Every Telegram extension begins the same way — deriving the shared container path from the bundle identifier:

```swift
let appBundleIdentifier = Bundle.main.bundleIdentifier!
guard let lastDotRange = appBundleIdentifier.range(of: ".", options: [.backwards]) else {
    return
}
let baseAppBundleId = String(appBundleIdentifier[..<lastDotRange.lowerBound])

let appGroupName = "group.\(baseAppBundleId)"
let maybeAppGroupUrl = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: appGroupName
)
```

The extension's bundle ID is always `{baseAppBundleId}.{ExtensionName}` — so `ph.telegra.Telegraph.Share`, `ph.telegra.Telegraph.NotificationService`, etc. The code strips the last component to recover the base app bundle ID, then constructs the App Group identifier `group.ph.telegra.Telegraph`.

The shared container has a fixed layout:

```
{AppGroupContainer}/
├── telegram-data/
│   ├── accounts-metadata/       ← AccountManager SQLite
│   ├── account-{id}/
│   │   ├── postbox/
│   │   │   ├── db/              ← Postbox SQLite (encrypted)
│   │   │   └── media/           ← MediaBox file cache
│   │   └── ...
│   ├── logs/
│   │   ├── widget-logs/
│   │   └── siri-logs/
│   └── temp/
├── broadcast-coordination/      ← IPC for screen sharing
└── embedded-broadcast-coordination/
```

## The Bootstrap Pattern

Every extension repeats the same initialization sequence. It's worth understanding this pattern in full because it reveals the minimum viable Telegram stack:

**Step 1: Derive encryption parameters.** The Postbox database is encrypted with AES. The key and salt are derived from the device and bundle ID:

```swift
let rootPath = appGroupUrl.path + "/telegram-data"
let deviceSpecificEncryptionParameters = BuildConfig.deviceSpecificEncryptionParameters(
    rootPath, baseAppBundleId: baseAppBundleId
)
let encryptionParameters: (Data, Data) = (
    deviceSpecificEncryptionParameters.key,
    deviceSpecificEncryptionParameters.salt
)
```

`BuildConfig.deviceSpecificEncryptionParameters` uses the Keychain to store a randomly generated key tied to the device. If the key doesn't exist yet, it creates one. Both the main app and all extensions derive the same key because they share the same Keychain access group (via the App Group entitlement).

**Step 2: Create an `AccountManager`.** The AccountManager reads `accounts-metadata/` to discover which accounts exist:

```swift
let accountManager = AccountManager<TelegramAccountManagerTypes>(
    basePath: rootPath + "/accounts-metadata",
    isTemporary: true,
    isReadOnly: false,
    useCaches: false,
    removeDatabaseOnError: false
)
```

Notice `isTemporary: true` and `useCaches: false`. Extensions don't maintain long-lived caches — they open the database, read what they need, and close.

**Step 3: Create stub `AccountAuxiliaryMethods`.** The main app registers real implementations for resource fetching, thumbnail generation, and background uploads. Extensions don't need any of that:

```swift
private let accountAuxiliaryMethods = AccountAuxiliaryMethods(
    fetchResource: { account, resource, ranges, _ in return nil },
    fetchResourceMediaReferenceHash: { resource in return .single(nil) },
    prepareSecretThumbnailData: { _ in return nil },
    backgroundUpload: { _, _, _ in return .single(nil) }
)
```

Every extension defines this exact same stub. It returns `nil` for everything — extensions never download media, never prepare thumbnails, never upload in the background. They only read data the main app has already fetched.

**Step 4: Initialize TempBox and logging.**

```swift
TempBox.initializeShared(
    basePath: rootPath,
    processType: "widget",  // or "siri", etc.
    launchSpecificId: Int64.random(in: Int64.min ... Int64.max)
)

let logsPath = rootPath + "/logs/widget-logs"
setupSharedLogger(rootPath: logsPath, path: logsPath)
```

Each extension gets its own log directory. The `processType` string ensures temp files don't collide between processes.

## Extension 1: Share

**Source:** `Telegram/Share/ShareRootController.swift` (120 lines)

The Share Extension is remarkably thin. `ShareRootController` is a `UIViewController` that serves as the extension's principal class, but it immediately delegates to `ShareRootControllerImpl` — a type defined in the `ShareExtensionContext` module inside TelegramUI:

```swift
@objc(ShareRootController)
class ShareRootController: UIViewController {
    private var impl: ShareRootControllerImpl?

    override func loadView() {
        super.loadView()

        // ... derive baseAppBundleId, buildConfig, appGroupUrl, encryptionParameters ...

        self.impl = ShareRootControllerImpl(
            initializationData: ShareRootControllerInitializationData(
                appBundleId: baseAppBundleId,
                appBuildType: buildConfig.isAppStoreBuild ? .public : .internal,
                appGroupPath: appGroupUrl.path,
                apiId: buildConfig.apiId,
                apiHash: buildConfig.apiHash,
                languagesCategory: languagesCategory,
                encryptionParameters: encryptionParameters,
                appVersion: appVersion,
                bundleData: buildConfig.bundleData(withAppToken: nil, ...),
                useBetaFeatures: !buildConfig.isAppStoreBuild,
                makeTempContext: { accountManager, appLockContext, ... in
                    return makeTempContext(
                        sharedContainerPath: appGroupUrl.path,
                        rootPath: rootPath,
                        // ...
                    )
                }
            ),
            getExtensionContext: { [weak self] in
                return self?.extensionContext
            }
        )
    }
}
```

Why this split? The Share extension's BUILD target only compiles `ShareRootController.swift` directly. The heavy lifting — account selection UI, peer picker, message sending, file attachment handling — lives in `ShareExtensionContext`, which is a component inside TelegramUI. This means the Share extension gets the full Telegram UI framework (via `TelegramUIFramework`) but keeps its own source minimal.

The `makeTempContext` closure is the key architectural bridge. It creates a temporary `AccountContext` — the same protocol the main app uses for dependency injection — but configured for extension use: no background task management, no WebSocket connection, no state synchronization. Just enough to read from Postbox, resolve peers, and send messages.

One clever detail: the `openUrl` handler walks the responder chain to find `UIApplication` and call `open(_:)`, which in an extension context triggers the system to open the URL in the main app:

```swift
self.impl?.openUrl = { [weak self] url in
    guard let self, let url = URL(string: url) else { return }
    let _ = self.openURL(url)
}

@objc func openURL(_ url: URL) -> Bool {
    var responder: UIResponder? = self
    while responder != nil {
        if let application = responder as? UIApplication {
            application.open(url, options: [:], completionHandler: nil)
            return true
        }
        responder = responder?.next
    }
    return false
}
```

**Supported content types:** The Share extension accepts a broad range — files, movies, images, URLs, text, audio, vCards, Apple Wallet passes. The activation rule in the Info.plist uses a complex `SUBQUERY` predicate over UTI conformance.

## Extension 2: NotificationService

**Source:** `Telegram/NotificationService/Sources/NotificationService.swift` (~175KB, the largest extension by far)

This is the most complex extension. It's a `UNNotificationServiceExtension` that intercepts push notifications before they're displayed, decrypts the payload, and enriches the notification with:

- Decrypted message text
- Sender name and avatar
- Inline media thumbnails
- Lottie sticker first-frame renders
- Opus-to-AAC audio conversion
- Communication notification formatting (for iOS 15+ notification management)

The file is enormous because it inlines many utilities that the main app gets from framework modules. Extensions have a ~50MB memory limit, so the NotificationService can't afford to load the full TelegramUI framework. Instead, it depends on a carefully curated subset:

```
Postbox, TelegramCore, MtProtoKit, SwiftSignalKit,
OpenSSLEncryptionProvider, WebPBinding, RLottieBinding,
GZip, AppLockState, NotificationsPresentationData,
TelegramUIPreferences, ConvertOpusToAAC, PersistentStringHash
```

Notice what's *missing*: no AsyncDisplayKit, no Display module, no ComponentFlow, no TelegramUI. This extension never shows UI — it processes data and modifies `UNNotificationContent`.

The file even includes its own `DrawingContext` class — a self-contained Core Graphics wrapper for rendering avatars and sticker thumbnails — because importing the Display module would pull in too many dependencies:

```swift
private final class DrawingContext {
    let size: CGSize
    let scale: CGFloat
    let scaledSize: CGSize
    let bytesPerRow: Int
    private let bitmapInfo: CGBitmapInfo
    let length: Int
    let bytes: UnsafeMutableRawPointer
    private let data: Data
    private let context: CGContext

    init(size: CGSize, scale: CGFloat = 1.0, opaque: Bool = false, clear: Bool = false) {
        // Manual bitmap context allocation with DeviceGraphicsContextSettings
        self.bytes = malloc(self.length)
        self.context = CGContext(
            data: self.bytes,
            width: Int(self.scaledSize.width),
            height: Int(self.scaledSize.height),
            bitsPerComponent: 8,
            bytesPerRow: self.bytesPerRow,
            space: deviceColorSpace,
            bitmapInfo: self.bitmapInfo.rawValue,
            releaseCallback: nil,
            releaseInfo: nil
        )!
    }
}
```

It also includes `DeviceGraphicsContextSettings` for P3 color space detection, `convertLottieImage` for rendering a single Lottie animation frame, and `avatarRoundImage`/`avatarViewLettersImage` for generating circular avatar images — all duplicated from code that exists in the main app's Display module.

The `accountAuxiliaryMethods` stub is identical to every other extension:

```swift
private let accountAuxiliaryMethods = AccountAuxiliaryMethods(
    fetchResource: { account, resource, ranges, _ in return nil },
    fetchResourceMediaReferenceHash: { resource in return .single(nil) },
    prepareSecretThumbnailData: { _ in return nil },
    backgroundUpload: { _, _, _ in return .single(nil) }
)
```

The extension has at most ~30 seconds to process a notification before iOS kills it. That's enough time to decrypt the payload and query Postbox for the sender's name and avatar, but not enough to download anything new from the network.

## Extension 3: NotificationContent

**Source:** `Telegram/NotificationContent/NotificationViewController.swift` (59 lines)

The `UNNotificationContentExtension` is the simplest extension. It displays rich notification UI when the user long-presses a notification. The view controller is essentially empty:

```swift
class NotificationViewController: UIViewController, UNNotificationContentExtension {
    // Minimal implementation - the heavy lifting is in TelegramUI
}
```

It depends on the full `TelegramUIFramework` because it needs to render the actual notification content UI. Its Info.plist declares support for two notification categories: `withReplyMedia` and `withMuteMedia`, with a content size ratio of `0.0001` (minimal height until content is loaded).

## Extension 4: Widget (WidgetKit)

**Source:** `Telegram/WidgetKitWidget/TodayViewController.swift` (~38KB)

The Widget extension displays recent chats on the home screen. It follows the standard WidgetKit `TimelineProvider` pattern:

```swift
@available(iOS 14.0, *)
private func getCommonTimeline(
    friends: [Friend]?,
    in context: TimelineProviderContext,
    completion: @escaping (Timeline<SimpleEntry>) -> ()
) {
    if context.isPreview {
        completion(Timeline(
            entries: [SimpleEntry(date: Date(), contents: .preview)],
            policy: .atEnd
        ))
        return
    }

    // ... bootstrap: derive appGroupUrl, rootPath, encryption params ...

    TempBox.initializeShared(
        basePath: rootPath,
        processType: "widget",
        launchSpecificId: Int64.random(in: Int64.min ... Int64.max)
    )
}
```

The widget reads peer data through the `WidgetItems` module, which defines `WidgetDataPeer` and `WidgetDataPeers` — lightweight data structures the main app writes to the shared container. The widget doesn't open Postbox directly for timeline updates; instead, it reads serialized peer data that the main app has pre-computed.

For Siri Intent configuration (when the user edits the widget to choose specific contacts), the widget *does* open Postbox via `accountTransaction`:

```swift
accountTransaction(
    rootPath: rootPath,
    id: accountId,
    encryptionParameters: encryptionParameters,
    isReadOnly: true,
    useCopy: false,
    transaction: { postbox, transaction -> [Friend] in
        var peers: [Peer] = []
        for renderedPeer in transaction.getTopChatListEntries(groupId: .root, count: 50) {
            if let peer = renderedPeer.peer,
               !(peer is TelegramSecretChat),
               !peer.isDeleted {
                peers.append(peer)
            }
        }
        return mapPeersToFriends(
            accountId: accountId,
            accountPeerId: accountPeerId,
            mediaBox: postbox.mediaBox,
            peers: peers
        )
    }
)
```

Notice `isReadOnly: true` — the widget never writes to Postbox. It uses the `transaction.getTopChatListEntries` API to fetch the top 50 chats and maps them to `Friend` objects (an Intents framework type) with pre-rendered avatar images.

The avatar rendering pipeline in the widget is substantial. It generates circular avatar images with gradient letter backgrounds (for contacts without photos), caches them in the MediaBox's cached representations system, and serves them as `INImage` instances. Each avatar is rendered at 50x50 points, with P3 color space support.

**Widget dependencies** are deliberately minimal: `Postbox`, `TelegramCore`, `SwiftSignalKit`, `WidgetItems`, `WidgetItemsUtils`, `AppLockState`, `OpenSSLEncryptionProvider`. No TelegramUI framework.

## Extension 5: SiriIntents

**Source:** `Telegram/SiriIntents/IntentHandler.swift` (1,398 lines)

The Siri extension handles voice commands: "Send a message to John on Telegram," "Call John on Telegram," "Read my unread messages." It's the second most complex extension.

The `IntentHandler` class is the entry point. It routes intents to specialized handlers based on type:

```swift
@objc(IntentHandler)
class IntentHandler: INExtension {
    override public func handler(for intent: INIntent) -> Any {
        if #available(iOSApplicationExtension 12.0, iOS 12.0, *) {
            if intent is SelectAvatarFriendsIntent {
                return AvatarsIntentHandler()
            } else if intent is SelectFriendsIntent {
                return FriendsIntentHandler()
            } else {
                return DefaultIntentHandler()
            }
        } else {
            return DefaultIntentHandler()
        }
    }
}
```

`DefaultIntentHandler` conforms to five intent handling protocols simultaneously:

```swift
class DefaultIntentHandler: INExtension,
    INSendMessageIntentHandling,
    INSearchForMessagesIntentHandling,
    INSetMessageAttributeIntentHandling,
    INStartCallIntentHandling,
    INSearchCallHistoryIntentHandling
{
    private let accountPromise = Promise<Account?>()
    private let allAccounts = Promise<[(AccountRecordId, PeerId, Bool)]>()

    private let resolvePersonsDisposable = MetaDisposable()
    private let actionDisposable = MetaDisposable()
    private let searchDisposable = MetaDisposable()
```

The intent handler maintains an actual `Account` object — not just Postbox read access, but a full account with network capabilities. This is because `INSendMessageIntent.handle` needs to actually send a message through the Telegram API:

```swift
public func handle(intent: INSendMessageIntent, completion: @escaping (INSendMessageIntentResponse) -> Void) {
    self.actionDisposable.set((self.accountPromise.get()
    |> castError(IntentHandlingError.self)
    |> take(1)
    |> mapToSignal { account -> Signal<Void, IntentHandlingError> in
        guard let account = account else { return .fail(.generic) }
        guard let recipient = intent.recipients?.first,
              let customIdentifier = recipient.customIdentifier,
              customIdentifier.hasPrefix("tg") else {
            return .fail(.generic)
        }

        let peerIdValue = Int64(String(
            customIdentifier[customIdentifier.index(customIdentifier.startIndex, offsetBy: 2)...]
        ))!
        let peerId = PeerId(peerIdValue)

        account.shouldBeServiceTaskMaster.set(.single(.now))
        return standaloneSendMessage(
            account: account,
            peerId: peerId,
            text: intent.content ?? "",
            attributes: [],
            media: nil,
            replyToMessageId: nil
        )
        |> afterDisposed {
            account.shouldBeServiceTaskMaster.set(.single(.never))
        }
    }
    |> deliverOnMainQueue).start(error: { _ in
        completion(INSendMessageIntentResponse(code: .failureRequiringAppLaunch, ...))
    }, completed: {
        completion(INSendMessageIntentResponse(code: .success, ...))
    }))
}
```

The `shouldBeServiceTaskMaster` toggle is critical. It tells the account's network layer to temporarily become active (`.now`), perform the send, then go back to sleep (`.never`). Without this, the extension's Account would never establish a network connection because it's not the "service task master" (that's normally the main app).

**Contact resolution** uses a pipeline that bridges iOS Contacts framework identifiers to Telegram peer IDs:

1. Siri provides `INPerson` objects with `contactIdentifier` (Apple Contacts ID)
2. `matchingDeviceContacts` queries the local address book
3. `matchingCloudContacts` queries Postbox for Telegram users matching those phone numbers
4. Results map back to `INPerson` with a `customIdentifier` of `"tg{peerId}"`

**App lock awareness:** Every handler method checks for app lock state before proceeding. If the app is locked with a passcode, the extension returns `.failureRequiringAppLaunch`:

```swift
if let data = try? Data(contentsOf: URL(fileURLWithPath: appLockStatePath(rootPath: rootPath))),
   let state = try? JSONDecoder().decode(LockState.self, from: data),
   isAppLocked(state: state) {
    completion(INSendMessageIntentResponse(code: .failureRequiringAppLaunch, ...))
    return
}
```

The lock state is stored as a JSON file in the shared container, not in UserDefaults — because `AppLockState` needs to be readable by all extensions without any additional framework overhead.

## Extension 6: BroadcastUpload

**Source:** `Telegram/BroadcastUpload/BroadcastUploadExtension.swift` (~23KB)

The screen sharing extension uses iOS's `RPBroadcastSampleHandler` to capture the screen during group calls. It has two operating modes, abstracted behind a `BroadcastUploadImpl` protocol:

```swift
private protocol BroadcastUploadImpl: AnyObject {
    func initialize(rootPath: String)
    func processVideoSampleBuffer(sampleBuffer: CMSampleBuffer)
    func processAudioSampleBuffer(data: Data)
}
```

**Mode 1: `InProcessBroadcastUploadImpl`** — Serializes video frames to shared memory and hands them to the main app process via `IpcGroupCallBufferBroadcastContext`:

```swift
private final class InProcessBroadcastUploadImpl: BroadcastUploadImpl {
    private var screencastBufferClientContext: IpcGroupCallBufferBroadcastContext?

    func initialize(rootPath: String) {
        let screencastBufferClientContext = IpcGroupCallBufferBroadcastContext(
            basePath: rootPath + "/broadcast-coordination"
        )
        self.screencastBufferClientContext = screencastBufferClientContext

        self.statusDisposable = (screencastBufferClientContext.status
        |> deliverOnMainQueue).start(next: { [weak self] status in
            switch status {
            case .active:
                wasRunning = true
            case let .finished(reason):
                self?.finish(with: reason)
            }
        })
    }

    func processVideoSampleBuffer(sampleBuffer: CMSampleBuffer) {
        guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else { return }
        var orientation = CGImagePropertyOrientation.up
        if let orientationAttachment = CMGetAttachment(
            sampleBuffer, key: RPVideoSampleOrientationKey as CFString, attachmentModeOut: nil
        ) as? NSNumber {
            orientation = CGImagePropertyOrientation(rawValue: orientationAttachment.uint32Value) ?? .up
        }
        if let data = serializePixelBuffer(buffer: pixelBuffer) {
            self.screencastBufferClientContext?.setCurrentFrame(data: data, orientation: orientation)
        }
    }
}
```

The IPC mechanism uses files in `broadcast-coordination/` — the extension writes serialized pixel buffers, and the main app reads them. This is the simplest approach because extensions can't use XPC or direct IPC.

**Mode 2: `EmbeddedBroadcastUploadImpl`** — Creates its own `OngoingGroupCallContext` with a WebRTC stack and streams directly:

```swift
private final class EmbeddedBroadcastUploadImpl: BroadcastUploadImpl {
    private var callContext: OngoingGroupCallContext?
    private let screencastCapturer: OngoingCallVideoCapturer

    func initialize(rootPath: String) {
        let clientContext = IpcGroupCallEmbeddedBroadcastContext(
            basePath: rootPath + "/embedded-broadcast-coordination"
        )
        // When active, creates OngoingGroupCallContext with WebRTC
        // Exchanges join payloads via IPC files
    }
}
```

This mode is more complex — the extension runs its own WebRTC connection for the screencast stream, negotiating the join payload through file-based IPC with the main app. The `OngoingCallVideoCapturer(isCustom: true)` creates a custom video capturer that receives frames from the broadcast handler rather than from a camera.

## The Framework Dependency Graph

The BUILD file assembles five shared frameworks that extensions select from:

```
MtProtoKitFramework      ← Protocol implementation
  ↓
SwiftSignalKitFramework  ← Reactive programming
  ↓
PostboxFramework         ← Encrypted database
  ↓
TelegramCoreFramework    ← Account, messages, API
  ↓
TelegramUIFramework      ← UI components (heaviest)
```

Each extension cherry-picks the frameworks it needs:

| Extension | MtProto | SignalKit | Postbox | Core | UI |
|-----------|---------|----------|---------|------|----|
| Share | via UI | via UI | via UI | via UI | **yes** |
| NotificationService | **yes** | **yes** | **yes** | **yes** | **yes** |
| NotificationContent | via UI | via UI | via UI | via UI | **yes** |
| Widget | — | **yes** | **yes** | **yes** | — |
| SiriIntents | — | **yes** | **yes** | **yes** | — |
| BroadcastUpload | — | **yes** | — | — | **yes** |

The Widget and SiriIntents extensions avoid TelegramUIFramework entirely — they can't afford the memory overhead of loading the entire UI layer just to read some peer data. The NotificationService loads everything because it might need to process encrypted payloads (MtProtoKit) or interact with the account state (TelegramCore).

All frameworks are built with `extension_safe = True` in Bazel, which ensures they don't use APIs forbidden in extension contexts (like `UIApplication.shared`).

## The Entitlements Chain

Extensions inherit their App Group entitlement from the main app's configuration. The BUILD file generates entitlements dynamically:

```python
app_groups_fragment = """
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.{telegram_bundle_id}</string>
    </array>
    <key>application-identifier</key>
    <string>{telegram_team_id}.{telegram_bundle_id}</string>
""".format(
    telegram_team_id=telegram_team_id,
    telegram_bundle_id=telegram_bundle_id
)
```

Each extension has its own `plist_fragment` for its entitlements. The Share extension's entitlements include the same App Group but a different bundle identifier suffix:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>group.ph.telegra.Telegraph</string>
</array>
```

The signing profiles are extension-specific — each extension needs its own `.mobileprovision` file: `Share.mobileprovision`, `NotificationService.mobileprovision`, `Widget.mobileprovision`, `Intents.mobileprovision`, `BroadcastUpload.mobileprovision`, `NotificationContent.mobileprovision`.

## Architectural Lessons

**1. Extensions are first-class module consumers.** Telegram doesn't have special "extension-only" code paths through the core modules. Extensions use the same `Postbox`, `TelegramCore`, and `AccountManager` APIs that the main app uses. The differentiation happens at the framework dependency level — lighter extensions simply don't link the heavier frameworks.

**2. Code duplication is sometimes correct.** The NotificationService duplicates `DrawingContext`, avatar rendering, and color space utilities from the Display module. This seems wasteful, but the alternative — importing Display and its transitive dependencies — would blow the extension's memory budget. The 50MB memory limit is a hard constraint that overrides DRY principles.

**3. File-based IPC works.** The BroadcastUpload extension communicates with the main app through files in the shared container. No XPC, no Darwin notifications for data transfer, no shared memory regions. File I/O through the App Group container is the lowest-common-denominator IPC mechanism, and it's reliable enough for streaming video frames.

**4. The bootstrap tax is real.** Every extension pays the cost of: derive bundle ID → find App Group → derive encryption key → open AccountManager → find current account → open Postbox with encryption. This sequence takes measurable time (hundreds of milliseconds) and appears verbatim in every extension. Telegram accepts this cost rather than trying to cache state across extension invocations (which iOS doesn't guarantee will persist).

**5. App lock is a cross-process concern.** The lock state is stored as a simple JSON file that every extension checks before proceeding. This is simpler and more reliable than using UserDefaults or the Keychain for inter-process lock state — a plain file read has no framework dependencies and no race conditions with the main app's writes.
