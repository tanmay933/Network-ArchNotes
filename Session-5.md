# Computer Networks — Session 5
## Evolution of HTTP: HTTP/1.0 → HTTP/1.1 → HTTP/2 → QUIC + HTTP/3

> **Goal:** Understand why each HTTP generation changed, what problem the previous version had, and the exact mechanism used to fix it.

---

# 1. Big Picture

HTTP kept the same high-level meaning, but changed how that meaning is encoded and transported.

| Version | Main RFC(s) | Main problem / limitation |
|---|---|---|
| HTTP/1.0 | RFC 1945, May 1996 | One response per connection |
| Early HTTP/1.1 | RFC 2068, Jan 1997 | Persistence becomes default |
| HTTP/1.1 | RFC 2616 → 7230–7235 → 9110–9112 | Head-of-line blocking, header repetition, framing problems |
| HTTP/2 | RFC 7540 → 9113 | Multiplexing fixes HTTP-level HOL, but TCP HOL remains |
| HTTP/3 | RFC 9114 + QUIC RFC 9000 | QUIC removes TCP-level HOL and supports connection migration |

**Core mental model:**

```text
HTTP/1.0
  ↓
One request → one TCP connection

HTTP/1.1
  ↓
Keep the TCP connection open
  ↓
But pipelining creates HTTP-level HOL blocking

HTTP/2
  ↓
One TCP connection
  ↓
Many streams + stream IDs + binary frames
  ↓
HTTP-level HOL mostly fixed
  ↓
TCP still has one ordered byte stream

HTTP/3
  ↓
HTTP over QUIC over UDP
  ↓
Independent QUIC streams
  ↓
TCP-level HOL eliminated
```

The teacher's closing slide summarizes the progression as five wire formats and the problem each stage ends on (page 51). 

---

# 2. HTTP/1.0 — RFC 1945

## 2.1 Basic request/response

Example:

```http
GET / HTTP/1.0

```

The server responds with:

```http
HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 263

<html>...</html>
```

The **blank line** separates headers from the body.

### Important

HTTP/1.0 normally closes the TCP connection after the response.

So:

```text
TCP handshake
    ↓
request
    ↓
response
    ↓
FIN / connection closes
```

The next request needs another TCP connection.

---

# 3. HTTP/1.0's Major Problems

## 3.1 One request per TCP connection

A page with many assets means many handshakes.

The PDF gives an example:

- 24 objects
- 200 ms RTT
- 24 TCP handshakes = 4.8 s
- 24 request/response round trips = 4.8 s
- approximately 9.6 s total

The key issue is repeated connection setup and slow-start/congestion-window costs.

---

## 3.2 No Host header

HTTP/1.0 did not provide the modern required `Host` mechanism.

Conceptually:

```text
one IP → one website
```

If two websites share the same IP, HTTP/1.0 cannot reliably select the intended virtual host using the HTTP request.

HTTP/1.1 fixes this with:

```http
Host: site-a.local
```

The PDF also connects this idea across layers:

```text
HTTP/1.0 → Host
TLS      → SNI
HTTP/2   → :authority
Ingress  → host routing
```

The common problem is:

> Many logical destinations need to share one underlying connection/address.

---

## 3.3 Weak caching validation

HTTP/1.0 uses:

```http
Last-Modified:
If-Modified-Since:
```

A client can ask:

```http
If-Modified-Since: <timestamp>
```

If the resource has not changed, the server can return:

```http
304 Not Modified
```

No body needs to be transferred.

### Problem with timestamps

The PDF highlights three weaknesses:

1. No sub-second precision.
2. Clock differences/drift.
3. `mtime` is not an identity for the bytes.

---

# 4. HTTP/1.0 What It Could Not Do

According to the PDF:

| Limitation | Fixed by |
|---|---|
| Non-persistent connection | HTTP/1.1 |
| Limited methods/headers | HTTP/1.1 |
| Thin timestamp-based caching | HTTP/1.1 |
| No Host | HTTP/1.1 |

### Important correction

HTTP/1.0 is sometimes described as having head-of-line blocking.

The teacher explicitly corrects this:

> True HOL blocking requires a queue.

HTTP/1.0 does not have pipelining, so its problem is better described as **serialization cost**: one round trip per object.

---

