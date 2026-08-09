# UDP (User Datagram Protocol) --- System Design Notes

> **Scope:** These notes focus on UDP for backend engineering, System
> Design interviews, and scalable real-world systems. The goal is to
> understand why UDP exists, what guarantees it provides and does not
> provide, where it is useful, and how modern systems build on top of
> it.

------------------------------------------------------------------------

# 1. What Is UDP?

**UDP (User Datagram Protocol)** is a **Transport-layer protocol** used
for communication between application processes over IP networks.

UDP is:

-   connectionless
-   lightweight
-   datagram-oriented
-   low-overhead
-   not inherently reliable
-   not inherently ordered

The central idea is:

> **UDP sends independent datagrams without establishing a TCP-style
> connection or providing TCP's reliability mechanisms.**

Conceptually:

``` text
Application
     |
     | Data
     ↓
    UDP
     |
     | Datagram
     ↓
     IP
     |
     ↓
   Network
```

UDP is useful when an application values properties such as:

-   low latency
-   low overhead
-   simple request/response communication
-   application-controlled reliability
-   tolerance for some packet loss
-   datagram/message boundaries

------------------------------------------------------------------------

# 2. Why Does UDP Exist?

TCP provides many useful guarantees:

-   reliable delivery
-   ordered delivery
-   retransmission
-   flow control
-   congestion control
-   connection establishment

But these guarantees have costs.

Some applications do not want TCP's complete behavior.

For example, imagine a live voice call:

``` text
Voice packets:

1 → 2 → 3 → 4 → 5
```

Suppose packet 2 is lost.

TCP's ordered byte-stream behavior can cause later data to wait for the
missing data.

For a live conversation, waiting for old audio may be worse than simply
losing a tiny piece of audio.

The application may prefer:

``` text
1 → 2 lost → 3 → 4 → 5
```

rather than:

``` text
1 → 2 lost
         ↓
     wait for 2
         ↓
3, 4, 5 delayed
```

UDP gives the application more control over what to do with loss,
ordering, retransmission, and timing.

------------------------------------------------------------------------

# 3. UDP vs TCP --- Core Mental Model

A useful first comparison:

``` text
TCP
→ "Give me a reliable, ordered byte stream."

UDP
→ "Send these independent datagrams; I will decide
   what reliability/ordering behavior I need."
```

  ---------------------------------------------------------------------------
  Property                TCP                     UDP
  ----------------------- ----------------------- ---------------------------
  Connection              Connection-oriented     Connectionless

  Reliability             Built-in                Not built-in

  Ordering                Built-in                Not guaranteed

  Retransmission          Built-in                Not built-in

  Acknowledgements        Built-in                Not built-in

  Flow control            Built-in                Not built-in

  TCP-style congestion    Built-in                Not built-in
  control                                         

  Data model              Byte stream             Datagram/message-oriented

  Header size             Larger                  8 bytes

  Overhead                Higher                  Lower

  Application control     Less transport behavior More transport behavior can
                          exposed                 be application-controlled
  ---------------------------------------------------------------------------

The important System Design question is not:

> "Which one is faster?"

It is:

> **What transport guarantees does the application actually need?**

------------------------------------------------------------------------

# 4. UDP Is Connectionless

UDP does not establish a TCP-style connection before sending data.

With TCP:

``` text
Client
  |
  | SYN
  ↓
Server
  |
  | SYN-ACK
  ↓
Client
  |
  | ACK
  ↓
Connection established
```

With UDP:

``` text
Client
  |
  | UDP datagram
  ↓
Server
```

There is no UDP three-way handshake.

This makes UDP useful for communication where the overhead or latency of
establishing a connection is undesirable.

However:

> **Connectionless does not mean "no state can exist anywhere."**

An application using UDP can still maintain its own state.

For example:

``` text
Client
  |
  | UDP
  ↓
Game Server
  |
  └── maintains player state
```

UDP itself simply does not establish and maintain a TCP-style transport
connection.

------------------------------------------------------------------------

# 5. UDP Datagram

UDP is **datagram-oriented**.

Each UDP transmission is treated as a separate datagram.

Conceptually:

``` text
Datagram 1 → "HELLO"
Datagram 2 → "WORLD"
```

Unlike TCP, UDP preserves the application's datagram/message boundaries
at the transport API.

The receiver can conceptually receive:

``` text
"HELLO"
"WORLD"
```

as separate datagrams.

This is an important difference:

``` text
TCP
→ byte stream

UDP
→ datagrams
```

------------------------------------------------------------------------

# 6. UDP Header

A UDP header is only **8 bytes**.

It contains four fields:

``` text
┌────────────────────────────────────┐
│ Source Port       │ Destination   │
│                   │ Port          │
├────────────────────────────────────┤
│ Length            │ Checksum      │
├────────────────────────────────────┤
│              Data                  │
└────────────────────────────────────┘
```

