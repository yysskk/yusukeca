---
title: "Building Nostr Apps in Swift: A Hands-On Guide to swift-nostr"
description: "From keypairs to encrypted DMs, Lightning zaps, and remote signing — a practical, code-first walkthrough of building a Nostr client in Swift with swift-nostr."
date: 2026-07-06
tags: ["swift", "nostr", "tutorial", "ios"]
---

In [the previous post](/blog/introducing-swift-nostr/) I introduced [swift-nostr](https://github.com/yysskk/swift-nostr), my Swift library for the Nostr protocol. This post is the practical follow-up: a walkthrough of the pieces you actually assemble when building a Nostr app — identity, events, relays, feeds, encrypted messages, Lightning zaps, and remote signing — with working code for each.

Everything here targets swift-nostr 0.6.0 on Swift 6.3+ (iOS 17, macOS 14, tvOS 17, watchOS 10, visionOS 1). If you want runnable versions of these snippets, the companion repo [swift-nostr-examples](https://github.com/yysskk/swift-nostr-examples) contains a unit-tested case study for nearly every section below — more on that at the end.

## 1. Setup: one package, four libraries

Add the package to your `Package.swift` (or via Xcode's *File → Add Package Dependencies*):

```swift
dependencies: [
    .package(url: "https://github.com/yysskk/swift-nostr", from: "0.6.0")
]
```

The package vends four library products, and picking the right ones up front saves confusion later:

- **`NostrCore`** — protocol primitives: `Event`, `KeyPair`, `EventSigner`, `Filter`, bech32/NIP-19 encoding, NIP-44 encryption, and a single-relay WebSocket transport.
- **`NostrClient`** — the high-level actor-based client: multi-relay pool, publishing, subscriptions, private DMs, outbox routing.
- **`NostrWalletConnect`** — NIP-47 wallet payments over Nostr.
- **`NostrConnect`** — NIP-46 remote signing.

One thing to know: `NostrClient` is built on `NostrCore` but does **not** re-export it. Since you'll constantly touch core types like `Event` and `Filter`, add both products to your target and import both:

```swift
import NostrClient
import NostrCore
```

The same applies to `NostrWalletConnect` and `NostrConnect` — their APIs surface core types, so bring `NostrCore` along.

## 2. Keys and identity

A Nostr identity is a secp256k1 keypair. The public key is who you are; the private key signs everything you publish. In NIP-19's bech32 encoding they're the familiar `npub1…` and `nsec1…` strings:

```swift
let keyPair = try KeyPair()               // new random identity
print(try keyPair.npub)                   // npub1…
print(try keyPair.nsec)                   // nsec1… — guard this

let imported = try KeyPair(nsec: "nsec1…")   // import an existing key
```

### Mnemonic identities (NIP-06)

If you'd rather give users a recoverable seed phrase than a raw `nsec`, NIP-06 derives the keypair from a BIP-39 mnemonic:

```swift
let (mnemonic, derived) = try KeyPair.generate(wordCount: 12)
print(mnemonic.phrase)     // "abandon ability able …"
print(try derived.npub)
```

The same phrase always derives the same keys, so a user can restore their identity on a new device from words they wrote down.

### Storing keys safely (NIP-49)

Persisting a raw private key is risky. NIP-49 encrypts it under a password — scrypt for key derivation, XChaCha20-Poly1305 for the encryption — producing an `ncryptsec1…` string that's safe to store or export:

```swift
let encrypted = try EncryptedPrivateKey.encrypt(
    privateKey: keyPair.privateKey,
    password: "correct horse battery staple",
    logN: 16)                              // scrypt cost; 16 is the NIP-49 default
let ncryptsec = encrypted.ncryptsec        // store this

// Later — wrong password throws NostrError.decryptionFailed
let restored = try EncryptedPrivateKey(ncryptsec: ncryptsec)
    .decrypt(password: "correct horse battery staple")
let restoredKeyPair = try KeyPair(privateKey: restored)
```

On Apple platforms you'd typically keep the `ncryptsec` (or the raw key) in the Keychain; NIP-49 shines when the key has to leave the device — exports, backups, cross-app handoff.

## 3. Events from first principles

Everything on Nostr — notes, profiles, follows, reactions, receipts — is a signed `Event`. Understanding the signing flow pays off even if the high-level client normally handles it for you:

1. You assemble an `UnsignedEvent`: pubkey, kind, tags, content, timestamp.
2. The event **id** is the SHA-256 hash of its canonical NIP-01 serialization.
3. An `EventSigner` produces a Schnorr signature over that id.
4. Anyone can verify the event against the author's public key.

```swift
import NostrCore

let signer = EventSigner(keyPair: keyPair)
let unsigned = UnsignedEvent(pubkey: signer.publicKey, kind: .textNote, content: "gm")
let event = try signer.sign(unsigned)

try event.verify()      // recomputes the id, checks the signature
```

Verification matters on the read side too: relays are untrusted, so a client should treat an event with a bad signature as noise. The kind (`1` for a text note, `0` for profile metadata, and so on) determines both the meaning and the storage semantics — regular, replaceable, ephemeral, or addressable.

## 4. Connect and publish

`NostrClient` is an actor that manages a pool of relay connections, so one instance serves your whole app. Nostr apps talk to several relays at once — for redundancy and reach — and the pool handles that for you:

```swift
import NostrClient
import NostrCore

let client = NostrClient()
try await client.connect(to: ["wss://relay.damus.io", "wss://nos.lol"])
try await client.setNsec("nsec1…")

let note = try await client.publishTextNote(content: "Hello, Nostr!")
print(note.event.id)
print("Accepted by \(note.result.acceptedRelays.count) relay(s)")
```

`publishTextNote` returns both the signed event and a `PublishResult` describing the per-relay outcome — which relays accepted, which rejected and why. A `PublishStrategy` controls how long a publish waits: `.firstAck` resolves on the first acknowledgment, `.quorum(n)` waits for `n` relays, and `.allSettled` waits for every relay to answer. For a snappy UI, `.firstAck` with the full result arriving later is a good default.

Replies and reactions build on the same flow. A reply is a kind-1 note carrying NIP-10 threading tags (the thread root, the parent, and the author being answered), and the client assembles them from the event you're replying to:

```swift
try await client.publishReply(to: parentEvent, content: "Great point!")
try await client.publishReaction(to: parentEvent, content: "🤙")
```

If you ever need manual control over tags — say, building a reply offline — the `Tag` helpers in `NostrCore` (`.event(_:marker:)`, `.pubkey(_:)`) plus `EventSigner.signTextNote(content:tags:)` produce the same events without a client.

## 5. Read: filters, subscriptions, and fetches

Reads in Nostr are driven by *filters*: you describe the events you want (by kind, author, tag, time range, limit) and relays stream back matches, live as they arrive. swift-nostr models subscriptions as `AsyncSequence`s, which fits SwiftUI and structured concurrency beautifully:

```swift
// A user's live timeline
let timeline = try await client.subscribeToUserTimeline(pubkey: somePubkey)
for await event in timeline.events {
    print(event.content)
}
```

When the `for await` loop ends — or the surrounding `Task` is cancelled, say because a view disappeared — the subscription closes automatically. No unsubscribe bookkeeping.

For anything custom, build a `Filter` yourself:

```swift
let filter = Filter(kinds: [1], authors: [pubkey1, pubkey2], limit: 100)
for await event in try await client.events(filters: [filter]) {
    print(event.id)
}
```

And for one-shot questions ("what's this user's profile?") there are fetch helpers that query, collect, and return:

```swift
let metadata = try await client.fetchMetadata(pubkey: somePubkey)
print(metadata?.name ?? "Unknown")
```

If you need relay-level detail — per-relay end-of-stored-events markers, notices, auth challenges — `client.subscribe(filters:)` exposes the raw stream; the [Advanced Usage guide](https://yysskk.github.io/swift-nostr/documentation/nostrclient/advancedusage) covers it.

## 6. Private direct messages (NIP-17)

Nostr's modern DM standard, NIP-17, is genuinely private: messages are encrypted with NIP-44, then *gift-wrapped* (NIP-59) so that even the sender's identity is hidden from relays, and routed only to the small set of relays each user designates for DMs (their kind-10050 relay list).

That's a lot of moving parts, and swift-nostr hides all of them behind three calls. First, advertise where you receive DMs and connect your inbox:

```swift
// NIP-17 suggests keeping this list small — one to three relays.
try await client.publishDirectMessageRelayList(relays: ["wss://inbox.example.com"])
try await client.connectDirectMessageInboxRelays()
```

Receiving is a stream of already-decrypted, parsed messages:

```swift
for await message in try await client.directMessages() {
    print("\(message.senderPubkey): \(message.content)")
}
```

Sending encrypts, gift-wraps, and routes to each party's DM relays — the recipient's *and* yours, so the conversation syncs across your own devices:

```swift
try await client.sendDirectMessage("Hello privately!", to: recipientPubkey)
```

Reactions to DMs, encrypted file messages (kind 15), and disappearing messages via NIP-40 expiration timestamps all build on this same flow.

## 7. Zaps and wallet payments (NIP-57 + NIP-47)

*Zaps* are Lightning payments recorded on Nostr — the tipping mechanism that makes the network feel alive. A full zap involves two protocols working together:

1. **NIP-57** describes the intent: a signed kind-9734 *zap request* goes to the recipient's LNURL endpoint, which returns a Lightning invoice; once paid, the recipient's wallet publishes a kind-9735 *zap receipt*.
2. **NIP-47 (Nostr Wallet Connect)** actually pays the invoice, by sending encrypted commands to a remote wallet over Nostr itself.

Building and signing the zap request is an `EventSigner` one-liner:

```swift
let zapRequest = try EventSigner(keyPair: keyPair).signZapRequest(
    recipientPubkey: authorPubkey,
    amountMillisats: 21_000,
    relays: ["wss://relay.damus.io"],      // where the receipt should be published
    zappedEventId: note.id,
    comment: "⚡️")
```

Paying happens through `NostrWalletConnect`. The user pastes (or scans) their wallet's `nostr+walletconnect://` connection string once, and from then on your app can pay through it:

```swift
import NostrWalletConnect
import NostrCore

let connection = WalletConnection(uri: try WalletConnectURI(string: "nostr+walletconnect://…"))

// Pay any bolt11 invoice.
let payment = try await connection.payInvoice("lnbc…")
print(payment.preimage)

// Or complete a zap end to end: resolve the recipient's LNURL-pay endpoint
// (the library includes Lightning-address and LNURL helpers), fetch the
// invoice for the zap request, and pay it — in one call.
let zap = try await connection.payZap(
    lnurlPay: lnurlPay,                    // resolved LNURLPayResponse
    amountMillisats: 21_000,
    zapRequest: zapRequest)
print(zap.preimage)
```

The library implements the full NIP-47 command set — `get_balance`, `get_info`, `make_invoice`, `lookup_invoice`, `list_transactions`, keysend — plus a `notifications()` stream for payment events, and helpers to decode bolt11 invoices and verify kind-9735 receipts before you render "⚡ 21 sats" in the UI.

## 8. Remote signing (NIP-46)

The safest place for a user's key is *not in your app at all*. NIP-46 delegates signing to a remote signer (a "bunker") that holds the key; your app sends it signing requests over Nostr and gets signed events back. The `NostrConnect` library implements both connection flows.

**Signer-initiated** — the user pastes a `bunker://` token from their signer app:

```swift
import NostrConnect
import NostrCore

let signer = try RemoteSigner(bunker: try BunkerURI(string: "bunker://…"))

let userPubkey = try await signer.userPublicKey()
let signed = try await signer.sign(
    UnsignedEvent(pubkey: userPubkey, kind: .textNote, content: "Signed by my bunker"))
print(signed.id, try signed.verify())   // signatures are verified locally
```

**Client-initiated** — your app generates a `nostrconnect://` invitation, shows it as a QR code, and waits for the signer to accept:

```swift
let invitation = try NostrConnectURI.invitation(clientKeyPair: clientKeyPair, relays: [relayURL])
displayQRCode(invitation.stringValue)

let signer = try RemoteSigner(invitation: invitation, clientKeyPair: clientKeyPair)
let remotePubkey = try await signer.awaitConnection()   // resolves once accepted
```

The part I find most satisfying: a `RemoteSigner` plugs directly into `NostrClient`. `setSigner(_:)` accepts any `NostrSigning` — local `EventSigner` or remote — so the rest of your publishing code doesn't care where signatures come from:

```swift
try await client.setSigner(signer)
guard let userPubkey = await client.publicKey else { return }
let signed = try await client.sign(
    UnsignedEvent(pubkey: userPubkey, kind: .textNote, content: "Hello from a bunker"))
try await client.publish(signed)
```

One caveat: the convenience helpers that must sign locally — the `publish*` shorthands and the direct-message APIs — throw `NostrError.localSignerRequired` when a remote signer is set, since NIP-17's gift-wrapping needs the key on-device. Design your signing path around `sign(_:)` + `publish(_:)` if bunker support matters to you.

## 9. Going further

A few features worth knowing about as your app grows past the basics:

- **The outbox model (NIP-65).** Instead of hoping everyone shares your relays, each user publishes a read/write relay list, and clients route accordingly. `subscribeOutbox` and `publishGossip` implement the routing for you — this is the single biggest thing you can do for censorship resistance and reach.
- **Relay authentication (NIP-42).** Some relays (paid relays, DM inboxes) require auth. Once a signer is set, the client answers AUTH challenges automatically and retries auth-required publishes.
- **Relay metadata (NIP-11).** `RelayInformation.fetch(fromRelayURLString:)` tells you a relay's supported NIPs, limits, and policies before you rely on it.
- **NIP-19 entities.** `NIP19Entity`, `NProfile`, `NEvent`, and `NAddr` encode and decode the `npub…`/`nevent…`/`naddr…` strings you'll encounter everywhere users share links.
- **Long-form content (NIP-23).** Kind-30023 articles — think blog posts on Nostr — with drafts and `naddr` addressing, new in 0.6.0.

## 10. Steal from the examples

Almost everything above exists as a runnable, tested case study in [swift-nostr-examples](https://github.com/yysskk/swift-nostr-examples). The repo is organized the way I wish more example projects were:

- **`Sources/NostrExamples`** — ~40 case studies grouped by topic (keys, events, relays, publishing, reading, social graph, messaging, zaps, wallet, remote signing), each a small documented type you can read top to bottom.
- **`Tests/NostrExamplesTests`** — a unit-test suite per case study.
- **`Sources/NostrExamplesUI` + `App/`** — a SwiftUI showcase app (iOS and macOS) that lets you browse every example and run several live against real relays.

```sh
git clone https://github.com/yysskk/swift-nostr-examples
cd swift-nostr-examples
swift test                                  # everything is offline-testable

brew install xcodegen                       # optional: the showcase app
cd App && xcodegen generate && open NostrExamplesShowcase.xcodeproj
```

The design principle behind the examples is worth stealing for your own app: keep protocol logic *pure* — build and inspect events, filters, and tags offline, where they're trivial to unit-test — and isolate the networked calls at the edges. Nostr's signed-JSON-everywhere design makes this unusually easy, and it's how the library itself is tested.

## Wrapping up

We covered the full arc of a Nostr client: keypairs and mnemonics, signing and verifying events, multi-relay publishing with acknowledgment strategies, async-sequence subscriptions, genuinely private DMs, Lightning zaps through a remote wallet, and keeping keys out of your app entirely with remote signing.

If you build something with swift-nostr, I'd genuinely love to see it. Issues, PRs, and questions are all welcome on [GitHub](https://github.com/yysskk/swift-nostr) — and the [API documentation](https://yysskk.github.io/swift-nostr/documentation/) goes deeper on everything touched here.
