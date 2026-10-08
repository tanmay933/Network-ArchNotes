# Session 7 — Building for Failures
## High-ROI Notes for MCQs + Interviews

> **Goal:** understand failures as stages of a request, decide whether a failure should be retried, retry safely when it should be, and design systems that survive dependency failure instead of making the failure worse.
>
> **Core discipline:** classify first. Retry second.
>
> Source: CN & Scaler Session 7 — *Building for Failures*. The session's agenda is recap/taxonomy → retry definition → retry decision → retry mechanics → video-player case study → system-level survival. fileciteturn10file0L120-L149

---

# 0. The Entire Session in One Picture

A request is not just:

```text
request → response
```

It is a sequence:

```text
1. DNS lookup
       ↓
2. TCP handshake
       ↓
3. TLS handshake
       ↓
4. Request write
       ↓
5. Server processing
       ↓
6. Response read
```

The session's key point is:

> Nothing fails "generically". It fails at a specific stage, and the stage changes the correct recovery action. fileciteturn10file0L227-L250

Then the decision pipeline is:

```text
FAILURE
   ↓
Where did it fail?
   ↓
Structural or transient?
   ↓
Is the operation idempotent?
   ↓
If retryable:
    backoff + jitter + cap
   ↓
If dependency repeatedly fails:
    circuit breaker
   ↓
If still unavailable:
    fail over / degrade / serve stale
```

This is the highest-value mental model of Session 7.

---

# 1. Session 7 Agenda

The session is built around one discipline:

```text
FAIL ON PURPOSE
```

The six sections are:

1. **Recap + failure taxonomy**
   - where a request actually breaks

2. **Is this even a retry?**
   - exact definition of retry

3. **When to retry**
   - classify status, idempotency, and failure type

4. **How to retry**
   - backoff
   - jitter
   - caps
   - circuit breakers

5. **Video player case study**
   - 96 failure modes condensed into useful categories

6. **Systems that survive**
   - failover
   - graceful degradation
   - stale responses
   - recovery storms / thundering herd

The source lays this out on the agenda slide. fileciteturn10file0L120-L149

---

# 2. Why Failure Handling Is Hard

A naive developer thinks:

```python
while True:
    try:
        return call()
    except:
        continue
```

A production system cannot do this.

Because:

```text
dependency fails
     ↓
client retries immediately
     ↓
more traffic hits dependency
     ↓
dependency gets even less healthy
     ↓
clients retry again
     ↓
outage gets worse
```

And when the service finally recovers:

```text
all clients retry together
        ↓
traffic spike
        ↓
service crashes again
```

So failure handling is not merely:

> "Try again."

It is:

> "Determine whether another attempt has any chance of helping, and if it does, control how the attempt happens."

---

# 3. Assignment Context: HTTP/1.1 Persistent Connection

Before the failure material, Session 7 includes the previous assignment context.

The calculator server must keep **one socket open** and handle multiple requests:

```text
GET /add?a=2&b=3 → 200 5
GET /sub?a=10&b=4 → 200 6
GET /mul?a=6&b=7 → 200 42
GET /div?a=9&b=3 → 200 3
GET /div?a=1&b=0 → 400
GET /add?a=x&b=3 → 400
GET /pow?a=2&b=8 → 404
POST /add → 405
GET /add (no Host) → 400
```

The important part is not arithmetic.

The challenge is keeping the connection alive and correctly determining:

```text
where request #1 ends
where request #2 begins
```

With persistent HTTP/1.1:

```text
EOF
```

can no longer automatically mean:

```text
request finished
```

The server has to consume exactly the bytes belonging to the current request. fileciteturn10file0L12-L44

### Why this matters for Session 7

Failures can happen while:

```text
connecting
writing
processing
reading
```

and the client must know what actually happened before deciding what to do.

---

# 4. Failure Taxonomy: Six Places to Fail

The source gives the six-stage request timeline:

| Stage | What happens |
|---|---|
| 1. DNS lookup | name → IP |
| 2. TCP handshake | SYN / SYN-ACK / ACK |
| 3. TLS handshake | certificate + key exchange |
| 4. Request write | request bytes leave client socket |
| 5. Server processing | server actually handles request |
| 6. Response read | response bytes arrive |

fileciteturn10file0L227-L250

---

# 5. Stage 1 — DNS Lookup Failures

Examples:

```text
DNS mapping unavailable
DNS timeout
network issue
DNS misconfigured
```

Important distinction:

```text
DNS timeout
```

is not the same as:

```text
DNS says the domain has no record
```

A timeout can be transient.

A valid:

```text
NXDOMAIN / no mapping
```

is fundamentally different.

### Mental model

```text
DNS failure
    ↓
Did the name resolve?
    ↓
No
    ↓
You have not reached a server yet.
```

Do not describe every DNS problem as "server failure".

---

# 6. Stage 2 — TCP Connect Failures

Examples:

```text
host unreachable
port not listening
client connection limit exceeded
server connection limit exceeded
TCP timeout
stale DNS IP
```

A TCP connection timeout means:

```text
you did not successfully establish the socket
```

You therefore do not know that the application server actually received your request.

### Common causes

```text
firewall
network blocker
server down
wrong IP
wrong port
overloaded connection accept path
packet loss
```

---

# 7. Stage 3 — TLS Failures

Examples:

```text
invalid certificate
protocol mismatch
cipher mismatch
SSL pinning mismatch
incorrect client/server clock
underlying TCP timeout
```

### Key idea

A TLS failure is not automatically an application-server failure.

For example:

```text
certificate invalid
```

is structural.

Retrying the same request with the same invalid certificate does not fix it.

But:

```text
temporary network interruption during handshake
```

may be transient.

So even "TLS failure" is not enough information.

You need the exact reason.

---

# 8. Stage 4 — Request Write Failures

The client has a connection and is trying to send bytes.

Possible failures:

```text
packet drop
write timeout
physical machine shutdown
network node goes down
rate limit
server closes connection
```

The request may or may not have reached the server.

That uncertainty matters for non-idempotent operations.

Example:

```text
POST /charge
```

Client writes request.

Connection dies.

Client does not know:

```text
Did server receive it?
Did server charge the card?
Did server crash before processing?
```

A blind retry could create a duplicate charge.

---

# 9. Stage 5 — Server Processing Failures

At this stage:

```text
the request reached the server
```

Potential outcomes:

```text
business logic failure
rate limiting
access denied
server-side exception
```

This is where HTTP status codes become especially useful.

A `500` tells you:

```text
server admits failure
```

but not:

```text
whether the operation already happened
```

That second question is why idempotency matters.

---

# 10. Stage 6 — Response Read Failures

The client has already sent the request and is waiting for the response.

Examples:

```text
read timeout
server drops connection silently
server-side error
rate limit
access denied
```

The most dangerous case is:

```text
request probably reached server
+
client never received final response
```

The client cannot automatically assume:

```text
operation did not happen
```

This is the core ambiguity behind safe retries.

---

# 11. Same Word, Different Failure

The source makes a critical point:

```text
DNS timeout
```

and:

```text
read timeout
```

are both "timeouts" in logs.

But they mean radically different things.

### DNS timeout

```text
DNS
 ↓
no IP
 ↓
no server reached
```

### Read timeout

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
request
 ↓
server may have processed it
 ↓
response did not arrive
```

Therefore:

> **Error class alone is insufficient. Failure stage matters.** fileciteturn10file0L242-L250

---

# 12. HTTP Status Codes Revisited

Session 5 taught the meanings.

Session 7 adds a new question:

> **Does receiving this status tell you whether trying again will change the outcome?**

| Class | Meaning | Retry? |
|---|---|---|
| 1xx | informational | not final |
| 2xx | success | no need |
| 3xx | redirection | depends |
| 4xx | client error | usually no |
| 5xx | server error | usually yes |

The source explicitly frames this as the new retry question added to the old status-code vocabulary. fileciteturn10file0L255-L281

---

# 13. The Definition of Retry

This is one of the most important definitions in the entire session.

## Retry

A retry means:

> **Trying the exact same step again, with nothing changed.**

Same:

```text
method
body
headers
endpoint
request semantics
```

You are betting that:

```text
same action
+
more time
=
different result
```

because the failure was transient.

The source defines retry this way and contrasts it with handling a changed request. fileciteturn10file0L305-L329

---

# 14. What Is NOT a Retry?

Example:

```text
request fails because access token expired
```

Then:

```text
refresh token
↓
receive new token
↓
send request with new Authorization header
```

That is **not** a retry.

Why?

Because:

```text
request #1 ≠ request #2
```

The second request contains new information.

It is:

```text
error handling
```

not:

```text
blind retry
```

---

# 15. Another Non-Retry Example

Suppose:

```text
CDN A is dead
```

and you switch to:

```text
CDN B
```

That is:

```text
failover
```

not simply retrying the same action.

Likewise:

```text
DASH → HLS
```

is:

```text
fallback
```

not a retry.

---

# 16. The Test for "Is This a Retry?"

Ask:

> **If I changed nothing about the request and only waited, would attempt #2 be identical to attempt #1?**

If:

```text
YES
```

it can be a retry.

If:

```text
NO
```

you are handling the failure in some other way.

Examples of changes:

```text
new token
different endpoint
different payload
different CDN
different codec
different stream
```

---

# 17. When Do We Retry?

Before touching a retry counter, classify the failure.

The session gives four questions:

```text
1. Where did it fail?
2. Is the server admitting fault?
3. Is the action idempotent?
4. Is the failure transient or structural?
```

fileciteturn10file0L349-L374

---

# 18. Question 1 — Where Did It Fail?

Compare:

```text
DNS failure
```

with:

```text
read timeout
```

The second occurs after a request may have reached the server.

Therefore the retry risk is different.

### Rule

Always locate the failure on:

```text
DNS → TCP → TLS → write → processing → read
```

before deciding.

---

# 19. Question 2 — Is the Server Admitting Fault?

Generally:

```text
5xx
```

means:

```text
server failed
```

while:

```text
4xx
```

means:

```text
client/request was rejected
```

This does not mean:

```text
all 5xx are safe to retry
```

or:

```text
all 4xx are impossible to retry
```

It is a classification signal, not the whole decision.

---

# 20. Question 3 — Is the Action Idempotent?

Definition:

> An operation is idempotent if performing it multiple times has the same intended effect as performing it once.

Examples:

```text
SET value = X
```

twice:

```text
X
```

still.

But:

```text
charge $100
```

twice:

```text
$200
```

not equivalent to one charge.

---

# 21. Idempotency Table

The session gives:

| Method | Idempotent? | Retry after timeout? |
|---|---|---|
| GET | yes | safe, blindly |
| PUT | yes | safe, blindly |
| DELETE | yes | safe, blindly |
| POST | no | only with idempotency key |

fileciteturn10file0L424-L453

### Important nuance

"Safe to retry" does not mean:

```text
always retry
```

It means:

```text
repeating the operation does not create a duplicate effect by definition.
```

You still need to consider:

```text
load
retry budget
server signals
timeouts
backoff
```

---

# 22. Why Timeouts Are Dangerous for POST

Imagine:

```text
POST /payment
```

The server receives it.

It charges the card.

Then:

```text
server → response
```

gets lost.

Client sees:

```text
timeout
```

Client thinks:

```text
"payment failed"
```

and sends POST again.

Result:

```text
charge #1
charge #2
```

### Fix

Use an:

```text
Idempotency-Key
```

Example:

```http
POST /payment
Idempotency-Key: abc123
```

Server stores:

```text
abc123 → result
```

If request #2 arrives with:

```text
abc123
```

server returns the original result instead of performing the payment again.

---

# 23. Question 4 — Transient vs Structural

Ask:

> **Will waiting 200 ms change anything?**

If yes:

```text
possibly transient
```

If no:

```text
structural
```

Examples:

### Structural

```text
404
geo-block
invalid credentials
unsupported codec
invalid certificate
preflight failure
```

### Potentially transient

```text
503
connection timeout
read timeout
temporary DNS server failure
temporary CDN edge failure
network switch
```

---

# 24. No Point Retrying

The session explicitly lists:

```text
404
before handshake completes in a structural way
geo-blocks
preflight failures
auth failing outright
```

as cases where blind retry is not useful. fileciteturn10file0L377-L399

### 404

If:

```text
resource does not exist
```

retrying 200 ms later normally does nothing.

### Exception

The session later introduces a specific live-stream exception.

More on that below.

---

# 25. Geo-Block

Suppose:

```text
request rejected because user is in prohibited geography
```

Retrying:

```text
same request
same user
same geography
```

does not change the policy.

Therefore:

```text
NO RETRY
```

You need:

```text
different entitlement
different location
different resource
```

if such a legitimate alternative exists.

---

# 26. Preflight Failure

A browser can reject a request before the actual request is sent.

Example:

```text
CORS preflight
```

fails.

Then:

```text
actual API request
```

was never sent.

Retrying the actual API call does not fix the preflight policy.

The correct action is:

```text
fix CORS / policy / request
```

---

# 27. Auth Failure

If credentials are simply invalid:

```text
401/403
```

blindly repeating the same request with the same credentials changes nothing.

But:

```text
expired token
```

may allow a different flow:

```text
refresh token
↓
new Authorization header
↓
new request
```

Again:

```text
not a retry
```

It is a corrected request.

---

# 28. Worth Retrying

The source highlights cases where nothing has permanently decided against the request:

```text
500 / 502 / 503 / 504
connection timeout
read timeout
DNS server not responding
network switch
CDN edge/load balancer momentarily unavailable
cache not found
TLS handshake failure on an otherwise healthy host
```

fileciteturn10file0L403-L420

The key phrase is:

> **Nothing was decided against you.**

---

# 29. "Retryable" Does Not Mean "Retry Immediately"

Bad:

```text
retryable
→ retry now
→ retry now
→ retry now
```

Correct:

```text
retryable
→ controlled retry policy
```

That means:

```text
max attempts
+
backoff
+
jitter
+
possibly Retry-After
```

and sometimes:

```text
circuit breaker
```

---

# 30. The Naive Retry Loop

The session gives the classic bad implementation:

```python
while True:
    try:
        return call_upstream(request)
    except TimeoutError:
        continue