# 5. Early HTTP/1.1 — RFC 2068, January 1997

The biggest change:

## Persistent connections become the default.

HTTP/1.0:

```text
default → close
```

HTTP/1.1:

```text
default → keep connection open
```

You can explicitly close it with:

```http
Connection: close
```

Example:

```text
one TCP handshake
    ↓
GET /
response
    ↓
GET /style.css
response
    ↓
GET /app.js
response
    ↓
same TCP connection
```

This removes repeated TCP handshakes.

---

# 6. Host Header

HTTP/1.1 requires:

```http
Host: site-a.local
```

Without Host, the server returns:

```http
400 Bad Request
```

This enables:

```text
same IP
same port
different Host values
different websites
```

### MCQ trap

**Host is not an HTTP/2 invention.**

HTTP/1.1 introduced the required Host mechanism.

HTTP/2 represents the authority using the `:authority` pseudo-header.

---

# 7. ETag — A Fingerprint Instead of a Clock

HTTP/1.1 adds:

```http
ETag: "55274ac22c651b90"
```

A client can later send:

```http
If-None-Match: "55274ac22c651b90"
```

If unchanged:

```http
HTTP/1.1 304 Not Modified
```

No full response body is needed.

## ETag vs Last-Modified

| Last-Modified | ETag |
|---|---|
| Timestamp | Validator/fingerprint |
| Time-based | Representation-based |
| Can suffer from timestamp limitations | Can distinguish representations more precisely |
| HTTP/1.0 mechanism | HTTP/1.1 mechanism |

The PDF example uses a SHA-256-derived value as an ETag.

---

# 8. Strong vs Weak ETags

Weak ETag:

```http
W/"..."
```

A weak validator means the representations are considered equivalent for the validation purpose, but not necessarily byte-for-byte identical.

The PDF emphasizes:

> Range requests need a strong validator.

Also remember:

```text
If-None-Match → caching / conditional retrieval
If-Match      → optimistic concurrency / write protection
```

---

# 9. Accept-* and Vary

HTTP can negotiate different representations.

Example:

```http
Accept-Encoding: gzip
```

Server:

```http
Content-Encoding: gzip
```

The PDF example shows a large reduction in body size through compression.

### Vary

```http
Vary: Accept-Encoding
```

Think of `Vary` as:

> "This URL can have different answers depending on this request header."

So a cache must consider the relevant request header when determining the cached representation.

### Important trap

```text
no-cache ≠ no-store
```

According to the PDF:

- `no-cache` → revalidate before using the cached response.
- `no-store` → do not store it.

---

# 10. Range Requests

HTTP/1.1 supports partial retrieval.

Request:

```http
Range: bytes=100-199
```

Response:

```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 100-199/202000
Content-Length: 100
```

## Suffix range

```http
Range: bytes=-50
```

Means:

> Give me the last 50 bytes.

## Unsatisfiable range

```http
Range: bytes=999999-
```

can produce:

```http
416 Range Not Satisfiable
Content-Range: bytes */202000
```

### Uses

- Video scrubbing
- Resumable downloads
- Reading file footers
- Large data files such as Parquet/ZIP structures

---

# 11. Chunked Transfer Encoding

Sometimes the sender does not know the total body length in advance.

HTTP/1.1 can use:

```http
Transfer-Encoding: chunked
```

instead of:

```http
Content-Length:
```

Example:

```text
37
<55 bytes of data>
0
```

`37` is hexadecimal:

```text
0x37 = 55
```

The final:

```text
0
```

marks the end.

### Mental model

Chunked encoding combines:

```text
length-prefix each chunk
+
delimiter marking the end
```

The PDF connects this to FastCGI framing from Session 3.

---

# 12. Other HTTP/1.1 Features

### OPTIONS

Ask what the server supports.

Commonly encountered with CORS preflight.

### PUT

Create/replace a resource.

### DELETE

Remove a resource.

### TRACE

Echo the request back.

The PDF notes that it is commonly disabled because of security concerns.

### Via

Each proxy can append itself.

Useful for understanding proxy chains.

### Expect: 100-continue

Allows a client to ask whether it should send a large request body before uploading it.

Useful for large uploads.

### Cache-Control

Introduces a richer caching vocabulary:

```text
max-age
s-maxage
no-cache
no-store
```

