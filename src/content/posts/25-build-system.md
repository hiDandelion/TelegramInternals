---
title: "Build System: Bazel, 274 Modules, and Reproducible Builds"
description: "The final post in the series. A comprehensive walkthrough of how Telegram iOS's Bazel build system assembles 274 Swift/Objective-C modules and 26 vendored C/C++ libraries into a single IPA. Covers MODULE.bazel and the dependency graph, the Make.py orchestration layer, build configurations (debug_arm64, release_arm64), code signing strategies (git-encrypted profiles, Xcode-managed, local directory), code generation pipelines (localized strings, FlatBuffers, Intents), the .bazelrc compiler flags, release optimizations (WMO, -Osize, dead_strip), and practical advice for navigating and building the codebase."
published: 2025-08-05
tags:
  - Build & Extensions
toc: true
lang: en
abbrlink: 25-build-system
pin: 1
---

This is the final post in the series. We've walked through every layer of Telegram iOS — from the reactive signal system through the encrypted database, the networking stack, the UI framework, and the feature implementations. Now we look at the system that compiles all of it: 274 Swift and Objective-C modules, 26 vendored C/C++ libraries, 6 app extensions, and a Watch app, assembled by Bazel into a single IPA.

## Why Bazel?

Telegram iOS doesn't use Xcode's native build system. It uses Bazel — Google's open-source build tool originally designed for their monorepo. The reasons are practical:

1. **Reproducible builds.** Given the same source and the same Bazel version, you get identical output. This matters when you ship to hundreds of millions of users.
2. **Modular compilation.** Each of the 274 modules is a separate `swift_library` target. Bazel tracks dependencies between them precisely — if you change `TelegramCore`, it recompiles only `TelegramCore` and its dependents, not the entire project.
3. **Remote caching.** Build artifacts are content-addressable. A CI server can populate a cache, and developer machines fetch pre-built modules instead of recompiling them.
4. **Extension/framework assembly.** Bazel's `ios_framework` and `ios_extension` rules handle the complex task of building shared frameworks with the correct linking, embedding, and code signing — something that's painful to configure in Xcode for 5 frameworks and 6 extensions.

The project pins Bazel 8.4.2 with a specific SHA256 hash in `versions.json`:

```json
{
    "app": "12.4",
    "xcode": "26.2",
    "bazel": "8.4.2:45e9388abf21d1107e146ea366ad080eb93cb6a5f3a4a3b048f78de0bc3faffa",
    "macos": "26"
}
```

This ensures every developer and CI machine uses the exact same Bazel binary. The SHA256 hash prevents supply chain attacks on the build tool itself.

## MODULE.bazel: The Dependency Root

Bazel 8+ uses `MODULE.bazel` (Bzlmod) instead of the older `WORKSPACE` file. Telegram's module definition is 70 lines:

```python
bazel_dep(name = "bazel_features", version = "1.33.0")
bazel_dep(name = "bazel_skylib", version = "1.8.1")
bazel_dep(name = "platforms", version = "0.0.11")
```

These are the only external dependencies fetched from the Bazel Central Registry. Everything else is local:

```python
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

Telegram vendors *its own forks* of `rules_apple`, `rules_swift`, `rules_xcodeproj`, and `apple_support` under `build-system/bazel-rules/`. This is unusual but deliberate — the Telegram team needs to patch these rules for their specific build requirements without waiting for upstream releases.

The build configuration is injected as a Bazel module too:

```python
bazel_dep(name = "build_configuration")
local_path_override(
    module_name = "build_configuration",
    path = "./build-input/configuration-repository",
)
```

This `build-input/configuration-repository/` directory is generated at build time by `Make.py`. It contains a `variables.bzl` file with all the build parameters — bundle ID, API credentials, team ID, APS environment, feature flags. More on this shortly.

External tools are fetched as `http_file` with pinned hashes:

```python
http_file(
    name = "cmake_tar_gz",
    urls = ["https://github.com/Kitware/CMake/releases/download/v4.1.2/cmake-4.1.2-macos-universal.tar.gz"],
    sha256 = "3be85f5b999e327b1ac7d804cbc9acd767059e9f603c42ec2765f6ab68fbd367",
)

