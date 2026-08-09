# TCP (Transmission Control Protocol) --- System Design Notes

> **Scope:** These notes focus on TCP for backend engineering, System
> Design interviews, and scalable real-world systems. They include the
> relevant TCP/IP model context because TCP makes much more sense when
> you understand where it sits in the networking stack.

------------------------------------------------------------------------

# 1. TCP in the TCP/IP Model

## What Is the TCP/IP Model?

The **TCP/IP model** is a practical layered networking model describing
how data is communicated between devices using standardized networking
protocols.

The commonly taught TCP/IP model has **4 layers**:

``` text
┌────────────────────────────────────┐
│ 4. Application                     │
│ HTTP, HTTPS, DNS, SMTP, FTP, etc. │
├────────────────────────────────────┤
│ 3. Transport                       │
│ TCP, UDP                           │
├────────────────────────────────────┤
│ 2. Internet                        │
│ IP, ICMP, etc.                     │
├────────────────────────────────────┤
│ 1. Network Access / Link           │
│ Ethernet, Wi-Fi, etc.              │
└────────────────────────────────────┘
```

TCP belongs to:

> **Layer 3 of the 4-layer TCP/IP model --- Transport layer.**

It corresponds roughly to the **Transport layer (Layer 4)** of the OSI
model.

### TCP/IP vs OSI

The OSI model has 7 layers:

``` text
OSI                         TCP/IP
────────────────────────────────────────
Application ─────────┐
Presentation         ├──→ Application
Session ─────────────┘
Transport ─────────────→ Transport
Network ───────────────→ Internet
Data Link ───────────┐
Physical ────────────┴──→ Network Access
```

The mapping is approximate rather than perfectly one-to-one.

### Why learn both?

For System Design:

-   **OSI** is useful for understanding networking concepts and
    troubleshooting by layer.
-   **TCP/IP** is closer to the protocol architecture used by the
    Internet.
-   TCP itself is a real protocol, not merely a conceptual OSI
    component.

------------------------------------------------------------------------

# 2. Important TCP/IP Model Corrections

The supplied TCP/IP article is broadly useful, but several statements
need qualification.

## ⚠️ Correction: TCP/IP layers don't personally perform all the listed functions

For example, saying:

> "The Application layer provides encryption."

is too broad.

A more accurate view is:

``` text
Application layer
    ↓
Application protocols
    ↓
HTTP / DNS / SMTP / etc.

Security
    ↓
TLS or application-level security
```

Similarly, session management and data formatting can be handled by
specific application protocols or libraries rather than by the TCP/IP
Application layer itself.

The layer is a conceptual grouping of protocols and functionality.

------------------------------------------------------------------------

## ⚠️ Correction: UDP does have error detection

It is incorrect to say that UDP provides "no error checking."

UDP includes a **checksum** for detecting corruption.

What UDP does **not** provide by itself is TCP-style:

-   reliable delivery
-   retransmission
-   ordered delivery
-   TCP-style flow control
-   TCP-style congestion control

So:

``` text
UDP
→ Can detect corruption
→ Does not automatically recover from loss
→ Does not guarantee ordering
```

------------------------------------------------------------------------

## ⚠️ Correction: TCP does not guarantee business-level accuracy

TCP provides reliable transport of a byte stream.

It does not guarantee:

> "The application operation happened exactly once."

For example:

``` text
Client
   |
   | POST /payment
   ↓
Server
   |
   | Payment succeeds
   |
   X Response lost
```

The client may see a timeout even though the payment succeeded.

This is why distributed systems need concepts such as:

-   idempotency
-   request IDs
-   deduplication
-   retries
-   transactional processing

------------------------------------------------------------------------

## ⚠️ Correction: IP fragmentation is more nuanced

The article says the Internet layer breaks large packets into smaller
ones and reassembles them at the destination.

That is a useful beginner-level description, but the exact behavior
differs between IPv4 and IPv6.

For System Design, the important idea is:

> Data may need to fit within the network's MTU, and fragmentation has
> performance and reliability implications.

You do not need to study fragmentation deeply yet.

------------------------------------------------------------------------

## ⚠️ Correction: ARP's layer placement is not perfectly clean

ARP maps an IP address to a link-layer address such as a MAC address on
a local network.

It is often associated with the Internet/Network layer in simplified
TCP/IP diagrams, but its behavior sits at the boundary between network
and link-layer concepts.

Don't spend too much time memorizing its layer number.

------------------------------------------------------------------------

## ⚠️ Correction: TCP/IP is not simply "open and controlled by nobody"

The Internet protocol suite is based on openly published standards
developed through organizations and standards processes, particularly
the **IETF**.

The useful System Design takeaway is:

> TCP/IP is based on widely implemented, openly documented Internet
> standards rather than being tied to one vendor's proprietary
> networking stack.

------------------------------------------------------------------------

# 3. What Is TCP?

**TCP (Transmission Control Protocol)** is a **connection-oriented
transport-layer protocol** used to provide reliable communication
between applications over an IP network.

TCP's main goal is to make communication over an unreliable packet
network behave like a:

> **Reliable, ordered byte stream between two application endpoints.**

Conceptually:

``` text
Application A
     |
     | Data
     ↓
    TCP
     |
     ↓
    IP
     |
     ↓
  Network
     |
     ↓
    IP
     |
     ↓
    TCP
     |
     ↓
Application B
```

The underlying IP network does **not** guarantee:

-   delivery
-   ordering
-   duplicate prevention
-   retransmission
-   congestion handling

TCP adds mechanisms to provide these transport-level properties.

------------------------------------------------------------------------

# 4. Why Does TCP Exist?

IP is fundamentally a **best-effort packet delivery protocol**.

Suppose data is transmitted as:

``` text
Packet 1
Packet 2
Packet 3
Packet 4
```

The network could produce:

``` text
Packet 1
Packet 3
Packet 4
Packet 2
```

or:

``` text
Packet 1
Packet 3
Packet 4

Packet 2 lost
```