---

# 13. HTTP/1.1 Connection Limits

The early HTTP/1.1 specification said clients **SHOULD NOT maintain more than 2 connections** to a server/origin.

Browsers eventually used more connections.

The PDF shows:

```text
6 connections per origin
18 with domain sharding
```

Domain sharding means using multiple hostnames to obtain more parallel TCP connections.

This was later abandoned as HTTP/2 made multiple connections unnecessary.

---

# 14. HTTP/1.1 Today — RFC 2616

RFC 2616 was the major 1999 HTTP/1.1 specification.

The PDF emphasizes that 1999 was mostly **tightening and documenting behavior**, rather than completely redesigning HTTP.

Important additions/details include:

- 307 Temporary Redirect
- 416 Range Not Satisfiable
- 417 Expectation Failed
- Host as MUST
- ETag requirement for resource creation
- transfer-coding details

### 504 trap

`504 Gateway Timeout` was **not** introduced in 1999.

The PDF explicitly says it was already in RFC 2068.

---

# 15. HTTP Message Framing

One of the most important concepts in HTTP/1.1:

> The receiver must know where one message ends and the next begins.

Common framing mechanisms include:

```text
Content-Length
Transfer-Encoding: chunked
connection close
```

### Dangerous ambiguity

Suppose a request contains both:

```http
Content-Length: 6
Transfer-Encoding: chunked
```

Different components may interpret the message differently.

That leads to:

# Request Smuggling

One component may see:

```text
one request
```

while another sees:

```text
two requests
```

The PDF describes this as:

> One byte stream. Two readings. One attack.

This is a major MCQ/security concept.

---

# 16. HTTP/1.1 Request Smuggling

Example concept:

```text
POST /echo
Content-Length: 6
Transfer-Encoding: chunked

0

GET /admin
```

If a proxy uses one framing rule and the origin uses another, they can disagree about where the request ends.

That can allow an attacker to inject a request the front-end did not recognize.

### HTTP/2 improvement

HTTP/2 makes this particular ambiguity unrepresentable because the frame has an explicit length field.

---

# 17. RFC 7230–7235 — 2014

HTTP/1.1 was split into six documents.

| RFC | Topic |
|---|---|
| 7230 | Message Syntax and Routing |
| 7231 | Semantics and Content |
| 7232 | Conditional Requests |
| 7233 | Range Requests |
| 7234 | Caching |
| 7235 | Authentication |

Think:

```text
One huge specification
        ↓
Six focused specifications
```

---

# 18. RFC 9110–9114 — 2022

The modern split is even cleaner.

| RFC | Meaning |
|---|---|
| 9110 | HTTP Semantics |
| 9111 | HTTP Caching |
| 9112 | HTTP/1.1 wire format |
| 9113 | HTTP/2 wire format |
| 9114 | HTTP/3 wire format |

### Extremely important mental model

```text
RFC 9110
    ↓
What HTTP means

RFC 9112
RFC 9113
RFC 9114
    ↓
How HTTP is encoded on the wire
```

The semantics are shared by HTTP/1.1, HTTP/2 and HTTP/3.

---

# 19. Why HTTP/1.1 Became Expensive

The PDF identifies five major costs:

| Problem | HTTP/2 / HTTP/3 solution |
|---|---|
| Head-of-line blocking | Stream IDs / multiplexing |
| Header repetition | HPACK |
| Too many connections | One multiplexed connection |
| Ambiguous framing | Binary frame headers |
| TCP setup/layering cost | QUIC / HTTP/3 |

---

# 20. Head-of-Line Blocking in HTTP/1.1

With pipelining:

```text
Request 1 → /big.txt → takes 2 seconds
Request 2 → /style.css → ready immediately
Request 3 → /app.js → ready immediately
```

But responses must be returned in order.

Therefore:

```text
/big.txt
   ↓
/style.css waits
   ↓
/app.js waits
```

The PDF measures approximately:

```text
pipelined:
all responses ≈ 2003 ms

separate connections:
fast responses ≈ 3–7 ms
slow file ≈ 2006 ms
```

---

# 21. Why HTTP/1.1 Has This HOL Problem

The crucial sentence:

> There is no request ID on the wire.

The client knows which response belongs to which request primarily by **order**.

