# ARCHITECTURE.md

# BOXX Architecture

**BOXX** is a basic Chromium extension built with **WXT + TypeScript** that acts as a local **Nostr signer** for web applications.

The extension exposes a NIP-07-compatible `window.nostr` provider to websites, allowing Nostr clients to request the user’s public key and request signatures without directly accessing the user’s private key.

BOXX is designed as a minimal, auditable signer MVP.

---

## 1. Goals

BOXX provides:

* Local Nostr key generation
* Local private key import using `nsec`
* Secure local encrypted key storage
* NIP-07-compatible `window.nostr` provider
* Public key access through `window.nostr.getPublicKey()`
* Event signing through `window.nostr.signEvent(event)`
* Per-origin connection approval
* User confirmation before signing events
* Simple popup interface for unlock, account status, and settings

---

## 2. Non-Goals

The first BOXX MVP does not include:

* Relay management
* Nostr social client features
* Multi-account support
* Hardware wallet support
* NIP-46 remote signing
* NIP-04 / NIP-44 encryption
* Lightning payments
* Cloud sync
* Mobile support
* Social recovery
* Advanced policy engine

These features can be added later.

---

## 3. Core Architecture

BOXX follows a standard Manifest V3 extension architecture.

```txt
Web Page
  │
  │ window.nostr
  ▼
Injected Provider Script
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
  ├── Permission Manager
  ├── Signing Engine
  └── Request Router
        │
        ▼
Popup / Approval UI
```

The extension is split into isolated runtime contexts:

| Context           | Responsibility                                                      |
| ----------------- | ------------------------------------------------------------------- |
| Injected Provider | Exposes `window.nostr` to the webpage                               |
| Content Script    | Bridges the webpage and extension runtime                           |
| Background Worker | Handles privileged logic, key access, permissions, and signing      |
| Popup UI          | Lets the user unlock BOXX, view account state, and approve requests |
| Options UI        | Manages settings, backup, reset, and trusted sites                  |

---

## 4. Recommended Stack

```txt
WXT
TypeScript
Manifest V3
React for popup/options UI
@noble/secp256k1 for Schnorr signing
@noble/hashes for hashing
nostr-tools for Nostr event helpers
chrome.storage.local for encrypted local storage
Web Crypto API for encryption
```

Recommended package setup:

```bash
pnpm dlx wxt@latest init boxx
cd boxx
pnpm install

pnpm add nostr-tools @noble/secp256k1 @noble/hashes
```

Optional UI dependencies:

```bash
pnpm add react react-dom
```

---

## 5. Project Structure

```txt
boxx/
├── README.md
├── ARCHITECTURE.md
├── package.json
├── tsconfig.json
├── wxt.config.ts
├── public/
│   └── icons/
│       ├── icon-16.png
│       ├── icon-32.png
│       ├── icon-48.png
│       └── icon-128.png
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
    │   │   ├── events.ts
    │   │   └── nip07.ts
    │   ├── vault/
    │   │   ├── encryption.ts
    │   │   ├── key-vault.ts
    │   │   └── session.ts
    │   ├── permissions/
    │   │   ├── origins.ts
    │   │   └── policies.ts
    │   └── messaging/
    │       ├── messages.ts
    │       ├── router.ts
    │       └── errors.ts
    ├── ui/
    │   ├── components/
    │   └── styles/
    └── types/
        ├── nostr.ts
        └── global.d.ts
```

---

## 6. Manifest Configuration

WXT generates the manifest from the project configuration and entrypoints.

Example `wxt.config.ts`:

```ts
import { defineConfig } from 'wxt';

export default defineConfig({
  manifest: {
    name: 'BOXX',
    description: 'A minimal Nostr signer for Chromium browsers.',
    version: '0.1.0',
    manifest_version: 3,
    permissions: [
      'storage'
    ],
    host_permissions: [
      '<all_urls>'
    ],
    action: {
      default_title: 'BOXX'
    }
  }
});
```

### Permission Notes

BOXX needs broad page access only because NIP-07 signers must be available to arbitrary Nostr web clients.