```

Problems:

### 1. No max count

A dead dependency produces:

```text
infinite loop
```

instead of:

```text
fast failure
```

### 2. No delay

Clients hammer the dependency:

```text
as fast as CPU/network allows
```

### 3. No coordination

Thousands of clients can retry together.

When the dependency recovers:

```text
all retry loops fire together
```

This creates:

```text
THUNDERING HERD
```

The source identifies all three problems. fileciteturn10file0L473-L493

---

# 31. Exponential Backoff

The session's retry function uses:

```python
delay = min(cap, base * 2**attempt)
```

Conceptually:

```text
attempt 0 → base
attempt 1 → 2 × base
attempt 2 → 4 × base
attempt 3 → 8 × base
...
```

until:

```text
cap
```

is reached.

Example from the session:

```text
base = 0.2 s
cap = 8 s
max_attempts = 5
```

fileciteturn10file0L497-L527

---

# 32. Why Exponential Backoff?

Without backoff:

```text
server fails
↓
client retry rate stays high
```

With exponential backoff:

```text
server fails
↓
retry interval grows
↓
load generated by each client decays
```

So backoff gives the dependency time to recover.

### Mental model

```text
failure
 ↓
wait longer
 ↓
try
 ↓
if still failing
 ↓
wait even longer
```

---

# 33. Max Attempts

Backoff alone is not enough.

Suppose:

```text
dependency is permanently dead
```

You could still keep waiting:

```text
0.2 s
0.4 s
0.8 s
1.6 s
3.2 s
...
```

forever.

Therefore:

```text
max_attempts
```

caps total damage.

The session describes this as:

> a dead dependency fails in seconds, not forever. fileciteturn10file0L503-L518

---

# 34. Jitter

Suppose 1,000 clients all fail at:

```text
t = 0
```

and all calculate:

```text
retry after 200 ms
```

Then:

```text
t = 200 ms
→ 1,000 retries
```

They synchronize.

This is bad.

### Jitter

Instead of:

```text
delay = 200 ms
```

use:

```text
random delay between 0 and 200 ms
```

The session calls this:

```text
FULL JITTER
```

with:

```python
random.uniform(0, delay)
```

fileciteturn10file0L497-L527

---

# 35. Full Jitter

Formula:

```text
exponential delay = min(cap, base × 2^attempt)

actual sleep =
    random(0, exponential delay)
```

### Example

If:

```text
delay = 800 ms
```

then clients might retry at:

```text
21 ms
134 ms
401 ms
723 ms
...
```

rather than:

```text
exactly 800 ms
```

This decorrelates clients.

---

# 36. Backoff vs Jitter

Do not confuse them.

### Backoff

Controls:

```text
how the retry interval grows
```

### Jitter

Controls:

```text
how synchronized clients are
```

Together:

```text
exponential backoff
+
full jitter
```

gives:

```text
less load
+
less synchronization
```

---

# 37. Three Retry Knobs

The session explicitly frames the three knobs:

### 1. Max attempts

```text
caps total damage
```

### 2. Exponential backoff

```text
spaces retries out
```

### 3. Full jitter

```text
decorrelates clients
```

fileciteturn10file0L503-L519

Memorize this table.

| Knob | Main purpose |
|---|---|
| Max attempts | limit total damage |
| Exponential backoff | reduce retry pressure over time |
| Jitter | prevent synchronized retries |

---

# 38. Retry-After

Sometimes the server tells the client exactly how long to wait.

Example:

```http
Retry-After: 10
```

The client should generally honor it.

The session highlights:

```text
429 Too Many Requests
503 Service Unavailable
408 Request Timeout
```

with their corresponding retry behavior. fileciteturn10file0L529-L552

---

# 39. 429 Too Many Requests

Meaning:

```text
server is healthy
you are asking too fast
```

If response says:

```http
Retry-After: 5
```

do not simply apply your own:

```text
exponential backoff
```

and ignore the server.

The server has better knowledge of:

```text
its own load
```

than your client does.

---

# 40. 503 Service Unavailable

Often means:

```text
temporary overload
maintenance
load shedding
dependency issue
```

If:

```http
Retry-After
```

is present:

```text
honor it
```

instead of blindly applying your own curve.

---

# 41. 408 Request Timeout

The source's framing:

```text
server gave up waiting for the client to finish sending
```

A retry can be reasonable because:

```text
the request was not rejected as an invalid operation
```

but the actual retry still needs the normal safety policy.

---

# 42. Rule: Server-Given Timing Beats Your Guess

A useful mental rule:

```text
Retry-After
      ↓
server knows its state
      ↓
trust it
```

The session summarizes:

> A server that hands you a number is smarter about its own load than your exponential curve is. fileciteturn10file0L529-L552

---

# 43. Circuit Breaker

Backoff protects:

```text
one client
```

from retrying too aggressively.

A circuit breaker protects:

```text
the dependency
```

from an entire fleet that knows it is down.

It also protects callers from repeatedly paying:

```text
connection timeout
+
read timeout
+
retry delay
```

when the dependency is already known to be unhealthy.

---

# 44. Circuit Breaker States

There are three:

```text
CLOSED
   ↓
OPEN
   ↓
HALF-OPEN
   ↓
CLOSED
```

---

# 45. CLOSED

Normal state.

```text
requests flow
failures counted
```

Example:

```text
request
 ↓
dependency
 ↓
response
```

Failures increment the breaker state.

---

# 46. OPEN

Failure threshold is crossed.

Now:

```text
DO NOT CALL DEPENDENCY
```

Requests fail locally.

Instead of:

```text
client
 ↓
network
 ↓
dead service
 ↓
timeout
```

you get:

```text
client
 ↓
circuit breaker
 ↓
fast failure
```

This is much cheaper.

The source explicitly says an OPEN breaker causes requests to fail instantly and locally. fileciteturn10file0L562-L580

---

# 47. HALF-OPEN

After a cooldown:

```text
allow ONE probe
```

If:

```text
probe succeeds
```

then:

```text
HALF-OPEN → CLOSED
```

If:

```text
probe fails
```

then:

```text
HALF-OPEN → OPEN
```

### Why only one probe?

You want to test:

```text
"is the dependency healthy?"
```

without releasing the entire fleet onto it.

---

# 48. Backoff vs Circuit Breaker

Very important MCQ/interview distinction:

| Mechanism | Protects | Scope |
|---|---|---|
| Backoff | upstream from one caller's retry pressure | per request/client |
| Circuit breaker | dependency from fleet-wide repeated calls | per dependency |

### Simple memory

```text
BACKOFF
= slow down

