# OSI Model Layers

## 1. What Is the OSI Model?

The **OSI (Open Systems Interconnection) model** is a conceptual model
that divides network communication into **7 layers**.

It helps us reason about networking by separating responsibilities:

``` text
┌──────────────────────────────┐
│ 7. Application               │
├──────────────────────────────┤
│ 6. Presentation              │
├──────────────────────────────┤
│ 5. Session                   │
├──────────────────────────────┤
│ 4. Transport                 │
├──────────────────────────────┤
│ 3. Network                   │
├──────────────────────────────┤
│ 2. Data Link                 │
├──────────────────────────────┤
│ 1. Physical                  │
└──────────────────────────────┘
```

The core idea is **separation of responsibilities**.

Instead of one enormous networking protocol handling everything,
different concerns are conceptually separated into layers.

For System Design, the OSI model is mainly useful as a **mental model
for understanding where different networking responsibilities occur**.

> **Important:** Modern Internet networking does not literally implement
> the OSI model as seven perfectly separate layers. The model is
> primarily conceptual.

------------------------------------------------------------------------

# 2. The Seven Layers at a Glance

  -----------------------------------------------------------------------------
  Layer             Name              Main responsibility     Common concepts
  ----------------- ----------------- ----------------------- -----------------
  7                 Application       Network services used   HTTP, FTP, SMTP,
                                      by applications         DNS

  6                 Presentation      Data representation,    Encoding,
                                      encoding, compression,  compression,
                                      encryption              encryption

  5                 Session           Establishing/managing   Session
                                      communication sessions  management

  4                 Transport         End-to-end delivery     TCP, UDP, ports,
                                      between applications    flow control

  3                 Network           Delivery between        IP, routing,
                                      networks                routers

  2                 Data Link         Delivery across a local MAC, frames,
                                      link                    Ethernet, Wi-Fi

  1                 Physical          Transmission of bits as Copper, fiber,
                                      physical signals        radio
  -----------------------------------------------------------------------------

A useful way to remember the overall flow:

``` text
Application data
      ↓
Presentation
      ↓
Session
      ↓
Transport → Segment
      ↓
Network   → Packet
      ↓
Data Link → Frame
      ↓
Physical  → Signals
```

On the receiving side, the process conceptually happens in reverse.

------------------------------------------------------------------------

# 3. Layer 7 --- Application Layer

## From My Notes

The Application layer consists of network applications such as:

-   Chrome
-   Firefox
-   etc.

It uses various protocols, for example:

-   **FTP** → file transfer
-   **HTTPS** → internet/web communication
-   **SMTP** → email transfer
-   **Telnet** → virtual terminals

The Application layer provides services to network applications in the
form of protocols.

## What does it actually mean?

The Application layer is where **application-level network protocols**
operate.

It defines how applications communicate over the network.

For example:

``` text
Browser
   ↓
HTTP/HTTPS
   ↓
Network
```

An email application might use:

``` text
Email application
       ↓
SMTP
       ↓
Network
```

A file-transfer application might use:

``` text
File transfer application
       ↓
FTP
       ↓
Network
```

### Important distinction

Chrome itself isn't technically an "Application-layer protocol."

Chrome is an **application** that uses application-layer protocols such
as HTTP/HTTPS.

So:

``` text
Chrome       → application
HTTPS        → protocol
```

Similarly:

``` text
Email client → application
SMTP         → protocol
```

### From your notes: protocols

-   **FTP** --- File Transfer Protocol
-   **HTTPS** --- HTTP secured using TLS
-   **SMTP** --- Simple Mail Transfer Protocol
-   **Telnet** --- remote terminal protocol

------------------------------------------------------------------------

# 4. Layer 6 --- Presentation Layer

## From My Notes

The Presentation layer receives data from the Application layer in terms
of characters and numbers.

It converts this into machine-understandable binary code.

You mentioned:

``` text
ASCII → EBCDIC
```

It then performs:

-   data compression
-   lossy or lossless compression
-   encryption
-   decryption

You mentioned SSL as an example of encryption/decryption.

## What is the Presentation layer responsible for?

Conceptually, it handles **how data is represented**.

Important responsibilities include:

### 1. Data representation / encoding

Different systems can represent data differently.

