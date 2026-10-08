# Computer Networks — Session 8
## Streaming Video at Large Scale
### MCQ + Interview + System Design Notes

> **Session 8 is the synthesis session:** the socket, HTTP, CDN, failure, and caching ideas from the earlier weeks are combined into one live-video system serving tens of millions of viewers.

---

# 0. The Entire Session in One Picture

The central system is:

```text
                         LIVE MATCH
                             │
              ┌──────────────┴──────────────┐
              │                             │
        SCORE / EVENTS                 VIDEO
              │                             │
      ┌───────┼────────┐           ┌────────┼─────────┐
      │       │        │           │        │         │
   Polling   SSE     WebSocket    HLS     CDN       ABR
      │       │        │           │        │         │
     304     held     frames     playlist  cache   bitrate
      │       │        │           │        │         │
      └───────┴────────┴───────────┴────────┴─────────┘
```

The session's main question is:

> **How do you deliver live, changing information to ~50 million phones without buying 50 million expensive long-lived server connections?**

The answer is not one protocol. It is choosing the right representation and delivery model for each kind of data.

---

# 1. Session 8 Agenda

The session is organized into eight parts:

1. **The brief**
   - one scorecard
   - polling vs push
   - WebSockets, SSE, MQTT

2. **What it costs**
   - held sockets
   - Little's Law
   - request vs byte billing
   - four architectures

3. **Making polling survive**
   - 304
   - TTLs
   - jitter
   - request collapsing
   - negative caching
   - diffs

4. **Same loop, bigger bytes**
   - HLS playlists
   - segments
   - 188-byte MPEG-TS packets

5. **Buffers and latency**
   - playback buffer
   - latency budget
   - spoilers
   - ABR

6. **Many origins, many CDNs**
   - steering
   - failover
   - stale playlists
   - cache rules

7. **Making the stream**
   - FFmpeg
   - HLS ladder
   - segment duration

8. **Closing**
   - four ideas
   - assignment
   - homework

The session agenda explicitly frames these as the major blocks.

---

# 2. The Big Problem: One Scorecard, 50 Million Phones

Imagine a cricket match.

The score changes only when something happens:

```text
25–45 seconds between balls
```

but:

```text
50M concurrent viewers
```

and roughly:

```text
20M average viewers
```

The scorecard itself is tiny:

```text
~4–5 KB JSON
~1 KB gzipped
```

So the interesting problem is not data size.

It is:

```text
HOW DO WE DELIVER A TINY CHANGING OBJECT
TO TENS OF MILLIONS OF CLIENTS?
```

---

# 3. Two Families of Architectures

The session contrasts:

## Pull / non-persistent

```text
client asks
   ↓
server answers
   ↓
connection ends
```

Examples:

```text
Polling
Conditional GET
Caching
CDN
```

Properties:

```text
stateless
cacheable
any server can answer
CDN-friendly
```

## Push / persistent

```text
client connects
   ↓
connection stays open
   ↓
server pushes updates
```

Examples:

```text
Long polling
SSE
WebSockets
MQTT
raw sockets
```

Properties:

```text
stateful
held connection
per-client server/socket cost
harder to cache
```

The source explicitly summarizes this distinction: polling uses ordinary GETs and can sit behind a CDN, while persistent approaches hold a connection for the match.

---

# 4. Polling

Basic polling:

```text
GET /score
wait
GET /score
wait
GET /score
...
```

Example:

```text
every 15 seconds
```

Advantages:

```text
simple
stateless
cacheable
CDN-friendly
easy horizontal scaling
```

Disadvantages:

```text
many requests
wasted requests when nothing changed
latency up to polling interval
```

---

# 5. Polling Freshness

If:

```text
poll interval = 15 s
```

then a change may take almost:

```text
15 s
```

to be noticed.

Worst-case freshness contribution:

```text
≈ polling interval
```

Then add cache TTLs.

The session's rule:

```text
worst-case staleness
=
poll interval
+
every cache TTL on the path
```

So:

```text
15 s poll
+
5 s edge TTL
+
5 s mid-tier TTL
=
25 s possible staleness
```

---

# 6. Conditional GET: 304

Instead of downloading the same scorecard every time:

```http
GET /match/412/score.json
If-None-Match: "v1841"
```

If nothing changed:

```http
HTTP/1.1 304 Not Modified
ETag: "v1841"
```

The client keeps its existing copy.

### Important

304 saves:

```text
response body
JSON parsing
rendering
```

but does NOT eliminate:

```text
request
round trip
headers
request billing
```

This is a major MCQ trap.

---

# 7. ETag as Version Number

For a changing scorecard, a useful design is:

```text
ETag = version
```

Example:

```text
ETag: "v1841"
```

Think of ETag as:

```text
fingerprint of representation
```

not:

```text
clock
```

`If-Modified-Since` can also be used, but timestamp precision is less useful for rapidly changing state.

---

# 8. Why Polling Can Still Scale

Suppose:

```text
50M clients
```

poll every:

```text
30 s
```

Peak request rate:

```text
50M / 30
≈ 1.67M requests/s
```

That sounds enormous.

But if:

```text
the response is cacheable
```

the origin does NOT necessarily receive 1.67M requests/s.

The CDN/edge absorbs most of them.

This is the key:

```text
PULL SCALES THROUGH CACHES
```

---

# 9. Why a Versioned Scorecard Is CDN-Friendly

The scorecard is:

```text
same answer for everyone
```

Therefore:

```text
client
  ↓
CDN edge
  ↓
cached scorecard
```

The edge can answer many viewers without contacting origin.

Contrast with:

```text
personalized data
```

where every viewer may need a different response.

---

# 10. The Four Main Delivery Choices

For the same scorecard:

| Approach | Main resource |
|---|---|
| Polling | requests |
| Long polling | held sockets |
| SSE | held sockets |
| WebSocket / MQTT | held sockets + broker/server state |

The key tradeoff:

```text
PULL
more requests
less connection state

PUSH
fewer explicit requests
more persistent connection state
```

---

# 11. Long Polling

Long polling:

```text
GET /score?after=1841
```

Server does not immediately respond.

Instead:

```text
wait for next change
```

When score changes:

```text
HTTP 200
new score
```

Client immediately asks again:

```text
GET /score?after=1842
```

### It looks like push

but technically:

```text
client starts every request
```

So it is still HTTP request/response.

---

# 12. Long Polling Cost

A held long-poll request is:

```text
a held socket
```

If:

```text
50M viewers
```

hold requests for most of the match:

```text
≈ 50M concurrent connections
```

Now you have a stateful scaling problem.

This is why:

```text
polling
```

and:

```text
long polling
```

can have radically different infrastructure costs even though both use HTTP.

---

# 13. Long Polling + Herd Synchronization

Long polling has one interesting property.

