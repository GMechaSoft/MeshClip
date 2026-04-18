# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

The project is a decentralized, P2P LAN clipboard synchronization system with a hybrid network model.

### Core Components
- **Android Client:** Native Kotlin using `Foreground Service` and the **Shizuku API** (via AIDL `IClipboardManager`) for reactive background clipboard access.
- **Windows Client:** C# (.NET 8+) using Win32 API P/Invoke calls on a **Single-Threaded Apartment (STA)** thread for clipboard interaction.
- **Linux Client:** 
    - **Engine (Daemon):** High-performance static binary written in **Go (Golang)**.
    - **Interface:** GTK4/libadwaita (Python) and GNOME Extension (GJS).
    - **IPC:** Communication between the Go daemon and UI layers is handled via **D-Bus**.

### Network Model
- **Discovery:** Uses **mDNS** (Multicast DNS) for passive node announcement.
- **Light Payloads (Text/URLs):** Transmitted via **UDP Broadcast** with a triple-burst strategy for fault tolerance.
- **Heavy Payloads (Files/Images):** Triggered by a UDP notice, then transferred via **on-demand TCP streaming**.
- **Security:** End-to-end encryption using **AES-256-GCM**. Pairing is handled via **QR codes** or **Numeric PIN (ECDH exchange)**.

## Development Guidelines

### Project Structure
The repository currently contains the design and specification phase in the `docs/` directory. Implementation follows the platform-specific stacks (Kotlin, C#, Go).

### Technical Constraints
- **Reactive Mode:** No polling is allowed. All clipboard monitoring must be event-driven.
- **Android Support:** Minimum SDK 29 (Android 10+).
- **Linux Integration:** The daemon must remain autonomous from the GUI; UI failures should not stop synchronization.