The Presentation layer conceptually ensures that communicating systems
agree on a representation.

Examples include:

-   character encoding
-   data formats
-   serialization

You mentioned:

``` text
ASCII → EBCDIC
```

This is a valid example of character encoding/representation conversion.

However, don't focus too much on ASCII vs EBCDIC for System Design.

More relevant modern examples are:

``` text
JSON
UTF-8
Protocol Buffers
MessagePack
```

### 2. Compression

Data can be compressed before transmission.

Two broad categories:

**Lossless compression**

``` text
Original
  ↓
Compressed
  ↓
Decompressed
  ↓
Exactly original data
```

Examples are appropriate when every bit matters.

**Lossy compression**

Some information is discarded to reduce size.

Useful for things such as:

-   images
-   audio
-   video

### 3. Encryption / Decryption

Conceptually:

``` text
Application data
       ↓
Encryption
       ↓
Encrypted representation
       ↓
Network
       ↓
Decryption
       ↓
Original data
```

This protects data from being readable by unauthorized parties.

## Correction / Clarification: SSL

Your idea that encryption belongs conceptually to the Presentation layer
is reasonable in the **OSI model**.

However:

> **SSL/TLS is not cleanly a modern OSI Layer 6 protocol.**

TLS is commonly positioned between application protocols and the
transport layer, and real-world protocol stacks don't map perfectly onto
OSI.

For example:

``` text
HTTP
 ↓
TLS
 ↓
TCP
 ↓
IP
```

So don't memorize:

> "TLS = Presentation layer."

Instead understand:

> **Encryption can be considered a Presentation-layer responsibility in
> the OSI conceptual model, while TLS is a real-world security protocol
> that doesn't map perfectly to one OSI layer.**

------------------------------------------------------------------------

# 5. Layer 5 --- Session Layer

## From My Notes

You wrote that the Session layer:

-   establishes connections
-   manages connections
-   uses helpers/APIs
-   handles authentication and authorization
-   keeps track of downloaded files such as images and data files

## What is the Session layer conceptually?

The Session layer is responsible for **establishing, managing, and
terminating logical communication sessions between applications**.

Conceptually:

``` text
Application A
      |
      ↓
   Session
      |
      ↓
Application B
```

It can deal with things such as:

-   session establishment
-   maintaining a session
-   synchronization
-   session termination
-   recovering/resuming a communication session in some models

## Correction needed: Authentication and authorization

Authentication and authorization are **not the core definition of the
OSI Session layer**.

They can occur as part of application/session management in real
systems, but you should not learn:

``` text
Session Layer = authentication + authorization
```

That's misleading.

For System Design, authentication/authorization are more commonly
handled through mechanisms such as:

-   sessions
-   cookies
-   tokens
-   OAuth
-   JWT
-   identity services

These don't map neatly to the OSI Session layer.

## Correction needed: "Keeping track of downloaded files"

The Session layer doesn't generally mean:

> "Keep track of downloaded images/data files."

That's more likely an **application-level concern**.

For example, a web application can track:

``` text
User
 ↓
Session ID
 ↓
Application state
```

But this isn't the same thing as the formal OSI Session layer.

## System Design importance

The Session layer is one of the OSI layers you should understand
**conceptually rather than deeply**.

Modern Internet stacks don't have a clean, universally implemented
Session layer like the OSI diagram suggests.

------------------------------------------------------------------------

# 6. Layer 4 --- Transport Layer

This is one of the **most important layers for System Design**.

## From My Notes

The Transport layer performs:

-   segmentation
-   flow control
-   error control

### Segmentation

Data is divided into segments.

You mentioned that segments contain:

-   port number
-   sequence number
-   data unit

The sequence number helps organize data.

The port number identifies which application the segment belongs to.

### Flow control

The amount of data transmitted is controlled.

For example:

``` text
Sender can send at 50 Mbps
Receiver can handle only 5 Mbps
```

The receiver can communicate that it cannot process data at the sender's
current rate.

If the transmission is too slow, the sender can increase the rate.

### Error control

Mechanisms such as retransmission can be used when data is
lost/corrupted.

Checksums can help detect corruption.

### Protocols

Transport layer protocols include:

-   TCP
-   UDP

You described:

-   TCP → connection-oriented
-   UDP → connectionless