Therefore:

```text
order = identifier
```

And if order is the identifier:

```text
response #1 must come before response #2
```

This prevents true out-of-order multiplexing.

---

# 22. Multiplexing Requires Identity

The PDF gives this pattern:

| Protocol | Identifier |
|---|---|
| TCP | Source/destination ports as part of connection identity |
| SS7/TCAP | Transaction ID |
| FastCGI | requestId |
| HTTP/1.1 | No request ID |
| HTTP/2 / HTTP/3 | Stream ID |

### Key idea

If multiple messages are going to share a connection and be processed independently, the messages need some form of identity.

---

# 23. HTTP/1.1 Header Bloat

The PDF gives an example:

```text
805 bytes headers/request
80 requests
= 64,320 bytes headers
```

Only:

```text
1,600 bytes
```

were new path information.

The repeated bytes were:

```text
62,720 bytes
```

or:

```text
97.5%
```

This motivates header compression in HTTP/2.

---

# 24. Nagle + Delayed ACK

The PDF demonstrates a subtle TCP interaction.

### Nagle

RFC 896:

> Avoid sending a small segment while an earlier one is unacknowledged.

### Delayed ACK

RFC 1122:

> The receiver may delay an ACK, waiting to piggyback it.

Together, they can create delays on persistent connections.

The example:

```text
write(head)
write(body)
```

can become:

```text
head sent
↓
body held by Nagle
↓
ACK delayed
↓
~40 ms
↓
body sent
```

Using:

```text
TCP_NODELAY
```

in the example reduces the measured total from about:

```text
174.3 ms → 3.2 ms
```

### MCQ trap

This interaction appears with a **persistent connection**. Closing the socket can force data out.

---

# 25. HTTP/2 — Why It Happened

SPDY came first.

Timeline:

```text
2009 → SPDY announced
2010–2014 → major browsers/servers deploy it
2015 → HTTP/2 standardized
2016 → SPDY removed from Chrome
2022 → RFC 9113 obsoletes RFC 7540
```

HTTP/2 was strongly influenced by SPDY.

---

# 26. HTTP/2 Connection Preface

The client sends exactly:

```text
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```

This is:

```text
24 octets
```

The strange-looking preface lets an HTTP/1.1 server fail cleanly instead of interpreting binary HTTP/2 frames as normal HTTP/1.1 text.

---

# 27. HTTP/2 over TLS — ALPN

During TLS negotiation:

```text
ClientHello:
    h2
    http/1.1

Server:
    chooses h2
```

This uses:

```text
ALPN
```

Application-Layer Protocol Negotiation.

The PDF notes that this adds no extra round trip because the negotiation rides inside the TLS handshake already taking place.

---

# 28. HTTP/2 Frame Header

This is one of the highest-value numbers in Session 5.

Every HTTP/2 frame begins with a:

# 9-byte header

Layout:

```text
Length              24 bits
Type                  8 bits
Flags                 8 bits
R                     1 bit
Stream Identifier    31 bits
```

So:

```text
24 + 8 + 8 + 1 + 31 = 72 bits = 9 bytes
```

The **31-bit stream identifier** is the critical new identity mechanism.

---

# 29. HTTP/2 Streams

A single TCP connection can carry many streams.

Example:

```text
TCP connection
 ├── stream 1 → /big.txt
 ├── stream 3 → /style.css
 ├── stream 5 → /app.js
 ├── stream 7 → /logo.png
 └── stream 9 → /hero.png
```

Client-initiated streams use odd IDs.

Server-initiated streams use even IDs.

Stream 0 represents the connection itself.

---

# 30. HTTP/2 Frame Types

Important frame types:

| Type | Frame | Purpose |
|---|---|---|
| 0x00 | DATA | Body data |
| 0x01 | HEADERS | Headers / opens stream |
| 0x02 | PRIORITY | Deprecated |
| 0x03 | RST_STREAM | Cancel one stream |
| 0x04 | SETTINGS | Connection settings |
| 0x05 | PUSH_PROMISE | Server push |
| 0x06 | PING | RTT / liveness |
| 0x07 | GOAWAY | Graceful shutdown |
| 0x08 | WINDOW_UPDATE | Flow control |
| 0x09 | CONTINUATION | Continue a large header block |

