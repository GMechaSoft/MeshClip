# Requirements Analysis Document: LAN Clipboard Synchronization

This document summarizes the project evolution, technical decisions, and consolidates the final requirements for the development of the decentralized clipboard synchronization system.

---

## 1. Project Evolution and Architectural Decisions

The project began as an idea for a multiplatform application to synchronize clipboards on Android 15+. During the feasibility analysis, the following critical decisions were made that shaped the current architecture:

1.  **Android 10+ Restrictions and Shizuku:** A standard Android app approach was discarded due to severe background reading restrictions of the `ClipboardManager`. It was decided to use the **Shizuku** API, which can operate via **Wireless Debugging (ADB)** for standard users, or via **Root** (to allow automatic service startup after a reboot).
2.  **Zero Polling (Energy Efficiency):** To avoid draining the battery by constantly checking for new text, a **100% reactive model** was established. The system subscribes to operating system events (e.g., `OnPrimaryClipChangedListener` via Binder in Android) so that the code only wakes up when a real copy occurs.
3.  **Abandonment of .NET MAUI for Android:** To interact at such a low level with the system (Shizuku + Foreground Services), multiplatform frameworks were discarded in favor of **native Kotlin**.
4.  **Removal of Persistent History:** It was decided to simplify the system by removing global clipboard persistence. The system will operate under a reactive model (*fire-and-forget*), where each node only manages its local clipboard.
5.  **Pivot from WebSockets to UDP Broadcast:** To avoid Single Points of Failure (SPOF) and drastically reduce resource consumption by not requiring multiple live TCP connections (P2P Mesh), **UDP Broadcast** was adopted for transmitting light text.
6.  **Adoption of Hybrid Network Model:** Due to UDP size limitations, it was defined that heavy transfers (images/files) will use UDP notifications followed by **on-demand TCP** transfers, implementing a FIFO cache of the last 3 elements.
7.  **Multi-Client Strategy:** Development of clients for Android, Windows, and Linux was determined. The Linux client will feature a high-performance engine written in **Go (Golang)** to ensure efficiency and deployment without external dependencies.

---

## 2. Functional Requirements

* **FR01 - Text Listening and Injection:** The system must reactively detect (without polling) the copying of plain text and URLs, transmit them to the network, and upon receipt, inject (write) them directly into the target operating system's clipboard.
* **FR02 - File Transfer:** The system must allow copying images or heavy files. The sender will notify availability via UDP and the receiver will establish an on-demand TCP connection to download the file via streaming.
* **FR03 - Cache Management:** The sending node must maintain only the last 3 files in memory/disk. Requests for older files must be rejected.
* **FR04 - Group Filtering:** The system must allow multiple clipboard networks to coexist in the same LAN. Devices will only process messages from their own `group_id`.
* **FR05 - Optical Pairing (QR):** The system must generate and read QR codes to transfer network credentials to devices with a camera (e.g., PC to Mobile).
* **FR06 - Numeric Pairing (PIN):** For devices without a camera (e.g., PC to PC), the system must allow pairing through visual confirmation of a 6-digit PIN.
* **FR07 - Network Fault Tolerance:** UDP emissions must be sent in bursts of 3 staggered attempts. Receivers must deduplicate processed messages.
* **FR08 - Native Configuration Interface (Linux):** The system must provide a minimalist graphical user interface for Linux desktop environments. This UI will be invoked only on demand to manage paired devices, render QR codes, and handle the visual validation flow of the PIN (ECDH).
* **FR09 - Desktop Environment Integration (GNOME):** The system must include a quick access module (Applet/Extension) residing in the top panel of the desktop. It will allow the user to read the connection status of the background daemon and trigger basic actions without opening the main application.
* **FR10 - Node Discovery (mDNS):** The system must have a network announcement mechanism. When a device with the app installed connects to the local network or starts the service, it must emit a presence announcement (mDNS broadcast) indicating its availability, device name, and pairing status, allowing other nodes to detect it automatically.

---

