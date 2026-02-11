---
title: "MTProto: Telegram's Custom Binary Protocol"
description: "A complete walkthrough of MtProtoKit — the Objective-C implementation of Telegram's MTProto 2.0 protocol, covering transport, encryption, session management, DH key exchange, and the message assembly pipeline."
published: 2025-07-07
tags:
  - Networking
toc: true
lang: en
abbrlink: 10-mtproto
pin: 16
---

Telegram doesn't use HTTP. It doesn't use gRPC or WebSocket-over-JSON. Instead, it speaks **MTProto** — a custom binary protocol designed from scratch for mobile messaging. MTProto handles authentication, encryption, message serialization, session management, and transport — all in one tightly integrated stack.

The iOS implementation lives in `MtProtoKit`, a 128-file Objective-C module with zero Swift. This post traces every layer of the protocol, from raw TCP bytes to decrypted API responses.

## Why a Custom Protocol?

Before we look at code, it's worth understanding why Telegram built MTProto instead of using TLS + HTTP/2 like everyone else:

1. **Binary efficiency.** MTProto's TL serialization is more compact than JSON or Protobuf for Telegram's specific type system. Every message is a fixed-layout binary blob with 4-byte-aligned fields.

2. **Built-in encryption.** Rather than layering TLS on top of HTTP, MTProto integrates encryption at the protocol level. The key exchange is a custom DH scheme that produces a 256-byte auth key, reused across sessions.

3. **Session semantics.** MTProto has first-class concepts for sessions, message acknowledgments, retransmissions, and server salt rotation. HTTP has none of these — you'd need to build them on top.

4. **Multiplexing.** A single MTProto connection can carry multiple API requests in a single encrypted container, reducing round trips. The container concept (grouping messages) is built into the protocol.

5. **Censorship resistance.** MTProto supports transport obfuscation — the TCP stream can be disguised as TLS traffic to evade deep packet inspection. This is impossible with standard HTTPS libraries.

## Module Architecture

`MtProtoKit` lives at `/submodules/MtProtoKit/` with this structure:

```
MtProtoKit/
├── PublicHeaders/MtProtoKit/    # 50+ .h files (public API)
├── Sources/                      # 78 .m files (implementation)
└── BUILD                         # Bazel build (swift_library wrapping ObjC)
```

The module depends only on `EncryptionProvider` (an abstraction over CommonCrypto) and Apple frameworks (Foundation, Security, SystemConfiguration, CFNetwork, libz). No third-party networking libraries.

The key classes form a layered stack:

```
MTContext              ← Global state: datacenters, auth keys, transport schemes
    │
MTProto               ← Protocol state machine: one per datacenter connection
    │
MTTransport           ← Abstract transport (MTTcpTransport is the concrete impl)
    │
MTTcpConnection       ← Raw TCP socket (via GCDAsyncSocket)
    │
MTMessageService      ← Pluggable services: requests, auth, time sync, resend
```

## MTContext: The Global Coordinator

`MTContext` is the shared singleton that manages state across all datacenter connections. It owns:

```objc
// MTContext.h — key properties
@property (nonatomic, strong, readonly) id<MTSerialization> serialization;
@property (nonatomic, strong, readonly) id<EncryptionProvider> encryptionProvider;
@property (nonatomic, strong, readonly) MTApiEnvironment *apiEnvironment;
@property (nonatomic, strong, readonly) id<MTKeychain> keychain;
```

**Datacenter management.** MTContext maintains the address set (IP, port, flags) for each datacenter:

```objc
- (void)setSeedAddressSetForDatacenterWithId:(NSInteger)datacenterId
                                   seedAddressSet:(MTDatacenterAddressSet *)seedAddressSet;
- (void)updateAddressSetForDatacenterWithId:(NSInteger)datacenterId
                                 addressSet:(MTDatacenterAddressSet *)addressSet
                                   forceUpdateSchemes:(bool)forceUpdateSchemes;
```

Each `MTDatacenterAddress` carries rich metadata:

```objc
@interface MTDatacenterAddress : NSObject
@property (nonatomic, strong, readonly) NSString *ip;
@property (nonatomic, readonly) uint16_t port;
@property (nonatomic, readonly) bool preferForMedia;   // Use for large downloads
@property (nonatomic, readonly) bool restrictToTcp;     // Don't try HTTP
@property (nonatomic, readonly) bool cdn;               // CDN node, not origin
@property (nonatomic, strong, readonly) NSData *secret; // Obfuscation key for proxy
@end
```

