---
title: "Media Downloads: From Cloud Resources to Cached Files"
description: "How Telegram iOS fetches, caches, and streams media — the MediaBox storage coordinator, resource type hierarchy, multi-datacenter downloads, CDN fallback, multipart uploads, file reference revalidation, and progressive loading."
published: 2025-07-09
tags:
  - Networking
toc: true
lang: en
abbrlink: 12-media-downloads
pin: 14
---

Every photo, video, sticker, voice message, and document in Telegram needs to be downloaded from the cloud, stored on disk, and served to the UI. This isn't a simple URL-to-file download — Telegram's media system handles multi-datacenter routing, CDN redirects, encrypted file decryption, file reference expiration, progressive streaming, and intelligent cache management.

This post traces the complete media pipeline from cloud resource identification to cached file delivery.

## MediaBox: The Storage Coordinator

`MediaBox` (in `Postbox/Sources/MediaBox.swift`) is the central coordinator for all media storage. Every resource — photos, videos, documents, thumbnails — flows through MediaBox.

```swift
// MediaBox.swift, line 137
public final class MediaBox {
    public let basePath: String
    public let isMainProcess: Bool

    private let statusQueue = Queue()
    private let concurrentQueue = Queue.concurrentDefaultQueue()
    public let dataQueue = Queue(name: "MediaBox-Data")
    private let cacheQueue = Queue()

    public let storageBox: StorageBox          // SQLite-backed persistent storage
    public let cacheStorageBox: StorageBox      // SQLite-backed cache storage

    private var statusContexts: [MediaResourceId: ResourceStatusContext] = [:]
    private var fileContexts: [MediaResourceId: MediaBoxFileContext] = [:]
}
```

**Four queues** handle different concerns:
- `statusQueue` — resource status updates (local vs remote, progress)
- `dataQueue` — serialized file I/O operations
- `concurrentQueue` — parallelizable disk reads
- `cacheQueue` — cached representation management

**Directory structure** on disk:

```
MediaBox/
├── cache/              # General cache (thumbnails, previews)
├── short-cache/        # Session-lived cache (temporary thumbnails)
├── animation-cache/    # Animation frame cache
├── storage/            # SQLite-backed persistent storage
└── cache-storage/      # SQLite-backed cache metadata
```

### The Fetch Injection Pattern

MediaBox doesn't know how to download files. Instead, it exposes a **dependency injection point**:

```swift
// MediaBox.swift, line 164
public var fetchResource: ((
    MediaResource,
    Signal<[(Range<Int64>, MediaBoxFetchPriority)], NoError>,
    MediaResourceFetchParameters?
) -> Signal<MediaResourceDataFetchResult, MediaResourceDataFetchError>)?
```

The app (TelegramCore) sets this closure during initialization. When MediaBox needs to fetch a resource, it calls this closure with:
1. The `MediaResource` to fetch
2. A signal of byte ranges with priorities (for progressive/partial loading)
3. Fetch parameters (tags, location context, content type)

The closure returns a signal that emits fetch results:

```swift
public enum MediaResourceDataFetchResult {
    case dataPart(resourceOffset: Int64, data: Data, range: Range<Int64>, complete: Bool)
    case resourceSizeUpdated(Int64)
    case progressUpdated(Float)
    case replaceHeader(data: Data, range: Range<Int64>)
    case moveLocalFile(path: String)
    case moveTempFile(file: TempBoxFile)
    case copyLocalItem(MediaResourceDataFetchCopyLocalItem)
    case reset
}
```

This separation means MediaBox (in the Postbox module) has no dependency on networking or API code. It just manages files and calls the injected fetch function when data is missing.

### File Storage Paths

Each resource gets two paths — partial and complete:

```swift
public struct ResourceStorePaths {
    public let partial: String    // basePath/{id}_partial
    public let complete: String   // basePath/{id}
}
```

Downloads write to the `_partial` file. When complete, the file is atomically renamed to the final path. This prevents the UI from reading a half-downloaded file.

### Resource Data Observation

MediaBox provides reactive observation of resource data:

```swift
public enum ResourceDataRequestOption {
    case complete(waitUntilFetchStatus: Bool)   // Wait for full download
    case incremental(waitUntilFetchStatus: Bool) // Stream as data arrives
}
```

Two subscriber types service these modes:
- `completeDataSubscribers` — notified only when the download finishes
- `progresiveDataSubscribers` — notified on every data chunk arrival

The `MediaResourceData` delivered to subscribers tells them what's available:

```swift
public struct MediaResourceData: Equatable {
    public let path: String      // File path on disk
    public let offset: Int64     // Start offset of available data
    public let size: Int64       // Size of available data
    public let complete: Bool    // Is the download finished?
}
```

## The MediaResource Protocol

Every downloadable resource implements the `MediaResource` protocol:

```swift
// MediaResource.swift
public protocol MediaResource: AnyObject {
    var id: MediaResourceId { get }        // Unique string identifier
    var size: Int64? { get }               // Known total size (nil if unknown)
    var streamable: Bool { get }           // Supports partial playback (default: false)
    var headerSize: Int32 { get }          // Bytes needed for format detection (default: 0)
    func isEqual(to: MediaResource) -> Bool
}
```

The `id` is a string like `"telegram-cloud-document-1-1234567890"` — globally unique, stable, used as the file name on disk.

`streamable` and `headerSize` control progressive loading behavior. A video file might declare `streamable = true` and `headerSize = 1024` — meaning the first 1KB (containing the moov atom) should be fetched first so playback can start while the rest downloads.

## Cloud Resource Types

TelegramCore defines a rich hierarchy of resource types in `SyncCore_CloudFileMediaResource.swift`. Each type knows which datacenter hosts it and how to construct the API request.

### CloudFileMediaResource

The base cloud file type for photos:

```swift
public final class CloudFileMediaResource: TelegramMediaResource {
    public let datacenterId: Int
    public let volumeId: Int64
    public let localId: Int32
    public let secret: Int64
    public let size: Int64?
    public let fileReference: Data?

    public var id: MediaResourceId {
        return MediaResourceId("telegram-cloud-file-\(datacenterId)-\(volumeId)-\(localId)-\(secret)")
    }
}
```

### CloudDocumentMediaResource

For documents and files with document-level access control:

```swift
public final class CloudDocumentMediaResource: TelegramMediaResource {
    public let datacenterId: Int
    public let fileId: Int64
    public let accessHash: Int64
    public let size: Int64?
    public let fileReference: Data?
    public let fileName: String?
}
```

### CloudPhotoSizeMediaResource

For photo thumbnails at specific sizes:

```swift
public final class CloudPhotoSizeMediaResource: TelegramMediaResource {
    public let datacenterId: Int
    public let photoId: Int64
    public let accessHash: Int64
    public let sizeSpec: String       // "s", "m", "x", "y", "w", "a"
    public let size: Int64
    public let fileReference: Data?
}
```

The `sizeSpec` maps to Telegram's standard photo sizes:
- `"s"` — 100x100 thumbnail
- `"m"` — 320x320 medium
- `"x"` — 800x800 large
- `"y"` — 1280x1280 extra large
- `"w"` — 2560x2560 maximum

### SecretFileMediaResource

For end-to-end encrypted files:

```swift
public final class SecretFileMediaResource: TelegramMediaResource {
    public let fileId: Int64
    public let accessHash: Int64
    public let containerSize: Int64
    public let decryptedSize: Int64
    public let datacenterId: Int
    public let key: SecretFileEncryptionKey   // AES key + IV

    public var streamable: Bool { return false }  // Can't seek in encrypted files
}
```

Secret files can't be streamed because decryption uses AES-IGE with chained IVs — you must decrypt sequentially from the start.

### Other Resource Types

- `CloudPeerPhotoSizeMediaResource` — user/chat profile pictures
- `CloudStickerPackThumbnailMediaResource` — sticker pack previews
- `CloudDocumentSizeMediaResource` — document thumbnails
- `LocalFileMediaResource` — on-device files (camera roll imports)
- `LocalFileReferenceMediaResource` — files referenced by path

All cloud resources implement `PostboxDecoder`/`PostboxEncoder` for persistence in Postbox. When a message is stored, its media resources are serialized alongside it.