## 3. Non-Functional Requirements

* **NFR01 - Transit Security:** All clipboard content (text or file metadata) must be encrypted end-to-end using **AES-256-GCM**.
* **NFR02 - Key Isolation:** Group credentials must never travel in plain text over the network (they are transferred via QR or temporary TCP tunnel with ECDH exchange).
* **NFR03 - Secure Storage:** Cryptographic keys must be protected using the native mechanisms of each OS (Android Keystore, Windows DPAPI to link the key to the current user account, and strict file permissions on Linux).
* **NFR04 - Performance:** The system must minimize battery and RAM consumption. Background services on desktop (Windows/Linux) must run without a main graphical interface (Headless/System Tray).
* **NFR05 - Mobile Compatibility:** The Android client must be compatible from Android 10 (API 29) onwards, discarding older versions in favor of modern OS APIs.
* **NFR06 - Interface Decoupling (D-Bus Architecture):** In the Linux client, the presentation layer (the graphical interface and the panel extension) must be strictly decoupled from the logic layer (the network Daemon). Both components must communicate in a standardized manner through the operating system bus (**D-Bus**), allowing the background engine to function autonomously even if the graphical interface fails or closes.

---

## 4. Technology Stack Definition

### 4.1 Android Client
* **Language:** Kotlin (MVVM, Coroutines).
* **UI:** Jetpack Compose.
* **System:** Foreground Service, Shizuku API (AIDL interface for background restriction bypass).
* **Network:** `DatagramSocket` (UDP) and TCP sockets (on-demand streaming).

### 4.2 Windows Client
* **Language:** C# (.NET 8+).
* **Compilation:** Standard (Non-AOT) to maintain full compatibility with the Windows UI ecosystem.
* **UI:** System Tray with WPF/WinForms.
* **System:** Win32 API (P/Invoke) on STA thread (mandatory for Windows Clipboard), `IHostedService`.
* **Network:** `UdpClient`, TCP sockets (on-demand streaming).

### 4.3 Linux Client
* **Frontend (UI):** GTK4 + libadwaita (written in Python). GNOME Extension (written in GJS).
* **Internal Communication:** D-Bus.
* **Engine (Daemon):** **Go (Golang)**. Natively compiled static binary. Use of *Goroutines* for concurrency, `os/exec` package (`wl-clipboard` for Wayland or `xclip` for X11) and `godbus/dbus` for IPC. Replaces Python in the core to eliminate external dependencies and versioning issues.

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
1.  **Announcement:** Passive emission (Broadcast) in plain text upon connecting to the WLAN.
2.  **Payload:** Hardware identifier (`device_id`), readable name (e.g., "Juan's Laptop") and status flag (`is_paired`).

**Optical Pairing Mechanism (QR):**
1.  The current device encodes the `master_key` and `group_id` into a structured string (JSON or URI).
2.  The Graphical Interface renders the string as a QR Code.
3.  The new device scans the QR (Out-of-Band channel).
4.  Credentials are saved in secure storage and the node joins the UDP listening network.

**Numeric Pairing Mechanism (PIN - ECDH):**
1.  Peer discovery via mDNS.
2.  Opening of local TCP socket.
3.  Exchange of ephemeral public keys (Elliptic Curve Diffie-Hellman).
4.  Derivation of shared SHA-256 Hash -> Extraction of 6-digit PIN.
5.  Visual confirmation by the user on both ends (MitM attack mitigation).
6.  Transmission of `group_id` and `master_key` encrypted with the ECDH secret.
7.  Closing of the TCP socket and start of UDP synchronization.

**Internal Communication Mechanism (IPC) in Linux:**
* The background Engine (**Go**) must register the service in the D-Bus session (e.g., `org.portapapeles.Service`).
* It must expose invocable methods (e.g., `GetStatus()`, `StartPairing()`) and emit signals (e.g., `OnNewTextCopied`, `OnDeviceConnected`) so that the GTK4 interface and the GJS extension can react in real-time.
