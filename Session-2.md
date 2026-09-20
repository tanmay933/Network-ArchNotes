# Session 2 — Computer Networks

## MCQ Preparation Notes

**Purpose:** Conceptual preparation for single-correct and multiple-correct MCQs.  
**Focus:** Understand distinctions, packet flow, protocol behavior, numbers, states, and common traps. No code required.

---

# 1. THEORY

## 1.1 OSI Model

The seven OSI layers:

| Layer | Name | Key Concepts |
|---|---|---|
| L7 | Application | HTTP, FTP, DNS, SMTP |
| L6 | Presentation | Encryption, compression, encoding |
| L5 | Session | Session management, authentication |
| L4 | Transport | TCP, UDP |
| L3 | Network | IP, ICMP, routing |
| L2 | Data Link | Ethernet, MAC, ARP, switches |
| L1 | Physical | Bits, signals, physical medium |

### Core MCQ idea

- **IP address → L3**
- **Port → L4**
- **MAC address → L2**
- **Ethernet → L2**
- **TCP/UDP → L4**

A packet is encapsulated as it moves down the stack.

Typical view:

```text
Application data
      ↓
TCP/UDP segment/datagram
      ↓
IP packet
      ↓
Ethernet frame
      ↓
Physical bits
```

---

## 1.2 IP, Routing, ARP and ICMP

### IP

IP provides logical addressing and packet forwarding between networks.

IP is **best effort**:
- No built-in guarantee of delivery
- No built-in ordering
- No TCP-style retransmission

### Routing

Routers use the destination IP to decide where to forward a packet.

```text
Host → Router → Router → Router → Destination
```

Each router represents another hop.

### TTL

IPv4 TTL is reduced at every router.

Purpose: prevent packets from looping forever.

### ARP

ARP resolves:

```text
IPv4 address → MAC address
```

For a destination on the local network, the host can ARP for the destination's MAC.

For a destination on another network, the host normally ARPs for the **default gateway's MAC**, not the remote server's MAC.

### Switch

A switch primarily operates at L2 and forwards Ethernet frames using MAC addresses.

### ICMP

ICMP is used for network control and diagnostics.

Examples:
- Ping → Echo Request / Echo Reply
- TTL expiration → Time Exceeded
- Unreachable destinations → ICMP error messages

ICMP is not TCP or UDP and does not use transport-layer ports.

---

## 1.3 TCP

TCP is:

- Connection-oriented
- Reliable
- Ordered
- Byte-stream oriented
- Provides retransmission/error recovery
- Provides flow control
- Provides congestion control
- Has more overhead/latency than UDP

### TCP 3-Way Handshake

```text
Client                 Server

  SYN ----------------->
      <---------------- SYN + ACK
  ACK ----------------->
```

The handshake establishes the TCP connection and synchronizes sequence numbers.

### Important trap

A SYN consumes one sequence number.

### TCP Close

Normally four segments are involved because each direction closes independently:

```text
FIN  →
     ← ACK
     ← FIN
ACK  →
```

### Important TCP states

**CLOSE_WAIT**
- Remote peer has closed its side.
- Local application has not yet closed its socket.

**TIME_WAIT**
- Appears after connection closure.
- TCP keeps the connection information temporarily.
- The Session 2 material describes this as 2 MSL.

### TCP is a byte stream

TCP does **not** preserve application message boundaries.

If an application sends:

```text
HELLO
WORLD
```

the receiver may observe:

```text
HELLOWORLD
```

or partial chunks.

Applications therefore need framing when they need message boundaries.

---

## 1.4 UDP

UDP is:

- Connectionless
- Low overhead
- Low latency
- No delivery guarantee
- No ordering guarantee
- No built-in retransmission

### UDP Header

UDP header = **8 bytes**

Fields:

- Source port
- Destination port
- Length
- Checksum

### Important trap

UDP does **not** mean packets will always be lost.