**Auth key management.** MTContext stores per-datacenter auth keys with three selectors:

```objc
typedef enum {
    MTDatacenterAuthInfoSelectorPersistent = 0,      // Long-term auth key
    MTDatacenterAuthInfoSelectorEphemeralMain = 1,    // Temp key for main connection
    MTDatacenterAuthInfoSelectorEphemeralMedia = 2    // Temp key for media downloads
} MTDatacenterAuthInfoSelector;
```

The persistent key (256 bytes) is generated during the DH handshake and stored in the keychain. Ephemeral keys are bound to the persistent key and rotated periodically — they provide forward secrecy so that compromising the persistent key doesn't reveal past traffic encrypted with expired ephemeral keys.

**Transport scheme selection.** When MTProto needs to connect, it asks MTContext for the best transport scheme:

```objc
- (MTTransportScheme *)chooseTransportSchemeForConnectionToDatacenterId:(NSInteger)datacenterId
                                                                 media:(bool)media
                                                         isProxy:(bool)isProxy;
```

MTContext tracks scheme reliability — it records failures and invalidates schemes that consistently fail, falling back to alternatives (different IPs, ports, or transport types).

**Global time.** MTContext maintains the difference between local clock and server time:

```objc
@property (nonatomic, readonly) NSTimeInterval globalTimeDifference;
```

This offset is critical because MTProto message IDs are timestamp-based. If the client's clock drifts too far from the server's, messages are rejected. MTContext corrects this automatically using time sync responses.

## MTProto: The Protocol State Machine

Each datacenter connection is managed by an `MTProto` instance. It's a state machine with these states:

```objc
// MTProto.m, line 59
typedef enum {
    MTProtoStateAwaitingDatacenterScheme        = 1,
    MTProtoStateAwaitingDatacenterAuthorization = 2,
    MTProtoStateAwaitingDatacenterAuthToken     = 8,
    MTProtoStateAwaitingTimeFixAndSalts         = 16,
    MTProtoStateAwaitingLostMessages            = 32,
    MTProtoStateStopped                         = 64,
    MTProtoStatePaused                          = 128
} MTProtoState;
```

These flags are combined as a bitmask. A typical startup progression:

1. **Paused** → `resume` called
2. **AwaitingDatacenterScheme** → MTContext provides a transport scheme
3. **AwaitingDatacenterAuthorization** → DH handshake generates auth key
4. **AwaitingTimeFixAndSalts** → Time sync + future salts fetched
5. **Ready** (all flags cleared) → API requests can flow

The state machine runs on a dedicated serial queue:

```objc
// MTProto.m, line 139
+ (MTQueue *)managerQueue {
    static MTQueue *queue = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        queue = [[MTQueue alloc] initWithName:"org.mtproto.managerQueue"];
    });
    return queue;
}
```

All MTProto operations dispatch onto this queue. The queue is shared across all MTProto instances — serializing access to connection state prevents race conditions without per-instance locks.

**Initialization** creates a fresh session and starts paused:

```objc
// MTProto.m, line 150
- (instancetype)initWithContext:(MTContext *)context
                   datacenterId:(NSInteger)datacenterId
           usageCalculationInfo:(MTNetworkUsageCalculationInfo *)usageCalculationInfo
             requiredAuthToken:(id)requiredAuthToken
    authTokenMasterDatacenterId:(NSInteger)authTokenMasterDatacenterId
{
    _context = context;
    _datacenterId = datacenterId;
    _messageServices = [[NSMutableArray alloc] init];
    _sessionInfo = [[MTSessionInfo alloc] initWithRandomSessionIdAndContext:_context];
    _shouldStayConnected = true;
    _mtState |= MTProtoStatePaused;
}
```

Callers add message services, configure flags, then call `[proto resume]` to activate.

## Message Services: The Plugin Architecture

MTProto doesn't hardcode what messages to send. Instead, it delegates to **message services** — pluggable objects that implement the `MTMessageService` protocol:

```objc
@protocol MTMessageService <NSObject>
@optional
- (MTMessageTransaction *)mtProtoMessageTransaction:(MTProto *)mtProto;
- (void)mtProto:(MTProto *)mtProto receivedMessage:(MTIncomingMessage *)message;
- (void)mtProtoConnectionStateChanged:(MTProto *)mtProto isConnected:(bool)isConnected;
@end
```