http_file(name = "meson_tar_gz", ...)    # Meson 1.6.0
http_file(name = "ninja-mac_zip", ...)   # Ninja v1.12.1
http_file(name = "flatbuffers_zip", ...)  # FlatBuffers v24.12.23
```

CMake, Meson, and Ninja are needed to build some third-party C/C++ libraries (like ffmpeg, dav1d, libvpx) that use their own build systems. FlatBuffers is used for serialization in `TelegramCore/FlatSerialization`.

## Telegram/BUILD: The 1,911-Line Centerpiece

The main BUILD file at `Telegram/BUILD` defines the entire app assembly. Let's walk through its structure.

### Build Flags

```python
bool_flag(name = "disableExtensions", build_setting_default = False)
bool_flag(name = "disableProvisioningProfiles", build_setting_default = False)
bool_flag(name = "projectIncludeRelease", build_setting_default = False)
bool_flag(name = "disableStripping", build_setting_default = False)
```

These flags control build behavior via `--//Telegram:disableExtensions=True` on the command line. `disableExtensions` is useful during development — building 6 extensions adds significant compile time.

### Configuration Injection

The BUILD file loads build parameters from the generated `variables.bzl`:

```python
load(
    "@build_configuration//:variables.bzl",
    "telegram_bazel_path",
    "telegram_use_xcode_managed_codesigning",
    "telegram_bundle_id",
    "telegram_aps_environment",
    "telegram_team_id",
    "telegram_enable_icloud",
    "telegram_enable_siri",
)
```

These values drive everything — from the bundle ID in every `ios_framework()` declaration to the entitlements in the app's code signing configuration.

### Code Generation: Localized Strings

```python
genrule(
    name = "GeneratedPresentationStrings",
    srcs = [
        "//build-system:GenerateStrings/GenerateStrings.py",
        "Telegram-iOS/en.lproj/Localizable.strings",
    ],
    cmd = '''
        python3 $(location //build-system:GenerateStrings/GenerateStrings.py) \
            --source=$(location Telegram-iOS/en.lproj/Localizable.strings) \
            --outImplementation=$(location GeneratedPresentationStrings/Sources/PresentationStrings.m) \
            --outHeader=$(location GeneratedPresentationStrings/PublicHeaders/PresentationStrings/PresentationStrings.h) \
            --outData=$(location GeneratedPresentationStrings/Resources/PresentationStrings.data)
    ''',
    outs = [
        "GeneratedPresentationStrings/PublicHeaders/PresentationStrings/PresentationStrings.h",
        "GeneratedPresentationStrings/Sources/PresentationStrings.m",
        "GeneratedPresentationStrings/Resources/PresentationStrings.data",
    ],
)
```

`GenerateStrings.py` reads the English `.strings` file and generates an Objective-C implementation with optimized key-to-string lookup. The generated `PresentationStrings.m` is then compiled as an `objc_library`:

```python
objc_library(
    name = "PresentationStrings",
    enable_modules = True,
    module_name = "PresentationStrings",
    srcs = ["GeneratedPresentationStrings/Sources/PresentationStrings.m"],
    hdrs = ["GeneratedPresentationStrings/PublicHeaders/PresentationStrings/PresentationStrings.h"],
    deps = [
        "//submodules/NumberPluralizationForm:NumberPluralizationForm",
        "//submodules/AppBundle:AppBundle",
    ],
)
```

Other languages are generated as empty files during the build — the actual translations are loaded at runtime from the server:

```python
empty_languages = ["ar", "be", "ca", "de", "es", "fa", "fr", "id", "it",
                    "ko", "ms", "nl", "pl", "pt", "ru", "tr", "uk", "uz"]

[
    genrule(
        name = "Localizable_{}.strings".format(language),
        outs = ["{}.lproj/Localizable.strings".format(language)],
        cmd = "touch $(OUTS)",
    ) for language in empty_languages
]
```

### The App Entry Point

