# tap. — Executive Summary & Software Development Plan

## Executive Summary

**tap.** is a hyper-local, strictly zero-server social discovery and peer-to-peer (P2P) sharing mobile application designed for adults (18+) that reimagines the tactile magic of the classic Bump app using modern short-range wireless hardware. Built around an intuitive four-verb everyday vocabulary—**`tap.`** (physically touching phones to exchange contact cards, photos, and files), **`nod.`** (making friendly, platonic connections), **`wink.`** (signaling mutual romantic or adult interest), and **`hush.`** (end-to-end encrypted, device-to-device chat)—the platform eliminates cloud servers, user accounts, and centralized data harvesting entirely. Users default to a privacy-first "Ghost Mode" and toggle "See Me" to broadcast ephemeral, cryptographically hashed attributes (nickname, age, race, red/blue political leaning, and customizable interests ranging from *Study Mate* and *Coffee & Chat* to *DTF*, *FWB*, and custom write-ins) to nearby peers, while a one-tap "Meet Someone New" control locally suppresses previously encountered contacts from the live proximity radar. Beyond digital discovery, **tap.** bridges the physical world through NFC/QR rear car window decals that broadcast driver-approved vibe cards in range and a server-free local commerce engine that lets nearby businesses broadcast Bluetooth BOGO offers and validate single-use discounts offline at the register.

---

## Software Development Plan

### 1. Architectural Philosophy & Zero-Server Mandate
The foundational engineering constraint of **tap.** is **100% local-first execution**. There are no backend REST/GraphQL APIs, no cloud user databases, no push-notification relay servers, and no centralized analytics pipelines. 
* **Cryptographic Identity:** Upon first launch, the client generates a local **Curve25519** elliptic-curve keypair. The public key acts as the user's sole peer identifier; no phone numbers, email addresses, or OAuth logins are collected.
* **Asymmetric Privacy:** Filtering and discovery happen exclusively on-device. Sensitive demographic and intent filters are never broadcast in plaintext over the air.

---

### 2. Core Technology Stack

| Layer | Recommended Technology | Engineering Rationale |
| :--- | :--- | :--- |
| **Cross-Platform UI** | **Flutter (Dart)** | Delivers a single high-frame-rate codebase for the obsidian-themed radar canvas, gesture drawers, and custom lower-case `tap.` typography while supporting direct native platform channels. |
| **Native Hardware Layer** | **Swift (iOS)** & **Kotlin (Android)** | Required for low-level access to background Bluetooth advertising, Ultra-Wideband ranging, Wi-Fi Aware / Multipeer Connectivity, NFC tag I/O, and high-frequency accelerometer streams. |
| **Encrypted Persistence** | **SQLCipher (AES-256)** | Encrypts the entire on-device SQLite database file at rest. The database decryption key is generated at install and stored inside the **iOS Keychain** and **Android Keystore**. |
| **P2P Cryptography** | **libsignal (Double Ratchet)** | Implements Curve25519 ECDH handshakes and the Double Ratchet algorithm over local sockets to guarantee forward secrecy for all **`hush.`** conversations. |
| **On-Device ML Safety** | **Apple Sensitive Content Analysis / CoreML & LiteRT** | Performs local, offline NSFW image classification on incoming P2P media attachments to satisfy 18+ App Store and Google Play safety requirements without server moderation. |

---

### 3. Wireless Discovery & Handshake Protocols

Because iOS and Android impose strict sandboxing on background radios and cross-platform Wi-Fi, the engineering team will implement a **three-phase progressive hardware handshake**:

#### Phase 1: Low-Power BLE Radar (`nod.` & `wink.`)
* **Advertising Packet (31 Bytes Max):** When "See Me" is active, the device advertises a compact Bluetooth Low Energy (BLE) packet containing:
  1. `16 bytes`: App-specific 128-bit Service UUID.
  2. `4 bytes`: Ephemeral Peer ID (rotated every 15 minutes to prevent third-party physical location tracking).
  3. `8 bytes`: Salted cryptographic hashes combining the user's active intent tags (`nod.` vs. `wink.`) and visibility rules (age range, political leaning, race).
  4. `3 bytes`: Device capability flags (OS type, UWB support, active vendor offer).
* **Zero-Knowledge Matching:** Scanning devices compute local hashes of their own filter criteria and compare them against incoming BLE packets. A blip only renders on the radar when mutual filter hashes align and the peer's public key is not flagged as suppressed.

#### Phase 2: The Physical `tap.` Engine (Accelerometer + UWB + GATT)
* **Coincident Shock Detection:** When a user holds their phone on the Radar screen or arms the payload grid (Nickname, Phone, Photo, Full vCard), the native layer samples the 3-axis accelerometer at `100 Hz`.
* **Spatial Verification:** Upon detecting a sharp deceleration spike (physical phone-to-phone tap), the app queries **Ultra-Wideband (UWB)** (*Nearby Interaction* on iOS, UWB Jetpack API on Android) to verify a peer device is within `< 0.5 meters` (`~1.5 ft`).
* **Key Exchange:** The devices open a temporary BLE GATT connection, perform an ECDH key exchange, verify the simultaneous shock timestamp, and trigger synchronized haptic feedback.

#### Phase 3: High-Bandwidth P2P Wi-Fi Bridge (`hush.` & Media Transfer)
* Once a **`tap.`**, **`nod.`**, or **`wink.`** handshake is approved, the devices upgrade off BLE to transfer photos, contact cards, and real-time **`hush.`** messages:
  * **iOS-to-iOS:** Apple **Multipeer Connectivity** framework.
  * **Android-to-Android:** **Wi-Fi Aware (NAN)**.
  * **Cross-Platform (iOS-to-Android):** One device programmatically initializes an ephemeral local Wi-Fi Hotspot (SoftAP) with a randomized SSID and WPA2 passphrase, passing credentials over the encrypted BLE channel so the peer joins silently and opens a local WebSocket (`ws://192.168.x.x:8080`).