If:

```text
20M clients
```

are waiting for a score update:

```text
wicket happens
```

many requests can complete together.

Then:

```text
20M clients
↓
immediately issue next request
```

This can create a synchronized request spike.

So long polling can trade:

```text
periodic polling herd
```

for:

```text
event-driven response herd
```

---

# 14. Long Polling and Timeouts

A held request must not exceed intermediary idle/read timeouts.

The session references:

```text
nginx proxy_read_timeout
ALB idle timeout
```

around:

```text
60 s
```

Therefore the server should answer before the connection is killed.

A common pattern:

```text
hold for <= timeout budget
or until event arrives
```

---

# 15. SSE — Server-Sent Events

SSE is:

```text
one HTTP response
that never ends
```

Example:

```http
GET /match/412/events
Accept: text/event-stream
```

Server sends:

```text
id: 1842
event: ball
data: {...}

id: 1843
event: ball
data: {...}
```

Each event is separated by:

```text
blank line
```

### Direction

```text
SERVER → CLIENT
```

Only.

If the client needs to send something:

```text
ordinary HTTP request
```

---

# 16. SSE Framing

SSE uses delimiter-based framing.

Conceptually:

```text
event
event
event

```

A blank line means:

```text
event complete
```

This connects directly to Session 2:

```text
framing can be length-based
or delimiter-based
```

SSE uses:

```text
delimiter
```

rather than a binary length field.

---

# 17. SSE Reconnection

SSE supports:

```http
Last-Event-ID: 1841
```

Client reconnects and tells server:

```text
"I last received event 1841."
```

Server can replay:

```text
1842
1843
1844
...
```

Therefore:

```text
versioned events
+
Last-Event-ID
```

make the stream resumable.

---

# 18. SSE Retry Field

SSE can include:

```text
retry: 5000
```

meaning the browser should wait approximately:

```text
5 seconds
```

before reconnecting.

This is protocol-level reconnect guidance.

---

# 19. SSE Browser Connection Limit

A key practical issue:

With HTTP/1.1:

```text
each browser tab
≈ separate SSE connection
```

The session uses:

```text
about six per browser
```

as the relevant browser-era constraint.

With HTTP/2:

```text
multiple streams
```

can be multiplexed over one connection.

This is why the homework asks you to open seven SSE tabs under HTTP/1.1 and observe one waiting, then repeat under HTTP/2.

---

# 20. WebSockets

WebSockets begin as HTTP:

```http
GET /live HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: ...
Sec-WebSocket-Version: 13
```

Server responds:

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: ...
```

Then:

```text
HTTP is finished as the application protocol
```

and the connection becomes:

```text
two-way framed WebSocket communication
```

---

# 21. Why 101?

HTTP status:

```text
101 Switching Protocols
```

means:

```text
server agrees to switch protocols
```

This is the key WebSocket handshake status.

---

# 22. Sec-WebSocket-Accept

The server computes:

```text
base64(
    SHA-1(
        Sec-WebSocket-Key
        +
        fixed WebSocket GUID
    )
)
```

and returns:

```http
Sec-WebSocket-Accept: ...
```

The client validates it.

### MCQ trap

It is:

```text
SHA-1
+
fixed GUID
+
Base64
```

Not:

```text
plain Base64(key)
```

---

# 23. WebSocket Frames

After the upgrade, each frame begins with:

```text
FIN | RSV | opcode
MASK | payload length
```

Then potentially:

```text
extended length
masking key
payload
```

---

# 24. WebSocket FIN / Opcode

The first byte contains:

```text
FIN
RSV
opcode
```

### FIN

Indicates:

```text
final fragment?
```

### Opcode

Identifies frame type.

Important opcodes:

```text
text
binary
ping
pong
close
```

---

# 25. WebSocket Payload Length

Length encoding:

```text
0–125
→ length directly

126
→ next 2 bytes contain length

127
→ next 8 bytes contain length
```

This is a length-based framing system.

Compare:

```text
SSE → delimiter
WebSocket → length
```

---

# 26. WebSocket Masking

Client-to-server WebSocket frames are masked.

Frame includes:

```text
masking key
```

The purpose emphasized in the session:

```text
prevent stray bytes from looking like a normal HTTP request
to a caching/intermediary proxy
```

### MCQ trap

```text
Client → Server
```

frames are masked.

Do not casually say:

```text
all WebSocket frames are masked
```

---

# 27. WebSocket vs SSE

| Property | SSE | WebSocket |
|---|---|---|
| Direction | server → client | both directions |
| Starts with HTTP | yes | yes |
| Persistent | yes | yes |
| Framing | delimiter | binary frame |
| Resume support | Last-Event-ID | application-defined |
| Best for | server push | interactive two-way communication |

---

# 28. WebSocket vs Polling

### Polling

```text
request
response
connection may close
repeat
```

### WebSocket

```text
HTTP handshake
↓
persistent socket
↓
frames in either direction
```

Polling is easier to cache and scale statelessly.

WebSockets are better when:

```text
low-latency bidirectional communication
```

is actually needed.

---

# 29. MQTT

MQTT introduces a broker:

```text
publisher
    ↓
  broker
   / \
  /   \
subscriber subscriber
```

Example:

```text
topic = match/412/score
```

The scorer publishes once:

```text
PUBLISH match/412/score
```

The broker fans it out to all subscribers.

---

# 30. MQTT QoS 1

QoS 1 means:

```text
at least once
```

Therefore duplicates can happen.

This means the application should make updates safe to duplicate.

The session's example uses:

```text
version number
```

so duplicate updates become harmless.

### Important

```text
QoS 1 ≠ exactly once
```

---

# 31. MQTT Retained Message

A retained message lets the broker keep:

```text
latest value
```

A new subscriber can immediately receive it.

This is conceptually:

```text
cache inside broker
```

Example:

```text
latest score = v1842
```

New phone subscribes:

```text
SUBSCRIBE
↓
broker immediately sends v1842
```

---

# 32. MQTT Keepalive

The example uses:

```text
keepalive = 60
```

and:

```text
PINGREQ
PINGRESP
```

The session highlights that keepalives are tiny but still cost energy/network activity because they:

```text
wake the phone radio
+
keep NAT mappings alive
```

---

# 33. MQTT Remaining Length

MQTT uses a variable-length integer.

Conceptually:

```text
7 data bits per byte
+
top bit = more bytes
```

This is another example of:

```text
type
length
value
```

style framing.

---

# 34. The Hotstar 50M Socket Story

The session uses Hotstar's architecture as the large-scale push case.

The progression:

```text
1. One node
   EMQX on c5.4xlarge
   ≈250k connections

2. Cluster
   publisher separated from subscribers
   ≈2M subscribers

3. ELB wall
   ≈500k without pre-warming
   ≈4M per pre-warmed Classic ELB

