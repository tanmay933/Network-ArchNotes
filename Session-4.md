# Computer Networks — Session 4 Notes
## nginx — and the code that explains it

> **Exam / MCQ study notes based on the Session 4 teaching material.**
>
> Focus: understand the teacher's reasoning, architecture, exact comparisons, important numbers, nginx configuration behaviour, source-code ideas, and common MCQ traps.

---

# 1. Session Overview

## Main theme

Session 4 is about **nginx and the architecture that makes it scale**.

The session follows four blocks:

1. **History, compressed**
   - Early web servers
   - Apache and CGI
   - Apache's prefork → worker → event evolution
   - Java/J2EE/servlets
   - C10K and the constraints around 2002

2. **What nginx changed**
   - Event loop
   - `epoll`
   - Master + workers
   - Reverse proxy
   - `sendfile`
   - Handling thousands of mostly-idle connections

3. **The features you configure**
   - Caching
   - LRU
   - Cache locking
   - Stale content
   - `location` precedence
   - Prefixes vs regexes
   - Upstreams and weighted round robin
   - `try_files` and fallbacks

4. **The source-code walkthrough**
   - nginx source tree
   - event loop
   - accept handling
   - stale epoll events
   - HTTP parser
   - HTTP phases
   - FastCGI records
   - memory pools / queues
   - ten important source-code stops

---

# 2. The Historical Lineage

## Important correction

The session explicitly corrects a common story.

### Correct lineage

```text
1990  CERN httpd
      ↓
1993  NCSA Mosaic       ← browser
      ↓
1993  NCSA HTTPd        ← server, invented CGI
      ↓
1994  McCool leaves
      ↓
1995  Apache
      ↓
1996  Apache becomes #1
```

### Important distinction

**Mosaic was a browser, not the first web server.**

- Berners-Lee's first web server was **CERN httpd** on a NeXT in 1990.
- NCSA Mosaic was a browser.
- NCSA HTTPd was a server and is associated in the notes with CGI.
- Apache descended from NCSA HTTPd.

### MCQ trap

If asked:

> "What was Berners-Lee's first web server?"

Answer:

**CERN httpd**

Not Mosaic.

---

# 3. Apache's Evolution

Apache spent years trying to make the per-connection worker cheaper.

## Apache 1.3 — prefork

```text
pool of processes
       ↓
one process handles a connection
```

The key improvement over fork-per-request was:

> **Fork the processes at startup instead of creating one for every request.**

So expensive process creation is moved off the request path.

But each process can still remain tied to a connection.

---

## Apache 2.0 — worker MPM

Processes run multiple threads.

```text
process
 ├── thread
 ├── thread
 ├── thread
 └── thread
```

This reduces the cost compared with one process per connection.

But a thread still consumes a stack and can remain tied to a connection.

The notes use **8 MB stack** as the default Linux example.

---

## Apache 2.4 — event MPM

The event model improves the keep-alive problem.

An event/listener mechanism parks idle keep-alive sockets on `epoll`.

A worker thread gets a connection when it has actual work.

### Key idea

```text
idle connection
      ↓
epoll waits
      ↓
activity happens
      ↓
worker thread gets the work
```

This prevents an idle keep-alive connection from unnecessarily owning a worker thread.

---

# 4. Java / J2EE's Answer

The same fundamental idea appeared in Java.

## CGI

Per request:

```text
fork()
exec()
load interpreter
parse script
run
exit
```

## Servlet

At startup:

```text
load class
instantiate once
```

Per request:

```text
service(req, res)
```

The object survives between requests.

### Same architectural move

The session emphasizes:

> Take expensive setup off the request path and pay for it once.

This is the same idea behind:

- Apache prefork
- Pre-threading
- FastCGI
- Connection pools
- Servlets

---

# 5. The 2002 Constraint

The session highlights three numbers:

```text
10,000
8 MB
1024
```

## 10,000

The **C10K** problem:

> Can one machine hold around 10,000 concurrent connections?

## 8 MB

Example default Linux thread stack reservation.

Therefore:

```text
10,000 threads × 8 MB
≈ 80 GB
```

of address space.

## 1024

`FD_SETSIZE` in the `select()` implementation discussed by the teacher.

The notes emphasize that increasing `ulimit` does **not** change this compile-time bitmap width.

### Important MCQ distinction

```text
ulimit -n
```

and

```text
FD_SETSIZE
```

are not the same thing.

For the `select()` limit discussed here:

> `FD_SETSIZE = 1024` is a compile-time ceiling.

---

# 6. Why nginx Was Created

## Igor Sysoev

The notes describe Igor Sysoev at Rambler starting nginx in **2002** specifically to address C10K.

The first public release was:

```text
4 October 2004
```

The notes emphasize that nginx was **not simply "a faster Apache."**

The architectural question was different.

### Apache asked

> How do I make the worker cheaper?

### nginx asked

> Why is there a worker per connection at all?

---

# 7. The nginx Insight

