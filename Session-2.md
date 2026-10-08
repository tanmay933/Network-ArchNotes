# Session 2 — Computer Networks
## High-ROI Notes for MCQs + Interviews

> **Goal:** understand the mechanisms, not memorize isolated facts.  
> These notes follow Session 2 closely, but compress the material into a cleaner revision/interview format.

---

# 0. Session Map

Session 2 connects five ideas:

1. **Sockets** — how applications reach TCP.
2. **OSI + encapsulation** — how data is wrapped as it moves through layers.
3. **SS7** — an older signalling stack that solves reliability/control differently.
4. **TCP / UDP / QUIC** — transport, handshakes, teardown, congestion and latency.
5. **Text protocols** — SMTP, POP3, IMAP and FTP, with framing in real protocols.

### The one idea tying the session together

**One layer's headers become another layer's payload/body.**

Examples:

```text
Ethernet → IP → TCP → HTTP
MTP2 → MTP3 → SCCP → TCAP → MAP → TPDU
```

And at the application boundary:

```text
TCP = byte stream
Application protocol = must decide how bytes form messages
```

---

# 1. Sockets

## 1.1 TCP server lifecycle

A minimal TCP server follows:

```text
socket → bind → listen → accept → read → write → close
```

### What each call does

| Call | Purpose |
|---|---|
| `socket()` | Create a socket/file descriptor |
| `bind()` | Attach server socket to local IP + port |
| `listen()` | Mark TCP socket as passive/listening |
| `accept()` | Accept a connection and obtain a new client socket |
| `read()` | Receive bytes |
| `write()` | Send bytes |
| `close()` | Close the socket |

**Important:** the listening socket and the connected client socket are different descriptors.

### `listen(fd, 1)`

The `1` in the session example is the **accept/backlog queue depth**, not a hard limit of one total connection.

This becomes relevant to SYN floods and SYN cookies.

### `htons()`

`htons` = **host-to-network short**.

- Network byte order = big-endian.
- Used for 16-bit values such as TCP/UDP ports.
- `htonl()` is for 32-bit values.
- Reverse direction: `ntohs()` / `ntohl()`.

---

## 1.2 TCP client

The client uses:

```text
socket → connect → read/write → close
```

Unlike the server, it normally does **not** explicitly:

```text
bind()
listen()
accept()
```

The kernel chooses an **ephemeral source port** for the client.

`connect()` triggers TCP connection establishment.

### SO_LINGER `{1,0}`

Normal `close()` performs a graceful TCP shutdown:

```text
FIN
```

With:

```c
struct linger l = {1, 0};
```

the session demonstrates an **abortive close**:

```text
RST
```

Effects:

- immediate teardown
- unsent data can be discarded
- avoids the normal TIME_WAIT behavior shown in the session
- useful for demonstrating TCP behavior
- **not a normal production strategy**

---

# 2. OSI Model

## 2.1 Seven layers

| Layer | Name | Examples / responsibility |
|---|---|---|
| L7 | Application | HTTP, FTP, DNS, SMTP |
| L6 | Presentation | encryption, compression, encoding |
| L5 | Session | session management, authentication |
| L4 | Transport | TCP, UDP |
| L3 | Network | IP, ICMP, routing |
| L2 | Data Link | Ethernet, MAC, ARP, switches |
| L1 | Physical | cables, radio, signals |

### MCQ anchors

```text
MAC       → L2
Ethernet  → L2
IP        → L3
ICMP      → L3
Port      → L4
TCP/UDP   → L4
HTTP      → L7
```

The OSI model is a conceptual model. Real implementations do not always map perfectly one protocol → one OSI layer.

---

# 3. Encapsulation

As data moves downward:

```text
Application data
      ↓
TCP segment / UDP datagram
      ↓
IP packet
      ↓
Ethernet frame
      ↓
Physical bits
```

At the receiver, the process is reversed.

### The key mental model

A layer does not need to understand the entire payload it carries.

For example:

```text
Ethernet payload = entire IP packet
IP payload       = entire TCP segment
TCP payload      = application bytes
```

