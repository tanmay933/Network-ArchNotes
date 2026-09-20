# Computer Networks — Session 1

## Network Programming 101

MCQ-focused revision notes based on the Session 1 course material.

---

## 1. TCP Server — Core Socket Flow

The canonical TCP server flow is:

```text
socket → bind → listen → accept → read/write → close
```

### `socket()`
Creates a socket.

- `AF_INET` → IPv4
- `SOCK_STREAM` → TCP-style byte stream

### `bind()`
Associates the socket with a local:

```text
IP address + port
```

`INADDR_ANY` means the server listens on all local interfaces/IP addresses.

### `listen()`
Turns the bound socket into a listening socket.

Important:

> `listen(fd, backlog)` does NOT mean "maximum number of simultaneous connections."

The backlog relates to the queue of connections waiting for the application to call `accept()`.

The effective backlog can also be limited by the OS (`net.core.somaxconn` on Linux).

### `accept()`
Returns a **new connected client socket FD**.

There are two different FDs:

```text
server_fd  → continues listening
client_fd  → communicates with one client
```

The listening socket remains available for future clients.

---

## 2. Accept Queue

A TCP connection can complete its handshake while the application has not yet called `accept()`.

The kernel can hold such connections in the accept queue.

Therefore:

- `listen()` + no `accept()` → connections may complete the handshake and wait in the queue.
- No `bind()` / no server listening on the requested port → connection can fail/refuse instead.

### MCQ trap

**Backlog ≠ maximum simultaneous connections.**

It is associated with the queue of connections waiting to be accepted.

---

## 3. TCP Is a Byte Stream

TCP does **not** preserve application message boundaries.

If the sender does:

```text
write("HELLO")
write("WORLD")
```

the receiver might observe:

```text
HELLOWORLD
```

or:

```text
HEL
LOWORLD
```

or other segmentation/coalescing.

Therefore:

> One `read()` does NOT necessarily correspond to one application message.

Application protocols need **framing**.

---

# 4. TCP Server vs TCP Client

## Server

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

## Client

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

The OS normally assigns the client an **ephemeral port** automatically.

A TCP connection is identified by both endpoints:

```text
client IP : client port
        ↕
server IP : server port
```

---

# 5. Ports

Ports identify application endpoints at the transport layer.

Session examples:

| Port | Example |
|---:|---|
| 21 | FTP |
| 25 | SMTP |
| 80 | HTTP |
| 110 | POP |
| 443 | HTTPS |
| 3000 | Node |
| 6379 | Redis |
| 8080 | Alternate HTTP |
| 2026 | Course server |

Course framing:

- `< 1024` → privileged/root ports
- `>= 1024` → ordinary users can generally use them

---

# 6. Byte Order

Network protocols use **network byte order**, which is big-endian.

Functions:

| Function | Meaning |
|---|---|
| `htons` | host → network, 16-bit |
| `htonl` | host → network, 32-bit |
| `ntohs` | network → host, 16-bit |
| `ntohl` | network → host, 32-bit |

Example:

```text
2026 = 0x07EA
```

On little-endian memory:

```text
EA 07
```

Network order:

```text
07 EA
```

### MCQ trap

`htons` is for a **16-bit** value such as a port.

---

# 7. DNS Resolution

The Session 1 material uses `gethostbyname()` as the example of hiding DNS resolution.

Conceptually:

```text
hostname
   ↓
/etc/hosts
   ↓
configured resolver
   ↓
DNS resolution
   ↓
IP address
```

DNS resolution can involve:

- local hosts file
- configured DNS resolver
- recursive resolution
- caching according to TTL

It can block.

The session notes that modern real-world code should use `getaddrinfo()` instead.

`gethostbyname()` is:

- IPv4-oriented
- not thread-safe

---

# 8. HTTP Basics

HTTP is an **application-layer protocol** carried over TCP in the Session 1 examples.

Basic request:

```text
GET / HTTP/1.1\r\n
Host: google.com\r\n
\r\n
```

Important:

- `\r\n` = CRLF
- HTTP headers are separated by CRLF
- A blank line marks the end of the header section

### MCQ trap

The blank line is meaningful framing; it is not just formatting.

---

# 9. curl Concepts

Useful conceptual flags:

```text
curl google.com
```

Basic HTTP request.

```text
curl -i google.com
```

Includes response headers.

```text
curl -vv https://google.com
```

Very verbose output, useful for debugging HTTP/TLS behavior.

---

# 10. Handling Many Clients

A simple server cannot necessarily handle many clients by processing each connection serially.

Session 1 introduces several approaches.

## Fork per client

Parent:

```text
accept
fork
continue accepting
```

Child:

```text
handle one client
```

`fork()` return values:

- `0` → child
- positive PID → parent
- `-1` → failure

### Limitation

Each connection gets a process.

Costs include:

- process memory
- process management
- scheduling/context switching

This becomes expensive at very high connection counts.

---

# 11. `select()`

`select()` allows one process/thread to monitor multiple file descriptors.

Course framing:

> `select()` scans the watched descriptors.

Therefore it is approximately:

```text
O(n)
```

The course material uses the traditional `FD_SETSIZE ≈ 1024` limitation.

### MCQ trap

`select()` is not "one syscall per connection."

It multiplexes multiple descriptors, but the watched set is repeatedly scanned.

---

# 12. `epoll`

Linux's scalable I/O multiplexing mechanism.

Conceptually:

```text
register interest once
        ↓
epoll_wait()
        ↓
receive ready descriptors
```

Instead of repeatedly scanning every watched descriptor, `epoll` reports descriptors that are ready.

Course framing:

```text
select → O(n)
epoll  → O(ready)
```

Related mechanisms:

- Linux → `epoll`
- BSD/macOS → `kqueue`
- Windows → IOCP

---

# 13. File Descriptors

Sockets are file descriptors.

Therefore the OS's open-file-descriptor limit matters.

```text
ulimit -n
```

can show the process/file-descriptor limit.

### Important distinction

`select()` has its traditional compile-time descriptor-set limitation.

`epoll` is instead primarily constrained by the OS file-descriptor limits and available resources.

---

# 14. Connection Pooling

Connection pooling means:

```text
open connection
      ↓
reuse it
      ↓
reuse it again
```

Instead of repeatedly creating connections.

Why?

It avoids repeatedly paying costs such as:

- TCP connection setup
- TLS handshake
- connection establishment overhead

### MCQ idea

Pooling is primarily about **amortising setup cost**.

---

# 15. `SO_REUSEADDR`

Useful when restarting a server and dealing with recently used socket addresses / `TIME_WAIT`.

Common scenario:

```text
server stops
   ↓
restart immediately
   ↓
"Address already in use"
```

`SO_REUSEADDR` can help with this situation.

---

# 16. Signals

Important signals from the session:

| Signal | Number | Meaning |
|---|---:|---|
| `SIGINT` | 2 | Interrupt |
| `SIGKILL` | 9 | Kill; cannot be caught |
| `SIGPIPE` | 13 | Broken pipe |
| `SIGTERM` | 15 | Termination request |

### SIGPIPE

Writing to a socket whose peer has already closed/reset the connection can trigger `SIGPIPE`.

Default behavior can terminate the process.

### SIGKILL

`SIGKILL` cannot be caught or ignored.

---

# 17. `SO_LINGER`

The Session 1 demo uses:

```text
SO_LINGER {1, 0}
```

for an **abortive close**.

Instead of a normal TCP FIN-based close, the connection can be reset with RST.

Important consequences:

- unsent data can be discarded
- connection closes quickly
- this is generally demo/special-purpose behavior, not normal graceful shutdown

---

# 18. OSI Model

The Session 1 mapping:

| Layer | Name | Examples |
|---:|---|---|
| 7 | Application | HTTP, echo protocol |
| 6 | Presentation | encoding/encryption/compression |
| 5 | Session | session management |
| 4 | Transport | TCP, ports |
| 3 | Network | IP, IP addresses |
| 2 | Data Link | Ethernet/Wi-Fi, MAC |
| 1 | Physical | copper/fibre/radio |

### MCQ mapping

```text
HTTP  → L7
TCP   → L4
IP    → L3
MAC   → L2
Cable → L1
```

Ports belong to the transport layer.

IP addresses belong to the network layer.

---

# 19. Protocol Design — Why Framing Exists

Because TCP is a byte stream, the receiver needs to know:

> Where does one application message end and the next begin?

Three common strategies:

## Fixed-length framing

Every message has exactly N bytes.

Pros:

- simple

Cons:

- padding
- inflexible message size

## Delimiter-based framing

A special marker indicates the end.

Example:

```text
MESSAGE\r\n
```

Problem:

If the delimiter can appear inside the payload, it must be escaped or otherwise handled.

## Length-prefix framing

Send the message length first:

```text
[length][payload]
```

Pros:

- binary-safe
- receiver knows exactly how many bytes to read

Potential issue:

- receiver needs the length first
- streaming/chunking may be needed for data whose size is not known beforehand

---

# 20. BCD

BCD = **Binary-Coded Decimal**.

Each decimal digit uses:

```text
4 bits
```

