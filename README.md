# BOXX

**BOXX** is a minimal Chromium extension for signing Nostr events.

It works as a local browser-based Nostr signer that exposes a NIP-07-compatible `window.nostr` provider to web applications, allowing users to connect their Nostr identity and sign events without exposing their private key to websites.

Repository:

```txt
github.com/zuyux/boxx
```

---

## Overview

BOXX is designed as a simple, auditable, and developer-friendly Nostr signer extension.

It allows Nostr web clients to request:

```ts
window.nostr.getPublicKey()
window.nostr.signEvent(event)
```

The private key remains inside the extension. Websites can request signatures, but they never receive direct access to the user’s secret key.

---

## Goals

BOXX aims to provide:

* A basic functional Nostr signer
* NIP-07-compatible browser API
* Local key generation
* Local `nsec` import
* Encrypted private key storage
* Public key exposure through `window.nostr.getPublicKey()`
* Event signing through `window.nostr.signEvent()`
* Per-origin connection permissions
* User approval before signing events
* Simple popup interface
* Clean WXT + TypeScript codebase

---

## Non-Goals

The first version of BOXX does not aim to be a full Nostr client.

It does not include:

* Relay management
* Nostr feed browsing
* Direct messaging
* NIP-46 remote signing
* Multi-account support
* Hardware signer support
* Social recovery
* Mobile support
* Cloud sync

These features may be added later.

---

## Tech Stack

BOXX is built with:

```txt
WXT
TypeScript
Manifest V3
Chromium Extensions API
Nostr event signing utilities
chrome.storage.local
Web Crypto API
```

Recommended cryptographic/event libraries:

```txt
nostr-tools
@noble/secp256k1
@noble/hashes
```

---

## Basic Architecture

```txt
Nostr Web App
    │
    │ window.nostr
    ▼
Injected Provider
    │
    │ window.postMessage()
    ▼
Content Script
    │
    │ chrome.runtime.sendMessage()
    ▼
Background Service Worker
    │
    ├── Key Vault
    ├── Origin Permissions
    ├── Event Validator
    └── Signing Engine
            │
            ▼
        Popup Approval UI
```

---

## Main Extension Contexts

### Injected Provider

The injected provider runs in the webpage context.

It exposes:

```ts
window.nostr.getPublicKey()
window.nostr.signEvent(event)
```

The injected provider does not access private keys. It only forwards requests to the extension.

---

### Content Script

The content script acts as a bridge between the webpage and the extension runtime.

Responsibilities:

* Inject the provider script
* Listen for provider messages
* Validate message source
* Attach the requesting origin
* Forward requests to the background worker
* Return responses to the webpage

The content script must never store or access private key material.

---

### Background Worker

The background worker is the trusted security boundary.

Responsibilities:

* Receive requests from content scripts
* Check origin permissions
* Manage unlock state
* Access encrypted vault
* Validate signing requests
* Ask the user for approval
* Sign Nostr events
* Return responses

The background worker is the only extension context allowed to access decrypted key material.

---

### Popup UI

The popup is the main user interface.

It should allow the user to:

* Create a new Nostr key
* Import an existing `nsec`
* Unlock BOXX
* View their public key
* See the current connection status
* Approve or reject signing requests
* Lock the signer
* Access settings

---

## NIP-07 API

BOXX exposes a minimal NIP-07-compatible interface:

```ts
window.nostr = {
  getPublicKey(): Promise<string>,
  signEvent(event: UnsignedNostrEvent): Promise<SignedNostrEvent>
}
```

Example unsigned event:

```ts
const event = {
  kind: 1,
  created_at: Math.floor(Date.now() / 1000),
  tags: [],
  content: "gm nostr"
};
```

Example usage from a Nostr web client:

```ts
const pubkey = await window.nostr.getPublicKey();

const signedEvent = await window.nostr.signEvent({
  kind: 1,
  created_at: Math.floor(Date.now() / 1000),
  tags: [],
  content: "Signed with BOXX"
});
```

---

## Security Model

BOXX assumes every website is untrusted by default.

Core rules:

```txt
Websites can request signatures.
Websites cannot access private keys.
Content scripts cannot access private keys.
Private keys are encrypted at rest.
Decrypted keys only live in memory.
Every origin must be permissioned.
Every signing request requires user approval in the MVP.
```

---

## Private Key Storage

BOXX stores the user’s private key encrypted in `chrome.storage.local`.

The vault should contain only encrypted data.

Example vault model:

```ts
type EncryptedVault = {
  version: 1;
  pubkey: string;
  ciphertext: string;
  salt: string;
  iv: string;
  kdf: {
    name: "PBKDF2";
    iterations: number;
    hash: "SHA-256";
  };
};
```

Private keys must never be stored in plaintext.

---

## Origin Permissions

BOXX manages permissions per website origin.

Example:

```ts
type OriginPermission = {
  origin: string;
  connected: boolean;
  publicKeyAccess: boolean;
  signEventAccess: boolean;
  createdAt: number;
  updatedAt: number;
};
```

Default MVP policy:

```txt
getPublicKey requires approval.
signEvent always requires explicit user approval.
```

Future versions may allow trusted origins to auto-sign selected low-risk event kinds.

---

## Signing Flow