4. Five clusters
   5 × 8 subscribe nodes
   >10M

5. Reverse bridge
   Go service on subscriber side
   ≈200 nodes
   ≈50M connections
```

The important lesson:

> **Persistent connections scale through machines.**

The architecture does not magically make 50M sockets cheap.

It distributes them.

---

# 35. Why the Publisher Is Separated

A useful architecture:

```text
             publisher
                │
        ┌───────┼────────┐
        ↓       ↓        ↓
     cluster  cluster  cluster
        ↓       ↓        ↓
      many    many      many
    subscribers
```

The publisher's job is:

```text
produce event once
```

The subscriber side handles:

```text
huge connection fan-out
```

This prevents the connection fleet from making the event producer itself the bottleneck.

---

# 36. Why ELB Becomes a Wall

A load balancer can distribute connections.

But:

```text
the load balancer itself holds state for every connection
```

Therefore:

```text
more persistent clients
→ more LB state
→ LB scaling limit
```

This is why Hotstar had to scale beyond a single ELB.

---

# 37. Reverse Bridge

The Hotstar solution eventually uses:

```text
subscriber clusters
        ↓
Go reverse bridge
        ↓
publisher
```

The bridge lets many subscriber-side machines consume from a smaller publisher-side architecture.

The key design principle:

```text
separate connection scaling
from
message production
```

---

# 38. What a Held Socket Costs

Persistent connections consume:

### Memory

```text
kernel socket buffers
TLS state
broker/client state
application structures
```

The session's example:

```text
250k phones
on a 32 GiB box
≈137 KB per connection
```

This is an illustrative all-in number from the session.

### File descriptors

The default:

```text
ulimit -n
```

may begin around:

```text
1024
```

so the OS limit must be configured appropriately.

---

# 39. Machine Capacity

The session's board uses roughly:

```text
100k–200k connections/box
```

as a practical planning range.

Hotstar measured:

```text
250k
```

on its example machine.

Do NOT memorize:

```text
250k = universal maximum
```

The real capacity depends on:

```text
memory
CPU
TLS
application state
kernel tuning
traffic
keepalives
```

---

# 40. Keepalive Cost at Scale

Suppose:

```text
20M idle phones
```

send a keepalive every:

```text
60 s
```

Average rate:

```text
20M / 60
≈333k keepalives/s
```

before the first actual match event.

So even "idle" persistent connections generate work.

---

# 41. Ephemeral Port Limitation

One client IP cannot create unlimited TCP connections to the same destination.

The session gives:

```text
32768–60999
```

as an example ephemeral range:

```text
28,232 ports
```

Therefore:

```text
250k test sockets
```

from one source IP cannot work.

You need roughly:

```text
250,000 / 28,232
≈ 8.9
```

so:

```text
at least 9 source IPs
```

in that test setup.

---

# 42. Little's Law

One of the most important formulas:

```text
L = λ × W
```

Where:

```text
L = items in system
λ = arrival rate
W = time each item spends in system
```

---

# 43. Little's Law for Polling

Example:

```text
λ ≈ 1.67M requests/s
W ≈ 0.03 s
```

Therefore:

```text
L ≈ 1.67M × 0.03
  ≈ 50,000 requests in flight
```

So polling can have:

```text
50k concurrent requests
```

even though:

```text
50M users
```

exist.

---

# 44. Little's Law for Push

For persistent push:

```text
λ ≈ 50M connections established
W ≈ 4 hours
```

The important result is:

```text
≈50M sockets remain held
```

because:

```text
W is hours
```

rather than:

```text
milliseconds
```

### Big insight

> Same audience, dramatically different concurrency because the connection lifetime changes.

---

# 45. "Nothing Is Free Per Connection"

Persistent systems pay for:

```text
memory
file descriptors
kernel state
TLS state
keepalive traffic
load balancer state
reconnection storms
rebalancing
```

Therefore:

```text
"we only send one tiny message"
```

does NOT mean:

```text
the system is cheap
```

The connection itself is the expensive resource.

---

# 46. Four Architectures for One Match

The session compares:

| Design | Approx. per-match cost |
|---|---:|
| S3 straight to phones | ~$4.7K |
| nginx fleet + ALB | ~$1.2–1.8K |
| CloudFront India | ~$12.5K |
| Push: 200 × c5.4xlarge | ~$0.9–1.6K |

These are the session's illustrative Sep 2026 list-price calculations, not universal current pricing.

### Key conclusion

At those assumptions:

```text
push = cheapest per match
CDN = most expensive
```

But cost alone is not enough.

You also need:

```text
failure behavior
operational complexity
scaling limits
freshness
user experience
```

---

# 47. Request Fee vs Byte Fee

A CDN can charge for:

```text
number of requests
+
bytes transferred
```

For tiny objects:

```text
request fee dominates
```

For large objects:

```text
byte fee dominates
```

The session gives a break-even around:

```text
11 KB
```

under its CloudFront India price assumptions.

---

# 48. Scorecard vs Video Segment Economics

Scorecard:

```text
~1 KB
```

Playlist:

```text
~230 B
```

Segments:

```text
~450 KB – 1.6 MB
```

Therefore:

```text
scorecard/playlist
→ request-fee world

video segments
→ byte-fee world
```

This is a major system-design insight.

---

# 49. Four Core Cost Numbers

For the session's 50M-viewer scenario:

```text
30-second polling
≈ 1.67M requests/s peak

15-second polling
≈ 3.33M requests/s

30-second polling over a 4-hour match
≈ 9.6B requests

At ~1 KB each
≈ 9.6 TB
```

If polling every 15 seconds:

```text
≈19.2B requests
≈19.2 TB
```

---

# 50. Why Caching Changes the Economics

Without a cache:

```text
50M clients
↓
origin
```

With a cache:

```text
50M clients
↓
CDN edges
↓
few origin fetches
```

So caching transforms:

```text
many client requests
```

into:

```text
few upstream requests
```

This is why:

> **Pull scales through caches.**

---

# 51. Request Collapsing / Cache Lock

Suppose a cache entry expires.

Without collapsing:

```text
1,000 requests arrive
↓
1,000 origin requests
```

With request collapsing:

```text
1 request → origin
999 requests wait/share result
```

Nginx's:

```text
proxy_cache_lock
```

is an example.

This protects the origin from a cache-expiry stampede.

---

# 52. Thundering Herd in Polling

Suppose:

```text
all clients poll every 30 seconds
```

If synchronized:

```text
t=30
→ huge spike

t=60
→ huge spike