UDP doesn't provide TCP-style acknowledgements and retransmission.

------------------------------------------------------------------------

# 7. Segmentation

Suppose the application wants to send:

``` text
Large application data
       ↓
┌──────┬──────┬──────┬──────┐
│ Seg1 │ Seg2 │ Seg3 │ Seg4 │
└──────┴──────┴──────┴──────┘
```

The Transport layer can divide it into smaller units.

For TCP, the resulting units are commonly called **TCP segments**.

## Correction about ports and sequence numbers

Your understanding is directionally correct, but the exact wording needs
refinement.

A TCP segment contains fields such as:

-   source port
-   destination port
-   sequence number
-   acknowledgement number
-   flags
-   checksum
-   receive window
-   payload

The **destination port** helps identify the receiving
application/service.

The **sequence number** helps TCP keep track of the byte stream and
properly order/reassemble data.

So don't think:

> "The segment has one port number telling which application it belongs
> to."

Instead:

> **Source and destination ports identify the communicating application
> endpoints, while TCP sequence numbers track the position of data
> within the byte stream.**

------------------------------------------------------------------------

# 8. Flow Control

Flow control prevents a fast sender from overwhelming a receiver.

Example:

``` text
Sender
50 Mbps
   |
   ↓
Network
   |
   ↓
Receiver
5 Mbps capacity
```

If the sender continuously sends at 50 Mbps, the receiver may not be
able to keep up.

TCP uses mechanisms such as a **receive window** to control how much
unacknowledged data can be in flight based on the receiver's capacity.

Conceptually:

``` text
Receiver:
"I can currently accept approximately this much more data."

             ↓

Sender adjusts transmission accordingly.
```

## Important distinction

Don't confuse:

**Flow control**

with

**Congestion control**.

This distinction is important.

``` text
Flow Control
→ Can the RECEIVER keep up?

Congestion Control
→ Can the NETWORK keep up?
```

TCP deals with both.

------------------------------------------------------------------------

# 9. Error Control

Networks aren't perfectly reliable.

Data can be:

-   lost
-   corrupted
-   duplicated
-   reordered

Protocols can use mechanisms to detect problems and potentially recover.

For example:

``` text
Sender
  |
  | Segment 1
  | Segment 2
  | Segment 3
  ↓
Receiver

Segment 2 lost

Receiver:
"I received 1 and 3,
but I'm missing something."
```

TCP can use acknowledgements and retransmissions to recover.

## Checksum vs hash vs signature

These should not be treated as interchangeable.

-   **Checksum** → primarily detects accidental corruption/transmission
    errors.
-   **Cryptographic hash** → integrity/fingerprinting mechanism with
    stronger properties.
-   **Digital signature** → provides
    integrity/authenticity/non-repudiation properties using
    cryptography.

For Transport-layer networking, **TCP's checksum** is the relevant
concept here.

------------------------------------------------------------------------

# 10. TCP

TCP provides a **connection-oriented, reliable, ordered byte stream**.

Conceptually:

``` text
Application
    ↓
   TCP
    ↓
Reliable ordered communication
    ↓
Network
```

TCP provides mechanisms for:

-   sequencing
-   acknowledgements
-   retransmission
-   flow control
-   congestion control
-   connection management

This is why applications can rely on TCP to hide many network-level
problems.

------------------------------------------------------------------------

# 11. UDP

UDP is **connectionless** and provides fewer delivery guarantees.

Conceptually:

``` text
Application
    ↓
   UDP
    ↓
Network
```

UDP does not provide TCP-style:

-   acknowledgements
-   retransmission
-   ordered delivery
-   connection establishment

If a UDP packet is lost:

``` text
Sender ─── Packet ───X──→ Receiver
```

UDP itself doesn't retransmit it.

## Correction: "UDP is faster"

Your statement:

> UDP is faster because there is no ACK and retransmission.

is **directionally useful but oversimplified**.

Better:

> UDP has lower protocol overhead and does not impose TCP's reliability
> mechanisms, which can make it preferable for latency-sensitive
> applications. But UDP is not inherently guaranteed to be faster in
> every scenario.

UDP is useful when the application prefers:

``` text
low overhead / low latency
        ↓
over
reliable ordered delivery
```

Examples can include:

-   real-time communication
-   gaming
-   some video/audio applications
-   DNS

