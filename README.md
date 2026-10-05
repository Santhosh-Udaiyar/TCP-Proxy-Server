# TCP Proxy Server

> **Educational Computer Networks Project**  
> Standalone Java 17 + Maven TCP proxy server built exclusively with raw Java sockets (`ServerSocket` and `Socket`), featuring bidirectional streaming, connection telemetry, Swing graphical dashboard, connection limits, and host authorization security.

---

## Table of Contents
1. [Introduction & Architectural Overview](#introduction--architectural-overview)
2. [Computer Networks Concepts Explained](#computer-networks-concepts-explained)
   - [TCP Proxy Concept](#1-tcp-proxy-concept)
   - [Raw Sockets & TCP Handshake](#2-raw-sockets--tcp-handshake)
   - [Bidirectional Forwarding](#3-bidirectional-forwarding)
   - [Threading & Concurrency Model](#4-threading--concurrency-model)
   - [Connection Lifecycle](#5-connection-lifecycle)
   - [Security & Authorization Policy](#6-security--authorization-policy)
3. [Project Structure](#project-structure)
4. [Configuration (`proxy.properties`)](#configuration-proxyproperties)
5. [User Interface & Dashboard](#user-interface--dashboard)
   - [Swing Graphical Dashboard](#swing-graphical-dashboard)
   - [Interactive Headless CLI Mode](#interactive-headless-cli-mode)
6. [Compilation & Execution](#compilation--execution)
7. [Automated Testing & End-to-End Verification](#automated-testing--end-to-end-verification)
8. [Acceptable Use & Ethical Boundary Notice](#acceptable-use--ethical-boundary-notice)

---

## Introduction & Architectural Overview

A **TCP Proxy** operates at Layer 4 (Transport Layer) of the OSI model. Rather than terminating high-level application protocols (such as HTTP, SSH, or SMTP), it intercepts raw byte streams between a client application and an authorized destination server.

```
+--------------------+               +---------------------------------------+               +----------------------------+
|                    |               |            TCP PROXY SERVER           |               |                            |
|     TCP CLIENT     |  TCP Socket   |                                       |  TCP Socket   |     DESTINATION SERVER     |
| (Browser/Curl/App) | ------------> | ServerSocket(port)                    | ------------> | (Web server / Database /   |
|                    | <------------ |                                       | <------------ |  Echo / Microservice)      |
|                    |               |  Worker 1: Client -> Dest Forwarder   |               |                            |
|                    |               |  Worker 2: Dest -> Client Forwarder   |               |                            |
+--------------------+               |  Telemetry & Byte Accounting          |               +----------------------------+
                                     +---------------------------------------+
```

### Key Highlights
- **100% Raw Sockets**: Zero third-party proxy frameworks (no Netty, Apache MINA, or Spring Boot). Built strictly with `java.net.ServerSocket` and `java.net.Socket`.
- **Pure Java 17 + Maven**: Standard toolchain with zero unapproved runtime dependencies.
- **Bidirectional Streaming**: Paired dedicated worker threads handle upstream and downstream data concurrently with low latency (`TCP_NODELAY` enabled).
- **Traffic Telemetry**: Tracks active connections, total sessions, cumulative bytes sent/received, failed/rejected counts, and average connection durations.
- **Swing Graphical Dashboard**: Real-time table view, status badges, metric cards, and integrated log console.
- **Security Whitelisting**: Destination target authorization prevents open-relay abuse.

---

## Computer Networks Concepts Explained

### 1. TCP Proxy Concept
A proxy acts as an intermediary. At Layer 4, the proxy establishes two separate TCP connections:
1. **Client-to-Proxy**: The client initiates a TCP connection to the proxy's listening port.
2. **Proxy-to-Destination**: Upon accepting the client, the proxy opens a second TCP connection to the target server (`destination.host:destination.port`).

Once both TCP sockets are connected, the proxy transparently relays bytes from the client socket's `InputStream` to the destination socket's `OutputStream`, and vice versa. The client interacts with the proxy as if it were communicating directly with the real server.

### 2. Raw Sockets & TCP Handshake
In the BSD socket abstraction implemented by Java:
- `java.net.ServerSocket`: Binds to a local port and transitions the operating system TCP stack into the `LISTEN` state. It monitors incoming `SYN` packets from clients, completes the 3-way handshake (`SYN` $\to$ `SYN-ACK` $\to$ `ACK`), places fully established connections into the OS backlog queue, and returns a new `java.net.Socket` when `accept()` is called.
- `java.net.Socket`: Represents an endpoint of a full-duplex TCP connection. It exposes an `InputStream` (backed by the OS receive buffer) and an `OutputStream` (backed by the OS send buffer).
- `setTcpNoDelay(true)`: Disables Nagle's algorithm (`TCP_NODELAY`), minimizing latency for small TCP packets by forcing immediate transmission without buffering packets until an MSS (Maximum Segment Size) is reached.

### 3. Bidirectional Forwarding
TCP connections are **full-duplex**: data can flow in both directions simultaneously. To prevent blocking:
- If a single thread tried to read from the client and write to the server before checking for server responses, deadlocks would occur whenever both peers send or wait simultaneously.
- To achieve high-throughput, non-blocking forwarding, the proxy instantiates **two concurrent `Forwarder` tasks** for every connection:
  - **Upstream Forwarder (`Client -> Destination`)**: Reads from `clientSocket.getInputStream()`, writes to `destSocket.getOutputStream()`, flushes, and increments `bytesSent`.
  - **Downstream Forwarder (`Destination -> Client`)**: Reads from `destSocket.getInputStream()`, writes to `clientSocket.getOutputStream()`, flushes, and increments `bytesReceived`.
- A `CountDownLatch(2)` synchronizes completion. When one peer closes its output (sending `FIN`), the corresponding forwarder propagates the half-close via `socket.shutdownOutput()`, allowing pending data in flight to drain gracefully.

### 4. Threading & Concurrency Model
The server utilizes Java's `java.util.concurrent` primitives:
- **`acceptThread`**: A dedicated daemon thread running the `serverSocket.accept()` loop.
- **Cached Worker Pool (`ExecutorService`)**: Dynamically allocates worker threads for connection bridging and forwarding tasks.
- **Atomic Variables**: Thread-safe telemetry using lock-free CAS (Compare-And-Swap) operations (`AtomicLong`, `AtomicInteger`, `AtomicBoolean`).
- **Concurrent Collections**: `ConcurrentHashMap` stores active connections without lock contention, and `CopyOnWriteArrayList` safely broadcasts server events to UI listeners.

### 5. Connection Lifecycle
Every proxy session progresses through a well-defined state machine:

```
[Inbound Connection]
         |
         v
    [CONNECTING] ----(Host unauthorized OR Limit exceeded)----> [REJECTED / FAILED]
         |                                                             |
   (Socket connected)                                            (Socket closed)
         |
         v
      [ACTIVE] <--- Bidirectional I/O Stream Forwarding --->
         |
    (EOF / Disconnect / Server Stop)
         |
         v
      [CLOSED] ---> Updates duration, byte counters & logs
```

### 6. Security & Authorization Policy
Open proxies on the public internet pose severe security risks (such as unauthorized tunneling, DDoS amplification, and network boundary evasion). This implementation enforces strict security:
- **Destination Whitelist**: Configured in `allowed.destinations`. Any connection request directed toward a host not present in this whitelist is immediately rejected with a `SecurityException`.
- **Port Bounds Verification**: Strict boundary checks ($1 \le \text{port} \le 65535$).
- **Connection Limit Throttling**: A configurable `max.connections` ceiling protects against resource exhaustion and file descriptor starvation attacks.

---

## Project Structure

```
tcp-proxy-server/
├── pom.xml                               # Maven project configuration (Java 17, JUnit 5)
├── README.md                             # Comprehensive technical documentation
├── logs/
│   └── proxy.log                         # Persistent server log file
├── config/
│   └── proxy.properties                  # Runtime proxy configuration
├── src/main/java/com/proxy/
│   ├── Main.java                         # Application entry point (GUI or CLI launcher)
│   ├── server/
│   │   ├── ProxyServer.java              # ServerSocket listener, lifecycle & limits
│   │   ├── ProxyConnection.java          # Client-to-destination session bridge
│   │   └── Forwarder.java                # Unidirectional raw stream forwarding worker
│   ├── model/
│   │   ├── ConnectionInfo.java           # Per-connection state, byte counters & duration
│   │   └── TrafficStats.java             # Global telemetry metrics accumulator
│   ├── gui/
│   │   └── ProxyDashboard.java           # Swing monitoring dashboard & live log console
│   ├── config/
│   │   └── ProxyConfig.java              # Properties loader, validator & whitelist checker
│   └── util/
│       └── Logger.java                   # Thread-safe multi-target logging utility
└── src/test/java/com/proxy/
    ├── config/
    │   └── ProxyConfigTest.java          # Validation and security whitelist tests
    ├── model/
    │   ├── ConnectionInfoTest.java       # State and byte tracking unit tests
    │   └── TrafficStatsTest.java         # Metric accumulation and calculation tests
    ├── server/
    │   └── ProxyServerTest.java          # Integration tests (lifecycle, forwarding, limits)
    └── VerificationRunner.java           # Standalone end-to-end verification harness
```

---

## Configuration (`proxy.properties`)

The proxy is configured via `config/proxy.properties`:

| Property Key | Default Value | Description |
|---|---|---|
| `listen.port` | `9090` | Local port on which the proxy listens for client connections. |
| `destination.host` | `127.0.0.1` | Authorized destination server IP or hostname. |
| `destination.port` | `8080` | Target destination server port. |
| `max.connections` | `50` | Maximum number of concurrent active connections allowed. |
| `connection.timeout`| `5000` | Socket timeout (ms) when connecting to the destination server. |
| `allowed.destinations`| `127.0.0.1,localhost,::1` | Comma-separated whitelist of permitted destination hosts. |

---

## User Interface & Dashboard

### Swing Graphical Dashboard
To start the application with the Swing GUI:
```bash
java -jar target/tcp-proxy-server-1.0.0.jar --gui
```

#### Features:
1. **Interactive Controls**: Update `listen.port`, `destination.host`, `destination.port`, and `max.connections` directly from the dashboard and click **Start Proxy** or **Stop Proxy**.
2. **Telemetry Cards**: Live visual cards showing:
   - **Active Connections**
   - **Total Connections**
   - **Data Transferred** (formatted in B, KB, MB, GB)
   - **Average Duration** (ms or seconds)
   - **Failed / Rejected Connections**
3. **Session Table**: Displays each connection with columns:
   - ID (`#1`, `#2`, ...)
   - Client Address (`IP:port`)
   - Destination (`host:port`)
   - Start Time (`HH:mm:ss.SSS`)
   - Duration (updates dynamically every 500ms)
   - Bytes Sent
   - Bytes Received
   - Status (Color-coded badges for `ACTIVE`, `CONNECTING`, `CLOSED`, `FAILED`, `REJECTED`)
4. **Filters**: Filter table rows by *All Connections*, *Active Only*, *Closed Only*, or *Failed Only*.
5. **Real-Time Log Console**: Live server logs with an auto-scroll toggle and clear console button.

---

### Interactive Headless CLI Mode
For servers, remote shells, or headless environments:
```bash
java -jar target/tcp-proxy-server-1.0.0.jar --cli
```

#### Supported Interactive Commands:
- `status` — View current running state, ports, and active connection count.
- `stats` — Print detailed traffic telemetry (bytes sent/received, averages, rejected counts).
- `conns` — Print chronological session history with individual byte counts and durations.
- `clear` — Clear inactive sessions from memory.
- `stop` or `exit` — Perform a clean, graceful shutdown.
- `help` — List available CLI commands.

---

## Compilation & Execution

### Prerequisites
- **JDK 17** or higher
- **Apache Maven 3.8+**

### 1. Build Project
```bash
mvn clean package
```

### 2. Run the Application
- **Default (GUI on Desktop, CLI in Headless)**:
  ```bash
  java -jar target/tcp-proxy-server-1.0.0.jar
  ```
- **Force Graphical Dashboard**:
  ```bash
  java -jar target/tcp-proxy-server-1.0.0.jar --gui
  ```
- **Force Headless CLI Mode**:
  ```bash
  java -jar target/tcp-proxy-server-1.0.0.jar --cli
  ```
- **Specify Custom Configuration**:
  ```bash
  java -jar target/tcp-proxy-server-1.0.0.jar --config config/proxy.properties
  ```

---

## Automated Testing & End-to-End Verification

### 1. Run Unit & Integration Tests (JUnit 5)
```bash
mvn test
```
The test suite validates:
- **`ProxyConfigTest`**: Default fallback, stream loading, port boundary checks, and security whitelist enforcement.
- **`ConnectionInfoTest`**: Status transitions, duration calculation, and byte accumulation.
- **`TrafficStatsTest`**: Atomic counter increments, average duration calculation, byte formatting, and reset.
- **`ProxyServerTest`**: Mock destination server integration, bidirectional forwarding, multiple concurrent clients, connection ceiling enforcement, and graceful shutdown.

### 2. Run the End-to-End Verification Runner
A dedicated verification harness tests the full lifecycle end-to-end:
```bash
java -cp "target/classes;target/test-classes" com.proxy.VerificationRunner
```

**Verification Steps Executed:**
1. Starts a local TCP echo server on port `9876`.
2. Starts the `ProxyServer` on port `9877` targeting `127.0.0.1:9876`.
3. Connects Client 1 through the proxy, sends data, and verifies bidirectional echoing.
4. Spawns concurrent clients and verifies simultaneous bidirectional transfer.
5. Sends an excess connection and verifies that the `max.connections` ceiling cleanly rejects it.
6. Asserts cumulative byte counts and traffic statistics.
7. Shuts down the proxy server gracefully and confirms active sockets are closed.

---

## Acceptable Use & Ethical Boundary Notice
> **IMPORTANT SECURITY & EDUCATIONAL NOTICE**:  
> This TCP proxy server is developed strictly for **educational purposes** within Computer Networks curricula. It must **only** be deployed and tested against systems that you own or are explicitly authorized to assess. The software strictly validates destinations against an authorization whitelist and expressly forbids open-proxy operation.