Each field is 16 bits.

  ---------------------------------------------------------------------------
  Field                                         Size Purpose
  --------------------- ---------------------------- ------------------------
  Source Port                                16 bits Identifies the
                                                     sender-side port

  Destination Port                           16 bits Identifies the receiving
                                                     application/process

  Length                                     16 bits Total length of UDP
                                                     header + payload

  Checksum                                   16 bits Detects
                                                     corruption/misdelivery
  ---------------------------------------------------------------------------

The total UDP header is:

``` text
16 + 16 + 16 + 16
= 64 bits
= 8 bytes
```

------------------------------------------------------------------------

# 7. UDP Port Numbers

UDP uses port numbers to identify application endpoints.

Port range:

``` text
0 – 65535
```

Port `0` is reserved.

For example:

``` text
Client
IP:   192.168.1.10
Port: 53000

        UDP

Server
IP:   8.8.8.8
Port: 53
```

Port 53 is commonly used by DNS.

The IP address identifies the host/network endpoint, while the port
helps identify the application endpoint on that host.

This allows multiple applications to use the network simultaneously.

------------------------------------------------------------------------

# 8. UDP Checksum

A common misconception is:

> "UDP has no error checking."

That is not correct.

UDP has a **checksum** used to detect corruption.

Conceptually:

``` text
Sender
  |
  | UDP data + checksum
  ↓
Network
  |
  | data may become corrupted
  ↓
Receiver
  |
  | checksum verification
  ↓
Valid / invalid
```

If the checksum indicates corruption, the datagram is discarded rather
than repaired by UDP.

The important distinction is:

``` text
Checksum
→ detects corruption

Reliability
→ retransmission/recovery
```

UDP provides the first, but not the second.

UDP does not automatically retransmit a corrupted/lost datagram.

------------------------------------------------------------------------

# 9. UDP Checksum: IPv4 vs IPv6

The checksum behavior has an important qualification.

### IPv4

UDP checksum can be optional under the IPv4 specification.

### IPv6

UDP checksum is generally mandatory, with specific exceptions defined
for certain IPv6 mechanisms.

For System Design:

> Remember that UDP does have checksums. Do not say "UDP performs no
> error checking."

You do not need to memorize the standards-level exceptions yet.

------------------------------------------------------------------------

# 10. UDP Pseudo Header

UDP checksum calculation uses information from a **pseudo header**.

The pseudo header is **not transmitted as part of the UDP header**.

It conceptually includes information such as:

-   source IP address
-   destination IP address
-   IP protocol number
-   UDP length

The purpose is to make checksum verification more robust against things
such as:

-   accidental misdelivery
-   incorrect IP addressing information
-   corruption affecting protocol/length information

Conceptually:

``` text
IP information
     +
UDP header
     +
UDP payload
     ↓
Checksum calculation
```

At the receiver, the checksum is recalculated and compared.

### Important clarification

The pseudo header does **not** route the packet or guarantee delivery to
the correct host.

IP is responsible for addressing and routing.

The pseudo header simply contributes information to UDP's checksum
calculation.

------------------------------------------------------------------------

# 11. UDP and IP

UDP operates above IP.

When an application sends data:

``` text
Application
     |
     | Data
     ↓
UDP
     |
     | UDP header + data
     ↓
IP
     |
     | IP header + UDP datagram
     ↓
Network
```

The UDP datagram is encapsulated inside an IP packet.

At the receiver:

``` text
Network
   ↓
IP
   ↓
UDP
   ↓
Application
```

UDP removes/interprets its transport header and delivers the datagram to
the appropriate application endpoint.

------------------------------------------------------------------------

# 12. UDP Encapsulation

The useful networking mental model is:

``` text
Application data
       ↓
UDP datagram
       ↓
IP packet
       ↓
Link-layer frame
       ↓
Bits/signals
```

Compare with TCP:

``` text
Application data
       ↓
TCP segment
       ↓
IP packet
       ↓
Link-layer frame
       ↓
Bits/signals
```

This is important because UDP does not replace IP.

They solve different problems:

``` text
UDP
→ process-to-process transport

IP
→ host-to-host addressing and routing
```

------------------------------------------------------------------------

# 13. What UDP Does NOT Guarantee

UDP does not inherently guarantee:

### Delivery

A datagram can be lost.

``` text
Sender
  |
  | Datagram
  X
  |
Receiver
```

UDP does not automatically retransmit it.

------------------------------------------------------------------------

### Ordering

Suppose the sender transmits:

``` text
1 → 2 → 3 → 4
```

The receiver could observe:

``` text
1 → 3 → 4 → 2
```

UDP itself does not reorder these datagrams.

------------------------------------------------------------------------