This is why the session says:

> **One layer's headers are another layer's body.**

---

# 4. L2 / L3 Details

## 4.1 Ethernet

Typical Ethernet II frame:

```text
Destination MAC   6 B
Source MAC        6 B
EtherType         2 B
Payload           46–1500 B
FCS               4 B
```

Maximum frame in the session's example:

```text
14 B header
+ 1500 B payload
+ 4 B FCS
= 1518 B
```

### EtherType

IPv4 is represented by:

```text
0x0800
```

---

## 4.2 IPv4 packet

The session's typical IPv4 header is **20 bytes**.

Important fields:

- Version / IHL
- Total Length
- TTL
- Protocol
- Header checksum
- Source IP
- Destination IP

`Protocol = 6` identifies TCP.

### TTL

IPv4 TTL is reduced at each router.

Purpose:

**prevent packets from circulating forever in routing loops.**

---

## 4.3 TCP over Ethernet: MSS calculation

For a typical 1500-byte MTU:

```text
1500
- 20 IPv4 header
- 20 TCP header
= 1460 bytes
```

So the typical TCP MSS is:

```text
1460 bytes
```

### Do not confuse

**MTU** = maximum IP packet size that fits in the stated Ethernet payload.

**MSS** = maximum TCP application payload in one TCP segment under the stated assumptions.

---

# 5. Ethernet vs Wi-Fi

The physical medium changes the L1/L2 problem, while L3 and above can remain unchanged.

## Ethernet

Collision detection historically used:

```text
CSMA/CD
```

## Wi-Fi

Uses:

```text
CSMA/CA
```

Why not collision detection?

A radio cannot reliably listen for another transmission while it is transmitting; its own signal overwhelms what it is trying to hear.

So Wi-Fi uses:

```text
sense
→ defer if busy
→ random backoff
→ transmit
→ wait for ACK
→ retransmit/back off if needed
```

The session highlights roughly **2 ms of contention delay** before a frame can leave in its example.

### Key distinction

```text
Ethernet → detect collisions
Wi-Fi    → avoid collisions
```

---

# 6. ARP, Routing and ICMP

## 6.1 ARP

ARP maps:

```text
IPv4 address → MAC address
```

### Local destination

If the destination is on the same local network:

```text
ARP for destination's MAC
```

### Remote destination

The host normally does **not** ARP for the remote server.

It ARPs for:

```text
default gateway's MAC
```

The router then handles the next hop.

### Switch

A switch primarily forwards Ethernet frames using:

```text
MAC addresses
```

### Router

A router forwards packets using network-layer information, especially:

```text
destination IP
```

---

## 6.2 ICMP

ICMP is used for network control and diagnostics.

Examples:

```text
ping             → Echo Request / Echo Reply
TTL expires      → Time Exceeded
unreachable host → ICMP error
```

### Trap

ICMP is **not TCP or UDP** and does not use transport-layer ports.

---

# 7. Starlink / Layering Example

The session uses satellite networking to show why layering matters.

A packet can traverse:

```text
Laptop
→ Ethernet / Wi-Fi
→ Starlink terminal
→ RF uplink
→ LEO satellite
→ laser crosslink
→ ground gateway
→ Internet
```

The physical path can change dramatically, yet L3 and above do not need to know that the packet went through space.

The session's example uses a satellite around **550 km** altitude.

### Principle

**L3 and above are insulated from changes in the physical/link implementation.**

---

# 8. SS7

SS7 is a telecommunications signalling system developed long before today's TCP/IP Internet.

Its major idea:

> **Move signalling/control out of the voice channel.**

## 8.1 Before SS7

In-band signalling mixed:

```text
voice + control tones
```

This meant control information travelled through the same channel as the conversation.

## 8.2 SS7

SS7 uses a separate signalling network.

```text
Voice bearer
64 kbps DS0

        separate from

Signalling network
MTP / SCCP / TCAP / MAP / ISUP
```

This is similar to the modern **control-plane vs data-plane** separation idea.

---

# 9. SS7 Stack