t=90
→ huge spike
```

Solution:

```text
jitter
```

The session's example uses:

```text
±20%
```

around a 15-second polling interval.

---

# 53. Polling Jitter Example

Base:

```text
15 s
```

Jitter:

```text
0.8 + random() × 0.4
```

Therefore:

```text
12 s to 18 s
```

instead of exactly:

```text
15 s
```

This spreads requests over time.

---

# 54. Negative Caching

Negative caching means caching errors such as:

```text
404
503
```

for a short period.

Why?

Imagine:

```text
origin struggling
+
50M clients requesting same object
```

If a 503 is cached for:

```text
2 seconds
```

the edge can answer:

```text
503
```

without sending every request to origin.

---

# 55. Negative Cache TTL Must Be Short

The session's example:

```text
200 → 10 s
404 → 1 s
500/502/503/504 → 2 s
```

The principle:

> A negative TTL must be much shorter than the time it takes the missing/failed thing to become available.

Otherwise:

```text
thing becomes available
but cache keeps saying "missing"
```

---

# 56. Why 404 Is Dangerous for Live Content

Normal static 404:

```text
resource truly does not exist
```

Caching is often fine.

Live segment:

```text
segment not available YET
```

Caching 404 can be disastrous.

Example:

```text
new segment being created
↓
temporary 404
↓
edge caches 404
↓
segment becomes available
↓
edge still returns 404
```

This can stall playback.

---

# 57. Stale Playlist Problem

A playlist changes frequently.

Example:

```text
playlist lists:
seg_000001 ... seg_000010
```

Origin removes:

```text
seg_000001
```

but CDN may still have an older playlist.

If the CDN serves that stale playlist:

```text
player requests seg_000001
↓
origin/CDN says 404
↓
playback stalls
```

This is why stale-if-error must be used carefully for live playlists.

---

# 58. Segment vs Playlist

This distinction is extremely important.

## Playlist

```text
small
changes every few seconds
control/index data
```

Stale copy can be harmful because:

```text
it may point to deleted segments
```

## Segment

```text
large
immutable once published
```

Stale copy is generally harmless.

This is why:

> **Segments can be cached almost forever; playlists need freshness.**

---

# 59. HLS: Playlist as Control, Segments as Data

HLS separates:

```text
playlist
```

from:

```text
media segments
```

Playlist tells the player:

```text
what segments exist
```

Segments contain:

```text
actual media
```

This is essentially:

```text
small changing index
+
large immutable objects
```

The same architecture appeared earlier as:

```text
versioned scorecard
+
numbered pieces
```

---

# 60. HLS Playlist Example

```m3u8
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:2

#EXTINF:4.000000,
seg_000002.ts

#EXTINF:4.000000,
seg_000003.ts
```

Important tags:

### `#EXT-X-TARGETDURATION`

Maximum/target segment duration guidance.

### `#EXT-X-MEDIA-SEQUENCE`

Sequence number of the first listed segment.

### `#EXTINF`

Duration of the following segment.

---

# 61. Versioned Scorecard = HLS Playlist

Conceptually:

```text
score.json
version = 1842
recent changes...
```

and:

```text
index.m3u8
MEDIA-SEQUENCE = 2
segment list...
```

are the same architecture:

```text
small changing index
+
numbered pieces
+
pieces can be cached
```

This is one of the most important synthesis ideas of the entire course.

---

# 62. Playlist Traffic vs Segment Traffic

The session gives approximately:

| | Playlist | Segment |
|---|---:|---:|
| Size | ~230 B gzipped | ~450 KB–1.6 MB |
| Changes | every ~4 s | never |
| Cache TTL | ≤ half target duration | minutes → forever |
| Stale copy | potentially harmful | generally harmless |
| Billing | requests | bytes |

---

# 63. 50M Players Polling HLS

If:

```text
target duration = 4 s
```

and every player polls every:

```text
4 s
```

then:

```text
50M / 4
=
12.5M requests/s
```

If they poll every:

```text
1 s
```

then:

```text
50M requests/s
```

This is why:

```text
playlist caching
+
reasonable polling
```

is essential.

---

# 64. MPEG-TS: 188-Byte Packets

Traditional HLS `.ts` segments can contain:

```text
188-byte MPEG transport stream packets
```

Each packet:

```text
4-byte header
+
184-byte payload
```

The header includes:

```text
sync byte
PID
continuity counter
```

---

# 65. The `0x47` Sync Byte

MPEG-TS packets begin with:

```text
0x47
```

This allows a receiver to identify packet boundaries.

The session connects this to framing:

```text
fixed-length packet
+
sync byte
```

---

# 66. PID

PID:

```text
Packet Identifier
```

is:

```text
13 bits
```

It identifies what stream/table the packet belongs to.

Examples conceptually:

```text
video PID
audio PID
PAT
PMT
```

Multiple types share the same transport stream.

---

# 67. Why 188 Bytes?

The historical transport-stream packet size is:

```text
188 bytes
```

The session notes its relationship to:

```text
ATM
```

This is a good MCQ fact.

---

# 68. MPEG-TS vs fMP4/CMAF

Traditional:

```text
.ts
→ fixed 188-byte packets
```

Modern CMAF/fMP4:

```text
boxes
→ size
→ type
→ payload
```

So:

```text
MPEG-TS = fixed-size packet framing
fMP4 = box/TLV-like framing
```

---

# 69. Segment Size Example

The session's measured example:

```text
720p  ≈ 1,613,416 B
480p  ≈   823,440 B
360p  ≈   452,328 B
```

The number of 188-byte packets is exactly:

```text
720p → 8,582
480p → 4,380
360p → 2,406
```

---

# 70. Buffer

A video player's buffer is:

```text
amount of playable media already downloaded
but not yet watched
```

If network slows:

```text
buffer decreases
```

If network is faster than playback:

```text
buffer increases
```

---

# 71. Buffer Is a Safety Margin

Think:

```text
buffer
=
time available to survive network variability
```

More buffer:

```text
more safety
higher latency
```

Less buffer:

```text
lower latency
less safety
more stalls
```

Therefore:

> **Latency is a budget you spend on safety.**

---

# 72. Latency Budget

Example in the session:

```text
encode                  ~2 s
segment fills            4 s
to CDN                   ~2 s
player hold-back        12 s
--------------------------------
≈20 s behind stadium
```

The exact network/encode numbers are illustrative, but the principle is important:

```text
latency is additive
```

---

# 73. Segment Duration Tradeoff

Longer segments:

```text
+ more buffering efficiency
+ fewer playlist/segment requests
- higher latency
- slower adaptation
```

Shorter segments:

```text
+ lower latency
+ faster ABR reaction
- more requests
- more overhead
- less buffer per segment
```

---

# 74. Spoiler Problem

Suppose:

```text
stadium event happens at t=0
```

Scorecard receives it:

```text
t≈10 s
```

Video shows it:

```text
t≈20 s
```

Then:

```text
score says WICKET
video hasn't shown wicket yet
```