------------------------------------------------------------------------

# 12. TCP vs UDP

  -----------------------------------------------------------------------
  Feature                 TCP                     UDP
  ----------------------- ----------------------- -----------------------
  Connection-oriented     Yes                     No

  Reliable delivery       Yes                     No

  Ordered delivery        Yes                     No

  Retransmission          Yes                     No

  ACK-based reliability   Yes                     No

  Flow control            Yes                     No TCP-style flow
                                                  control

  Congestion control      Yes                     No TCP-style congestion
                                                  control

  Overhead                Higher                  Lower

  Typical use             HTTP, database          DNS,
                          connections, many APIs  real-time/specialized
                                                  communication
  -----------------------------------------------------------------------

### System Design takeaway

The decision isn't:

> "TCP is good, UDP is bad."

It is:

> **What guarantees does the application need, and what latency/overhead
> trade-offs are acceptable?**

------------------------------------------------------------------------

# 13. Layer 3 --- Network Layer

## From My Notes

The Network layer:

-   receives segments from Transport
-   converts them into packets
-   is responsible for sending packets across networks
-   works through routers

Its functions include:

-   logical addressing
-   routing
-   path determination

It deals with:

-   IPv4
-   IPv6

The sender and receiver IP addresses are assigned to packets.

The source and destination IP addresses generally remain the same
throughout the journey.

The MAC addresses change at each network hop.

You also mentioned:

-   OSPF
-   BGP
-   IS-IS

for routing/path determination.

------------------------------------------------------------------------

# 14. Packet

Conceptually:

``` text
Application Data
      ↓
Transport
      ↓
Segment
      ↓
Network
      ↓
Packet
```

The Network layer is concerned with getting data from one network to
another.

------------------------------------------------------------------------

# 15. Logical Addressing --- IP

The Network layer uses **logical addressing**, primarily IP addresses.

Examples:

``` text
IPv4:
192.168.1.10

IPv6:
2001:db8::1
```

The packet contains source and destination IP information.

Conceptually:

``` text
Source IP             Destination IP
     ↓                       ↓
┌───────────────────────────────────┐
│ IP Header + Transport Data        │
└───────────────────────────────────┘
```

------------------------------------------------------------------------

# 16. IP Addresses Generally Remain the Same Across Routing

Suppose:

``` text
Computer A
192.168.1.10

        ↓

Router 1

        ↓

Router 2

        ↓

Server B
203.0.113.20
```

The IP packet conceptually has:

``` text
Source IP = A
Destination IP = B
```

and those addresses generally remain the same as the packet crosses
routers.

However, there are important exceptions involving things such as
**NAT**.

So the more accurate statement is:

> **In ordinary IP routing, source and destination IP addresses remain
> unchanged across intermediate routers; NAT can modify IP addresses at
> network boundaries.**

------------------------------------------------------------------------

# 17. MAC Addresses Change Between Hops

Suppose:

``` text
Computer A
    ↓
Router 1
    ↓
Router 2
    ↓
Server B
```

Each local network link has its own Data Link-layer addressing.

Conceptually:

``` text
Hop 1:
A MAC → Router 1 MAC

Hop 2:
Router 1 MAC → Router 2 MAC

Hop 3:
Router 2 MAC → Server B MAC
```

So:

``` text
IP:
A → B
(stays logically end-to-end)

MAC:
A → Router 1
Router 1 → Router 2
Router 2 → B
(changes per local link)
```

This distinction is **very important**.

------------------------------------------------------------------------

# 18. Correction: "MAC address can't be changed"

Your note says:

> MAC address can't be changed as it is a hardware ID.

This is not strictly correct.

A MAC address is associated with a network interface and is typically
assigned by the manufacturer, but operating systems/software can often
**change or spoof the MAC address presented by an interface**.

For System Design, the important point is:

> **MAC addresses operate at the local-link/Data Link layer, whereas IP
> addresses are used for routing across networks.**

Don't rely on "MAC can never change" as a rule.

------------------------------------------------------------------------

# 19. Correction: `225.225.225.0`

You wrote:

``` text
225.225.225.0
```

and described it as:

> first three combinations represent the network and the last one
> represents the host.

The concept you're describing is **subnetting / subnet masks**, but the
example is incorrect.