```python
objc_library(
    name = "Main",
    module_name = "Main",
    srcs = ["Telegram-iOS/main.m"],
)

swift_library(
    name = "Lib",
    module_name = "Lib",
    srcs = glob(["Telegram-iOS/Application.swift"]),
    data = [
        ":Icons", ":AppResources", ":AppIntentVocabularyResources",
        ":InfoPlistStringResources",
        "//submodules/LegacyComponents:LegacyComponentsResources",
        "//submodules/TelegramUI:TelegramUIResources",
        "//submodules/TelegramUI:TelegramUIAssets",
        "//submodules/TelegramUI/Components/Calls/CallScreen:Assets",
        ":GeneratedPresentationStrings/Resources/PresentationStrings.data",
    ],
    deps = [
        "//submodules/TelegramUI:TelegramUI",
        "//third-party/boringssl:crypto",
    ],
)
```

The main app has only two source files compiled directly in its BUILD target: `main.m` (Objective-C entry point) and `Application.swift` (custom `UIApplication` subclass). Everything else lives in `submodules/`.

### The Five Shared Frameworks

Telegram builds five `ios_framework` targets that are shared between the main app and extensions:

```python
ios_framework(name = "MtProtoKitFramework",
    deps = ["//submodules/MtProtoKit:MtProtoKit"],
    extension_safe = True,
    ipa_post_processor = strip_framework, ...)

ios_framework(name = "SwiftSignalKitFramework",
    deps = ["//submodules/SSignalKit/SwiftSignalKit:SwiftSignalKit"],
    extension_safe = True, ...)

ios_framework(name = "PostboxFramework",
    frameworks = [":SwiftSignalKitFramework"],
    deps = ["//submodules/Postbox:Postbox"],
    extension_safe = True, ...)

ios_framework(name = "TelegramCoreFramework",
    frameworks = [":MtProtoKitFramework", ":SwiftSignalKitFramework", ":PostboxFramework"],
    deps = ["//submodules/TelegramCore:TelegramCore"],
    extension_safe = True, ...)

ios_framework(name = "TelegramUIFramework",
    frameworks = [":MtProtoKitFramework", ":SwiftSignalKitFramework",
                  ":PostboxFramework", ":TelegramCoreFramework"],
    deps = ["//submodules/TelegramUI:TelegramUI"],
    extension_safe = True, ...)
```

The framework dependency chain is strict: `TelegramCoreFramework` embeds references to `MtProtoKitFramework`, `SwiftSignalKitFramework`, and `PostboxFramework`. This means an extension that links `TelegramCoreFramework` automatically gets the other three.

Each framework is built with `extension_safe = True` (no UIApplication.shared calls) and stripped in release builds via `StripFramework.sh`:

```bash
for f in $1/*.framework; do
    binary_name=$(echo $(basename $f) | sed -e "s/\.framework//")
    strip -ST $f/$binary_name
done
```

### Dynamic Entitlements

Entitlements are built from fragments that vary based on the bundle ID:

```python
# Apple Pay only for official builds
apple_pay_merchants = official_apple_pay_merchants \
    if telegram_bundle_id == "ph.telegra.Telegraph" else []

# Associated domains only for official builds
associated_domains_fragment = "" if telegram_bundle_id not in official_bundle_ids else """
<key>com.apple.developer.associated-domains</key>
<array>
    <string>applinks:telegram.me</string>
    <string>applinks:t.me</string>
    <string>applinks:*.t.me</string>
    <string>webcredentials:t.me</string>
    <string>webcredentials:telegram.org</string>
</array>
"""

# iCloud only if enabled in config
icloud_fragment = "" if not telegram_enable_icloud else """
<key>com.apple.developer.icloud-services</key>
<array><string>CloudKit</string><string>CloudDocuments</string></array>
"""
```

All fragments are concatenated into a single `plist_fragment` rule:

```python
plist_fragment(
    name = "TelegramEntitlements",
    extension = "entitlements",
    template = "".join([
        aps_fragment, app_groups_fragment, siri_fragment,
        associated_domains_fragment, icloud_fragment,
        apple_pay_merchants_fragment, unrestricted_voip_fragment,
        carplay_fragment, communication_notifications_fragment,
        notification_filtering_fragment, signin_fragment,
    ])
)
```

The `plist_fragment` rule (defined in `build-system/bazel-utils/plist_fragment.bzl`) substitutes `{key}` placeholders with values from `--define` flags and wraps everything in valid plist XML.

### The Final Assembly

