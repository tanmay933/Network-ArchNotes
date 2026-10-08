# Computer Networks — Session 1
## Network Programming 101

These notes are a compact but complete revision of Session 1. They are designed for **MCQ preparation first**, while keeping the concepts useful for **future networking/system-design interviews**.

> **Core idea:** start at the socket, understand what the OS is doing, then move upward to HTTP, concurrency, TLS, and finally protocol design.

---

# 1. Session Map

The session moves bottom-up:

1. TCP server and the seven socket calls
2. Ports, backlog, accept queue, and failure cases
3. Signals and connection-close behavior
4. Using existing clients: telnet, netcat, OpenSSL
5. TCP client, ephemeral ports, byte order, DNS
6. OSI layer mapping
7. HTTP and curl
8. Handling many clients: `fork`, `select`, `epoll`
9. File-descriptor limits and connection pooling
10. Socket options and packet inspection
11. SSL/TLS
12. Protocol design: text vs binary, framing, BCD, ASN.1, TLV
13. Hex/Base64, RPC, gRPC/Protobuf

---

# 2. TCP Server: The Seven System Calls

A TCP server is built around this sequence:

```text
socket → bind → listen → accept → read/write → close
```

The material's important point is that frameworks such as Express, Flask, and `net/http` ultimately wrap this kind of OS-level networking machinery.

## 2.1 `socket()`

Creates a socket/file descriptor.

Important arguments used in the session:

- `AF_INET` → IPv4
- `SOCK_STREAM` → TCP-style byte stream

## 2.2 `bind()`

Associates the socket with a local:

```text
IP address + port
```

`INADDR_ANY` means listen on all local interfaces/IP addresses.

## 2.3 `listen()`

Turns the socket into a listening socket and establishes the queueing behavior for connections waiting to be accepted.

```text
listen(fd, backlog)
```

### Critical distinction

`backlog` is **not** the maximum number of simultaneous clients.

It relates to the queue of connections waiting for the application to call `accept()`.

The effective queue size is also affected by OS limits. The course specifically mentions Linux's `net.core.somaxconn`.

## 2.4 `accept()`

Returns a **new connected client socket FD**.

There are therefore two different sockets:

```text
server_fd → keeps listening
client_fd → communicates with one client
```

`accept()` does not replace the listening socket.

## 2.5 `read()` / `write()`

Used to receive/send bytes on the connected client socket.

## 2.6 `close()`

Closes the socket/file descriptor.

---

# 3. What Happens If a Server Skips a Call?

This is a high-value MCQ area.

## No `bind()`

If nothing owns/listens on the requested port, the connection can fail immediately.

The course's example:

```text
client sends SYN
      ↓
nothing is listening
      ↓
RST
      ↓
connection refused
```

**Behavior:** fast and obvious; the client immediately knows nobody is listening.

## `bind()` + `listen()` but no `accept()`

The kernel can complete the TCP handshake and place the established connection in the accept queue.

```text
TCP handshake completes
        ↓
kernel queues connection
        ↓
application never calls accept()
        ↓
client waits / may eventually time out
```

**Important contrast:**

- No listener → immediate refusal
- Listener exists but application does not accept → connection can establish and wait in the queue

This can look like an overloaded/hung server.

---

# 4. The Accept Queue

The second argument to `listen()` is a queue-related parameter:

```text
listen(server_fd, backlog)
```

The course emphasizes three facts:

1. The kernel can finish the TCP handshake before `accept()` is called.
2. The completed connections wait for the application in a queue.
3. The exact full-queue behavior is not portable.

Depending on the system, a full queue may produce refusal or an indefinitely waiting client. Do not build application logic around one behavior observed on one machine.

Linux can cap the effective value using `net.core.somaxconn`.

### MCQ

**Backlog ≠ maximum simultaneous connections.**

---

# 5. TCP Is a Byte Stream, Not a Message Protocol

This is one of the most important concepts in the session.

TCP provides a **reliable ordered byte stream**. It does not preserve the boundaries between application-level writes/messages.

Suppose the sender does:

```text
write("HELLO")
write("WORLD")
```

The receiver might observe:

```text
HELLOWORLD
```

or:

```text
HEL
LOWORLD
```

or another segmentation/coalescing of the bytes.

Therefore:

> **One `read()` does not necessarily equal one application message.**

If an application needs messages, the application protocol must define **framing**.

We return to this in the protocol-design section.

---

# 6. Server vs Client

## TCP server

```text
socket
  ↓
bind
  ↓
listen
  ↓
accept
  ↓
read/write
  ↓
close
```

## TCP client

```text
socket
  ↓
connect
  ↓
read/write
  ↓
close
```

A normal TCP client does not need:

- `bind()`
- `listen()`
- `accept()`

The kernel normally assigns the client an **ephemeral port**.

### Why the server needs a known port

The client needs a stable destination to connect to.

### How a connection is distinguished

Think of the two endpoints:

```text
client IP : client port
        ↕
server IP : server port
```

The combination of endpoint addresses/ports distinguishes connections.

---

# 7. Ports

Ports are transport-layer identifiers for application endpoints.

## Session examples

| Port | Service/example |
|---:|---|
| 21 | FTP |
| 25 | SMTP |
| 80 | HTTP |
| 110 | POP |
| 443 | HTTPS |
| 3000 | Node |
| 6379 | Redis |
| 8080 | Alternate HTTP |
| 2026 | Session server |

## The 1024 boundary

The course uses this simplified framing:

```text
< 1024   privileged/root
≥ 1024   ordinary users can generally use them
```

The important security idea is not simply memorising the number.

Binding to port 80 traditionally requires elevated privileges. If the application itself runs with those privileges, a bug in its request parser can become a much more serious compromise.

A common deployment pattern is therefore:

```text
front proxy → 80/443
       ↓
unprivileged application → 8080/etc.
```

### MCQ

- 80 → HTTP
- 443 → HTTPS
- 25 → SMTP
- 21 → FTP
- 110 → POP
- 6379 → Redis

---

# 8. Signals and Socket Failure

The session highlights `SIGPIPE` as a dangerous server-side behavior.

## Important signals

| Signal | Number | Session meaning |
|---|---:|---|
| `SIGINT` | 2 | Interrupt, e.g. Control-C |
| `SIGKILL` | 9 | Kill; cannot be caught |
| `SIGPIPE` | 13 | Write to a closed/reset peer |
| `SIGTERM` | 15 | Termination request |

Also mentioned:

- `SIGHUP` → e.g. SSH disconnect
- `SIGSTOP` / `SIGTSTP` → stopping/control-related signals

## `SIGPIPE`

If a process writes to a socket whose peer has already closed/reset the connection, `SIGPIPE` can be generated.

By default, the process can be terminated.

The session demonstrates this with two writes:

```text
first write → SIGPIPE
second write → never reached
```

### MCQ trap

`SIGPIPE` is not an ordinary returned application error in the demonstrated default behavior; it can kill the process.

---

# 9. `SO_LINGER` and RST

The session uses:

```text
SO_LINGER = {1, 0}
```

for an **abortive close**.

Conceptually:

```text
normal close → FIN-based shutdown
SO_LINGER {1,0} → RST
```

The demo uses it to force a reset so that the server experiences the closed/reset peer and the `SIGPIPE` behavior.

### Remember

- FIN → normal TCP close behavior
- RST → reset/abortive termination
- `SO_LINGER {1,0}` in the session → force RST on close

This is a special-purpose/demo behavior, not the normal graceful shutdown strategy.

---

# 10. Existing Network Clients

You do not always need to write a client yourself.

The session gives three useful tools:

| Tool | Main use in the session |
|---|---|
| `telnet` | Manually speak text protocols |
| `openssl s_client` | Connect to TLS-encrypted services |
| `nc` / netcat | General TCP testing |

## `telnet`

If a protocol is text-based, you can type the protocol manually.

Example:

```bash
telnet google.com 80
```

Then send:

```text
GET / HTTP/1.1
Host: google.com

```

The blank line terminates the HTTP headers.

The session demonstrates Google returning an HTTP response, including a `301 Moved Permanently` response.

### Core idea