### Duplicate prevention

A datagram can potentially be duplicated by the network/path.

UDP does not provide TCP-style duplicate suppression for the
application.

------------------------------------------------------------------------

### Retransmission

If a datagram is lost:

``` text
UDP
→ does not automatically retransmit it
```

The application can implement its own retransmission mechanism if
needed.

------------------------------------------------------------------------

### Flow control

UDP does not provide TCP's receive-window mechanism.

An application can send faster than the receiver can process data.

The application/system therefore needs to handle the consequences
appropriately.

------------------------------------------------------------------------

### TCP-style congestion control

UDP itself does not provide TCP's congestion-control machinery.

This is an important responsibility for protocols built over UDP,
especially on the public Internet.

------------------------------------------------------------------------

# 14. UDP Does Not Mean "No Reliability"

This is an important System Design nuance.

It is more accurate to say:

> **UDP itself does not provide reliable delivery.**

An application or protocol built on top of UDP can add reliability.

For example:

``` text
Application Protocol
    |
    ├── sequence numbers
    ├── acknowledgements
    ├── retransmission
    ├── congestion control
    └── encryption
    |
   UDP
    |
   IP
```

This is essentially the approach taken by protocols such as **QUIC**.

Therefore:

``` text
UDP
≠ inherently unreliable applications

UDP
= transport with minimal built-in guarantees
```

------------------------------------------------------------------------

# 15. Why Real-Time Applications Often Use UDP

Consider a live voice call.

Suppose:

``` text
Audio packet 100
Audio packet 101
Audio packet 102
Audio packet 103
```

Packet 101 is lost.

If packet 101 is retransmitted after 200 ms, it may arrive too late to
be useful.

For a live conversation:

``` text
Slight audio loss
        ↓
Often tolerable

Large delay
        ↓
Very noticeable
```

Therefore, an application may prefer:

``` text
packet 100
packet 101 lost
packet 102
packet 103
```

rather than delaying everything until packet 101 arrives.

This is a **latency vs reliability trade-off**.

------------------------------------------------------------------------

# 16. Important Caveat: UDP Is Not Automatically Better for Streaming

Do not memorize:

> "Streaming uses UDP."

Real systems are more complicated.

Video can be delivered using:

-   TCP-based protocols
-   HTTP
-   HTTP/2
-   HTTP/3 over QUIC/UDP
-   application-specific protocols

The correct design question is:

> **Does the application benefit from datagram semantics, low transport
> overhead, application-controlled reliability, or reduced impact from
> packet loss?**

------------------------------------------------------------------------

# 17. UDP Applications

## DNS

DNS commonly uses UDP for ordinary queries.

Why?

-   requests are often small
-   responses are often small
-   no TCP-style connection setup is needed for each query
-   low overhead is useful

Conceptually:

``` text
Client
  |
  | UDP query
  ↓
DNS Server
  |
  | UDP response
  ↓
Client
```

DNS can also use TCP and other transports in situations where required.

------------------------------------------------------------------------

# 18. DHCP

**DHCP (Dynamic Host Configuration Protocol)** uses UDP.

DHCP is used for dynamically configuring network hosts.

It involves small control messages and has special requirements during
initial network configuration.

A simplified view:

``` text
Client
  |
  | DHCP message
  ↓
DHCP Server
```

DHCP commonly uses UDP ports:

``` text
Server → UDP 67
Client → UDP 68
```

------------------------------------------------------------------------

# 19. VoIP

**VoIP (Voice over IP)** can use UDP for real-time audio communication.

The key trade-off is:

``` text
Small packet loss
       ↓
Maybe tolerable

Large delay
       ↓
Conversation becomes difficult
```

Therefore, minimizing latency can be more important than guaranteeing
delivery of every packet.

------------------------------------------------------------------------

# 20. NTP

**NTP (Network Time Protocol)** commonly uses UDP for time
synchronization.

The communication is generally lightweight and request/response
oriented.

UDP's low overhead is useful here.

------------------------------------------------------------------------

# 21. RIP

**RIP (Routing Information Protocol)** uses UDP to exchange routing
information.

This is a useful example showing that UDP is not only for multimedia.

It can also be used for network-control protocols where simple datagram
communication is appropriate.

------------------------------------------------------------------------

# 22. QUIC and HTTP/3

One of the most important modern examples of UDP usage is **QUIC**.

HTTP/3 is built on QUIC.

Conceptually:

``` text
HTTP/1.1 / HTTP/2
       ↓
      TCP
       ↓
      IP

HTTP/3
       ↓
      QUIC
       ↓
      UDP
       ↓
      IP
```

At first this may seem strange:

> "If UDP does not provide reliability, why would HTTP use it?"

Because QUIC implements transport-like functionality above UDP.

QUIC provides features including:

-   reliable delivery
-   ordered delivery within individual streams
-   congestion control
-   encryption
-   multiplexed streams
-   connection migration capabilities

This is a critical System Design lesson:

> **UDP can be used as a minimal transport foundation while a
> higher-level protocol implements the exact behavior it needs.**

------------------------------------------------------------------------

# 23. UDP + QUIC vs TCP

A simplified comparison:

``` text
Traditional:
HTTP
 ↓
TCP
 ↓
IP

Modern:
HTTP/3
 ↓
QUIC
 ↓
UDP
 ↓
IP
```

QUIC can avoid some limitations associated with TCP's single ordered
byte stream.

For example:

``` text
QUIC connection
 ├── Stream A
 ├── Stream B
 └── Stream C
```

Loss affecting one stream does not necessarily block delivery of
unrelated streams in the same way TCP's connection-wide byte ordering
can affect HTTP/2 multiplexing.

This is one reason QUIC is important to understand when learning modern
backend networking.

------------------------------------------------------------------------

# 24. UDP and Message Boundaries

One major difference from TCP:

### TCP

``` text
write("HELLO")
write("WORLD")

Receiver may read:
"HELLOWORLD"
or
"HEL"
"LOWO"
"RLD"
```

TCP is a byte stream.

### UDP

``` text
send("HELLO")
send("WORLD")
```

The datagrams remain separate at the UDP transport interface.

Conceptually:

``` text
Datagram 1 → "HELLO"
Datagram 2 → "WORLD"
```

This makes UDP useful for protocols where individual messages/datagrams
matter.

------------------------------------------------------------------------

# 25. UDP and Packet Size

UDP is message-oriented, but large UDP datagrams can encounter problems
when they exceed the network path's **MTU (Maximum Transmission Unit)**.

Fragmentation can have undesirable effects:

-   if one fragment is lost, the whole datagram may become unusable
-   fragmentation adds overhead
-   large datagrams may be more fragile across networks

Therefore, applications/protocols often prefer appropriately sized
datagrams.

QUIC, for example, is designed with path MTU considerations in mind.

For System Design, you mainly need to understand:

> **UDP does not eliminate the underlying IP network's packet-size
> constraints.**

------------------------------------------------------------------------

# 26. UDP Failure Scenarios

## Packet loss

``` text
Sender
  |
  | UDP Datagram
  X
  |
Receiver
```

UDP does not retransmit it.

The application decides what to do.

------------------------------------------------------------------------

## Packet reordering

``` text
Sender:
1 → 2 → 3

Receiver:
1 → 3 → 2
```

UDP does not reorder them.

The application can add sequence numbers if ordering matters.

------------------------------------------------------------------------

## Packet duplication

The application may receive duplicate datagrams.

If duplicates matter, the application protocol needs a way to detect
them.

------------------------------------------------------------------------

## Network congestion

UDP itself does not provide TCP's congestion-control mechanism.

An aggressive UDP sender can contribute significantly to network
congestion.

Well-designed UDP-based Internet protocols therefore need appropriate
congestion-control behavior.

------------------------------------------------------------------------

## Receiver overload

UDP does not provide TCP's receive-window flow control.

If data arrives faster than the application can process it:

``` text
Packets
   ↓
Receive buffers
   ↓
Buffers fill
   ↓
Packets may be dropped
```

------------------------------------------------------------------------

# 27. UDP and DDoS

UDP's connectionless nature can be abused in denial-of-service attacks.

A **UDP flood** can send large volumes of UDP traffic toward a target.

Conceptually:

``` text
Attacker(s)
   ├──────── UDP ────────┐
   ├──────── UDP ────────┤
   ├──────── UDP ────────┤
   └──────── UDP ────────┘
                          ↓
                    Target / Network
```

Potential consequences:

-   bandwidth exhaustion
-   CPU consumption
-   packet-processing overhead
-   socket/buffer pressure
-   downstream infrastructure overload

UDP can also be involved in **reflection/amplification attacks**, where
attackers exploit third-party services to generate traffic toward a
victim.

------------------------------------------------------------------------

# 28. UDP Flood --- Important Correction

A simplified explanation sometimes says:

> "The target receives UDP packets on random closed ports and sends ICMP
> Destination Unreachable for each one."

That can happen in some situations, but it should not be treated as a
universal behavior.

Modern systems may:

-   rate-limit ICMP responses
-   filter packets
-   drop traffic
-   have no application listening on the port
-   process packets through firewalls/load balancers
-   behave differently depending on network configuration

The System Design lesson is more important:

> **Large volumes of UDP traffic can consume network and
> packet-processing resources even when the target application does not
> accept the traffic.**

------------------------------------------------------------------------

# 29. UDP DDoS Mitigation

Common defenses include:

-   rate limiting
-   firewalls
-   network ACLs
-   traffic filtering
-   upstream filtering
-   DDoS protection services
-   load balancing
-   capacity planning
-   monitoring and alerting

For large Internet-facing systems, DDoS protection is often handled
upstream by specialized infrastructure rather than by the application
server alone.

------------------------------------------------------------------------

# 30. UDP and Backend Architecture

UDP can appear in different parts of a real architecture.

For example:

``` text
              Clients
                 |
                UDP
                 ↓
          ┌──────────────┐
          │ UDP Gateway  │
          └──────┬───────┘
                 |
              Backend
                 |
       ┌─────────┴─────────┐
       ↓                   ↓
    Database            Cache
```

Potential use cases include:

-   real-time gaming
-   media/voice systems
-   DNS
-   network-control systems
-   QUIC/HTTP/3
-   telemetry or specialized protocols

The exact architecture depends heavily on the application's
requirements.

------------------------------------------------------------------------

# 31. UDP at Scale

UDP can be attractive at scale because it does not require TCP-style
per-connection state.

With TCP:

``` text
Millions of clients
        ↓
Potentially millions of TCP connections
        ↓
Connection state/resources
```

UDP can instead work with independent datagrams:

``` text
Millions of datagrams
        ↓
No TCP handshake per sender/receiver pair
```

However, this does **not** mean UDP is free.

Large UDP workloads can still consume:

-   bandwidth
-   CPU
-   kernel/network buffers
-   socket buffers
-   NIC capacity
-   load-balancer resources
-   application processing capacity

So:

> **UDP reduces certain transport overheads; it does not remove
> scalability problems.**

------------------------------------------------------------------------

# 32. UDP and Application-Level Reliability

An application can build reliability over UDP.

For example:

``` text
Application Protocol
 ├── sequence number
 ├── ACK
 ├── retransmission
 ├── timeout
 ├── duplicate detection
 └── congestion control
          ↓
         UDP
          ↓
          IP
```

Suppose:

``` text
Sender:
Message #10

Receiver:
ACK #10
```

If the ACK doesn't arrive:

``` text
Sender
  |
  | Timeout
  ↓
Retransmit #10
```

The application can implement this behavior itself.

But once you start adding enough features, you are effectively building
transport-like functionality.

This is one reason established protocols such as QUIC are valuable.

------------------------------------------------------------------------

# 33. UDP and Reliability Trade-offs

There is no universally correct choice.

Imagine an application with:

``` text
Requirement A:
Every message must arrive.

Requirement B:
Messages must arrive in order.

Requirement C:
Latency must be extremely low.
```

You need to decide which guarantees are actually necessary.

For example:

### File transfer

``` text
Reliability → extremely important
Ordering    → important
Loss        → unacceptable
```

TCP is a natural fit.

### Live voice

``` text
Latency     → extremely important
Small loss  → potentially acceptable
Old data    → may become useless quickly
```

UDP or a UDP-based protocol may be appropriate.

### Online game

``` text
Latency     → very important
State       → frequently updated
Old updates → may become obsolete
```

Some game traffic may benefit from UDP-style datagrams and
application-specific handling.

------------------------------------------------------------------------

# 34. Important Nuance: "UDP Is Faster"

A common statement is:

> "UDP is faster because it has no ACKs or retransmissions."

This is directionally understandable but too simplistic.

UDP can have lower protocol overhead because it does not require TCP's
built-in:

-   connection establishment
-   acknowledgements
-   retransmission
-   ordered byte-stream delivery
-   TCP flow control
-   TCP congestion control

But:

``` text
Lower transport overhead
        ≠
Always lower end-to-end latency
        ≠
Always higher throughput
```

Actual performance depends on:

-   network quality
-   packet loss
-   application behavior
-   congestion
-   implementation
-   payload size
-   protocol design

A UDP application that implements its own reliability may also add
substantial overhead.

------------------------------------------------------------------------

# 35. UDP vs TCP in System Design

Use TCP when you generally need:

-   reliable delivery
-   ordered byte stream
-   retransmission
-   built-in flow control
-   built-in congestion control
-   broad compatibility with existing application protocols

Use UDP when you generally need:

-   datagram/message semantics
-   low transport overhead
-   low latency
-   application-specific handling of loss/order/retransmission
-   no TCP connection setup
-   a foundation for a protocol such as QUIC

Do not choose UDP merely because:

> "UDP is faster."

Choose it because its **transport semantics match the application**.

------------------------------------------------------------------------

# 36. TCP vs UDP --- Example

Imagine two applications.

## Application A --- File Download

``` text
File:
A B C D E F G H
```

If `D` is lost:

``` text
A B C _ E F G H
```

The application needs the complete file.

It makes sense to use reliable transport.

``` text
TCP
→ retransmit D
→ preserve ordering
→ deliver complete stream
```