or potentially duplicate data.

IP itself does not provide the application with a reliable ordered byte
stream.

TCP exists to deal with these problems.

It provides mechanisms for:

-   connection establishment
-   reliable delivery
-   ordered delivery
-   retransmission
-   duplicate handling
-   flow control
-   congestion control
-   connection termination

------------------------------------------------------------------------

# 5. TCP/IP Encapsulation

One of the most important networking concepts for System Design is
**encapsulation**.

When sending data:

``` text
Application data
      ↓
TCP segment
      ↓
IP packet
      ↓
Link-layer frame
      ↓
Bits/signals on the medium
```

A simplified representation:

``` text
┌────────────────────────────────────────────┐
│ Link-layer frame                           │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ IP packet                            │  │
│  │                                      │  │
│  │  ┌────────────────────────────────┐  │  │
│  │  │ TCP segment                    │  │  │
│  │  │                                │  │  │
│  │  │ TCP header + application data  │  │  │
│  │  └────────────────────────────────┘  │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

Terminology:

``` text
Application → data/message
TCP        → segment
IP         → packet
Link layer → frame
Physical   → bits/signals
```

These terms are useful to know for interviews.

------------------------------------------------------------------------

# 6. TCP's Core Mental Model

The most important thing to understand is:

> **TCP gives an application a reliable, ordered byte stream.**

Suppose an application writes:

``` text
HELLO WORLD
```

TCP treats the data as a stream of bytes.

It does **not** preserve application-level message boundaries.

For example, the application might perform:

``` text
write("HELLO")
write("WORLD")
```

The receiver is not guaranteed to receive:

``` text
"HELLO"
"WORLD"
```

It could receive:

``` text
"HELLOWORLD"
```

or:

``` text
"HEL"
"LOWO"
"RLD"
```

The application protocol must define how messages are framed if message
boundaries matter.

This is extremely important for backend engineering.

------------------------------------------------------------------------

# 7. TCP Endpoints and Ports

A TCP connection is associated with endpoints.

For example:

``` text
Client:
IP   = 192.168.1.10
Port = 50000

        ↓ TCP connection

Server:
IP   = 203.0.113.10
Port = 443
```

A connection can be identified conceptually using:

``` text
Source IP
Source Port
Destination IP
Destination Port
Protocol = TCP
```

This is commonly called the **4-tuple**:

``` text
(source IP, source port,
 destination IP, destination port)
```

This matters because one server can handle many simultaneous
connections.

``` text
Client A : 50001 ──┐
Client B : 50002 ──┼──> Server : 443
Client C : 50003 ──┘
```

The destination port can be the same while different client IP/port
combinations distinguish the connections.

------------------------------------------------------------------------

# 8. TCP Connection

TCP is **connection-oriented**.

Before normal data transfer begins, TCP establishes a connection between
the two endpoints.

``` text
Client                     Server
  |                          |
  | ---- Establish --------> |
  |                          |
  | <--- Establish --------- |
  |                          |
  | ===== Data Transfer ===> |
  | <==== Data Transfer ==== |
```

The connection allows TCP to maintain state about the communication.

This state includes things such as:

-   sequence numbers
-   acknowledgements
-   receive windows
-   connection state
-   retransmission information
-   congestion-control state

------------------------------------------------------------------------

# 9. TCP Three-Way Handshake

TCP uses a **three-way handshake** to establish a connection.

``` text
Client                                  Server
  |                                       |
  | -------- SYN -----------------------> |
  |                                       |
  | <------- SYN + ACK ------------------ |
  |                                       |
  | -------- ACK -----------------------> |
  |                                       |
  |          Connection established       |
```

The three messages are:

1.  **SYN**
2.  **SYN-ACK**
3.  **ACK**

------------------------------------------------------------------------

# 10. Step 1 --- SYN

The client sends a TCP segment with the **SYN flag** set.

``` text
Client
  |
  | SYN
  ↓
Server
```

The client is essentially saying:

> "I want to establish a TCP connection."

The client also communicates an **initial sequence number (ISN)**.

------------------------------------------------------------------------

# 11. Step 2 --- SYN-ACK

The server responds with:

``` text
SYN + ACK
```

The server is essentially saying:

> "I received your request, and I also want to establish the
> connection."

The server provides its own initial sequence number and acknowledges the
client's SYN.

------------------------------------------------------------------------

# 12. Step 3 --- ACK

The client sends an acknowledgement.

``` text
Client                         Server
  |                              |
  | -------- SYN ------------->  |
  | <------- SYN + ACK --------  |
  | -------- ACK ------------->  |
  |                              |
```

The connection can now proceed to data transfer.

------------------------------------------------------------------------

# 13. Why Three Steps?

The handshake allows both sides to establish that:

-   the other endpoint is reachable
-   both sides can send and receive
-   initial sequence numbers are established
-   TCP connection state can be synchronized

The third message is important because it confirms that the client
received the server's response.

------------------------------------------------------------------------

# 14. TCP Sequence Numbers

Sequence numbers are fundamental to TCP reliability and ordering.

Suppose the sender has:

``` text
A B C D E F G H
```

TCP tracks positions in the byte stream using sequence numbers.

If segments arrive out of order:

``` text
Segment 1 → bytes A B C
Segment 3 → bytes G H
Segment 2 → bytes D E F
```

TCP can use sequence information to reconstruct:

``` text
A B C D E F G H
```

rather than simply processing the data in arrival order.

------------------------------------------------------------------------

# 15. TCP Is a Byte Stream

TCP does not understand:

-   HTTP requests
-   JSON objects
-   database queries
-   application messages

It sees bytes.

For example:

``` text
Application:
"Hello"
"World"
```

TCP may transmit:

``` text
"HelloWorld"
```

and the receiver may read:

``` text
"Hel"
"loWo"
"rld"
```

The application protocol must define message boundaries.

------------------------------------------------------------------------

# 16. TCP and Backend Protocols

Suppose a backend receives:

``` text
POST /users
Content-Type: application/json

