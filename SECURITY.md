# Security

Keywarden is a zero-knowledge password manager: it is designed so that your passwords cannot be read by anyone without your master password — not by a server (there isn't one), not by someone who copies your database, and not by the app's own developer.

## Threat model

**Protected against**

- Theft of the database file or a backup — contents are ciphertext only.
- A curious or malicious observer of the disk or a synced backup folder.
- Casual access to an unlocked machine — the vault auto-locks on inactivity, on system lock, and on sleep, and copied secrets clear from the clipboard automatically.

**Not protected against**

- A compromised operating system (keylogger, memory scraper, malware running as you). No local app can defend the vault once your machine is owned while unlocked.
- A forgotten master password with no saved recovery key. This is unrecoverable by design.

## How it works

| Layer | Choice |
| --- | --- |
| Master password -> key | **Argon2id** (memory 64 MiB, 3 iterations), unique 128-bit salt |
| Data encryption | **XChaCha20-Poly1305** (AEAD), a fresh random nonce per record |
| Integrity | Each record is bound to its id and schema version as additional authenticated data, so ciphertexts cannot be swapped or moved |
| Key hierarchy | The password-derived key (KEK) only unwraps a random data key (DEK); the DEK encrypts the vault. A **recovery key** is a second, independent wrapping of the DEK |
| Password change | Re-wraps the DEK under the new key — the vault is not re-encrypted, and the recovery key keeps working |
| Randomness | CSPRNG only (libsodium / Web Crypto); never `Math.random()` |
| In memory | Keys are zeroed on lock; secrets never touch disk in plaintext and never reach logs |

Keys (KEK, DEK, recovery key) never leave the main process and never reach the UI layer.

## Network activity

Keywarden is offline-first. It makes exactly two kinds of outbound request, both optional and both initiated by you:

- **Favicon fetch** — when you enter a site URL, the app fetches that site's icon directly from its own domain (no third-party service). Requests to private, loopback, and link-local addresses are blocked.
- **Breach check** — when you press "Check for breaches", it queries Have I Been Pwned using **k-anonymity**: only the first 5 characters of a password's SHA-1 hash are sent, never the password or the full hash.

There is no account system, no analytics, and no telemetry.

## Reporting a vulnerability

If you believe you have found a security issue, please **do not open a public issue**. Instead, open a [private security advisory](https://github.com/Skaikru0518/keywarden-releases/security/advisories/new) on this repository, or contact the maintainer directly. Include steps to reproduce and the affected version. You will get an acknowledgement as soon as possible.

## A note on signing

Installers are not yet code-signed, so Windows SmartScreen may warn on first install. Background updates are delivered through the same signed-by-GitHub release channel and are verified by the updater against the release manifest.