#### NFC Integration (Car Decals & Physical Badges)
* Users can write a custom NDEF payload to passive NFC/QR vinyl stickers. When another user scans the sticker in a parking lot or campus area, the app checks if the sticker's public key fingerprint is currently broadcasting within local BLE range and surfaces that user's designated "Car Mode" card.

---

### 4. Local Encrypted Data Schema (SQLCipher)

The database must maintain five core tables strictly siloed within the app's encrypted sandbox:

1. **`My_Profile` (Single Row):** Stores `identity_private_key`, `identity_public_key`, `nickname`, `avatar_local_path`, `active_intents_json`, `visibility_rules_json`, and `ghost_mode` boolean.
2. **`Known_Peers`:** Indexed by `peer_pubkey`. Tracks `nickname`, `avatar_hash`, `last_seen_timestamp`, `trust_level` (`0` = seen, `1` = connected, `-1` = blocked), and `is_suppressed` (`BOOLEAN`).
   * *Meet Someone New Execution:* Tapping **"Meet Someone New"** executes `UPDATE Known_Peers SET is_suppressed = 1 WHERE trust_level >= 0;`, instructing the BLE scanner to drop packets from all previously met peers immediately.
3. **`Double_Ratchet_State`:** Stores per-peer `root_key`, `send_chain_key`, `receive_chain_key`, and `message_index` to prevent replay attacks over local sockets.
4. **`Local_Messages`:** Stores `message_id`, `peer_pubkey`, `direction`, `message_type` (text, local image path, vCard, vendor promo), `content`, and `timestamp`. Includes a single-query "Burn Thread" purge method.
5. **`Local_Wallet`:** Stores nearby merchant offers (`offer_id`, `vendor_pubkey`, `offer_title`, `crypto_token`, `status`: `DETECTED | CLAIMED | REDEEMED`).

---

### 5. App Store Compliance & On-Device Guardrails

Because the app supports 18+ intent tags (`DTF`, `FWB`) and unmoderated P2P file sharing, passing Apple App Store and Google Play review requires strict client-side safeguards:
* **Local Age Gate:** Mandatory date-of-birth verification during onboarding (stored locally; blocks installation setup for users under 18).
* **Automated Media Blurring:** All images received over P2P Wi-Fi are passed through Apple's *Sensitive Content Analysis* API (iOS) or an embedded *LiteRT* NSFW classifier (Android) before rendering. Flagged images remain blurred behind a tap-to-reveal warning.
* **Zero-Latency Blocklist:** Blocking a peer sets `trust_level = -1` in `Known_Peers`, immediately severing active sockets and rejecting future BLE/UWB handshakes at the radio layer.

---

### 6. Offline Merchant Ecosystem & Monetization

To enable local business promotions without central ad servers:
* **Merchant "Vendor Mode" / BLE Beacons:** Participating businesses run a lightweight Vendor Mode on an in-store tablet or plug in a low-cost BLE beacon near their entrance that broadcasts a cryptographically signed offer packet (e.g., *"BOGO Iced Latte — 50 ft"*) within a `~100 ft` radius.
* **Non-Intrusive Discovery:** When a user opens **tap.** near a participating venue, a subtle, dismissible proximity pill appears above the bottom navigation bar.
* **Offline NFC / QR Redemption:** Claiming an offer saves a signed single-use token in `Local_Wallet`. At checkout, the customer taps their phone via NFC (or displays a rotating QR code) to the merchant's device, which validates the cryptographic signature offline and increments an on-device redemption counter.
* **Consumer Freemium IAP:** Core discovery, **`tap.`**, **`nod.`**, **`wink.`**, and **`hush.`** remain 100% free to maximize local network density, supplemented by an optional native In-App Purchase tier for multi-profile presets (e.g., *Campus Mode* vs. *Nightlife Mode*) and custom badge themes.

---

### 7. Phased Development Roadmap

* **Phase 1 — Foundation & Security Core (Weeks 1–3):**
  * Initialize Flutter project structure, custom `tap.` UI theme, and onboarding flow (18+ gate, Ghost Mode default).
  * Integrate SQLCipher, Keychain/Keystore key management, and Curve25519 identity generation.
* **Phase 2 — Cross-Platform BLE Radar & Filtering (Weeks 4–7):**
  * Build Swift and Kotlin native modules for background BLE advertising and scanning.
  * Implement salted hash matching for **`nod.`** and **`wink.`** filters and wire up the **"Meet Someone New"** suppression engine.
* **Phase 3 — Physical `tap.` Engine & `hush.` P2P Chat (Weeks 8–12):**
  * Implement 100 Hz accelerometer spike detection paired with UWB ranging for physical **`tap.`** exchanges.
  * Build the P2P Wi-Fi bridge (Multipeer Connectivity, Wi-Fi Aware, and cross-platform SoftAP WebSockets) with Double Ratchet encryption for **`hush.`**.
  * Add NFC NDEF read/write support for rear car window decals.
* **Phase 4 — Safety, Local Perks & Store Launch (Weeks 13–16):**
  * Integrate on-device NSFW image blurring, Vendor Mode BLE beacon broadcasting, and offline NFC coupon validation.
  * Conduct multi-device field testing (iOS-to-iOS, Android-to-Android, and iOS-to-Android) in high-density environments prior to App Store and Google Play submission.