Think of SS7 as a **whole stack**, not as one protocol.

| Function | SS7 | Rough TCP/IP counterpart |
|---|---|---|
| L7 | MAP, INAP, CAP, ISUP | HTTP, DNS, SMTP, SIP |
| L5–L6 | TCAP | application/TLS framing |
| L4 | SCCP | TCP/UDP |
| L3 | MTP3 | IP |
| L2 | MTP2 | Ethernet/PPP/HDLC |
| L1 | MTP1 | physical medium |

### Important

**TCP is one L4 protocol. SS7 is a complete protocol stack.**

---

# 10. SS7 Encapsulation

The session's SMS example shows deep nesting:

```text
MTP2
  → MTP3
    → SCCP
      → TCAP
        → Invoke
          → MAP
            → SMS-SUBMIT
              → TP-UD
```

The exact names are less important than the pattern:

```text
outer header
    contains
       inner protocol message
          contains
             application payload
```

This is the same structural idea as:

```text
Ethernet → IP → TCP → HTTP
```

---

# 11. Why SMS Was 140 Bytes

The session explains the limit as stacked protocol constraints.

Approximate chain:

```text
MTP2 SIF        → 272/273-octet ceiling in the session material
MAP field       → ~200 octets
SMS TP-UD       → 140 octets
```

The **smallest relevant limit wins**.

The design avoided segmentation/reassembly for the SMS payload in this system.

### Important concept

The 140-byte SMS limit was not an arbitrary application decision. It was tied to constraints in the underlying signalling stack.

---

# 12. GSM-7 Encoding

Why can an SMS contain **160 characters in 140 bytes**?

Because GSM-7 uses:

```text
7 bits / character
```

Available bits:

```text
140 × 8 = 1120 bits
```

Characters:

```text
1120 / 7 = 160
```

So:

```text
140 bytes → 160 GSM-7 characters
```

The characters are packed continuously into bytes.

### Example from the session

```text
12 chars × 7 bits = 84 bits
```

which needs:

```text
11 octets = 88 bits
```

with padding.

### Unicode / UCS-2

The session notes:

```text
140 / 2 = 70 characters
```

when using UCS-2.

This is why non-Latin text/emoji can reduce the number of characters that fit in one SMS.

---

# 13. SMS Signalling Flow

High-level flow:

```text
Phone A
  ↓
serving switch
  ↓
SMSC
  ↓
HLR lookup
  ↓
destination serving MSC
  ↓
Phone B
```

The important architectural idea:

**SMSC provides store-and-forward behavior.**

Once the message is accepted and queued, delivery becomes the SMSC's responsibility.

This is why SMS can continue to work even though the sender and receiver are not simultaneously connected.

---

# 14. SS7 ISUP: Phone Calls

SS7 signalling sets up and tears down the call; the voice itself uses a separate bearer.

Main signalling sequence:

```text
IAM  → Initial Address Message
ACM  → Address Complete
ANM  → Answer Message
REL  → Release
RLC  → Release Complete
```

Conceptually:

```text
setup signalling
→ voice conversation
→ teardown signalling
```

During the actual conversation, the SS7 signalling path is not carrying the voice.

---

# 15. SIP / VoIP

The session contrasts old SS7 telephony with modern IP-based signalling.

SIP is text-shaped signalling over IP.

Typical flow:

```text
INVITE
→ 100 Trying
→ 180 Ringing
→ 200 OK
→ ACK
```

Media then uses:

```text
RTP / SRTP
```

typically over UDP.

### Important separation

```text
SIP  → signalling/control
RTP  → media
```

This repeats the same control/data separation idea seen in SS7.

---

# 16. SS7 vs TCP: Where Reliability Lives

This is one of the highest-value conceptual comparisons.

### SS7

Reliability is primarily:

```text
per hop
```

Adjacent signalling points handle reliability.

### TCP

Reliability is:

```text
end to end
```

The endpoints provide the reliable byte stream across arbitrary intermediate networks.

### Session comparison