A common example is:

``` text
IP:
192.168.1.25

Subnet mask:
255.255.255.0
```

With `/24`:

``` text
192.168.1 | 25
    network | host
```

However, the exact division depends on the **prefix length/subnet
mask**.

For System Design, you only need basic subnet concepts initially. Don't
get bogged down in subnetting mathematics yet.

------------------------------------------------------------------------

# 20. Routing

The Network layer is responsible for determining how packets move
through interconnected networks.

Conceptually:

``` text
Computer A
    |
    ↓
Router 1
   / \
  ↓   ↓
 R2   R3
  \   /
   ↓ ↓
 Router 4
    |
    ↓
Computer B
```

The routing system determines which path packets should take.

Your graph analogy is useful:

``` text
Network = graph
Routers = nodes
Links = edges
```

Routing algorithms/protocols can consider things such as:

-   cost
-   reachability
-   topology
-   policy
-   path length

------------------------------------------------------------------------

# 21. Routing Protocols

You mentioned:

### OSPF --- Open Shortest Path First

An **interior gateway routing protocol** used within an autonomous
system.

### BGP --- Border Gateway Protocol

Used to exchange routing information between autonomous systems and is
fundamental to Internet-scale routing.

### IS-IS --- Intermediate System to Intermediate System

Another routing protocol used within networks/autonomous systems.

You don't need to learn their packet formats or detailed algorithms yet.

For System Design, the important idea is:

> **Routing protocols allow networks/routers to determine how traffic
> should reach destinations.**

------------------------------------------------------------------------

# 22. Layer 2 --- Data Link Layer

## From My Notes

The Data Link layer receives packets from the Network layer.

Logical addressing has already been done at the Network layer.

The Data Link layer performs **physical/link-level addressing**, such as
MAC addressing.

The resulting unit is called a **frame**.

It controls how data is placed onto and received from the transmission
medium.

The medium can be:

-   optical fiber
-   copper wire
-   air/wireless

The layer includes **Media Access Control (MAC)** functionality.

It also performs error detection.

You mentioned a shared network where multiple devices might try to
transmit simultaneously.

The Data Link layer can determine whether the shared medium is available
before transmitting.

You mentioned:

**CSMA --- Carrier Sense Multiple Access**

------------------------------------------------------------------------

# 23. Frame

The conceptual encapsulation is:

``` text
Application data
      ↓
Segment
      ↓
Packet
      ↓
Frame
      ↓
Bits/signals
```

So:

``` text
Layer 3 → Packet
Layer 2 → Frame
Layer 4 → Segment
```

These terms are useful to know for interviews.

------------------------------------------------------------------------

# 24. MAC Address

MAC addresses are used for **local-link communication**.

For example:

``` text
Computer
   ↓
Switch
   ↓
Router
```

The Data Link layer helps devices communicate over that particular
link/network segment.

This is different from IP addressing.

Think:

``` text
IP address
→ Where is the destination across networks?

MAC address
→ Which local interface/device should receive this frame on this link?
```

This distinction is much more useful than memorizing MAC as simply a
"hardware ID."

------------------------------------------------------------------------

# 25. Media Access Control

**MAC** also stands for **Media Access Control** in this context.

It deals with how devices access a shared transmission medium.

Suppose:

``` text
       Shared medium
      /      |      \
     A       B       C
```

If A, B and C transmit simultaneously, their signals may interfere
depending on the technology.

The Data Link layer can implement mechanisms controlling access to the
medium.

------------------------------------------------------------------------

# 26. CSMA

You mentioned:

**CSMA --- Carrier Sense Multiple Access**

The basic idea:

> A device checks/senses whether the medium is currently being used
> before transmitting.

Conceptually:

``` text
Device wants to transmit
          ↓
Is medium busy?
     /           \
   Yes            No
    ↓              ↓
 Wait          Transmit
```

The exact behavior depends on the underlying technology.

For System Design, understand the **shared-medium access problem**,
rather than memorizing CSMA details.

------------------------------------------------------------------------

# 27. Error Detection at Data Link Layer

The Data Link layer can detect corruption in frames.

Conceptually:

``` text
Sender
  ↓
Frame + error-detection information
  ↓
Network medium
  ↓
Receiver
  ↓
Check
  ↓
Corrupted?
```