{"name":"Priyank"}
```

TCP does not understand:

``` text
POST
/users
JSON
```

TCP only transports bytes.

HTTP is responsible for interpreting those bytes as an HTTP request.

Therefore:

``` text
HTTP
→ understands application-level messages

TCP
→ reliably transports the bytes
```

This separation is fundamental.

------------------------------------------------------------------------

# 17. TCP Acknowledgements

TCP uses acknowledgements (**ACKs**) to confirm received data.

``` text
Sender                     Receiver
  |                           |
  | ---- Data --------------> |
  |                           |
  | <--------- ACK ---------- |
```

TCP acknowledgement numbers are generally **cumulative**.

Conceptually:

``` text
Sender sends:
bytes 1–100

Receiver:
ACK = 101
```

This means the receiver has received everything through byte 100 and
expects byte 101 next.

------------------------------------------------------------------------

# 18. Retransmission

Suppose:

``` text
Sender                         Receiver
  |                              |
  | ---- Segment 1 ------------> |
  | ---- Segment 2 ------X       |
  | ---- Segment 3 ------------> |
  |                              |
```

Segment 2 is lost.

TCP can detect that something is missing and retransmit the required
data.

``` text
Sender
  |
  | Segment 2
  |      X
  |
  | ---- Retransmit Segment 2 -> Receiver
```

This is one of the reasons TCP provides reliable delivery.

------------------------------------------------------------------------

# 19. How Does TCP Detect Loss?

TCP can detect loss through mechanisms including:

-   retransmission timeout
-   duplicate acknowledgements
-   fast retransmit

A simplified case:

``` text
Expected:
1 → 2 → 3 → 4

Received:
1 → 3 → 4
```

The receiver can repeatedly indicate that it is still waiting for
missing data.

The sender can infer that something is wrong and retransmit.

------------------------------------------------------------------------

# 20. TCP Checksum

TCP includes a **checksum**.

Its purpose is to detect corruption in the TCP segment.

``` text
Sender
  ↓
TCP data + checksum
  ↓
Network
  ↓
Receiver
  ↓
Checksum verification
```

If corruption is detected, the segment is treated as invalid.

### Important distinction

Checksum is primarily for **detecting accidental corruption**.

It is not equivalent to:

-   cryptographic hashes
-   digital signatures
-   authentication

------------------------------------------------------------------------

# 21. Flow Control

Flow control prevents a sender from overwhelming the receiver.

Suppose:

``` text
Sender
can send very quickly
      |
      ↓
Receiver
has limited buffer
```

If the sender keeps sending data faster than the receiver can
process/store it, the receiver's buffers could fill.

TCP uses a **receive window (`rwnd`)** to communicate how much
additional data the receiver can currently accept.

Conceptually:

``` text
Receiver:
"I can accept another 64 KB."

             ↓

Sender limits outstanding data accordingly.
```

------------------------------------------------------------------------

# 22. Flow Control vs Congestion Control

This distinction is extremely important.

## Flow Control

Question:

> **Can the receiver keep up?**

``` text
Sender → Receiver
          ↑
       Receiver capacity
```

TCP uses the receive window.

## Congestion Control

Question:

> **Can the network keep up?**

``` text
Sender
  ↓
Network
  ↓
Receiver

     ↑
Network congestion
```

TCP uses congestion-control mechanisms to avoid overwhelming the
network.

Therefore:

``` text
Flow Control
→ protects the receiver

Congestion Control
→ protects the network
```

------------------------------------------------------------------------

# 23. Congestion Control

Imagine:

``` text
Client A ──┐
Client B ──┤
Client C ──┼──> Network bottleneck
Client D ──┤
Client E ──┘
```

If everyone sends traffic aggressively, network queues can become
overloaded.

This can cause:

-   packet loss
-   increased latency
-   retransmissions
-   reduced throughput

TCP adapts its sending behavior based on network conditions.

Important concepts include:

-   **Congestion Window (`cwnd`)**
-   **Slow Start**
-   **Congestion Avoidance**
-   **Fast Retransmit**
-   **Fast Recovery**

You don't need to memorize every implementation detail initially.

The key idea is:

> **TCP dynamically controls how much data it puts into the network
> based on perceived congestion.**

------------------------------------------------------------------------

# 24. Receive Window vs Congestion Window

TCP's sending behavior is influenced by both:

``` text
Receive Window (`rwnd`)
→ receiver capacity

Congestion Window (`cwnd`)
→ network capacity
```

A useful conceptual model is:

``` text
Effective sending limit ≈ min(rwnd, cwnd)
```

This is a simplified mental model rather than a complete description of
every modern TCP implementation.

------------------------------------------------------------------------

# 25. TCP Connection Termination

TCP connections need to be closed.

A common graceful termination uses a **four-segment exchange** because
each direction of a TCP connection can be shut down independently.

``` text
Client                         Server
  |                              |
  | -------- FIN --------------> |
  | <------- ACK --------------- |
  |                              |
  | <------- FIN --------------- |
  | -------- ACK --------------> |
  |                              |
  |        Connection closed     |
```

This is commonly called **TCP four-way connection termination**.

------------------------------------------------------------------------

# 26. TCP Connection States

TCP maintains connection state.

Common states include:

-   `LISTEN`
-   `SYN-SENT`
-   `SYN-RECEIVED`
-   `ESTABLISHED`
-   `FIN-WAIT-1`
-   `FIN-WAIT-2`
-   `CLOSE-WAIT`
-   `CLOSING`
-   `LAST-ACK`
-   `TIME-WAIT`
-   `CLOSED`

You don't need to memorize every state immediately.

For System Design, these are particularly useful:

``` text
LISTEN
ESTABLISHED
TIME-WAIT
CLOSE-WAIT
CLOSED
```

------------------------------------------------------------------------

# 27. LISTEN

A server waiting for incoming TCP connections can be in:

``` text
LISTEN
```

For example:

``` text
Backend Server
Port 443
     ↓