The key services:

| Service | Purpose |
|---------|---------|
| `MTRequestMessageService` | Queues API requests, routes responses |
| `MTDatacenterAuthMessageService` | DH key exchange handshake |
| `MTTimeSyncMessageService` | Clock synchronization with server |
| `MTResendMessageService` | Handles `bad_msg_notification` errors |
| `MTBindKeyMessageService` | Binds ephemeral keys to persistent key |

When the transport is ready to send data, MTProto asks each service for its pending transactions via `mtProtoMessageTransaction:`. The service returns an `MTMessageTransaction` containing `MTOutgoingMessage` objects. MTProto then assembles, encrypts, and sends them.

## The DH Key Exchange: Four Stages

Before any API requests can flow, the client needs an auth key. The handshake is implemented in `MTDatacenterAuthMessageService.m` and follows this state machine:

```objc
// MTDatacenterAuthMessageService.m, line 94
typedef enum {
    MTDatacenterAuthStageWaitingForPublicKeys = 0,
    MTDatacenterAuthStagePQ                   = 1,
    MTDatacenterAuthStageReqDH                = 2,
    MTDatacenterAuthStageKeyVerification      = 3,
    MTDatacenterAuthStageDone                 = 4
} MTDatacenterAuthStage;
```

**Stage 1: req_pq_multi.** The client sends a 16-byte random nonce. The server responds with `ResPq` (constructor `0x05162463`) containing:
- The client's nonce echoed back
- A 16-byte server nonce
- A semiprime `pq` (product of two primes)
- RSA key fingerprints

**Stage 2: req_DH_params.** The client:
1. Factors `pq` into `p` and `q` using trial division (`MTFactorize`)
2. Selects an RSA public key matching one of the server's fingerprints
3. Encrypts `(p, q, nonce, server_nonce, new_nonce)` with RSA-OAEP
4. Sends the encrypted payload

The public keys are hardcoded — separate sets for production and testing:

```objc
// MTDatacenterAuthMessageService.m, line 46
static NSArray<MTDatacenterAuthPublicKey *> *defaultPublicKeys(bool isProduction) {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        testingPublicKeys = @[
            [[MTDatacenterAuthPublicKey alloc] initWithPublicKey:@"-----BEGIN RSA PUBLIC KEY-----\n"
             "MIIBCgKCAQEAyMEdY1aR+sCR3ZSJrtztKTKqigvO/vBf..."]
        ];
        productionPublicKeys = @[
            [[MTDatacenterAuthPublicKey alloc] initWithPublicKey:@"-----BEGIN RSA PUBLIC KEY-----\n"
             "MIIBCgKCAQEA6LszBcC1LGzyr992NzE0ieY+BSaOW622..."]
        ];
    });
}
```

Key selection matches server fingerprints against local keys:

```objc
static MTDatacenterAuthPublicKey *selectPublicKey(
    id<EncryptionProvider> encryptionProvider,
    NSArray<NSNumber *> *fingerprints,
    NSArray<MTDatacenterAuthPublicKey *> *publicKeys)
{
    for (NSNumber *nFingerprint in fingerprints) {
        for (MTDatacenterAuthPublicKey *key in publicKeys) {
            uint64_t keyFingerprint = [key fingerprintWithEncryptionProvider:encryptionProvider];
            if ([nFingerprint unsignedLongLongValue] == keyFingerprint) {
                return key;
            }
        }
    }
    return nil;
}
```

**Stage 3: Server DH params.** The server responds with `ServerDhInnerData` (`0xb5890dba`) containing the DH parameters:
- `g` — the generator (validated to be in `{2, 3, 4, 5, 6, 7}` via `MTCheckIsSafeG`)
- `dhPrime` (`p`) — a large safe prime (verified via `MTCheckIsSafePrime`)
- `gA` — the server's public value `g^a mod p`
- `serverTime` — used for time synchronization

**Stage 4: set_client_DH_params.** The client:
1. Generates a random `b`
2. Computes `gB = g^b mod p` (validated via `MTCheckIsSafeGAOrB`)
3. Computes `authKey = gA^b mod p` (the shared 256-byte secret)
4. Sends `gB` to the server, encrypted with a temporary AES key derived from `new_nonce` and `server_nonce`

