---
title: "Introducing swift-nostr: Nostr for the Swift Ecosystem"
description: "swift-nostr 0.6.0 is out — an open-source Swift library for the Nostr protocol, covering 33 NIPs with private DMs, Lightning zaps, wallet payments, and remote signing."
date: 2026-07-05
tags: ["swift", "nostr", "open-source", "ios"]
---

For the past several months I've been building [swift-nostr](https://github.com/yysskk/swift-nostr), an open-source Swift library for the [Nostr](https://nostr.com) protocol. With today's [0.6.0 release](https://github.com/yysskk/swift-nostr/releases/tag/0.6.0) the feature set is broad enough — 33 NIPs, four libraries, and a full documentation site — that it deserves a proper introduction.

## What is Nostr?

Nostr ("Notes and Other Stuff Transmitted by Relays") is a minimal, open protocol for decentralized social networking. Your identity is a keypair, not an account: every post, profile update, and message is a JSON *event* signed with your key, and clients publish those events to simple WebSocket servers called *relays*. There is no central server to ban you or lose your data — switch relays and your identity comes with you. On top of that simple core, the community layers optional extensions called [NIPs](https://github.com/nostr-protocol/nips) (Nostr Implementation Possibilities): encrypted direct messages, Lightning payments, long-form articles, and much more.

## Why another Nostr library?

When I started exploring Nostr from Swift, I wanted a library that felt like it was designed for the language rather than ported to it. That became the design brief for swift-nostr:

- **Modern concurrency, end to end.** The client is an actor, every public type is `Sendable`, and the package compiles under Swift 6.3 strict concurrency. No callbacks, no delegate soup.
- **Subscriptions are `AsyncSequence`s.** Reading a timeline is a `for await` loop; ending the loop closes the subscription. Structured concurrency does the cleanup.
- **The whole protocol, not just the basics.** Full NIP-01 events and subscriptions, plus contact lists, encrypted DMs, zaps, wallet payments, remote signing, the outbox model, proof of work, and more — 33 NIPs in total.
- **Layered, so you take only what you need.** Protocol primitives are separate from the high-level client, and the wallet and remote-signing stacks live in their own libraries.

## Four libraries, one package

The package vends four products:

- **`NostrCore`** — the primitives: `Event`, `KeyPair`, `EventSigner`, `Filter`, bech32/NIP-19 encoding, NIP-44 encryption, and a single-relay WebSocket transport. Schnorr signatures over secp256k1 under the hood.
- **`NostrClient`** — the high-level, actor-based client: a multi-relay pool, publishing with acknowledgment strategies, live subscriptions, NIP-17 private DMs, NIP-65 outbox routing, and one-shot fetch helpers.
- **`NostrWalletConnect`** — NIP-47 Nostr Wallet Connect: pay Lightning invoices and zaps through a remote wallet, check balances, create invoices, and stream wallet notifications.
- **`NostrConnect`** — NIP-46 remote signing: delegate signing to a *bunker* that holds the user's key, so your app never touches an `nsec`.

## A taste of the API

Generate an identity, connect to a couple of relays, and publish a note:

```swift
import NostrClient
import NostrCore

let keyPair = try KeyPair()
print(try keyPair.npub)

let client = NostrClient()
try await client.connect(to: ["wss://relay.damus.io", "wss://nos.lol"])
try await client.setNsec(try keyPair.nsec)

let note = try await client.publishTextNote(content: "Hello, Nostr!")
print("Accepted by \(note.result.acceptedRelays.count) relay(s)")
```

Reading is just as direct — subscriptions are async sequences:

```swift
let timeline = try await client.subscribeToUserTimeline(pubkey: somePubkey)
for await event in timeline.events {
    print(event.content)
}
```

End-to-end encrypted direct messages (NIP-17) arrive already decrypted and parsed:

```swift
for await message in try await client.directMessages() {
    print("\(message.senderPubkey): \(message.content)")
}
try await client.sendDirectMessage("Hello privately!", to: recipientPubkey)
```

And paying a Lightning zap through a remote wallet (NIP-47) is one call:

```swift
import NostrWalletConnect

let connection = WalletConnection(uri: try WalletConnectURI(string: "nostr+walletconnect://…"))
let payment = try await connection.payInvoice("lnbc…")
print(payment.preimage)
```

## What's new in 0.6.0

The 0.6.0 release is the largest so far:

- **Ten new NIPs**, including proof of work (NIP-13), long-form articles (NIP-23), search (NIP-50), lists and sets (NIP-51), event counts (NIP-45), reporting (NIP-56), and `ncryptsec` private-key encryption (NIP-49).
- **A new `NostrConnect` library** for NIP-46 remote signing — both the signer-initiated `bunker://` flow and the client-initiated `nostrconnect://` flow, with a `NostrSigning` protocol so a remote signer plugs straight into `NostrClient`.
- **A spec-compliance fix to NIP-44 encryption**, validated against the official test vectors, so encrypted payloads now interoperate with other Nostr clients.

## Learn by example

Alongside the library there's a companion repo, [swift-nostr-examples](https://github.com/yysskk/swift-nostr-examples): nearly forty small, self-contained, unit-tested "case studies" covering everything from generating a keypair to reconstructing threads, building zap requests, and connecting to a bunker. It also ships a SwiftUI showcase app for iOS and macOS that lets you browse every example and run several of them live against real relays. If you learn best by reading working code, start there.

## Getting started

swift-nostr requires Swift 6.3+ and runs on iOS 17, macOS 14, tvOS 17, watchOS 10, and visionOS 1. Add it with Swift Package Manager:

```swift
dependencies: [
    .package(url: "https://github.com/yysskk/swift-nostr", from: "0.6.0")
]
```

The [API documentation](https://yysskk.github.io/swift-nostr/documentation/) covers all four libraries, with Getting Started and Advanced Usage guides for each.

The library is MIT-licensed and contributions are welcome. If you build something with it — or run into something that could be better — I'd love to hear about it on [GitHub](https://github.com/yysskk/swift-nostr). I'm also writing a hands-on implementation guide that walks through building a Nostr client with swift-nostr from scratch — that post is up next.
