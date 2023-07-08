# Sigauth Specifications

`status:draft` `auth:dolu`

## Introduction
**Sigauth** is a passwordless authentication protocol.

## Project Objectives
**Sigauth** aims to replace LNURL-Auth protocol.

LNURL Auth has been wildly adopted but has some limitation.
- Bitcoin wallets mostly failed to implement it correctly (wrong deriavation path)
- You can't import a same seed in two different wallets without having different accounts (it can be related to the first bullet point or because they are not all using [BIP 32](https://bips.xyz/32))
- Custodial wallets do not follow the specification (they could follow the specification, by generating a dedicated LNURL-Auth private key, but they don't)
- Most LNURL-Auth implementations are vulnerable to MITM/phishing.
- Using your wallet (with money on it) to login is a terrible idea. Using a Lightning network node is worst.

## Features
- Passwordless authentication
- [BIP 39](https://bips.xyz/39) private key

## Technical Architecture

Sigauth uses a "proximity check" to prevent Man in the middle (MITM) and Phishing attacks. 

### Transport methods

- Safe
  - WebRTC (LAN only)
  - Redirection
- Unsafe
  - Polling
  > Polling is for remote login only. ⚠️ MITM/Phishing attack are possible. Only use it when safe methods are not possible. **Service can decide to not implement it.**

### Private key generation

Sigauth uses [BIP 39](https://bips.xyz/39) to generate private keys. You can find different implementations in many languages in the official [BIP 39 specs](https://bips.xyz/39#reference-implementation).

### Authentication flow

- `Signer` Application where your private key is stored
- `Service` Website/Application you want to login. `service.com` url will be used as example

### 1. [Service] Challenge generation

A challenge is generated and stored by the `Service`. It should be unique (there cannot be 2 identical ones at the same time), for example, hex of random 32 bytes.
```js
crypto.randomBytes(32).toString('hex')
// b5780fe40bcd49eeb1714ac061ce6fdd71377c1f858cd0358a47ecd17232024c
```

`Service` creates a JSON with these different informations.
```json
{
  "id": "7b4078a8dda9ee7f886389602b5d930f590bcf55e57e5e7e7a1dd47c313ca4ea",
  "challenge": "b5780fe40bcd49eeb1714ac061ce6fdd71377c1f858cd0358a47ecd17232024c",
  "callback": "https://service.com/verify",
  "origin": "service.com",
  "transports": ["webrtc", "redirect", "polling"],
  "signaling": "wss://service.com"
}
```
- `id`: SHA256 result of the challenge encoded
  - `SHA256(JSON.stringify({challenge, callback, origin, transports, signaling}))`
- `challenge`: random string previously generated
- `callback`: callback url that should be called later with signed result
- `origin`: root url of the service
- `transports`: transport methods supported by the `service`
- `signaling`: Websocket url used for signaling WebRTC informations

`Service` stringify this JSON then encode it in base64url.
```js
base64url.encode(JSON.stringify(json))
//eyJpZCI6IjdiNDA3OGE4ZGRhOWVlN2Y4ODYzODk2MDJiNWQ5MzBmNTkwYmNmNTVlNTdlNWU3ZTdhMWRkNDdjMzEzY2E0ZWEiLCJjaGFsbGVuZ2UiOiJiNTc4MGZlNDBiY2Q0OWVlYjE3MTRhYzA2MWNlNmZkZDcxMzc3YzFmODU4Y2QwMzU4YTQ3ZWNkMTcyMzIwMjRjIiwiY2FsbGJhY2siOiJodHRwczovL3NlcnZpY2UuY29tL3ZlcmlmeSIsIm9yaWdpbiI6InNlcnZpY2UuY29tIiwidHJhbnNwb3J0cyI6WyJ3ZWJydGMiLCJyZWRpcmVjdCIsInBvbGxpbmciXSwic2lnbmFsaW5nIjoid3NzOi8vc2VydmljZS5jb20ifQ
```

`Serivce` can now expose a QR Code (for `webrtc` and `polling`) containing the previously base64url string generated and/or link with handler `sigauth:[base64url]` (for `redirect`)

### 2. [Signer] Challenge signature

#### Common steps (WebRTC, Redirection, Polling)

`Signer` decodes the challenge from base64url to JSON.
```js
base64url.decode("eyJpZCI6IjdiNDA3OGE4ZGRhOWVlN2Y4ODYzODk2MDJiNWQ5MzBmNTkwYmNmNTVlNTdlNWU3ZTdhMWRkNDdjMzEzY2E0ZWEiLCJjaGFsbGVuZ2UiOiJiNTc4MGZlNDBiY2Q0OWVlYjE3MTRhYzA2MWNlNmZkZDcxMzc3YzFmODU4Y2QwMzU4YTQ3ZWNkMTcyMzIwMjRjIiwiY2FsbGJhY2siOiJodHRwczovL3NlcnZpY2UuY29tL3ZlcmlmeSIsIm9yaWdpbiI6InNlcnZpY2UuY29tIiwidHJhbnNwb3J0cyI6WyJ3ZWJydGMiLCJyZWRpcmVjdCIsInBvbGxpbmciXSwic2lnbmFsaW5nIjoid3NzOi8vc2VydmljZS5jb20ifQ")
```
`Signer` should verify that the `id` challenge is correct by calculating SHA256 of the challenge. It aims to verify the challenge has not been altered.
```js
challenge.id === SHA256(JSON.stringify({challenge, callback, origin, transports, signaling}))
```

`Signer` prompt the user if he wants to login on `service.com` (`challenge.origin`).
When accepted, `Signer` add its public key (Hex) to the current challenge object.

```js
const challenge = {
    "id": "7b4078a8dda9ee7f886389602b5d930f590bcf55e57e5e7e7a1dd47c313ca4ea",
    "challenge": "b5780fe40bcd49eeb1714ac061ce6fdd71377c1f858cd0358a47ecd17232024c",
    "callback": "https://service.com/verify",
    "origin": "service.com",
    "transports": ["webrtc", "redirect", "polling"],
    "publicKey": "HEX public key"
}
```

`Signer` signs (using schnorr) the challenge using its private key
```js
const sig = schnorr.sign(challenge, privateKey)
```

`Signer` adds 2 query string parameters to the callback URL
- `token`: containing the modified challenge, base64url encoded
- `sig`: containing signature in hex format

```js
const tokenBase64url = base64url.encode(JSON.stringify(challenge))
const callbackUrl = `https://service.com/verify?token=${tokenBase64url}&sig=${sig}`
```

#### WebRTC

WebRTC is used when `service` and `signer` are on the same device or not, but connected to the same local network.
- Chrome (Windows, Linux, Mac OS, ...) opening `service.com` and `signer` is a browser extension or is on another device, like a mobile app

Communication using WebRTC is established directly between the `Signer` and the client part of the `Service` (eg: the browser where the QR Code is displayed).

`Signer` scans QR code generated by `Service`

`Signer` verifies that `transports` properties contains `webrtc`.

⚠️ TODO WebRTC Signaling explanation

> Follow [Common steps (WebRTC, Redirection, Polling)](#common-steps)

Once the callback URL is generated, `Signer` has to send it to the `Service` using WebRTC. Once it has been received, `Service` can then use it in its workflow (HTTP call, redirect, etc).

`Service` can now verify. See the [3. [Service] Challenge verification](#service-verification) step.


#### Redirection

Redirection is used when `service` and `signer` are on the same device.
- Safari (iOS) opening `service.com` and `signer` is an iOS mobile app installed on the same device


User goes on `service.com` and click on a link (`sigauth:eyJjaGFsbGVuZ...`).\
`Signer` has `sigauth` handler registered. The app is opened and decodes the challenge.\
`Signer` verifies that `transports` properties contains `redirect`.

> Follow [Common steps (WebRTC, Redirection, Polling)](#common-steps)

`Signer` add a query string paramter to the callback url to indicate it's a redirect methode.
```
https://service.com/verify?token=XXX&sig=XXX&redirect=true
```
`Signer` opens the callback URL

`Service` can now verify. See the [3. [Service] Challenge verification](#service-verification) step.

#### Polling

⚠️ TODO. LNURL-Auth like flow

### 3. [Service] Challenge verification

`Service` should verify the signed challenge received has not been altered by checking initial properties.
```
challenge.id === initialChallenge.id
challenge.challenge === initialChallenge.challenge
challenge.callback === initialChallenge.callback
challenge.origin === initialChallenge.origin
challenge.transport === initialChallenge.transport
```

`Signer` should validate the signature
```js
schnorr.verify(sig, challenge, challenge.publicKey)
```

If the transport method is `redirect`, `Service` should redirect the user to a specified URL, like in the OAuth2 flow.


## (Optional) Fetching user info from other networks

Because Sigauth uses [BIP 39](https://bips.xyz/39), other protocols using BIP 39 can be integrated into Sigauth to fetch user data.
This part can be used by the `Service` to auto populate user info, like name, avatar, website, etc.

Before signing the challenge, `Signer` can add an optional property into the challenge object containing required information for fetching user data.
```json
{
    "id": "7b4078a8dda...c313ca4ea",
    "challenge": "b5780fe40bc...17232024c",
    "callback": "https://service.com/verify",
    "origin": "service.com",
    "transports": ["webrtc", "redirect", "polling"],
    "publicKey": "HEX public key",
    "PROTOCOL_NAME": {
      ...
    }
}
```

### [Nostr](https://github.com/nostr-protocol/nostr)
`Signer` can add some Nostr relays where the user informations are stored.
```json
{
    "id": "7b4078a8dda...c313ca4ea",
    "challenge": "b5780fe40bc...17232024c",
    "callback": "https://service.com/verify",
    "origin": "service.com",
    "transports": ["webrtc", "redirect", "polling"],
    "publicKey": "HEX public key",
    "nostr": {
      "relays": ["wss://relay1.com", "wss://relay2.com"]
    }
}
```



## References

LNURL Auth specification links
- https://github.com/lnurl/luds/blob/luds/04.md
- https://github.com/lnurl/luds/blob/luds/05.md
- https://github.com/lnurl/luds/blob/luds/17.md


## Sepcial thanks

Luke Childs for [his initial proposition](https://github.com/nostr-protocol/nips/issues/154)
