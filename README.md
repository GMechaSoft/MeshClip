# Shared Clipboard P2P

A decentralized, high-performance LAN clipboard synchronization system. This project allows seamless sharing of text, URLs, and files across Android, Windows, and Linux devices within the same network without relying on a central server.

## 🚀 Features

- **Cross-Platform Sync:** Native clients for Android, Windows, and Linux.
- **Zero-Polling Architecture:** Event-driven clipboard monitoring for maximum battery efficiency.
- **Hybrid Network Model:** 
  - **mDNS Discovery:** Passive node announcement.
  - **UDP Broadcast:** High-speed delivery for light payloads (Text/URLs) using a triple-burst strategy.
  - **TCP Streaming:** On-demand transfer for heavy payloads (Images/Files).
- **Military-Grade Security:**
  - End-to-end encryption using **AES-256-GCM**.
  - Secure pairing via **QR Codes** or **Numeric PIN (ECDH exchange)**.
  - Key storage utilizing platform-specific secure enclaves (e.g., Android Keystore).

## 🏗️ Technical Stack

| Platform | Language/Framework | Core Mechanism |
| :--- | :--- | :--- |
| **Android** | Kotlin / Jetpack Compose | Foreground Service + Shizuku API (AIDL) |
| **Windows** | C# / .NET 8+ | Win32 API (STA Thread) |
| **Linux** | Go (Engine) / Python (UI) | D-Bus IPC + GTK4/libadwaita |

## 📁 Project Structure

```text
shared-clipboard/
├── docs/               # Design specifications, Action Plans, and Architecture ADRs
├── android/            # Android client source code (Kotlin)
├── windows/            # Windows client source code (C#)
├── linux/              # Linux daemon (Go) and UI (Python/GJS)
└── README.md            # Project overview
```

## 🛠️ Development Setup

Detailed setup guides are available in the `docs/` directory. 

### General Requirements
- **Versioned Toolchains:** To ensure environment reproducibility, use:
  - `SDKMAN!` for JDK and Kotlin.
  - `nvm` for Node.js.
  - `pyenv`/`venv` for Python.
- **Network:** A local network supporting mDNS and UDP Broadcast.

## 🛡️ Security Model

1. **Pairing:** Devices must be paired out-of-band (QR/PIN) to exchange a `group_id` and `master_key`.
2. **Encryption:** All data is encrypted before transmission. Only paired devices sharing the same `group_id` can decrypt the traffic.
3. **Isolation:** No data is ever stored on external clouds or servers.