CIRCUIT BREAKER
= stop calling
```

---

# 49. Video Player Case Study

The session uses a video player because its failure problem is harder than a normal API client.

Why?

An API client can show:

```text
exception
5xx
error
```

A video player can fail silently as:

```text
frozen frame
spinner
rebuffering
```

The user experiences the failure directly.

The source also notes that one play session can make hundreds of requests:

```text
manifest
key
segments
```

and each may have a different retry policy. fileciteturn10file0L587-L623

---

# 50. Video Player Has a Human Retry Budget

An API can:

```text
retry for 30 seconds
```

and eventually return an error.

A video player cannot comfortably do:

```text
spinner for 30 seconds
```

The viewer tolerates only a few seconds of rebuffering.

Therefore:

```text
technical correctness
```

is not enough.

The retry policy must also respect:

```text
user experience budget
```

---

# 51. Not Every Video Failure Is a Network Failure

The source explicitly gives:

```text
DRM key denied
unsupported codec
decoder crash
```

These are not necessarily transient network problems.

The correct action may be:

```text
fallback
```

rather than:

```text
retry
```

fileciteturn10file0L604-L618

---

# 52. Video Network Failure: DNS

If:

```text
DNS server is not responding
```

a player may:

```text
retry
```

or:

```text
switch to another CDN hostname
```

This is more powerful than repeatedly asking the same dead path.

### Key distinction

```text
retry same CDN
```

vs

```text
fail over to another CDN
```

The latter is a different action.

---

# 53. Video Connection / Read Timeout

Manifest and segment requests do not necessarily have the same urgency.

Therefore the player can use:

```text
different timeout budgets
```

for:

```text
manifest
segment
key
```

A retry may be valid but:

```text
tuned per request type
```

---

# 54. Network Switch: 4G → 5G

Suppose a segment is downloading.

Device switches:

```text
4G → 5G
```

The IP address may change.

The in-flight request can therefore become:

```text
dead
```

not merely:

```text
slow
```

The correct action:

```text
fresh request
```

This is a new request after an environment change.

---

# 55. Wi-Fi Goes Offline During Playback

A smart player does not immediately destroy playback.

If enough data is already buffered:

```text
network unavailable
↓
buffer continues playback
↓
wait
↓
retry once buffer becomes low
```

This is:

```text
deferred retry
```

rather than:

```text
immediate retry storm
```

The session explicitly describes this behavior. fileciteturn10file0L627-L657

---

# 56. CDN Edge Failure

Suppose:

```text
CDN A
```

is unhealthy.

A multi-CDN player may:

```text
switch to CDN B
```

rather than:

```text
retry CDN A
```

This is:

```text
failover
```

not:

```text
retry
```

---

# 57. The 404 Exception: Live Video Segments

Normally:

```text
404
→ don't retry
```

But live streaming has an important exception.

The newest live segment may return:

```text
404
```

because:

```text
origin is still encoding it
```

or:

```text
CDN has not propagated it yet
```

The segment may exist:

```text
hundreds of milliseconds later
```

Therefore:

```text
404 on live-edge segment
→ short bounded retry
```

The session recommends roughly:

```text
2–3 attempts
+
small fixed delay
```

and explicitly says this exception should not be generalized to ordinary on-demand assets. fileciteturn10file0L661-L682

---

# 58. Important MCQ Trap: "404 Never Retry"

Wrong.

Correct:

```text
Normal 404
→ don't retry

Live-stream newest segment 404
→ bounded short retry may be appropriate
```

Why?

Because:

```text
404 meaning depends on system semantics
```

not merely the number.

---

# 59. Custom Application Status Codes

The source mentions application-specific codes such as:

```text
419
474
475
```

These are not standard HTTP meanings in the same way as the core RFC-defined status vocabulary.

The lesson is:

> Your own system's status codes need the same classify-before-retry discipline.

For example:

```text
419
```

may indicate:

```text
token/session expiry
```

which should be handled as authentication logic rather than blindly retried.

Do not memorize these numbers as universal HTTP meanings.

Memorize:

```text
application-specific codes need documented semantics
```

---

# 60. Sometimes the Answer Is FALL BACK

The session gives several examples.

| Failure | Better action |
|---|---|
| DRM key denied | lower DRM level / different key system |
| unsupported codec | alternate codec rendition |
| DASH manifest failure | HLS fallback |
| repeated segment/init failure | bounded retries, then fatal coded error |
| ad failure | continue to content |
| iOS media-state failure | rebuild AVPlayer pipeline |

fileciteturn10file0L685-L717

---

# 61. DRM Key Denied

Suppose:

```text
DRM key request
↓
denied
```

Retrying the exact same request:

```text
same identity
same entitlement
same key
```

will likely return:

```text
same denial
```

A better strategy may be:

```text
fallback to lower DRM security level
```

or:

```text
different key system
```

This is:

```text
fallback
```

not:

```text
retry
```

---

# 62. Unsupported Codec

If a device genuinely cannot decode:

```text
AV1
```

retrying the same AV1 segment does not make the hardware support AV1.

Instead:

```text
choose another rendition
```

such as a supported codec.

Again:

```text
different action
```

not:

```text
retry
```

---

# 63. DASH → HLS

If:

```text
DASH manifest
```

fails, a player may switch to:

```text
HLS rendition
```

rather than repeatedly retrying DASH.

This is another example of:

```text
fallback
```

---

# 64. Repeated Child Manifest / Segment Failure

These can be:

```text
network-retryable
```

initially.

But repeated failures should not create:

```text
infinite retry
```

Eventually the player escalates to:

```text
fatal coded error
```

The source gives examples such as:

```text
8018
8022
```

in the case-study framing. fileciteturn10file0L701-L717

---

# 65. Ad Failure

An ad request that fails or times out can have:

```text
near-zero user time budget
```

So the player may:

```text
skip ad
↓
play content
```

rather than:

```text
retry ad repeatedly
```

This is graceful degradation.

---

# 66. iOS ResetMedia

The source includes a platform-specific case:

```text
ResetMedia
```

The failure is not necessarily:

```text
network
```

It can be:

```text
device media pipeline state
```

The recovery action is:

```text
tear down
+
rebuild
```

the media player pipeline.

This is a strong reminder:

> The correct recovery action depends on the failure domain.

---

# 67. Application-Level Signals Matter

Transport signals include:

```text
HTTP status
timeout
connection error
```

But application responses can contain richer instructions:

```text
degraded
use lower bitrate
maintenance
try again in N minutes
```

The session's point:

> Mature retry policy reads both transport-layer and application-layer signals. fileciteturn10file0L721-L740

### Therefore

Do not build retry logic around:

```text
status code only
```

---

# 68. System-Level Resilience

Now the session moves from:

```text
individual request
```

to:

```text
entire system
```

The key question becomes:

> What happens when the dependency or infrastructure itself is unavailable?

A robust system uses:

```text
failover
+
feature flags
+
graceful degradation
+
stale data
+
controlled recovery
```

---

# 69. Stale-If-Error

Session 6 showed:

```text
origin dies
↓
edge cache can still serve stale data
```

Session 7 extends this idea.

`stale-if-error` allows a cache to:

```text
serve the last known-good copy
```

even if the origin cannot currently be reached.

This is explicitly described in the session as the cache choosing:

```text
known-stale data
```

instead of:

```text
nothing
```

fileciteturn10file0L759-L781

---

# 70. Stale Data Can Be Better Than No Data

This is a deep reliability principle.

Suppose:

```text
perfectly fresh data = unavailable
```

You have two options:

```text
A. show nothing
B. show slightly stale data
```

For many systems:

```text
B > A
```

Examples:

```text
stale recommendations
cached product information
slightly stale price
lower-quality video
```

The system sacrifices freshness to preserve availability.

---

# 71. Fastly Incident and Concentration Risk

The session references the June 8, 2021 Fastly incident.

Its framing:

```text
85% of Fastly's network returning errors
~1 minute to detect
~49 minutes to 95% recovered
```

The point is not to memorize the incident timeline.

The point is:

```text
CDN itself can fail
```

Session 6 asked:

```text
what if origin dies?
```

Session 7 asks:

```text
what if edge/region dies?
```

---

# 72. Failover

Failover means:

```text
Primary
   ↓
