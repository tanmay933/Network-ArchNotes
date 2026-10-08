# Session 6 — (Ab)using CDNs
## High-ROI Notes for MCQs + Interviews

> **Goal:** understand why CDNs exist, what an edge actually does, how caching behaves, how a CDN reduces latency/origin load, and why putting everything at the edge creates both power and risk.
>
> These notes follow the supplied Session 6 material. Numerical examples and product-specific behavior are retained where the session uses them.

---

# 0. Session Map

Session 6 asks six questions:

1. **Why does the edge exist?**  
   Physics: distance creates latency and loss.

2. **What is a CDN?**  
   A reverse proxy/cache deployed close to users, with many PoPs and often anycast.

3. **What can be cached?**  
   Cache keys, TTLs, `cf-cache-status`, revalidation, stale content, and techniques for personalised pages.

4. **How does a CDN move bytes faster?**  
   BBR, origin connection pooling, multiplexing, request collapsing, and tiered caches.

5. **What else does the edge do?**  
   WAF, rate limiting, DDoS absorption, API gateway/authentication.

6. **What happens when you trust the edge?**  
   TLS termination, visibility, concentration, lock-in, and single points of failure.

### Core idea

A CDN is not magic.

It mostly takes ideas from earlier sessions and **distributes them geographically**:

```text
reverse proxy
+ cache
+ connection pooling
+ ETag revalidation
+ BBR
+ multiplexing
+ consistent hashing
+ framing
+ security controls
        ↓
     many cities
```

The supplied PDF explicitly frames the closing idea this way: a CDN invents very little; it distributes techniques already learned in Sessions 2–5.

---

# 1. Why the Edge Exists: Physics

## 1.1 Speed of light is a hard budget

The session gives:

```text
light in vacuum  ≈ 299,792 km/s
light in fibre   ≈ 204,218 km/s
```

The fibre value is roughly two-thirds of vacuum speed because glass has a refractive index around:

```text
1.468
```

The session's derived value:

```text
≈ 4.90 ms per 1000 km
```

one way.

### Important

You cannot optimise this away with:

- a faster CPU
- a faster database
- more application threads

If the bytes physically have to travel a long distance, propagation delay remains.

---

# 2. RTT Is the Real Application Tax

A long path hurts because many protocols need round trips.

The session's example:

```text
Mumbai → Ashburn, Virginia
RTT floor ≈ 126 ms
```

before application processing.

The material illustrates that a long-distance connection can spend hundreds of milliseconds on handshake round trips before useful application data is returned.

### Mental model

```text
distance
   ↓
RTT
   ↓
handshakes / request-response
   ↓
TTFB
```

This is why moving the edge closer to the user matters.

---

# 3. What a CDN Actually Is

A CDN is essentially:

```text
reverse proxy
+
cache
+
TLS termination
+
security
+
connection management
+
geographically distributed PoPs
```

The session describes it as:

> Session 4's reverse proxy, rented in 300+ cities.

Conceptually:

```text
User
  ↓
Nearby CDN PoP
(cache / TLS / WAF)
  ↓
shield / tiered cache
  ↓
Origin
```

The CDN owns the connections on both sides.

That is important.

It can make:

```text
User ↔ CDN
```

short and cheap, while managing:

```text
CDN ↔ Origin
```

as a separate long-distance connection.

---

# 4. Why a CDN Can Be Faster

A CDN has two major advantages.

## 4.1 It saves handshakes

The CDN can keep warm connections to the origin.

Instead of:

```text
user → origin
new connection
new TLS handshake
request
```

you get:

```text
user → nearby edge

edge → already-warm origin connection
```

## 4.2 It can delete the origin round trip entirely

If the object is cached:

```text
user
 ↓
CDN HIT
 ↓
response
```

The origin is never contacted.

### The session's benchmark

For the Mumbai → Washington-style path:

| Configuration | TTFB |
|---|---:|
| Direct, new connection | 621 ms |
| Direct, connection kept alive | 208 ms |
| Edge, no origin pool | 625 ms |
| Edge, warm origin pool | 233 ms |
| Edge, warm pool + kept-alive user connection | 209 ms |
| Cache HIT | 34 ms |
| Cache HIT + kept-alive connection | 11 ms |

### Critical lesson

**A cache miss is not automatically faster just because there is a CDN.**

The session explicitly shows an edge with no origin pool being slightly worse than direct.

The CDN earns its value through:

```text
warm connections
+
cache hits
```

---

# 5. CDN and Anycast

A CDN can announce the same IP from many locations.

Example:

```text
same IP
   ↓
BOM
DEL
MAA
SIN
LHR
IAD
...
```

Routing/BGP determines which PoP receives traffic.

The session demonstrates this with Cloudflare:

```text
colo=BOM
```

or another city code.

### Important distinction

The client does not explicitly choose:

```text
"send me to Mumbai"
```

The network routing system chooses the path to an announced location.

### Anycast mental model

```text
ONE IP
  ↓
many locations
  ↓
routing chooses a nearby/reachable PoP
```

