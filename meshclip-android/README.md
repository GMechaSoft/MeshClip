# MeshClip Android

Android client for the MeshClip decentralized P2P LAN clipboard synchronization system.

## 🚀 Overview

The Android client provides reactive, background clipboard synchronization using a hybrid network model. To bypass Android's strict background clipboard restrictions (introduced in Android 10), this client leverages the **Shizuku API** to interact with the system clipboard via a privileged process.

## 🛠 Technical Stack

- **Language:** Kotlin
- **Minimum SDK:** 29 (Android 10)
- **UI Framework:** Jetpack Compose (Material 3)
- **Privileged Access:** Shizuku API (via AIDL `IClipboardManager`)
- **Concurrency:** Kotlin Coroutines & Flow
- **Networking:**
    - **Discovery:** mDNS
    - **Light Payloads:** UDP Broadcast (Triple-burst strategy)
    - **Heavy Payloads:** TCP Streaming
- **Security:** AES-256-GCM (End-to-End Encryption)

## 📋 Prerequisites

Before building or running the app, ensure you have:
- [ ] JDK 17 or 21 (Managed via SDKMAN!)
- [ ] Android Studio (Latest stable version)
- [ ] A physical Android device (API 29+) with **Wireless Debugging** enabled.
- [ ] **Shizuku** app installed and running on the device.

## 🏗 Installation & Setup

1. **Clone the repository** (or navigate to the `meshclip-android` directory).
2. **Open in Android Studio**: Import the project as a Gradle project.
3. **Shizuku Setup**:
    - Start Shizuku on your device via ADB or Wireless Debugging.
    - Grant the app Shizuku permissions when prompted.
4. **Build**: Use the provided `./gradlew` wrapper.
    ```bash
    ./gradlew assembleDebug
    ```

## ⚙️ Architecture

- **Foreground Service:** Ensures the clipboard monitor remains active and visible to the system.
- **Reactive Engine:** Event-driven monitoring (no polling) to detect clipboard changes instantly.
- **Network Layer:** Handles the triple-burst UDP transmission (0ms, 100ms, 300ms) for high reliability in LAN environments.

## 🛡 Security

All payloads are encrypted using AES-256-GCM. Device pairing is performed via ECDH exchange triggered by QR codes or Numeric PINs.