### High-value distinction

`RST_STREAM`:

```text
cancel ONE stream
```

It does not kill the entire TCP connection.

That is a major improvement over HTTP/1.1 behavior.

---

# 31. HTTP/2 Important Defaults

The PDF gives these defaults:

```text
HEADER_TABLE_SIZE       = 4096
INITIAL_WINDOW_SIZE     = 65535
MAX_FRAME_SIZE          = 16384
```

`MAX_CONCURRENT_STREAMS` has no universal fixed default in the cited material.

---

# 32. HPACK

HTTP/2 uses:

# HPACK

Its three important mechanisms are:

1. Static table
2. Dynamic table
3. Huffman coding

---

## 32.1 Static Table

HPACK has:

```text
61 static entries
```

They are fixed and identical everywhere.

Example:

```text
:index 2 = :method: GET
```

So common headers can be represented by a small index.

---

## 32.2 Dynamic Table

Starts from index:

```text
62
```

It stores values previously sent on **this connection**.

Example:

```text
first request:
300-byte cookie

later request:
1–2 byte index
```

This is why HTTP/2 benefits from keeping a single connection.

---

## 32.3 Huffman Coding

HPACK also supports a fixed Huffman coding table.

The PDF gives examples such as:

```text
"e" → 5 bits
"X" → 8 bits
```

It is only used when it actually makes the representation smaller.

---

# 33. HPACK Compression Example

The PDF demonstrates:

```text
HTTP/1.1 text      = 174 bytes
HPACK first time   = 79 bytes
HPACK second time  = 6 bytes
```

The second request can be tiny because the connection's dynamic table is already warmed up.

### Key concept

The dynamic table is **per connection**.

Therefore domain sharding, which creates multiple connections, becomes harmful to HPACK efficiency.

---

# 34. HPACK and CRIME

Why not simply gzip all headers?

Because compression can leak information through compressed size.

If attacker-controlled input is compressed alongside a secret, the attacker may infer parts of the secret from size differences.

The PDF's simplified example:

```text
guess = wrong → compressed size 412
guess = correct → compressed size 411
```

A one-byte difference can leak information.

HPACK's design avoids an adaptive compression dictionary in the same way.

Sensitive fields can also be marked:

```text
never indexed
```

---

# 35. HTTP/2 Multiplexing

HTTP/1.1:

```text
one connection
requests in order
responses in order
```

HTTP/2:

```text
one connection
many streams
frames from streams can interleave
responses can arrive out of order
```

Example:

```text
stream 3 → style.css → 4.9 ms
stream 7 → logo.png  → 5.7 ms
stream 9 → icon.png  → 6.9 ms
stream 1 → big.txt   → 2005.9 ms
```

The fast resources no longer have to wait for the slow resource at the HTTP layer.

---

# 36. Important HTTP/2 Trap

HTTP/2 provides multiplexing capability.

But the application can still accidentally serialize the work.

Example:

```python
for event in conn.receive_data(data):
    self.serve(event.stream_id)
```

If `serve()` blocks, everything can still wait.

Another design might use:

```text
one thread per stream
```

The lesson:

> A protocol can give you permission to run concurrently; your implementation can still serialize the work.

---

# 37. HTTP/2 Server Push

HTTP/2 server push was standardized.

But real-world deployment largely abandoned it.

The PDF says:

- Chrome removed support in 2022.
- Only a small fraction of HTTP/2 sites used it.
- The server often does not know what the client already has cached.
- Pushing an already-cached resource wastes bandwidth.

The PDF identifies:

# 103 Early Hints

as a replacement direction.

Instead of sending the resource, the server can tell the client:

```text
You are probably going to need style.css
```

Then the client checks its own cache and decides.

---

# 38. HTTP/2's Remaining Problem

HTTP/2 multiplexes streams **above TCP**.

But TCP is still:

```text
one ordered byte stream
```

Suppose one TCP packet is lost.

TCP cannot deliver later bytes to the application until the missing bytes are recovered.

Therefore:

```text
TCP packet loss
      ↓
TCP waits for missing bytes
      ↓
multiple HTTP/2 streams may stall
```

This is:

# TCP-level Head-of-Line Blocking

The PDF experiment shows a 300 ms stall affecting streams whose bytes were behind the gap.

---