It means UDP itself does not guarantee delivery or ordering.

Typical uses include:
- DNS
- Real-time media
- Gaming
- WebRTC

For real-time data, a late retransmission can sometimes be less useful than simply dropping the old data.

---

## 1.5 TCP vs UDP

| TCP | UDP |
|---|---|
| Connection-oriented | Connectionless |
| Reliable delivery | No delivery guarantee |
| Ordered byte stream | No ordering guarantee |
| Retransmission | No built-in retransmission |
| Flow control | No built-in flow control |
| Congestion control | No built-in congestion control |
| Higher overhead | Lower overhead |
| Higher latency | Lower latency |

---

## 1.6 QUIC and HTTP/3

QUIC rebuilds transport functionality in user space over UDP.

Key ideas:

- Runs over UDP
- Integrates TLS 1.3
- Supports independent streams
- Avoids TCP's cross-stream head-of-line blocking
- HTTP/3 uses QUIC

### Why QUIC matters

HTTP/2 can multiplex multiple streams over TCP, but TCP provides one ordered byte stream. A blocked TCP packet can therefore block delivery of data belonging to other streams.

QUIC provides independent streams, reducing this cross-stream blocking problem.

---

## 1.7 TCP Performance and Handshakes

### RTT matters

Connection setup costs round trips.

The Session 2 material highlights:

- TCP handshake introduces RTT cost
- TLS adds handshake cost
- TLS 1.3 reduces handshake overhead compared with older TLS
- QUIC integrates transport and TLS setup
- TCP Fast Open can send data during the SYN for suitable returning connections

Core idea:

> Bandwidth can be increased by buying capacity; network RTT is a latency property.

---

## 1.8 BBR

Traditional congestion control such as Reno/CUBIC relies heavily on packet loss as a congestion signal.

BBR instead estimates:

- Bottleneck bandwidth
- Round-trip time

Core concept:

```text
Estimated bottleneck bandwidth × RTT
```

This is used to estimate the amount of data the network can keep in flight.

---

## 1.9 Ethernet / IP Packet Sizes

Important Session 2 numbers:

### Ethernet II

- Destination MAC = 6 bytes
- Source MAC = 6 bytes
- EtherType = 2 bytes
- Payload = 46–1500 bytes
- FCS = 4 bytes

Typical maximum Ethernet frame:

```text
14-byte Ethernet header
+ 1500-byte payload
+ 4-byte FCS
= 1518 bytes
```

### Typical TCP over Ethernet

For a 1500-byte MTU:

```text
1500 MTU
- 20-byte IPv4 header
- 20-byte TCP header
= 1460-byte MSS
```

### Important distinction

- **MTU** → maximum IP packet payload that can fit in the Ethernet payload in this typical example.
- **MSS** → maximum TCP application payload carried in one TCP segment under the stated assumptions.

---

## 1.10 SMTP

SMTP is used for sending/relaying email.

Important ports:

- **25** → server-to-server relay
- **587** → message submission, commonly with STARTTLS
- **465** → implicit TLS

SMTP is text-based and designed for extensibility.

### SMTP envelope vs message headers

Envelope:

```text
MAIL FROM:<sender>
RCPT TO:<receiver>
```

Message headers:

```text
From:
To:
```

These do not have to match.

### DATA termination

SMTP DATA uses a line containing a single dot to terminate the message body.

---

## 1.11 MIME and Base64

Base64 converts binary data into text characters.

Important number:

```text
3 bytes → 4 Base64 characters
```

This creates approximately **33% encoding overhead**.

MIME also supports multipart messages using boundaries.

---

## 1.12 POP3

POP3 is an email retrieval protocol.

Common commands:

- USER
- PASS
- STAT
- LIST
- RETR
- DELE
- QUIT

POP3S uses port **995**.

The Session 2 material notes that deletion is applied at `QUIT`.

---

## 1.13 IMAP vs POP3

### POP3