------------------------------------------------------------------------

## Application B --- Live Voice

Suppose:

``` text
Audio:
A B C D E F G
```

If `D` is lost:

``` text
A B C _ E F G
```

Waiting for `D` may cause a noticeable delay.

The application may prefer:

``` text
A B C [small gap] E F G
```

rather than delaying the entire conversation.

This illustrates:

``` text
Reliability vs latency
```

rather than simply:

``` text
TCP = good
UDP = bad
```

------------------------------------------------------------------------

# 37. UDP and Time-Sensitive Data

A key concept is **data freshness**.

Consider a game player's position:

``` text
t=1 → x=100
t=2 → x=105
t=3 → x=110
```

Suppose the t=2 update is lost.

By t=3, retransmitting the old position may not be useful.

The newest state is more valuable:

``` text
Old update
→ less useful

Latest update
→ more useful
```

This is one reason datagram-oriented communication can be useful in
real-time systems.

------------------------------------------------------------------------

# 38. UDP and Idempotency

UDP itself does not solve duplicate requests.

If an application sends:

``` text
Request #123
```

and it is duplicated:

``` text
Request #123
Request #123
```

the application may process it twice unless it has duplicate detection.

For operations such as payments or order creation, this is dangerous.

The application may use:

-   request IDs
-   idempotency keys
-   sequence numbers
-   deduplication

Again:

``` text
Transport behavior
        ≠
Business correctness
```

------------------------------------------------------------------------

# 39. UDP and Timeouts

Because UDP does not provide TCP's connection semantics, applications
using UDP often need their own handling for:

-   response timeouts
-   retries
-   stale data
-   duplicate packets
-   missing packets

Example:

``` text
Client
  |
  | UDP request
  ↓
Server
  |
  X response lost
  |
Client timeout
```

The application must decide whether to:

``` text
retry
ignore
fallback
return error
```

This is a distributed-systems decision, not something UDP decides for
you.

------------------------------------------------------------------------

# 40. UDP and Observability

At scale, UDP systems need visibility into:

-   packet loss
-   latency
-   jitter
-   throughput
-   dropped packets
-   receive-buffer overflow
-   retransmission if implemented by the application/protocol
-   congestion
-   error rates

For real-time systems, **jitter** is particularly important.

### Jitter

Jitter means variation in packet arrival timing.

Example:

``` text
Packet 1 → 20 ms
Packet 2 → 21 ms
Packet 3 → 80 ms
Packet 4 → 22 ms
```

The average latency alone does not fully describe the user experience.

This is particularly relevant to:

-   VoIP
-   gaming
-   real-time media

------------------------------------------------------------------------

# 41. Common Misconceptions

## Misconception 1: "UDP has no error checking."

False.

UDP has a checksum.

What it lacks is TCP-style **reliable recovery**.

------------------------------------------------------------------------

## Misconception 2: "UDP guarantees packets arrive."

False.

Datagrams may be lost.

------------------------------------------------------------------------

## Misconception 3: "UDP guarantees ordering."

False.

Datagrams may arrive out of order.

------------------------------------------------------------------------

## Misconception 4: "UDP retransmits lost packets."

False.

UDP itself does not retransmit.

An application/protocol built over UDP can implement retransmission.

------------------------------------------------------------------------

## Misconception 5: "UDP has no state anywhere."

Too broad.

UDP is connectionless at the transport level, but applications using UDP
can maintain state.

------------------------------------------------------------------------

## Misconception 6: "UDP is always faster than TCP."

Too simplistic.

UDP has lower transport overhead and different semantics, but actual
end-to-end performance depends on the system.

------------------------------------------------------------------------

## Misconception 7: "UDP is only used for gaming and video."

False.

Examples include:

-   DNS
-   DHCP
-   NTP
-   RIP
-   QUIC
-   VoIP
-   gaming
-   real-time media

------------------------------------------------------------------------

## Misconception 8: "If UDP doesn't provide reliability, it is useless for reliable systems."

False.

Protocols such as QUIC implement reliability and other transport
functionality above UDP.

------------------------------------------------------------------------

## Misconception 9: "UDP is automatically suitable for streaming."

Not necessarily.

Streaming systems can use several transport/application combinations.

The requirements determine the choice.

------------------------------------------------------------------------

## Misconception 10: "UDP packets cannot be lost if the network is reliable."

False.

IP networks provide best-effort delivery.

UDP does not add a reliability guarantee.

------------------------------------------------------------------------

# 42. Important Corrections / Clarifications from the Source Article

### ⚠️ "UDP does not guarantee error checking."

This is inaccurate.

UDP has a checksum for error detection.

Correct statement:

> UDP does not provide reliable error recovery, retransmission, or
> ordering. It does have a checksum for detecting corruption.

------------------------------------------------------------------------