# 39. QUIC — Why UDP?

QUIC uses UDP as its underlying IP protocol.

Protocol numbers in the PDF:

```text
TCP  = 6
UDP  = 17
SCTP = 132
```

Why not create another transport protocol directly?

Because many middleboxes:

```text
NATs
firewalls
CGNATs
corporate proxies
routers
```

may drop unfamiliar protocols.

UDP is already widely permitted.

So QUIC effectively uses:

```text
UDP
+
QUIC transport
+
TLS
+
HTTP/3
```

---

# 40. QUIC Lives in User Space

A key design difference:

```text
TCP changes
→ usually require kernel/network-stack updates

QUIC changes
→ can be shipped with application/software updates
```

The PDF describes QUIC as a userspace protocol.

---

# 41. QUIC + HTTP/3 Handshake

Traditional:

```text
TCP + TLS 1.3 + HTTP/2
≈ 3 RTT cold
```

QUIC + HTTP/3:

```text
≈ 1 RTT handshake
≈ 2 RTT until response in the PDF's cold connection example
```

The important conceptual point:

> QUIC combines the transport and cryptographic handshake rather than doing separate TCP and TLS handshakes.

The PDF's measured example at 150 ms RTT:

```text
TCP + TLS 1.3 + h2
≈ 456.6 ms

QUIC + HTTP/3
≈ 319.8 ms
```

Do not memorize these benchmark values as universal; understand the RTT reduction.

---

# 42. QUIC 0-RTT

On a resumed connection, QUIC can send application data immediately:

```text
0-RTT
```

PDF example:

```text
TCP + TLS 1.3 + h2        ≈ 456.6 ms
QUIC 1-RTT                ≈ 319.8 ms
QUIC 0-RTT                ≈ 161.7 ms
```

### Security trade-off

0-RTT data can be replayed.

Therefore it is safest for operations that are replay-safe/idempotent.

Good example:

```text
GET
```

Dangerous example:

```text
POST that charges a credit card
```

The PDF mentions:

```text
425 Too Early
```

as a possible response when early data should not be processed.

---

# 43. HTTP/3 Removes TCP HOL

HTTP/2:

```text
HTTP streams
      ↓
TCP
      ↓
one ordered byte stream
```

HTTP/3:

```text
HTTP streams
      ↓
QUIC streams
      ↓
UDP
```

Each QUIC stream has its own:

- stream ID
- offset

Data is ordered **within a stream**, but streams do not depend on one global ordered byte stream.

So:

```text
stream A loses data
        ↓
stream A waits

stream B did not lose data
        ↓
stream B can continue
```

This is the major HTTP/3 advantage.

---

# 44. QUIC Connection IDs

TCP connection identity is tied to:

```text
source IP
source port
destination IP
destination port
```

This is the TCP four-tuple.

Change the IP:

```text
Wi-Fi → LTE
```

and the TCP connection normally dies.

QUIC uses a:

# Connection ID

The connection can survive an address change.

Example:

```text
Wi-Fi
  ↓
IP changes
  ↓
LTE
  ↓
same QUIC connection ID
  ↓
connection continues
```

This is called connection migration.

---

# 45. QUIC Connection ID Privacy

A stable visible identifier could become a tracking identifier.

Therefore QUIC connection IDs must not expose information that allows an external observer to correlate a user across connections.

The PDF notes that endpoints can issue IDs from a pool and rotate them.

---

# 46. QPACK — HTTP/3 Header Compression

HTTP/2 uses:

```text
HPACK
```

HTTP/3 uses:

```text
QPACK
```

Why?

Because HPACK's dynamic table is shared mutable state.

With TCP:

```text
everything is ordered
```

So if an entry is inserted before another frame references it, the decoder knows what it means.

QUIC intentionally allows streams to progress independently.

So a stream might reference an entry that has not arrived yet.

That could reintroduce blocking.

---

# 47. QPACK's Solution

QPACK moves dynamic table updates onto dedicated streams.

Conceptually:

```text
Encoder stream
    ↓
table insertions

Decoder stream
    ↓
acknowledges what it has seen
```

The request streams are separated from this mutable table state.

### Key principle

> Shared mutable state + out-of-order delivery = danger.

This is one of the most important conceptual lessons of the PDF.

---