LISTEN
```

It is ready to accept new TCP connections.

------------------------------------------------------------------------

# 28. ESTABLISHED

After the handshake:

``` text
Client ←──── TCP ────→ Server
```

the connection enters:

``` text
ESTABLISHED
```

Data can be exchanged.

------------------------------------------------------------------------

# 29. TIME-WAIT

After connection termination, one endpoint can enter:

``` text
TIME-WAIT
```

This is important because old delayed packets from the previous
connection should not be confused with packets from a new connection
using the same connection identifiers.

It also allows the endpoint to retransmit the final ACK if necessary.

### System Design relevance

A large number of connections being created and destroyed can lead to
many sockets in states such as `TIME-WAIT`.

This matters when dealing with:

-   high request rates
-   short-lived connections
-   ephemeral ports
-   connection-heavy architectures

------------------------------------------------------------------------

# 30. CLOSE-WAIT

`CLOSE-WAIT` means the remote endpoint has requested closure, but the
local application has not completely closed its side yet.

A large number of `CLOSE-WAIT` connections can indicate that an
application isn't properly closing connections.

This can become a resource problem.

------------------------------------------------------------------------

# 31. TCP/IP Working --- Sending Data

The complete simplified flow is:

``` text
Application
    │
    │ Data
    ▼
Transport / TCP
    │
    │ Segment
    ▼
Internet / IP
    │
    │ Packet
    ▼
Network Access / Link
    │
    │ Frame
    ▼
Physical medium
    │
    │ Bits/signals
    ▼
      Network
```

### Step 1 --- Application Layer

The application creates data.

Examples:

-   browser creates an HTTP request
-   email client creates an SMTP message
-   application creates a database request

### Step 2 --- Transport Layer

TCP receives the application data and adds transport information.

It handles concepts such as:

-   source/destination ports
-   sequence numbers
-   acknowledgements
-   reliability
-   flow control
-   congestion control

The resulting unit is a **TCP segment**.

### Step 3 --- Internet Layer

IP encapsulates the TCP segment into an IP packet.

The packet contains source and destination IP addresses.

Routers use IP addressing to forward the packet between networks.

### Step 4 --- Network Access / Link Layer

The IP packet is encapsulated into a link-layer **frame** suitable for
the local network.

Examples:

-   Ethernet
-   Wi-Fi

The frame is transmitted over the local medium.

------------------------------------------------------------------------

# 32. Receiving Data

The process is approximately reversed:

``` text
Physical medium
      ↓
Network Access / Link
      ↓
IP / Internet
      ↓
TCP / Transport
      ↓
Application
```

More specifically:

``` text
Signals/bits
    ↓
Frame
    ↓
IP packet
    ↓
TCP segment
    ↓
Application data
```

The receiver:

1.  receives the link-layer frame
2.  checks link-layer information/error detection
3.  extracts the IP packet
4.  processes IP addressing/routing information
5.  extracts the TCP segment
6.  uses TCP sequence numbers/ACKs/reassembly/reliability mechanisms
7.  delivers the resulting byte stream to the application

------------------------------------------------------------------------

# 33. TCP and HTTP/HTTPS

A common backend architecture looks like:

``` text
Browser
   |
 HTTP
   |
 TLS
   |
 TCP
   |
 IP
   |
Internet
   |
 IP
   |
 TCP
   |
 TLS
   |
 HTTP
   |
Backend
```

TCP doesn't know that the bytes represent HTTP.

### HTTPS

HTTPS is essentially:

``` text
HTTP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

TLS provides encryption and authentication properties.

TCP provides:

-   reliable byte-stream delivery
-   ordering
-   retransmission
-   flow control
-   congestion control

These are different responsibilities.

------------------------------------------------------------------------

# 34. TCP Connection Establishment + TLS

When using HTTPS over TCP, there can be multiple stages:

``` text
Client                    Server
  |                         |
  | ---- TCP handshake ---> |
  |                         |
  | <--- TCP established -- |
  |                         |
  | ---- TLS handshake ---> |
  | <--- TLS messages ----- |
  |                         |
  | ===== HTTPS data =====> |
```

This matters because connection setup contributes to latency.

Modern HTTP/3 uses QUIC instead of TCP, which changes the
connection-establishment story.

------------------------------------------------------------------------

# 35. TCP in Backend Systems

TCP appears throughout backend architectures.

For example:

``` text
                  ┌───────────────┐
                  │    Client     │
                  └───────┬───────┘
                          │
                        TCP
                          ↓
                  ┌───────────────┐
                  │ Load Balancer │
                  └───────┬───────┘
                          │
                        TCP
                          ↓
                  ┌───────────────┐
                  │ Backend       │
                  └───┬───────┬───┘
                      │       │
                    TCP     TCP
                      │       │
                      ↓       ↓
                   Redis    Database
```

Examples of communication that commonly use TCP:

``` text
Browser → Load Balancer
Load Balancer → Backend
Backend → Redis
Backend → MySQL/PostgreSQL
Backend → Kafka
```

The exact transport can vary by technology/configuration, but TCP is
extremely common in backend infrastructure.

------------------------------------------------------------------------

# 36. TCP Responsibilities vs Other Layers

A good System Design mental model is:

``` text
HTTP
→ What does this request mean?

TLS
→ Is the communication encrypted/authenticated?

TCP
→ Can I reliably transport this byte stream?

IP
→ Where should this packet go?

Ethernet/Wi-Fi
→ How do I communicate over this local link?
```

This separation prevents a lot of confusion.

------------------------------------------------------------------------

# 37. TCP Connection Overhead

Suppose every request creates a new TCP connection:

``` text
Request 1
→ TCP handshake
→ Request
→ Response
→ Close

Request 2
→ TCP handshake
→ Request
→ Response
→ Close

Request 3
→ TCP handshake
→ Request
→ Response
→ Close
```

There is repeated overhead.

At scale, this can become expensive.

------------------------------------------------------------------------

# 38. Connection Reuse / Keep-Alive

Instead, connections can be reused:

``` text
TCP connection
      |
      ├── Request 1
      ├── Request 2
      ├── Request 3
      ├── Request 4
      └── Request 5
```

This avoids repeatedly establishing TCP connections.

HTTP/1.1 supports persistent connections, and HTTP/2 provides
multiplexing over a connection.

Connection reuse can reduce:

-   handshake overhead
-   latency
-   CPU work
-   connection churn

------------------------------------------------------------------------

# 39. Connection Pooling

Backend applications commonly maintain pools of established connections.

``` text
Application
     |
     ↓
┌─────────────────────┐
│ Connection Pool     │
│                     │
│ TCP connection 1    │
│ TCP connection 2    │
│ TCP connection 3    │
│ TCP connection 4    │
└──────────┬──────────┘
           ↓
        Database
```

Instead of creating a new connection for every operation, the
application can reuse existing connections.

This is common with:

-   database clients
-   Redis clients
-   HTTP clients

### Why it matters at scale

Without pooling:

``` text
Request
  ↓
Create connection
  ↓
Use connection
  ↓
Destroy connection
```

With pooling:

``` text
Request
  ↓
Borrow connection
  ↓
Use connection
  ↓
Return connection to pool
```

Pooling reduces connection churn, but an excessively large pool can also
overload the downstream service.

So connection pooling has a trade-off:

``` text
Too few connections
→ requests wait

Too many connections
→ downstream overload/resource exhaustion
```

------------------------------------------------------------------------

# 40. TCP Head-of-Line Blocking

TCP provides ordered delivery.

That is useful, but it creates a trade-off.

Suppose:

``` text
Segment 1 → received
Segment 2 → LOST
Segment 3 → received
Segment 4 → received
```

TCP cannot simply deliver:

``` text
1
3
4
```

to the application if the application expects an ordered byte stream.

It waits for the missing data:

``` text
1
2 ← missing
3
4
```

So later data may be blocked behind earlier missing data.

This is called **Head-of-Line (HOL) blocking**.

------------------------------------------------------------------------

# 41. Why HOL Blocking Matters

HTTP/2 can multiplex multiple logical streams over one TCP connection.

Conceptually:

``` text
                 TCP Connection
                      |
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
     Stream A      Stream B      Stream C
```

If packet loss affects the TCP connection, TCP's ordered byte stream can
delay data for multiple HTTP/2 streams.

This is one motivation behind QUIC/HTTP/3.

------------------------------------------------------------------------

# 42. TCP vs QUIC / HTTP/3

Modern systems also use **QUIC**, which runs over UDP and implements
transport-like features at a higher layer.

Conceptually:

``` text
HTTP/2:
HTTP/2
  ↓
TCP
  ↓
IP

HTTP/3:
HTTP/3
  ↓
QUIC
  ↓
UDP
  ↓
IP
```

The important point is not:

> "UDP is better than TCP."

Instead:

> **QUIC deliberately builds reliability, congestion control,
> encryption, and multiplexed streams above UDP while changing
> connection and stream behavior.**

This allows modern applications to avoid some limitations associated
with TCP.

------------------------------------------------------------------------

# 43. TCP vs UDP

  -------------------------------------------------------------------------------
  Property                TCP                     UDP
  ----------------------- ----------------------- -------------------------------
  Connection              Connection-oriented     Connectionless

  Reliable delivery       Yes                     No built-in reliable delivery

  Ordered delivery        Yes                     No

  Retransmission          Yes                     No built-in retransmission

  TCP-style ACKs          Yes                     No

  TCP-style flow control  Yes                     No

  TCP-style congestion    Yes                     No
  control                                         

  Data model              Byte stream             Datagram/message-oriented

  Checksum                Yes                     Yes

  Overhead                Higher                  Lower

  Typical use             Reliable streams        Low-latency/datagram-oriented
                                                  applications
  -------------------------------------------------------------------------------

The important design question is:

> **Do I need TCP's guarantees, or do I need different transport
> behavior?**

------------------------------------------------------------------------

# 44. UDP Use Cases

UDP can be useful for applications where low latency and
application-specific behavior matter.

Examples include:

-   real-time media
-   online gaming
-   VoIP
-   DNS
-   QUIC

But do not memorize:

> "UDP = streaming/gaming and TCP = everything else."

Real systems are more nuanced.

For example, modern video delivery can use TCP-based protocols, while
QUIC/HTTP/3 uses UDP as its underlying transport.

The real question is:

> **Which transport properties does the application need?**

------------------------------------------------------------------------

# 45. TCP Failure Scenarios

TCP improves reliability, but it cannot make the network magically
reliable.

## Packet loss

``` text
Client
  |
  | Packet
  X
  |
Server
```

TCP may retransmit.

This adds latency.

------------------------------------------------------------------------

## High latency

A connection may remain healthy but be slow.

``` text
Client ──────────────── Server
             500ms
```

The application can still time out.

------------------------------------------------------------------------

## Server crash

TCP cannot make a crashed server process the request.

``` text
Client
  |
  | TCP
  X
Server crashed
```

------------------------------------------------------------------------

## Network partition

Two healthy services may be unable to communicate.

``` text
Service A
    |
    X
    |
Service B
```

Both machines can be alive while the network path between them is
broken.

------------------------------------------------------------------------

## Connection silently becoming unusable

A network path can disappear without either endpoint immediately
knowing.

This is why applications often use:

-   timeouts
-   keepalive mechanisms
-   health checks
-   retries

------------------------------------------------------------------------

# 46. TCP Does NOT Guarantee Application Success

This is one of the most important System Design concepts.

Suppose:

``` text
Client
  |
  | TCP request
  ↓
Server
```

The server receives the request and successfully updates the database:

``` text
UPDATE balance ...
```

But the server crashes before sending the response.

The client sees:

``` text
Timeout
```

What does the client know?

It does **not** know whether:

``` text
Operation failed
```

or:

``` text
Operation succeeded but response was lost
```

TCP only provides transport-level reliability.