```txt
1. A website calls window.nostr.signEvent(event)
2. BOXX receives the request
3. BOXX checks the website origin
4. BOXX validates the event structure
5. BOXX opens an approval prompt
6. The user reviews the event
7. The user approves or rejects the request
8. If approved, BOXX signs the event
9. The signed event is returned to the website
```

---

## Event Validation

Before signing, BOXX should validate:

* `kind` is a number
* `created_at` is a number
* `created_at` is not too far in the future
* `tags` is an array of arrays
* `content` is a string
* Event size is within safe limits
* Origin is approved
* Extension is unlocked
* User approved the request

BOXX should warn users before signing sensitive events such as:

* Profile metadata updates
* Contact list updates
* Deletion events
* Large events
* Unknown high-risk event kinds

---

## Project Structure

Recommended structure:

```txt
boxx/
├── README.md
├── ARCHITECTURE.md
├── package.json
├── tsconfig.json
├── wxt.config.ts
├── public/
│   └── icons/
└── src/
    ├── entrypoints/
    │   ├── background.ts
    │   ├── content.ts
    │   ├── injected.ts
    │   ├── popup/
    │   │   ├── index.html
    │   │   ├── main.tsx
    │   │   └── App.tsx
    │   └── options/
    │       ├── index.html
    │       ├── main.tsx
    │       └── App.tsx
    ├── core/
    │   ├── nostr/
    │   │   ├── keys.ts
    │   │   ├── signer.ts
    │   │   └── events.ts
    │   ├── vault/
    │   │   ├── encryption.ts
    │   │   ├── key-vault.ts
    │   │   └── session.ts
    │   ├── permissions/
    │   │   └── origins.ts
    │   └── messaging/
    │       ├── messages.ts
    │       └── router.ts
    └── types/
        └── nostr.ts
```

---

## Installation

Clone the repository:

```bash
git clone https://github.com/zuyux/boxx.git
cd boxx
```

Install dependencies:

```bash
pnpm install
```

Run the development server:

```bash
pnpm dev
```

---

## Load in Chromium

After running the development server:

1. Open Chromium or Chrome.
2. Go to:

```txt
chrome://extensions
```

3. Enable **Developer mode**.
4. Click **Load unpacked**.
5. Select the generated WXT output directory.

Usually this will be:

```txt
.output/chrome-mv3
```

---

## Build

Build the extension:

```bash
pnpm build
```

Build specifically for Chrome:

```bash
pnpm build --browser chrome
```

Package the extension:

```bash
pnpm zip
```

---

## Development Milestones

### Milestone 0 — Project Setup

* Initialize WXT project
* Configure TypeScript
* Load extension in Chromium
* Open popup successfully

### Milestone 1 — Key Management

* Generate Nostr private key
* Import `nsec`
* Derive public key
* Encrypt private key locally
* Unlock signer with password
* Lock signer manually

### Milestone 2 — NIP-07 Provider

* Inject `window.nostr`
* Implement `getPublicKey()`
* Implement `signEvent()`
* Bridge webpage requests to background worker
* Return typed responses and errors

### Milestone 3 — Signing Approval

* Detect requesting origin
* Show approval prompt
* Display event preview
* Approve signing
* Reject signing
* Store origin permissions

### Milestone 4 — Security Hardening

* Add event validation
* Add auto-lock
* Prevent secret logging
* Add permission review UI
* Test malicious website requests

---

## Testing

Recommended test areas:

```txt
Key generation
nsec import
Public key derivation
Event hashing
Event signing
Vault encryption
Vault decryption
Message routing
Origin permission checks
User rejection flow
Locked signer behavior
```

Manual browser test:

```html
<button id="pubkey">Get Public Key</button>
<button id="sign">Sign Event</button>

<script>
  document.getElementById("pubkey").onclick = async () => {
    const pubkey = await window.nostr.getPublicKey();
    console.log(pubkey);
  };

  document.getElementById("sign").onclick = async () => {
    const event = await window.nostr.signEvent({
      kind: 1,
      created_at: Math.floor(Date.now() / 1000),
      tags: [],
      content: "Testing BOXX"
    });

    console.log(event);
  };
</script>
```

---

## Roadmap

Possible future improvements:

* Multi-account support
* NIP-04 support
* NIP-44 support
* NIP-46 remote signer mode
* Hardware signer support
* Event kind policies
* Trusted site permissions
* Encrypted backup file
* QR import/export
* Relay permission profiles
* Bitcoin and Lightning integrations

---

## Security Notice

BOXX is experimental software.

Do not use with high-value keys until the code has been reviewed, tested, and audited. Follow standard self-custody practices, including backing up your secret recovery phrase. You hold your own keys; we cannot restore lost phrases or recover funds. Bugs or software issues may still cause losses. Use at your own risk.

Recommended safety practices:

```txt
Use a test key during development.
Never paste your main nsec into unknown builds.
Review signing requests carefully.
Do not approve unknown websites.
Lock BOXX when not in use.
```

---

## Contributing

Contributions are welcome.

Useful contribution areas:

* NIP-07 provider implementation
* Secure vault encryption
* Signing approval UX
* Event validation
* WXT extension structure
* Tests
* Documentation
* Security review

Suggested workflow:

```bash
git checkout -b feature/my-change
pnpm install
pnpm dev
pnpm build
```

Then open a pull request with a clear explanation of the change.

---

## License

MIT