`telnet` is not an HTTP-only tool. It is simply a client that can establish a TCP connection and let you type bytes.

## Netcat

Examples from the session:

```bash
nc localhost 2026
nc -l 9000
```

Use it as a quick TCP client or listener.

## `openssl s_client`

For TLS connections, plain telnet is not enough because TLS requires a handshake.

Example:

```bash
openssl s_client -connect example.com:443
```

### MCQ

- Plain text TCP testing → `telnet` / `nc`
- TLS connection inspection → `openssl s_client`

---

# 11. TCP Client and Ephemeral Ports

A client typically does not explicitly call `bind()`.

Instead:

```text
connect()
   ↓
kernel selects ephemeral source port
```

The session gives a typical Linux ephemeral range of:

```text
32768–60999
```

The exact range is OS/configuration dependent; treat the numbers as the session's example, not a universal constant.

## Why ephemeral ports matter

They are finite resources.

The session also notes that `TIME_WAIT` can keep recently used ports occupied for roughly 60 seconds in its example.

A very busy client can therefore run into ephemeral-port/resource exhaustion.

### Server vs client

```text
SERVER                         CLIENT
known listening port           kernel-chosen ephemeral port
bind()                         usually no bind()
listen()                       connect()
accept()                       read/write
```

---

# 12. Network Byte Order

Different machines may represent multi-byte integers differently in memory.

Network protocols use **network byte order = big-endian**.

## Conversion functions

| Function | Meaning | Size |
|---|---|---:|
| `htons()` | host → network | 16-bit |
| `htonl()` | host → network | 32-bit |
| `ntohs()` | network → host | 16-bit |
| `ntohl()` | network → host | 32-bit |

Mnemonic:

```text
h = host
n = network
s = short (16-bit)
l = long (32-bit)
```

## Example from the session

```text
2026 = 0x07EA
```

Little-endian representation:

```text
EA 07
```

Network/big-endian order:

```text
07 EA
```

Therefore a port number is commonly passed through:

```c
htons(port)
```

### MCQ trap

`htons()` is **16-bit**. `htonl()` is **32-bit**.

---

# 13. DNS Hidden Inside a Function Call

The session uses:

```c
gethostbyname("localhost")
```

as an example of how a simple function call can hide substantial networking work.

Conceptually:

```text
hostname
   ↓
/etc/hosts
   ↓
configured DNS resolver
   ↓
DNS resolution / recursive lookup
   ↓
IP address
```

The session describes:

1. `/etc/hosts` can be checked first.
2. A query can then go to the configured resolver.
3. The resolver can perform recursive resolution.
4. Results can be cached according to TTL.

DNS resolution can block for significant time when the result is not already cached.

## `gethostbyname()` limitations

The session specifically notes that it is:

- IPv4-oriented
- not thread-safe
- capable of blocking

The session recommends `getaddrinfo()` for real-world code.

### MCQ trap

Changing `/etc/hosts` can override normal DNS resolution for a matching hostname and can therefore produce confusing stale-resolution behavior.

---

# 14. OSI Model Mapping

The session says that the socket program has been touching multiple layers without explicitly naming them.

| Layer | Name | Session examples |
|---:|---|---|
| 7 | Application | HTTP, echo protocol |
| 6 | Presentation | encoding, encryption, compression |
| 5 | Session | session management |
| 4 | Transport | TCP, ports |
| 3 | Network | IP addresses |
| 2 | Data Link | Ethernet, Wi-Fi, MAC addresses |
| 1 | Physical | copper, fibre, radio |

### High-value mapping

```text
HTTP   → L7
TCP    → L4
Port   → L4
IP     → L3
MAC    → L2
Cable  → L1
```

### MCQ trap

Ports are **not** Layer 3. Ports belong to the transport layer.

IP addresses are Layer 3.

---

# 15. HTTP Basics

The session's HTTP example is a text protocol carried over TCP.

A minimal request is:

```text
GET / HTTP/1.1\r\n
Host: google.com\r\n
\r\n
```

## Important syntax

`\r\n` = **CRLF** (Carriage Return + Line Feed).

HTTP headers are separated by CRLF.

A blank line marks the end of the header section.

