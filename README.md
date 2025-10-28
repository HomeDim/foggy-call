# 🌫️ FogGY CALL

**Foggy Call** is an open-source, peer-to-peer (P2P) video and audio communication application built with a core focus on **absolute user privacy and metadata minimization.**

The project's architectural design deliberately minimizes server footprint, ensuring that your communication traffic leaves no trace on any centralized infrastructure.

**Metaphor:** Like fog that dissipates quickly, communication traffic flows directly and transiently between participants, ensuring no centralized records of the media stream are kept.

---

## 🌟 Core Privacy Principles

Our commitment to privacy dictates the following architectural choices:

1.  **P2P Communication:** All media traffic (audio/video) is transmitted **directly** between peer devices using WebRTC's secure channel (SRTP/DTLS).
2.  **Serverless Signaling (Stateless):** The Go Server acts purely as a **temporary switchboard** for initial signaling messages (SDP Offer/Answer). It is designed to be **stateless, log-free, and transient.**
3.  **Cryptographic ID:** User identification is based on **Public Keys**, eliminating the need for traditional usernames, passwords, or phone numbers.
4.  **No TURN Relay:** We **consciously exclude the use of TURN servers** for media relaying. This choice ensures maximum privacy by preventing the collection of any media metadata (traffic volume, duration, precise routing) on a third-party or self-hosted server.
    * *Trade-off:* This compromises connection reliability in the presence of strict, symmetric NATs (common in corporate or university networks).

---

## 🛠️ Technology Stack

| Component | Technology | Role |
| :--- | :--- | :--- |
| **Client Application** | **Kotlin / Android SDK** | Peer-Terminal, handling key generation (Android Keystore), WebRTC sessions, and media streams. |
| **Signaling Server** | **Go (WebSocket Server)** | Minimalist, high-performance, stateless relay for signaling data. |
| **Network Traversal** | **WebRTC + STUN** | Relies solely on STUN for public address resolution (NAT Traversal). |
| **Encryption** | **DTLS / SRTP** | End-to-end encryption for all media streams. |

---

## 🏗️ Architecture and Data Flow

The system comprises equal peer-terminals connected via a transient signaling server.

1.  **Registration:** Both peers maintain an asynchronous WSS connection (`->>`) to the Go Server, registering their Public Key ID.
2.  **Signaling:** The Go Server relays the SDP Offer/Answer and ICE candidates between peers based on the target Public Key.
3.  **Connection:** Once the ICE process is complete, media traffic flows **P2P directly** between the Android apps, bypassing the Go Server entirely.

For detailed component relationships and the dynamic call flow:
* [**Static Architecture Diagram**](docs/ARCHITECTURE.md)
* [**Sequence Diagram (Call Flow)**](docs/ARCHITECTURE.md)

---

## 👨‍💻 Getting Started (Development)

The project is split into the **Android Client** and the **Go Signaling Server**.

### Prerequisites

* Go: 1.18+
* Kotlin / Android Studio: Latest stable version
* VPS/Cloud Host: Required for deploying the Go Server over WSS (TLS is mandatory).

### Setting up the Go Server

1.  Clone the Go Server repository portion.
2.  Configure your TLS certificates.
3.  Run the server (e.g., on port 443).

### Building the Android Client

1.  Open the project in Android Studio.
2.  Update the WebSocket URL in the relevant configuration file to point to your live Go Server address.
3.  Build and run the application.

---

## 🤝 Contribution

We welcome contributions, particularly in the areas of security auditing, WebRTC optimization, and P2P connection stability.

Please review the [**CONTRIBUTING.md**](CONTRIBUTING.md) guide before submitting your first Pull Request.

---

## ⚖️ License

This project is released under the **GPLv3**.