Depending on the protocol, mechanisms such as **Frame Check Sequence
(FCS)/CRC** can be used.

### Important distinction

There can be error detection at multiple layers.

For example:

``` text
Transport → TCP checksum
Data Link → FCS/CRC
```

These aren't redundant in exactly the same way because they protect
different protocol layers and scopes.

------------------------------------------------------------------------

# 28. Layer 1 --- Physical Layer

## From My Notes

The Physical layer converts data represented as bits into physical
signals that can travel through a medium.

Examples:

-   copper cable → electrical signals
-   optical fiber → optical/light signals
-   wireless → radio signals through air

On the receiving side:

``` text
Signals
   ↓
Bits
```

The Physical layer is concerned with physically transmitting those bits.

------------------------------------------------------------------------

# 29. Physical Layer Flow

Sending:

``` text
Frame
  ↓
Bits
  ↓
Physical signals
  ↓
Copper / Fiber / Radio
```

Receiving:

``` text
Copper / Fiber / Radio
        ↓
Physical signals
        ↓
Bits
        ↓
Frame
```

Then the higher layers process the data.

------------------------------------------------------------------------

# 30. End-to-End Encapsulation

This is a useful way to tie all seven layers together.

Suppose your browser sends:

``` text
GET /users/123
```

Conceptually:

``` text
Application
    ↓
HTTP data
    ↓
Transport
    ↓
TCP segment
    ↓
Network
    ↓
IP packet
    ↓
Data Link
    ↓
Ethernet/Wi-Fi frame
    ↓
Physical
    ↓
Electrical / optical / radio signals
```

At the receiver:

``` text
Signals
   ↓
Bits
   ↓
Frame
   ↓
Packet
   ↓
Segment
   ↓
Application data
```

This process is called **encapsulation** when going down the stack and
**decapsulation** when going back up.

------------------------------------------------------------------------

# 31. Important Real-World Qualification

Your notes describe this as:

``` text
Application
 ↓
Presentation
 ↓
Session
 ↓
Transport
 ↓
Network
 ↓
Data Link
 ↓
Physical
```

That is correct as the **OSI conceptual model**.

But modern systems don't necessarily implement seven literal independent
layers.

For example, a common Internet-oriented model is closer to:

``` text
Application
     ↓
Transport
     ↓
Internet
     ↓
Link
```

HTTP, TLS, TCP, IP, Ethernet/Wi-Fi don't fit perfectly into the seven
OSI layers.

Therefore:

> **Use OSI primarily as a conceptual framework for understanding
> networking responsibilities, not as a literal description of every
> modern networking stack.**

------------------------------------------------------------------------

# System Design Perspective

## Why should a backend engineer care about OSI?

Because System Design involves connecting machines.

Consider:

``` text
                    Client
                       |
                    Internet
                       |
                       ↓
                Load Balancer
                       |
             ┌─────────┴─────────┐
             ↓                   ↓
        Backend A           Backend B
             |                   |
             └─────────┬─────────┘
                       ↓
                    Redis
                       |
                       ↓
                   Database
```

Every network communication here relies on concepts from networking.

For example:

### Client → Load Balancer

Could involve:

``` text
DNS
 ↓
IP
 ↓
TCP
 ↓
TLS
 ↓
HTTP
```

### Backend → Redis

Could involve:

``` text
IP
 ↓
TCP
 ↓
Redis protocol
```

### Backend → Database

Could involve:

``` text
IP
 ↓
TCP
 ↓
Database protocol
```

The OSI model helps you understand where the responsibilities are.

------------------------------------------------------------------------

# System Design Implications

Networking concepts become important when considering:

## Latency

Every network hop adds potential latency.

``` text
Service A
   ↓ 10ms
Service B
   ↓ 15ms
Database
   ↓ 10ms
Service B
   ↓ 15ms
Service A
```

Too many synchronous network calls can make an architecture slow.

## Reliability

Networks can fail.

``` text
Service A ───X─── Service B
```

So distributed systems need:

-   timeouts
-   retries
-   failure handling
-   circuit breakers
-   fallback strategies

## Scalability

As traffic grows:

``` text
1 server
   ↓
10 servers
   ↓
100 servers
   ↓
1000 servers
```

network communication becomes increasingly important.

