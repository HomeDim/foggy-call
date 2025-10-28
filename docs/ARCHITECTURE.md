# 🏗️ Foggy Call Architecture

This document details the static and dynamic structure of the **Foggy Call** system. The architecture is designed to enforce maximum user privacy by minimizing the server's role in the communication path, aligning with our 'traceless' design philosophy.

## 1. Architectural Goals and Constraints

| Goal / Constraint | Description | Status |
| :--- | :--- | :--- |
| **P1: Privacy by Design** | No logging or persistent storage of communication metadata on the signaling server. | **Mandatory** |
| **P2: End-to-End Encryption** | All media streams must be encrypted (DTLS/SRTP). | **Mandatory** |
| **C1: Serverless Media** | The signaling server **must not** relay media traffic (no TURN). This prioritizes privacy over 100% connection reliability. | **Mandatory Trade-off** |
| **C2: Server Role** | The Go Server must be **stateless** and act only as a transient message switchboard. | **Mandatory** |
| **G1: Peer-to-Peer Focus** | Prioritize direct P2P connection attempts exclusively. | **Mandatory** |

---

## 2. Static Architecture (Mermaid Graph)

This diagram identifies the core components (containers) within the system and their logical responsibilities.

```mermaid
graph LR
    subgraph System Boundary: Foggy Call
        A[Android App (Peer-Terminal)]
        B[Go Server (Signaling Relay)]
    end
    
    C[STUN Server (Public)]
    D[The Internet]
    
    %% Connections
    A -- 1. WSS (Signaling) --> B
    B --> A
    
    A -- 2. UDP (Binding Request) --> C
    C --> A
    
    A -- 3. P2P Traffic (DTLS/SRTP) --> A
    
    %% Context
    D -- Host, Route --> B
    D -- Access --> A
````

### Component Details

| Component | Responsibility | Technology |
| :--- | :--- | :--- |
| **Android Mobile Application** | Manages the full lifecycle of a call, from key generation and registration to media transmission. Must maintain an **active WSS connection** (Ready State) via an Android Service. | Kotlin, WebRTC Android SDK, Android Keystore |
| **Go Signaling Server** | Acts as a temporary lookup service (`Public Key -> WSS Connection`). Its only task is to route signaling messages between registered peers. | Go, WebSockets |
| **STUN Server** | Essential for WebRTC to overcome basic NATs and firewalls by discovering the client's public network identity. | UDP Protocol |

-----

## 3\. Dynamic Architecture (Mermaid Sequence Diagram)

This diagram illustrates the step-by-step process required for App A to establish a secure P2P connection with App B.

```mermaid
sequenceDiagram
    participant AppA as Android App A (Caller)
    participant AppB as Android App B (Receiver)
    participant GoServer as Go Server (WSS Relay)
    participant STUN as STUN Server
    
    Note over AppA, AppB: Phase 1: Registration and Address Discovery
    
    AppA->>GoServer: Establish WSS Connection & Register PubKey A
    AppB->>GoServer: Establish WSS Connection & Register PubKey B
    
    AppA->>STUN: UDP Binding Request
    STUN-->>AppA: Public IP/Port (Candidate A)

    Note over AppA, AppB: Phase 2: Signaling (SDP Exchange)

    AppA->>AppA: Create SDP Offer
    AppA->>GoServer: JSON: {target: Key B, payload: Offer}
    GoServer->>AppB: Forward SDP Offer
    
    AppB->>AppB: Set Remote Description (Offer)
    AppB->>AppB: Generate SDP Answer

    AppB->>GoServer: JSON: {target: Key A, payload: Answer}
    GoServer->>AppA: Forward SDP Answer
    
    AppA->>AppA: Set Remote Description (Answer)
    
    Note over AppA, AppB: ICE candidates continue exchanging via Go Server

    Note over AppA, AppB: Phase 3: P2P Tunnel and Media Flow

    AppA->>AppB: Establish Direct P2P Tunnel (DTLS/SRTP)
    
    AppA-->>AppB: Encrypted Audio/Video (P2P)
    AppB-->>AppA: Encrypted Audio/Video (P2P)
    
    AppA->>GoServer: Close WSS (Unregister)
    AppB->>GoServer: Close WSS (Unregister)
```
