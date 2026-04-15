# Architecture Decision Record (ADR): Android Client Architecture

## Status
Proposed

## Context
The Android client requires a high level of system integration (Shizuku, Foreground Services) and network complexity (Hybrid UDP/TCP), while needing to remain maintainable, testable, and decoupled from volatile OS APIs. The previous plan followed a linear implementation strategy, which risked coupling business logic with infrastructure details.

## Decision
Implement the Android client using **Hexagonal Architecture (Ports and Adapters)** combined with **Domain-Driven Design (DDD)** principles.

### Architectural Layers

#### 1. Domain Layer (The Core)
- **Responsibilities:** Pure business logic, entities, and use cases.
- **Dependencies:** Zero dependencies on Android Framework or third-party libraries.
- **Key Components:**
    - **Entities:** `ClipboardPayload`, `Device`, `SyncSession`.
    - **Use Cases (Interactors):** `SyncClipboardUseCase`, `PairDeviceUseCase`, `EncryptPayloadUseCase`.
    - **Ports (Interfaces):** `ClipboardPort`, `NetworkPort`, `SecurityPort`.

#### 2. Infrastructure Layer (Adapters)
- **Responsibilities:** Implementation of the Ports defined in the Domain.
- **Dependencies:** Android SDK, Shizuku API, Java Networking.
- **Key Components:**
    - **Clipboard Adapter:** Implementation using `IClipboardManager.aidl` and Shizuku Binder.
    - **Network Adapter:** Implementation of UDP Triple-Burst and TCP Streaming.
    - **Security Adapter:** AES-256-GCM implementation using `Android Keystore`.

#### 3. Presentation Layer (User Interface)
- **Responsibilities:** Rendering state and capturing user intent.
- **Dependencies:** Jetpack Compose, ViewModel.
- **Pattern:** MVVM (Model-View-ViewModel), where ViewModels invoke Domain Use Cases.

## Consequences
- **Pros:**
    - **Testability:** Business logic can be tested via JVM JUnit tests without Android devices.
    - **Flexibility:** The clipboard access mechanism (Shizuku) can be replaced without touching the sync logic.
    - **Maintainability:** Clear separation of concerns prevents "God Classes" (e.g., a Service that does networking, encryption, and UI updates).
- **Cons:**
    - **Boilerplate:** Increased number of files due to the separation of interfaces and implementations.
    - **Complexity:** Slightly steeper learning curve for developers unfamiliar with Clean Architecture.

## Alignment with SOLID
- **S:** Each class has one reason to change (e.g., `UdpNetworkAdapter` only changes if the network protocol changes).
- **O:** New transport protocols can be added by implementing `NetworkPort`.
- **L:** All adapters are interchangeable via their respective ports.
- **I:** Interfaces are segregated by functionality (Clipboard vs Network vs Security).
- **D:** High-level Use Cases depend on abstractions (Ports), not low-level implementations.