However, access should be controlled internally through a strict origin permission system.

Example:

```txt
Extension can inject provider into many sites.
But each site still needs explicit user approval before accessing signing features.
```

---

## 7. NIP-07 Provider

BOXX exposes the following minimal NIP-07 API:

```ts
window.nostr = {
  getPublicKey(): Promise<string>,
  signEvent(event: UnsignedNostrEvent): Promise<SignedNostrEvent>
};
```

### MVP API

```ts
type UnsignedNostrEvent = {
  kind: number;
  created_at: number;
  tags: string[][];
  content: string;
};

type SignedNostrEvent = UnsignedNostrEvent & {
  id: string;
  pubkey: string;
  sig: string;
};
```

### Future API Extensions

Later versions may add:

```ts
window.nostr.getRelays()
window.nostr.nip04.encrypt()
window.nostr.nip04.decrypt()
window.nostr.nip44.encrypt()
window.nostr.nip44.decrypt()
```

---

## 8. Runtime Flow

### 8.1 Public Key Request

```txt
1. Web app calls window.nostr.getPublicKey()
2. Injected provider sends request to content script
3. Content script forwards request to background worker
4. Background worker checks origin permissions
5. If origin is not approved, user is prompted
6. If approved, background returns public key
7. Provider resolves promise to web app
```

### 8.2 Event Signing Request

```txt
1. Web app calls window.nostr.signEvent(unsignedEvent)
2. Injected provider forwards event request
3. Content script forwards request to background worker
4. Background validates request origin
5. Background validates event structure
6. Background opens approval UI
7. User approves or rejects signing request
8. If approved, background signs event
9. Signed event is returned to web app
```

---

## 9. Message Protocol

All communication between contexts should use typed messages.

```ts
export type BoxxRequest =
  | {
      type: 'NOSTR_GET_PUBLIC_KEY';
      origin: string;
      requestId: string;
    }
  | {
      type: 'NOSTR_SIGN_EVENT';
      origin: string;
      requestId: string;
      event: UnsignedNostrEvent;
    };

export type BoxxResponse =
  | {
      type: 'SUCCESS';
      requestId: string;
      result: unknown;
    }
  | {
      type: 'ERROR';
      requestId: string;
      error: {
        code: string;
        message: string;
      };
    };
```

Recommended error codes:

```txt
USER_REJECTED
ORIGIN_NOT_ALLOWED
EXTENSION_LOCKED
INVALID_EVENT
KEY_NOT_FOUND
SIGNING_FAILED
INTERNAL_ERROR
```

---

## 10. Injected Script

The injected script runs in the webpage context and creates `window.nostr`.

Because content scripts run in an isolated extension world, BOXX should inject a page-level provider script.

Example responsibility:

```txt
src/entrypoints/injected.ts
```

```ts
window.nostr = {
  async getPublicKey() {
    return requestBoxx('NOSTR_GET_PUBLIC_KEY');
  },

  async signEvent(event) {
    return requestBoxx('NOSTR_SIGN_EVENT', { event });
  }
};
```

The injected script should never access private keys.

It only forwards requests.

---

## 11. Content Script

The content script acts as a bridge.

```txt
src/entrypoints/content.ts
```

Responsibilities:

* Inject provider into page
* Listen for `window.postMessage()` requests
* Validate message source
* Attach current page origin
* Forward requests to background worker
* Return responses to injected provider

The content script must not hold private keys.

---

## 12. Background Worker

```txt
src/entrypoints/background.ts
```

The background worker is the security boundary.

Responsibilities:

* Receive all provider requests
* Check origin permissions
* Handle unlock state
* Access encrypted key vault
* Request user approval
* Sign Nostr events
* Return responses
* Clear sensitive memory on lock

The background worker is the only extension context allowed to access decrypted private key material.

---

## 13. Key Vault

BOXX stores private keys encrypted in `chrome.storage.local`.

Private keys must never be stored in plaintext.

### Vault Model

```ts
type EncryptedVault = {
  version: 1;
  pubkey: string;
  ciphertext: string;
  salt: string;
  iv: string;
  kdf: {
    name: 'PBKDF2';
    iterations: number;
    hash: 'SHA-256';
  };
};
```