```text
Header 1\r\n
Header 2\r\n
\r\n
<body...>
```

### MCQ trap

The blank line is meaningful protocol framing, not merely visual formatting.

---

# 16. `curl` as a Networking Tool

The session treats curl as a teaching/debugging instrument, not merely a download command.

## Basic request

```bash
curl google.com
```

## Include response headers

```bash
curl -i google.com
```

## Very verbose debugging

```bash
curl -vv https://google.com 2>&1 | less
```

Use verbose output to inspect request/response and TLS-related behavior.

### MCQ

`-i` → include response headers.

`-vv` → very verbose output.

---

# 17. One Client at a Time Is Not Enough

The simple echo server processes one client at a time.

For many clients, the session presents two broad approaches:

```text
1 process per connection
        vs
1 process handling many sockets
```

---

# 18. `fork()` Per Connection

Typical model:

```text
parent
  ↓
accept()
  ↓
fork()
 ↙   ↘
child  parent
 ↓       ↓
client   accept next client
```

The child handles the connected client.

The parent continues accepting new clients.

## `fork()` return values

| Return value | Meaning |
|---:|---|
| `0` | You are in the child |
| `> 0` | You are in the parent; value is child's PID |
| `-1` | `fork()` failed |

The session emphasizes that the same code executes in both processes; the return value tells each process which branch it is in.

## File-descriptor handling

After `fork()`, both processes initially have their own references to the descriptors.

Typical pattern:

- Child handles the client and closes its client FD when finished.
- Parent closes its copy of the client FD and goes back to `accept()`.

## Advantages

- Simple mental model
- OS handles process scheduling
- A child has separate memory, helping isolate a crash

## Cost

A process per client means:

- memory overhead
- process-management overhead
- scheduling/context-switching overhead

The session frames it as reasonable for hundreds of clients but poor at very large counts such as ten thousand.

---

# 19. `select()`

`select()` allows one process/thread to monitor multiple file descriptors.

Instead of blocking on one socket:

```text
many sockets
     ↓
 select()
     ↓
which ones are ready?
```

The course's model:

```text
select → O(n)
```

because the watched descriptor set is repeatedly scanned.

The traditional limitation highlighted by the session is:

```text
FD_SETSIZE ≈ 1024
```

### Why `select()` still exists

It is portable and available across many systems.

### MCQ trap

`select()` multiplexes descriptors; it does not mean one independent process/thread per connection.

---

# 20. `epoll()`

`epoll` is the Linux mechanism presented for scalable I/O multiplexing.

Conceptually:

```text
register interest once
        ↓
   epoll_wait()
        ↓
ready descriptors
```

Instead of repeatedly walking every watched descriptor, the kernel can return descriptors that are ready.

The course framing is:

```text
select → O(n)
epoll  → O(ready)
```

## Platform comparison from the session

| Platform | Mechanism |
|---|---|
| Linux | `epoll` |
| BSD/macOS | `kqueue` |
| Windows | IOCP |

The session associates this event-driven style with systems such as nginx and Node.

### High-value distinction

```text
select → repeatedly scan the watched set
epoll  → return ready descriptors
```

---

# 21. File Descriptors and `ulimit`

Sockets are represented as file descriptors.

Therefore, the OS/process limit on open file descriptors becomes a scalability limit.

Useful command:

```bash
ulimit -n
```

The session gives 1024 as a common example of a file-descriptor ceiling.

### Important distinction

Do not confuse:

- `FD_SETSIZE` → traditional `select()` descriptor-set limit
- `ulimit -n` → process/open-file-descriptor limit

Both can matter, but they are different constraints.

---

# 22. Connection Pooling

Instead of repeatedly opening new connections:

```text
open connection
      ↓
reuse
      ↓
reuse
      ↓
reuse
```

A connection pool keeps connections available for reuse.

## Why?

Repeatedly establishing a connection costs resources/time, including:

- TCP connection setup
- TLS handshake, when TLS is used
- connection establishment overhead

Pooling **amortises setup cost** across multiple requests.

The session's client-side diagram connects three ideas:

```text
client connection limit
        +
file descriptor limit
        +
connection pooling
```