- Primarily client-oriented mailbox state
- Mail is retrieved/downloaded to the client

### IMAP

- Mailbox state remains on the server
- Server-side folders
- Server-side flags
- Server-side search
- Partial fetching
- IDLE support

Memory hook:

```text
POP  → Pull mail
IMAP → Manage mailbox on server
```

---

## 1.14 Protocol Framing

TCP is a byte stream, so application protocols need framing.

Three major approaches:

### Delimiter-based

A special marker indicates the end.

Examples:
- SMTP lone dot
- HTTP CRLF CRLF
- MIME boundary
- Redis CRLF

### Length-based

A length tells the receiver how many bytes belong to the message.

Examples:
- HTTP Content-Length
- SMS TPDU
- SS7 MTP2 length information
- Protocol Buffers varint-style length framing

### Both

Some protocols combine structural delimiters with explicit lengths.

Examples from the Session 2 material include:
- IMAP literals
- HTTP headers + Content-Length
- HTTP/2 binary framing
- POP3 dotted body

### MCQ trap

TCP itself does **not** provide application message framing.

---

## 1.15 FTP

FTP separates control and data connections.

### Control

Server listens on port **21**.

### Active FTP

The server initiates the data connection toward the client.

### Passive FTP

The client initiates both control and data connections.

Passive FTP is generally more NAT/firewall friendly.

---

# 2. HIGH-ROI MCQ TRAPS

1. MAC address → **L2**, not L3.
2. IP address → **L3**.
3. Port number → **L4**.
4. Remote destination → host normally ARPs for **default gateway MAC**.
5. Router forwards using **destination IP**.
6. Switch forwards Ethernet frames using **MAC**.
7. ICMP is not TCP/UDP.
8. TCP is a **byte stream**, not a message protocol.
9. UDP does not guarantee delivery; it does not mean guaranteed packet loss.
10. TCP handshake = **SYN → SYN-ACK → ACK**.
11. SYN consumes a sequence number.
12. CLOSE_WAIT usually means the peer closed but the local application has not closed.
13. TIME_WAIT is associated with connection teardown.
14. QUIC runs over **UDP**, not TCP.
15. QUIC provides independent streams.
16. Base64 = **3 bytes → 4 characters**.
17. SMTP MAIL FROM is envelope information, not necessarily the same as `From:`.
18. FTP has separate control and data connections.
19. Passive FTP has the client initiate the data connection.
20. TCP needs application-level framing when messages must be separated.

---

# 3. MCQs

## A. Single Correct

### Q1
Which OSI layer is associated with IP?

A. L2  
B. L3  
C. L4  
D. L7

### Q2
Which address is primarily used by Ethernet switches?

A. IP address  
B. Port number  
C. MAC address  
D. TCP sequence number

### Q3
A host wants to reach a server on another network. Which MAC address does it normally need for the first Ethernet frame?

A. Remote server's MAC  
B. DNS server's MAC  
C. Default gateway's MAC  
D. Broadcast MAC permanently

### Q4
Which protocol is used by ping?

A. TCP  
B. UDP  
C. ICMP  
D. ARP

### Q5
Which statement about TCP is correct?

A. TCP preserves application message boundaries  
B. TCP is an unordered datagram protocol  
C. TCP provides an ordered byte stream  
D. TCP has no retransmission mechanism

### Q6
What is the correct TCP connection establishment sequence?

A. ACK → SYN → SYN-ACK  
B. SYN → ACK → SYN-ACK  
C. SYN → SYN-ACK → ACK  
D. SYN-ACK → SYN → ACK

### Q7
Which TCP state indicates that the remote side has closed while the local application has not yet closed?

A. TIME_WAIT  
B. CLOSE_WAIT  
C. SYN_SENT  
D. LISTEN

### Q8
QUIC runs over:

A. TCP  
B. UDP  
C. ICMP  
D. Ethernet

### Q9
How many bytes are in the UDP header?

