---
title: "TelegramApi: The Generated Type System"
description: "How Telegram iOS turns the TL schema into 95,000 lines of type-safe Swift — constructor ID registries, binary serialization, the triple-return function pattern, flags-based optional fields, and secret chat layer versioning."
published: 2025-07-08
tags:
  - Networking
toc: true
lang: en
abbrlink: 11-telegram-api
pin: 15
---

Every API call Telegram makes — fetching messages, sending photos, checking passwords — goes through a layer of **generated Swift code** that handles binary serialization and deserialization. This layer lives in the `TelegramApi` module: 50 Swift files totaling over 95,000 lines, all generated from Telegram's TL (Type Language) schema.

This post explains how the generated code works, why it's structured the way it is, and how TelegramCore consumes it.

## The TL Schema

Telegram's API is defined in TL (Type Language), a schema language invented by Telegram. Each type and function has a **constructor ID** — a 32-bit integer that uniquely identifies it on the wire. The TL schema defines hundreds of types (User, Message, Chat, Update, ...) and hundreds of RPC functions (messages.getHistory, auth.sendCode, ...).

The iOS codebase doesn't include the `.tl` schema files or the generator tool. Instead, the generated Swift files are committed directly as source code. When the API changes (new fields, new methods), someone regenerates the files and commits the diff. This means the generated code is stable, auditable, and doesn't require build-time code generation.

## Module Structure

```
TelegramApi/Sources/
├── Api0.swift            # Namespace definition + constructor registry (parsers dict)
├── Api1.swift … Api40.swift   # Generated types and functions, split across 41 files
├── Buffer.swift          # Binary read/write primitives
├── DeserializeFunctionResponse.swift  # Response parser wrapper
├── TelegramApiLogger.swift            # Debug logging
├── SecretApiLayer8.swift              # Secret chat protocol v8
├── SecretApiLayer46.swift             # Secret chat protocol v46
├── SecretApiLayer73.swift             # Secret chat protocol v73
├── SecretApiLayer101.swift            # Secret chat protocol v101
└── SecretApiLayer144.swift            # Secret chat protocol v144
```

The 41 `Api*.swift` files split generated types across files to avoid hitting Swift compiler limits on single-file size. `Api40.swift` alone is 732KB with hundreds of function definitions.

## The Api Namespace

`Api0.swift` defines the top-level enum hierarchy:

```swift
// Api0.swift, line 2
public enum Api {
    // Response type namespaces (22 sub-namespaces)
    public enum account {}
    public enum auth {}
    public enum channels {}
    public enum messages {}
    public enum upload {}
    // ... 17 more

    // RPC function namespaces
    public enum functions {
        public enum account {}
        public enum auth {}
        public enum channels {}
        public enum messages {}
        public enum upload {}
        // ... 17 more
    }
}
```

Two parallel hierarchies:
- `Api.messages.*` — response/result types (what the server returns)
- `Api.functions.messages.*` — RPC functions (what the client calls)

Top-level types like `Api.User`, `Api.Message`, `Api.Chat` live directly in the `Api` enum without a sub-namespace.

## The Constructor ID Registry

Every TL type has a unique 32-bit constructor ID. When the server sends a binary response, the first 4 bytes identify which type follows. To parse it, the client needs a lookup table.

`Api0.swift` contains a massive dictionary mapping constructor IDs to parser closures:

```swift
// Api0.swift, line 50
fileprivate let parsers: [Int32 : (BufferReader) -> Any?] = {
    var dict: [Int32 : (BufferReader) -> Any?] = [:]
    dict[-1471112230] = { return $0.readInt32() }          // int
    dict[570911930]   = { return $0.readInt64() }           // long
    dict[571523412]   = { return $0.readDouble() }          // double
    dict[-1255641564] = { return parseString($0) }          // string
    dict[-1132882121] = { return Api.Bool.parse_boolFalse($0) }
    dict[-1720552011] = { return Api.Bool.parse_boolTrue($0) }
    dict[829899656]   = { return Api.User.parse_user($0) }
    // ... thousands more entries
}()
```