```python
ios_application(
    name = "Telegram",
    bundle_id = "{telegram_bundle_id}",
    families = ["iphone", "ipad"],
    minimum_os_version = "13.0",
    provisioning_profile = select({
        ":disableProvisioningProfilesSetting": None,
        "//conditions:default": ":Telegram_xcode_profile"
            if telegram_use_xcode_managed_codesigning
            else "@build_configuration//provisioning:Telegram.mobileprovision",
    }),
    entitlements = ":TelegramEntitlements.entitlements",
    frameworks = [
        ":MtProtoKitFramework",
        ":SwiftSignalKitFramework",
        ":PostboxFramework",
        ":TelegramCoreFramework",
        ":TelegramUIFramework",
    ],
    extensions = select({
        ":disableExtensionsSetting": [],
        "//conditions:default": [
            ":ShareExtension",
            ":NotificationContentExtension",
            ":NotificationServiceExtensionv1",
            ":IntentsExtension",
            ":WidgetExtension",
            ":BroadcastUploadExtension",
        ],
    }),
    deps = [":Main", ":Lib"],
)
```

This single `ios_application` rule produces the IPA. Bazel resolves the entire transitive dependency graph — from `Main` and `Lib` through `TelegramUI` and its 200+ transitive submodule dependencies, through the five shared frameworks and six extensions — and produces a signed, ready-to-install application.

## Make.py: The Orchestration Layer

Nobody invokes Bazel directly. `build-system/Make/Make.py` (1,360 lines of Python) wraps Bazel with configuration management, code signing, and convenience commands.

### Build Configurations

```python
def set_configuration(self, configuration):
    if configuration == 'debug_arm64':
        self.configuration_args = ['-c', 'dbg', '--ios_multi_cpus=arm64',
                                    '--watchos_cpus=arm64_32']
    elif configuration == 'debug_sim_arm64':
        self.configuration_args = ['-c', 'dbg', '--ios_multi_cpus=sim_arm64',
                                    '--watchos_cpus=arm64_32']
    elif configuration == 'release_arm64':
        self.configuration_args = [
            '-c', 'opt',
            '--ios_multi_cpus=arm64',
            '--watchos_cpus=arm64_32',
            '--apple_generate_dsym',
            '--output_groups=+dsyms',
        ] + self.common_release_args
```

Four configurations: `debug_arm64` (device debug), `debug_sim_arm64` (simulator debug), `release_sim_arm64` (simulator release), `release_arm64` (App Store release). Each sets the compilation mode (`dbg` or `opt`), CPU architecture, and Watch architecture.

### Common Build Arguments

```python
self.common_args = [
    '--announce_rc',                                    # Print resolved config
    '--features=swift.use_global_module_cache',         # Share Clang module cache
    '--verbose_failures',                               # Detailed errors
    '--remote_cache_async',                             # Non-blocking cache uploads
]

self.common_build_args = [
    '--features=swift.skip_function_bodies_for_derived_files',  # Faster indexing
    '--jobs={}'.format(os.cpu_count()),                          # Max parallelism
]
```

### Release Optimizations

```python
self.common_release_args = [
    '--features=swift.opt_uses_wmo',            # Whole Module Optimization
    '--features=swift.opt_uses_osize',          # -Osize instead of -O
    '--features=dead_strip',                    # Remove unused code
    '--objc_enable_binary_stripping',           # Strip Obj-C binaries
]
```

Whole Module Optimization (WMO) lets the Swift compiler see all functions in a module at once, enabling aggressive inlining and devirtualization. `-Osize` optimizes for binary size rather than speed — important for a 274-module app where code size directly impacts download size and launch time. Dead stripping removes unreachable code at link time.

### Code Signing Resolution

Make.py supports three code signing sources:

**1. Git-encrypted profiles** (for CI):
```python
class GitCodesigningSource:
    def __init__(self, repo_url, private_key_path, team_id, profile_type):
        # Clones encrypted git repo via SSH
        # Decrypts .mobileprovision and .p12 files
        # Profile types: development, adhoc, appstore, enterprise
```

**2. Local directory** (for manual setup):
```python
class DirectoryCodesigningSource:
    def __init__(self, path):
        # Reads profiles directly from a local directory
```