unhealthy
   ↓
Backup
```

Backup might be:

```text
different region
different provider
different CDN
```

### Important

The client does not keep asking the dead primary forever.

It changes the destination.

That is:

```text
failover
```

not:

```text
retry
```

The source makes this distinction explicitly. fileciteturn10file0L785-L803

---

# 73. Circuit Breaker + Failover

These work together:

```text
primary failing
       ↓
circuit breaker OPEN
       ↓
stop calling primary
       ↓
fail over to backup
```

This avoids:

```text
retry dead dependency
```

and instead uses:

```text
alternative dependency
```

---

# 74. Feature Flags

A feature flag lets you turn off expensive/non-essential functionality during an incident.

Example:

```text
recommendations = OFF
```

while:

```text
core playback = ON
```

Or:

```text
personalisation = OFF
```

while:

```text
page delivery = ON
```

This reduces load on the critical path without requiring a code deployment.

---

# 75. Graceful Degradation

Graceful degradation means:

> **Answer something worse rather than answer nothing.**

Examples from the session:

```text
full bitrate
   ↓
lower bitrate

live recommendations
   ↓
cached recommendations

fresh price
   ↓
slightly stale price

full DRM level
   ↓
lower DRM level
```

The source explicitly frames the strategy as:

```text
Answer smaller, not "no".
```

fileciteturn10file0L785-L803

---

# 76. Failover vs Graceful Degradation

Do not mix them.

### Failover

Change:

```text
WHO serves the request
```

Example:

```text
CDN A → CDN B
```

### Graceful degradation

Change:

```text
WHAT quality/features the request gets
```

Example:

```text
1080p → 480p
```

---

# 77. The Second Outage

One of the most important ideas in Session 7:

> **The moment a dependency recovers can be more dangerous than the original failure.**

Why?

During the outage:

```text
clients are waiting
```

When service recovers:

```text
all clients wake up
↓
all retry
↓
traffic spikes
↓
recovering service overloads
↓
service crashes again
```

The source calls this the:

```text
second outage
```

and describes a recovering service at only about 1% normal capacity potentially receiving close to normal traffic immediately. fileciteturn10file0L807-L826

---

# 78. Why Jitter Helps Recovery

Without jitter:

```text
failure at t=0
↓
all clients calculate same backoff
↓
all retry at t=1
↓
all retry at t=2
```

With jitter:

```text
client A → 0.2s
client B → 0.7s
client C → 1.1s
client D → 1.8s
...
```

Recovery traffic is spread over time.

---

# 79. Half-Open Probe Staggering

A circuit breaker can also create synchronization.

Bad:

```text
10,000 breakers
cooldown ends
↓
10,000 half-open probes
```

Better:

```text
stagger half-open probes
```

so the recovering dependency sees:

```text
small controlled traffic
```

instead of:

```text
instant fleet-wide probe
```

---

# 80. Slow Start During Recovery

A recovering service may not immediately be capable of:

```text
100% normal traffic
```

It may need time to:

```text
warm caches
re-establish connections
load data
initialize workers
```

Therefore:

```text
accept traffic gradually
```

rather than:

```text
jump immediately to full load
```

This is:

```text
slow-start recovery
```

---

# 81. The Final Five-Step Flowchart

The session's synthesis gives one flowchart for essentially every failure:

```text
1. CLASSIFY
   ↓
2. STRUCTURAL?
   ↓
3. TRANSIENT → RETRY
   ↓
4. DEPENDENCY-LEVEL FAILURE?
   ↓
5. FAIL OVER / DEGRADE / SERVE STALE
```

The exact source wording is:

```text
1. Classify
2. Structural?
3. Transient → retry
4. Dependency-level
5. No good answer left
```

with the corresponding actions shown on the synthesis slide. fileciteturn10file0L830-L855

---

# 82. Step 1 — Classify

Ask:

```text
Where did it fail?

4xx or 5xx?

Idempotent?
```

Do not touch the retry counter until you know this.

---

# 83. Step 2 — Structural?

Examples:

```text
404
geo-block
preflight failure
invalid credentials
unsupported codec
```

Action:

```text
STOP RETRYING
```

Handle the problem.

---

# 84. Step 3 — Transient → Retry

If failure is transient:

```text
retry
```

but safely:

```text
exponential backoff
+
full jitter
+
max attempts
```

And:

```text
Retry-After
```

takes precedence when provided.

---

# 85. Step 4 — Dependency-Level Failure

If repeated failures indicate:

```text
dependency is down
```

trip:

```text
circuit breaker
```

Then:

```text
fail fast
```

instead of paying timeout cost repeatedly.

---

# 86. Step 5 — No Good Answer Left

If the primary path is unavailable:

```text
fail over
```

or:

```text
degrade
```

or:

```text
serve stale
```

The objective is:

```text
availability
```

not:

```text
perfect ideal response at any cost
```

---

# 87. The Most Important Distinctions

## Retry vs handling

```text
Retry:
same request again

Handling:
change something based on failure
```

## Retry vs failover

```text
Retry:
same dependency/action

Failover:
different dependency/path
```

## Retry vs fallback

```text
Retry:
same operation

Fallback:
different valid representation/strategy
```

## Backoff vs circuit breaker

```text
Backoff:
slow down retries

Circuit breaker:
stop calling dependency
```

## Failover vs graceful degradation

```text
Failover:
change provider/path

