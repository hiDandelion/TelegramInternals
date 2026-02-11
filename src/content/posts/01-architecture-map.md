---
title: "The 274-Module Monolith: Architecture Map of Telegram iOS"
description: "A top-down aerial survey of the Telegram iOS codebase — its Bazel build system, 274 submodules, dependency graph, and how to navigate it all."
published: 2026-02-05
tags:
  - Orientation
toc: true
lang: en
abbrlink: 01-architecture-map
pin: 25
---

This is the first post in a series that will walk you through the entire Telegram iOS codebase. The goal is not to give you vague architectural summaries — it's to explain *why* each piece exists and how it works, with real code, so you can actually apply these patterns in your own projects.

Telegram for iOS is one of the most sophisticated open-source iOS applications ever built. It runs on a custom protocol (MTProto), custom reactive framework (SwiftSignalKit), custom persistence layer (Postbox), custom UI framework (AsyncDisplayKit fork + ComponentFlow), and custom build system (Bazel) — all with **zero third-party Swift/ObjC dependencies**. Understanding how these pieces fit together is the key to unlocking the codebase.

## Why Everything Is Custom

Before we map the architecture, we need to understand *why* Telegram built so much from scratch. The short answer: **they had to**.

Telegram iOS development began around 2013-2014, long before:
- **SwiftUI** (2019) or **Combine** (2019) existed
- **Swift Package Manager** was production-ready
- **RxSwift** was mature enough for a messaging app's performance needs
- **Core Data** could handle the access patterns of a real-time messaging database

A messaging app has extreme requirements that off-the-shelf solutions don't meet well:

1. **Chat history is append-heavy** with random-access reads and "hole"-based pagination — Core Data's object-graph model is wrong for this.
2. **UI must hit 60fps** while rendering complex message bubbles with text entities, media thumbnails, reactions, and reply chains — UIKit's `UITableView` cannot achieve this without offloading layout and drawing to background threads.
3. **The protocol is proprietary** (MTProto) — there's no HTTP/REST library to adopt; the binary protocol requires custom serialization, encryption, and session management.
4. **Reactivity must be lightweight** — RxSwift's operator complexity and GCD-based scheduler abstractions add overhead that Telegram avoids with `pthread_mutex_t`-based primitives.

The result is a codebase where nearly every layer is purpose-built. The upside: unmatched performance and control. The downside: a steep learning curve — which this blog series aims to flatten.

## Top-Level Directory Structure

```
Telegram-iOS/
├── Telegram/                    # Main app target + all extensions
│   ├── Telegram-iOS/            # App entry point, resources, assets
│   ├── Share/                   # Share extension
│   ├── NotificationService/     # Push notification decryption
│   ├── NotificationContent/     # Rich notification UI
│   ├── BroadcastUpload/         # Screen recording extension
│   ├── Watch/                   # watchOS companion app
│   ├── WidgetKitWidget/         # Home/Lock screen widgets
│   ├── SiriIntents/             # Siri shortcut handlers
│   └── BUILD                    # Bazel build rules for all targets
│
├── submodules/                  # 274 Swift/ObjC library modules
│   ├── TelegramCore/            # Business logic & API
│   ├── TelegramUI/              # Main UI module
│   ├── Postbox/                 # SQLite persistence layer
│   ├── SSignalKit/              # Reactive framework (ObjC + Swift)
│   ├── MtProtoKit/              # MTProto protocol implementation
│   ├── AsyncDisplayKit/         # High-performance UI nodes
│   ├── Display/                 # UIKit framework layer
│   ├── ComponentFlow/           # Declarative component system
│   ├── AccountContext/           # DI boundary protocol
│   └── ... (265 more)
│
├── third-party/                 # 24 vendored C/C++ libraries
│   ├── webrtc/                  # Voice/video calls
│   ├── boringssl/               # TLS/crypto
│   ├── ffmpeg/                  # (via FFMpegBinding)
│   ├── dav1d/                   # AV1 video codec
│   ├── libvpx/                  # VP8/VP9 video codec
│   ├── opus/                    # Audio codec
│   ├── mozjpeg/                 # JPEG optimization
│   ├── libjxl/                  # JPEG XL support
│   └── ...
│
├── build-system/                # Bazel rules & build scripts
│   ├── bazel-rules/             # rules_apple, rules_swift, rules_xcodeproj
│   ├── Make/                    # Python build orchestration (Make.py)
│   └── bazel-utils/             # Helper Starlark functions
│
├── BUILD.bazel                  # Root build config
├── MODULE.bazel                 # Bazel module dependencies
├── versions.json                # Version pinning
└── README.md                    # Compilation guide
```