It does not guarantee that the **business operation** was successfully
completed.

This leads to important distributed-system concepts such as:

-   idempotency
-   retries
-   request IDs
-   deduplication
-   transactional processing

------------------------------------------------------------------------

# 47. TCP Retries vs Application Retries

There are different levels of retry.

## TCP-level retransmission

TCP retransmits lost network data.

``` text
TCP segment lost
      ↓
TCP retransmits
```

## Application-level retry

The application retries an entire operation.

``` text
POST /payment
      ↓
Timeout
      ↓
Application retries
```

These are **not the same thing**.

TCP can successfully retransmit bytes while the application still
experiences a timeout or failure.

------------------------------------------------------------------------

# 48. TCP and Timeouts

TCP has its own timing and retransmission mechanisms, but applications
should still use application-level timeouts.

For example:

``` text
Backend A
   |
   | Request
   ↓
Backend B
```

If B doesn't respond:

``` text
A waits forever
```

is usually unacceptable.

Instead:

``` text
A
|
| Request
|
|---- 2 seconds ----|
                    ↓
                 Timeout
```

The application can then decide whether to:

-   retry
-   return an error
-   use fallback
-   fail fast

------------------------------------------------------------------------

# 49. TCP and Scalability

A server handling many TCP connections consumes resources.

Each connection can require:

-   memory
-   socket state
-   buffers
-   file descriptors
-   CPU for processing
-   load-balancer resources

For example:

``` text
1,000 clients
      ↓
1,000 connections

1,000,000 clients
      ↓
Potentially huge connection state
```

Therefore, large-scale systems need to think carefully about:

-   connection pooling
-   keep-alive
-   maximum connections
-   load balancing
-   connection timeouts
-   backpressure
-   resource limits

------------------------------------------------------------------------

# 50. Backpressure

Backpressure is a broader systems concept closely related to flow
control.

Suppose:

``` text
Producer
  ↓
Fast
  ↓
Consumer
  ↓
Slow
```

If the producer keeps sending data faster than the consumer can process
it, queues/buffers grow.

Eventually:

``` text
Memory
  ↓
Buffer fills
  ↓
Resource exhaustion
```

TCP's flow-control mechanisms provide network-level backpressure from
receiver toward sender.

This concept will become useful later when studying:

-   message queues
-   Kafka
-   streaming
-   distributed systems
-   reactive systems

------------------------------------------------------------------------

# 51. TCP's Main Trade-offs

## Advantages

### Reliable

Lost data can be retransmitted.

### Ordered

Data is delivered to the application in stream order.

### Error detection

TCP uses checksums to detect corruption.

### Flow control

Protects the receiver from being overwhelmed.

### Congestion control

Helps prevent excessive network congestion.

### Widely supported

TCP is fundamental to Internet networking and is supported almost
everywhere.

------------------------------------------------------------------------

## Disadvantages

### Connection setup overhead

The connection requires establishment before normal data transfer.

### Reliability adds overhead

Acknowledgements and retransmissions consume bandwidth and processing.

### Ordered delivery can cause HOL blocking

Missing data can delay later data.

### Connection state consumes resources

Large numbers of connections can become expensive.

### Retransmissions increase latency

A lost packet can require waiting before data can be delivered.

------------------------------------------------------------------------

# 52. TCP Header --- Important Fields

You don't need to memorize every field immediately, but know the
important ones:

``` text
┌─────────────────────────────────────┐
│ Source Port                         │
├─────────────────────────────────────┤
│ Destination Port                    │
├─────────────────────────────────────┤
│ Sequence Number                     │
├─────────────────────────────────────┤
│ Acknowledgement Number              │
├─────────────────────────────────────┤
│ Flags                               │
├─────────────────────────────────────┤
│ Receive Window                      │
├─────────────────────────────────────┤
│ Checksum                            │
├─────────────────────────────────────┤
│ Options                             │
├─────────────────────────────────────┤
│ Payload                             │
└─────────────────────────────────────┘
```

Important fields:

-   Source port
-   Destination port
-   Sequence number
-   Acknowledgement number
-   Flags
-   Receive window
-   Checksum

------------------------------------------------------------------------

# 53. Important TCP Flags

### SYN

Used to establish a connection.

### ACK

Indicates acknowledgement.

### FIN

Used for graceful connection termination.

### RST

Used to immediately reset/abort a connection.

### PSH

Associated with pushing data toward the receiving application; modern
application developers generally do not need to manage this directly.

### URG

Related to urgent data; rarely important in modern backend System
Design.

For your learning:

``` text
SYN → connection establishment
ACK → acknowledgement
FIN → graceful close
RST → reset/abort
```

is sufficient initially.

------------------------------------------------------------------------

# 54. TCP in a Real System --- Example

Consider a simplified Amazon-like shopping system:

``` text
                 Client
                    |
                  HTTPS
                    |
                   TCP
                    |
                    ↓
              Load Balancer
                    |
                  TCP
                    |
                    ↓
             Order Service
              /          \
             /            \
          TCP              TCP
           ↓                ↓
       Redis             Database
```

TCP is involved in multiple communication paths.

But each layer has a different responsibility:

``` text
HTTP
→ application semantics

TLS
→ encryption/authentication

TCP
→ reliable ordered byte transport

IP
→ routing

Ethernet/Wi-Fi
→ local-link communication
```

------------------------------------------------------------------------

# 55. System Design Perspective

TCP matters in System Design because it directly affects:

-   latency
-   throughput
-   connection management
-   resource usage
-   reliability
-   failure handling
-   scalability

When designing a large system, remember:

> **A network connection is not free.**

It consumes resources on potentially multiple components:

``` text
Client
  ↓
Load Balancer
  ↓
Backend
  ↓
Database
```

Each layer may maintain its own connection state.

------------------------------------------------------------------------

# 56. Important System Design Trade-offs

## Persistent connections

### Benefits

-   lower connection-establishment overhead
-   lower latency
-   fewer handshakes
-   less connection churn

### Costs