A connection does not need an entire thread/process while it is idle.

It needs:

- A small amount of state.
- A place in a list.
- CPU only when it has work.

So the model becomes:

```text
many connections
      ↓
event loop
      ↓
only active events receive CPU
```

The session summarizes the model as:

```text
One process per core.
One event loop each.
Thousands of connections.
No connection owns a dedicated worker.
```

---

# 8. nginx Master + Workers

The architecture is approximately:

```text
                  nginx master
                 /     |      \
                /      |       \
           worker 0  worker 1  worker N
              ↓         ↓         ↓
           event      event      event
            loop       loop       loop
```

## Master process

The master:

- Binds the listening port.
- Reads configuration.
- Forks workers.
- Handles signals.

The master **does not handle normal client connections**.

## Workers

Workers:

- Run event loops.
- Handle client connections.
- Perform the request processing.

The session also mentions cache manager/cache loader processes.

### MCQ trap

The nginx master process is **not** the process serving every request.

---

# 9. nginx Process Count

The session shows:

```bash
ps -o pid,ppid,args -C nginx
```

The number of worker processes does not continually increase with traffic.

This is a major contrast with:

```text
fork-per-request
```

The workers are created as a pool.

### Core idea

```text
traffic ↑
does NOT mean
process count ↑ for every request
```

---

# 10. nginx's Event Loop

The session presents a compact version of nginx's event loop:

```c
timer = ngx_event_find_timer();
ngx_process_events(cycle, timer, flags);   /* epoll_wait() */

ngx_event_process_posted(cycle, &ngx_posted_accept_events);
ngx_shmtx_unlock(&ngx_accept_mutex);
ngx_event_expire_timers();
ngx_event_process_posted(cycle, &ngx_posted_events);
```

The most important line conceptually is:

```c
ngx_process_events(...)
```

which corresponds to waiting for events such as `epoll_wait()`.

## Important idea

The worker spends most of its life **asleep waiting for work**.

The timer is supplied as an argument to the wait.

The session explicitly says:

> nginx does not separately poll for timeouts.

---

# 11. Posted Events

Handlers do not execute inside the polling operation itself.

Instead:

```text
event becomes ready
      ↓
event gets posted
      ↓
event loop drains posted events
      ↓
handler runs
```

This separation is important when understanding nginx's source.

---

# 12. Worker Load Balancing

The session shows:

```c
ngx_accept_disabled =
    ngx_cycle->connection_n / 8
    - ngx_cycle->free_connection_n;
```

The idea is:

> Once a worker is using more than roughly seven-eighths of its connection slots, it should stop competing for new connections and allow other workers to take them.

### Why?

To avoid overloading one worker while other workers have capacity.

---

# 13. `accept_mutex`

Historically, workers needed a userspace mechanism to coordinate accepting connections.

The notes discuss:

```text
accept_mutex
```

But modern Linux provides kernel mechanisms such as:

- `EPOLLEXCLUSIVE`
- `SO_REUSEPORT`

The notes say `accept_mutex` is **off by default since nginx 1.11.3**.

### MCQ trap

Do not assume `accept_mutex` is the modern required solution.

The kernel can now help distribute connection acceptance.

---

# 14. `select()` vs `epoll()` — nginx's View

The teacher gives an especially important comparison:

> **select is O(watched). epoll is O(active).**

## `select()`

The watch list is in the application.

It is rebuilt on each call.

The work includes:

```text
build
+
kernel scan
+
application scan
```

The session summarizes this as:

```text
O(n) + O(n) + O(n)
```

for the different pieces of the work.

## `epoll`

The watch list lives in the kernel.

Descriptors are registered with:

```text
epoll_ctl()
```

and readiness is obtained with:

```text
epoll_wait()
```

The useful work is proportional to the number of ready events.

---

# 15. The 10,000 Idle / 10 Active Example

Suppose:

```text
10,000 watched connections
10 active
```

The teacher's simplified comparison is:

### `select`

```text
~30,000 units of work
```

because the set is rebuilt/scanned in multiple stages.

### `epoll`

```text
10
```

because only the ready events need to be returned/handled.

### Key question

The fundamental architectural question is:

> **Who owns the watch list?**

- `select()` → application.
- `epoll` → kernel.

---

# 16. `select()` Descriptor Ceiling

For the `select()` model:

```text
FD_SETSIZE = 1024
```

The limit is compiled into the fixed-width bitmap.

Therefore:

```bash
ulimit -n 65536
```

does **not** make `select()` capable of representing arbitrary descriptors beyond its bitmap width.

### MCQ trap

If a question says:

> "Increase `ulimit -n` to 65536 to remove the `select()` FD_SETSIZE limit."

Answer:

**False.**

`ulimit` does not change the compile-time bitmap width.

---

# 17. Stale `epoll` Events

The session discusses a subtle event-loop bug.

`epoll_wait()` returns a **batch** of events.

Suppose:

```text
event 1
event 7
```