This is how geography becomes part of the service.

---

# 6. CDN Architecture

Typical path:

```text
                         ┌── edge PoP
                         │
User ──→ CDN anycast ────┼── edge PoP ──→ shield ──→ origin
                         │
                         └── edge PoP
```

The edge may provide:

- cache
- TLS
- WAF
- rate limiting
- authentication
- API gateway
- code execution

The shield/tiered cache reduces repeated origin requests.

---

# 7. Consistent Hashing

A distributed cache needs to answer:

> Which cache owns this URL?

A naive strategy:

```text
hash(url) % N
```

looks simple but has a major scaling problem.

## 7.1 Modulo hashing

If:

```text
N = 10
```

and you add an 11th cache:

```text
N = 11
```

the modulo result changes for a huge fraction of keys.

The session benchmark:

```text
100,000 URLs
10 caches → 11 caches

hash(url) % N
≈ 90.9% URLs move
```

That is disastrous.

A capacity upgrade can become:

```text
cache miss explosion
→ origin load spike
→ cache stampede
```

---

# 8. Consistent Hashing

Consistent hashing puts servers on a logical ring.

A key maps to a point on the ring and is owned by a nearby server.

When a server is added:

```text
only the new server's share
```

needs to move.

The session gives:

```text
consistent hash, 1 point/server
≈ 3.3% URLs move

consistent hash, 160 points/server
≈ 8.2% URLs move
```

The virtual points help distribute load more evenly.

### Important mental model

```text
Modulo hashing:
add server
→ almost everything remaps

Consistent hashing:
add server
→ mostly the new server's fair share moves
```

### Why this matters for CDNs

Adding capacity should **not** cause a near-total cache flush.

---

# 9. CDN Players in the Session

The session discusses:

| CDN | Session emphasis |
|---|---|
| Akamai | Ghost edge servers, ESI, FastTCP → BBR |
| Amazon CloudFront | AWS edge, BBR to viewers |
| Fastly | Varnish + H2O, instant purge |
| Google Media CDN | QUIC / BBR |
| Broadpeak | Operator/telco video CDN |
| Cloudflare | nginx history → Oxy/Pingora architecture |

The broader lesson:

**The software ideas from Sessions 3–5 are the building blocks inside CDN infrastructure.**

---

# 10. Cache Keys

A cache must decide whether two requests represent the same object.

The session's default key:

```text
scheme + host + path + query
```

For example:

```text
https://example.com/app.css?v=2
```

The key includes:

```text
scheme
host
path
query
```

but by default may not include:

```text
cookies
many headers
```

### The danger

Suppose:

```text
/account
```

returns personalised HTML based on:

```text
Cookie: user=Alice
```

but the cookie is not part of the cache key.

Then:

```text
Alice's response
```

could become the shared cached response for:

```text
Bob
```

That is not merely a performance bug.

It is a security bug.

---

# 11. Cache Eligibility

The session's Cloudflare example says the default configuration caches many static extensions:

```text
css
js
png
jpg
webp
woff
mp4
pdf
zip
...
```

while HTML and JSON are not normally eligible by the same default extension rule.

Responses containing things such as:

```text
Set-Cookie
Cache-Control: private
Cache-Control: no-store
Cache-Control: no-cache
Cache-Control: max-age=0
```

are also treated as non-cacheable/bypass cases in the session's example.

### Session default TTL examples

```text
200 / 301 → 2 hours
404       → 3 minutes
others    → not cached
```

These are **session-specific Cloudflare defaults**, not universal HTTP rules.

---

# 12. Cache HIT vs MISS

The simplest lifecycle:

```text
First request:
User → CDN → MISS → Origin → CDN → User

Later request:
User → CDN → HIT → User
```

The session's demo:

```text
x-cache: MISS
```

means the first request paid the origin round trip.

Then:

```text
x-cache: HIT
```

means the CDN served the cached object without contacting the origin.

---

# 13. `cf-cache-status`

Know these eight states.

| Status | Meaning | Origin contacted? |
|---|---|---|
| `MISS` | Eligible but not cached | Yes |
| `HIT` | Fresh cached object served | No |
| `EXPIRED` | Cached object expired and was fetched again | Yes |
| `REVALIDATED` | TTL passed; origin validated existing object with 304 | Cheap origin check |
| `UPDATING` | Stale object served while refresh happens in background | Async refresh |
| `STALE` | Stale object served because origin failed | Origin failed |
| `DYNAMIC` | Not eligible for caching | Yes |
| `BYPASS` | Eligible but origin explicitly prevented caching | Yes |

### High-value distinctions

```text
MISS
```

means:

> could be cached, but there was no cached copy.

```text
DYNAMIC
```

means:

> it was not eligible for the cache.

```text
BYPASS
```

means:

> caching was bypassed because the origin said not to cache.

```text
REVALIDATED
```

connects to Session 5:

```text
ETag
If-None-Match
304 Not Modified
```

```text
STALE
```

connects forward to Session 7:

```text
serve something old because the origin is failing
```

---

# 14. Cache Revalidation