You eventually encounter:

-   load balancers
-   reverse proxies
-   service discovery
-   CDNs
-   service meshes
-   connection pools

## Security

Network communication may need:

-   TLS
-   authentication
-   authorization
-   firewalls
-   network segmentation
-   encryption

------------------------------------------------------------------------

# Trade-offs and Failure Modes

Networking introduces problems that local function calls don't have.

### Packet loss

``` text
A ──────X──────> B
```

Can cause retransmission or application-level failure.

### High latency

``` text
A ───────────────> B
       500 ms
```

Can cause poor user experience.

### Congestion

Too much traffic can overwhelm network capacity.

### Connection failure

A previously established connection can disappear.

### Server failure

The network may be healthy while the destination service is down.

### Response loss

The server may successfully perform an operation while the response
never reaches the client.

This is particularly important for operations such as:

``` text
POST /payment
```

because retrying may cause duplicate operations.

------------------------------------------------------------------------

# From Your Notes → Important Conceptual Chain

Your notes contain this overall model:

``` text
Application
    ↓
Presentation
    ↓
Session
    ↓
Transport
    ↓
Network
    ↓
Data Link
    ↓
Physical
```

The data becomes progressively wrapped:

``` text
Application Data
       ↓
     Segment
       ↓
      Packet
       ↓
      Frame
       ↓
      Bits / Signals
```

And on the receiving side:

``` text
Bits / Signals
       ↓
     Frame
       ↓
     Packet
       ↓
    Segment
       ↓
Application Data
```

This is **encapsulation and decapsulation**.

------------------------------------------------------------------------

# Corrections / Clarifications

These are the points from your notes that are worth remembering
differently.

### 1. Chrome/Firefox are not protocols

They are applications that use application-layer protocols such as
HTTP/HTTPS.

### 2. TLS ≠ strictly Presentation layer

Encryption is conceptually associated with Presentation in OSI, but
modern TLS doesn't map cleanly to Layer 6.

### 3. Authentication/authorization ≠ Session layer's primary purpose

The Session layer conceptually manages communication sessions.
Authentication/authorization are primarily application/security concerns
in modern systems.

### 4. Download tracking ≠ Session layer

An application can track downloaded files, but that isn't a fundamental
OSI Session-layer responsibility.

### 5. Ports and sequence numbers

Ports identify application endpoints; TCP sequence numbers track byte
positions/order in the TCP stream.

### 6. Checksum, hash, and digital signature are different

They shouldn't be grouped together as equivalent error-control
mechanisms.

### 7. UDP isn't simply "faster"

UDP has less protocol overhead and fewer guarantees. Its suitability
depends on application requirements.

### 8. `225.225.225.0` is not the subnet-mask example you intended

`255.255.255.0` is the common `/24` example.

### 9. MAC addresses aren't immutable hardware IDs

They are associated with interfaces but can often be modified/spoofed by
software.

### 10. IP addresses aren't absolutely immutable throughout a journey

Ordinary routing preserves source/destination IPs across routers, but
mechanisms such as NAT can modify them.

### 11. OSI isn't the literal Internet architecture

The seven-layer model is conceptual. Modern protocols don't always map
cleanly onto exactly one OSI layer.

------------------------------------------------------------------------

# Worth Adding

These weren't explicitly in your notes but are important enough for
System Design.

## 1. Flow Control vs Congestion Control

You mentioned flow control.

Add this distinction:

``` text
Flow Control
→ Can the RECEIVER keep up?

Congestion Control
→ Can the NETWORK keep up?
```

TCP deals with both.

------------------------------------------------------------------------

## 2. Encapsulation

You already described the process, but the technical term is important:

> **Encapsulation** = each networking layer adds its own metadata/header
> around data from the layer above.

``` text
Application Data
       ↓
TCP Header + Data
       ↓
IP Header + TCP Segment
       ↓
Frame Header + IP Packet + Frame Trailer
```

------------------------------------------------------------------------

## 3. OSI vs TCP/IP Model

You should eventually understand that the practical Internet is
generally explained using the **TCP/IP model**, which has fewer layers.

A rough mapping:

``` text
OSI                         TCP/IP

Application ───────┐
Presentation ──────┼────── Application
Session ───────────┘

Transport ──────────────── Transport

Network ────────────────── Internet

Data Link ────────┐
Physical ─────────┴─────── Link
```