are returned.

While processing event 1, the program closes/reuses the file descriptor that event 7 referred to.

Now event 7 may refer to a **new client** using the recycled fd.

That is a stale event problem.

---

# 18. nginx's Generation Flag

nginx uses a clever technique.

It stores an instance/generation value associated with the connection.

The low bit of a pointer can be borrowed because aligned structs have that bit available.

The code effectively checks whether the event belongs to the current connection instance.

If it is stale:

```text
skip it
```

### Why this is clever

- No extra large structure needed.
- Reuses an otherwise unused bit.
- One branch handles stale events.

### MCQ idea

The purpose of the instance/generation check is:

> **Prevent processing an event belonging to an old/recycled file descriptor.**

---

# 19. `sendfile()`

The session presents `sendfile()` as another major nginx optimization.

## Traditional `read()` + `write()`

Conceptually:

```text
disk
 ↓
kernel page cache
 ↓
user-space buffer
 ↓
kernel socket buffer
 ↓
NIC
```

This involves more copying and user/kernel transitions.

## `sendfile()`

Conceptually:

```text
disk
 ↓
kernel page cache
 ↓
socket
 ↓
NIC
```

The application does not touch the bytes.

---

# 20. Why `sendfile()` Helps

The session says:

```text
read() + write()
→ 4 copies
→ 2 mode switches per chunk
```

versus:

```text
sendfile()
→ 2 copies
→ 0 mode switches per chunk
```

as presented in the teaching model.

The important conceptual point is:

> **The data never enters the application process.**

---

# 21. When `sendfile()` Cannot Be Used

If the application needs to **touch/modify the bytes**, they need to enter user space.

Examples from the session's reasoning:

- gzip
- TLS processing
- templating

Therefore the notes point out the tension between:

```text
gzip on
```

and:

```text
sendfile on
```

because compression needs access to the data.

The session also mentions **kTLS** as a mechanism relevant to this problem.

---

# 22. `sendfile()` Benchmark Trap

The teacher gives an important benchmarking lesson.

A loopback test showed approximately:

```text
read     → 37.4 ms, 2048 syscalls
mmap     → 23.3 ms, 4 syscalls
sendfile → 33.2 ms, 1 syscall
```

The key point is **not** that `sendfile()` always produces the fastest single-transfer wall-clock time.

On loopback:

- There is no physical NIC bottleneck.
- DMA behaviour differs.
- Memory bandwidth can dominate.
- curl competes for CPU.

### What is definitely real?

```text
2048 syscalls → 1 syscall
64 MB never enters the application's address space
```

The benefit becomes especially valuable as concurrency increases because saved CPU can be used for other requests.

### MCQ trap

Do not conclude:

> "`sendfile()` always makes one file transfer dramatically faster."

The teacher's point is that it reduces copying/syscall overhead and creates **headroom under concurrency**.

---

# 23. Reverse Proxy

## Forward proxy

A forward proxy sits near the **clients**.

Example:

```text
client → proxy → internet
```

The client knows/uses the proxy.

The session uses Squid as the example.

## Reverse proxy

A reverse proxy sits near the **servers**.

```text
clients → nginx → application servers
```

The client usually does not know it is there.

---

# 24. What nginx Does as a Reverse Proxy

The session highlights several capabilities:

- Cache responses.
- Route by path.
- Route by host.
- Distribute requests across upstreams.
- Weight upstream servers.
- Use least-connections.
- Fall back when an upstream fails.
- Terminate TLS.
- Absorb slow clients.

### Important advantage

nginx can hold many slow client connections while the application servers deal with actual application work.

So:

```text
slow phone / client
      ↓
    nginx
      ↓
fast application connection
```

The application server does not need to directly deal with the slow client.

---

# 25. nginx Cache

Example configuration:

```nginx
proxy_cache_path temp/cache
    levels=1:2
    keys_zone=demo:10m
    max_size=100m
    inactive=60s
    use_temp_path=off;
```

There are two important structures:

```text
shared memory index
+
cache data on disk
```

---

# 26. `keys_zone`

Example:

```nginx
keys_zone=demo:10m
```

The notes explain that this is:

> **10 MB of shared memory holding the cache index structures, not the cached data itself.**

It contains structures such as:

- rbtree
- LRU queue

The teacher gives a rough rule:

```text
~8,000 keys per MB
```

---

# 27. `levels=1:2`

```nginx
levels=1:2
```

creates two directory levels on disk.

Purpose:

> Avoid putting a huge number of files into a single directory.

This improves filesystem lookup/manageability.

---

# 28. `max_size`

```nginx
max_size=100m
```

sets the disk cache size ceiling.

The cache manager evicts entries from the LRU tail to stay under the configured size.

---

# 29. `inactive`

```nginx
inactive=60s
```

means an object can be evicted if it has not been **read/accessed** for 60 seconds, regardless of its TTL.

### Important distinction

`inactive` is **not the same thing** as:

```nginx
proxy_cache_valid
```

The session explicitly warns students not to confuse them.

---

# 30. `proxy_cache_valid` vs `inactive`

Think:

### `proxy_cache_valid`

Controls how long the cached response is considered **valid/fresh**.

### `inactive`

Controls eviction based on **lack of use/read activity**.

These are different mechanisms.

### MCQ trap

If:

```text
proxy_cache_valid = 10s
inactive = 60s
```

do not simply say "60 seconds means the object is valid for 60 seconds."

They control different things.

---

# 31. nginx Cache LRU

The cache uses an LRU-style queue.

Conceptually:

```text
tail
 ↓
oldest candidate
 ↓
check expiry
 ↓
check whether object is being used
 ↓
delete if safe
```

The session shows logic similar to:

```c
q = ngx_queue_last(&cache->sh->queue);
fcn = ngx_queue_data(q, ...);

if (fcn->count == 0) {
    ngx_http_file_cache_delete(...);
}
```

### Important condition

An object currently being streamed to a client must not be evicted from under that client.

Therefore:

```text
count == 0
```

is important.

---

# 32. `proxy_cache_lock`

Configuration:

```nginx
proxy_cache_lock on;
```

Purpose:

> Prevent a cache stampede.

### Without cache locking

Suppose a popular object expires.

Many requests arrive simultaneously.

All see:

```text
MISS
```

and all hit the origin.

### With cache locking

```text
first request
     ↓
origin
     ↓
refresh cache

other requests
     ↓
wait
     ↓
receive refreshed object
```

### MCQ phrase

**`proxy_cache_lock` = anti-stampede mechanism.**

The notes say it is **off by default**.

---

# 33. `proxy_cache_use_stale`

Example:

```nginx
proxy_cache_use_stale
    error timeout updating
    http_500 http_502 http_503 http_504;
```

Purpose:

> Serve an existing stale cached response when the origin is unhealthy or certain failure conditions occur.

### Design trade-off

Would the user prefer:

```text
slightly old page
```

or:

```text
502 error
```

Often the stale page is better.

---

# 34. `updating`

The notes also mention:

```text
updating
```

The idea is:

```text
one request refreshes
+
others continue receiving stale content
```

This avoids making every request wait for the refresh.

---

# 35. Cache Debugging

The session recommends:

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

This exposes whether the request was a cache:

```text
MISS
HIT
```

etc.

Example:

```bash
curl -i localhost:8082/ | grep X-Cache
```

### Practical MCQ

If asked what `$upstream_cache_status` helps expose:

> The cache status of the upstream response.

---

# 36. nginx `location` Precedence

This is one of the most important MCQ/configuration topics.

Consider:

```nginx
location = /exact { ... }

location ^~ /images/ { ... }

location ~ \.php$ { ... }

location / { ... }
```

The order in which these are written is **not** simply the matching order.

---

# 37. Exact Match

```nginx
location = /exact
```

Exact match wins immediately.

Example:

```text
/exact
```

→ exact location.

Once it matches, the search stops.

---

# 38. `^~` Prefix

```nginx
location ^~ /images/
```

This is a prefix match.

It is also special because:

> If it is the selected longest prefix, it suppresses regex searching.

Example:

```text
/images/a.php
```

matches the `/images/` prefix.

Because of `^~`, nginx does not continue to regex matching.

---

# 39. Regex Location

```nginx
location ~ \.php$
```

Regex locations are tested in **file order**.

They do **not** choose the longest regex.

The first matching regex wins.

### MCQ trap

Regex matching is:

```text
FIRST matching regex
```

not:

```text
longest regex
```

---

# 40. Generic Prefix

```nginx
location /
```

This is the fallback/general prefix.

The session summarizes it as:

> Longest prefix, used only if no regex matched.

---

# 41. Location Precedence — Memorize

For the specific forms taught:

```text
1. Exact =
2. Longest prefix
3. ^~ prefix can suppress regex search
4. Regex ~ / ~* — first matching regex in file order
5. General prefix if no regex wins
```

A practical mental model:

```text
Exact?
  ↓ no
Find longest prefix
  ↓
Was it ^~?
  ↓ yes → use it
  ↓ no
Check regexes in file order
  ↓
First regex match wins
  ↓ none
Use longest prefix
```

---

# 42. Example

Given:

```nginx
location = /exact { ... }
location ^~ /images/ { ... }
location ~ \.php$ { ... }
location / { ... }
```

### Request

```text
/exact
```

→ **1**

### Request

```text
/images/a.php
```

→ **2**

The regex might match `.php`, but `^~` suppresses the regex search.

### Request

```text
/other/a.php
```

→ **3**

The regex matches.

### Request

```text
/anything
```

→ **4**

No regex match, so the general prefix is used.

---

# 43. Why Prefixes Are Cheap and Regexes Are More Expensive

The session connects the rules to the data structures.

## Prefixes

Stored in a tree.

The tree can be walked efficiently according to URI segments.