Pooling reduces repeated handshake cost, while the number of open sockets is still a resource constraint.

---

# 23. Socket Options

## `INADDR_ANY`

Means listen on every local interface/address.

Convenient in development, but the session says this should be a deliberate production decision.

If you only intend to listen on one interface/address, bind specifically to it.

## Promiscuous mode

The session mentions promiscuous mode in the context of packet sniffing.

On a hub, frames reach every port, making it possible to observe traffic not addressed to your host.

The important conceptual link is:

```text
packet visibility → sniffing → why plaintext traffic is dangerous
```

## `SO_REUSEADDR`

Useful when restarting a server and the previous socket is associated with `TIME_WAIT`.

Without appropriate reuse behavior, an immediate restart can produce:

```text
address already in use
```

### MCQ

`SO_REUSEADDR` is strongly associated with restarting/rebinding sockets when recently used addresses are still relevant to TCP state.

---

# 24. Inspect the Actual Bytes: tcpdump / Wireshark

When debugging networking, do not rely only on application logs.

The session's message is:

> Stop guessing. Look at the bytes.

Example capture:

```bash
sudo tcpdump -ni any port 2026 -w 1.pcap
```

Read as ASCII:

```bash
tcpdump -r local_capture.pcap -A
```

Read as hex + ASCII:

```bash
tcpdump -r local_capture.pcap -X
```

Wireshark provides a graphical way to inspect the same packet-level behavior.

### What this helps reveal

- actual bytes sent
- protocol headers
- framing
- ASCII vs binary representation
- packet-level behavior

---

# 25. SSL / TLS

The session's key statement is:

> **SSL/TLS secures TCP, not HTTP.**

The idea is that TLS sits between the application protocol and TCP.

Therefore the same TLS machinery can protect:

- HTTPS
- SSH-related network connections
- SMTP connections
- database connections
- other application protocols carried over sockets

## Simplified TLS flow from the session

### 1. Keys and certificates

Asymmetric cryptography is used to establish/agree on a shared secret.

Then symmetric cryptography is used for the actual data because it is much faster.

```text
asymmetric crypto
      ↓
establish shared secret
      ↓
symmetric crypto
      ↓
application data
```

### 2. Certificate and identity

A certificate binds a public key to an identity.

The CA trust chain is what allows the client to trust that binding.

### 3. Certificate pinning

Pinning removes the normal broad CA trust decision from the equation by trusting a specific certificate/key identity.

The session highlights this as common in mobile applications.

### MCQ traps

- TLS is not the same thing as HTTP.
- HTTPS is HTTP protected by TLS.
- TLS can protect protocols other than HTTP.
- Symmetric encryption is used for efficient bulk data after the secure setup.

---

# 26. Text Protocols vs Binary Protocols

The session contrasts two design styles.

| Text | Binary |
|---|---|
| Human-readable | Machine-oriented |
| Easy to inspect with telnet/logs | Usually needs tooling |
| Easy to debug manually | Compact and unambiguous |
| Can be verbose | Often more efficient to parse |
| Whitespace/case rules matter | Explicit binary representation |

The course uses HTTP as an example of how far text protocols can go, while noting that HTTP/2 is binary.

### Why text is attractive

You can:

```text
telnet → type it → inspect it
```

This is a major debugging advantage.

### Why binary is attractive

Binary protocols can be:

- compact
- explicit
- efficient to parse
- less dependent on whitespace/text conventions

The tradeoff is reduced human readability.

---

# 27. Message Framing

## The problem

TCP provides bytes, not messages.

So a protocol designer must answer:

> **How does the receiver know where one message ends?**

The session presents three core strategies.

---

## 27.1 Fixed-Length Framing

Every message has exactly `N` bytes.

```text
[message: N bytes]
[message: N bytes]
[message: N bytes]
```

### Advantages

- very simple parsing
- no ambiguity about message boundaries

### Disadvantages

- padding may be required
- message size is inflexible

### Best recall phrase

**Fixed length = simple, but rigid.**

---

## 27.2 Delimiter-Based Framing

A sentinel marks the end.

Example:

```text
MESSAGE\r\n
```