Instead of downloading an object again:

```text
CDN → origin
If-None-Match: "abc"
```

Origin:

```text
304 Not Modified
```

The CDN can continue using its stored body.

### Why this matters

You save:

```text
body transfer
```

even though the CDN still has to contact the origin.

This is different from:

```text
HIT
```

where the origin is not contacted at all.

---

# 15. Making Personalised Pages Cacheable

A personalised page seems uncacheable:

```text
full page = shared content + personal content
```

The session gives four techniques.

## 15.1 SSI — Server-Side Includes

The server stores/cache-shares a page shell and fills holes before sending it.

```text
server
  ↓
shared shell
+
personal fragments
```

## 15.2 ESI — Edge-Side Includes

Move the assembly to the CDN edge.

```text
origin:
shared shell
+ holes

edge:
fills holes per user
```

This means the shared shell can be cached at the edge.

## 15.3 Ajax

Push personalisation all the way to the browser.

```text
CDN caches shared page
       ↓
browser fetches personal data separately
```

## 15.4 Railgun-style delta

Instead of caching the whole dynamic page:

```text
previous version
+
only changed bytes
```

The session describes Cloudflare Railgun as a historical system and then connects the idea to newer compression-dictionary transport.

---

# 16. SSI vs ESI vs Ajax

Remember the direction in which personalisation moves:

```text
SSI
server

ESI
edge

Ajax
browser
```

The shared content can therefore travel once while only personal pieces are recomputed closer to the user.

### Core principle

**Move personalisation outward so the large shared part becomes cacheable.**

---

# 17. ESI Trade-Off

The session's benchmark:

```text
100 logged-in users
same page
39,676 bytes each
```

### Origin assembles everything

```text
origin requests:       100
origin → edge bytes:   3,984,200
user bytes:            3,992,400
```

### ESI

```text
origin requests:       201
origin → edge bytes:   83,454
user bytes:            3,994,793
```

### Key conclusion

ESI reduced origin → edge bytes by roughly:

```text
98%
```

but:

```text
user download size did NOT shrink
origin request count actually increased
```

Why?

Because personal fragments become separate requests.

### Interview answer

> ESI moves page assembly from the origin to the edge. It reduces the amount of shared page data fetched from the origin, but it does not inherently reduce the bytes sent to users, and it can increase the number of fragment requests.

---

# 18. Railgun and Compression Dictionaries

The session compares historical Railgun with modern compression-dictionary transport.

Example:

```text
~24 KB news page
```

Compression results:

```text
gzip-6                  4,276 B
deflate + recent dict     598 B
zstd + recent dict        436 B
zstd + 1-hour-old dict  1,277 B
```

The key idea:

```text
receiver already has old version
        ↓
use old version as dictionary
        ↓
send only what changed efficiently
```

The dictionary must be sufficiently similar/recent and shared correctly.

---

# 19. Cache Poisoning

A cache is shared state.

That makes it an attack surface.

Example:

```text
attacker sends malicious header
        ↓
origin reflects header into response
        ↓
header is NOT included in cache key
        ↓
malicious response gets cached
        ↓
other users receive it
```

The session's example uses:

```text
X-Forwarded-Host
```

### Fixes from the session

1. Drop the untrusted header.
2. Include it in the cache key.
3. Use `Vary`.

### Core rule

> If a request attribute can change the response, the cache must account for that attribute somehow.

---

# 20. Cache Deception

Different bug, same root problem.

Example:

```text
/account/settings/fake.css
```

The attacker tricks the cache into thinking:

```text
".css" → static → cacheable
```

but the response is actually:

```text
private account data
```

The cache then stores sensitive content.

### Fix

Do not decide cacheability using extension alone.

Respect:

```text
Content-Type
Cache-Control
origin intent
```

### Poisoning vs deception

```text
Cache poisoning:
attacker puts a malicious response INTO the shared cache.

Cache deception:
attacker tricks the cache INTO storing a private response.
```

---

# 21. BBR vs Loss-Based Congestion Control

The session connects CDN performance back to Session 2.

## Loss-based approach

Simplified CUBIC/Reno mental model:

```text
increase sending rate
        ↓
queue fills
        ↓
packet loss
        ↓
reduce rate
        ↓
repeat
```

This creates the classic:

```text
TCP sawtooth
```

### Problem on long CDN paths

Loss may result from:

```text
wireless errors
long paths
other causes
```

Loss is therefore not always an accurate indication of congestion.

---

# 22. BBR

BBR means:

```text
Bottleneck Bandwidth and Round-trip propagation time
```

It estimates:

```text
bottleneck bandwidth
+
minimum RTT
```

and paces traffic toward that model.

### Key idea

```text
loss ≠ necessarily congestion
```

The session describes BBR as measuring the path rather than simply filling the pipe until loss and then halving.

### Why CDN operators can use it

The CDN controls the sender on the long:

```text
CDN → user
```

or:

```text
CDN → origin
```

leg.

No client or router replacement is required for the sender-side algorithm change.

---