## Regexes

Stored as an array.

They are walked in order for each request.

Therefore:

```text
prefix → tree
regex  → array
```

### Core lesson

> **The data structure explains the matching rule.**

Prefixes naturally support longest-match lookup.

Regexes naturally follow array order.

---

# 44. nginx Upstreams

nginx can distribute requests across upstream application servers.

Example conceptual setup:

```text
             nginx
            /     \
        server A  server B
```

Upstream selection can use strategies such as:

- Weighted round robin
- Least connections

---

# 45. Smooth Weighted Round Robin

The session shows nginx's smooth weighted round-robin implementation.

Suppose:

```text
A : B = 3 : 1
```

### Naive round robin

Could produce bursts such as:

```text
A A A B
```

### Smooth weighted round robin

Spreads selections more smoothly:

```text
A B A A
```

while still maintaining the same overall ratio.

The example in the notes sends 24 requests and observes:

```text
6 → server with weight 1
18 → server with weight 3
```

which is exactly:

```text
3 : 1
```

### Why smooth?

It avoids sending a burst of traffic to one backend just because of the weighting pattern.

---

# 46. `try_files`

A common nginx pattern:

```nginx
location / {
    try_files $uri $uri/ /index.html?$args;
}
```

Conceptually:

```text
Is this a real file?
        ↓ no
Is this a real directory?
        ↓ no
Use fallback /index.html
```

This is widely useful for:

- SPAs
- WordPress-style routing
- Rails/Django applications
- Pretty URLs

---

# 47. Network Fallback with `try_files`

The session also shows:

```nginx
location /maybe/ {
    try_files $uri @app;
}

location @app {
    proxy_pass http://app;
}
```

Here:

```text
try local file
     ↓ not found
named location
     ↓
application upstream
```

---

# 48. Named Locations

Example:

```nginx
location @app {
    proxy_pass http://app;
}
```

A named location is:

> Reachable through nginx internal control flow such as `try_files` or `error_page`, not directly as a URL.

### MCQ trap

You cannot request:

```text
GET /@app
```

to directly enter:

```nginx
location @app
```

It is a named internal target.

---

# 49. The Last `try_files` Argument

In:

```nginx
try_files $uri $uri/ /index.html?$args;
```

the final argument is the fallback.

It is **not tested as a file candidate** in the same way as the preceding arguments.

### Think:

```text
candidate
candidate
fallback
```

---

# 50. nginx Source Tree

The session gives a useful source-code map.

## `src/core/`

nginx's core library:

- string
- array
- list
- hash
- rbtree
- queue
- memory pool
- buffer
- configuration
- cycle
- connection

nginx has its own core abstractions.

---

## `src/event/`

The event loop.

Contains modules for:

- epoll
- kqueue
- select
- poll
- devpoll
- eventport

They share a common interface.

---

## `src/http/`

HTTP implementation:

- HTTP state machine
- phase engine
- upstream framework
- HTTP modules/directives

---

## `src/os/unix/`

Operating-system/syscall layer:

- `sendfile`
- process spawning
- shared memory
- atomics
- OS-specific operations

---

## `src/stream/`

The same broad architecture for:

- raw TCP
- UDP

This allows nginx to act as an **L4 load balancer**.

---

## `src/mail/`

Protocol proxying for:

- IMAP
- POP3
- SMTP

### Big architecture lesson

```text
same core
+
different protocol modules
```

---

# 51. How to Navigate nginx Source

The session says line numbers drift.

Function names are much more stable.

Example:

```bash
grep -rn "^ngx_epoll_process_events" src/
```

This searches for the function definition.

### Important habit

Navigate source by:

```text
function names
+
directory structure
```

rather than relying forever on exact line numbers.

The PDF's line numbers are pinned to nginx **1.31.5** and may change in later versions.

---

# 52. Two Core Files to Understand First

## `src/core/ngx_queue.h`

An intrusive doubly linked list.

The link pointers live inside the node.

This allows list insertion without allocating a separate list-node object.

It is used in structures such as:

- Cache LRU
- Posted-event queues

---

## `src/core/ngx_palloc.h`

The nginx memory pool.

The session's key idea:

> nginx does not free individual request allocations one-by-one on the hot path.

Instead:

```text
request
   ↓
allocate from request pool
   ↓
request finishes
   ↓
destroy entire pool
```

### Benefit

Less individual `free()` work and fewer cleanup branches.

---

# 53. nginx Memory Pool

Traditional approach:

```text
malloc
malloc
malloc
free
free
free
```

nginx-style pool idea:

```text
request pool
 ├── allocation
 ├── allocation
 ├── allocation
 └── allocation

request ends
     ↓
destroy pool
```

### MCQ concept

The memory pool is useful because request-scoped allocations can be cleaned up collectively.

---

# 54. Ten Important Source-Code Stops

The session gives this roadmap:

| Concept | File | Function |
|---|---|---|
| Process model | `os/unix/ngx_process_cycle.c` | `ngx_start_worker_processes` |
| Event loop | `event/ngx_event.c` | `ngx_process_events_and_timers` |
| Accept + herd | `event/ngx_event_accept.c` | `ngx_event_accept` |
| epoll + stale events | `event/modules/ngx_epoll_module.c` | `ngx_epoll_process_events` |
| HTTP parser | `http/ngx_http_parse.c` | `ngx_http_parse_request_line` |
| Phases | `http/ngx_http_core_module.c` | `ngx_http_core_run_phases` |
| Routing | `http/ngx_http_core_module.c` | `ngx_http_core_find_static_location` |
| sendfile | `os/unix/ngx_linux_sendfile_chain.c` | `ngx_linux_sendfile` |
| Cache LRU | `http/ngx_http_file_cache.c` | `ngx_http_file_cache_expire` |
| FastCGI records | `http/modules/ngx_http_fastcgi_module.c` | `ngx_http_fastcgi_process_record` |

---

# 55. nginx HTTP Parser

The parser is not simply:

```text
parse entire string
```

It is a **resumable state machine**.

The notes mention:

```text
27 states
```

for the request line alone.

The parser does not rely on:

- regex
- `strtok`
- substring allocation

Instead it processes bytes according to the current state.

---

# 56. Why the HTTP Parser Must Be Resumable

A request line may arrive in multiple TCP segments.

Example:

```text
segment 1 → "GET /us"
segment 2 → "ers"
segment 3 → " HTTP/1.1\r\n"
```

The parser may be interrupted after any byte.

Therefore nginx stores parser state:

```text
r->state
```

When more bytes arrive:

```text
resume from previous state
```

### Core event-driven idea

> Non-blocking code must be able to stop and resume at arbitrary points.

---

# 57. HTTP Phase Engine

nginx processes requests through phases.

The session lists:

```text
POST_READ
→ SERVER_REWRITE
→ FIND_CONFIG
→ REWRITE
→ POST_REWRITE
→ PREACCESS
→ ACCESS
→ POST_ACCESS
→ PRECONTENT
→ CONTENT
→ LOG
```

There are **11 phases** in the presentation.

---

# 58. `return` Means Suspended

The phase engine looks conceptually like:

```c
while (ph[r->phase_handler].checker) {
    rc = ph[r->phase_handler].checker(
        r,
        &ph[r->phase_handler]
    );

    if (rc == NGX_OK) {
        return;
    }
}
```

The important interpretation:

> In this event-driven flow, `return` can mean **the request is suspended/not done**, not necessarily that the entire request has completed.

The request can resume later when another event fires.

---

# 59. Examples of Modules in Phases

The session gives these examples:

```text
limit_req
→ PREACCESS

auth_basic
→ ACCESS

try_files
→ PRECONTENT

proxy_pass
fastcgi_pass
static content
→ CONTENT
```

### MCQ idea

If asked why a directive "did not fire" when expected, think:

> Which HTTP phase does the module run in?

---

# 60. FastCGI in nginx

The session revisits FastCGI at the source level.

A FastCGI record contains:

```text
version
type
requestId
contentLength
paddingLength
reserved
```

Then:

```text
8-byte record header
+
contentData[contentLength]
+
padding
```

---

# 61. FastCGI Framing

The key idea:

```text
length inside every record header
+
empty record between streams
```

So:

### Length

The header says how many bytes of content follow.

### Empty record

Signals:

```text
end of stream
```

For example:

```text
empty PARAMS
→ no more environment variables

empty STDIN
→ no more request body
```

### MCQ trap

If the empty PARAMS record is missing, the receiver cannot know that the parameter stream has ended.

---

# 62. The Session's Implementation Ladder

The course repository builds the architecture step-by-step:

```text
01-fork
→ fork() per client

02-thread
→ pthread per client

03-select
→ one thread + select()
→ 1024 wall

04-epoll
→ one thread + epoll()

05-sendfile
→ read vs mmap vs sendfile

06-fastcgi
→ FastCGI wire protocol

07-nginx
→ real nginx configuration demos
```

This is important because each step fixes a limitation of the previous one.

---

# 63. What Each Demo Demonstrates

## `01-fork`

```text
fork() per client
```

Shows process-per-connection cost.

## `02-thread`

```text
pthread per client
```

Shows thread-based concurrency and stack cost.

## `03-select`

Shows the:

```text
1024
```

descriptor wall.

## `04-epoll`

Shows many connections without the same `select()` scanning/FD ceiling.

## `05-sendfile`

Compares:

```text
read
mmap
sendfile
```

and counts syscalls/copies.

## `06-fastcgi`

Shows the FastCGI protocol at the wire level.

## `07-nginx`

Combines:

- routing
- caching
- upstream weights
- fallbacks

into a real nginx configuration.

---

# 64. Key Numbers to Memorize