Let's break down each layer.

## The Bazel Build System

Telegram uses **Bazel** (version 8.4.2 as of this writing) instead of Xcode's native build system. This is unusual for iOS projects but makes perfect sense when you have 274 modules to compile.

### Why Bazel?

1. **Reproducible builds**: Every dependency version is pinned. The `versions.json` file locks Xcode (26.2), Bazel (8.4.2), and macOS SDK (26).
2. **Incremental compilation**: Bazel's content-addressed cache means changing one file only recompiles the affected module and its dependents — not the entire project.
3. **Module isolation**: Each of the 274 submodules has its own `BUILD` file defining a `swift_library` or `objc_library` target. Dependencies are declared explicitly, preventing accidental coupling.
4. **Xcode project generation**: Developers don't manually maintain an Xcode project. Instead, `rules_xcodeproj` generates one from Bazel targets.

### How It Works

The root `BUILD.bazel` is minimal — it just sets up SourceKit for IDE support:

```python
# BUILD.bazel
load("@sourcekit_bazel_bsp//rules:setup_sourcekit_bsp.bzl", "setup_sourcekit_bsp")

setup_sourcekit_bsp(
    name = "setup_sourcekit_bsp_telegram_project",
    bazel_wrapper = "./build-input/bazel-8.4.2-darwin-arm64",
    files_to_watch = ["**/*.swift"],
    index_flags = ["config=index_build"],
    targets = ["//Telegram:Telegram"],
)
```

The real action happens in `MODULE.bazel`, which declares Bazel module dependencies with **local path overrides** for Apple build rules:

```python
# MODULE.bazel
bazel_dep(name = "rules_xcodeproj")
local_path_override(
    module_name = "rules_xcodeproj",
    path = "./build-system/bazel-rules/rules_xcodeproj",
)

bazel_dep(name = "rules_apple", repo_name = "build_bazel_rules_apple")
local_path_override(
    module_name = "rules_apple",
    path = "./build-system/bazel-rules/rules_apple",
)

bazel_dep(name = "rules_swift", repo_name = "build_bazel_rules_swift")
local_path_override(
    module_name = "rules_swift",
    path = "./build-system/bazel-rules/rules_swift",
)
```

Notice all the rules are vendored locally under `build-system/bazel-rules/` — there are no network fetches during build. This is intentional: Telegram wants builds to be entirely reproducible without depending on external registries.

The main app target is defined in `Telegram/BUILD`:

```python
# Telegram/BUILD (simplified)
load("@build_bazel_rules_apple//apple:ios.bzl", "ios_application", "ios_extension")
load("@build_bazel_rules_swift//swift:swift.bzl", "swift_library")
load("@build_configuration//:variables.bzl",
    "telegram_bundle_id",
    "telegram_team_id",
    "telegram_aps_environment",
    # ...
)
```

Build configuration (API keys, bundle IDs, team IDs) is injected from an external `build-input/configuration-repository/` directory that is **not** in the repo — each developer provides their own credentials.

### Building the App

To generate an Xcode project:

```bash
python3 build-system/Make/Make.py \
    --cacheDir="$HOME/telegram-bazel-cache" \
    generateProject \
    --configurationPath=build-system/template_minimal_development_configuration.json \
    --xcodeManagedCodesigning
```

This runs `Make.py` (a large Python orchestration script) that invokes Bazel to compile all 274 modules, then generates an Xcode project you can open and run.

## The 10-Layer Dependency Stack

Every module in the codebase sits on a dependency stack. Here are the 10 foundational layers, from bottom to top:

```
Layer 10: TelegramUI          ← Main UI, 1000+ files
Layer 9:  TelegramPresentationData  ← Themes, strings, colors
Layer 8:  ComponentFlow        ← Declarative UI components
Layer 7:  Display + AsyncDisplayKit  ← High-performance UI
Layer 6:  AccountContext       ← DI boundary between Core and UI
Layer 5:  TelegramCore         ← Business logic, state sync, API facade
Layer 4:  TelegramApi          ← Auto-generated API schema (50 files)
Layer 3:  Postbox              ← SQLite persistence layer
Layer 2:  SwiftSignalKit       ← Reactive programming primitives
Layer 1:  SSignalKit (ObjC)    ← ObjC reactive layer for MtProtoKit
Layer 0:  MtProtoKit           ← MTProto protocol (ObjC/C++)
```