The auth key ID is the lower 64 bits of `SHA-1(authKey)`. This ID is sent with every encrypted message as a fast lookup for the server to find the right key without decrypting.

All bignum operations use the `MTExp`, `MTModSub`, `MTModMul` functions from `MTEncryption.m`, which wrap CommonCrypto's bignum support.

## Message Encryption: v1 and v2 KDF

Once the auth key exists, every message is encrypted with AES-256 in IGE (Infinite Garble Extension) mode. The AES key and IV are derived per-message using a **Key Derivation Function** (KDF).

**MTProto 1.0 KDF** (legacy, `messageEncryptionKeyForAuthKey:messageKey:toClient:`):

```objc
// MTMessageEncryptionKey.m, line 7
int x = toClient ? 8 : 0;

// Four SHA-1 computations mixing messageKey with different authKey slices
sha1_a = SHA1(messageKey + authKey[x..x+32])
sha1_b = SHA1(authKey[32+x..48+x] + messageKey + authKey[48+x..64+x])
sha1_c = SHA1(authKey[64+x..96+x] + messageKey)
sha1_d = SHA1(messageKey + authKey[96+x..128+x])

// AES key: 8 bytes from sha1_a + 12 from sha1_b + 12 from sha1_c = 32 bytes
aesKey = sha1_a[0:8] + sha1_b[8:20] + sha1_c[4:16]

// AES IV: 12 bytes from sha1_a + 8 from sha1_b + 4 from sha1_c + 8 from sha1_d = 32 bytes
aesIv = sha1_a[8:20] + sha1_b[0:8] + sha1_c[16:20] + sha1_d[0:8]
```

**MTProto 2.0 KDF** (current, `messageEncryptionKeyV2ForAuthKey:messageKey:toClient:`):

```objc
// MTMessageEncryptionKey.m, line 70
int xValue = toClient ? 8 : 0;

sha256_a = SHA256(messageKey + authKey[xValue..xValue+36])
sha256_b = SHA256(authKey[40+xValue..76+xValue] + messageKey)

aesKey = sha256_a[0:8] + sha256_b[8:24] + sha256_a[24:32]  // 32 bytes
aesIv  = sha256_b[0:8] + sha256_a[8:24] + sha256_b[24:32]  // 32 bytes
```

The `x` offset (0 for client→server, 8 for server→client) ensures the same auth key produces different encryption keys in each direction, preventing reflection attacks.

The **message key** itself is the middle 128 bits of `SHA-256(authKey[88+x..120+x] + plaintext)`. This means the message key is a function of both the auth key and the actual plaintext — changing even one byte of the message changes the key used to encrypt it.

## The Message Assembly Pipeline

The heart of MtProtoKit is `transportReadyForTransaction:` in `MTProto.m` (line 948). This method runs when the transport is ready to send data. Here's the complete flow:

### Step 1: Get the Auth Key

```objc
MTDatacenterAuthInfoSelector authInfoSelector;
MTDatacenterAuthKey *authKey = [self getAuthKeyForCurrentScheme:scheme
                                             createIfNeeded:true
                                           authInfoSelector:&authInfoSelector];
```

This selects persistent or ephemeral key based on connection type (main vs media) and whether temp auth keys are enabled. If no key exists, it triggers the DH handshake and returns — no messages can be sent yet.

### Step 2: Collect Messages from Services

```objc
for (id<MTMessageService> messageService in _messageServices) {
    if ([messageService respondsToSelector:@selector(mtProtoMessageTransaction:)]) {
        MTMessageTransaction *transaction = [messageService mtProtoMessageTransaction:self];
        if (transaction != nil) {
            [messageTransactions addObject:transaction];
        }
    }
}
```

Each service contributes its pending messages. `MTRequestMessageService` returns queued API calls. The auth service returns DH handshake messages. The time sync service returns ping messages.

### Step 3: Build Acknowledgments

```objc
NSArray *scheduledAcks = [_sessionInfo scheduledMessageConfirmations];
if (scheduledAcks.count != 0) {
    MTBuffer *msgsAckBuffer = [[MTBuffer alloc] init];
    [msgsAckBuffer appendInt32:(int32_t)0x62d6b459]; // msgs_ack constructor
    [msgsAckBuffer appendInt32:(int32_t)0x1cb5c415]; // vector
    [msgsAckBuffer appendInt32:(int32_t)scheduledAcks.count];
    for (NSNumber *messageId in scheduledAcks) {
        [msgsAckBuffer appendInt64:[messageId longLongValue]];
    }
}
```

