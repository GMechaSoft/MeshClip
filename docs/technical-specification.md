# Technical Specification: Decentralized Clipboard Synchronization System (LAN)

## 1. General Description
Multiplatform system for real-time synchronization of clipboard events within a local area network (WLAN). It employs a distributed architecture (Peer-to-Peer, without a central server) and a hybrid network model (UDP/TCP) to optimize performance.

**Operating Principle:** The system is **100% reactive (Zero Polling)**; it does not perform periodic queries but subscribes to operating system events to wake up only when a copy occurs. Data travels with end-to-end encryption (AES-256-GCM). There is no global persistence: each node manages its own clipboard and emits transient events.

---

## 2. Client Architecture (By Operating System)

### 2.1 Android Client (Android 10+ / API 29+)
* **Architecture:** Hexagonal Architecture (Ports & Adapters) with DDD.
* **Language and UI:** Kotlin / Jetpack Compose. MVVM Pattern.
* **Layering:**
    - **Domain:** Pure Kotlin logic, Entities, Use Cases, and Ports.
    - **Infrastructure:** Adapters for Shizuku (Clipboard), Sockets (Network), and Keystore (Security).
    - **Presentation:** Jetpack Compose UI and ViewModels.
* **Background and Concurrency:** `Foreground Service` (Mandatory persistent notification). `Kotlin Coroutines` for asynchronous network handling.
* **Network Layer:** `java.net.DatagramSocket` (UDP) and TCP sockets (on-demand streaming).
* **Core / Clipboard:** **Shizuku** API. Compatible with **Wireless ADB** and **Root**. Use of AIDL interfaces (`IClipboardManager`) to register listeners and execute background read/writes in pure reactive mode.
* **Storage:** Android Keystore System.

### 2.2 Windows Client
* **Language and Execution:** C# (.NET 8+). To maintain full compatibility with the Windows UI ecosystem.
* **Interface (UI):** System Tray application built with WPF or WinForms.
* **Background:** Embedded `IHostedService` (Worker).
* **Network Layer:** Native `UdpClient` and TCP sockets (on-demand streaming).
* **Core / Clipboard:** P/Invoke calls (`User32.dll`) to the Win32 API. All clipboard operations are delegated to an STA (Single-Threaded Apartment) thread.
* **Storage:** Windows DPAPI (`ProtectedData.Protect` via `crypt32.dll`) linked to the user account.

### 2.3 Linux Client
The Linux client is divided into a high-performance engine and a modern graphical interface:

* **Engine (Daemon):** Written in **Go (Golang)**. Compiled as a native static binary, independent and without external dependencies. Uses *Goroutines* for concurrency, the `os/exec` package (`wl-clipboard` for Wayland or `xclip` for X11) to interact with the clipboard, and `godbus/dbus` for IPC communication.
* **Interface (Frontend):**
    1.  **GUI Frontend:** Native GTK4 + libadwaita application (written in Python) that communicates with the Daemon via D-Bus to render QR codes and process pairings.
    2.  **GNOME Extension:** GJS (JavaScript) module in the top panel for quick status reading (D-Bus) and notifications (`libnotify`).

---

## 3. Network Architecture (Hybrid Model)

### 3.1 Light Loads (Text and URLs)
* **Protocol:** UDP Broadcast.
* **Mechanism:** The packet travels encrypted to the broadcast IP address (e.g., `192.168.1.255`). Nodes intercept the packet, decrypt it, and inject the text.

### 3.2 Heavy Loads (Images and Files)
* **Protocol:** UDP Notification + On-demand TCP Transfer.
* **Mechanism:** 1. The sender emits a UDP notice (`type: file_notice`) with its TCP port.
    2. Receivers establish a direct TCP socket connection to download the file via streaming.
* **Retention Policy:** Senders keep only the **last 3 transmitted files** in cache. Requests for purged IDs are rejected.

### 3.3 Fault Tolerance
* **Burst:** Triple emission for each event (T=0ms, 100ms, 300ms) to mitigate the unreliable nature of UDP.
* **Deduplication:** Receivers store recent IDs. Packets with a duplicate `id` are discarded at the network layer.

---

## 4. Security, Pairing, and Internal Communication

* **Logical Isolation:** UDP packets include a plain text `group_id` for early filtering.
* **Encryption:** AES-256-GCM. Protects the `payload` (text or file metadata) and ensures packet integrity.
* **JSON Packet Structure:** Contains: version, `group_id`, message `id`, `type` (text/file_notice), `tcp_port`, `iv` (Base64) and `payload` (Base64).

### 4.1 Discovery Mechanism (mDNS)
Before any key exchange can occur, devices must be aware of their "neighbors" on the local network.