### ⚠️ "UDP relies on IP/ICMP for error reporting."

This should not be interpreted as UDP gaining TCP-like reliability from
ICMP.

ICMP can communicate certain network errors, but:

``` text
ICMP error reporting
        ≠
UDP reliable delivery
```

UDP itself still does not retransmit lost datagrams.

------------------------------------------------------------------------

### ⚠️ "Pseudo header ensures delivery to the correct host."

Not literally.

IP addressing/routing handles delivery.

The pseudo header contributes IP-level information to UDP checksum
validation.

------------------------------------------------------------------------

### ⚠️ "UDP flood always causes ICMP Destination Unreachable responses."

Not universally.

ICMP responses may be filtered, rate-limited, or absent depending on the
system and network.

The important security lesson is that high-volume UDP traffic can
exhaust network or packet-processing resources.

------------------------------------------------------------------------

# 43. What You Need to Understand Deeply

### Must understand deeply

1.  UDP is a **Transport-layer protocol**.
2.  UDP is **connectionless**.
3.  UDP is **datagram-oriented**.
4.  UDP has an **8-byte header**.
5.  UDP uses **source and destination ports**.
6.  UDP has a **checksum**.
7.  UDP does not provide built-in:
    -   reliable delivery
    -   ordering
    -   retransmission
    -   TCP-style flow control
    -   TCP-style congestion control
8.  UDP is useful when:
    -   latency matters
    -   some loss may be acceptable
    -   datagram semantics are useful
    -   application-specific reliability is desirable
9.  UDP does not automatically mean "faster."
10. UDP does not automatically mean "unreliable application."
11. Applications can implement reliability above UDP.
12. QUIC is a major modern example of this.
13. UDP can still face:

-   packet loss
-   reordering
-   duplication
-   congestion
-   receiver overload
-   DDoS traffic

14. UDP and TCP solve different transport requirements.

### Useful but don't over-study initially

-   Exact UDP pseudo-header format
-   Exact checksum algorithm
-   Historical routing protocol details
-   Every UDP application
-   Detailed UDP flood mechanics
-   Low-level kernel socket implementation

------------------------------------------------------------------------

# 44. Important Terminology

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  UDP                                 User Datagram Protocol

  Datagram                            Independent UDP message

  Connectionless                      No TCP-style connection
                                      establishment

  Port                                Identifies an application endpoint

  Source port                         Sender-side UDP port

  Destination port                    Receiver-side UDP port

  Checksum                            Detects corruption/misdelivery

  Pseudo header                       IP information used in UDP checksum
                                      calculation

  Packet loss                         Datagram fails to reach the
                                      application

  Reordering                          Datagrams arrive in a different
                                      order

  Duplication                         Same datagram may be observed more
                                      than once

  Retransmission                      Sending missing data again

  Flow control                        Regulating data according to
                                      receiver capacity

  Congestion control                  Regulating traffic according to
                                      network conditions

  MTU                                 Maximum Transmission Unit

  Jitter                              Variation in packet arrival timing

  QUIC                                Modern transport protocol built
                                      over UDP

  HTTP/3                              HTTP protocol using QUIC

  VoIP                                Voice over IP

  DNS                                 Domain Name System

  DHCP                                Dynamic Host Configuration Protocol

  NTP                                 Network Time Protocol

  DDoS                                Distributed Denial of Service
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 45. System Design Interview Questions

1.  What is UDP?
2.  Why is UDP called connectionless?
3.  What is the difference between UDP and TCP?
4.  What does UDP guarantee?
5.  What does UDP not guarantee?
6.  Does UDP perform error checking?
7.  Explain the UDP header.
8.  Why is the UDP header only 8 bytes?
9.  What are UDP ports used for?
10. What is a UDP checksum?
11. What is the UDP pseudo header?
12. Why might an application choose UDP over TCP?
13. Why can packet loss be acceptable for VoIP?
14. Why might packet loss be less acceptable for file transfer?
15. What happens when a UDP datagram is lost?
16. Does UDP retransmit lost packets?
17. Does UDP preserve ordering?
18. Can UDP packets be duplicated?
19. Can an application build reliable communication on UDP?
20. Why does QUIC use UDP?
21. What is the relationship between QUIC and HTTP/3?
22. Why can UDP be useful for gaming?
23. Why is UDP useful for DNS?
24. What is the difference between UDP's lack of reliability and an
    unreliable application?
25. What problems can occur if a UDP sender sends too much traffic?
26. How can a UDP service be protected from UDP floods?
27. What is jitter?
28. What is MTU and why does it matter for UDP?
29. How does UDP interact with IP?
30. What happens at the receiver when a UDP datagram arrives?
31. Why doesn't UDP need a TCP-style handshake?
32. Why doesn't "UDP is faster" fully explain when to use UDP?
33. How would you implement request IDs and duplicate detection over
    UDP?