Degradation:
reduce quality/features
```

---

# 88. High-ROI Retry Decision Table

| Situation | Retry? | Better action |
|---|---|---|
| 404 normal asset | No | stop |
| Geo-block | No | handle policy |
| Invalid credentials | No | correct credentials |
| Expired token | Not same request | refresh token |
| 500 | Usually yes | backoff + jitter |
| 502 | Usually yes | backoff + jitter |
| 503 | Usually yes | honor Retry-After |
| 504 | Usually yes | backoff + jitter |
| Connection timeout | Often yes | retry safely |
| Read timeout on GET | Yes, generally safe | retry |
| Read timeout on POST | Dangerous | idempotency key / inspect semantics |
| Live-edge segment 404 | Exception | 2–3 bounded attempts |
| CDN A down | Don't hammer A | fail over to CDN B |
| Unsupported codec | No | alternate rendition |
| DRM denial | No | alternate DRM path |
| Wi-Fi drops while buffered | Defer | keep playing, retry later |
| Recovering dependency | Controlled | jitter + slow start |

---

# 89. Idempotency: Deep Mental Model

Think in terms of:

```text
operation effect
```

If repeating the request produces the same final state:

```text
idempotent
```

Example:

```text
PUT /user/7/name
body = "Tanmay"
```

Repeated:

```text
set name = Tanmay
set name = Tanmay
set name = Tanmay
```

Final state is the same.

---

# 90. Non-Idempotent Example

```text
POST /orders
```

could create:

```text
order #123
```

then retry could create:

```text
order #124
```

So the client cannot safely assume:

```text
timeout → nothing happened
```

---

# 91. Idempotency Key Flow

For a non-idempotent request:

```text
Client
  |
  | POST
  | Idempotency-Key: K123
  ↓
Server
  |
  | process K123
  ↓
result stored
```

If timeout occurs:

```text
Client retries K123
       ↓
Server sees K123 already processed
       ↓
return stored result
```

Thus:

```text
same logical operation
```

can safely survive a retry.

---

# 92. Failure Stage + Idempotency

These two dimensions combine.

### Case A

```text
GET
+
read timeout
```

Potentially safe:

```text
retry
```

### Case B

```text
POST payment
+
read timeout
```

Dangerous:

```text
server may already have charged
```

### Case C

```text
POST payment
+
connection never established
```

The operation likely did not reach the server, but a robust client should still use a consistent idempotency design for payment operations.

### Core rule

> **You need both failure location and operation semantics.**

---

# 93. Retry Budget

A retry consumes resources:

```text
client CPU
network bandwidth
server CPU
server connections
latency
user patience
```

Therefore:

```text
retry budget
```

should be considered.

For a video player:

```text
few seconds
```

may be the practical budget.

For a background job:

```text
minutes/hours
```

may be acceptable.

---

# 94. User Experience Is Part of Retry Design

A technically correct policy can still be a bad product.

Example:

```text
retry with exponential backoff for 2 minutes
```

is mathematically reasonable.

For a live video:

```text
spinner for 2 minutes
```

is unacceptable.

So retry policy depends on:

```text
system semantics
+
failure type
+
user-visible latency budget
```

---

# 95. Cache + Failure Connection to Session 6

Session 6:

```text
CDN cache
```

was primarily discussed as:

```text
performance
```

Session 7 adds:

```text
availability
```

A stale cache can allow:

```text
origin down
↓
still serve old response
```

Therefore caching can be:

```text
performance mechanism
+
resilience mechanism
```

---

# 96. Why "Serve Stale" Is Not a Retry

Suppose:

```text
origin unavailable
```

A retry would be:

```text
ask origin again
```

Serving stale is:

```text
don't ask origin
use known-good previous data
```

Therefore:

```text
stale-if-error ≠ retry
```

It is a degradation strategy.

---

# 97. Retry Storm

A retry storm looks like:

```text
dependency fails
 ↓
10k clients retry
 ↓
dependency receives 10k extra requests
 ↓
dependency becomes more unhealthy
 ↓
more failures
 ↓
more retries
```

This is a positive feedback loop:

```text
failure
→ retries
→ load
→ more failure
→ more retries
```

---

# 98. Thundering Herd

The specific synchronization problem:

```text
many clients
+
same retry schedule
+
same recovery moment
```

creates:

```text
THUNDERING HERD
```

Mitigation:

```text
jitter
+
staggered probes
+
slow start
```

---

# 99. Retry Storm vs Thundering Herd

They are related but not identical.

### Retry storm

Too many retries during failure.

```text
failure
→ clients repeatedly call
```

### Thundering herd

Many clients act simultaneously.

```text
same timing
→ synchronized load spike
```

You can have:

```text
retry storm without perfect synchronization
```

and:

```text
thundering herd during recovery
```

---

# 100. Circuit Breaker Prevents Both

When:

```text
breaker OPEN
```

clients do not call the dependency.

This prevents:

```text
continuous retry load
```

Then:

```text
HALF-OPEN
```

allows controlled testing.

---

# 101. Graceful Degradation Examples

The session's examples are worth remembering:

```text
lower bitrate
instead of
no video
```

```text
cached recommendations
instead of
no recommendations
```

```text
slightly stale price
instead of
spinner
```

This is a general reliability philosophy:

> Preserve the critical path, sacrifice optional quality first.

---

# 102. Critical Path Thinking

Suppose a page contains:

```text
core content
recommendations
ads
personalisation
analytics
```

During an incident:

```text
core content = essential
recommendations = optional
personalisation = optional
analytics = optional
```

Feature flags can disable the optional pieces.

Then:

```text
critical path survives
```

---

# 103. Video Player as a System

A video player may depend on:

```text
DNS
CDN
manifest
DRM
codec
segments
ads
decoder
network
device media pipeline
```

Therefore a single "retry policy" is insufficient.

Different components need:

```text
different failure classifications
different retry budgets
different fallbacks
```

---

# 104. Manifest vs Segment

A manifest tells the player:

```text
what streams/segments exist
```

A segment is:

```text
actual media chunk
```

Therefore a manifest failure can be more fundamental.

A segment failure may be recoverable through:

```text
retry
different CDN
different bitrate
```

The player can sometimes continue from buffered content even while segment fetching is temporarily broken.

---

# 105. CDN Switching in Video

Multi-CDN:

```text
CDN A
CDN B
CDN C
```

If:

```text
A unhealthy
```

switch:

```text
A → B
```

This reduces dependence on one CDN.

It also demonstrates the Session 6 lesson:

```text
concentration creates dependency
```

and Session 7 response:

```text
failover
```

---

# 106. Failure Recovery Hierarchy

A useful hierarchy:

```text
LEVEL 1
Fix the request
    ↓
LEVEL 2
Retry safely
    ↓
LEVEL 3
Circuit-break unhealthy dependency
    ↓
LEVEL 4
Fail over
    ↓
LEVEL 5
Gracefully degrade
    ↓