Any incoming messages that haven't been acknowledged yet are batched into a single `msgs_ack` message.

### Step 4: Assign Message IDs and Sequence Numbers

```objc
bool monotonityViolated = false;
int64_t messageId = [_sessionInfo generateClientMessageId:&monotonityViolated];
int32_t messageSeqNo = [_sessionInfo takeSeqNo:message.requiresConfirmation];
```

Message IDs encode the current time — upper 32 bits are Unix timestamp, lower 32 bits are fractional seconds. They must be strictly monotonic within a session. If `generateClientMessageId` detects a monotonicity violation (clock went backwards), it sets the flag and MTProto resets the session.

Sequence numbers follow MTProto's even/odd convention:
- **Even** seqNo for "content-related" messages (API requests that need acknowledgment)
- **Odd** seqNo for service messages (acks, pings)

### Step 5: Container Assembly

Multiple messages are grouped into a single encrypted container:

```objc
static const NSUInteger MTMaxContainerSize = 3 * 1024; // 3 KB

// If multiple messages, wrap in a msg_container (0x73f1f8dc)
if (preparedMessages.count > 1) {
    encryptedData = [self _dataForEncryptedContainerWithMessages:preparedMessages
                                                       authKey:authKey
                                                   sessionInfo:_sessionInfo
                                                    quickAckId:&quickAckId
                                                       address:scheme.address
                                                extendedPadding:extendedPadding];
}
```

The 3KB container limit prevents oversized packets that might be fragmented or rejected. If a single message exceeds this, it's sent alone.

### Step 6: Encryption

For each container (or single message), encryption proceeds:

1. **Construct plaintext:** `salt (8 bytes) + sessionId (8 bytes) + messageId (8 bytes) + seqNo (4 bytes) + messageDataLength (4 bytes) + messageData + padding`
2. **Compute message key:** Middle 128 bits of `SHA-256(authKey[88..120] + plaintext)`
3. **Derive AES key/IV:** Using the v2 KDF described above
4. **Encrypt:** AES-256-IGE mode
5. **Prepend header:** `authKeyId (8 bytes) + messageKey (16 bytes) + encryptedPayload`

The padding is 12-1024 random bytes, padded to a 16-byte boundary (AES block size). Extended padding (up to 1024 bytes) is used when a proxy is active to further obscure message lengths.

### Step 7: Transport Transmission

The encrypted data is wrapped in an `MTTransportTransaction` and handed to the transport:

```objc
transactionReady(@[[[MTTransportTransaction alloc]
    initWithPayload:encryptedData
    expectsDataInResponse:true
    needsQuickAck:quickAckRequested]]);
```

## Session Management

### Session IDs

`MTSessionInfo` generates a random 64-bit session ID at creation:

```objc
_sessionInfo = [[MTSessionInfo alloc] initWithRandomSessionIdAndContext:_context];
```

The session ID is included in every encrypted message. The server tracks active sessions and delivers updates only to currently connected sessions.

### Salts

Server salts are 64-bit values with validity windows (defined by message ID ranges). The server provides future salts proactively via `MTFutureSaltsMessage`. Salt management in `MTDatacenterAuthInfo`:

```objc
@interface MTDatacenterAuthInfo : NSObject
@property (nonatomic, strong, readonly) NSData *authKey;       // 256 bytes
@property (nonatomic, readonly) int64_t authKeyId;             // SHA1(authKey)[0:8]
@property (nonatomic, readonly) int32_t validUntilTimestamp;   // For ephemeral keys
@property (nonatomic, strong, readonly) NSArray *saltSet;      // [MTDatacenterSaltInfo]
@end
```

Salt selection picks the salt with the most remaining valid message IDs:

```objc
// MTDatacenterAuthInfo.m — salt selection
for (MTDatacenterSaltInfo *saltInfo in _saltSet) {
    NSInteger remaining = [saltInfo validMessageCountAfterId:messageId];
    if (remaining > bestRemaining) {
        bestRemaining = remaining;
        bestSalt = saltInfo.salt;
    }
}
```