# 23. Connection Pooling

Every new origin connection can require:

```text
TCP setup
+
TLS setup
```

So a CDN should reuse connections.

The session benchmark:

| Edge configuration | New connections | Reuse | Mean TTFB |
|---|---:|---:|---:|
| No pool | 300 | 0% | 160 ms |
| 32 worker pools | 164 | 45.3% | 112 ms |
| 8 worker pools | 52 | 82.7% | 74 ms |
| 1 shared pool | 4 | 98.7% | 57 ms |

### Key idea

```text
Amortise the setup cost.
```

One connection can serve many requests.

---

# 24. Why a Shared Pool Can Beat Per-Worker Pools

Suppose each worker owns a separate connection pool.

Traffic is split:

```text
worker A → small amount of traffic
worker B → small amount
worker C → small amount
...
```

Each worker may not receive enough requests to keep its connections warm.

A shared pool lets:

```text
all workers
   ↓
same pool
   ↓
same warm connections
```

The session connects this idea to Cloudflare's Pingora architecture.

---

# 25. Multiplexing at the CDN

HTTP/2:

```text
many streams
      ↓
one connection
```

A CDN can apply the same principle on the origin side.

Conceptually:

```text
many users
   ↓
many edge streams
   ↓
few origin connections
   ↓
origin
```

### Important session caveat

The session notes that many CDN → origin hops still use HTTP/1.1 by default in the referenced setup.

Therefore:

**Pooling is often the immediately useful optimisation; multiplexing depends on the protocol/configuration actually enabled.**

---

# 26. Request Collapsing

Suppose the cache is cold.

```text
50 users request the same object
```

Without collapsing:

```text
50 cache misses
→ 50 origin requests
```

With collapsing:

```text
1 origin request
→ 49 users wait
→ result cached
→ everyone receives HIT
```

### Mental model

```text
Never fetch the same object twice at the same time.
```

This is essentially single-flight behavior.

---

# 27. Tiered Cache / Origin Shield

Request collapsing works at one PoP.

Tiered caching extends the idea across PoPs.

Without tiering:

```text
300 edges
→ 300 MISSes
→ 300 origin requests
```

With a regional parent/shield:

```text
300 edges
     ↓
regional shield
     ↓
1 origin fetch
```

### Two scales of the same idea

```text
request collapsing
= one PoP

tiered cache / origin shield
= many PoPs
```

Both prevent a thundering herd from reaching the origin.

---

# 28. CDN Security Perimeter

A CDN started as:

```text
cache
```

but eventually became:

```text
security perimeter
```

because traffic already converges there.

The session covers:

```text
WAF
rate limiting
DDoS protection
API gateway
authentication
header manipulation
edge code
```

---

# 29. WAF

WAF:

```text
Web Application Firewall
```

The session's key warning:

> A WAF is a denylist of spellings. The bug still lives at the origin.

A WAF may detect common patterns such as:

```text
SQL injection
XSS
path traversal
```

but attackers can transform the same malicious payload.

Example:

```text
' OR 1=1 --
```

might be blocked.

But:

```text
'/**/OR/**/1=1--
```

can bypass a naive pattern.

Normalisation can improve detection:

```text
raw
→ URL decode
→ strip comments
→ normalise
→ inspect
```

But the arms race never ends.

### Correct security model

WAF:

```text
defence layer
```

not:

```text
actual SQL injection fix
```

The actual origin fix is:

```text
parameterised queries / bound parameters
```

---

# 30. Rate Limiting

A statement like:

```text
100 requests / 10 seconds
```

is incomplete.

The algorithm matters.

The session compares:

| Algorithm | Burst at window edge | Normal page load |
|---|---:|---|
| Fixed window | 200 in 1 sec | passes |
| Sliding log | 100 | passes |
| Sliding window estimate | 104 | passes |
| Token bucket | 109 | passes |
| Leaky bucket, burst=0 | 10 | 97% refused |

---

# 31. Fixed Window Problem

Suppose:

```text
limit = 100 / 10 seconds
```

A client sends:

```text
100 requests at 9.999 s
+
100 requests at 10.001 s
```

That is:

```text
200 requests
```

within roughly:

```text
2 ms
```

even though each fixed window individually respected the limit.

### Mental model

```text
fixed windows have boundary bursts
```

---

# 32. Sliding Window

A sliding-window approach looks at the current and previous window with weighting.

The session says Cloudflare's 2017 approach used an estimated sliding window:

```text
two counters
+
weighted previous window
```

The session reports:

```text
400M requests
≈ 0.003% wrongly allowed or blocked
```

This is an example of a cheap approximation being good enough at large scale.

---

# 33. Token Bucket

Token bucket allows controlled bursts.

Mental model:

```text
tokens accumulate
        ↓
request spends token
        ↓
empty bucket
        ↓
request must wait/reject
```

This is why token bucket can handle normal browser bursts better than a strict leaky bucket with zero burst.

---

# 34. Geography Changes Rate Limits

The session highlights a subtle CDN issue:

```text
counts are per data center
```