```text
1990
→ CERN httpd

1993
→ NCSA Mosaic / NCSA HTTPd

1995
→ Apache

1999
→ C10K problem named

2002
→ nginx development begins

4 Oct 2004
→ first public nginx release

2011
→ nginx 1.0.0

8 MB
→ example/default Linux thread stack in the session

10,000
→ C10K

1024
→ FD_SETSIZE / select bitmap width discussed

1.11.3
→ accept_mutex default off

Linux 4.5+
→ EPOLLEXCLUSIVE mentioned

nginx 1.31.5
→ version used by this session/source walkthrough
```

---

# 65. High-Yield MCQ Traps

## Trap 1

**Mosaic was the first web server.**

❌ False.

Mosaic was a browser.

---

## Trap 2

**Apache prefork creates a new process for every request.**

❌ Not in the model being taught.

Apache prefork creates a **pool of processes at startup**.

The process can still be associated with a connection.

---

## Trap 3

**nginx has one worker per connection.**

❌ False.

The key nginx idea is the opposite:

> workers run event loops handling many connections.

---

## Trap 4

**The nginx master handles client requests.**

❌ False.

The master manages:

- binding
- configuration
- workers
- signals

Workers handle normal connections.

---

## Trap 5

**`ulimit -n 65536` removes `FD_SETSIZE = 1024`.**

❌ False.

`FD_SETSIZE` is a compile-time bitmap limitation for `select()`.

---

## Trap 6

**`select()` is O(active connections).**

❌ False in the teacher's model.

```text
select → O(watched)
epoll  → O(active/ready)
```

---

## Trap 7

**nginx's event loop constantly polls for timeouts.**

❌ False.

The timer is supplied to the event wait.

---

## Trap 8

**Regex location chooses the longest matching regex.**

❌ False.

Regex locations are checked in **file order**; the first matching regex wins.

---

## Trap 9

**`^~` means regex has higher priority.**

❌ False.

A selected `^~` prefix suppresses regex searching.

---

## Trap 10

**`proxy_cache_lock` is mainly for making cache entries live longer.**

❌ False.

It is an **anti-cache-stampede** mechanism.

---

## Trap 11

**`inactive=60s` means the cache object is valid for exactly 60 seconds.**

❌ False.

`inactive` is about lack of access/read activity and eviction.

`proxy_cache_valid` controls freshness/validity.

---

## Trap 12

**`sendfile()` always makes a single loopback transfer faster.**

❌ False.

The session's benchmark specifically demonstrates that wall-clock improvement is not guaranteed on loopback.

The major benefit is reduced copying/syscall overhead and more CPU headroom under concurrency.

---

## Trap 13

**A named nginx location can be requested directly by URL.**

❌ False.

Named locations such as:

```nginx
location @app
```

are internal targets reached by directives such as `try_files` or `error_page`.

---

## Trap 14

**The final `try_files` argument is another file candidate.**

❌ False.

The final argument is the fallback.

---

# 66. Configuration MCQ Practice

## Q1

Given:

```nginx
location = /exact { }
location ^~ /images/ { }
location ~ \.php$ { }
location / { }
```

What handles:

```text
/images/a.php
```

### Answer

```text
location ^~ /images/
```

Because the selected `^~` prefix suppresses regex matching.

---

## Q2

What handles:

```text
/other/a.php
```

### Answer

```text
location ~ \.php$
```

because the regex matches and no `^~` prefix suppresses regex evaluation.

---

## Q3

What handles:

```text
/anything
```

### Answer

```text
location /
```

because no regex matches.

---

## Q4

What is the purpose of:

```nginx
proxy_cache_lock on;
```

### Answer

Prevent many simultaneous cache misses from all hitting the origin.

---

## Q5

What is the purpose of:

```nginx
proxy_cache_use_stale error timeout updating;
```

### Answer

Allow nginx to serve stale cached content under specified failure/update conditions.

---

# 67. Architecture Questions

## Q6

Why did nginx choose an event loop instead of simply making threads cheaper?

### Answer

Because the deeper problem is that **idle connections do not need dedicated workers at all**.

---

## Q7

Why is a slow client dangerous for an application server?

### Answer

Without a reverse proxy/event-driven front end, the application server may have to hold resources while waiting on a slow client.

nginx can absorb those slow connections and keep application workers focused on useful work.

---

## Q8

Why is nginx's master/worker model scalable?

### Answer

A fixed pool of workers handles many connections through event loops instead of creating a new worker for every connection/request.

---

# 68. Source-Code Reasoning Questions

## Q9

Why does nginx's HTTP parser need a state variable?

### Answer

Because a request can arrive across multiple TCP segments and parsing can stop at any byte. The parser must resume from where it stopped.

---

## Q10

Why are nginx memory pools useful?

### Answer

Request allocations can be made from a request-scoped pool and the entire pool can be destroyed when the request finishes, avoiding many individual `free()` operations.

---

## Q11

Why does nginx check connection instance/generation when processing epoll events?

### Answer