| Property | SS7 | TCP |
|---|---|---|
| Sequencing | FSN | 32-bit sequence numbers |
| ACK | BSN/BIB | ACK numbers / SACK |
| Retransmission | signalling-specific mechanisms | RTO / fast retransmit |
| Flow control | SIB | sliding window |
| Error detection | CRC-16 | checksum |
| Addressing | point codes / SSN / GT | IP + port |
| Setup | SCCP classes as needed | 3-way handshake |
| Congestion | hop-by-hop | end-to-end |
| Reliability scope | per hop | end to end |

### Interview insight

There is no universal rule saying "reliability belongs at L4."

It can be placed where the system's design needs it.

---

# 17. TCP

TCP provides:

- connection-oriented communication
- reliable delivery
- ordered delivery
- byte-stream abstraction
- retransmission/error recovery
- flow control
- congestion control

Cost:

- more protocol state
- more overhead
- connection setup latency

---

# 18. TCP 3-Way Handshake

```text
Client                         Server

SYN  ------------------------>

       <---------------------- SYN + ACK

ACK  ------------------------>
```

### Why three messages?

Each side needs to establish/confirm the other's sequence space.

Example from the session:

```text
Client ISN = 1000
Server ISN = 5000

SYN        seq=1000
SYN+ACK    seq=5000, ack=1001
ACK        ack=5001
```

### Critical MCQ fact

**SYN consumes one sequence number.**

Therefore:

```text
seq=1000
→ next expected sequence = 1001
```

The same rule applies to FIN:

**FIN also consumes one sequence number.**

---

# 19. SYN Floods and SYN Cookies

Between:

```text
SYN
```

and:

```text
final ACK
```

the server normally has state for a half-open connection.

An attacker can send many spoofed SYNs and never complete the handshake.

Result:

```text
backlog fills
→ legitimate connections suffer
```

## SYN cookies

The server avoids storing all the state in the normal way.

Instead, connection information is encoded into the server's initial sequence number using a keyed construction.

When the client's ACK returns:

```text
ACK
→ server validates cookie
→ reconstructs connection information
```

### Why not always use SYN cookies?

The session notes a tradeoff: limited bits in the ISN mean some TCP options may not survive cleanly under cookie mode.

So the session presents them as a **defence activated under pressure**, not simply "always on."

---

# 20. TCP Close

TCP is bidirectional, so each direction closes independently.

Typical sequence:

```text
Client                     Server

FIN  -------------------->

       <------------------ ACK

       <------------------ FIN

ACK  -------------------->
```

### Half-close

After one side sends FIN:

- it will send no more data
- it can still receive data
- the other side can continue until it sends its own FIN

This is a valid TCP state, not necessarily an error.

---

# 21. Important TCP States

## CLOSE_WAIT

Meaning:

```text
peer has closed
+
local application has not called close()
```

If sockets accumulate in CLOSE_WAIT, suspect an application bug.

The session explicitly frames this as:

**look at your code, not the network.**

## TIME_WAIT

After active connection teardown, TCP retains connection information for:

```text
2 MSL
```

The purpose is to prevent delayed old segments from interfering with a future connection using the same tuple.

### Practical consequence

TIME_WAIT contributes to:

```text
"address already in use"
```

during rapid server restarts.

This is why:

```text
SO_REUSEADDR
```

is commonly useful.

---

# 22. TCP Is a Byte Stream

This is one of the most important facts from Sessions 1–2.

If an application sends:

```text
HELLO
WORLD
```

TCP does **not** promise the receiver will read:

```text
HELLO
WORLD
```

It may see:

```text
HELLOWORLD
```

or:

```text
HEL
LOWORLD
```

or other chunks.

### Therefore

TCP provides:

```text
ordered bytes
```

not:

```text
messages
```

The application protocol must provide framing.

---

# 23. UDP

UDP is intentionally lightweight.

Properties:

- connectionless
- low overhead
- low latency
- no built-in delivery guarantee
- no built-in ordering
- no built-in retransmission
- no TCP-style flow control
- no TCP-style congestion-control machinery

### UDP header

Exactly:

```text
8 bytes
```

Fields:

```text
Source Port       2 B
Destination Port  2 B
Length            2 B
Checksum          2 B
```

### Payload

The session gives a maximum UDP payload of approximately:

```text
65,507 bytes
```

under IPv4 assumptions.

### Important trap

UDP does **not** mean:

> packets will always be lost.

It means:

> UDP itself does not guarantee delivery or ordering.

Common uses from the session:

- DNS
- WebRTC
- gaming
- real-time media / streaming

For real-time media, a retransmitted frame may arrive too late to be useful.

---

# 24. TCP vs UDP

| TCP | UDP |
|---|---|
| Connection-oriented | Connectionless |
| Reliable | No delivery guarantee |
| Ordered byte stream | No ordering guarantee |
| Retransmission | No built-in retransmission |
| Flow control | No built-in TCP flow control |
| Congestion control | No built-in TCP congestion control |
| More overhead/state | Lower overhead |
| Higher setup/latency cost | Lower setup cost |

Do not interpret this as "TCP is always better."

The application determines what it needs.

---

# 25. QUIC + HTTP/3

QUIC essentially rebuilds transport functionality in user space over UDP.

Key properties from the session:

- runs over UDP
- integrates TLS 1.3
- supports independent streams
- used by HTTP/3
- avoids TCP's cross-stream head-of-line blocking problem

---

## 25.1 Why HTTP/2 over TCP can still block

HTTP/2 multiplexes many streams over one TCP connection.

But TCP guarantees one ordered byte stream.

If one TCP packet is lost:

```text
stream A data blocked
+
stream B data also waits
+
stream C data also waits
```

because TCP cannot deliver later bytes before the missing earlier bytes.

This is **cross-stream head-of-line blocking**.

---

## 25.2 QUIC

QUIC gives streams independent sequence spaces.

So:

```text
packet for Stream A lost
→ Stream A waits

Stream B
→ can continue
```

### Why user space?

TCP lives largely in the OS kernel.

Changing TCP behavior requires OS/network-stack deployment.

QUIC can be implemented and updated in user space, allowing faster protocol evolution.

---

# 26. RTT and Connection Setup Cost

Every round trip before application data is a latency tax.

The session's simplified comparison:

```text
TCP             → ~1 RTT before data

TCP + TLS 1.2   → more RTTs

TCP + TLS 1.3   → fewer RTTs than TLS 1.2

TCP Fast Open   → can put data in SYN for suitable returning connections

QUIC / HTTP/3   → transport + TLS setup integrated
```

The exact timing depends on connection state and configuration; the important concept is:

**RTT is a latency constraint, not something bandwidth purchase simply eliminates.**

The session illustrates this with long-distance latency such as Mumbai ↔ Virginia.

---

# 27. BBR

Traditional loss-oriented algorithms such as Reno/CUBIC treat packet loss as a major congestion signal.

The session's simplified picture:

```text
increase rate
→ queue grows
→ packet drops
→ rate decreases
→ repeat
```

This produces the familiar sawtooth behavior.

## BBR

BBR estimates:

```text
bottleneck bandwidth
+
round-trip propagation time
```

Conceptually:

```text
BDP ≈ bottleneck bandwidth × RTT
```

BBR tries to operate near the path's actual capacity while keeping queues smaller.

### Interview idea

**Loss is not always congestion.**

A packet can be lost because of a lossy link, radio conditions, etc.

That is one reason a purely loss-driven congestion signal can be misleading.

---

# 28. Framing

TCP gives a byte stream.

Therefore an application protocol must answer:

> **Where does one message end and the next begin?**

The session presents three broad approaches.

## 28.1 Delimiter-based

Use a special marker.

Examples:

```text
SMTP          → lone "."
HTTP/1.1      → CRLF CRLF between headers and body
MIME          → boundary
Redis         → CRLF
```

### Advantage

Simple and can be human-readable.

### Cost

The delimiter must be escaped or otherwise prevented from being confused with payload.

---

## 28.2 Length-based

Send the length and then exactly that many bytes.

Examples:

```text
HTTP Content-Length
SMS TPDU length
SS7 MTP2 length information
Protobuf length prefix
```