Course material notes that this can use about half the space of ASCII decimal digits.

Mentioned uses include:

- SIM cards
- SMS
- card transactions

### MCQ trap

BCD represents **decimal digits**, not arbitrary binary values.

---

# 21. ASN.1

ASN.1 provides a language-neutral way to define data structures.

Conceptually:

```text
schema
  ↓
encoder/decoder
  ↓
wire representation
```

Mentioned uses:

- X.509
- LDAP
- SNMP
- telecom protocols

The important MCQ concept is:

> Define the data structure once and generate/use encoders and decoders rather than manually inventing a wire format for every language.

---

# 22. TLV

TLV:

```text
Type | Length | Value
```

Example conceptually:

```text
TYPE → what is this?
LENGTH → how many bytes?
VALUE → actual data
```

Why it is useful:

- self-describing fields
- parser knows how many bytes to skip
- unknown fields can often be skipped using their length
- helps forward compatibility

Mentioned session examples include:

- TLS extensions
- ISO 8583
- protobuf-style structures

### MCQ trap

The key advantage is that the **length tells the receiver where the field ends**.

---

# 23. Hex and Base64

## Hex

One byte:

```text
2 hex characters
```

Useful for:

- debugging
- inspecting binary protocols
- reading packet dumps

## Base64

Converts:

```text
3 bytes → 4 characters
```

Therefore overhead is approximately:

```text
33%
```

Base64 is useful when binary data must travel through a text-oriented channel.

### MCQ trap

Base64 is **encoding**, not encryption.

---

# 24. RPC

RPC = Remote Procedure Call.

Conceptually:

```text
function name
+ arguments
+ framing
        ↓
network
        ↓
server executes operation
        ↓
response
```

RPC requires serialization/marshalling.

A remote call is not equivalent to a local function call because the network introduces:

- latency
- timeouts
- failures
- retries
- ambiguity around whether the remote operation completed

---

# 25. gRPC

gRPC combines RPC with:

- Protocol Buffers
- HTTP/2

Typical flow:

```text
.proto schema
      ↓
generated code
      ↓
protobuf binary serialization
      ↓
HTTP/2 transport
```

Important characteristics:

- schema-first
- generated client/server code
- binary serialization
- HTTP/2 multiplexing

Tradeoff:

> Binary protocols are generally less human-readable/debuggable than simple text protocols.

---

# 26. High-ROI MCQ Traps

Memorise these distinctions:

### `listen()` vs `accept()`

```text
listen  → makes socket listening / controls accept queue
accept  → obtains a connected client socket
```

### Listening FD vs client FD

```text
server_fd → keeps listening
client_fd → handles one client
```

### TCP vs message protocol

```text
TCP → byte stream
Application protocol → must provide framing
```

### `select()` vs `epoll`

```text
select → repeatedly scans watched set
epoll  → returns ready descriptors
```

### `htons()` vs `htonl()`

```text
htons → 16-bit
htonl → 32-bit
```

### Port vs IP

```text
Port → transport/application endpoint
IP   → network-layer address
```

### Hex vs Base64

```text
Hex     → 2 chars / byte
Base64  → 3 bytes / 4 chars
```

### Fixed vs delimiter vs length-prefix

```text
Fixed        → known size
Delimiter    → sentinel marks end
Length-prefix→ length tells receiver size
```

### RPC

```text
remote call ≠ local function call
```

Network failure and latency are fundamental differences.

---

# 27. Session 1 Must-Know Checklist

- [ ] TCP server socket lifecycle
- [ ] `socket`, `bind`, `listen`, `accept`
- [ ] listening FD vs client FD
- [ ] accept queue
- [ ] TCP byte-stream behavior
- [ ] TCP client lifecycle
- [ ] ephemeral ports
- [ ] network byte order
- [ ] `htons` / `htonl` / `ntohs` / `ntohl`
- [ ] DNS resolution concept
- [ ] HTTP request structure
- [ ] CRLF and header termination
- [ ] fork-per-client
- [ ] `select`
- [ ] `epoll`
- [ ] FD limits
- [ ] connection pooling
- [ ] `SO_REUSEADDR`
- [ ] important signals
- [ ] `SO_LINGER`
- [ ] OSI layer mapping
- [ ] TCP framing problem
- [ ] fixed/delimiter/length-prefix framing
- [ ] BCD
- [ ] ASN.1
- [ ] TLV
- [ ] hex
- [ ] Base64
- [ ] RPC
- [ ] gRPC

---

# Session 1 Status

**Completed for MCQ preparation.**

Focus on distinctions, terminology, numbers, and cause/effect rather than memorising implementation code.