## The Download Pipeline

### Entry Point: fetchedMediaResource

The high-level API for triggering a download:

```swift
// FetchedMediaResource.swift, line 28
public func fetchedMediaResource(
    mediaBox: MediaBox,
    userLocation: MediaResourceUserLocation,
    userContentType: MediaResourceUserContentType,
    reference: MediaResourceReference,
    range: (Range<Int64>, MediaBoxFetchPriority)? = nil,
    statsCategory: MediaResourceStatsCategory = .generic,
    continueInBackground: Bool = false
) -> Signal<FetchResourceSourceType, FetchResourceError>
```

The `MediaResourceReference` is an enum that captures the context of why this resource is being fetched:

```swift
public enum MediaResourceReference {
    case media(media: AnyMediaReference, resource: MediaResource)
    case avatar(peer: PeerReference, resource: MediaResource)
    case stickerPack(stickerPack: StickerPackReference, media: AnyMediaReference)
    case wallpaper(wallpaper: WallpaperReference?, resource: MediaResource)
    case story(peer: PeerReference, id: Int32, media: AnyMediaReference)
    // ... more cases
}
```

This context matters for **file reference revalidation** (explained below).

### MultipartFetch: The Download State Machine

For actual network downloads, `MultipartFetch.swift` implements the core download logic. It handles:

**Decryption state** for secret files:

```swift
// MultipartFetch.swift, line 10
private final class MultipartDownloadState {
    let aesKey: Data
    var aesIv: Data
    let decryptedSize: Int64?
    var currentSize: Int64 = 0

    func transform(offset: Int64, data: Data) -> Data {
        if self.aesKey.count != 0 {
            var decryptedData = data
            assert(offset == self.currentSize)   // Must decrypt sequentially!
            decryptedData.withUnsafeMutableBytes { bytes in
                self.aesIv.withUnsafeMutableBytes { iv in
                    MTAesDecryptBytesInplaceAndModifyIv(bytes, count, self.aesKey, iv)
                }
            }
            // Trim padding on the last chunk
            if self.currentSize + Int64(decryptedData.count) > self.decryptedSize! {
                decryptedData.count = Int(self.decryptedSize! - self.currentSize)
            }
            return decryptedData
        }
        return data
    }
}
```

The AES IV is modified in-place after each block — this is AES-IGE mode where each block's IV depends on the previous ciphertext. The `assert(offset == self.currentSize)` enforces sequential decryption.

**CDN fallback:** When a file is popular, Telegram redirects downloads to partner CDNs:

```swift
private enum MultipartFetchDownloadError {
    case generic
    case switchToCdn(id: Int32, token: Data, key: Data, iv: Data, partHashes: [Int64: Data])
    case reuploadToCdn(masterDatacenterId: Int32, token: Data)
    case revalidateMediaReference
    case hashesMissing
    case fatal
}
```

The CDN flow:
1. Client requests file from origin datacenter
2. Server responds with `upload.fileCdnRedirect` containing CDN ID, token, encryption key/IV, and part hashes
3. Client switches to CDN and downloads encrypted parts
4. Client decrypts using the provided key/IV and verifies part hashes
5. If the CDN is missing parts, client sends `upload.reuploadCdnFile` to tell the origin to push to the CDN

**Multi-datacenter routing** via `DownloadWrapper`:

```swift
private struct DownloadWrapper {
    let consumerId: Int64
    let resourceId: String?
    let datacenterId: Int32
    let isCdn: Bool
    let network: Network
    let useMainConnection: Bool
}
```

Each resource specifies which datacenter hosts it. The download wrapper creates an MTProto connection to that specific datacenter, handling auth token exchange if it's not the user's home DC.

### Intelligent Part Sizing

Downloads are split into parts with adaptive sizing:

- Small files (< 10MB): 128KB parts using `upload.getFile`
- Large files (> 10MB): Up to 1MB parts using `upload.getFile` with the "big parts" protocol
- Part boundaries are aligned to power-of-two offsets for efficient caching

### Hash Verification