This dictionary is built once at process startup (it's a lazily-initialized file-level `let`). Every known constructor ID maps to a closure that reads a `BufferReader` and returns the parsed type.

The global parse function dispatches through this registry:

```swift
public static func parse(_ reader: BufferReader, signature: Int32) -> Any? {
    if let parser = parsers[signature] {
        return parser(reader)
    } else {
        telegramApiLog("Type constructor \(String(UInt32(bitPattern: signature), radix: 16)) not found")
        return nil
    }
}
```

Unknown constructor IDs are logged but don't crash — the API adds new types frequently, and the client may lag behind the server.

## Binary Serialization: The Buffer Class

All serialization goes through `Buffer`, a manual memory management class:

```swift
// Buffer.swift, line 200
public class Buffer: CustomStringConvertible {
    public var data: UnsafeMutableRawPointer?
    public var _size: UInt = 0
    private var capacity: UInt = 0
    private let freeWhenDone: Bool
}
```

This is raw `malloc`/`realloc`/`free` — no Foundation `Data` overhead. The class provides typed append methods:

```swift
buffer.appendInt32(value)   // 4 bytes, little-endian
buffer.appendInt64(value)   // 8 bytes, little-endian
buffer.appendDouble(value)  // 8 bytes, IEEE 754
buffer.appendBytes(ptr, length: n)
```

Reading uses `BufferReader`, which tracks a position cursor:

```swift
reader.readInt32()    // -> Int32?
reader.readInt64()    // -> Int64?
reader.readDouble()   // -> Double?
reader.readBuffer(n)  // -> Buffer? (n bytes)
reader.skip(n)        // advance cursor
```

Every read method returns an optional — if the buffer is exhausted, parsing fails gracefully with `nil`.

### TL String/Bytes Encoding

TL has its own string encoding, distinct from both protobuf and JSON:

```swift
// Buffer.swift, line 80
public func serializeBytes(_ value: Buffer, buffer: Buffer, boxed: Bool) {
    var length: Int32 = Int32(value.size)
    var padding: Int32 = 0
    if length >= 254 {
        var tmp: UInt8 = 254
        buffer.appendBytes(&tmp, length: 1)      // Marker byte
        buffer.appendBytes(&length, length: 3)    // 3-byte length (little-endian)
        padding = roundUp(length, multiple: 4) - length
    } else {
        buffer.appendBytes(&length, length: 1)    // 1-byte length
        padding = roundUp(length + 1, multiple: 4) - (length + 1)
    }
    buffer.appendBytes(value.data!, length: UInt(length))
    // Pad to 4-byte boundary
    var zero: UInt8 = 0
    for _ in 0..<padding { buffer.appendBytes(&zero, length: 1) }
}
```

Short strings (< 254 bytes): 1-byte length prefix + data + padding to 4-byte boundary.
Long strings (>= 254 bytes): `0xFE` marker + 3-byte length + data + padding.

Parsing mirrors this:

```swift
// Buffer.swift, line 168
public func parseBytes(_ reader: BufferReader) -> Buffer? {
    if let tmp = reader.readBytesAsInt32(1) {
        if tmp == 254 {
            let len = reader.readBytesAsInt32(3)  // 3-byte length
            let padding = roundUp(length, multiple: 4) - length
        } else {
            let length = Int(tmp)                 // 1-byte length
            let padding = roundUp(length + 1, multiple: 4) - (length + 1)
        }
        let buffer = reader.readBuffer(length)
        reader.skip(paddingBytes)
        return buffer
    }
}
```

Everything is 4-byte aligned. This makes parsing fast (no unaligned reads) and predictable.

### Boxed vs Unboxed

The `boxed` parameter on every serialize function controls whether to prepend the constructor ID:

```swift
func serializeInt32(_ value: Int32, buffer: Buffer, boxed: Bool) {
    if boxed {
        buffer.appendInt32(-1471112230) // int constructor ID
    }
    buffer.appendInt32(value)
}
```

**Boxed** (with constructor ID): Used when the receiver doesn't know the type in advance — polymorphic fields, top-level responses.

**Unboxed** (without constructor ID): Used inside known structures where the field type is fixed by the schema — no need to waste 4 bytes identifying it.

## Type Generation Pattern

Every TL type becomes a Swift enum with associated values. Here's the pattern through a real example.

### Simple Type: Bool

The simplest polymorphic type — two constructors, no fields:

```swift
public extension Api {
    enum Bool: TypeConstructorDescription {
        case boolFalse    // Constructor ID: -1132882121 (0xBC799737)
        case boolTrue     // Constructor ID: -1720552011 (0x997275B5)

        public func serialize(_ buffer: Buffer, _ boxed: Swift.Bool) {
            switch self {
            case .boolFalse:
                if boxed { buffer.appendInt32(-1132882121) }
            case .boolTrue:
                if boxed { buffer.appendInt32(-1720552011) }
            }
        }
    }
}
```

No data, just the constructor ID distinguishing true from false.

### Complex Type: User

`Api.User` has two constructors — `user` (with 22 fields) and `userEmpty` (with just an ID):

```swift
// Api29.swift, line 684
public extension Api {
    enum User: TypeConstructorDescription {
        public class Cons_user {
            public var flags: Int32
            public var flags2: Int32
            public var id: Int64
            public var accessHash: Int64?
            public var firstName: String?
            public var lastName: String?
            public var username: String?
            public var phone: String?
            public var photo: Api.UserProfilePhoto?
            public var status: Api.UserStatus?
            public var botInfoVersion: Int32?
            public var restrictionReason: [Api.RestrictionReason]?
            public var botInlinePlaceholder: String?
            public var langCode: String?
            public var emojiStatus: Api.EmojiStatus?
            public var usernames: [Api.Username]?
            public var storiesMaxId: Api.RecentStory?
            public var color: Api.PeerColor?
            public var profileColor: Api.PeerColor?
            public var botActiveUsers: Int32?
            public var botVerificationIcon: Int64?
            public var sendPaidMessagesStars: Int64?
            // ... init with all 22 parameters
        }

        public class Cons_userEmpty {
            public var id: Int64
        }

        case user(Cons_user)
        case userEmpty(Cons_userEmpty)
    }
}
```

Each constructor's fields are wrapped in a class (`Cons_user`, `Cons_userEmpty`). The enum cases wrap these classes as associated values. This pattern is chosen over struct associated values because classes allow reference semantics — large types like `Cons_user` (22 fields) aren't copied when pattern-matched.

### The Flags Pattern

Most TL types have optional fields controlled by a bitmask. The `flags` (and sometimes `flags2`) field is an `Int32` where each bit controls the presence of a specific optional field:

```swift
// Serialization: only write the field if its flag bit is set
public func serialize(_ buffer: Buffer, _ boxed: Swift.Bool) {
    switch self {
    case .user(let _data):
        if boxed { buffer.appendInt32(829899656) }  // Constructor ID
        serializeInt32(_data.flags, buffer: buffer, boxed: false)
        serializeInt32(_data.flags2, buffer: buffer, boxed: false)
        serializeInt64(_data.id, buffer: buffer, boxed: false)
        if Int(_data.flags) & Int(1 << 0) != 0 {   // Bit 0: accessHash
            serializeInt64(_data.accessHash!, buffer: buffer, boxed: false)
        }
        if Int(_data.flags) & Int(1 << 1) != 0 {   // Bit 1: firstName
            serializeString(_data.firstName!, buffer: buffer, boxed: false)
        }
        if Int(_data.flags) & Int(1 << 5) != 0 {   // Bit 5: photo
            _data.photo!.serialize(buffer, true)     // Boxed (polymorphic)
        }
        // ... more optional fields
    }
}
```

The flags field also encodes boolean properties that have no associated data. For example, in the `User` type:
- Bit 10 = `self` (is this the current user?)
- Bit 11 = `contact` (is this user in contacts?)
- Bit 13 = `bot` (is this a bot?)

These bits are checked directly from the flags field without a separate boolean.

### Deserialization

Each constructor has a static `parse_*` method:

```swift
public static func parse_user(_ reader: BufferReader) -> User? {
    var _1: Int32?     // flags
    _1 = reader.readInt32()
    var _2: Int32?     // flags2
    _2 = reader.readInt32()
    var _3: Int64?     // id
    _3 = reader.readInt64()
    var _4: Int64?     // accessHash (optional)
    if Int(_1!) & Int(1 << 0) != 0 {
        _4 = reader.readInt64()
    }
    var _5: String?    // firstName (optional)
    if Int(_1!) & Int(1 << 1) != 0 {
        _5 = parseString(reader)
    }
    // ... read all fields

    // Validate: required fields must be non-nil, optional fields must match flags
    let _c1 = _1 != nil
    let _c2 = _2 != nil
    let _c3 = _3 != nil
    let _c4 = (Int(_1!) & Int(1 << 0) == 0) || _4 != nil
    let _c5 = (Int(_1!) & Int(1 << 1) == 0) || _5 != nil
    // ...
    if _c1 && _c2 && _c3 && _c4 && _c5 /* && ... */ {
        return Api.User.user(Cons_user(
            flags: _1!, flags2: _2!, id: _3!,
            accessHash: _4, firstName: _5, /* ... */
        ))
    }
    return nil  // Parse failure
}
```

The validation pattern `(flag not set) || (value parsed)` ensures that optional fields are only required when the flags say they should be present. If any required field is nil, the entire parse returns nil.

### Vector Parsing

Arrays are serialized as TL vectors: a magic number (`481674261`), a count, and then each element:

```swift
// Serialization
buffer.appendInt32(481674261)                    // Vector magic
buffer.appendInt32(Int32(items.count))           // Element count
for item in items {
    item.serialize(buffer, true)                 // Each element, boxed
}

// Deserialization (generic)
public static func parseVector<T>(
    _ reader: BufferReader,
    elementSignature: Int32,
    elementType: T.Type
) -> [T]? {
    guard let count = reader.readInt32() else { return nil }
    var array = [T]()
    for _ in 0..<count {
        var signature = elementSignature
        if elementSignature == 0 {               // Dynamic type: read constructor ID
            guard let sig = reader.readInt32() else { return nil }
            signature = sig
        }
        guard let item = Api.parse(reader, signature: signature) as? T else { return nil }
        array.append(item)
    }
    return array
}
```

When `elementSignature` is 0, each element has its own constructor ID (heterogeneous arrays). When it's non-zero, all elements share the same type (homogeneous arrays like `[Int32]`).

## The Function Triple

Every generated API function returns a tuple of three values:

```swift
(FunctionDescription, Buffer, DeserializeFunctionResponse<T>)
```

Here's a concrete example — `messages.getHistory`:

```swift
// Api40.swift, line 6802
public extension Api.functions.messages {
    static func getHistory(
        peer: Api.InputPeer,
        offsetId: Int32,
        offsetDate: Int32,
        addOffset: Int32,
        limit: Int32,
        maxId: Int32,
        minId: Int32,
        hash: Int64
    ) -> (FunctionDescription, Buffer, DeserializeFunctionResponse<Api.messages.Messages>) {
        let buffer = Buffer()
        buffer.appendInt32(1143203525)             // Function constructor ID
        peer.serialize(buffer, true)               // Boxed: InputPeer is polymorphic
        serializeInt32(offsetId, buffer: buffer, boxed: false)
        serializeInt32(offsetDate, buffer: buffer, boxed: false)
        serializeInt32(addOffset, buffer: buffer, boxed: false)
        serializeInt32(limit, buffer: buffer, boxed: false)
        serializeInt32(maxId, buffer: buffer, boxed: false)
        serializeInt32(minId, buffer: buffer, boxed: false)
        serializeInt64(hash, buffer: buffer, boxed: false)

        return (
            FunctionDescription(
                name: "messages.getHistory",
                parameters: [("peer", String(describing: peer)), ...]
            ),
            buffer,
            DeserializeFunctionResponse { (buffer: Buffer) -> Api.messages.Messages? in
                let reader = BufferReader(buffer)
                var result: Api.messages.Messages?
                if let signature = reader.readInt32() {
                    result = Api.parse(reader, signature: signature)
                        as? Api.messages.Messages
                }
                return result
            }
        )
    }
}
```

The three components:

1. **`FunctionDescription`** — Name and stringified parameters, used for logging. When the network layer logs an outgoing request, it calls `apiFunctionDescription(of:)` which formats the description into a readable string, with sensitive fields (message text, phone numbers) redacted.

2. **`Buffer`** — The serialized binary payload. This is exactly the bytes that will be encrypted and sent over MTProto.

3. **`DeserializeFunctionResponse<T>`** — A closure that knows how to parse the expected response type. It reads the constructor ID from the response buffer and dispatches through the global parser registry.

These three are defined in `DeserializeFunctionResponse.swift`:

```swift
public final class FunctionDescription {
    public let name: String
    public let parameters: [(String, Any)]
}

public final class DeserializeFunctionResponse<T> {
    private let f: (Buffer) -> T?

    public func parse(_ buffer: Buffer) -> T? {
        return self.f(buffer)
    }
}

public protocol TypeConstructorDescription {
    func descriptionFields() -> (String, [(String, Any)])
}
```

## How TelegramCore Consumes the API

The calling pattern in TelegramCore is elegant:

```swift
// Holes.swift — fetching message history
let request = network.request(
    Api.functions.messages.getHistory(
        peer: inputPeer,
        offsetId: offsetId,
        offsetDate: 0,
        addOffset: addOffset,
        limit: Int32(selectedLimit),
        maxId: maxId,
        minId: minId,
        hash: 0
    )
)
```

The compiler enforces type safety end-to-end:
- `Api.functions.messages.getHistory` expects specific parameter types (`Api.InputPeer`, `Int32`, etc.)
- It returns `(FunctionDescription, Buffer, DeserializeFunctionResponse<Api.messages.Messages>)`
- `network.request()` accepts this triple and returns `Signal<Api.messages.Messages, MTRpcError>`
- The caller pattern-matches the response:

```swift
request |> mapToSignal { result -> Signal<Void, NoError> in
    let messages: [Api.Message]
    let chats: [Api.Chat]
    let users: [Api.User]

    switch result {
    case let .messages(data):
        messages = data.messages
        chats = data.chats
        users = data.users
    case let .messagesSlice(data):
        messages = data.messages
        chats = data.chats
        users = data.users
    case let .channelMessages(data):
        messages = data.messages
        chats = data.chats
        users = data.users
    case .messagesNotModified:
        return .complete()
    }
    // Process messages, chats, users...
}
```

There's no JSON parsing, no `Codable`, no string keys. The binary is parsed directly into Swift enums with associated values. A typo in a field name is a compile error, not a runtime crash.

## Sensitive Data Redaction

The logging layer in `Serialization.swift` redacts sensitive fields automatically:

```swift
// Serialization.swift, line 17
private let redactChildrenOfType: [String: Set<String>] = [
    "Message.message": Set(["message"]),          // Message text
    "User.user": Set(["phone"]),                  // Phone numbers
    "InputContact.inputPhoneContact": Set(["phone"]),
    "DraftMessage.draftMessage": Set(["message"]),
]

private let redactFunctionParameters: [String: Set<String>] = [
    "messages.sendMessage": Set(["message"]),     // Outgoing message text
    "messages.sendMedia": Set(["message"]),
    "messages.saveDraft": Set(["message"]),
]
```

When `Logger.shared.redactSensitiveData` is true, these fields are replaced with `[[redacted]]` in log output. This prevents message content and phone numbers from appearing in debug logs.

## Secret Chat Layers

Telegram's end-to-end encrypted chats use a separate type system that's versioned independently. The module includes five layer versions:

| File | Layer | Size |
|------|-------|------|
| `SecretApiLayer8.swift` | 8 | 23 KB |
| `SecretApiLayer46.swift` | 46 | 67 KB |
| `SecretApiLayer73.swift` | 73 | 67 KB |
| `SecretApiLayer101.swift` | 101 | 70 KB |
| `SecretApiLayer144.swift` | 144 | 73 KB |

Each layer is a self-contained namespace with its own parser registry:

```swift
public struct SecretApi101 {
    fileprivate static let parsers: [Int32 : (BufferReader) -> Any?] = { ... }()

    public static func parse(_ buffer: Buffer) -> Any? {
        let reader = BufferReader(buffer)
        if let signature = reader.readInt32() {
            return parse(reader, signature: signature)
        }
        return nil
    }
}
```

Keeping all five layers allows the app to communicate with peers running older versions. When starting a secret chat, the two parties negotiate the highest common layer. The app can simultaneously handle conversations on different layers.

The types grow with each version — `SecretApi144` includes types for reactions, message effects, and extended media that `SecretApi8` knows nothing about.

## Design Insights

### Why Enums, Not Structs or Classes?

Using Swift enums for polymorphic types gives pattern matching at the call site:

```swift
switch user {
case .user(let data):
    print(data.firstName ?? "Unknown")
case .userEmpty:
    print("Ghost user")
}
```

The compiler warns if a case is unhandled. When the server adds a new constructor (like `userPremium`), the app won't compile until every switch statement handles it — or has a default case.

### Why Classes for Constructor Data?

The `Cons_user` classes (not structs) avoid value-type copying overhead. A `User` with 22 fields would require deep copies on every Swift assignment if it were a struct. With a class, pattern matching only copies a reference.

### Why Pre-Generated, Not Runtime?

Committing generated code means:
- No build-time dependency on a code generator
- Git diffs show exactly what API changed
- The compiler catches type mismatches at build time
- No runtime reflection or dynamic dispatch for serialization

The tradeoff is 95,000 lines of generated code in the repository. But at Telegram's scale, correctness matters more than repository aesthetics.

### Why Not Codable?

Swift's `Codable` protocol is designed for JSON/property list round-tripping. TL serialization has features `Codable` can't express:
- **Constructor IDs** — type-discriminated unions where the discriminator is a 4-byte integer, not a string key
- **Flag-based optionals** — a bitmask controls which fields are present, not JSON null
- **4-byte alignment** — padding rules for strings and bytes
- **Boxed vs unboxed** — the same type serializes differently depending on context

Building on `Codable` would require fighting the framework at every turn. The generated code is simpler and faster.

## Summary

The TelegramApi module is a direct translation of Telegram's TL schema into Swift's type system. The patterns — constructor ID registry, flags-based optionals, the function triple, boxed vs unboxed serialization — are all driven by the TL protocol's design. The result is a type-safe, zero-reflection, compile-time-checked API layer that can serialize and deserialize any Telegram protocol message in microseconds.

In the next post, we'll see how this API layer connects to the media download system — where resources identified by datacenter IDs and file references are fetched, cached, and streamed.