The receiver reads until it sees the delimiter.

### Advantage

Simple when the delimiter is guaranteed to be outside the payload.

### Problem

If the delimiter can occur inside the payload, the protocol needs escaping or another mechanism.

### Example from the session

HTTP's blank line separates its header section from the body.

### Best recall phrase

**Delimiter = sentinel marks the boundary.**

---

## 27.3 Length-Prefix Framing

Send the size first:

```text
[length][payload]
```

Example idea:

```text
Content-Length: 219
```

The receiver reads exactly the indicated number of bytes.

### Advantages

- binary-safe
- explicit message size
- robust for arbitrary payloads

### Important requirement

The receiver must first obtain the length correctly before it can know how many payload bytes to read.

### Best recall phrase

**Length prefix = size tells you where the message ends.**

---

# 28. Framing Comparison

| Strategy | Boundary | Main advantage | Main drawback |
|---|---|---|---|
| Fixed | Always N bytes | Simplest | Rigid/padding |
| Delimited | Sentinel | Simple | Delimiter escaping |
| Length-prefix | Explicit size | Robust/binary-safe | Must parse length first |

### MCQ shortcut

```text
Fixed       → known size
Delimited   → sentinel
Length      → size field
```

---

# 29. BCD — Binary-Coded Decimal

BCD represents **decimal digits** using binary-coded nibbles.

Each decimal digit uses:

```text
4 bits
```

Example from the session:

```text
decimal:  9   8   7   6   5
nibbles: 1001 1000 0111 0110 0101
hex:      98   76   5F
```

The important conceptual point is that BCD encodes each decimal digit separately rather than treating the entire number as an ordinary binary integer.

The session mentions uses such as:

- SIM cards
- SMS
- card transactions

It also notes the space advantage compared with ASCII decimal digits.

### MCQ trap

BCD is about representing **decimal digits**, not arbitrary binary values.

---

# 30. ASN.1

ASN.1 is a language-neutral way to describe structured data.

The session's model:

```text
schema
   ↓
code generation
   ↓
encoder + decoder
   ↓
wire representation
```

The key idea is **schema-first data description**.

Instead of hand-writing a parser for every language, the structure can be described once and encoder/decoder code can be generated.

The session mentions ASN.1 in:

- X.509
- LDAP
- SNMP
- telecom protocols

## Example schema from the session

```text
MyModule DEFINITIONS ::= BEGIN
    Item ::= SEQUENCE {
        itemCode    INTEGER (1..99999),
        color       VisibleString ("Black" | "Blue" | "Brown"),
        isTaxable   BOOLEAN
    }
END
```

You do not need to memorise this syntax for the main conceptual question.

### Remember

**ASN.1 = describe the data structure once; generate/use the codec.**

---

# 31. TLV — Type, Length, Value

TLV means:

```text
Type | Length | Value
```

A stream can contain repeated fields:

```text
<Type><Length><Value>
<Type><Length><Value>
...
```

## Why TLV is powerful

The length tells the parser how many bytes belong to the value.

Therefore an implementation can often skip a field it does not understand:

```text
unknown Type
     ↓
read Length
     ↓
skip Value
     ↓
continue parsing
```

This is a major reason the session associates TLV with **forward compatibility**.

The session mentions:

- TLS extensions
- ISO 8583
- protobuf-style structures

### MCQ

**Type = what is it?**

**Length = how many bytes?**

**Value = the data.**

The most important advantage is that the **length tells the receiver where the field ends**.

---

# 32. Hex

Hexadecimal is useful for inspecting binary data.

One byte is represented by:

```text
2 hex characters
```

Example:

```text
1 byte = 8 bits = 2 hex digits
```

Hex is useful for:

- packet dumps
- debugging binary protocols
- spotting repeated tags/lengths
- reading `tcpdump -X` output

### Mental model

**Hex = good for reading binary bytes.**

---

# 33. Base64

Base64 converts binary data into text-safe characters.

The core ratio is:

```text
3 bytes → 4 characters
```

Therefore the encoded representation has about **33% more bytes** than the original for large inputs.

Common text-oriented uses mentioned by the session:

- Basic authentication data
- JWT-related data
- email attachments
- data URIs

### Critical distinction

Base64 is **encoding, not encryption**.

Anyone who can decode it can recover the original bytes.

### Hex vs Base64

| Hex | Base64 |
|---|---|
| Mainly for reading/debugging | Mainly for text-safe transport |
| 2 chars per byte | 3 bytes → 4 chars |
| Easy to inspect | More compact than hex for transport |

---

# 34. RPC — Remote Procedure Call

RPC makes a remote operation look conceptually like a function call.

The session summarises it as:

```text
function name
+ arguments
+ framing
      ↓
   network
      ↓
server operation
      ↓
 response
```

## Marshalling

Objects/arguments are converted into bytes.

```text
objects → bytes
```

This is **marshalling**.

The reverse process is **demarshalling/unmarshalling**.

## The crucial RPC insight

A remote call is **not** equivalent to a local function call.

The network introduces:

- latency
- timeouts
- connection failures
- retries
- partial failures
- ambiguity about whether the server completed the operation before the client timed out

Example:

```text
client sends request
      ↓
server completes operation
      ↓
response is lost
      ↓
client times out
      ↓
client retries
```

Now the operation might execute twice unless the system is designed for safe retries/idempotency.

### Interview-level takeaway

RPC hides network messaging behind an API, but it cannot remove the fundamental unreliability/latency of the network.

---

# 35. gRPC and Protocol Buffers

The session presents:

> **gRPC is RPC with protobuf underneath.**

The conceptual stack is:

```text
.proto schema
     ↓
code generation
     ↓
protobuf serialization
     ↓
HTTP/2
```

## `.proto`

Schema-first definition.

Client/server code can be generated in different languages.

## Protobuf

Provides compact binary serialization.

The session highlights its relationship with tagged/length-oriented encoding and its ability to add fields without breaking older clients when designed compatibly.

## HTTP/2

The session describes HTTP/2 as binary and multiplexed:

```text
one connection
     ↓
multiple streams
```

## Tradeoff

Compared with plain text protocols:

- better compactness/efficiency
- generated code
- less human-readable
- requires tooling for inspection

### MCQ shortcut

```text
RPC       → remote function-call abstraction
gRPC      → RPC framework using protobuf + HTTP/2
protobuf  → binary serialization/schema system
```

---

# 37. MCQ Traps — Rapid Revision

## Socket lifecycle

```text
socket → creates socket
bind   → assigns local IP/port
listen → listening socket + queue behavior
accept → NEW connected client FD
```

## FDs

```text
server_fd → keeps accepting
client_fd → communicates with one client
```

## Backlog

```text
backlog ≠ max simultaneous clients
backlog → waiting/accept queue behavior
```

## TCP

```text
TCP → byte stream
TCP ≠ message boundaries
```

## Client

```text
client usually does not bind
kernel assigns ephemeral source port
```

## Byte order

```text
network order → big-endian
htons  → 16-bit host → network
htonl  → 32-bit host → network
ntohs  → 16-bit network → host
ntohl  → 32-bit network → host
```

## DNS

```text
gethostbyname → IPv4-oriented, not thread-safe
getaddrinfo   → preferred real-world API in session
```

## OSI

```text
HTTP → L7
TCP  → L4
Port → L4
IP   → L3
MAC  → L2
```

## HTTP

```text
CRLF = \r\n
blank line = end of HTTP header section
```

## `fork`

```text
0   → child
>0  → parent; child's PID
-1  → failure
```

## Multiplexing

```text
select → O(n), repeated scanning
 epoll → O(ready), ready descriptors
```

## Platform mapping

```text
Linux      → epoll
BSD/macOS  → kqueue
Windows    → IOCP
```

## Limits

```text
FD_SETSIZE → select's traditional descriptor-set limit
ulimit -n → open FD/process limit
```

## Signals

```text
SIGINT  → 2
SIGKILL → 9, cannot be caught
SIGPIPE → 13, broken pipe/write to closed peer
SIGTERM → 15
```

## Close behavior

```text
FIN → normal close
RST → reset/abortive close
SO_LINGER {1,0} → RST in session demo
```