Data flows **up** through these layers: MtProtoKit receives encrypted binary data from Telegram servers, SSignalKit wraps it in reactive signals, Postbox persists it in SQLite, TelegramCore exposes it through the `TelegramEngine` facade, and TelegramUI renders it.

User actions flow **down**: UI events become TelegramEngine method calls, which become Postbox transactions and network requests, which become MTProto wire-format messages.

### The TelegramEngine Facade

The bridge between TelegramCore and the UI is `TelegramEngine` — a facade class with lazy-loaded subsystems:

```swift
// submodules/TelegramCore/Sources/TelegramEngine/TelegramEngine.swift
public final class TelegramEngine {
    public let account: Account

    public init(account: Account) {
        self.account = account
    }

    public lazy var peers: Peers = {
        return Peers(account: self.account)
    }()

    public lazy var messages: Messages = {
        return Messages(account: self.account)
    }()

    public lazy var stickers: Stickers = {
        return Stickers(account: self.account)
    }()

    public lazy var calls: Calls = {
        return Calls(account: self.account)
    }()

    public lazy var contacts: Contacts = {
        return Contacts(account: self.account)
    }()

    public lazy var privacy: Privacy = {
        return Privacy(account: self.account)
    }()

    public lazy var payments: Payments = {
        return Payments(account: self.account)
    }()

    public lazy var resources: Resources = {
        return Resources(account: self.account)
    }()

    public lazy var data: EngineData = {
        return EngineData(accountPeerId: self.account.peerId, postbox: self.account.postbox)
    }()

    // ... 10+ more subsystems: auth, secureId, peersNearby, localization,
    //     themes, historyImport, resolve, orderedLists, itemCache, notices, preferences
}
```

This is the **table of contents** for all business logic. When you see UI code calling `context.engine.messages.sendMessage(...)` or `context.engine.peers.fetchProfile(...)`, that's going through these lazy subsystems into TelegramCore's implementation.

There's also a `TelegramEngineUnauthorized` for the login flow — it only exposes `auth`, `localization`, `payments`, and `itemCache`:

```swift
public final class TelegramEngineUnauthorized {
    public let account: UnauthorizedAccount

    public lazy var auth: Auth = {
        return Auth(account: self.account)
    }()

    public lazy var localization: Localization = {
        return Localization(account: self.account)
    }()

    public lazy var payments: Payments = {
        return Payments(account: self.account)
    }()

    public lazy var itemCache: ItemCache = {
        return ItemCache(account: self.account)
    }()
}
```

And an enum to discriminate between the two states:

```swift
public enum SomeTelegramEngine {
    case unauthorized(TelegramEngineUnauthorized)
    case authorized(TelegramEngine)
}
```

This pattern — authorized vs unauthorized state as a first-class concept — runs through the entire codebase, from the engine to the app delegate to the UI context.

## Module Categories

The 274 submodules break down into roughly these categories:

### Core Infrastructure (10 modules)

These are the foundational frameworks that everything else depends on:

| Module | What it does |
|--------|-------------|
| `SSignalKit` | Objective-C reactive framework (for MtProtoKit) |
| `SwiftSignalKit` | Swift reactive framework (Signal, Promise, Disposable) |
| `Postbox` | SQLite-based persistence with reactive views |
| `MtProtoKit` | MTProto 2.0 protocol implementation (ObjC) |
| `TelegramApi` | Auto-generated API types from TL schema |
| `TelegramCore` | Business logic, state sync, TelegramEngine facade |
| `AccountContext` | Protocol defining DI boundary between Core and UI |
| `BuildConfig` / `BuildConfigExtra` | Compile-time configuration |
| `EncryptionProvider` | Crypto abstraction over BoringSSL |
| `ManagedFile` | File I/O abstraction |

### UI Framework (8 modules)

The custom rendering and navigation stack:

| Module | What it does |
|--------|-------------|
| `AsyncDisplayKit` | Background-thread-safe UI nodes (forked from Texture) |
| `Display` | Telegram's UIKit layer: ViewController, NavigationController, ListView |
| `ComponentFlow` | Declarative component system (Telegram's SwiftUI alternative) |
| `TelegramPresentationData` | Themes, colors, strings, localization |
| `TelegramBaseController` | Base view controller with common behaviors |
| `TabBarUI` | Custom tab bar implementation |
| `ItemListUI` | Settings-style table view framework |
| `LegacyComponents` | Older ObjC UI components still in use |

### UI Components (100+ modules)

Individual reusable UI elements. A small sampling:

| Module | What it does |
|--------|-------------|
| `AlertUI` | Custom alert dialogs |
| `AvatarNode` | User/group avatar rendering |
| `ChatListUI` | Chat list screen |
| `ChatMessageBackground` | Message bubble rendering |
| `ContactListUI` | Contacts screen |
| `ContextUI` | Long-press context menus |
| `GalleryUI` | Full-screen media gallery |
| `SearchUI` | Search UI framework |
| `ShareController` | Share sheet |
| `SettingsUI` | Settings screens |
| `PeerInfoUI` | User/group/channel profile |
| `UndoUI` | Undo toast notifications |
| `PremiumUI` | Telegram Premium subscription UI |
| `TranslateUI` | Message translation UI |
| `StickerPackPreviewUI` | Sticker pack preview |
| `PasscodeUI` | App lock passcode entry |

Each of these is a self-contained `swift_library` with its own `BUILD` file and explicit dependency declarations.

### Media & Animation (15+ modules)

| Module | What it does |
|--------|-------------|
| `MediaPlayer` | Audio/video playback |
| `Camera` | Camera capture |
| `MediaEditor` | Photo/video editing |
| `AnimatedStickerNode` | Animated sticker rendering |
| `lottie-ios` / `LottieCpp` / `rlottie` | Lottie animation support |
| `MetalEngine` | Metal GPU rendering for animations |
| `TelegramAudio` | Audio recording and playback |
| `FFMpegBinding` | FFmpeg video processing |
| `WebPBinding` | WebP image format |
| `StickerResources` / `WallpaperResources` | Asset management |
| `ImageBlur` / `FastBlur` | Image blur effects |
| `TinyThumbnail` | Progressive thumbnail loading |

### Networking & Data (5+ modules)

| Module | What it does |
|--------|-------------|
| `NetworkLogging` | Network debug logging |
| `Reachability` | Network connectivity detection |
| `FetchManagerImpl` | Download queue management |
| `CloudData` | iCloud sync |
| `CryptoUtils` | Cryptographic helpers |

### Utility Modules (30+ modules)

Small, focused utilities:

| Module | What it does |
|--------|-------------|
| `Emoji` | Emoji handling |
| `PhoneNumberFormat` | Phone number formatting (via `libphonenumber`) |
| `TextFormat` | Text entities (bold, italic, code blocks) |
| `Markdown` | Markdown parsing |
| `Crc32` | Checksum utilities |
| `MergeLists` | Diff algorithm for list updates |
| `GZip` | Compression |
| `HexColor` | Hex color parsing |
| `PersistentStringHash` | Stable string hashing |
| `MimeTypes` | MIME type detection |

## Third-Party Vendors

The `third-party/` directory contains 24 vendored C/C++ libraries. These are **not** fetched via a package manager — they're committed to the repository (or linked as git submodules) for build reproducibility.

Key vendors:

| Library | Purpose | Why vendored |
|---------|---------|-------------|
| `webrtc` | Voice/video calls | Custom build configuration for Telegram's VoIP |
| `boringssl` | TLS/SSL/crypto | MTProto encryption primitives |
| `opus` / `opusfile` / `ogg` | Audio codec | Voice messages and calls |
| `libvpx` | VP8/VP9 video | Video calls and encoding |
| `dav1d` | AV1 video decode | Modern video format support |
| `openh264` | H.264 video | Video calls |
| `mozjpeg` | JPEG optimization | Compressed image uploads |
| `libjxl` | JPEG XL | Next-gen image format |
| `libyuv` | Color space conversion | Video processing |
| `rnnoise` | Neural noise suppression | Call audio quality |
| `webp` | WebP image format | Sticker support |
| `libprisma` | Image processing | Photo editing |
| `flatc` / `flatbuffers` | Binary serialization | Efficient data encoding |
| `ZipArchive` | ZIP compression | Theme/sticker pack import |
| `AppCenter` | Crash reporting | Production diagnostics |

## App Extensions

The Telegram app includes **7 extension targets**, each defined in `Telegram/BUILD`:

1. **Share Extension** (`Telegram/Share/`): Lets users share content from other apps to Telegram chats. Uses a stripped-down version of the chat picker UI.

2. **NotificationService Extension** (`Telegram/NotificationService/`): Intercepts push notifications before they're displayed. Decrypts the encrypted payload (MTProto pushes are encrypted), fetches message content if needed, and generates rich notification content.

3. **NotificationContent Extension** (`Telegram/NotificationContent/`): Provides custom UI for notification long-press previews.

4. **BroadcastUpload Extension** (`Telegram/BroadcastUpload/`): Enables screen recording/sharing in group calls using `ReplayKit`.

5. **Watch App** (`Telegram/Watch/`): A full watchOS companion app with 172 source files in the extension, supporting message viewing and quick replies.

6. **WidgetKit Widgets** (`Telegram/WidgetKitWidget/`): Home screen and Lock screen widgets showing recent chats, shortcuts, and status.

7. **Siri Intents** (`Telegram/SiriIntents/`): 26 intent handlers for Siri shortcuts — send messages, make calls, etc.

These extensions share core modules (`Postbox`, `TelegramCore`, `SwiftSignalKit`) via shared app group containers, but each runs in its own process with strict memory limits.

## How to Navigate the Codebase

With 274 modules, finding where a feature lives can be daunting. Here's a practical guide:

### "I want to find the UI for feature X"

Most feature UIs live in a dedicated module under `submodules/`. The naming is consistent:
- Chat list → `ChatListUI`
- Contacts → `ContactListUI`
- Calls → `CallListUI`
- Settings → `SettingsUI`
- Sticker packs → `StickerPackPreviewUI`
- Payments → `BotPaymentsUI`
- Premium → `PremiumUI`

If the feature is part of the main chat experience, it's in `TelegramUI/Sources/Chat/`.

### "I want to find how feature X talks to the API"

Look in `TelegramCore/Sources/TelegramEngine/`. The engine is organized by subsystem:
- `TelegramEngine/Messages/` — send, edit, delete, search messages
- `TelegramEngine/Peers/` — user profiles, groups, channels
- `TelegramEngine/Auth/` — login, 2FA, session management
- `TelegramEngine/Stickers/` — sticker packs, emoji
- `TelegramEngine/Calls/` — voice/video call management
- `TelegramEngine/Privacy/` — privacy settings, blocked users

### "I want to understand how data is stored"

Look in `Postbox/Sources/`. Key files:
- `Postbox.swift` — Transaction API
- `MessageHistoryTable.swift` — Message storage
- `PeerTable.swift` — User/group/channel storage
- `ChatListTable.swift` — Chat list ordering
- `ViewTracker.swift` — Reactive query subscriptions

### "I want to see the raw API calls"

Look in `TelegramApi/Sources/`. The `Api0.swift` through `Api49.swift` files contain auto-generated types matching the Telegram TL schema. These are called from `TelegramCore/Sources/Network/`.

### "I want to understand the UI framework"

Start with:
- `AsyncDisplayKit/` — the node-based rendering system
- `Display/Source/` — ViewController, NavigationController, ListView
- `ComponentFlow/Source/` — the declarative component protocol

## Key Files Reference

Here are the 10 most important files to understand the architecture:

| File | Lines | What it does |
|------|-------|-------------|
| `submodules/TelegramCore/Sources/TelegramEngine/TelegramEngine.swift` | 124 | Facade API — table of contents for all business logic |
| `submodules/TelegramUI/Sources/AppDelegate.swift` | ~3,300 | App initialization, account switching, lifecycle |
| `submodules/TelegramUI/Sources/ApplicationContext.swift` | ~1,500 | Authorized vs unauthorized context management |
| `submodules/TelegramUI/Sources/TelegramRootController.swift` | ~500 | Root tab controller with chat list, contacts, settings |
| `submodules/SSignalKit/SwiftSignalKit/Source/Signal.swift` | ~140 | The core reactive primitive (Signal type) |
| `submodules/Postbox/Sources/Postbox.swift` | ~3,000 | Transaction API for all database operations |
| `submodules/TelegramCore/Sources/Account/Account.swift` | ~2,500 | Account lifecycle and core state |
| `submodules/TelegramCore/Sources/Network/Network.swift` | ~1,800 | Network layer wrapping MTProto |
| `submodules/Display/Source/Navigation/NavigationController.swift` | ~2,000 | Custom navigation stack |
| `submodules/ComponentFlow/Source/Base/Component.swift` | ~200 | Declarative component protocol |

## What's Next

In the [next post](/posts/02-app-lifecycle/), we'll trace the complete app launch path — from the Objective-C `main.m` entry point, through `Application.swift` and the massive `AppDelegate`, to the moment your chat list appears on screen. We'll see how the authorized/unauthorized context split works, how multi-account support is implemented, and how `TelegramRootController` assembles the four main tabs.

---

*This post covers: `BUILD.bazel`, `MODULE.bazel`, `versions.json`, `Telegram/BUILD`, `submodules/TelegramCore/Sources/TelegramEngine/TelegramEngine.swift`*
