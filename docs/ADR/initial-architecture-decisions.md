# Architecture Decision Records (ADR): LAN Clipboard Synchronization

This document acts as the central record of key architectural decisions made during the system design. It complements functional and technical requirements by detailing the "why" behind each strategic choice.

---

## 1. Decision Log (ADR Log)

### ADR-001: Use of Shizuku for Clipboard Access on Android

* **Context:** Starting with Android 10 (API 29), background applications cannot access the `ClipboardManager` for security reasons. This prevents automatic synchronization if the app is not on screen.
* **Decision:** Use the **Shizuku** API to interact with the operating system's binder interface (`IClipboardManager`) with Shell/ADB or Root privileges.
* **Consequences:** Allows transparent reading and writing of the clipboard in the background. Requires the user to have the Shizuku app installed and activated via Wireless Debugging or Root.

### ADR-002: Implementation of a 100% Reactive Model (Zero Polling)

* **Context:** Constant monitoring of the clipboard using timers (*polling*) generates excessive CPU consumption and drains battery on mobile devices.
* **Decision:** Subscribe directly to operating system events (e.g., `OnPrimaryClipChangedListener` on Android via binder).
* **Consequences:** Code only executes when a real change occurs, maximizing energy efficiency. On Windows, this requires the use of STA (Single-Threaded Apartment) threads to handle clipboard window messages.

### ADR-003: Adoption of Native Kotlin over Multiplatform Frameworks

* **Context:** .NET MAUI was evaluated to simplify development, but the low-level integration required by Shizuku and Binder management on Android is extremely complex outside the native ecosystem.
* **Decision:** Develop the Android client exclusively in **native Kotlin**.
* **Consequences:** Direct access to low-level APIs and better performance, although it implies maintaining separate codebases for each platform.

### ADR-004: Hybrid P2P Network Model (UDP Broadcast + On-Demand TCP)

* **Context:** Persistent TCP connections (such as WebSockets) between multiple nodes generate overhead. UDP alone does not guarantee delivery of large files.
* **Decision:** Use **UDP Broadcast** for notifications and light texts, and file transfer via **on-demand TCP** (socket opening after UDP notice, streaming transfer, and closing).
* **Consequences:** Low latency for text and high reliability for files, eliminating the embedded HTTP server and reducing overhead.
* **Justification (TCP vs HTTP):** HTTP was discarded for file transfer due to its request/response model and header/parsing overhead. TCP allows a more efficient direct streaming model in LAN, with lower latency, greater control over data flow (chunking, retries), and without the need to expose HTTP endpoints on each node.

---

## 2. Functional Requirements

* **FR01 - Text Listening and Injection:** The system must reactively detect (without polling) the copying of plain text and URLs, transmit them to the network, and upon receipt, inject them directly into the target operating system's clipboard.
* **FR02 - File Transfer:** The system must allow copying images or heavy files. The sender will notify availability and the receiver will establish an on-demand TCP connection to download the file via streaming.
* **FR03 - Cache Management:** The sending node must maintain only the last 3 files in memory/disk. Requests for older files must be rejected.
* **FR04 - Group Filtering:** The system must allow multiple clipboard networks to coexist in the same LAN. Devices will only process messages from their own `group_id`.
* **FR05 - Optical Pairing (QR):** The system must generate and read QR codes to transfer network credentials to devices with a camera.
* **FR06 - Numeric Pairing (PIN):** For devices without a camera, the system must allow pairing through visual confirmation of a 6-digit PIN (derived from ECDH).
* **FR07 - Network Fault Tolerance:** UDP emissions must be sent in bursts of 3 staggered attempts. Receivers must deduplicate messages.
* **FR08 - Native Configuration Interface (Linux):** Provide a minimalist UI invocable on demand to manage devices and pairings.
* **FR09 - Desktop Environment Integration (GNOME):** Include an Applet/Extension in the top panel for status reading and quick actions.
* **FR10 - Node Discovery (mDNS):** Emit presence announcements upon connecting to the network to allow automatic node detection.

---

## 3. Non-Functional Requirements

* **NFR01 - Transit Security:** End-to-end encryption with **AES-256-GCM**.
* **NFR02 - Key Isolation:** Credentials never travel in plain text; they are transferred via QR or temporary TCP tunnel with ECDH.
* **NFR03 - Secure Storage:** Use of native mechanisms (Android Keystore, Windows DPAPI, strict file permissions on Linux).
* **NFR04 - Performance:** Minimize battery and RAM consumption. Background services without main GUI (Headless).
* **NFR05 - Mobile Compatibility:** Support from Android 10 (API 29) onwards.
* **NFR06 - Interface Decoupling:** Architecture based on **D-Bus** to separate the UI from the logic of the daemon on Linux.

---

## 4. Technology Stack Definition

### 4.1 Android Client
* **Language:** Kotlin (MVVM, Coroutines).
* **System:** Foreground Service, Shizuku API (AIDL interface).
* **Network:** `DatagramSocket` (UDP) and on-demand TCP sockets.

### 4.2 Windows Client
* **Language:** C# (.NET 8+).
* **Compilation:** Standard (Non-AOT) to ensure full compatibility with UI libraries.
* **System:** Win32 API (P/Invoke) on STA thread, `IHostedService`.
* **Network:** `UdpClient`, on-demand TCP sockets.

### 4.3 Linux Client
* **Frontend (UI):** GTK4 + libadwaita (Python) and GNOME Extension (GJS).
* **Engine (Daemon):** **Go (Golang)** as a static binary. Use of Goroutines and `godbus/dbus` for IPC.

---

## 5. Protocols and Data Structures

**UDP Packet Structure:**
```json
{
  "v": 1,
  "group_id": "PlainText",
  "id": "Deduplication_Timestamp",
  "type": "text | file_notice",
  "tcp_port": "Only if file",
  "iv": "InitializationVector_Base64",
  "payload": "EncryptedText_Or_Metadata_Base64"
}
```

**Discovery Mechanism (mDNS):**
1. **Announcement:** Passive emission of presence upon connecting to the WLAN.
2. **Payload:** `device_id`, readable name, and `is_paired` flag.

**Numeric Pairing Mechanism (PIN - ECDH):**
1. Discovery via mDNS and start of direct TCP socket.
2. Exchange of ephemeral public keys (ECDH).
3. Derivation of shared SHA-256 Hash to extract the 6-digit PIN.
4. Visual confirmation and sending of credentials encrypted with the ECDH secret.