## Socket options

```text
INADDR_ANY  → all local interfaces
SO_REUSEADDR → helps restart/rebind situations involving recent socket state
```

## TLS

```text
TLS protects TCP connections; not just HTTP
asymmetric → setup/key agreement
symmetric  → bulk data
certificate → identity binding
CA chain   → trust
pinning    → trust specific identity/key instead of broad CA trust
```

## Framing

```text
Fixed       → N bytes
Delimited   → sentinel
Length      → size field
```

## Encoding

```text
BCD      → 4 bits per decimal digit
Hex      → 2 chars per byte
Base64   → 3 bytes → 4 chars
Base64   → encoding, NOT encryption
```

## Structured protocols

```text
ASN.1 → schema + generated codec
TLV   → Type + Length + Value
RPC   → remote operation abstraction
```

## gRPC

```text
gRPC → RPC + protobuf + HTTP/2
```

---

# 38. Interview-Level Mental Models

## What actually happens when a client connects?

```text
Client
  |
  | TCP handshake
  v
Kernel
  |
  | completed connection queued
  v
accept()
  |
  | returns new FD
  v
Application
```

The listening FD stays available for new connections.

## Why can one `read()` be insufficient?

Because TCP does not understand application messages. It only delivers an ordered byte stream.

## Why does nginx care about `epoll`?

A high-concurrency server does not want one blocking operation/process per connection. It wants to efficiently identify which sockets are ready and process those.

## Why do connection pools matter?

TCP/TLS setup is work. Reusing established connections spreads that setup cost over many requests.

## Why do protocols need framing?

The receiver needs a deterministic rule for deciding where one application message ends.

## Why use binary protocols?

Compactness, explicit structure, and efficient parsing.

## Why use text protocols?

Human readability and easy debugging are extremely valuable.

## Why is RPC harder than a local function call?

The remote machine can be slow, unreachable, or can finish the operation while the response is lost.

---

# 39. Practical Commands From the Session

### Compile and run echo server

```bash
gcc -o echo_server echo_server.c
nc localhost 2026
```

### Test HTTP manually

```bash
telnet google.com 80
```

Then:

```text
GET / HTTP/1.1
Host: google.com

```

### Test a TLS service

```bash
openssl s_client -connect example.com:443
```

### Netcat listener

```bash
nc -l 9000
```

### Inspect FD limit

```bash
ulimit -n
```

### Capture traffic

```bash
sudo tcpdump -ni any port 2026 -w 1.pcap
```

### Read capture as ASCII

```bash
tcpdump -r local_capture.pcap -A
```

### Read capture as hex + ASCII

```bash
tcpdump -r local_capture.pcap -X
```

---

# 40. Session Homework / Practical Extensions

The course suggests these exercises because they turn the concepts into actual networking intuition:

1. **Use Wireshark/tcpdump** on your laptop. Redis traffic is a useful example; the session notes RESP as text and length-prefixed.
2. **Write a server and client** in your own language and identify the same underlying socket calls.
3. **Extend the echo server** so messages have actual structure. Then decide which framing strategy to use.
4. **Implement Base64 from scratch** to understand grouping bits rather than treating the operation as a black box.
5. **Build a TLV protocol** with nested/optional fields, then add a new field and test whether an older parser can skip it.
6. **Build a small curl-like client** that follows redirects, prints headers, and handles chunked encoding.

The final course challenge is deliberately simple:

> Break something on port 2026 and trace the abstraction back to the seven socket calls.

The point is to recognise that higher-level networking libraries still depend on the same underlying primitives.

---

# 36. Final Takeaway

The entire session can be reduced to one progression:

```text
OS socket primitives
        ↓
TCP connections
        ↓
byte streams
        ↓
application protocols
        ↓
framing + encoding
        ↓
concurrency + scalability
        ↓
TLS + observability
        ↓
RPC / gRPC / custom protocols
```

The most important conceptual chain is:

> **TCP gives you a byte stream. Your application must define how bytes become messages. Once you have many clients, you must manage sockets efficiently. Once the protocol becomes structured, you must choose framing and encoding deliberately.**