Every downloaded part is verified against SHA-256 hashes provided by the server. Parts are grouped into 128KB clusters, each with its own hash. If verification fails, the part is re-downloaded.

## File References: The Expiring Token Problem

Telegram's API uses **file references** — opaque tokens that must accompany every file download request. They expire after ~24 hours or when the associated entity changes.

```swift
// FetchedMediaResource.swift, line 16
public final class TelegramCloudMediaResourceFetchInfo: MediaResourceFetchInfo {
    public let reference: MediaResourceReference
    public let preferBackgroundReferenceRevalidation: Bool
    public let continueInBackground: Bool
}
```

When a download fails with `FILE_REFERENCE_EXPIRED`, the system needs to **revalidate** — re-fetch the entity (message, peer profile, sticker pack) that contains the file to get a fresh token.

This is why `MediaResourceReference` exists. It tells the revalidation system *where the file came from*:
- `.media(message, resource)` → re-fetch the message via `messages.getMessages`
- `.avatar(peer, resource)` → re-fetch the peer's full info
- `.stickerPack(pack, media)` → re-fetch the sticker pack
- `.story(peer, id, media)` → re-fetch the story

The `MediaReferenceRevalidationContext` deduplicates concurrent revalidation requests — if 10 images from the same message expire simultaneously, only one API call is made.

After revalidation:
1. The fresh entity arrives with updated `fileReference` fields
2. The download system extracts the new reference from the resource
3. The download retries automatically with the new token

This is invisible to the UI — it just sees a brief stall in the progress bar.

## Multipart Upload

Uploading follows a similar pattern in `MultipartUpload.swift`.

```swift
public enum MultipartUploadSource {
    case resource(MediaResourceReference)
    case data(Data)
    case custom(Signal<MediaResourceData, NoError>)
    case tempFile(TempBoxFile)
}
```

**Adaptive part sizing:**

```swift
// Upload part sizes (adaptive based on total file size):
// < 10MB:   128KB parts, "small file" protocol
// 10-512MB: 256KB parts, "big file" protocol
// > 512MB:  512KB-1MB parts, "big file" protocol
```

**Parallel upload:** Up to 30 parts can be in-flight simultaneously for large files:

```swift
// The upload manager tracks three states per part:
// - uploadingParts: currently sending to server
// - uploadedParts: server acknowledged, not yet committed
// - committedOffset: contiguous range fully acknowledged
```

**Encryption for secret chats:**

```swift
// MultipartUpload.swift, line 27
private final class MultipartUploadState {
    let aesKey: Data
    var aesIv: Data
    var effectiveSize: Int64 = 0

    func transform(data: Data) -> Data {
        if self.aesKey.count != 0 {
            var encryptedData = data
            // Pad to 16-byte boundary (AES block size)
            while encryptedData.count % 16 != 0 { encryptedData.count += 1 }
            // Fill padding with random bytes
            arc4random_buf(bytes + encryptedData.count - paddingSize, paddingSize)
            // Encrypt in-place, modifying IV for next block
            MTAesEncryptBytesInplaceAndModifyIv(bytes, count, self.aesKey, iv)
            return encryptedData
        }
        return data
    }
}
```

The upload result provides the handle needed to attach the file to a message:

```swift
enum MultipartUploadResult {
    case progress(Float)
    case inputFile(Api.InputFile)         // For normal files
    case inputSecretFile(Api.InputEncryptedFile, Int64, SecretFileEncryptionKey)  // For E2E
}
```

## Cache Management

### Two-Tier Cache with TTL

MediaBox manages two cache tiers:

```swift
public enum CachedMediaRepresentationKeepDuration {
    case general       // Long-lived (days/weeks, configurable)
    case shortLived    // Session-based (cleared more aggressively)
}
```

The user controls cache limits through settings:

```swift
public func setMaxStoreTimes(general: Int32, shortLived: Int32, gigabytesLimit: Int32)
```

`TimeBasedCleanup` periodically scans the cache directories and evicts files based on:
1. **Age** — files older than the configured TTL
2. **Total size** — when total cache exceeds `gigabytesLimit`, oldest files are removed (LRU)