A. 4  
B. 8  
C. 20  
D. 40

### Q10
Base64 converts:

A. 2 bytes into 3 characters  
B. 3 bytes into 4 characters  
C. 4 bytes into 3 characters  
D. 8 bytes into 4 characters

### Q11
SMTP server-to-server relay normally uses:

A. 21  
B. 25  
C. 80  
D. 110

### Q12
Which protocol maintains mailbox state primarily on the server?

A. POP3  
B. FTP  
C. IMAP  
D. SMTP

### Q13
Which FTP mode has the client initiate the data connection?

A. Active  
B. Passive  
C. Relay  
D. Proxy

### Q14
Which statement is true?

A. TCP provides message boundaries  
B. UDP guarantees ordered delivery  
C. TCP is a byte stream  
D. IP guarantees delivery

### Q15
What does TTL primarily help prevent?

A. Duplicate MAC addresses  
B. Infinite IP routing loops  
C. TCP retransmission  
D. DNS cache poisoning

---

# B. Multiple Correct

**Select ALL correct options.**

### Q16
Which are properties of TCP?

A. Reliable delivery  
B. Ordered byte stream  
C. Flow control  
D. Congestion control  
E. No connection establishment

### Q17
Which are associated with UDP?

A. 8-byte header  
B. No built-in delivery guarantee  
C. Connectionless operation  
D. Mandatory retransmission  
E. Low overhead

### Q18
Which are true about QUIC?

A. It runs over UDP  
B. It integrates TLS 1.3  
C. It supports independent streams  
D. It is exactly the same transport protocol as TCP  
E. HTTP/3 uses it

### Q19
Which are true about ARP?

A. It maps IPv4 addresses to MAC addresses  
B. It operates on the local network  
C. A host normally ARPs for its gateway's MAC when the destination is remote  
D. It replaces TCP  
E. It is used to establish the TCP three-way handshake

### Q20
Which are true about ICMP?

A. Ping uses ICMP  
B. It can report TTL expiration  
C. It is a transport protocol like TCP  
D. It can report unreachable destinations  
E. It uses TCP ports for addressing

### Q21
Which statements about SMTP are correct?

A. Port 25 is used for server-to-server relay  
B. Port 587 is used for message submission  
C. MAIL FROM is envelope information  
D. SMTP `From:` must always equal MAIL FROM  
E. A single-dot line can terminate DATA

### Q22
Which statements about Base64 are correct?

A. It encodes binary data as text characters  
B. 3 bytes become 4 Base64 characters  
C. It adds approximately 33% overhead  
D. It compresses data by approximately 33%  
E. It is a form of encryption

### Q23
Which are framing techniques?

A. Delimiter  
B. Length prefix  
C. Fixed-size messages  
D. TCP automatically preserving send boundaries  
E. Combining length and delimiters

### Q24
Which are true about FTP?

A. It has a separate control connection  
B. Control normally uses port 21  
C. It has a separate data connection  
D. Passive mode has the client initiate the data connection  
E. Active mode always uses UDP

### Q25
Which statements are correct?

A. A switch primarily forwards using MAC addresses  
B. A router forwards packets based on IP information  
C. IP is best-effort  
D. ICMP is a transport protocol  
E. MAC addresses are Layer 3 addresses

---

# ANSWER KEY

## Single Correct

1. **B**  
2. **C**  
3. **C**  
4. **C**  
5. **C**  
6. **C**  
7. **B**  
8. **B**  
9. **B**  
10. **B**  
11. **B**  
12. **C**  
13. **B**  
14. **C**  
15. **B**

## Multiple Correct

16. **A, B, C, D**  
17. **A, B, C, E**  
18. **A, B, C, E**  
19. **A, B, C**  
20. **A, B, D**  
21. **A, B, C, E**  
22. **A, B, C**  
23. **A, B, C, E**  
24. **A, B, C, D**  
25. **A, B, C**