This creates a:

```text
10-second spoiler
```

---

# 75. Fixing Spoilers

Stamp score updates with:

```text
wall-clock time
```

Example:

```json
{
  "v": 1842,
  "ball": "15.4",
  "at": "2026-09-27T03:16:44.1Z"
}
```

HLS can expose:

```text
#EXT-X-PROGRAM-DATE-TIME
```

The app can then:

```text
wait until playback reaches that wall-clock time
↓
reveal score
```

This is synchronization between:

```text
metadata timeline
+
media timeline
```

---

# 76. ABR — Adaptive Bitrate

ABR means:

> The player chooses which rendition to download next.

Example ladder:

```text
360p  → 800 kbps
480p  → 1500 kbps
720p  → 3000 kbps
```

If network becomes bad:

```text
720p
 ↓
480p
 ↓
360p
```

If network improves:

```text
360p
 ↓
480p
 ↓
720p
```

---

# 77. Throughput-Based ABR

Measure:

```text
how quickly recent segments arrived
```

Then choose the highest rendition that fits estimated throughput.

Problem:

```text
throughput estimate can fluctuate
```

which can cause oscillation.

---

# 78. Buffer-Based ABR

Use:

```text
buffer level
```

as the main signal.

Conceptually:

```text
buffer nearly empty
→ lower quality

buffer healthy/full
→ raise quality
```

The session references research showing buffer-based adaptation can reduce rebuffers at similar video rates.

---

# 79. The ABR Policy

The important product rule:

```text
STALL
is worse than
LOWER QUALITY
```

So:

```text
better 480p without interruption
>
1080p followed by a stall
```

This is the same principle as Session 7:

```text
graceful degradation
```

---

# 80. Low-Latency HLS

LL-HLS reduces latency using:

```text
Parts
Blocking reload
Preload hints
Delta updates
```

---

# 81. Parts

Instead of waiting for an entire:

```text
4 s segment
```

the stream can expose smaller:

```text
~1 s parts
```

The player can start consuming them earlier.

---

# 82. Blocking Playlist Reload

Instead of polling:

```text
GET playlist
→ immediate response
→ wait
→ GET again
```

the request can be held until:

```text
new media is available
```

This is essentially:

```text
long polling
```

applied to the HLS playlist.

---

# 83. Preload Hints

The server can tell the client:

```text
the next media part is likely to appear here
```

so the client can prepare for it.

The session notes this replaced the role previously associated with HTTP/2 push in this context.

---

# 84. Delta Updates

A full playlist contains:

```text
old entries
+
new entries
```

A delta update can communicate:

```text
only what changed
```

HLS uses:

```text
EXT-X-SKIP
```

to omit old entries.

This reduces playlist response size.

---

# 85. LL-HLS Latency

The session's illustrative relationship:

```text
normal 4 s segments
→ player hold-back ~12 s
```

With:

```text
~1 s parts
```

hold-back can be around:

```text
~3 s
```

under the cited Apple constraint:

```text
PART-HOLD-BACK ≥ 3 parts
```

---

# 86. Many Origins

Large video systems may use:

```text
Origin 1
Origin 2
...
```

with multiple packagers.

Example:

```text
Android
   ↓
CDN A
   ↓
Origin 1

Android
   ↓
CDN B
   ↓
Origin 2
```

Multiple origins provide:

```text
capacity
redundancy
failover
```

---

# 87. Multi-CDN Architecture

Example:

```text
                    client
                      │
               content steering
                 /         \
                /           \
          CDN A             CDN B
          edge              edge
           │                 │
        mid-tier           mid-tier
           │                 │
        Origin 1          Origin 2
```

This avoids:

```text
single CDN dependency
```

---

# 88. CDN Selection Methods

The session gives several choices.

## DNS

DNS can hand out a CDN based on:

```text
region
ISP
```

Problem:

```text
DNS is cached
```

so switching can be slow.

---

## Master Playlist

List equivalent renditions across CDNs.

The player can have:

```text
CDN A URL
CDN B URL
```

for the same media.

---

## Content Steering

A JSON control file can say:

```text
preferred CDN = A
```

and clients refresh it according to its TTL.

This allows more dynamic control than DNS.

---

## Client Failover

If:

```text
CDN A errors repeatedly
```

the client can:

```text
trip a circuit breaker
↓
use CDN B for the next segment
```

This directly connects Session 7's circuit breaker to video delivery.

---

# 89. Stale Playlists: The Dangerous Case

For live playlists:

```text
stale
```

does not necessarily mean:

```text
safe
```

A stale playlist may point at:

```text
deleted segments
```

which causes:

```text
404
→ player stall
```

Therefore:

```text
stale-if-error
```

must be applied differently to:

```text
playlist
```

and:

```text
immutable segment
```

---

# 90. Segment Deletion

The session's example:

```text
target duration = 4 s
```

and:

```text
hls_delete_threshold = 1
```

means an unreferenced segment may disappear quickly.

The RFC-oriented rule shown in the session says a removed segment should remain available for its duration plus the playlist window in that example.

The important practical lesson:

> **Never serve a stale live playlist without considering whether the segments it names still exist.**

---

# 91. FFmpeg: Simple HLS Command

The session demonstrates:

```bash
ffmpeg -re -i input.mp4 \
  -codec: copy \
  -hls_time 4 \
  -hls_list_size 5 \
  -hls_playlist_type event \
  -f hls index.m3u8
```

---

# 92. Why `-hls_time 4` Is Not a Promise

This is a major trap.

If using:

```text
-codec: copy
```

FFmpeg can only cut at existing keyframes.

So:

```text
-hls_time 4
```

means:

```text
target ≈4 s
```

not:

```text
guaranteed 4 s
```

The session's actual run produced:

```text
8.33 s segments
```

because x264's existing keyframe interval was:

```text
250 frames
```

at:

```text
30 fps
```

giving:

```text
250 / 30
≈8.33 s
```

---

# 93. Keyframes Control Segment Boundaries

If you need:

```text
4-second segments
```

you need keyframes aligned to:

```text
4-second boundaries
```

For:

```text
30 fps
```

that means:

```text
4 × 30 = 120 frames
```

The session's ladder uses:

```text
-g 120
-keyint_min 120
```

for that case.

---

# 94. Why 24 FPS Matters

At:

```text
24 fps
```

a 120-frame GOP gives:

```text
120 / 24 = 5 s
```

So the same:

```text
-g 120
```

does NOT mean:

```text
4 s
```

at every frame rate.

This is a very good MCQ/interview trap.

### Formula

```text
segment duration ≈ GOP frames / FPS
```

---

# 95. The Session's "One Homework" Insight

The session specifically asks you to:

1. Run the ladder on a 24 fps file.
2. Prove from the playlist that segments are 5 s.
3. Switch to time-based keyframes using `force_key_frames`.
4. Prove the segments become 4 s.

