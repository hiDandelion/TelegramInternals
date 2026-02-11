---
title: "Encryption: MTProto Transport Security and Secret Chats"
description: "A deep dive into Telegram iOS's two encryption layers. Covers the MTProto 2.0 transport encryption — auth key negotiation via Diffie-Hellman, the two-step message key derivation using SHA-256, AES-256-IGE encryption, and the encrypt-then-MAC verification scheme. Then explores end-to-end Secret Chats — the SecretChatState state machine, DH key exchange with safe prime validation, the V1/V2 encryption protocols, key fingerprint verification, the Perfect Forward Secrecy rekeying protocol, and the SecretChatKeychain that manages multiple keys during transitions."
published: 2025-08-01
tags:
  - Feature Deep Dives
toc: true
lang: en
abbrlink: 23-encryption
pin: 3
---

Telegram uses two distinct encryption layers, each serving a different purpose:

1. **MTProto 2.0** — transport encryption applied to all traffic between the client and Telegram's servers. Every API call, every message, every file download is encrypted in transit. The server can decrypt this traffic (it needs to route messages, store history, etc.).

2. **Secret Chats** — end-to-end encryption between two clients. The server relays encrypted blobs it cannot decrypt. Only the two participants hold the key.

Both layers are implemented in the iOS client with careful attention to cryptographic safety. This post examines the code behind each.

## Part 1: MTProto Transport Encryption

### Auth Key Negotiation

Before any encrypted communication can happen, the client must establish an **auth key** with the server. This happens in `MTDatacenterAuthMessageService.m` through a multi-step Diffie-Hellman key exchange.

The process:

1. **Client sends `req_pq_multi`** — requests the server's public parameters
2. **Server responds with `pq`** — a large semiprime the client must factor
3. **Client factors `pq`** into `p` and `q`, then sends `req_DH_params` encrypted with the server's RSA public key
4. **Server responds with `server_DH_params_ok`** — containing `g`, `dh_prime` (the safe prime), and `g_a` (the server's DH public value)
5. **Client validates the prime**, generates a random `b`, computes `g_b = g^b mod dh_prime`, and sends it encrypted
6. **Both sides compute** `auth_key = g_a^b = g_b^a = g^(ab) mod dh_prime`

The resulting 256-byte auth key is stored persistently and used for all subsequent communication with that datacenter.

### Safe Prime Validation

The client doesn't blindly trust the server's DH parameters. `MTCheckIsSafePrime` in `MTEncryption.m` performs rigorous validation:

```c
bool MTCheckIsSafePrime(id<EncryptionProvider> provider, NSData *numberBytes,
                        id<MTKeychain> keychain)
{
    // Check against known good prime first (fast path)
    unsigned char goodPrime0[] = { /* 256 bytes */ };
    if (memcmp(goodPrime0, numberBytes.bytes, 256) == 0) {
        return true;
    }

    // Verify length is exactly 2048 bits
    if (numberBytes.length != 256) return false;

    // High bit must be set
    if (!(((uint8_t *)numberBytes.bytes)[0] & (1 << 7))) return false;

    // Miller-Rabin primality test with 64 iterations
    id<MTBignumContext> context = [provider createBignumContext];
    id<MTBignum> bnNumber = [context create];
    [context assignBinTo:bnNumber value:numberBytes];

    int result = [context isPrime:bnNumber numberOfChecks:64];

    if (result == 1) {
        // Verify (p-1)/2 is also prime (safe prime check)
        id<MTBignum> bnNumberMinusOne = [context create];
        [context subInto:bnNumberMinusOne a:bnNumber b:bnOne];

        id<MTBignum> halfPrime = [context create];
        [context rightShift1Bit:halfPrime a:bnNumberMinusOne];

        result = [context isPrime:halfPrime numberOfChecks:64];
    }

    // Cache result to avoid re-testing
    [keychain setObject:@(result == 1) forKey:primeKey group:@"primes"];
    return result == 1;
}
```

A **safe prime** is a prime `p` where `(p-1)/2` is also prime. This is critical for DH security — with a non-safe prime, an attacker could compute discrete logarithms more efficiently. The 64-iteration Miller-Rabin test gives a false-positive probability of less than 2^(-128).

Additionally, the DH public values are validated to be in a safe range:

```c
bool MTCheckIsSafeGAOrB(id<EncryptionProvider> provider, NSData *gAOrB, NSData *p) {
    // Must satisfy: 2^(2048-64) < g^x < p - 2^(2048-64)
    // This prevents small-subgroup attacks
}
```

### Message Key Derivation

Once the auth key is established, every message is encrypted with a per-message key derived from the auth key and message content. MTProto 2.0 uses a two-step derivation:

```objc
// MTMessageEncryptionKey.m
+ (MTMessageEncryptionKey *)messageEncryptionKeyV2ForAuthKey:(NSData *)authKey
    messageKey:(NSData *)messageKey toClient:(bool)toClient
{
    int x = toClient ? 8 : 0;

    // Step 1: SHA-256(msgKey + authKey[x..x+36])
    NSMutableData *sha256_a_data = [[NSMutableData alloc] init];
    [sha256_a_data appendData:messageKey];
    [sha256_a_data appendBytes:((uint8_t *)authKey.bytes) + x length:36];
    NSData *sha256_a = MTSha256(sha256_a_data);

    // Step 2: SHA-256(authKey[40+x..40+x+36] + msgKey)
    NSMutableData *sha256_b_data = [[NSMutableData alloc] init];
    [sha256_b_data appendBytes:((uint8_t *)authKey.bytes) + 40 + x length:36];
    [sha256_b_data appendData:messageKey];
    NSData *sha256_b = MTSha256(sha256_b_data);

    // Derive AES key (32 bytes) and IV (32 bytes) by interleaving
    // aes_key = sha256_a[0:8] + sha256_b[8:24] + sha256_a[24:32]
    // aes_iv  = sha256_b[0:8] + sha256_a[8:24] + sha256_b[24:32]
}
```

The `x` offset (`0` for client-to-server, `8` for server-to-client) ensures the two directions use different key material from the same auth key. The interleaving of two SHA-256 hashes creates a key derivation function that depends on both the auth key and the message content.

### AES-256-IGE Encryption

The actual encryption uses AES-256 in IGE (Infinite Garble Extension) mode, implemented in `MTAes.m`:

```objc
void MTAesEncryptInplace(NSMutableData *data, NSData *key, NSData *iv) {
    unsigned char aesIv[AES_BLOCK_SIZE * 2];
    memcpy(aesIv, iv.bytes, AES_BLOCK_SIZE * 2);

    AES_KEY aesKey;
    AES_set_encrypt_key(key.bytes, 256, &aesKey);

    AES_ige_encrypt(data.mutableBytes, data.mutableBytes,
                    data.length, &aesKey, aesIv, AES_ENCRYPT);
}
```

IGE mode uses a **two-block IV** (32 bytes instead of 16). Each ciphertext block depends on both the previous ciphertext block and the previous plaintext block. This provides an integrity property — if any ciphertext block is modified, the error propagates to all subsequent blocks (hence "Infinite Garble Extension").

### The Complete Message Structure

An encrypted MTProto message looks like this:

```
[auth_key_id: 8 bytes][msg_key: 16 bytes][encrypted_data: N bytes]

Where encrypted_data contains:
[server_salt: 8 bytes][session_id: 8 bytes][message_id: 8 bytes]
[seq_no: 4 bytes][message_length: 4 bytes][message_body: L bytes]
[padding: 12-1024 bytes]
```

The `msg_key` is derived from a portion of the SHA-256 of the message content plus a slice of the auth key. The recipient recomputes the msg_key from the decrypted content and verifies it matches — this provides **authenticate-then-decrypt** integrity protection.

## Part 2: Secret Chat End-to-End Encryption

### The Secret Chat State Machine

Secret chats go through a well-defined state machine, encoded in `SyncCore_SecretChatState.swift`:

```swift
public enum SecretChatEmbeddedState: PostboxCoding {
    case terminated
    case handshake(SecretChatHandshakeState)
    case basicLayer
    case sequenceBasedLayer(SecretChatSequenceBasedLayerState)
}

public enum SecretChatHandshakeState: PostboxCoding {
    case requested(g: Int32, p: MemoryBuffer, a: MemoryBuffer)
    case accepting(gA: MemoryBuffer, p: MemoryBuffer, a: MemoryBuffer)
    case accepted(gA: MemoryBuffer, b: MemoryBuffer, p: MemoryBuffer)
}
```

The lifecycle:

1. **`.handshake(.requested)`** — initiator generates DH parameters `g`, safe prime `p`, and secret `a`. Sends `g^a mod p` to the peer via the server.

2. **`.handshake(.accepting)`** — responder receives `g^a`, validates it, generates secret `b`. Not yet confirmed.

3. **`.handshake(.accepted)`** — responder computes `shared_key = (g^a)^b mod p = g^(ab) mod p` and sends `g^b` back.

4. **`.sequenceBasedLayer`** — both parties have the shared key. Messages can now be encrypted end-to-end. This state includes:
   - `layerNegotiationState` — which encryption layer version both clients support
   - `rekeyState` — current Perfect Forward Secrecy rekeying session (if any)
   - Sequence counters for message ordering

5. **`.terminated`** — the secret chat is closed. Keys are destroyed.

### Key Exchange: Creating a Secret Chat

The initial key exchange is triggered in `CreateSecretChat.swift`:

```swift
func _internal_createSecretChat(account: Account, peerId: PeerId)
    -> Signal<PeerId, CreateSecretChatError>
{
    return account.postbox.transaction { transaction -> Api.InputUser? in
        return transaction.getPeer(peerId).flatMap(apiInputUser)
    }
    |> mapToSignal { inputUser in
        // Generate DH parameters
        let g: Int32 = ...       // Safe generator (2-7)
        let p: Data = ...        // 2048-bit safe prime
        let a: Data = ...        // 256 random bytes (client secret)

        // Compute g^a mod p
        let gA = MTExp(encryptionProvider, gData, aData, pData)

        // Send to server
        return account.network.request(
            Api.functions.messages.requestEncryption(
                userId: inputUser,
                randomId: Int32.random(),
                gA: Buffer(data: gA)
            )
        )
    }
}
```

The responder validates the received `g^a` before proceeding:

```swift
if !MTCheckIsSafeGAOrB(encryptionProvider, gA.makeData(), p) {
    return state.withUpdatedEmbeddedState(.terminated)
}
```

This check ensures `g^a` is in the range `(2^(2048-64), p - 2^(2048-64))`, preventing small-subgroup attacks where an adversary could recover the shared secret.

### Message Encryption (V1 and V2)

Secret chat messages are encrypted with the shared key using a protocol similar to MTProto but with end-to-end semantics. The implementation lives in `SecretChatEncryption.swift`.

**V2 Encryption** (current):

```swift
func withEncryptedMessage(parameters: SecretChatEncryptionParameters,
                          payloadData: Data) -> Data
{
    case let .v2(role):
        var decryptedData = payloadData

        // Add random padding (minimum 12 bytes, aligned to 16)
        var randomBytes = Data(count: 128)
        SecRandomCopyBytes(nil, 128, &randomBytes)

        var take = 0
        while take < 12 {
            decryptedData.append(randomBytes[take..<(take+1)])
            take += 1
        }
        while decryptedData.count % 16 != 0 {
            decryptedData.append(randomBytes[take..<(take+1)])
            take += 1
        }

        // Additional random padding (0-71 bytes, multiple of 16)
        // This prevents message length analysis
        var remainingCount = Int(arc4random_uniform(UInt32(72 + 1 - take)))
        while remainingCount % 16 != 0 { remainingCount -= 1 }
        for _ in 0..<remainingCount {
            decryptedData.append(randomBytes[take..<(take+1)])
            take += 1
        }

        // Derive message key from shared key + padded plaintext
        let xValue: Int = (role == .creator) ? 0 : 8
        var keyData = Data()
        keyData.append(parameters.key.key[88+xValue..<88+xValue+32])
        keyData.append(decryptedData)
        let keyLarge = MTSha256(keyData)
        let msgKey = keyLarge[8..<24]  // 16 bytes

        // Derive AES key and IV from msgKey + shared key
        let (aesKey, aesIv) = messageKey(key: parameters.key, msgKey: msgKey,
                                          mode: parameters.mode)

        // Encrypt with AES-256-IGE
        let encryptedData = MTAesEncrypt(decryptedData, aesKey, aesIv)!

        // Final format: [key_fingerprint: 8][msg_key: 16][encrypted_data]
        var result = Data()
        result.append(/* key fingerprint — 8 bytes */)
        result.append(msgKey)
        result.append(encryptedData)
        return result
}
```

Key differences between V1 and V2:

| Aspect | V1 | V2 |
|--------|----|----|
| Message key derivation | SHA-1 of plaintext | SHA-256 of (key slice + plaintext) |
| Minimum padding | 0 bytes | 12 bytes |
| Maximum padding | 15 bytes | 1024 bytes |
| Direction separation | None | `xValue = 0` (creator) or `8` (participant) |
| Key slice used | None | 32 bytes from position 88+x |

V2's improvements prevent several attacks:
- The **minimum 12-byte padding** prevents exact-length fingerprinting
- The **variable additional padding** (0-71 bytes) makes traffic analysis harder
- The **directional key derivation** ensures creator→participant and participant→creator messages use different key material
- The **SHA-256 key derivation** from both the shared key and the message content provides a stronger MAC

### Message Decryption and Verification

Decryption reverses the process and includes integrity verification:

```swift
func withDecryptedMessageContents(parameters: SecretChatEncryptionParameters,
                                   data: MemoryBuffer) -> MemoryBuffer?
{
    case let .v2(role):
        // Flip the role for decryption (sender's xValue is opposite)
        let senderRole: SecretChatRole = (role == .creator) ? .participant : .creator
        let (aesKey, aesIv) = messageKey(key: parameters.key, msgKey: msgKey,
                                          mode: .v2(role: senderRole))

        let decryptedData = MTAesDecrypt(encryptedPayload, aesKey, aesIv)!

        // Extract payload length
        var payloadLength: Int32 = 0
        memcpy(&payloadLength, decryptedData, 4)

        let paddingLength = decryptedData.count - (Int(payloadLength) + 4)

        // Verify padding bounds
        if payloadLength <= 0 || payloadLength > decryptedData.count - 4
            || paddingLength < 12 || paddingLength > 1024 {
            return nil  // Invalid message
        }

        // Recompute message key and verify
        let xValue: Int = (role == .creator) ? 8 : 0
        var keyLargeData = Data()
        keyLargeData.append(parameters.key.key[88+xValue..<88+xValue+32])
        keyLargeData.append(decryptedData)
        let localMessageKey = MTSha256(keyLargeData)[8..<24]

        if localMessageKey != receivedMsgKey {
            Logger.shared.log("SecretChatEncryption", "message key doesn't match")
            return nil  // MAC verification failed
        }

        return decryptedData[4..<4+payloadLength]
}
```

The MAC verification is the critical security step. If a single bit of the ciphertext is altered, the recomputed message key won't match, and the message is rejected. This prevents both random corruption and active tampering.

### Key Fingerprints

Users can verify their secret chat's security by comparing key fingerprints displayed as a grid of emoji or an image:

```swift
public struct SecretChatKeyFingerprint: PostboxCoding, Equatable {
    public let sha1: SecretChatKeySha1Fingerprint    // 20 bytes
    public let sha256: SecretChatKeySha256Fingerprint // 32 bytes
}

public struct SecretChatKeySha256Fingerprint: PostboxCoding, Equatable {
    public let k0: Int64
    public let k1: Int64
    public let k2: Int64
    public let k3: Int64

    public init(digest: Data) {
        assert(digest.count == 32)
        // Pack 32 bytes into 4 Int64 values
    }
}
```

Both SHA-1 (for backward compatibility with older clients) and SHA-256 fingerprints are computed and stored. The UI renders these as a visual pattern that users can compare in person or over another channel.

### Perfect Forward Secrecy: Rekeying

Secret chats support periodic key rekeying for Perfect Forward Secrecy (PFS). If an old key is compromised, messages encrypted with newer keys remain secure.

The rekeying protocol in `SecretChatRekeySession.swift`:

```swift
func secretChatAdvanceRekeySessionIfNeeded(
    encryptionProvider: EncryptionProvider,
    transaction: Transaction,
    peerId: PeerId,
    state: SecretChatState,
    action: SecretChatRekeyServiceAction
) -> SecretChatState
```

The protocol:

1. **Initiator sends `pfsRequestKey`** — generates new `a`, computes `g^a`, sends it with a session ID
2. **Responder receives it** — generates new `b`, computes `g^b` and `new_key = (g^a)^b`
3. **Responder sends `pfsAcceptKey`** — includes `g^b` and the new key's fingerprint
4. **Initiator receives it** — computes `new_key = (g^b)^a`, verifies fingerprint matches, sends `pfsCommitKey`
5. **Both sides activate the new key** from this point forward

The key transition is managed through validity ranges:

```swift
public final class SecretChatKey: PostboxCoding, Equatable {
    public let fingerprint: Int64
    public let key: MemoryBuffer        // 256-byte shared secret
    public let validity: SecretChatKeyValidity
    public let useCount: Int32
}

public enum SecretChatKeyValidity: PostboxCoding, Equatable {
    case indefinite                                     // Initial key
    case sequenceBasedIndexRange(fromCanonicalIndex: Int32)  // Rekeyed
}
```

### The Secret Chat Keychain

Multiple keys can be active simultaneously during a rekeying transition. The `SecretChatKeychain` manages this:

```swift
public final class SecretChatKeychain: PostboxCoding, Equatable {
    public let keys: [SecretChatKey]

    public func latestKey(validForSequenceBasedCanonicalIndex index: Int32)
        -> SecretChatKey?
    {
        // Find the most recent key valid for this operation index
        var maxFromCanonicalIndex: (Int, Int32)?
        for i in 0..<self.keys.count {
            switch self.keys[i].validity {
            case .indefinite:
                break
            case let .sequenceBasedIndexRange(fromCanonicalIndex):
                if index >= fromCanonicalIndex {
                    if maxFromCanonicalIndex == nil
                        || maxFromCanonicalIndex!.1 < fromCanonicalIndex {
                        maxFromCanonicalIndex = (i, fromCanonicalIndex)
                    }
                }
            }
        }

        if let (keyIndex, _) = maxFromCanonicalIndex {
            return self.keys[keyIndex]
        }

        // Fall back to indefinite key (initial)
        for key in self.keys {
            if case .indefinite = key.validity {
                return key
            }
        }
        return nil
    }
}
```

Each outgoing message uses the latest key valid for its operation index. Old messages (before the rekey) use the old key; new messages use the new key. Once both sides confirm the rekey, old keys can be deleted.

## Cryptographic Primitives

The low-level crypto in `MTEncryption.m` uses Apple's CommonCrypto and OpenSSL (via BoringSSL):

```c
// SHA-1 (legacy, still used for some fingerprints)
NSData *MTSha1(NSData *data) {
    uint8_t digest[20];
    CC_SHA1(data.bytes, (CC_LONG)data.length, digest);
    return [[NSData alloc] initWithBytes:digest length:20];
}

// SHA-256 (primary hash for MTProto 2.0 and Secret Chat V2)
NSData *MTSha256(NSData *data) {
    uint8_t digest[CC_SHA256_DIGEST_LENGTH];
    CC_SHA256(data.bytes, (CC_LONG)data.length, digest);
    return [[NSData alloc] initWithBytes:digest length:CC_SHA256_DIGEST_LENGTH];
}

// RSA-OAEP (for encrypting DH parameters during auth)
NSData *MTRsaEncryptPKCS1OAEP(id<EncryptionProvider> provider,
                               NSString *key, NSData *data) {
    return [provider rsaEncryptPKCS1OAEPWithPublicKey:key data:data];
}
```

All bignum operations (modular exponentiation for DH, primality testing) go through an `EncryptionProvider` protocol that abstracts the underlying crypto library. This allows swapping implementations without changing the encryption logic.

## Security Properties Summary

| Property | MTProto | Secret Chats |
|----------|---------|-------------|
| Encryption | AES-256-IGE | AES-256-IGE |
| Key derivation | SHA-256 (two-step) | SHA-256 (one-step) |
| Integrity | msg_key verification | msg_key verification |
| Forward secrecy | Server-side key rotation | Client-side DH rekeying |
| Key length | 2048-bit DH → 256-byte key | 2048-bit DH → 256-byte key |
| Who holds keys | Client + server | Only the two clients |
| Prime validation | 64-round Miller-Rabin + safe prime check | Same |
| DH value validation | Range check: 2^(2048-64) < g^x < p-2^(2048-64) | Same |
| Padding | 12-1024 bytes | 12-1024 bytes (V2) |
| Direction separation | x=0 (client→server), x=8 (server→client) | x=0 (creator), x=8 (participant) |

The two layers complement each other: MTProto ensures all traffic is encrypted in transit (even metadata like typing indicators), while Secret Chats provide end-to-end encryption for conversations where the user doesn't want to trust the server. Both use the same cryptographic primitives, the same validation checks, and the same AES-IGE construction — a consistent, auditable implementation across the entire encryption stack.