LEVEL 6
Serve stale / cached known-good data
```

Not every system has every level.

The important idea is:

```text
don't repeatedly attack a known-dead path.
```

---

# 107. Session 7 Assignment

The assignment is:

> **Wrap a flaky client in a policy, and prove it survives.**

The source asks you to:

1. Take an existing client.
2. Build a resilient-call wrapper from scratch.
3. Classify failures.
4. Retry only when appropriate.
5. Use full jitter.
6. Cap max attempts.
7. Honor `Retry-After`.
8. Open a circuit after N consecutive failures.
9. Fail fast while the circuit is open.
10. Build a test server that can intentionally misbehave.

Endpoints suggested:

```text
/sometimes-500
```

Fails a configurable fraction of the time.

```text
/always-503-with-retry-after
```

Always returns 503 plus a retry instruction.

```text
/always-down
```

Used to trigger the circuit breaker.

Deliverable:

```text
wrapper
+
test server
+
short log/plot
```

showing:

```text
backoff curve
circuit opening
circuit closing
```

The source describes this assignment on page 35. fileciteturn10file0L859-L879

---

# 108. Homework Options

The source gives five options.

## 1. Simulate 1,000 clients

Compare:

```text
naive retry
vs
backoff + jitter
```

against a rate-limited server.

Measure:

```text
requests in first second
```

---

## 2. Find three video cases that aren't retries

For each:

```text
what it is instead
```

Examples:

```text
DRM denial → fallback
codec unsupported → alternate rendition
CDN failure → failover
```

---

## 3. Design an idempotency-key scheme

Answer:

```text
where are keys stored?
how long?
what does duplicate request return?
what happens if two requests arrive concurrently?
```

---

## 4. Study the Fastly postmortem

Map the incident onto:

```text
Classify
→ Structural?
→ Retry?
→ Dependency-level
→ Failover/degrade/stale
```

---

## 5. Highest-ROI homework

Instrument a real client that currently has no retry logic.

Add:

```text
classification
+
backoff
+
jitter
+
cap
```

Then intentionally break:

```text
Wi-Fi
or
port
```

and observe recovery.

The source explicitly marks this as the one to do if you only do one. fileciteturn10file0L883-L904

---

# 109. High-ROI MCQ Traps

1. **A timeout is not one type of failure.**
2. DNS timeout and read timeout happen at different request stages.
3. Retry means the same request again with nothing changed.
4. Refreshing an expired token is not a retry.
5. Switching CDN is failover, not retry.
6. Switching codec is fallback, not retry.
7. `5xx` generally indicates server failure, but does not prove the operation did not happen.
8. `4xx` generally indicates client/request rejection, but application-specific semantics matter.
9. GET is idempotent.
10. PUT is idempotent.
11. DELETE is idempotent.
12. POST is not inherently idempotent.
13. A POST can be made safely retryable using an idempotency key.
14. A timeout after a POST does not prove the server did nothing.
15. 404 normally should not be retried.
16. A live-stream newest segment can be a deliberate exception to 404 handling.
17. Exponential backoff alone does not prevent synchronization.
18. Full jitter randomizes within the computed delay.
19. Max attempts prevent infinite retry loops.
20. `Retry-After` should generally be honored.
21. Backoff protects against retry pressure from a caller.
22. Circuit breaker protects a dependency from repeated fleet-wide calls.
23. CLOSED means normal traffic.
24. OPEN means fail fast locally.
25. HALF-OPEN means controlled probe.
26. A circuit breaker does not mean "retry less"; it means "stop calling for now."
27. Stale-if-error is not a retry.
28. Failover changes the dependency/path.
29. Graceful degradation changes quality/features.
30. Jitter is especially important when many clients fail together.
31. Recovery itself can create a second outage.
32. Slow-start can protect a recovering service.
33. Half-open probes should be staggered at fleet scale.
34. A video player's retry budget is constrained by human tolerance.
35. Not every video failure is a network failure.
36. DRM denial may require fallback, not retry.
37. Unsupported codec requires another representation.
38. Ad failure may be handled by skipping the ad.
39. Application-layer response content can contain retry/fallback instructions.
40. Feature flags can disable non-essential functionality during incidents.
41. A cache can be a resilience mechanism, not only a performance mechanism.
42. Slightly stale data can be preferable to total unavailability.

---

# 110. Interview Questions

## Q1. What is a retry?

> A retry is sending the exact same operation again without changing the request, based on the belief that the original failure was transient.

---

## Q2. Why is a read timeout different from a DNS timeout?

> A DNS timeout means the request may never have reached a server. A read timeout happens after a connection and request may already exist, so the server may have partially or fully processed the operation.

---

## Q3. Why can retrying POST be dangerous?

> POST is not inherently idempotent. If the server completed the operation but the response was lost, retrying can perform the operation twice.

---

## Q4. How do you safely retry a payment POST?

> Give the logical operation an idempotency key. The server stores the result associated with that key and returns the same result if the key is seen again.

---

## Q5. Why use exponential backoff?

> It spaces retries farther apart as failures continue, reducing sustained pressure on the unhealthy dependency.

---

## Q6. Why add jitter?

> Without jitter, many clients that failed simultaneously calculate the same retry times and synchronize. Jitter decorrelates them.

---

## Q7. What is full jitter?

> Compute the exponential backoff delay, then choose a random delay uniformly between zero and that delay.

```text
delay = min(cap, base × 2^attempt)
sleep = random(0, delay)
```

---

## Q8. Why is max retry count necessary?

> A permanently unavailable dependency would otherwise create an infinite retry loop and consume resources indefinitely.

---

## Q9. What is a circuit breaker?

> A dependency-level mechanism that stops calling a known-unhealthy service and fails fast until a controlled probe determines that it has recovered.

---

## Q10. Explain CLOSED, OPEN and HALF-OPEN.

> CLOSED allows normal traffic and counts failures. OPEN rejects calls locally. HALF-OPEN allows a controlled probe; success closes the circuit, failure opens it again.

---

## Q11. Backoff vs circuit breaker?

> Backoff slows down retries from a caller. A circuit breaker stops calls to a dependency entirely once repeated failures indicate that continuing is pointless.

---

## Q12. Why honor Retry-After?

> The server has direct knowledge of its load/recovery schedule, so its explicit wait instruction is usually better than the client's guessed retry curve.

---

## Q13. Why isn't every 4xx permanent?

> 4xx generally represents client-side rejection, but some application-specific situations may require handling such as refreshing credentials or following application-specific semantics. The exact meaning still needs classification.

---

## Q14. When can a 404 be retryable?

> In this session's specific video case, a live stream's newest segment may temporarily return 404 while it is still being encoded or propagated. A short bounded retry can therefore be appropriate. Normal on-demand 404s should not be retried.

---

## Q15. What is graceful degradation?

> Returning a reduced-quality or reduced-feature result instead of failing completely.

Examples:

```text
1080p → 480p
live recommendations → cached recommendations
fresh data → slightly stale data
```

---

## Q16. What is failover?

> Switching from an unhealthy primary dependency to a different healthy dependency, such as another region or CDN.

---

## Q17. Why can recovery cause another outage?

> Clients that were waiting during the outage can resume simultaneously. Without jitter, staggered probes, and slow start, the recovering service can receive a sudden traffic spike larger than its available capacity.

---

## Q18. What is stale-if-error?

> A cache policy that allows a previously valid cached response to be served when the origin cannot be reached or fails.

---

## Q19. Why is stale data sometimes better than fresh-or-nothing?

> Because availability can be more valuable than perfect freshness. A slightly old response can still provide useful functionality while the dependency is recovering.

---

## Q20. Why is video retry logic harder than API retry logic?

> A player makes many different requests with different failure modes and retry budgets, and the failure is directly visible to the user as buffering or freezing. Some failures also require fallback rather than retry.

---

# 111. Common System Design Scenario

### Problem

Your API depends on:

```text
Payment Service
```

Payment service starts returning:

```text
503
```

### Bad solution

```text
retry forever
```

### Better solution

```text
503
↓
check Retry-After
↓
retry with bounded backoff + jitter
↓
if repeated failures
↓
circuit breaker OPEN
↓
fail fast
```

If another valid provider exists:

```text
fail over
```

For a payment, you must also ensure:

```text
idempotency key
```

because payment creation is non-idempotent.

---

# 112. Another System Design Scenario

### Problem

CDN A is unavailable.

### Bad

```text
retry CDN A forever
```

### Better

```text
detect repeated CDN A failure
↓
breaker/failure threshold
↓
switch to CDN B
```

For video:

```text
CDN A
↓
failure
↓
CDN B
↓
continue playback
```

If bandwidth is poor:

```text
1080p
↓
720p
↓
480p
```

This combines:

```text
failover
+
graceful degradation
```

---

# 113. Another Scenario: Recommendations Service Down

Critical page:

```text
product page
```

Optional dependency:

```text
recommendations
```

Do not make the entire page fail.

Instead:

```text
recommendations timeout
↓
circuit opens
↓
recommendation widget disabled
↓
product page still works
```

Could also use:

```text
cached recommendations
```

This is:

```text
graceful degradation
```

---

# 114. Failure Handling by Layer

```text
DNS
  ↓