The lesson:

> **Frame-based keyframe settings depend on FPS; time-based keyframes express the desired duration directly.**

---

# 96. FFmpeg ABR Ladder

The example uses three renditions:

```text
720p → 3000k video + 128k audio
480p → 1500k video + 96k audio
360p → 800k video + 64k audio
```

Resolutions:

```text
1280×720
854×480
640×360
```

This is a simplified demonstration ladder.

---

# 97. `split=3`

The FFmpeg pipeline:

```text
decode once
↓
split into 3 video branches
↓
encode 3 renditions
```

This avoids decoding the same source three separate times.

---

# 98. HLS Flags

The example uses:

```text
independent_segments
delete_segments
omit_endlist
program_date_time
```

Conceptually:

### `independent_segments`

Renditions are segmented around independently decodable boundaries.

### `delete_segments`

Old segments are eventually removed.

### `omit_endlist`

Live playlist does not have a final end marker.

### `program_date_time`

Associates media with wall-clock time.

---

# 99. The Master Playlist

A master playlist points to:

```text
720p playlist
480p playlist
360p playlist
```

The player chooses a rendition based on:

```text
network
buffer
device
policy
```

This is the entry point for ABR.

---

# 100. Four Ideas to Keep

The session ends with four major principles.

## 1. Pull scales through caches; push scales through machines.

```text
same answer for everyone
→ CDN/cache

different live state per client
→ persistent connection infrastructure
```

---

## 2. Number the pieces, then cache them forever.

Use:

```text
small changing index
+
numbered immutable pieces
```

Examples:

```text
scorecard + numbered updates
HLS playlist + segments
```

---

## 3. Latency is a budget you spend on safety.

Every component adds:

```text
poll interval
+
cache TTL
+
segment duration
+
player hold-back
```

Lower latency generally means:

```text
less safety/buffer
```

---

## 4. Know which bill you are on.

Roughly:

```text
small object
→ request fee matters

large object
→ byte fee matters
```

The session's price assumptions give a break-even around:

```text
11 KB
```

---

# 101. Session 8 and Previous Sessions

## Week 1 — Sockets

Persistent connections:

```text
socket
FD
ephemeral ports
```

show up directly in:

```text
WebSockets
MQTT
held SSE connections
```

---

## Week 2 — Framing

The session reuses:

```text
SSE → blank-line delimiter
WebSocket → length field
MPEG-TS → 188-byte packets
```

---

## Week 3 — Scale

```text
Little's Law
connection cost
load balancer limits
```

become the core of:

```text
50M persistent connections
```

---

## Week 4 — nginx

Appears through:

```text
proxy_read_timeout
proxy_cache_lock
cache behavior
```

---

## Week 5 — HTTP

Appears through:

```text
ETag
304
keep-alive
101 Upgrade
HTTP/2
```

---

## Week 6 — CDNs

Appears through:

```text
cache tiers
request collapsing
negative caching
stale-if-error
multi-CDN
```

---

## Week 7 — Failures

Appears through:

```text
jitter
circuit breaker
live-edge 404
failover
degradation
```

---

# 102. MCQ Traps — Web Protocols

1. `101` means Switching Protocols.
2. WebSocket starts as an HTTP Upgrade.
3. WebSocket is bidirectional.
4. SSE is server → client only.
5. SSE events are delimited by a blank line.
6. WebSocket frames are length-framed.
7. WebSocket payload length 0–125 is encoded directly.
8. WebSocket length 126 means read the next 2 bytes.
9. WebSocket length 127 means read the next 8 bytes.
10. Client-to-server WebSocket frames are masked.
11. `Sec-WebSocket-Accept` uses SHA-1 + fixed GUID + Base64.
12. SSE can resume using `Last-Event-ID`.
13. SSE can specify reconnect timing using `retry:`.
14. HTTP/2 multiplexes multiple streams over one connection.
15. MQTT uses a broker.
16. MQTT QoS 1 is at-least-once, not exactly-once.
17. MQTT retained messages act like a broker-side latest-value cache.
18. MQTT keepalives consume real resources.

---

# 103. MCQ Traps — Scale

1. Persistent connection ≠ free connection.
2. Every persistent connection consumes state.
3. A load balancer also holds connection state.
4. `ulimit -n` limits file descriptors.
5. One source IP has a finite ephemeral port range.
6. Little's Law is `L = λW`.
7. Polling can have low concurrency because request lifetime is short.
8. Long polling has high concurrency because requests are held.
9. Push can hold one socket per viewer for hours.
10. Keepalive traffic exists even when application data is idle.
11. Push scales through machines.
12. Pull scales through caches.

---

# 104. MCQ Traps — Caching

1. 304 does not mean zero network work.
2. 304 still costs a request and round trip.
3. ETag is a representation fingerprint.
4. Poll jitter prevents synchronized request spikes.
5. Request collapsing prevents many simultaneous cache misses from reaching origin.
6. Negative caching caches errors temporarily.
7. Negative TTL must be short.
8. Caching a live 404 can be harmful because the resource may appear soon.
9. Stale playlists can point to deleted segments.
10. Stale immutable segments are generally much safer.
11. Playlist TTL should be much shorter than segment TTL.

---

# 105. MCQ Traps — HLS

1. Playlist = control/index.
2. Segment = media/data.
3. Playlist changes frequently.
4. Segment usually does not change after publication.
5. `#EXT-X-TARGETDURATION` describes target segment duration.
6. `#EXT-X-MEDIA-SEQUENCE` identifies the sequence of the first listed segment.
7. `#EXTINF` gives the duration of the following segment.
8. HLS can use `.ts` segments or fMP4/CMAF.
9. MPEG-TS packets are 188 bytes.
10. MPEG-TS has a 4-byte header + 184-byte payload.
11. MPEG-TS sync byte is `0x47`.
12. PID is 13 bits.
13. `-hls_time` is a target, not a guaranteed duration.
14. With stream copy, existing keyframes constrain segment boundaries.
15. `120 frames / 24 fps = 5 s`.
16. `120 frames / 30 fps = 4 s`.

---

# 106. MCQ Traps — ABR / Latency

1. ABR chooses the next rendition.
2. Throughput-based ABR uses recent delivery rate.
3. Buffer-based ABR uses playback buffer.
4. Lower bitrate can be better than a stall.
5. More buffer generally increases safety but increases latency.
6. Shorter segments can reduce latency but increase request/overhead pressure.
7. LL-HLS uses parts.
8. Blocking reload is essentially long-poll behavior.
9. Delta updates send less playlist history.
10. `PROGRAM-DATE-TIME` connects media to wall-clock time.
11. Score spoilers can be prevented by delaying metadata until playback reaches its timestamp.