**3. Xcode-managed** (for quick development):
```python
class XcodeManagedCodesigningSource:
    # Sets telegram_use_xcode_managed_codesigning = True
    # Lets Xcode handle signing automatically
```

The resolved signing configuration is written to `build-input/configuration-repository/variables.bzl`:

```python
telegram_bundle_id = "ph.telegra.Telegraph"
telegram_api_id = "8"
telegram_api_hash = "..."
telegram_team_id = "..."
telegram_aps_environment = "production"
telegram_use_xcode_managed_codesigning = False
telegram_enable_siri = True
telegram_enable_icloud = True
```

## .bazelrc: Compiler Configuration

The `.bazelrc` file (41 lines) configures the build environment:

```
build --apple_platform_type=ios
build --enable_platform_specific_config
build --apple_crosstool_top=@local_config_apple_cc//:toolchain
```

These lines select the iOS Apple toolchain for all builds.

```
build --per_file_copt=".*\.m$","@-fno-objc-msgsend-selector-stubs"
build --per_file_copt=".*\.mm$","@-fno-objc-msgsend-selector-stubs"
```

This disables Objective-C message send selector stubs for all `.m` and `.mm` files. Selector stubs are an optimization in newer Clang versions that can cause compatibility issues with dynamic Objective-C patterns used throughout TelegramUI.

```
build --features=debug_prefix_map_pwd_is_dot
build --features=swift.cacheable_swiftmodules
build --features=swift.debug_prefix_map
build --features=swift.enable_vfsoverlays
```

`cacheable_swiftmodules` ensures Swift module outputs are deterministic (needed for remote caching). `enable_vfsoverlays` uses virtual filesystem overlays to speed up header lookups.

```
build --strategy=SwiftCompile=worker
```

This runs Swift compilation using Bazel's persistent worker mode — the `swiftc` process stays alive between compilations, avoiding the JIT warmup cost on each invocation. This is one of the biggest compile-time optimizations for Swift in Bazel.

The Swift profiling configuration is interesting:

```
build:swift_profile --jobs=1
build:swift_profile --features=-swift.enable_batch_mode
build:swift_profile --@build_bazel_rules_swift//swift:copt=-Xfrontend
build:swift_profile --@build_bazel_rules_swift//swift:copt=-warn-long-function-bodies=350
build:swift_profile --@build_bazel_rules_swift//swift:copt=-Xfrontend
build:swift_profile --@build_bazel_rules_swift//swift:copt=-warn-long-expression-type-checking=350
```

With `bazel build --config=swift_profile`, the build runs single-threaded and warns about any function body that takes more than 350ms to type-check. This is how the team identifies Swift type-checking bottlenecks.

## Module BUILD Patterns

### Pattern 1: Simple Swift Library

Most of the 274 modules follow this pattern:

```python
swift_library(
    name = "TelegramApi",
    module_name = "TelegramApi",
    srcs = glob(["Sources/**/*.swift"]),
    copts = ["-warnings-as-errors"],
    visibility = ["//visibility:public"],
)
```

`glob(["Sources/**/*.swift"])` includes all Swift files recursively. Visibility is public so any module can depend on it.

### Pattern 2: Module with Dependencies

```python
swift_library(
    name = "TelegramCore",
    srcs = glob(["Sources/**/*.swift"]),
    deps = [
        "//submodules/TelegramApi:TelegramApi",
        "//submodules/MtProtoKit:MtProtoKit",
        "//submodules/SSignalKit/SwiftSignalKit:SwiftSignalKit",
        "//submodules/Postbox:Postbox",
        "//submodules/CloudData:CloudData",
        "//submodules/EncryptionProvider:EncryptionProvider",
        # ... 20+ more deps
    ],
)
```

TelegramCore's dependency list is explicit — every module it imports must be listed. This is one of Bazel's strengths: you can't accidentally depend on a module without declaring it.

### Pattern 3: Objective-C + Swift Bridge

```python
objc_library(
    name = "LegacyComponents",
    srcs = glob(["Sources/**/*.m", "Sources/**/*.mm"]),
    hdrs = glob(["PublicHeaders/**/*.h"]),
    includes = ["PublicHeaders"],
    deps = [...],
)
```

