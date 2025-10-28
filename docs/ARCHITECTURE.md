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

## 2. Static Architecture (Container Diagram)

This diagram identifies the core components (containers) within the system and their logical responsibilities using the C4 Model notation.

![Статическая Архитектура Foggy Call](foggy-call-architecture-c4.svg)

```plantuml
@startuml Architecture_Static_Diagram_Updated
!include [https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml](https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml)

LAYOUT_WITH_LEGEND()

System_Boundary(C4_Sys, "Foggy Call System (Private P2P Voice/Video)") {
    
    Container(AndroidApp, "Android Mobile Application", "Kotlin / WebRTC Android SDK", "**Peer-Terminal**. Manages key cryptography, signaling, and media streams. Initiates and receives calls.")
    
    Container(GoServer, "Signaling Server (Relay)", "Go / WebSocket Server (WSS) / VPS", "Stateless, log-free switchboard. Relays transient SDP Offer/Answer and ICE candidates.")
}

System_Ext(STUNServer, "STUN Server (Public)", "UDP", "Identifies the client's public IP/port for NAT Traversal.")
System_Ext(Internet, "The Internet", "TCP/UDP", "Provides routing and connectivity between all nodes.")
Rel(AndroidApp, GoServer, "1. WebSocket Secure (WSS)", "Asynchronous exchange of signaling messages (JSON)")
Rel(AndroidApp, STUNServer, "2. UDP (STUN)", "Binding Request for external address resolution")
Rel(AndroidApp, AndroidApp, "3. Direct P2P Traffic", "Encrypted Audio/Video (SRTP/DTLS)")

Rel_U(GoServer, Internet, "Hosting/Connectivity", "WSS Port 443")
Rel_U(AndroidApp, Internet, "Network Access")

@enduml
```

| Component | Responsibility | Technology |
| :--- | :--- | :--- |
| Android Mobile Application | "Manages the full lifecycle of a call, from key generation and registration to media transmission.  | Must maintain an active WSS connection (Ready State) via an Android Service.","Kotlin, WebRTC Android SDK, Android Keystore" |
| Go Signaling Server | Acts as a temporary lookup service (Public Key -> WSS Connection).  | Its only task is to route signaling messages between registered peers.,"Go, WebSockets" |
| STUN Server | Essential for WebRTC to overcome basic NATs and firewalls by discovering the client's public network identity.  | UDP Protocol |

---

## 3. Dynamic Architecture (Sequence Diagram)

This diagram illustrates the step-by-step process required for App A to establish a secure P2P connection with App B.

![Динамическая Архитектура Foggy Call](foggy-call-architecture-sequence.svg)

```plantuml
@startuml VideoCall_Full_Sequence_Simplified_Internal
title Dynamic Call Flow: Establishing P2P Connection

participant AppA as "Android App A (Caller)"
participant AppB as "Android App B (Receiver)"
participant GoServer as "Go Server (WSS)"
participant STUN as "STUN Server (Public)"

autonumber

group Phase 1: Registration and Discovery (Ready State)
    note over AppA: AppA is launched
    AppA ->> GoServer: [async_rqst] Establish WSS Connection
    GoServer ->> AppA: [async_resp] Connection Established

    AppA ->> GoServer: [async_rqst] Register Public Key A
    note right of GoServer: Store: Map[Key A] = WSS-Connect A

    note over AppB: AppB is launched (in background Service)
    AppB ->> GoServer: [async_rqst] Establish WSS Connection
    GoServer ->> AppB: [async_resp] Connection Established

    AppB ->> GoServer: [async_rqst] Register Public Key B
    note right of GoServer: Store: Map[Key B] = WSS-Connect B

    note over AppA: Address Discovery (Pre-call)
    AppA ->> STUN: [async_rqst] Binding Request for Public IP
    STUN ->> AppA: [async_resp] Public IP and Port (Candidate A)
end

group Phase 2: Signaling (SDP Exchange)
    note over AppA: User A presses "Call"
    AppA -> AppA: Create SDP Offer
    
    AppA ->> GoServer: [async_rqst] JSON: {target_key: Key B, payload: SDP Offer}
    note right of GoServer: Relay to Key B
    GoServer ->> AppB: [async_rqst] Forward SDP Offer
    
    note over AppB: AppB receives incoming call
    AppB -> AppB: Set Remote Description (Offer)
    AppB -> AppB: Generate SDP Answer
    
    AppB ->> GoServer: [async_rqst] JSON: {target_key: Key A, payload: SDP Answer}
    GoServer ->> AppA: [async_rqst] Forward SDP Answer
    
    AppA -> AppA: Set Remote Description (Answer)
    note over AppA, AppB: ICE candidates continue exchanging via Go Server
end

group Phase 3: P2P Tunnel and Media Flow
    note over AppA, AppB: WebRTC completes ICE checks (Success)
    AppA ->> AppB: [async_rqst] Establish Direct P2P Tunnel (DTLS/SRTP)
    
    note over AppA, AppB: Media traffic is now transmitted directly
    AppA ->> AppB: [async_rqst] Encrypted Audio/Video
    AppB ->> AppA: [async_resp] Encrypted Audio/Video
    
    note over AppA, AppB: Call End
    AppA ->> GoServer: [async_rqst] Close WSS (Unregister)
    AppB ->> GoServer: [async_rqst] Close WSS (Unregister)
end
@enduml
```