Because a batched event can refer to a file descriptor that has since been closed and recycled for another connection.

---

# 69. Numerical / Reasoning Questions

## Q12 — Thread memory

If one thread reserves:

```text
8 MB
```

approximately how much address space is associated with:

```text
10,000 threads
```

Calculation:

```text
10,000 × 8 MB
= 80,000 MB
≈ 80 GB
```

### Answer

**~80 GB**

---

## Q13 — Little's Law

A service handles:

```text
5,000 requests/sec
```

with:

```text
200 ms average latency
```

Convert:

```text
200 ms = 0.2 sec
```

Then:

```text
concurrency
= 5,000 × 0.2
= 1,000
```

### Answer

**~1,000 requests/connections in flight**

---

## Q14 — Latency increase

A system processes:

```text
10,000 requests/sec
```

Latency changes from:

```text
100 ms → 500 ms
```

Initial:

```text
10,000 × 0.1 = 1,000
```

New:

```text
10,000 × 0.5 = 5,000
```

### Answer

Concurrency becomes **5× larger**.

---

# 70. One-Page Mental Model

```text
CGI
→ new process/request
→ expensive

Apache prefork
→ process pool
→ setup paid at startup
→ connection can still occupy process

Apache worker
→ threads
→ cheaper than processes
→ thread/stack still tied to connection

Apache event
→ idle keep-alives parked
→ worker only gets active work

nginx
→ asks why connection needs a worker
→ event loop
→ one worker handles many connections
→ master manages workers

select
→ watch list in application
→ rebuilt/scanned
→ O(watched)
→ FD_SETSIZE = 1024

epoll
→ watch list in kernel
→ epoll_ctl()
→ epoll_wait()
→ O(active/ready)

sendfile
→ file bytes don't enter user space
→ fewer copies/syscalls
→ useful for static files

reverse proxy
→ clients → nginx → app servers
→ absorbs slow clients
→ routing/cache/LB/TLS

cache
→ shared-memory index
→ disk data
→ LRU
→ cache_lock prevents stampede
→ stale content can survive origin failures

location
→ exact
→ longest prefix
→ ^~ can suppress regex
→ regex first match in file order
→ fallback prefix

upstream
→ weighted round robin
→ smooth weighting avoids bursts
→ least_conn alternative

try_files
→ test local file
→ test directory
→ final fallback

source
→ core
→ event
→ http
→ os/unix
→ stream
→ mail

HTTP parser
→ resumable state machine
→ request can arrive in multiple TCP segments

phases
→ 11 phases
→ modules execute in specific phases

FastCGI
→ length in record
→ empty record = stream delimiter
```

---

# 71. Final Four Ideas from the Teacher

## 1. Ask the other question

Apache spent years making the per-connection worker cheaper.

nginx asked:

> **Why is there a worker per connection at all?**

When everyone optimizes the same thing, sometimes the better opportunity is one abstraction level higher.

---

## 2. Check whether the constraint still exists

Thread-per-connection became unattractive partly because thread stacks/context switching were expensive.

Later technologies such as:

- goroutines
- virtual threads

changed those cost assumptions.

### Lesson

Do not confuse an old engineering constraint with a permanent law.

---

## 3. The data structure is the documentation

```text
prefixes → tree → longest-match behaviour

regexes → array → first-match behaviour
```

Understanding the data structure explains nginx's routing rules.

---

## 4. Amortise the setup

All of these follow the same pattern:

- Prefork
- Pre-thread
- Servlets
- Connection pools
- FastCGI
- Upstream keepalive

The shared idea:

> **Take expensive setup off the request path and pay for it once.**

---

# 72. Last-Minute Revision — 20 Facts

1. First web server → **CERN httpd**.
2. Mosaic → **browser**, not server.
3. NCSA HTTPd → server associated with CGI.
4. Apache → descended from NCSA HTTPd.
5. Apache evolution → **prefork → worker → event**.
6. CGI → expensive per-request process creation.
7. Servlet → object created once, requests handled by pooled threads.
8. C10K → **10,000 concurrent connections**.
9. nginx development started → **2002**.
10. nginx first public release → **4 Oct 2004**.
11. nginx's key question → **why have a worker per connection?**
12. nginx → master + worker processes + event loops.
13. `select` → **O(watched)**.
14. `epoll` → **O(active/ready)**.
15. `FD_SETSIZE` → **1024** in the session's `select()` discussion.
16. `ulimit -n` does not remove the `select()` bitmap ceiling.
17. `sendfile()` → avoid bringing file bytes into user space.
18. `proxy_cache_lock` → prevent cache stampede.
19. Regex locations → **first matching regex in file order**.
20. Final `try_files` argument → **fallback**.

---

# 73. Session 4 in One Sentence

> **nginx scales not by making a per-connection worker infinitely cheaper, but by eliminating the need for a dedicated worker for every idle connection and combining event-driven waiting, fixed workers, efficient data movement, reverse-proxying, caching, routing, and careful state management.**