If no valid salt is available, a zero salt is used — the server will respond with a new salt set.

### Message Acknowledgments

MTProto tracks which messages need acknowledgment:

```objc
static const NSUInteger MTMaxUnacknowledgedMessageCount = 64;
static const NSUInteger MTMaxUnacknowledgedMessageSize = 1 * 1024 * 1024; // 1 MB
```

When these thresholds are reached, MTProto forces an acknowledgment batch even if there are no pending API responses.

## Transport Layer: TCP with Obfuscation

The concrete transport is `MTTcpTransport`, which wraps `MTTcpConnection` (built on GCDAsyncSocket for async socket I/O).

### Connection Behavior

`MTTcpConnectionBehaviour` manages the reconnection state machine with these operations:

```objc
- (void)requestConnection;
- (void)connectionOpened;
- (void)connectionValidDataReceived;
- (void)connectionClosed;
```

It implements exponential backoff on failures — each failed connection attempt increases the delay before the next retry.

### TLS Obfuscation

When a proxy `secret` is configured, `MTTcpConnection` generates a fake TLS ClientHello packet to disguise the MTProto connection:

```objc
// MTTcpConnection.m — TLS Hello generation
// A compact DSL generates the TLS ClientHello:
NSString *code = @"S \"\\x16\\x03\\x01\\x02\\x00\\x01\\x00\\x01\\xfc\\x03\\x03\"\n"
                  "R 32\n"       // 32 random bytes
                  "S \"\\x20\"\n"
                  "R 32\n"       // Session ID (random)
                  // ... cipher suites, extensions, etc.
                  "D\n"          // Domain name (SNI)
                  "K\n"          // Curve25519 public key
```

The DSL commands:
- `S "..."` — append hex string
- `R N` — append N random bytes
- `D` — append domain name
- `G N` — append GREASE value at index N
- `K` — append Curve25519 public key
- `[` / `]` — push/pop length position (for TLS length fields)

This makes the connection indistinguishable from a normal TLS handshake to network-level observers.

## The Bridge to Swift: Network.swift

TelegramCore wraps MtProtoKit in `Network.swift`, bridging Objective-C delegates to SwiftSignalKit signals. The key method:

```swift
// Network.swift, line 1158
public func request<T>(
    _ data: (FunctionDescription, Buffer, DeserializeFunctionResponse<T>),
    tag: NetworkRequestDependencyTag? = nil,
    automaticFloodWait: Bool = true
) -> Signal<T, MTRpcError> {
    return Signal { subscriber in
        let request = MTRequest()

        request.setPayload(
            data.1.makeData() as Data,           // Serialized TL binary
            metadata: WrappedRequestMetadata(...),
            shortMetadata: WrappedRequestShortMetadata(...),
            responseParser: { response in
                if let result = data.2.parse(Buffer(data: response)) {
                    return BoxedMessage(result)
                }
                return nil
            }
        )

        request.completed = { (boxedResponse, timestamp, error) in
            if let error = error {
                subscriber.putError(error)
            } else if let result = (boxedResponse as? BoxedMessage)?.body as? T {
                subscriber.putNext(result)
                subscriber.putCompletion()
            }
        }

        self.requestService.add(request)
        return ActionDisposable { request.cancel() }
    }
}
```

This accepts the triple returned by generated API functions (covered in the next post) and returns a `Signal<T, MTRpcError>`. The pattern:

1. `data.1` (Buffer) becomes the request payload
2. `data.2` (DeserializeFunctionResponse) becomes the response parser
3. `data.0` (FunctionDescription) provides logging metadata
4. MTRequestMessageService queues the request
5. When MTProto sends and receives a response, the completion block fires
6. The parsed result flows through the Signal to the caller

Flood wait handling is built in — if the server returns a flood wait error and `automaticFloodWait` is true, the request service automatically retries after the wait period.

## Connection Status

`Network.swift` also translates MTProto delegate callbacks into a reactive `ConnectionStatus`:

```swift
public enum ConnectionStatus: Equatable {
    case waitingForNetwork
    case connecting(proxyAddress: String?, proxyHasConnectionIssues: Bool)
    case updating(proxyAddress: String?)
    case online(proxyAddress: String?)
}
```

This drives the "Connecting..." indicator in the UI. The status is derived from a combination of MTProto delegate flags:

```swift
private struct MTProtoConnectionFlags: OptionSet {
    static let NetworkAvailable         = MTProtoConnectionFlags(rawValue: 1)
    static let Connected                = MTProtoConnectionFlags(rawValue: 2)
    static let UpdatingConnectionContext = MTProtoConnectionFlags(rawValue: 4)
    static let PerformingServiceTasks   = MTProtoConnectionFlags(rawValue: 8)
    static let ProxyHasConnectionIssues = MTProtoConnectionFlags(rawValue: 16)
}
```

## Error Recovery

MtProtoKit handles several classes of errors:

**bad_msg_notification.** The server rejects a message (wrong seqNo, expired salt, time drift). MTProto adjusts the relevant parameter and resends via `MTResendMessageService`.

**Session reset.** If the server sends `new_session_created`, MTProto starts a new session. All pending messages are re-queued.

**Transport failure.** When the TCP connection drops, `MTTcpConnectionBehaviour` schedules a reconnection with backoff. `MTNetworkAvailability` monitors system connectivity to avoid reconnecting when there's no network.

**Monotonicity violation.** If `generateClientMessageId` detects the clock went backwards (VM migration, NTP adjustment, sleep/wake), it resets the session to avoid message ID conflicts.

**Scheme failure.** Failed connections are reported to MTContext, which deprioritizes that transport scheme and tries alternatives (different IP, port, or protocol).

## Tracing a Complete Request

Let's follow an `Api.functions.messages.getHistory` call from Swift to the wire:

1. **TelegramCore** calls `network.request(Api.functions.messages.getHistory(...))` → returns `Signal<Api.messages.Messages, MTRpcError>`
2. **Network.swift** creates an `MTRequest` with the serialized binary and response parser
3. **MTRequestMessageService** queues the request, returns it as an `MTMessageTransaction`
4. **MTProto.transportReadyForTransaction** collects all pending transactions from all services
5. **Message ID assignment:** `_sessionInfo.generateClientMessageId` returns a timestamp-based 64-bit ID
6. **Container assembly:** If multiple messages are pending, they're grouped into a `msg_container`
7. **Encryption:** Plaintext (salt + session + messages) → SHA-256 → message key → AES key/IV → AES-256-IGE
8. **Transport:** Encrypted bytes → `MTTransportTransaction` → `MTTcpTransport` → `MTTcpConnection` → GCDAsyncSocket → TCP
9. **Response received:** TCP bytes → `MTTcpConnection` → `MTProto.transportHasIncomingData`
10. **Decryption:** authKeyId lookup → AES-256-IGE decrypt → message key verification → plaintext
11. **Parsing:** `MTInternalMessageParser` reads constructor IDs → dispatches to `MTRpcResultMessage`
12. **Routing:** The RPC result's `requestMessageId` maps back to the `MTRequest`
13. **Deserialization:** The response parser (`DeserializeFunctionResponse`) turns binary into `Api.messages.Messages`
14. **Signal completion:** `request.completed` fires → `subscriber.putNext(result)` → caller receives the parsed Swift type

The entire path — from a type-safe Swift function call to encrypted binary on the wire and back — involves six modules (TelegramCore, TelegramApi, Network, MtProtoKit, EncryptionProvider, GCDAsyncSocket) but presents a clean `Signal<T, Error>` interface to the caller. The complexity is completely hidden behind `network.request()`.

## Key Takeaways

1. **Objective-C for a reason.** MtProtoKit is Objective-C because it predates Swift, but also because low-level crypto and binary manipulation benefit from C interop. Raw pointer arithmetic for AES, SHA, and buffer management is natural in ObjC.

2. **State machine, not callbacks.** MTProto's bitmask state machine cleanly expresses the multi-phase initialization (scheme → auth → time → ready). Each state gate prevents premature message sending.

3. **Service plugins.** The `MTMessageService` protocol makes the protocol extensible — auth, requests, time sync, and resend are all independent plugins sharing the same transport.

4. **Everything is a Signal.** Despite MtProtoKit being ObjC with delegates, `Network.swift` wraps everything in SwiftSignalKit signals. The caller never sees MTProto internals.

5. **Defense in depth.** Safety checks at every level — DH parameter validation, monotonicity checks, salt expiry, transport scheme failover, connection probing — make the protocol resilient to network issues and attacks.