### Advantage

Binary-safe and unambiguous.

### Cost

The receiver needs the length before it knows how much to read.

---

## 28.3 Both

Use structural delimiters plus explicit length.

Examples from the session:

- IMAP literals
- HTTP headers + Content-Length
- HTTP/2 binary framing
- POP3 response + dotted body

### High-value principle

```text
TCP ≠ message framing
```

---

# 29. SMTP

SMTP is a text-based mail transfer protocol.

The session emphasizes:

> The "S" is Simple, not Secure.

## Ports

| Port | Role |
|---|---|
| 25 | server-to-server relay |
| 587 | message submission, commonly STARTTLS |
| 465 | implicit TLS |

---

## 29.1 SMTP conversation

Typical sequence:

```text
EHLO
MAIL FROM:<sender>
RCPT TO:<receiver>
DATA
message headers
blank line
message body
.
QUIT
```

### Envelope vs message

Envelope commands:

```text
MAIL FROM
RCPT TO
```

Message headers inside DATA:

```text
From:
To:
Subject:
```

These **do not have to match**.

This is a major MCQ/interview trap.

---

## 29.2 SMTP framing

The DATA section ends with:

```text
.
```

on a line by itself.

Because the delimiter can appear in the message body, SMTP uses dot-stuffing behavior.

### Blank line

The blank line separates:

```text
headers
```

from:

```text
body
```

---

# 30. SMTP Security / Extensibility

SMTP originally did not provide modern authentication/security guarantees.

The session highlights later additions such as:

```text
STARTTLS
SPF
DKIM
DMARC
```

The protocol stayed text-oriented partly because intermediaries need to inspect, rewrite and add headers.

That also made extensions easier to bolt onto the protocol.

---

# 31. MIME + Base64

Text-oriented mail systems cannot safely carry arbitrary binary bytes directly.

MIME provides a structure for multipart content and binary-to-text transfer encodings.

## Base64

Core conversion:

```text
3 bytes → 4 Base64 characters
```

Therefore the encoded representation is approximately:

```text
4/3 = 1.333...
```

or about:

```text
33% larger
```

### Important

Base64 is:

```text
encoding
```

not:

```text
compression
```

and not:

```text
encryption
```

### MIME boundaries

Multipart messages use a boundary such as:

```text
Content-Type: multipart/mixed; boundary="..."
```

The boundary separates parts.

The session also notes the classic Base64 alphabet:

```text
A-Z
a-z
0-9
+
/
```

with:

```text
=
```

used for padding.

The session mentions a **76-character line length** in the MIME/Base64 discussion.

---

# 32. POP3

POP3 is a mail retrieval protocol.

Session commands:

```text
USER
PASS
STAT
LIST
RETR n
DELE n
QUIT
```

### Ports

```text
110 → POP3
995 → POP3S
```

### Important behavior

`DELE` marks a message for deletion.

The session notes deletion is applied when the POP3 session reaches:

```text
QUIT
```

### Security trap

`USER` and `PASS` are plaintext in basic POP3.

POP3S exists to provide TLS protection.

---

# 33. IMAP vs POP3

This is a high-value comparison.

| POP3 | IMAP |
|---|---|
| Mail primarily downloaded to client | Mailbox state remains on server |
| Simple retrieval model | Rich mailbox management |
| Limited mailbox state | Server-side flags |
| Basic mailbox structure | Server-side folders |
| Download then search locally | Server-side SEARCH |
| Full-message retrieval model | Partial FETCH possible |
| No IDLE-style push | IDLE support |

### Core mental model

```text
POP3 → pull mail to the client

IMAP → manage the mailbox on the server
```

IMAP commonly uses:

```text
143  → IMAP
993  → IMAPS
```

The session's page comparison specifically highlights server-side `\Seen`, `\Answered`, and `\Flagged` state, partial `FETCH`, and `IDLE`.

---

# 34. FTP

FTP deliberately separates:

```text
control connection
+
data connection
```

This is another example of control/data separation.

## Control

Normally:

```text
port 21
```

## Active mode

The client opens the control connection.

The **server initiates the data connection back toward the client**.

Historically the server data side uses port 20.

### Problem

This is difficult with:

```text
NAT
firewalls
```

because the server must connect inward toward the client.

---

## Passive mode

The client opens:

```text
control connection
+
data connection
```

The server tells the client which data port to use.

This is much more NAT/firewall friendly and became the practical default.

### MCQ hook

```text
Active  → server opens data connection
Passive → client opens data connection
```

---

# 35. FTP PASV Port Encoding

The session gives an example:

```text
227 Entering Passive Mode (127,0,0,1,117,48)
```

First four numbers:

```text
IP = 127.0.0.1
```

Last two numbers encode the 16-bit port:

```text
117 × 256 + 48
= 30000
```

This is network byte order expressed as two decimal 8-bit pieces.

### Connection to Session 1

This is the same byte-order concept behind:

```text
htons()
```

---

# 36. High-ROI Numbers

Memorize these.

| Fact | Value |
|---|---:|
| OSI layers | 7 |
| TCP minimum header | 20 B |
| UDP header | 8 B |
| Ethernet MAC | 6 B |
| Ethernet EtherType | 2 B |
| Ethernet FCS | 4 B |
| Typical Ethernet MTU payload | 1500 B |
| Typical max Ethernet frame | 1518 B |
| Typical IPv4 header | 20 B |
| Typical TCP MSS | 1460 B |
| SMTP relay | 25 |
| SMTP submission | 587 |
| SMTPS | 465 |
| POP3 | 110 |
| POP3S | 995 |
| IMAP | 143 |
| IMAPS | 993 |
| FTP control | 21 |
| TCP SYN | consumes 1 sequence number |
| TCP FIN | consumes 1 sequence number |
| TIME_WAIT | 2 MSL in session |
| Base64 | 3 bytes → 4 chars |
| GSM-7 | 7 bits/character |
| SMS TP-UD | 140 octets |
| GSM-7 SMS | 160 chars |
| UCS-2 example | 70 chars |

---

# 37. High-ROI MCQ Traps

1. **MAC → L2**, not L3.
2. **IP → L3**.
3. **Port → L4**.
4. **Ethernet → L2**.
5. **TCP/UDP → L4**.
6. Remote destination? ARP for the **gateway MAC**, not the remote server MAC.
7. Switches primarily use **MAC addresses**.
8. Routers use **destination IP/network information**.
9. ICMP is **not TCP/UDP**.
10. TCP = **ordered byte stream**, not message boundaries.
11. UDP does not guarantee delivery; it does not guarantee loss either.
12. TCP handshake = **SYN → SYN-ACK → ACK**.
13. **SYN consumes one sequence number.**
14. **FIN consumes one sequence number.**
15. CLOSE_WAIT usually means peer closed but local application has not closed.
16. TIME_WAIT is associated with TCP teardown and is **2 MSL** in this session.
17. SYN cookies defend against exhaustion of the half-open connection backlog.
18. QUIC runs over **UDP**.
19. HTTP/3 uses **QUIC**.
20. QUIC streams are independent; TCP is one ordered byte stream.
21. BBR estimates bottleneck bandwidth and RTT rather than treating loss as the sole congestion signal.
22. Base64 is encoding, **not encryption**.
23. Base64 = **3 bytes → 4 characters**.
24. SMTP `MAIL FROM` is envelope information.
25. SMTP `From:` is message-header information.
26. They do **not** have to match.
27. SMTP DATA ends with a **single-dot line**.
28. POP3 `DELE` is applied at `QUIT` in the session example.
29. POP3 stores/retrieves mail primarily on the client side.
30. IMAP keeps mailbox state on the server.
31. FTP has separate control and data connections.
32. Active FTP: **server initiates data connection**.
33. Passive FTP: **client initiates data connection**.
34. TCP itself does **not** provide application framing.
35. Delimiter framing requires care when the delimiter can occur in payload.
36. Length framing is robust for arbitrary binary payloads.
37. Ethernet payload contains the **IP packet**.
38. IP payload contains the **TCP segment**.
39. TCP payload contains **application bytes**.
40. SS7 reliability is primarily **per-hop**; TCP reliability is **end-to-end**.