This will make networking terminology less confusing when you encounter
real-world documentation.

------------------------------------------------------------------------

# Key Takeaways

1.  **OSI is a conceptual seven-layer model for understanding network
    communication.**
2.  Each layer has a distinct responsibility.
3.  **Application** → network protocols used by applications.
4.  **Presentation** → data representation, compression, encryption
    conceptually.
5.  **Session** → session establishment/management conceptually.
6.  **Transport** → end-to-end application communication, TCP/UDP,
    ports, reliability/flow control.
7.  **Network** → IP addressing and routing across networks.
8.  **Data Link** → local-link delivery, frames, MAC addressing, medium
    access.
9.  **Physical** → transmission of bits as physical signals.
10. Data is conceptually encapsulated as:

``` text
Data → Segment → Packet → Frame → Signals
```

11. On reception, the process is reversed.
12. **OSI is a conceptual model; modern networking doesn't map perfectly
    onto it.**
13. For System Design, **Layers 3--7 matter substantially more than the
    physical details of Layer 1.**
14. Transport and Network layers are especially important for
    understanding distributed backend systems.

------------------------------------------------------------------------

# Important Terminology

### Layer 7

-   Application layer
-   HTTP
-   HTTPS
-   FTP
-   SMTP
-   Telnet

### Layer 6

-   Presentation layer
-   Encoding
-   Compression
-   Encryption
-   Decryption
-   TLS

### Layer 5

-   Session layer
-   Session establishment
-   Session management
-   Session termination

### Layer 4

-   Transport layer
-   TCP
-   UDP
-   Segment
-   Port
-   Sequence number
-   Acknowledgement
-   Flow control
-   Congestion control
-   Retransmission

### Layer 3

-   Network layer
-   IP
-   IPv4
-   IPv6
-   Packet
-   Routing
-   Router
-   OSPF
-   BGP
-   IS-IS
-   NAT
-   Subnet

### Layer 2

-   Data Link layer
-   Frame
-   MAC address
-   Media Access Control
-   CSMA
-   Ethernet
-   Wi-Fi
-   CRC/FCS

### Layer 1

-   Physical layer
-   Bit
-   Signal
-   Copper
-   Fiber
-   Radio

### Across the stack

-   Encapsulation
-   Decapsulation
-   OSI model
-   TCP/IP model

------------------------------------------------------------------------

# Questions You Should Be Able to Answer

1.  Why does the OSI model divide networking into layers?
2.  What is the difference between an IP address and a MAC address?
3.  Why do we need both IP addresses and MAC addresses?
4.  Why does the destination IP generally stay the same while MAC
    addresses change at every hop?
5.  What is the difference between a packet, segment, and frame?
6.  What is encapsulation?
7.  Why does TCP need sequence numbers?
8.  What is the difference between flow control and congestion control?
9.  Why might an application choose UDP instead of TCP?
10. Why doesn't UDP guarantee that data reaches its destination?
11. What does the Network layer actually do when a packet travels
    through multiple routers?
12. What problem does routing solve?
13. Why isn't Chrome itself considered an Application-layer protocol?
14. Why doesn't TLS map perfectly to the OSI Presentation layer?
15. Why isn't authentication simply a Session-layer responsibility?
16. What happens to a request as it moves down the networking stack?
17. What happens to the received data as it moves back up the stack?
18. Why is the OSI model useful even though modern networks don't
    literally implement seven independent layers?

## Deeper "Why / What If" Questions

**Why:** If IP already identifies the destination machine, why do we
need a port number?

**What if:** A packet reaches the correct server but the destination
port is wrong. What happens?

**Why:** If TCP provides reliability, why can an application still
experience a failed request?

**What if:** A server receives and processes a payment request, but its
response is lost before reaching the client. What does the client know?

**Why:** Why can't routers simply route using MAC addresses all the way
from your laptop to a server on the other side of the Internet?

**What if:** You have 100 microservices and one user request requires 20
sequential network calls. What happens to latency and reliability?

**Why:** Why is a network call fundamentally different from calling
another function inside the same process?

These last few are particularly important: **they are where the
OSI/networking knowledge starts turning into System Design knowledge.**