# 48. QPACK Numbers

The PDF gives:

```text
QPACK_BLOCKED_STREAMS default = 0
QPACK_MAX_TABLE_CAPACITY default = 0
```

Static tables:

```text
HPACK → 61 entries
QPACK → 99 entries
```

---

# 49. HTTP/3 Discovery

How does a client know HTTP/3 is available?

### ALPN

Negotiated during TLS:

```text
h2
h3
```

### Alt-Svc

Server response can advertise:

```http
Alt-Svc: h3=":443"
```

The browser remembers it.

### HTTPS DNS record

Modern DNS can advertise protocol information using:

```text
HTTPS DNS RR
```

The PDF identifies this as DNS record type 65.

---

# 50. What If UDP Is Blocked?

The PDF cites measurements suggesting a minority of networks block UDP traffic.

So clients need fallback behavior.

Conceptually:

```text
Try QUIC / HTTP/3
      ↓
UDP blocked?
      ↓
Fallback to TCP / HTTP/2
```

But fallback introduces a downgrade surface because the network can influence which path gets used.

---

# 51. HTTP Adoption

The PDF's 2024 Web Almanac figures:

By request:

```text
HTTP/1.1 ≈ 15%
HTTP/2+  ≈ 85%
```

By homepage:

```text
HTTP/1.1 ≈ 21–22%
HTTP/2   ≈ 70–71%
HTTP/3   ≈ 7–9%
```

The important engineering takeaway:

> HTTP/1.1 is still alive.

Legacy systems, internal services, embedded devices, gateways, APIs and old load balancers can keep HTTP/1.1 around for years.

---

# 52. The Four Core Ideas

The final slide gives four ideas.

## 1. Multiplexing needs identity

HTTP/1.1:

```text
no request ID
→ order becomes identity
→ strict ordering
→ HOL blocking
```

HTTP/2:

```text
31-bit stream IDs
→ independent streams
```

HTTP/3:

```text
QUIC stream IDs
→ independent transport streams
```

---

## 2. Every fix creates the next problem

```text
Persistence
    ↓
pipelining
    ↓
HTTP HOL blocking
    ↓
HTTP/2 multiplexing
    ↓
TCP HOL blocking
    ↓
QUIC
    ↓
HPACK no longer fits perfectly
    ↓
QPACK
```

There is no final perfect protocol.

There is a sequence of engineering trade-offs.

---

## 3. Semantics and encoding are different

```text
RFC 9110
    ↓
what HTTP means

RFC 9112
    ↓
HTTP/1.1 encoding

RFC 9113
    ↓
HTTP/2 encoding

RFC 9114
    ↓
HTTP/3 encoding
```

The same HTTP semantics can be represented using different wire formats.

---

## 4. Deployed reality beats specification

The PDF gives examples:

- 302 behavior influenced the creation of 307.
- Browsers used more connections than the old recommendation.
- HTTP/2 server push was standardized but largely abandoned.
- QUIC uses UDP partly because middleboxes control which protocols survive the public Internet.

---

# 53. Highest-Value Numbers

Memorize these for MCQs:

```text
HTTP/1.0
→ RFC 1945
→ May 1996

Early HTTP/1.1
→ RFC 2068
→ January 1997

HTTP/1.1
→ RFC 2616
→ June 1999

HTTP/1.1 split
→ RFC 7230–7235
→ 2014

Modern HTTP
→ RFC 9110–9114
→ 2022

HTTP/2
→ RFC 7540 → RFC 9113

HTTP/3
→ RFC 9114

QUIC
→ RFC 9000

HTTP/2 frame header
→ 9 bytes

HTTP/2 stream ID
→ 31 bits

HPACK static table
→ 61 entries

HPACK dynamic table starts
→ index 62

HPACK example
→ 174 → 79 → 6 bytes

MAX_FRAME_SIZE default
→ 16384

INITIAL_WINDOW_SIZE default
→ 65535

HEADER_TABLE_SIZE default
→ 4096

HTTP/2 client stream IDs
→ odd

HTTP/2 server stream IDs
→ even

HTTP/2 connection stream
→ stream 0

HTTP/2 preface
→ 24 octets

QUIC
→ UDP

TCP protocol number
→ 6

UDP protocol number
→ 17

SCTP protocol number
→ 132

HTTP/3 0-RTT
→ faster, but replay risk
```