---

# 38. Interview Mental Models

## If asked: "Why does TCP need framing?"

Answer:

> TCP exposes a reliable ordered byte stream, not message boundaries. The application therefore needs a delimiter, length prefix, fixed-size format, or some combination to reconstruct messages.

---

## If asked: "Why does HTTP/2 still have head-of-line blocking?"

Answer:

> HTTP/2 multiplexes streams, but they share one TCP byte stream. TCP cannot deliver later bytes before an earlier lost byte, so a loss can stall multiple HTTP/2 streams.

---

## If asked: "How does QUIC improve this?"

Answer:

> QUIC runs over UDP and implements independent streams in user space. Loss on one stream does not require unrelated streams to wait, and QUIC integrates TLS 1.3.

---

## If asked: "Why is passive FTP more firewall-friendly?"

Answer:

> In passive FTP the client initiates both control and data connections, so the firewall/NAT does not need to accept a new inbound data connection initiated by the server.

---

## If asked: "Why can SMS fit 160 characters into 140 bytes?"

Answer:

> GSM-7 uses 7 bits per character. 140 bytes = 1120 bits, and 1120 / 7 = 160 characters.

---

## If asked: "Why was SS7 designed with a separate signalling network?"

Answer:

> To separate control/signalling from the voice bearer. SS7 moved signalling out of the voice channel, allowing call setup, routing and other control operations to use a dedicated packet-switched signalling network.

---

## If asked: "What is the difference between TCP and SS7 reliability?"

Answer:

> SS7 places reliability primarily between adjacent signalling points, while TCP provides reliability end-to-end between the communicating endpoints.

---

# 39. Final Mental Map

Remember the session as four recurring design decisions:

### 1. Encapsulation

```text
Outer protocol
    contains
inner protocol
    contains
payload
```

### 2. Framing

```text
How do I know where a message ends?

→ delimiter
→ length
→ both
```

### 3. Reliability

```text
SS7 → per hop
TCP  → end to end
QUIC → transport logic in user space
```

### 4. Control vs Data

```text
SS7 → signalling / voice
SIP  → signalling / RTP media
FTP  → control / data connection
SMTP → envelope / message content
```

These are not four unrelated topics. They are repeated network-design patterns.

---

# 40. 30-Second Revision

```text
OSI:
L7 app
L6 presentation
L5 session
L4 TCP/UDP
L3 IP/ICMP
L2 Ethernet/MAC/ARP
L1 physical

Encapsulation:
Ethernet > IP > TCP > HTTP

TCP:
reliable + ordered + byte stream
SYN → SYN-ACK → ACK
FIN consumes sequence number
CLOSE_WAIT = peer closed, app hasn't
TIME_WAIT = 2 MSL in session

UDP:
8-byte header
connectionless
no built-in reliability/order

QUIC:
UDP + TLS 1.3
independent streams
HTTP/3

Framing:
delimiter / length / both

Ethernet:
1500 MTU
20 IP + 20 TCP → 1460 MSS
1518-byte max frame in example

SS7:
separate signalling network
per-hop reliability
SMS = 140 octets
GSM-7 = 160 chars

SMTP:
25 relay
587 submission
465 implicit TLS
MAIL FROM ≠ necessarily From:
"." ends DATA

POP3:
110
995 TLS
DELE → applied at QUIT

IMAP:
143
993 TLS
server-side mailbox state

FTP:
21 control
active → server opens data
passive → client opens data
```

---

# Source Scope

These notes are derived from **Scaler Session 2 — Computer Networks** and the supplied Session 2 markdown. The focus is deliberately on the concepts, comparisons, packet structures, protocol behavior, numerical facts, and traps that are useful for both MCQs and later interview revision.

The PDF's final "What to Keep" section reinforces the four recurring ideas used here: **encapsulation, framing, location of reliability, and separation of control/data**.
