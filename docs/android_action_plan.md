# Action Plan: Android Client Implementation

## Context
Implementation of a decentralized P2P LAN clipboard synchronization client for Android. The system requires high-privilege background access to the clipboard via Shizuku API to bypass Android 10+ (API 29) background restrictions.

## Architecture
- **Pattern:** MVVM (Model-View-ViewModel).
- **UI:** Jetpack Compose.
- **Concurrency:** Kotlin Coroutines & Flow.
- **Persistence:** Foreground Service for network listening and clipboard monitoring.

## Implementation Strategy

### Phase 1: Privilege & System Integration
- Setup Shizuku API and `IClipboardManager.aidl` interface.
- Implement permission request flow (`Shizuku.checkSelfPermission`).
- Develop `ClipboardAidlHelper` to bridge the app with the system clipboard binder.
- **Constraint:** Zero-polling policy; use `OnPrimaryClipChangedListener`.

### Phase 2: Background Persistence
- Implement `ClipboardService` as a `Foreground Service`.
- Configure persistent notification to prevent OS process killing.
- Integrate the service lifecycle with Shizuku's binder management.

### Phase 3: Networking Engine (Hybrid Model)
- **UDP Engine:** Implement `DatagramSocket` with a triple-burst strategy (0ms, 100ms, 300ms) for text/URLs.
- **Deduplication:** Implement a message ID cache to discard redundant UDP packets.
- **mDNS Discovery:** Implement passive node announcement for P2P discovery.
- **TCP Streaming:** Implement on-demand TCP sockets for binary file transfer using streams to minimize RAM overhead.

### Phase 4: Security & Encryption
- **Cryption:** Implement AES-256-GCM for end-to-end encryption of all payloads.
- **Key Storage:** Use Android Keystore System to securely store the `master_key`.
- **Pairing Flow:**
    - QR-based exchange of `group_id` and keys.
    - ECDH (Elliptic Curve Diffie-Hellman) exchange with 6-digit PIN validation.

### Phase 5: UI & Final Integration
- Build configuration and pairing screens using Jetpack Compose.
- Implement state management in ViewModels for connection status.
- End-to-end testing with Windows and Linux clients.

## Critical Files to Create
- `service/ClipboardService.kt`: Foreground service core.
- `data/shizuku/ClipboardAidlHelper.kt`: Shizuku/AIDL bridge.
- `data/network/UdpNetworkManager.kt`: UDP burst logic.
- `data/network/TcpFileStreamer.kt`: Binary stream handler.
- `data/local/KeyStoreManager.kt`: Secure key management.

## Verification Plan
1. **Permission Test:** Confirm background clipboard access via Shizuku.
2. **Network Test:** Verify text synchronization across different OS platforms.
3. **Security Test:** Use Wireshark to ensure no plaintext data is transmitted.
4. **Stress Test:** Transfer files >100MB to verify streaming efficiency.