resolve / failover

TCP
  ↓
connect / timeout

TLS
  ↓
handshake / fallback or fix configuration

HTTP
  ↓
status classification

Application
  ↓
business semantics

Dependency
  ↓
circuit breaker

System
  ↓
failover / degradation / stale
```

The deeper the failure, the more system-level the response tends to become.

---

# 115. A Practical Retry Policy

A robust generic policy can be mentally represented as:

```text
call()
 ↓
failure
 ↓
classify
 ├── structural → handle / stop
 ├── auth → refresh / correct
 ├── fallback available → fallback
 └── transient
       ↓
    idempotent?
       ├── yes → retry
       └── no → idempotency key / safe handling
                    ↓
               Retry-After?
                    ├── yes → honor it
                    └── no → exponential backoff
                               +
                               full jitter
                    ↓
                max attempts
                    ↓
              still failing?
                    ↓
             circuit breaker
                    ↓
          failover / degrade / stale
```

---

# 116. Session 7 vs Session 6

| Session 6 | Session 7 |
|---|---|
| Move work/data closer | Survive when that path fails |
| CDN | Failure around CDN/origin |
| Cache | Stale-if-error |
| Edge | Edge can fail |
| DDoS protection | Dependency recovery |
| Cache hit | Known-good stale data |
| Performance | Reliability |
| Connection pooling | Retry/circuit control |
| Concentration risk | Failover |

### Connection

Session 6:

```text
"What if the origin is slow/dead?"
```

Session 7:

```text
"What if the edge, origin, network, or dependency is failing?"
```

---

# 117. Session 7 vs Session 8

Session 7 builds:

```text
retry
+
circuit breaker
+
fallback
+
degradation
```

Session 8 puts these into:

```text
adaptive bitrate
segment prefetch
CDN/tiered cache
DRM tiers
multi-CDN switching
manifest fallback
```

The source explicitly says the next session takes today's retry/fallback logic into a real large-scale video pipeline. fileciteturn10file0L908-L925

---

# 118. High-ROI Numbers / Exact Details

Memorize only numbers that are likely to matter for the source's MCQs/assignment:

```text
max_attempts example = 5
base backoff example = 0.2 s
cap example = 8.0 s

live segment 404:
≈ 2–3 bounded attempts
+ small fixed delay

Fastly incident framing:
85% network errors
~1 min detection
~49 min to 95% recovery

circuit breaker:
CLOSED → OPEN → HALF-OPEN

HTTP methods:
GET / PUT / DELETE → idempotent
POST → not inherently idempotent
```

Do not over-memorize benchmark numbers if the exam is conceptual.

---

# 119. 60-Second Revision

```text
SESSION 7 = BUILD FOR FAILURES

REQUEST STAGES
DNS
→ TCP
→ TLS
→ WRITE
→ SERVER PROCESSING
→ READ

FIRST RULE
classify before retrying

RETRY
same request
nothing changed

NOT RETRY
token refresh
failover
codec switch
CDN switch
fallback

WHEN?
transient
+
operation safe/idempotent

IDEMPOTENT
GET
PUT
DELETE

POST
not inherently idempotent
→ use idempotency key

RETRY MECHANICS
max attempts
+
exponential backoff
+
full jitter

Retry-After
→ honor server instruction

CIRCUIT BREAKER
CLOSED = normal
OPEN = fail fast
HALF-OPEN = one controlled probe

VIDEO
different requests
different policies
network failure ≠ DRM/codec failure

404
normally stop
BUT live-edge segment can temporarily 404

SYSTEM SURVIVAL
failover
+
feature flags
+
graceful degradation
+
stale-if-error

RECOVERY DANGER
service comes back
→ everyone retries together
→ second outage

FIX
jitter
+
staggered probes
+
slow start

FINAL FLOW
CLASSIFY
→ STRUCTURAL?
→ TRANSIENT RETRY
→ CIRCUIT BREAK
→ FAILOVER / DEGRADE / STALE
```

---

# 120. The Five Rules to Actually Remember

If you forget everything else, remember these:

### Rule 1
**Never retry before identifying where the request failed.**

### Rule 2
**A retry means the exact same request again.**

### Rule 3
**Retry only when waiting could plausibly change the outcome.**

### Rule 4
**Retry with bounded exponential backoff + full jitter, and honor Retry-After.**

### Rule 5
**When a dependency is known to be unhealthy, stop calling it: circuit break, fail over, degrade, or serve stale.**

---

# 121. Final Mental Model

```text
                 FAILURE
                    │
                    ▼
          ┌───────────────────┐
          │ WHERE DID IT FAIL?│
          └─────────┬─────────┘
                    │
                    ▼
            STRUCTURAL OR
             TRANSIENT?
              /         \
             /           \
       STRUCTURAL       TRANSIENT
          │                 │
          ▼                 ▼
      STOP / HANDLE      IDEMPOTENT?
                            /    \
                           /      \
                         YES       NO
                          │         │
                          │         ▼
                          │     IDEMPOTENCY
                          │        KEY /
                          │      SAFE FLOW
                          │
                          ▼
                  Retry-After?
                    /       \
                  YES        NO
                   │          │
                   ▼          ▼
                HONOR      BACKOFF
                             +
                           JITTER
                             +
                           CAP
                             │
                             ▼
                       STILL FAILING?
                             │
                             ▼
                     CIRCUIT BREAKER
                             │
                             ▼
                  FAILOVER / DEGRADE
                             │
                             ▼
                       SERVE STALE
                    WHEN APPROPRIATE
```

The central idea is simple:

> **A resilient system does not merely retry failures. It classifies them, retries only when another attempt can help, controls the retry load, and has a plan for when the dependency remains unavailable.**

That is the whole point of Session 7.