* **Presence Announcement:** Upon starting the service or connecting to a new Wi-Fi network, the node uses the mDNS (Multicast DNS) protocol to announce its presence passively.
* **Exposed Data:** The announcement is in plain text and contains only non-sensitive public information: unique hardware identifier (device_id), readable name (e.g., "Juan's Laptop") and a boolean flag indicating if it already belongs to a group (is_paired).
* **Detection:** User interfaces (Android UI, GTK4, GNOME Extension) consume this discovery service to populate the "Available Devices" list when the user wishes to start a pairing process.

### 4.2 Pairing Flows (Key Distribution)
1.  **New Group:** Local random generation of `group_id` and `master_key` on the first node.
2.  **Optical (QR):**
    * The current device encodes the `master_key` and `group_id` into a structured string (JSON or URI).
    * The Graphical Interface renders the string as a QR Code.
    * The new device scans the QR (Out-of-Band channel).
    * Credentials are saved in the secure store and the node joins the UDP listening network.
3.  **Numeric (PIN - ECDH):** Used between desktop clients.
    * Local discovery via mDNS.
    * Establishment of a direct TCP socket.
    * Exchange of ephemeral public keys (ECDH).
    * Visual confirmation of a 6-digit PIN on both screens to mitigate MitM attacks.
    * Sending of credentials encrypted with the ECDH secret and subsequent closing of the TCP socket.

### 4.3 Internal Communication Protocol (IPC on Linux)
To ensure decoupling between the graphical interface and the background engine on Linux, the operating system bus is used:
* **Mechanism:** Registration of the service in the D-Bus session (e.g., `org.portapapeles.Service`).
* **Exposed Methods:** Invocable by the UI and the Extension (e.g., `GetStatus()`, `StartPairing()`, `GenerateQR()`).
* **Emitted Signals:** Real-time event transmission to update the UI (e.g., `OnNewTextCopied`, `OnDeviceConnected`, `OnPairingFailed`).

---

## 5. Specific Interface and Integration Requirements (Linux)

To meet usability standards in desktop environments without compromising the performance of the network engine, the following formal requirements are established:

* **FR08 - Native Configuration Interface:** The system must provide a minimalist UI invocable on demand. Its sole responsibility is to manage paired devices, render QR codes, and handle the visual validation flow of the PIN (ECDH).
* **FR09 - Environment Integration (GNOME):** The system must include an Applet/Extension in the top panel of the desktop to read the connection status of the daemon and trigger basic actions (e.g., force synchronization) without opening the main application.
* **NFR06 - Architectural Decoupling:** The presentation layer (GTK4 UI and GJS Extension) must be strictly decoupled from the logic layer (the **Go** Daemon). The background engine must continue synchronizing the clipboard autonomously even if the graphical interface processes close or fail.

---

## 6. User Stories

**Main Epic:** Transparent clipboard synchronization on LAN without cloud dependency, supporting hybrid transfer (text/files) and decentralized pairing.

### US 01: Silent Synchronization on Android
**As a** user of an Android 10+ device,
**I want** the service to read and write to my clipboard from the background,
**So that** I can synchronize text with my other devices without having to keep the application open on screen.

* **Acceptance Criteria:**
    * The app must validate Shizuku permissions on startup.
    * It must register an `OnPrimaryClipChangedListener` through the system interface (AIDL).
    * A foreground service must keep the UDP socket listening and emit the 3-packet burst upon detecting a local copy.

### US 02: Windows Client - Efficiency and Heavy Transfer
**As a** Windows user,
**I want** to send files or images from my explorer to the network and keep the application in the system tray,
**So that** I can transfer heavy resources without impacting my PC's performance or consuming excessive memory.

* **Acceptance Criteria:**
    * The executable must be compiled in .NET and run efficiently in the background.
    * Visual interaction must be limited to the system tray.
    * When copying a file, the system will notify via UDP and establish an on-demand TCP connection for the transfer, removing the file from temporary memory if it exceeds the 3-item limit.
    * The AES key must be stored encrypted using DPAPI.

### US 03: Linux Client - Desktop Integration and D-Bus
**As a** user of a Linux environment (GNOME/GTK),
**I want** to manage pairing (QR/PIN) and see the network status from my native interface or top panel,
**So that** I can interact with the synchronization engine intuitively and adapted to the graphical ecosystem.

* **Acceptance Criteria:**
    * The synchronization service (Daemon written in Go) must start with the user (Systemd) and interact silently with `wl-clipboard`/`xclip`.
    * The Daemon must register its API on D-Bus.
    * A GTK4 interface (Python) must be able to request the Daemon to generate the string for the QR or start the ECDH PIN validation.
    * A GNOME extension must read the Daemon's status via D-Bus and emit native system notifications.