---

# 54. MCQ Trap Sheet

### Trap 1
**HTTP/1.0 has true pipelining HOL blocking.**

❌ False.

HTTP/1.0 did not pipeline. Its main cost was serialization / one round trip per object.

---

### Trap 2
**HTTP/1.1 requires Host.**

✅ Yes.

---

### Trap 3
**HTTP/2 uses one TCP connection and one request ID.**

❌ No.

HTTP/2 uses many **streams**, each identified by a **31-bit stream ID**.

---

### Trap 4
**HTTP/2 eliminates all head-of-line blocking.**

❌ No.

It removes HTTP-level HOL caused by response ordering, but TCP can still cause cross-stream HOL.

---

### Trap 5
**HTTP/3 uses TCP.**

❌ No.

HTTP/3 runs over QUIC, which runs over UDP.

---

### Trap 6
**QUIC is insecure because UDP has no encryption.**

❌ Wrong reasoning.

QUIC integrates TLS and requires encryption.

---

### Trap 7
**QUIC 0-RTT is always safe.**

❌ No.

Replay attacks are the important trade-off.

---

### Trap 8
**HPACK and QPACK are the same.**

❌ No.

HPACK is for HTTP/2; QPACK is for HTTP/3.

---

### Trap 9
**ETag is just another timestamp.**

❌ No.

ETag is a representation validator/fingerprint.

---

### Trap 10
**`no-cache` means don't store.**

❌ No.

`no-cache` means revalidate before reuse.

`no-store` means do not store.

---

### Trap 11
**Range request returns 200.**

Usually ❌.

A successful partial response uses:

```text
206 Partial Content
```

---

### Trap 12
**416 means malformed HTTP request.**

❌ Not exactly.

416 means the requested range cannot be satisfied.

---

### Trap 13
**RST_STREAM closes the whole HTTP/2 connection.**

❌ No.

It cancels one stream.

---

### Trap 14
**HTTP/2 server push was deprecated by RFC 9113.**

❌ According to this session's source, no.

It was retained in the RFC, although implementations largely removed/abandoned it.

---

### Trap 15
**The main reason HTTP/2 is binary is that binary is inherently faster than text.**

❌ Oversimplification.

The session's key point is:

> HTTP/2 adds explicit lengths and identifiers; binary framing follows from that design.

---

# 55. One-Minute Revision

```text
HTTP/1.0
    ↓
close after response
    ↓
expensive repeated connections
    ↓
no Host
    ↓
Last-Modified caching

HTTP/1.1
    ↓
persistent connections
    ↓
Host
    ↓
ETag
    ↓
Range
    ↓
chunked encoding
    ↓
pipelining
    ↓
HOL blocking

HTTP/2
    ↓
binary frames
    ↓
31-bit stream IDs
    ↓
multiplexing
    ↓
HPACK
    ↓
HTTP HOL reduced
    ↓
TCP HOL remains

HTTP/3
    ↓
QUIC over UDP
    ↓
independent streams
    ↓
TCP HOL gone
    ↓
connection migration
    ↓
QPACK
    ↓
0-RTT
```

---

# 56. Final Exam Mental Model

If a question asks:

### "Why HTTP/1.1?"
Think:

**Persistence + Host + better caching + ranges + chunking.**

### "Why HTTP/2?"
Think:

**Streams + IDs + binary frames + multiplexing + HPACK.**

### "Why HTTP/3?"
Think:

**QUIC + UDP + independent streams + no TCP HOL + migration + 0-RTT.**

### "What is the deepest lesson?"
Think:

> **Multiplexing needs identity. Every fix creates a new constraint. Semantics can stay stable while wire encoding changes.**

---

# 57. Source Homework / Practical Tasks

The PDF lists five homework tasks:

1. Extend `fetch_page.py` to N connections and find the latency knee at 150 ms RTT.
2. Build ETags from mtime and modify a file twice within one second.
3. Decode a HEADERS frame's HPACK block by hand.
4. Change `SETTINGS_QPACK_BLOCKED_STREAMS` from 0 to 16.
5. Block UDP with iptables and measure HTTP/3 fallback.

The PDF specifically calls **task 3** the one to do if only one task is completed.