The `StorageBox` (SQLite-backed) tracks access times for each file, enabling efficient LRU eviction without scanning the filesystem.

### Cached Representations

MediaBox supports **derived representations** — thumbnails, blurred previews, and transcoded versions of original resources:

```swift
public protocol CachedMediaResourceRepresentation {
    var uniqueId: String { get }
    var keepDuration: CachedMediaRepresentationKeepDuration { get }
}
```

For example, a photo resource might have cached representations for:
- Blurred placeholder (10x10 thumbnail)
- Album thumbnail (200x200)
- Full-screen display (based on device resolution)

Each representation is stored separately with its own cache entry. The `fetchCachedResourceRepresentation` closure (injected like `fetchResource`) handles generating these on demand.

## Progressive Loading and Streaming

### How Streaming Works

When `streamable = true`, the download pipeline handles random-access requests:

1. The video player needs bytes at offset X
2. MediaBox emits a range request `[(X..<X+N, .maximum)]` through the ranges signal
3. The fetch function translates this to an `upload.getFile` call with the specific offset
4. Data arrives and is written to the partial file at the correct offset
5. The player reads from the partial file as data becomes available

The `headerSize` property optimizes this — the first N bytes (typically 1KB for video moov atoms) are always fetched first, even if the player requests a later offset. This allows the player to parse format metadata before seeking.

### replaceHeader

Some progressive downloads need to update the file header after the initial fetch:

```swift
case replaceHeader(data: Data, range: Range<Int64>)
```

This handles cases where the server provides a revised header (e.g., with correct duration/keyframe index) after the initial download starts.

## Complete Download Flow

Tracing a photo download from tap to display:

1. **UI** calls `fetchedMediaResource(mediaBox:reference:.media(message, photoResource))`
2. **FetchedMediaResource** wraps the resource with `TelegramCloudMediaResourceFetchInfo`
3. **MediaBox** checks if the file exists at `basePath/{resourceId}` → if so, signals `.Local`
4. **MediaBox** calls the injected `fetchResource` closure with the resource and ranges signal
5. **MultipartFetch** examines the resource type → `CloudPhotoSizeMediaResource` → creates `Api.InputFileLocation`
6. **DownloadWrapper** opens an MTProto connection to the resource's datacenter
7. **MTProto** sends `upload.getFile(location, offset: 0, limit: 131072)` (128KB part)
8. **Server** responds with file data (or CDN redirect, or FILE_REFERENCE_EXPIRED)
9. **MultipartFetch** verifies hash, emits `.dataPart(offset: 0, data: ..., complete: true)`
10. **MediaBox** writes data to `basePath/{resourceId}_partial`, renames to `basePath/{resourceId}`
11. **MediaBox** notifies subscribers with `MediaResourceData(path: ..., complete: true)`
12. **UI** reads the file and displays the image

If step 8 returns `FILE_REFERENCE_EXPIRED`:
- The download pauses
- `MediaReferenceRevalidationContext` re-fetches the message containing the photo
- The message's updated `fileReference` is extracted
- The download retries with the fresh reference
- Steps 7-11 continue as normal

The entire pipeline — from user tap to displayed image — typically takes 50-200ms on a fast connection. On slow connections, the progressive data subscribers ensure the UI updates incrementally as data arrives.

## Key Takeaways

1. **Dependency inversion.** MediaBox lives in Postbox (no networking dependency). Download logic is injected at runtime. This makes MediaBox testable and reusable across different networking implementations.

2. **Dual-path storage.** Partial files prevent UI from reading corrupt data. Atomic rename guarantees file integrity.

3. **Resource types carry routing info.** Each resource knows its datacenter, so the download system can route directly to the right server without a lookup step.

4. **File references are the hardest part.** The revalidation system is complex but necessary — without it, cached messages with expired tokens would silently fail to load media. The `MediaResourceReference` enum ensures every resource can be revalidated through its parent entity.

5. **Secret files are special.** AES-IGE chained decryption prevents seeking and streaming. Secret files must be downloaded completely before they can be displayed.

6. **Cache management is user-controllable.** The two-tier TTL system with configurable size limits gives users control over storage usage while keeping the most-accessed files warm.