not necessarily one global counter.

A botnet spread across:

```text
330 cities
```

could potentially receive:

```text
330 local budgets
```

if the rate limiter is purely per-PoP.

### Therefore

Rate limiting has two dimensions:

```text
algorithm
+
geography
```

---

# 35. DDoS Protection

A volumetric DDoS attack is fundamentally a capacity problem.

The session cites:

```text
31.4 Tbps
```

for its largest-record example.

The point is not the exact record.

The important architecture is:

```text
attacker
   ↓
many CDN locations
   ↓
traffic distributed by anycast
   ↓
huge aggregate capacity
   ↓
origin protected
```

Your origin has:

```text
one/few pipes
```

A large CDN has:

```text
hundreds of locations
+
massive aggregate capacity
```

### Why a CDN is uniquely useful here

You cannot practically buy enough local inbound capacity to absorb a giant volumetric flood yourself.

You rent the CDN's distributed capacity.

---

# 36. DDoS and Concentration Risk

The same centralisation that helps absorb attacks creates another problem.

If:

```text
many websites
   ↓
same CDN
```

then:

```text
CDN outage
   ↓
many websites affected
```

The session references the 2021 Fastly incident as the example of concentration risk.

### Principle

```text
distributed protection
can create
centralised dependency
```

---

# 37. API Gateway at the Edge

An API gateway can perform:

```text
authenticate
scrub
route
add headers
remove headers
rate-limit
fail over
```

before the request reaches the origin.

Example flow:

```text
Client
 ↓
CDN edge
 ↓
verify JWT
 ↓
set trusted X-User
 ↓
origin
```

---

# 38. JWT at the Edge

The session's gateway demo:

```text
no token
→ 401

valid token
→ request allowed

expired JWT
→ 401

alg:none forgery
→ 401

client sends X-User: admin
→ ignored / overwritten
```

### Important security rule

The client must not be allowed to choose the identity header.

Bad:

```text
Client → X-User: admin → Origin
```

Good:

```text
Client → JWT
         ↓
      Edge verifies
         ↓
      Edge derives identity
         ↓
      Edge sets X-User
         ↓
      Origin
```

---

# 39. The Origin Must Not Be Directly Reachable

This is one of the most important security points in the session.

Suppose:

```text
Client
 ↓
CDN
 ↓
origin
```

and the origin is also directly reachable:

```text
attacker ─────────→ origin
```

Then the attacker can bypass:

```text
WAF
rate limiting
JWT verification
DDoS filtering
```

### Therefore

Edge security is only real if the origin trusts traffic from the edge and cannot simply be reached by attackers.

The session suggests:

```text
mutual TLS
```

or:

```text
tunnel
```

so the origin only accepts authenticated edge connections.

---

# 40. Header Hygiene

The edge can also remove information that should not leak.

Example from the session:

```text
X-Powered-By: PHP/5.4.45
```

The edge can strip such headers before returning the response.

### Why?

Reduce:

```text
information leakage
```

and unnecessary implementation exposure.

---

# 41. Cloudflare Workers

A CDN edge can execute application logic.

The session describes Workers as:

```text
V8 isolate
```

rather than:

```text
new OS process
```

or:

```text
new VM
```

The session's comparison:

```text
Worker startup ≈ 5 ms
≈ 3 MB
```

versus the larger process/container startup model discussed earlier.

### Core idea

The runtime is already resident at the PoP.

Your code runs inside it.

```text
request
 ↓
existing edge runtime
 ↓
your code
```

rather than:

```text
request
 ↓
fork process
 ↓
boot environment
 ↓
run code
```

---

# 42. Worker Example

The session's gateway Worker conceptually does:

```text
request
 ↓
rate limit
 ↓
verify JWT
 ↓
set trusted identity header
 ↓
fetch origin
```

It can also implement:

```text
failover
ESI assembly
custom routing
authentication
```

This makes the CDN an execution layer, not merely a cache.

---

# 43. The Four Closing Ideas

The session's final slide compresses the entire lesson into four ideas.

## 43.1 Move the work, or move the data

You cannot beat propagation speed.

So either:

```text
move data closer
→ cache
```

or:

```text
move personalisation outward
→ SSI
→ ESI
→ Ajax
```

### Every CDN optimisation is largely one of these two moves.

---

## 43.2 The edge is where traffic converges

Once everything passes through the edge:

```text
cache
↓
WAF
↓
rate limiter
↓
DDoS protection
↓
auth
↓
code execution
```

all follow the traffic.

Speed gets you to use a CDN.

Control makes it part of your architecture.

---

## 43.3 It is Sessions 2–5, rented by the city

Examples:

```text
Reverse proxy       → Session 4
Connection pool     → Session 3
ETag revalidation   → Session 5
BBR                  → Session 2
Multiplexing         → Session 5
Consistent hashing  → distributed systems idea
Framing              → Session 2
```

A CDN packages these techniques and distributes them geographically.

---

## 43.4 Concentration cuts both ways

A CDN gives you:

```text
massive distributed capacity
```

but creates:

```text
dependency
```

One configuration failure can affect many customers simultaneously.

---

# 44. High-ROI Numbers

Memorise the conceptual numbers used in the session.

| Fact | Session value |
|---|---:|
| Light in vacuum | 299,792 km/s |
| Light in fibre | 204,218 km/s |
| Fibre one-way per 1000 km | ~4.90 ms |
| Mumbai ↔ Ashburn example RTT floor | ~126 ms |
| TLS 1.3 cold TCP + HTTP/2 | 3 RTT |
| QUIC + HTTP/3 cold 1-RTT example | ~2 RTT |
| Typical cache TTL for 200/301 in session example | 2 h |
| Typical 404 TTL in session example | 3 min |
| Modulo hashing movement: 10 → 11 caches | ~90.9% |
| Consistent hashing, 1 point/server | ~3.3% |
| Consistent hashing, 160 points/server | ~8.2% |
| ESI origin-byte reduction in benchmark | ~98% |
| Fixed-window example | 100 / 10 s |
| DDoS example | 31.4 Tbps |
| Worker startup example | ~5 ms |
| Worker memory example | ~3 MB |
| Cloudflare PoPs in demo framing | 330+ |
| Rate-limit false allow/block reported in session benchmark | ~0.003% |

### Important

Some numbers are:

```text
benchmarks
```

or:

```text
session-specific provider defaults
```

not universal protocol constants.

---

# 45. High-ROI MCQ Traps

1. A CDN is **not just a cache**.
2. A CDN can also be a reverse proxy, TLS endpoint, WAF, rate limiter, API gateway and code-execution layer.
3. A CDN does **not automatically make cache misses faster**.
4. A cache HIT can remove the origin round trip completely.
5. Warm origin connection pooling saves handshake cost even on cache misses.
6. Anycast uses one announced IP across many locations.
7. The client does not manually select the PoP in normal anycast routing.
8. `hash(url) % N` causes massive key movement when N changes.
9. Consistent hashing limits movement when servers are added.
10. Cache key ≠ necessarily all request headers/cookies.
11. If response changes based on an attribute not represented in the cache key, cache correctness/security can break.
12. `MISS` ≠ `DYNAMIC`.
13. `MISS` means eligible but absent.
14. `DYNAMIC` means not eligible.
15. `BYPASS` means caching was bypassed by origin/cache-control behavior.
16. `REVALIDATED` usually means a cached object was checked with conditional validation such as 304.
17. `STALE` means stale content may be served because the origin is unavailable.
18. ESI reduces origin work; it does not necessarily reduce bytes sent to the user.
19. ESI can increase the number of origin requests because fragments are fetched separately.
20. Cache poisoning and cache deception are different attacks.
21. WAF is a defence layer, not a replacement for parameterised queries.
22. Fixed-window rate limiting has boundary burst problems.
23. Token bucket permits controlled bursts.
24. Rate limits can be local to each PoP.
25. DDoS protection works partly by distributing attack traffic across many locations.
26. A CDN can become a single point of failure/dependency.
27. Edge JWT verification is useless if the origin remains directly reachable.
28. The origin should trust only authenticated edge traffic when edge security is relied upon.
29. A Worker executes code at the edge; it is not simply another origin server.
30. BBR does not treat packet loss as the sole congestion signal.
31. Connection pooling and multiplexing are different:
    - pooling = reuse connections
    - multiplexing = multiple logical streams over one connection
32. Request collapsing prevents multiple simultaneous fetches of the same missing object.
33. Tiered caching extends that idea across multiple PoPs.
34. `SSI → ESI → Ajax` means personalisation moves:
    - server
    - edge
    - browser
35. Railgun/dictionary compression uses previously known content to reduce what must be transferred.
36. The CDN edge can see plaintext if it terminates TLS for the application.
37. Putting security at the edge increases the trust placed in the CDN provider.

---

# 46. Interview Questions

## Q1. Why do CDNs improve latency?

**Answer:**

> They move content and request processing closer to users, reducing propagation distance. They can also terminate client connections nearby, reuse warm origin connections, and serve cache hits without contacting the origin.

---

## Q2. Is a CDN cache hit the only reason a CDN is faster?

**Answer:**

> No. Even on cache misses, the CDN can save connection setup by maintaining warm origin connections. But a cache miss can still be as slow as or slower than direct access if the edge provides no useful optimisation.

---

## Q3. Why is `hash(key) % N` bad for a distributed cache?

**Answer:**

> Changing N changes the modulo result for most keys. Adding one cache can therefore remap most objects, producing a near-total cache miss and potentially a cache stampede against the origin.

---

## Q4. How does consistent hashing solve that?

**Answer:**

> It maps servers and keys onto a logical ring so adding a server mostly moves only the new server's share of keys. Virtual nodes/points improve load distribution.

---

## Q5. What is a cache key?

**Answer:**

> It is the identity the cache uses to decide whether two requests should receive the same stored response.

---

## Q6. Why can an incomplete cache key become a security vulnerability?

**Answer:**