The legacy Objective-C code (LegacyComponents, AsyncDisplayKit) uses `objc_library` with explicit header declarations. Swift modules that depend on these use Bazel's built-in Clang module generation.

### Pattern 4: Generated Code

```python
models = glob(["Models/*.fbs"])

genrule(
    name = "GenerateModels",
    srcs = models,
    tools = ["//third-party/flatc:flatc_bin"],
    cmd_bash = """
        FLATC="$(pwd)/$(location //third-party/flatc:flatc_bin)"
        "$$FLATC" --require-explicit-ids --swift -o "$$BUILD_DIR" {models}
    """,
    outs = ["{name}_generated.swift" for each model],
)

swift_library(
    name = "FlatSerialization",
    srcs = generated_models,
    deps = ["//submodules/TelegramCore/FlatBuffers"],
)
```

FlatBuffers schemas in `.fbs` files are compiled into Swift at build time. The generated code is then compiled as a regular Swift library.

## Third-Party Dependencies

The `third-party/` directory contains 26 vendored libraries. They're vendored (committed directly to the repo) rather than fetched at build time because:

1. Many require custom build configurations for iOS
2. Some have been patched for Telegram's specific needs
3. Vendoring eliminates supply chain risks from upstream repository changes
4. Build reproducibility — no network fetches during compilation

The most complex third-party BUILD files:

- **webrtc** — 149KB BUILD file. Google's WebRTC is massive, with dozens of `cc_library` targets for audio codecs, video codecs, network transport, and SRTP. Building it for iOS requires careful platform-specific configuration.
- **ffmpeg** — 331 lines. Pre-built static libraries (`libavutil.a`, `libavcodec.a`, `libavformat.a`, `libswresample.a`) with header declarations. FFmpeg is built separately (using its native configure/make system via CMake and Meson) and the products are committed.
- **boringssl** — Generated build rules from CMake. BoringSSL (Google's OpenSSL fork) provides all cryptographic primitives for MtProtoKit.

## Xcode Project Generation

Bazel can generate an Xcode project for IDE features:

```python
xcodeproj(
    name = "Telegram_xcodeproj",
    bazel_path = telegram_bazel_path,
    project_name = "Telegram",
    top_level_targets = [
        top_level_target(":Telegram", target_environments = ["device", "simulator"]),
        ":iOSAppUITestSuite",
    ],
    xcode_configurations = {
        "Debug": {"//command_line_option:compilation_mode": "dbg"},
        "Release": {"//command_line_option:compilation_mode": "opt"},
    },
    default_xcode_configuration = "Debug"
)
```

Running `bazel run //Telegram:Telegram_xcodeproj` generates a `.xcodeproj` that uses Bazel as the underlying build system but provides Xcode's code completion, navigation, and debugging features. The `rules_xcodeproj` integration is complex — it needs to generate index information for all 274 modules — which is why Telegram vendors a custom fork.

## The Build Flow End-to-End

Here's what happens when you run `python3 build-system/Make/Make.py build --configurationPath=... --buildNumber=12345 --configuration=release_arm64`:

1. **Parse configuration JSON.** `BuildConfiguration.from_json()` reads API keys, bundle ID, team ID, signing source.
2. **Resolve code signing.** Clone/decrypt the signing repo, or use Xcode-managed signing.
3. **Write `variables.bzl`.** Generate the configuration repository at `build-input/configuration-repository/`.
4. **Copy provisioning profiles.** Place `.mobileprovision` files at `build-input/configuration-repository/provisioning/`.
5. **Invoke Bazel.** Assemble the full command line:
   ```
   bazel-8.4.2 build //Telegram:Telegram \
     -c opt \
     --ios_multi_cpus=arm64 \
     --apple_generate_dsym \
     --features=swift.opt_uses_wmo \
     --features=swift.opt_uses_osize \
     --features=dead_strip \
     --jobs=10 \
     --define=buildNumber=12345 \
     --define=telegramVersion=12.4
   ```
6. **Bazel resolves the dependency graph.** Starting from `//Telegram:Telegram`, it discovers all transitive dependencies — 274 submodules, 26 third-party libraries, 5 frameworks, 6 extensions.
7. **Parallel compilation.** Independent modules compile in parallel across all CPU cores. Swift modules use persistent workers. Unchanged modules are skipped (cache hit).
8. **Framework assembly.** The five `ios_framework` targets are built, stripped, and signed.
9. **Extension compilation.** Each extension is compiled with its own set of framework dependencies.
10. **IPA creation.** Everything is assembled into a signed IPA with the correct bundle structure.
11. **DSYM generation.** Debug symbols are produced as separate dSYM bundles for crash symbolication.

## Navigating the Codebase

After 25 posts covering every layer of Telegram iOS, here's practical advice for finding your way around:

**Finding where a feature lives:**
1. Start with `TelegramUI/Sources/`. The AppDelegate, root controller, and major coordinators live here.
2. Feature-specific UI is usually in `TelegramUI/Components/`. Components like `ChatControllerInteraction`, `PeerInfoScreen`, `MediaEditorScreen` each have their own subdirectory.
3. Business logic is in `TelegramCore/Sources/TelegramEngine/`. The 19 engine subsystems (`Messages`, `Peers`, `Privacy`, `Payments`, etc.) organize all API operations.
4. Data models are in `TelegramCore/Sources/` and `Postbox/Sources/`.
5. Network-level code is in `MtProtoKit/Sources/` and `TelegramApi/Sources/`.

**Understanding a data flow:**
1. Find the UI in `TelegramUI/Components/`.
2. Trace the `Signal` it subscribes to — it usually comes from a `TelegramEngine` method.
3. The engine method delegates to an `_internal_` function that queries Postbox views or makes API calls.
4. Postbox views are defined in `Postbox/Sources/` — each view type subscribes to specific table changes.
5. API calls go through `TelegramCore` → `TelegramApi` → `MtProtoKit` → the network.

**The module dependency hierarchy** (simplified):

```
Layer 0: Foundation, UIKit, SwiftUI (Apple frameworks)
Layer 1: SSignalKit, SwiftSignalKit (reactive primitives)
Layer 2: Postbox (database), MtProtoKit (networking)
Layer 3: TelegramApi (generated API schema)
Layer 4: TelegramCore (business logic, TelegramEngine)
Layer 5: AccountContext (DI boundary)
Layer 6: Display, AsyncDisplayKit (rendering)
Layer 7: ComponentFlow (declarative components)
Layer 8: TelegramPresentationData (theming)
Layer 9: Feature modules (ChatListUI, PeerInfoUI, etc.)
Layer 10: TelegramUI (app assembly, AppDelegate)
```

Every module depends only on layers below it. This strict layering is enforced by Bazel's dependency declarations — you literally cannot import a module without declaring it in your BUILD file.

## What We've Learned

Over 25 posts, we've seen how a 274-module iOS app is architected for performance, maintainability, and scale:

1. **Custom reactive primitives** (SwiftSignalKit) instead of Combine — because the codebase predates Combine by years and the team needs precise control over threading and disposal.
2. **Custom persistence** (Postbox) instead of Core Data — because messaging requires atomic writes across multiple tables with real-time view subscriptions.
3. **Custom UI rendering** (AsyncDisplayKit/Display) instead of standard UIKit — because 60fps scrolling through media-heavy chat lists demands off-main-thread layout and rendering.
4. **Custom component framework** (ComponentFlow) instead of SwiftUI — because the team needs precise control over view lifecycle, animation timing, and performance characteristics.
5. **Bazel instead of Xcode's build system** — because 274 modules need reproducible, cacheable, parallel compilation with fine-grained dependency tracking.
6. **Everything vendored** — because a messaging app used by hundreds of millions of people can't afford supply chain dependencies that might break, change behavior, or introduce vulnerabilities.

The recurring theme is **control**. Telegram's team controls every layer of the stack because messaging is a domain where latency, reliability, and security are non-negotiable. The cost is complexity — 274 modules is a lot of code to maintain. The benefit is that every performance bottleneck, every edge case, every platform quirk can be addressed at the exact layer where it occurs, without waiting for upstream fixes or working around framework limitations.

This is not the right approach for most apps. But for a messaging app that serves hundreds of millions of daily active users across every iOS device from iPhone 6s to iPhone 17, it's the approach that works.