### Unlock Flow

```txt
1. User enters password
2. Password derives encryption key
3. Vault decrypts private key
4. Private key is stored only in memory
5. Extension becomes unlocked
6. User may lock manually
7. Session auto-locks after timeout
```

### Security Rules

```txt
Never expose nsec to webpage
Never expose private key to content script
Never store decrypted key in chrome.storage
Never send decrypted key through messages
Never log private keys
Never sync encrypted vault using chrome.storage.sync
```

---

## 14. Signing Engine

```txt
src/core/nostr/signer.ts
```

Responsibilities:

* Normalize event
* Attach user pubkey
* Calculate event ID
* Sign event ID using Schnorr signature
* Return signed event

Signing flow:

```ts
export async function signNostrEvent(
  unsignedEvent: UnsignedNostrEvent,
  privateKey: Uint8Array
): Promise<SignedNostrEvent> {
  const event = {
    ...unsignedEvent,
    pubkey: getPublicKey(privateKey)
  };

  const id = getEventHash(event);
  const sig = await schnorr.sign(id, privateKey);

  return {
    ...event,
    id,
    sig
  };
}
```

The signer should reject events that already contain suspicious or inconsistent fields.

---

## 15. Event Validation

Before signing, BOXX should validate:

```txt
kind is a number
created_at is a number
created_at is not too far in the future
tags is an array of arrays
content is a string
event size is within safe limits
origin is approved
extension is unlocked
user approved this request
```

Optional policy checks:

```txt
Warn on kind 0 profile metadata update
Warn on kind 3 contact list update
Warn on kind 5 deletion event
Warn on very large content
Warn on encrypted direct message events
Warn on unknown high-risk event kinds
```

---

## 16. Origin Permission Model

BOXX should manage permissions by website origin.

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

Example stored permissions:

```json
{
  "https://iris.to": {
    "connected": true,
    "publicKeyAccess": true,
    "signEventAccess": true
  },
  "https://snort.social": {
    "connected": true,
    "publicKeyAccess": true,
    "signEventAccess": false
  }
}
```

Default policy:

```txt
getPublicKey requires approval
signEvent always requires explicit confirmation in MVP
```

Future policy:

```txt
Allow trusted sites to auto-sign selected low-risk event kinds.
```

---

## 17. Approval UI

The approval UI should clearly show:

```txt
Requesting site
Requested action
Public key being used
Event kind
Event content preview
Tags preview
Raw event JSON toggle
Approve button
Reject button
```

For signing requests, the user should understand what they are signing.

Example approval screen:

```txt
Sign Nostr Event

Site:
https://example.com

Kind:
1 - Short Text Note

Content:
gm nostr

Actions:
[Reject] [Sign]
```

---

## 18. Popup UI

The popup is the user’s main control surface.

MVP screens:

```txt
Welcome
Create Key
Import Key
Unlock
Account
Pending Request
Settings
Backup Warning
```

### Popup Account Screen

Should show:

```txt
BOXX is unlocked
npub...
Connected sites
Lock button
Settings button
```

### Locked Screen

Should show:

```txt
Password input
Unlock button
Import key option
Create key option
```

---

## 19. Options UI

The options page handles slower configuration.

Recommended settings:

```txt
View public key
Export nsec after password confirmation
Change password
Delete local key
Manage connected sites
Set auto-lock timeout
View extension version
```

Dangerous actions must require confirmation.

---

## 20. Storage Layout

Use `chrome.storage.local`.

```ts
type BoxxStorage = {
  vault?: EncryptedVault;
  permissions: Record<string, OriginPermission>;
  settings: {
    autoLockMinutes: number;
    requireApprovalForEverySignature: boolean;
    showAdvancedEventJson: boolean;
  };
};
```

Do not use `localStorage` for secrets.

Do not use `chrome.storage.sync` for keys.

---

## 21. Session State

Unlocked private key material should only live in memory.