-   connections consume resources while idle
-   too many persistent connections can exhaust resources
-   load balancers and servers need connection limits/timeouts

------------------------------------------------------------------------

## Connection pooling

### Benefits

-   reuses established connections
-   reduces connection creation overhead
-   controls the number of downstream connections

### Costs

-   too-small pools create waiting
-   too-large pools can overload downstream systems
-   stale/broken connections need handling

------------------------------------------------------------------------

## Retries

### Benefits

-   can recover from transient failures

### Risks

-   can multiply load during outages
-   can create retry storms
-   can duplicate business operations

This is why retries often need:

-   bounded attempts
-   timeouts
-   exponential backoff
-   jitter
-   idempotency where appropriate

------------------------------------------------------------------------

# 57. Common Misconceptions

## Misconception 1: "TCP guarantees the request succeeds."

False.

TCP guarantees properties about transport of the byte stream, not
business-operation success.

------------------------------------------------------------------------

## Misconception 2: "TCP sends one ACK for every packet."

Not necessarily.

TCP acknowledgements are generally cumulative and implementations can
use different acknowledgement strategies.

------------------------------------------------------------------------

## Misconception 3: "TCP preserves application message boundaries."

False.

TCP is a byte stream.

If message boundaries matter, the application protocol must define
framing.

------------------------------------------------------------------------

## Misconception 4: "TCP is always slower than UDP."

Too simplistic.

TCP has additional mechanisms and overhead, but actual application
performance depends on network conditions, protocol behavior,
implementation, and workload.

------------------------------------------------------------------------

## Misconception 5: "TCP prevents packet loss."

False.

Packets can still be lost.

TCP **detects/reacts to loss and can retransmit data**.

------------------------------------------------------------------------

## Misconception 6: "If TCP connected successfully, the server is healthy."

Not necessarily.

The connection can exist while the application is overloaded,
malfunctioning, or unable to complete a particular request.

------------------------------------------------------------------------

## Misconception 7: "TCP handles authentication."

No.

TCP provides transport-level communication properties.

Authentication is handled by higher-level mechanisms such as TLS or
application authentication.

------------------------------------------------------------------------

## Misconception 8: "TCP and HTTP are the same thing."

No.

``` text
HTTP
→ application protocol

TCP
→ transport protocol
```

HTTP can use TCP, but they solve different problems.

------------------------------------------------------------------------

## Misconception 9: "UDP does no error checking."

False.

UDP has a checksum, but it does not provide TCP-style reliable recovery,
retransmission, or ordering.

------------------------------------------------------------------------

## Misconception 10: "The TCP/IP model and OSI model are interchangeable."

Not exactly.

They are related conceptual models with approximate layer mappings.

TCP/IP is more directly associated with the Internet protocol suite.

------------------------------------------------------------------------

# 58. What You Need to Understand Deeply

For System Design and backend engineering, prioritize:

### Must understand deeply

1.  TCP is a **reliable ordered byte stream**.
2.  TCP is **connection-oriented**.
3.  TCP belongs to the **Transport layer**.
4.  TCP three-way handshake.
5.  Sequence numbers.
6.  ACKs.
7.  Retransmission.
8.  Flow control.
9.  Congestion control.
10. TCP vs UDP.
11. TCP connection reuse / keep-alive.
12. Connection pooling.
13. TCP failure does not equal application-operation failure.
14. TCP head-of-line blocking.
15. Why network calls introduce latency and failure.
16. Basic TCP/IP encapsulation:

``` text
Data → Segment → Packet → Frame → Bits
```

17. The distinction between:

``` text
Flow control → receiver
Congestion control → network
```

### Useful but don't over-study initially

-   Every TCP state
-   Exact TCP header layout
-   Detailed congestion-control algorithms
-   Exact retransmission timeout calculations
-   Every TCP flag
-   Low-level packet parsing
-   Kernel implementation details
-   Detailed IP fragmentation mechanics

These become useful if you go deeper into networking, operating systems,
or performance engineering.

------------------------------------------------------------------------

# 59. Important Terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  TCP                                 Transmission Control Protocol

  TCP/IP                              Internet protocol suite/model

  Segment                             Data unit associated with TCP

  Packet                              Data unit associated with IP

  Frame                               Link-layer data unit

  Byte stream                         Ordered sequence of bytes exposed
                                      by TCP

  Connection-oriented                 Connection is established before
                                      normal data transfer

  SYN                                 TCP connection-establishment flag

  ACK                                 Acknowledgement

  FIN                                 Graceful connection termination

  RST                                 Connection reset

  Sequence number                     Tracks positions in TCP's byte
                                      stream

  Receive window (`rwnd`)             Receiver-advertised capacity

  Congestion window (`cwnd`)          Sender's estimate/control of
                                      network capacity

  Flow control                        Prevents receiver overload

  Congestion control                  Helps prevent network overload

  Retransmission                      Sending lost data again

  Checksum                            Detects corruption

  4-tuple                             Source IP/port + destination
                                      IP/port

  Keep-alive                          Reusing a TCP connection

  Connection pool                     Collection of reusable connections

  HOL blocking                        Later data waits behind missing
                                      earlier data

  QUIC                                Modern transport protocol
                                      implemented over UDP

  HTTP/3                              HTTP version built over QUIC

  MTU                                 Maximum Transmission Unit of a
                                      network link/path
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 60. System Design Interview Questions

1.  What problem does TCP solve?
2.  Why is TCP called connection-oriented?
3.  Explain the TCP three-way handshake.
4.  Why are three messages needed for the handshake?
5.  What are sequence numbers used for?
6.  How does TCP provide reliable delivery?
7.  What happens when a TCP packet is lost?
8.  What is the difference between flow control and congestion control?
9.  What is the TCP receive window?
10. What is the congestion window?
11. Why does TCP use acknowledgements?
12. Why does TCP retransmit data?
13. What is TCP's connection termination process?
14. What is `TIME-WAIT` and why does it exist?
15. What is `CLOSE-WAIT`?
16. Why can too many TCP connections become a scalability problem?
17. Why is connection pooling useful?
18. Why is keep-alive useful?
19. What is TCP head-of-line blocking?
20. Why did HTTP/3 move from TCP to QUIC over UDP?
21. What happens if the server processes a request but the TCP response
    never reaches the client?