> If a response varies according to something omitted from the key, two logically different responses can collide. For example, a personalised response based on a cookie could be served to another user.

---

## Q7. Cache poisoning vs cache deception?

**Answer:**

> Poisoning stores an attacker-controlled response in the shared cache. Deception causes the cache to store a private/sensitive response as if it were a cacheable resource.

---

## Q8. Why does ESI help?

**Answer:**

> It lets the CDN cache the shared page shell and assemble user-specific fragments at the edge, reducing repeated origin rendering and transfer of the shared shell.

---

## Q9. Does ESI reduce user bandwidth?

**Answer:**

> Not necessarily. The session's benchmark shows nearly the same bytes delivered to users. The main saving was origin-to-edge traffic and origin-side page assembly.

---

## Q10. Why use connection pooling?

**Answer:**

> TCP/TLS setup has latency. Reusing an existing origin connection amortises that setup cost over many requests.

---

## Q11. Why is a shared connection pool better than one pool per worker?

**Answer:**

> A per-worker pool fragments traffic, so each worker may not see enough requests to keep its connections warm. A shared pool lets all workers reuse the same warm connections.

---

## Q12. What is request collapsing?

**Answer:**

> When multiple clients request the same missing object simultaneously, the CDN allows one request to fetch it from the origin while the others wait for that result instead of generating many identical origin requests.

---

## Q13. What is a tiered cache?

**Answer:**

> It places a parent/shield cache between edge PoPs and the origin. Multiple edge misses can therefore collapse into one fetch from the origin.

---

## Q14. Why does BBR help on long paths?

**Answer:**

> BBR estimates bottleneck bandwidth and propagation RTT instead of relying primarily on packet loss as the congestion signal. This can avoid unnecessarily reducing throughput when loss is not actually caused by congestion.

---

## Q15. Why isn't a WAF enough to prevent SQL injection?

**Answer:**

> A WAF detects patterns and attackers can vary their representation. The actual vulnerability is unsafe SQL construction at the origin. Parameterised queries are the real fix.

---

## Q16. Why is fixed-window rate limiting weak?

**Answer:**

> Requests can cluster at the boundary of two windows. A 100/10-second limit can permit 200 requests in a very short interval if 100 arrive at the end of one window and another 100 at the beginning of the next.

---

## Q17. Why does a CDN help against DDoS?

**Answer:**

> A CDN has distributed capacity across many locations. Anycast spreads traffic across that network, allowing the CDN to absorb volumetric traffic that would overwhelm a single origin link.

---

## Q18. Why must the origin be protected from direct access?

**Answer:**

> Otherwise attackers can bypass the CDN's WAF, rate limiter, authentication and DDoS protection by contacting the origin directly.

---

## Q19. What does a Worker add to a CDN?

**Answer:**

> It allows application logic to execute at the edge, such as authentication, rate limiting, routing, failover, or response assembly, without sending every request to the origin.

---

## Q20. What is the main architectural downside of putting everything behind one CDN?

**Answer:**

> Concentration. You gain huge distributed capacity and centralised controls, but a provider outage, configuration error, or trust failure can affect many dependent applications simultaneously.

---

# 47. CDN Request Walkthrough

For a cacheable static asset:

```text
Client
  ↓
Anycast IP
  ↓
Nearest/reachable CDN PoP
  ↓
Cache lookup
  ↓
HIT
  ↓
Response
```

No origin request.

---

For a cold cache:

```text
Client
  ↓
CDN PoP
  ↓
MISS
  ↓
Shield / tiered cache
  ↓
Origin
  ↓
Cache object
  ↓
Return response
```

---

For personalised content:

```text
Client
  ↓
CDN
  ↓
cache decision
  ↓
dynamic / ESI / Worker
  ↓
personal fragments
  ↓
response
```

---

For authenticated APIs:

```text
Client
  ↓
CDN
  ↓
rate limit
  ↓
JWT verification
  ↓
trusted identity header
  ↓
origin
```

---

# 48. What the Session's Lab Actually Models

The supplied session includes a `cdn-lab`.

The repo structure described in the PDF includes:

```text
origin/origin.py
edge/edge.py
tools/laggy.py
cloudflare/
```

The edge implementation combines:

```text
cache
pool
WAF
rate limit
JWT
ESI
```

The session is explicit about limitations:

- `laggy.py` adds delay but does not model packet loss/bandwidth.
- BBR vs CUBIC therefore needs real Linux `tc netem` work.
- All lab hops use HTTP/1.1.
- Anycast/BGP cannot be meaningfully simulated on one machine.
- The worker-pool benchmark is a simulation rather than a real kernel accept queue.
- Some numbers are models rather than physical measurements.

### Important study habit

Do not memorise a benchmark as a universal law.

Know:

```text
what was measured
+
what was simulated
+
what was assumed
```

---

# 49. Session Assignment

The assignment is:

> Put something real behind the edge and prove what the CDN did.

Build:

```text
small origin
+
Cloudflare edge
```

Serve:

```text
1 cacheable static asset
+
1 personalised HTML page
```