---

# 107. MCQ Traps — Cost

1. Tiny objects can be dominated by request fees.
2. Large objects can be dominated by byte/egress fees.
3. The session's illustrative break-even is around 11 KB.
4. 30-second polling for 50M viewers gives ~1.67M requests/s.
5. 15-second polling gives ~3.33M requests/s.
6. 4-hour match = 14,400 seconds.
7. 20M average viewers × 14,400 / 30 ≈ 9.6B requests.
8. At 1 KB each, ≈9.6 TB.
9. 20M idle connections with 60-second keepalives ≈333k keepalives/s.
10. These are scenario calculations, not universal limits.

---

# 108. Interview Questions

## Q1. Why can polling scale better than WebSockets?

Because polling is stateless and cacheable. A CDN can answer many identical requests without maintaining one persistent application connection per viewer. WebSockets require persistent per-client connection state.

---

## Q2. Why would you choose WebSockets instead of polling?

When you genuinely need:

```text
low-latency
bidirectional
interactive
```

communication.

Examples:

```text
multiplayer controls
collaborative editing
trading interfaces
interactive live systems
```

---

## Q3. When is SSE better than WebSocket?

When the communication is primarily:

```text
server → client
```

and you want:

```text
HTTP semantics
simple event framing
browser reconnect behavior
Last-Event-ID
```

without requiring bidirectional messaging.

---

## Q4. Why not use WebSockets for every viewer?

Because:

```text
50M viewers
=
50M persistent connection states
```

which consumes:

```text
memory
FDs
LB state
TLS state
keepalives
machines
```

If the data is identical and cacheable, pull/CDN is often much cheaper operationally.

---

## Q5. Explain Little's Law in this system.

```text
L = λW
```

Polling:

```text
high λ
small W
→ moderate L
```

Push:

```text
connection establishment rate
×
hours-long W
→ huge L
```

The connection lifetime is the critical difference.

---

## Q6. Why does 304 not eliminate CDN request cost?

Because:

```text
304
```

is still a request/response exchange.

It saves:

```text
body bytes
parsing
rendering
```

but not:

```text
request
headers
round trip
request fee
```

---

## Q7. Why is a stale playlist more dangerous than a stale segment?

A stale playlist can point to segments that have already been deleted.

A stale immutable segment is still the same media object.

Therefore:

```text
stale playlist → potentially broken playback
stale segment → usually harmless
```

---

## Q8. Why can a 404 cache break live video?

Because a live segment may return 404 temporarily before it exists.

If the CDN caches that 404:

```text
segment appears
but CDN continues returning cached 404
```

The player can stall.

---

## Q9. Why isn't `-hls_time 4` guaranteed to produce 4-second segments?

Because segment boundaries are constrained by keyframes when stream copying/no re-encoding is used.

If keyframes are 8.33 seconds apart:

```text
-hls_time 4
```

cannot magically cut at a non-keyframe boundary.

---

## Q10. Why does `-g 120` mean 4 seconds at 30 fps but 5 seconds at 24 fps?

Because:

```text
duration = frames / FPS
```

Therefore:

```text
120 / 30 = 4 s
120 / 24 = 5 s
```

---

## Q11. How does ABR prevent stalls?

It can detect:

```text
falling throughput
or
falling buffer
```

and switch to a lower bitrate rendition before the playback buffer reaches zero.

---

## Q12. What is content steering?

A mechanism for dynamically telling clients which CDN/origin path should be preferred.

It can react faster than DNS because the steering information is controlled at the application/content layer and refreshed according to its TTL.

---

## Q13. Why use multiple CDNs?

To reduce:

```text
single-provider dependency
regional failures
capacity bottlenecks
```

and provide:

```text
failover
```

---

## Q14. Why can DNS-based CDN steering be slow?

Because DNS responses are cached by:

```text
resolvers
clients
intermediaries
```

so a change does not instantly reach every viewer.

---

## Q15. Why does MQTT QoS 1 require idempotent updates?

Because:

```text
at-least-once
```

delivery allows duplicates.

If update:

```text
v1842
```

arrives twice, the application should safely recognize that both represent the same logical state.

---

# 109. System Design Scenario

## Design: Live Cricket for 50M Users

Separate the problem.

### Scorecard

```text
versioned JSON
↓
conditional GET
↓
CDN
↓
jittered polling
```

Why?

```text
small
same for everyone
cacheable
stateless
```

### Video

```text
HLS
↓
playlist
↓
CDN
↓
immutable segments
```

### Multiple CDNs

```text
content steering
+
client failover
```

### Failures

```text
circuit breaker
+
fallback CDN
+
stale carefully
```

### Playback

```text
ABR
+
buffer
+
lower rendition before stall
```

---

# 110. The Architecture

```text
                 LIVE EVENT SOURCE
                        │
          ┌─────────────┴─────────────┐
          │                           │
      SCORE DATA                   VIDEO
          │                           │
   versioned JSON                encoder/packager
          │                           │
     CDN / cache                 HLS ladder
          │                           │
   50M clients                 master playlist
                                      │
                             ┌────────┴────────┐
                             │                 │
                           CDN A             CDN B
                             │                 │
                         segments           segments
                             │                 │
                             └────────┬────────┘
                                      │
                                   PLAYER
                                      │
                              ABR + buffer
                                      │
                                  DISPLAY
```

---

# 111. The Four Big Design Choices

## If data is identical for everyone:

```text
CACHE IT
```

## If data is different per client:

```text
PERSISTENT CONNECTION
```

may be justified.

## If objects change frequently:

```text
small index
+
immutable numbered pieces
```

## If quality can vary:

```text
ABR
```

---

# 112. The Cost Decision Tree

```text
Is the response identical for many users?
        │
       YES
        ↓
Can it be cached?
        │
       YES
        ↓
      CDN
        │
       NO
        ↓
Need real-time server push?
        │
     ┌──┴──┐
    NO     YES
    │       │
  Poll    Push
          │
   ┌──────┼────────┐
   │      │        │
  SSE   WebSocket MQTT
   │      │        │
one-way two-way broker fanout
```

---

# 113. The Most Important Synthesis

The entire course can now be reduced to:

```text
SOCKETS
   ↓
how connections work

FRAMING
   ↓
how bytes become messages

SCALE
   ↓
how many connections/requests can exist

NGINX
   ↓
how one origin handles traffic

HTTP
   ↓
how clients communicate efficiently

CDN
   ↓
move identical data closer

FAILURES
   ↓
retry / failover / degrade

VIDEO
   ↓
combine everything
```

---

# 114. 60-Second Revision