34. How would you implement retransmission over UDP if the application
    needed it?
35. What are the trade-offs of implementing reliability yourself?

------------------------------------------------------------------------

# 46. Why / What If Questions

### Why?

If UDP does not guarantee delivery, why is it useful?

### What if?

A voice packet is lost and can only be retransmitted after 300 ms.

Would retransmitting it necessarily improve the user's experience?

### Why?

Why might an online game prefer a newer position update over
retransmitting an old one?

### What if?

UDP packets arrive in this order:

``` text
1 → 3 → 2 → 4
```

What would UDP itself do?

### Why?

Why does UDP need a checksum if it does not provide reliable delivery?

### What if?

A UDP application needs reliable delivery.

What mechanisms would you need to add above UDP?

### Why?

Why does QUIC use UDP if QUIC itself provides reliability?

### What if?

A UDP server receives packets faster than its application can process
them.

What could happen?

### Why?

Why can a large UDP datagram be problematic on a network with a smaller
MTU?

### What if?

A payment request sent using UDP is duplicated.

What prevents the payment from being processed twice?

### Why?

Why is this a business/application problem rather than a UDP problem?

### What if?

A UDP-based real-time system has:

``` text
Average latency = 30 ms
```

but some packets take:

``` text
300 ms
```

Is average latency enough to describe the system?

What other metric matters?

### Why?

Why can UDP be involved in DDoS attacks?

### What if?

A system implements reliability, ordering, acknowledgements,
retransmission, and congestion control on top of UDP.

What kind of protocol is it beginning to resemble?

------------------------------------------------------------------------

# 47. Key Takeaways

1.  **UDP is a Transport-layer protocol.**
2.  UDP is **connectionless**.
3.  UDP is **datagram-oriented**, unlike TCP's byte-stream model.
4.  UDP has a small **8-byte header**.
5.  UDP uses **source and destination ports**.
6.  UDP has a **checksum for error detection**.
7.  UDP does not provide TCP-style:
    -   reliable delivery
    -   ordered delivery
    -   retransmission
    -   flow control
    -   congestion control
8.  UDP can be useful when **latency and simplicity matter more than
    guaranteed delivery**.
9.  UDP does not inherently mean "faster"; it means **different
    transport semantics and less built-in machinery**.
10. UDP does not inherently mean an application must be unreliable.
11. Reliability can be implemented above UDP when required.
12. **QUIC** is an important example of a sophisticated protocol built
    over UDP.
13. **HTTP/3 uses QUIC over UDP.**
14. UDP is commonly associated with DNS, DHCP, NTP, VoIP, gaming,
    real-time media, and QUIC.
15. UDP can experience loss, reordering, duplication, congestion,
    receiver overload, and DDoS traffic.
16. UDP is not a replacement for TCP in every situation.
17. The correct System Design question is: \> **What transport
    guarantees does the application need?**
18. Transport-level behavior is separate from application/business
    correctness.
19. If an operation must not be duplicated, concepts such as
    **idempotency keys, request IDs, and deduplication** may be required
    regardless of transport.
20. For real-time systems, **latency, jitter, freshness, and packet
    loss** can matter more than perfect delivery.

------------------------------------------------------------------------

# 48. TCP + UDP --- Final Mental Model

You should now think of TCP and UDP as two different transport choices:

``` text
                     Application
                         |
              ┌──────────┴──────────┐
              ↓                     ↓
             TCP                   UDP
              |                     |
      Reliable byte stream     Independent datagrams
      Ordered                  No built-in ordering
      Retransmission           No built-in retransmission
      Flow control             No TCP-style flow control
      Congestion control       No TCP-style congestion control
              |                     |
              └──────────┬──────────┘
                         ↓
                         IP
                         ↓
                    Network Access
```

A useful decision framework:

``` text
Need reliable ordered stream?
        |
       YES
        ↓
       TCP
```

``` text
Need datagrams + low transport overhead +
application-specific handling of loss/order?
        |
       YES
        ↓
       UDP
```

But modern systems add another important option:

``` text
Need modern multiplexed reliable transport
with different behavior from TCP?
        |
       YES
        ↓
      QUIC
        ↓
       UDP
```

The key distinction is:

``` text
TCP:
"Transport reliability is built into the transport."

UDP:
"Transport gives me minimal guarantees;
 the application/protocol can decide what it needs."

QUIC:
"Build a modern transport protocol over UDP."
```

And for System Design:

``` text
TCP/UDP choice
      ↓
Latency
      ↓
Reliability
      ↓
Packet loss
      ↓
Connection/state overhead
      ↓
Scalability
      ↓
Application semantics
      ↓
Business correctness
```

That is the level at which you should reason about UDP when designing
backend systems.