```ts
type SessionState = {
  unlocked: boolean;
  pubkey?: string;
  privateKey?: Uint8Array;
  unlockedAt?: number;
  expiresAt?: number;
};
```

When the browser restarts or the service worker dies, BOXX should return to locked state.

---

## 22. Security Model

BOXX assumes webpages are untrusted.

Security principles:

```txt
The webpage can request signatures.
The webpage must never access private keys.
The content script must never access private keys.
Only the background worker may access decrypted key material.
Every origin must be permissioned.
Every signature should be user-approved in the MVP.
All messages must be validated.
All event payloads must be validated.
```

### Threats

| Threat                               | Mitigation                        |
| ------------------------------------ | --------------------------------- |
| Malicious website requests signature | Show explicit approval UI         |
| Website impersonates another site    | Use browser-provided origin       |
| Content script compromise            | No key access in content script   |
| Storage leak                         | Encrypt vault before storage      |
| User signs malicious event           | Show readable event preview       |
| Overbroad permissions                | Keep manifest permissions minimal |
| Silent signing abuse                 | Disable auto-signing in MVP       |

---

## 23. Build Commands

Development:

```bash
pnpm dev
```

Build:

```bash
pnpm build
```

Build for Chrome:

```bash
pnpm build --browser chrome
```

Package:

```bash
pnpm zip
```

---

## 24. Development Workflow

Recommended implementation order:

```txt
1. Initialize WXT + TypeScript project
2. Add popup UI
3. Add key generation
4. Add encrypted vault
5. Add unlock flow
6. Add injected NIP-07 provider
7. Add content script bridge
8. Add background request router
9. Add getPublicKey()
10. Add signEvent()
11. Add approval UI
12. Add origin permissions
13. Add settings page
14. Add tests
15. Package extension
```

---

## 25. Testing Strategy

### Unit Tests

Test:

```txt
Key generation
nsec import/export
Public key derivation
Event hashing
Event signing
Vault encryption/decryption
Permission checks
Message validation
```

### Integration Tests

Test:

```txt
Injected provider appears on webpage
window.nostr.getPublicKey() works
window.nostr.signEvent() opens approval flow
Rejected request returns error
Locked extension blocks signing
Approved origin can request public key
Unapproved origin is blocked
```

### Manual Test Page

Create a local test page:

```html
<button id="pubkey">Get Public Key</button>
<button id="sign">Sign Event</button>

<script>
  document.getElementById('pubkey').onclick = async () => {
    console.log(await window.nostr.getPublicKey());
  };

  document.getElementById('sign').onclick = async () => {
    const event = await window.nostr.signEvent({
      kind: 1,
      created_at: Math.floor(Date.now() / 1000),
      tags: [],
      content: 'Testing BOXX signer'
    });

    console.log(event);
  };
</script>
```

---

## 26. MVP Milestones

### Milestone 0 — Project Setup

```txt
WXT initialized
TypeScript configured
Extension loads in Chromium
Popup opens correctly
```

### Milestone 1 — Local Key Management

```txt
Generate Nostr key
Import nsec
Encrypt key locally
Unlock with password
Lock extension
Display npub
```

### Milestone 2 — NIP-07 Provider

```txt
Inject window.nostr
Implement getPublicKey
Implement signEvent
Bridge messages safely
Return typed errors
```

### Milestone 3 — User Approval

```txt
Detect requesting origin
Show approval popup
Approve/reject public key access
Approve/reject signing request
Store trusted origins
```

### Milestone 4 — Security Hardening

```txt
Event validation
Auto-lock
No secret logging
Permission review
Manual audit
Test malicious request cases
```

---

## 27. Future Roadmap

Possible future improvements:

```txt
Multi-account support
NIP-04 encryption
NIP-44 encryption
NIP-46 remote signer mode
Hardware signer support
Session permissions
Event kind policies
Relay permission profiles
Bitcoin/Lightning integrations
Encrypted backup file
QR-based key import/export
BOXX mobile companion
```

---

## 28. Design Principle

BOXX should behave like a simple signing vault.

The browser page can ask.

The user decides.

The extension signs.

The private key never leaves BOXX.
