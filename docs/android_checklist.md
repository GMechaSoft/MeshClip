# Android App Development Checklist

## 1. Development Environment (Versioned & Isolated)
- [ ] **JDK Management:** Install [SDKMAN!](https://sdkman.io/) to manage JDK versions (Recommended: JDK 17 or 21).
- [ ] **Kotlin Management:** Use SDKMAN! to manage Kotlin compiler versions.
- [ ] **Node.js Management:** Install [nvm](https://github.com/nvm-sh/nvm) for any tooling/utility scripts.
- [ ] **Python Environment:** Use `pyenv` or `venv` for any auxiliary automation scripts to avoid contaminating Ubuntu 24 system Python.
- [ ] **IDE:** Android Studio (installed via JetBrains Toolbox for isolated version management).
- [ ] **Gradle:** Use the project's `gradlew` wrapper exclusively.

## 2. Hardware & Device Requirements
- [ ] **Physical Android Device:** API 29 (Android 10) or higher.
- [ ] **Wireless Debugging:** Enabled on the device for Shizuku access.
- [ ] **USB Connectivity:** Validated data cable for initial ADB setup.

## 3. Technical Dependencies & Frameworks
- [ ] **Architecture Validation:** Project structure implements Domain, Infrastructure, and Presentation layers (Hexagonal/DDD).
- [ ] **Shizuku API:** Added to `build.gradle` and properly linked.
- [ ] **AIDL Definition:** `IClipboardManager.aidl` implemented and compiled.
- [ ] **UI Framework:** Jetpack Compose with Material 3.
- [ ] **Asynchrony:** Kotlin Coroutines and Flow for non-blocking I/O.
- [ ] **Security:** Android Keystore System for AES key storage.

## 4. Network & Diagnostic Tools
- [ ] **Packet Analysis:** Wireshark or tcpdump installed on Ubuntu 24.
- [ ] **ADB (Android Debug Bridge):** Installed and configured for `logcat` monitoring.
- [ ] **mDNS/UDP Environment:** Local network with UDP Broadcast and mDNS enabled.
- [ ] **QR Utility:** Tool for generating/scanning pairing codes.

## 5. Validation Criteria
- [ ] **Architectural Integrity:** No leak of infrastructure dependencies (Shizuku/Sockets) into the Domain layer.
- [ ] **Shizuku Permission:** App can successfully request and obtain Shizuku permissions.
- [ ] **Background Access:** Ability to detect clipboard changes while the app is not in the foreground.
- [ ] **Network Integrity:** Verified 3-burst UDP transmission (0ms, 100ms, 300ms).
- [ ] **Encryption:** Verified AES-256-GCM payload encryption via packet capture.
- [ ] **Memory Efficiency:** TCP file streaming does not spike RAM usage.