Then demonstrate:

### Cache

```text
static asset:
MISS → HIT

personalised page:
DYNAMIC
```

Then make the page cacheable and measure the change in:

```text
origin request count
```

### Cache-key security

Demonstrate:

```text
header affects response
but is ignored by cache key
```

Then fix it.

### Performance

Measure:

```text
cold origin
vs
warm edge
vs
cache HIT
```

using TTFB.

### One edge feature

Add exactly one:

```text
WAF
OR
rate limit
OR
Worker token verification
```

Show:

```text
one blocked request
+
one allowed request
```

Deliverable:

```text
code
+
one-page write-up
+
numbers
+
cache-key explanation
```

---

# 50. Homework / Practical Work

The session gives five homework directions.

## 1. Run the lab

```bash
./setup.sh
./run_all.sh
```

Then read:

```text
edge/edge.py
```

Change one flag such as:

```text
--pool 0
--waf off
--strip-unkeyed
```

and explain which measured number changes and why.

---

## 2. Reproduce BBR vs CUBIC for real

Use a Linux system with:

```text
tc netem
```

to introduce approximately:

```text
1% loss
150 ms RTT
```

Then compare:

```text
CUBIC
vs
BBR
```

on a large download.

---

## 3. Reproduce cache poisoning

On a site you control:

```text
poison cache
→ fix it
```

Try the three fixes from the session:

```text
drop the header
add header to key
Vary on header
```

Then reason about the best default.

---

## 4. Inspect real `cf-cache-status`

Run:

```bash
curl -I <site>
```

on multiple sites.

For each, identify:

```text
cache status
Cache-Control
likely cache key
```

Find a HIT that looks suspicious and explain why.

---

## 5. Measure the speed-of-light floor

For five user locations:

```text
compute theoretical RTT floor
+
measure real ping
```

Then ask:

> How much of the gap could a CDN actually remove?

This is the highest-ROI homework if you only do one.

---

# 51. Final Mental Map

Remember Session 6 as this chain:

```text
DISTANCE
   ↓
RTT
   ↓
EDGE
   ↓
CACHE / POOL / BBR
   ↓
LESS ORIGIN WORK
   ↓
SECURITY + CONTROL
   ↓
MORE TRUST IN THE CDN
```

And the four deepest ideas:

```text
1. Move the work, or move the data.

2. The edge is where traffic converges.

3. A CDN is mostly Sessions 2–5 distributed geographically.

4. Concentration cuts both ways.
```

---

# 52. 60-Second Revision

```text
CDN
= geographically distributed reverse proxy/cache

WHY?
distance → propagation delay → RTT

EDGE BENEFITS
nearby TLS/connection
warm origin pools
cache HITs
security
edge compute

ANYCAST
one IP → many PoPs
BGP/routing chooses path

CONSISTENT HASHING
hash(key)%N → massive remapping
consistent hash → only a fair share moves

CACHE KEY
scheme + host + path + query in session default
danger if response varies on omitted cookies/headers

CACHE STATUS
MISS       → eligible, not cached
HIT        → cached
EXPIRED    → TTL passed
REVALIDATED→ conditional validation
UPDATING   → stale served + background refresh
STALE      → stale served because origin failed
DYNAMIC    → not cache eligible
BYPASS     → origin says don't cache

PERSONALISATION
SSI → server
ESI → edge
Ajax → browser
Railgun/dictionaries → send changes efficiently

BBR
measure bandwidth + min RTT
loss ≠ automatically congestion

POOLING
reuse origin connections

MULTIPLEXING
many streams / one connection

COLLAPSING
one origin fetch / many simultaneous users

TIERED CACHE
many PoPs / one parent fetch / origin

WAF
useful defence layer
not the real SQLi fix

RATE LIMITING
fixed window has boundary bursts
token bucket allows controlled bursts

DDoS
distributed capacity + anycast

API GATEWAY
auth + scrub + route at edge

JWT
edge verifies → edge sets trusted identity

ORIGIN
must not be directly reachable if edge security is trusted

WORKERS
execute code at the edge

RISK
TLS keys + traffic visibility
provider dependency
single point of failure
lock-in
```

---

# 53. The Most Important Connections to Earlier Sessions

| Session 6 topic | Earlier idea |
|---|---|
| CDN reverse proxy | Session 4 nginx |
| Origin pooling | Session 3 keepalive/amortisation |
| BBR | Session 2 congestion |
| Anycast | Session 2 addressing |
| Cache revalidation | Session 5 ETag / 304 |
| Multiplexing | Session 5 HTTP/2 |
| Framing | Session 2 |
| Consistent hashing | distributed ownership |
| ESI | server-side/edge composition |
| WAF | security at the traffic boundary |
| Workers | server-side execution moved to the edge |

### Final takeaway

The CDN is best understood as:

```text
old networking + old caching + old proxying
             +
geographic distribution
             +
centralised control
```

That combination is what turns a simple reverse proxy into infrastructure capable of serving huge numbers of users while simultaneously becoming the application's security and trust boundary.