22. Can TCP guarantee that a payment was processed exactly once?
23. Why do backend services need application-level timeouts if TCP is
    reliable?
24. When would you choose UDP instead of TCP?
25. Explain the difference between a TCP segment, IP packet, and
    Ethernet frame.
26. What is encapsulation in the TCP/IP model?
27. How does the TCP/IP model differ from the OSI model?
28. Why is flow control different from congestion control?
29. Why can a connection pool that is too large hurt a system?
30. What happens if a service retries requests during a network outage?
31. Why can TCP reliability still result in an application-level
    timeout?
32. Why does TCP's byte-stream model require application-level message
    framing?

------------------------------------------------------------------------

# 61. Why / What If Questions

### Why?

If TCP is reliable, why can an HTTP request still time out?

### What if?

The client sends:

``` text
POST /payment
```

The server processes the payment successfully, but the connection breaks
before the response reaches the client.

What does the client know?

### Why?

Why can't TCP simply deliver segment 3 to the application if segment 2
was lost?

### What if?

A server has 1 million clients and every client maintains a separate TCP
connection.

What resources become important?

### Why?

Why is connection pooling useful for database communication?

### What if?

A client creates thousands of short-lived TCP connections per second.

What problems might appear?

### Why?

Why can a single lost TCP packet affect multiple HTTP/2 streams?

### What if?

The receiver can process only 10 MB/s, but the sender tries to transmit
100 MB/s.

Which TCP mechanism is relevant?

### What if?

The network between two services is congested, even though both servers
are healthy.

Which TCP mechanism reacts to this?

### Why?

Why doesn't TCP retransmission solve duplicate payment problems?

### What if?

A backend service retries an operation after a TCP timeout.

Could the original operation already have succeeded?

What system-design mechanism would you need to make the operation safe?

### Why?

Why can opening a TCP connection for every HTTP request hurt a
high-traffic backend?

### What if?

A connection pool has 10 connections but 1,000 concurrent requests need
the database.

What happens?

### What if?

A connection pool has 10,000 connections and the database can
efficiently handle only 500.

Why might increasing the pool make the system worse?

### Why?

Why can persistent connections improve latency but still create
resource-management problems?

------------------------------------------------------------------------

# 62. Key Takeaways

1.  **TCP is a Layer 4 transport protocol in OSI terminology and a
    Transport-layer protocol in the common 4-layer TCP/IP model.**
2.  TCP provides a **reliable, ordered byte stream**.
3.  TCP is **connection-oriented**.
4.  TCP uses the **three-way handshake** to establish connections.
5.  TCP uses **sequence numbers** for ordering/tracking bytes.
6.  TCP uses **ACKs** to acknowledge received data.
7.  TCP can **retransmit lost data**.
8.  TCP uses **flow control** to avoid overwhelming the receiver.
9.  TCP uses **congestion control** to respond to network congestion.
10. TCP uses **checksums** for corruption detection.
11. TCP does not understand HTTP, JSON, database queries, or business
    operations.
12. TCP does not guarantee that a business operation succeeded.
13. TCP is a **byte stream**, not a message protocol.
14. TCP connection setup and maintenance have latency and resource
    costs.
15. Connection pooling and keep-alive reduce unnecessary connection
    overhead.
16. TCP's ordered delivery can cause **head-of-line blocking**.
17. TCP reliability comes with trade-offs in latency, overhead, and
    connection state.
18. UDP is not simply "better/faster"; it provides a different set of
    guarantees.
19. TCP/IP encapsulation follows the useful mental model:
    `text     Data → Segment → Packet → Frame → Bits`
20. **Flow control protects the receiver; congestion control protects
    the network.**
21. TCP is fundamental to understanding backend service-to-service
    communication.
22. At scale, TCP becomes part of the system's **latency, reliability,
    and resource-management story**.
23. A successful TCP connection does not imply successful
    application-level execution.
24. Network-level retries and application-level retries solve different
    problems.

------------------------------------------------------------------------

# 63. What to Learn Next

A good networking progression for System Design is:

``` text
OSI Model
   ↓
TCP/IP Model
   ↓
TCP
   ↓
UDP
   ↓
IP Addressing & Subnetting
   ↓
DNS
   ↓
HTTP / HTTPS
   ↓
TLS
   ↓
Load Balancing
   ↓
Reverse Proxies
   ↓
CDNs
   ↓
Sockets / Connection Management
   ↓
Network Latency & Performance
```

If your roadmap is following networking fundamentals in order, **UDP**
is a natural next topic because comparing TCP and UDP reinforces why
different transport protocols exist.

After UDP, **IP Addressing and Subnetting** will make the
Internet/Network layer much easier to understand.

------------------------------------------------------------------------

# 64. Final Mental Model

When you encounter networking in a System Design problem, think from the
bottom upward:

``` text
Application
    │
    │ HTTP / DNS / SMTP / etc.
    ▼
Transport
    │
    │ TCP / UDP
    ▼
Internet
    │
    │ IP
    ▼
Network Access
    │
    │ Ethernet / Wi-Fi
    ▼
Physical medium
```

For TCP specifically:

``` text
Application data
      ↓
TCP
      ↓
Reliable + ordered byte stream
      ↓
IP
      ↓
Routing across networks
      ↓
Link layer
      ↓
Actual local transmission
```

And for System Design, keep this distinction in your head:

``` text
TCP reliability
        ≠
Application reliability
        ≠
Business-operation correctness
```

For example:

``` text
TCP says:
"The bytes were delivered reliably."

Application says:
"I received the request."

Business logic says:
"The payment was committed."

These are three different guarantees.
```

That distinction becomes increasingly important as you move from
networking into distributed systems.
