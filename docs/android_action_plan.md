# Action Plan: Android Client Implementation (Hexagonal Architecture & DDD)

## Context
Implementation of a decentralized P2P LAN clipboard synchronization client for Android. The system uses a high-privilege background access model via Shizuku API to bypass Android 10+ (API 29) restrictions, organized under a Hexagonal Architecture to ensure testability and maintainability.

## Architectural Blueprint
- **Domain Layer:** Pure Kotlin. Contains Entities, Use Cases, and Ports. No Android dependencies.
- **Infrastructure Layer:** Adapters implementing Domain Ports (Shizuku, UDP/TCP Sockets, Keystore).
- **Presentation Layer:** Jetpack Compose UI and ViewModels invoking Domain Use Cases.
- **Dependency Flow:** Presentation $\rightarrow$ Domain $\leftarrow$ Infrastructure.

## Implementation Strategy

### Phase 1: Domain Definition (The Core)
- [ ] **Define Entities:** Implement `ClipboardPayload`, `Device`, and `SyncSession` value objects.
- [ ] **Define Ports:** Create interfaces for `ClipboardPort`, `NetworkPort`, and `SecurityPort`.
- [ ] **Implement Use Cases:**
    - `SyncClipboardUseCase`: Logic for handling clipboard changes and triggering network emission.
    - `PairDeviceUseCase`: Logic for device discovery and credential exchange.
    - `DeduplicationService`: Pure logic to prevent processing duplicate UDP packets.

### Phase 2: Infrastructure Adapters (The Implementation)
- [ ] **Clipboard Adapter:**
    - Setup `IClipboardManager.aidl`.
    - Implement `ShizukuClipboardAdapter` implementing `ClipboardPort`.
    - Implement reactive listener via `OnPrimaryClipChangedListener`.
- [ ] **Network Adapter:**
    - Implement `UdpNetworkAdapter` with the triple-burst strategy (0ms, 100ms, 300ms).
    - Implement `TcpStreamAdapter` for binary file streaming.
    - Implement mDNS discovery for node announcement.
- [ ] **Security Adapter:**
    - Implement `AndroidKeyStoreAdapter` implementing `SecurityPort` for AES-256-GCM.
- [ ] **Persistence Adapter:**
    - Implement `DeviceRepository` for storing paired devices and group IDs.

### Phase 3: Presentation Layer & Integration
- [ ] **DI Setup:** Configure Hilt/Koin to bind Ports to their respective Adapters.
- [ ] **ViewModels:** Implement `SyncViewModel` and `PairingViewModel` to bridge Domain use cases to the UI.
- [ ] **UI Screens:** Build Pairing, Settings, and Status screens using Jetpack Compose.
- [ ] **Foreground Service:** Implement `ClipboardService` as the primary entry point that initializes the infrastructure and Domain use cases.

### Phase 4: Validation & Stress Testing
- [ ] **Unit Testing:** Validate Domain use cases and deduplication logic via JVM tests.
- [ ] **Integration Testing:** Verify Shizuku permissions and background clipboard access.
- [ ] **Network Validation:** Use Wireshark to verify AES-256-GCM and triple-burst UDP.
- [ ] **Performance Testing:** Transfer files >100MB to verify TCP streaming efficiency.

## Critical Package Structure
- `com.gmechasoft.meshclip.domain.model`
- `com.gmechasoft.meshclip.domain.ports`
- `com.gmechasoft.meshclip.domain.usecase`
- `com.gmechasoft.meshclip.infrastructure.clipboard`
- `com.gmechasoft.meshclip.infrastructure.network`
- `com.gmechasoft.meshclip.infrastructure.security`
- `com.gmechasoft.meshclip.presentation.ui`
- `com.gmechasoft.meshclip.presentation.viewmodel`

## Verification Plan
1. **Architectural Audit:** Ensure no Android/Third-party leaks in the `domain` package.
2. **Functional Test:** Text sync across Android $\leftrightarrow$ Windows/Linux.
3. **Security Audit:** Confirm zero plaintext transmission of credentials.
4. **Stability Test:** Verify service persistence under OS memory pressure.