```text
SESSION 8 = VIDEO AT LARGE SCALE

50M VIEWERS
↓
same scorecard
↓
PULL THROUGH CACHE

POLLING
GET every N seconds
stateless
cacheable
304 saves body, not request

LONG POLL
GET
↓
server waits
↓
responds when news arrives
held socket

SSE
one HTTP response
never ends
server → client
blank line framing
Last-Event-ID
HTTP/1.1 connection limit issue

WEBSOCKET
HTTP Upgrade
101 Switching Protocols
two-way
framed
client frames masked
126 → 2-byte extended length
127 → 8-byte extended length

MQTT
broker
topics
fan-out
QoS 1 = at least once
retained message = latest-value cache

SCALE
L = λW
polling:
short W → fewer concurrent requests
push:
hours-long W → millions of sockets

HLS
playlist = control
segment = data
playlist changes
segment immutable

MPEG-TS
188 bytes
4 header
184 payload
0x47 sync
13-bit PID

CACHE
playlist short TTL
segment long TTL
beware stale playlist
negative 404 TTL very short

LATENCY
encode
+
segment
+
CDN
+
hold-back

ABR
throughput-based
or
buffer-based

STALL > LOWER QUALITY

MULTI-CDN
DNS
master playlist
content steering
client failover

LL-HLS
parts
blocking reload
preload hints
delta updates

FFMPEG
-hls_time = target
not guarantee
keyframes control boundaries

FORMULA
duration = GOP frames / FPS

120 / 30 = 4s
120 / 24 = 5s

FINAL FOUR
1. Pull scales through caches; push through machines.
2. Number pieces, cache immutable pieces.
3. Latency is a safety budget.
4. Know whether you're paying per request or per byte.
```

---

# 115. The 15 Things I Would Memorize

If you have very little time, memorize these:

1. `L = λW`
2. `101 Switching Protocols` = WebSocket upgrade success.
3. WebSocket = bidirectional framed connection.
4. SSE = one-way server → client.
5. SSE event ends with blank line.
6. WebSocket client frames are masked.
7. `126` = next 2 bytes give length.
8. `127` = next 8 bytes give length.
9. `Sec-WebSocket-Accept = Base64(SHA-1(key + GUID))`.
10. MQTT QoS 1 = at least once.
11. HLS playlist = changing control/index; segments = media.
12. MPEG-TS packet = 188 bytes.
13. `120 / 30 = 4s`, `120 / 24 = 5s`.
14. `304` saves body transfer but not request/round-trip cost.
15. Pull scales through caches; push scales through machines.

---

# 116. Final Mental Model

```text
                 50 MILLION VIEWERS
                         │
             ┌───────────┴───────────┐
             │                       │
         SCORECARD                 VIDEO
             │                       │
      Is it identical?          Encode once
             │                       │
            YES                   HLS ladder
             │                       │
           CACHE                 playlist
             │                       │
            CDN                  segments
             │                       │
         304 / GET                  CDN
             │                       │
       jitter polling        multi-CDN failover
                                     │
                                     ▼
                                   PLAYER
                                     │
                              buffer + ABR
                                     │
                              lower quality
                              before stall
```

And underneath everything:

```text
CACHE
→ reduces origin work

LITTLE'S LAW
→ tells you concurrency

FRAMING
→ tells you how bytes become messages

CDN
→ moves identical data closer

FAILOVER
→ handles dependency failure

ABR
→ handles changing network capacity

BUFFER
→ trades latency for safety
```

> **The core lesson of Session 8:** large-scale live video is not one "video streaming" problem. It is a collection of networking problems—framing, connection state, caching, cost, failure, latency, and adaptation—assembled into one system.

---

# 117. Assignment

The session's laptop assignment combines the earlier CDN/failure work.

Build:

```text
origin: nginx + score.json + FFmpeg ladder
        ↓
mid-tier: nginx cache
        ↓
simulated clients
```

Measure four things:

### 1. Collapsing

Compare origin requests for:

```text
proxy_cache_lock OFF
vs
proxy_cache_lock ON
```

The goal is to show request collapsing.

### 2. Staleness

For each ball:

```text
seen time - written time
```

Compare:

```text
30 s TTL
vs
5 s TTL
```

### 3. Break It

Serve a stale playlist using:

```text
hls_delete_threshold 1
```

Observe:

```text
404s
```

Then fix the configuration and show a clean log.

### 4. Spoiler

Show:

```text
score ahead of video
```

Then delay the score until:

```text
PROGRAM-DATE-TIME
```

passes it.

Deliverable:

```text
code
+
one-page write-up
+
numbers
```

---

# 118. Homework

## 1. Video.js

Use DevTools:

```text
log 2 minutes
↓
throttle network
↓
observe rendition switch
```

Goal:

```text
connect ABR behavior to actual requests
```

---

## 2. Recalculate the Four Bills

Repeat the architecture comparison using:

```text
15 s polling
```

and:

```text
$0.02/GB
+
no request fee
```

Then determine which architecture wins.

The point:

> **Architecture economics depend on the pricing model.**

---

## 3. SSE + HTTP/1.1 vs HTTP/2

Serve the scorecard using SSE.

Open:

```text
7 tabs
```

Observe the HTTP/1.1 connection limitation.

Then switch to:

```text
HTTP/2
```

and observe multiplexing.

---

## 4. Hotstar PubSub

Read the Hotstar architecture and map its five steps to:

```text
Week 3 scaling
+
Week 4 nginx/load-balancing
```

---

## 5. Highest-ROI Homework

Run the HLS ladder on:

```text
24 fps
```

Prove:

```text
segments = 5 s
```

Then use:

```text
force_key_frames
```

to make:

```text
segments = 4 s
```

This is the single homework task the session recommends if you only do one.

---

# 119. Final Exam Cheat Sheet

```text
POLLING
stateless + cacheable
latency ≈ polling interval

304
body saved
request still happens

LONG POLL
held request
held socket
event completes request

SSE
HTTP response stays open
server → client
blank line delimiter
Last-Event-ID

WEBSOCKET
HTTP Upgrade
101
two-way
length-framed
client masked

MQTT
broker
topic
QoS 1 = at least once
retained = latest value

LITTLE'S LAW
L = λW

PUSH
W = hours
→ millions of sockets

PULL
W = milliseconds
→ much smaller concurrency

HLS
playlist = index
segments = media

MPEG-TS
188 bytes
0x47
13-bit PID

CACHE
playlist short TTL
segment long TTL

404
normal static file → can cache
live segment → dangerous

ABR
throughput or buffer
stall > lower quality

LATENCY
encode + segment + CDN + holdback

SPOILER
wall-clock timestamps
PROGRAM-DATE-TIME

MULTI-CDN
DNS
master playlist
content steering
client failover

LL-HLS
parts
blocking reload
preload hints
delta updates

FFMPEG
-hls_time = target
keyframes determine actual boundaries

DURATION
= GOP / FPS

CORE
pull → caches
push → machines
```
