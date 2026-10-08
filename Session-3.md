# Computer Networks — Session 3 Notes
## One Process, Many Clients — Scaling Servers

> **Exam focus:** These notes are optimized for MCQ-based revision. They preserve the terminology, examples, numbers, comparisons, code ideas, and framing used in the Session 3 teaching material.

---

# 1. Session Overview

### Main question
**How do servers scale? How does one box hold thousands of clients at the same time?**

### Five blocks

1. **Multiplexing** — `fork()`, threads, `select()`, `epoll`, event loops, timeouts and retries.
2. **CGI / FastCGI** — how a server runs application code.
3. **Web Servers?** — HTTP servers vs systems using other protocols such as Redis, MySQL, MQTT and Kafka.
4. **Limits & Solutions** — file descriptors, `ulimit`, C10K/C10M, memory, context switches, syscalls and queueing.
5. **In the Wild** — Apache, HAProxy, Varnish and nginx.

---

# 2. Multiplexing

## 2.1 Three ways to serve many clients

### A. Fork per request/client

A whole process is created for each client.

**Advantages**
- Simple model.
- Strong process isolation.

**Cost**
- Full process image.
- Page tables.
- File descriptors.
- Scheduler entry.
- Expensive per connection.

### B. Thread per request/client

Cheaper than a process, but still expensive at large scale.

Every thread still needs a **stack** before it performs useful work.

### C. One loop, many sockets

One thread watches many sockets using mechanisms such as:

- `select()`
- `epoll`

The loop wakes when sockets are ready.

**Key idea:** avoid dedicating a process/thread to every connection.

---

## 2.2 What production systems actually do

Production systems usually **combine** the approaches rather than choosing only one.

Typical structure:

```text
                Worker pool
        ┌──────────┼──────────┐
        ↓          ↓          ↓
     Worker 1   Worker 2   Worker 3
        ↓          ↓          ↓
      epoll      epoll      epoll
        ↓          ↓          ↓
     sockets    sockets    sockets
```

### Teacher's framing

- Pre-fork or pre-thread a worker pool.
- Each worker runs its own `select`/`epoll` loop.
- Processes provide isolation.
- The event loop provides concurrency.

`SO_REUSEPORT` can allow the kernel to spread new connections across workers.

---

# 3. `fork()`

## What happens?

Typical server flow:

```text
socket()
bind()
listen()
accept()
fork()
```

After `fork()`:

- Child handles the client.
- Parent continues accepting clients.
- Child eventually exits.

### Important costs

Each connection gets a process with:

- Process image
- Page tables
- File descriptors
- Scheduler state

This makes fork-per-connection expensive at high concurrency.

### MCQ trap

**Question:** Why is fork-per-client expensive?

**Answer:** Because each connection incurs process-related resource and scheduling overhead.

---

# 4. `select()`

## Basic idea

`select()` lets one thread watch many file descriptors.

Typical pattern:

```c
FD_ZERO(&readfds);
FD_SET(server_fd, &readfds);

for (...) {
    FD_SET(clients[i], &readfds);
}

select(maxfd + 1, &readfds, NULL, NULL, NULL);
```

Then check:

```c
FD_ISSET(fd, &readfds)
```

to see which descriptor is ready.

---

## 4.1 Important arguments

Conceptually:

```text
select(
    maxfd + 1,
    readfds,
    writefds,
    exceptfds,
    timeout
)
```

### `readfds`
Wait for data/readability.

### `writefds`
Wait until a socket can accept/write data.

### `exceptfds`
Exceptional conditions.

### Final argument = timeout

If:

```c
select(maxfd + 1, &readfds, NULL, NULL, NULL);
```

the timeout is `NULL`.

### Therefore:

**`NULL` timeout = block indefinitely.**

---

## 4.2 `select()` has an important limitation

The teacher emphasizes:

```text
FD_SETSIZE = 1024
```

`fd_set` is a fixed-width bitmap.

Therefore descriptor **1024 cannot be represented** in this setup.

### MCQ trap

Do NOT confuse:

- `FD_SETSIZE`
- application/server connection count
- `ulimit -n`

They are related in the teacher's discussion but are not literally the same mechanism.

For this session, **1024 is the number to remember for the `select()` ceiling/default-limit discussion.**

---

## 4.3 Why `select()` scales poorly

`select()` is presented as:

```text
O(n) per call
```

Why?

1. The application rebuilds the fd set on every iteration.
2. The kernel scans the descriptors.
3. Ready and idle descriptors are both part of the scan.

So:

```text
10,000 idle sockets
```

still cost scanning work similar to:

```text
10,000 busy sockets
```

---

# 5. `epoll`

## Core idea

Instead of repeatedly giving the kernel the entire set:

```text
register fd once
       ↓
epoll_wait()
       ↓
return ready fds
```

### Main operations

```text
epoll_ctl()
```

Registers/configures file descriptors.

```text
epoll_wait()
```

Returns descriptors/events that are actually ready.

---

## `select()` vs `epoll`

| Feature | `select()` | `epoll` |
|---|---|---|
| Registration | Rebuild set repeatedly | Register fd |
| Returned work | Scan/check all watched fds | Ready events |
| Teacher's complexity framing | O(n) | O(ready) |
| Large idle connection set | Expensive | Much better |
| Platform example | Portable classic API | Linux |
| Related mechanisms | — | kqueue, IOCP, io_uring |

### Related mechanisms from the notes

- `kqueue` — BSD/macOS
- `IOCP` — Windows
- `io_uring` — modern Linux

### MCQ trap

Do **not** describe `select()` as "stateless" and `epoll` as "stateful."

The important difference is:

> `select()` repeatedly supplies/scans the set; `epoll` keeps registrations and returns ready events.

---

# 6. Timeouts and Retries

A server can be:

- Reading
- Writing
- Accepting

A server should not wait forever.

### `select()` timeout

The timeout is the last argument.

```c
select(maxfd + 1, &readfds, NULL, NULL, NULL);
```

means:

```text
wait forever
```

### Client-side rule

The client should:

1. Set a deadline.
2. Retry if appropriate.
3. Use backoff.

### MCQ trap

A `NULL` timeout is **not** "no waiting."

It means **block indefinitely**.

---

# 7. CGI

## What is CGI?

CGI is a way for the web server to run application code.

The major characteristic:

> **Every request creates/runs a new copy of the program.**

Typical architecture:

```text
HTTP request
     ↓
Apache/httpd
     ↓
   CGI process
   ↙   ↓    ↘
env  stdin  stdout
          ↘
          stderr → error log
```

---

# 8. CGI's Three Pipes

The server wires up:

### `stdin`
Request data goes to the CGI program.

### `stdout`
CGI output goes back toward the client.

### `stderr`
Error/debug output goes to the server's error log.

### Memorize

```text
stdin  → request/body
stdout → response
stderr → error log
```

### MCQ trap

If asked:

**"Where does the CGI request body go?"**

Answer:

> **stdin**

Not `stdout`.

---

# 9. CGI Environment Variables

HTTP request information is translated into environment variables.

Examples:

```text
REQUEST_METHOD = GET / POST
QUERY_STRING = ...
SERVER_PROTOCOL = ...
GATEWAY_INTERFACE = ...
```

Example:

```text
User-Agent: curl
```

becomes:

```text
HTTP_USER_AGENT=curl
```

### Transformation rule

HTTP header:

```text
User-Agent
```

becomes:

```text
HTTP_USER_AGENT
```

Steps:

1. Uppercase.
2. Replace `-` with `_`.
3. Prefix with `HTTP_`.

---

# 10. CGI Response Framing

A CGI program can be extremely simple.

It reads its inputs and prints output.

Response structure:

```text
Content-Type: text/html

<html>...</html>
```

The **blank line** separates:

```text
headers
```

from:

```text
body
```

### MCQ trap

The blank line is a **delimiter/framing mechanism**.

---

# 11. Why CGI Is Expensive

CGI repeatedly pays the setup cost.

```text
request
   ↓
create process
   ↓
run program
   ↓
finish
   ↓
throw program/process away
```

Even if the actual application work is fast, process creation/execution happens repeatedly.

### Central scaling lesson

> **Move expensive setup off the request path and pay for it once at startup.**

This same idea appears in:

- Pre-forking
- Pre-threading
- FastCGI
- Connection pools
- Servlets

---

# 12. Alternatives to CGI

## `mod_php` / `mod_perl`

Application code can be compiled/loaded into the server.

### Advantage
No new process for every request.

### Problem
Application shares the server's address space.

A serious crash/segfault can affect the web server too.

---

# 13. FastCGI

## Core idea

Instead of creating a new application process per request:

> **Keep application processes alive and communicate with them over a socket.**

Typical architecture:

```text
Clients
   ↓ HTTP
nginx / httpd
   ↓ FastCGI
php-fpm
   ↓
long-lived worker pool
```

### Responsibilities

**nginx/httpd**
- Handles client connections.
- Handles slow clients.
- TLS.
- Buffering.

**php-fpm / application pool**
- Runs application code.
- Uses long-lived workers.

---

# 14. FastCGI vs CGI

| CGI | FastCGI |
|---|---|
| New process per request | Long-lived processes |
| Setup repeatedly | Setup amortized |
| Pipes between server/process | Unix/TCP socket |
| Text/env-style interface | Binary typed records |
| Process discarded | Worker reused |

### Most important conceptual difference

**CGI:**

```text
request → create program → execute → destroy
```

**FastCGI:**

```text
request → existing worker → execute → reuse worker
```

---

# 15. FastCGI Records

A FastCGI request is a **stream of typed records**.

Important record types:

```text
FCGI_BEGIN_REQUEST
FCGI_PARAMS
FCGI_STDIN
FCGI_STDOUT
FCGI_STDERR
FCGI_END_REQUEST
```

---

## Request side

### `FCGI_BEGIN_REQUEST`
Starts the request.

### `FCGI_PARAMS`
Carries information corresponding to CGI environment variables.

Examples:

```text
REQUEST_METHOD
QUERY_STRING
SCRIPT_FILENAME
CONTENT_TYPE
CONTENT_LENGTH
HTTP_HOST
HTTP_USER_AGENT
```

### Empty `FCGI_PARAMS`

Means:

> **Parameters are finished.**

It is a structural delimiter.

### `FCGI_STDIN`

Carries the request body.

For example:

```text
name=alice&age=20
```

### Empty `FCGI_STDIN`

Means:

> **The request body/input stream is finished.**

---

# 16. FastCGI Response

Response records include:

```text
FCGI_STDOUT
FCGI_STDERR
FCGI_END_REQUEST
```

### `FCGI_STDOUT`

Application response.

Example:

```text
Content-Type: text/html

<html>...</html>
```

### `FCGI_STDERR`

Debug/error output → log.

### `FCGI_END_REQUEST`

Marks completion of the FastCGI request.

---

# 17. FastCGI Framing

The session connects FastCGI to the previous session's framing concepts.

There are two ideas:

### 1. Length

A record contains a content length.

The receiver knows exactly how many bytes to consume.

Therefore arbitrary bytes can exist in the payload.

### 2. Delimiter

A zero-length/empty record indicates that the relevant stream is finished.

So FastCGI uses both concepts.

### Remember

```text
length → tells you how much data
empty record → tells you stream finished
```

---

# 18. Servlets

Servlets solve the same broad problem:

> How can user code run for requests without creating a completely new process every time?

### Jetty/Tomcat

They can act as the runtime/server.

### Pre-threading

A pool of threads is created at startup.

A request is handed to an available thread.

### Isolation

Servlets are isolated by:

- Class loader
- Thread

rather than a separate process.

### Trade-off

Cheaper than process-per-request, but a runaway servlet can still hurt neighboring work.

---

# 19. "Isn't Everything a Web Server?"

Not everything uses HTTP.

## HTTP-speaking examples from the notes

- Wi-Fi router admin page
- Modern agent/cloud tooling
- Most Node applications
- Firestore
- CouchDB
- Kafka when a REST proxy is placed in front

## Non-HTTP examples

- Redis → RESP
- MySQL → binary wire protocol
- MQTT brokers
- Kafka native protocol

### Key lesson

> **The protocol affects which scaling techniques are available.**

---

# 20. HTTP Statelessness

HTTP is described here as **stateless**.

A request carries the information needed to process that request.

Therefore:

```text
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
```

can work more easily.

### Horizontal scaling

You can add servers to a pool and allow a load balancer to distribute requests.

### Important MCQ trap

**Stateless HTTP does NOT mean an application cannot have sessions/state.**

It means the protocol itself does not require the server to maintain connection/session state in order to understand each request.

---

# 21. Database Persistent Connections

Databases such as Redis/MySQL prefer persistent connections because repeatedly creating a connection costs:

1. TCP connection setup.
2. Handshake.
3. Authentication.
4. Session setup.

Therefore keeping connections alive avoids paying that cost on every query.

### Scaling consequence

The problem changes from:

```text
How many requests/sec?
```

toward:

```text
How many persistent connections can one machine hold?
```

---

# 22. Four Walls / Scaling Limits

The notes identify four major limits.

## 1. File descriptors

A connection consumes a descriptor.

Descriptors are limited.

The notes use **1024** as the important default/limit number in this discussion.

## 2. Memory

Costs include:

- Thread stack
- Per-connection buffers
- Other per-connection state

## 3. Context switches / kernel boundary

Reads/writes and system calls cross the user/kernel boundary.

At very high operation rates, that overhead matters.

## 4. Queueing

Latency and throughput are connected.

When work becomes slower, more requests/connections remain in flight.

That consumes resources.

---

# 23. C10K and C10M

## C10K

The classic 1999 problem:

> Can one machine hold **10,000 simultaneous persistent connections**?

The notes associate:

```text
C10K → 10,000
```

## C10M

A later scale target:

```text
C10M → 10,000,000
```

The notes mention modern demonstrations of roughly:

```text
2–3 million connections
```

on a single server.

### MCQ numbers

```text
C10K = 10,000
C10M = 10,000,000
```

---

# 24. `ulimit -n`

The notes show:

```bash
ulimit -n
```

with:

```text
1024
```

as the soft limit on many distributions.

### Important distinction

There are two related concepts worth separating:

- `FD_SETSIZE` → `select()`'s fixed descriptor-set ceiling.
- `ulimit -n` → process file-descriptor resource limit.

The teacher connects both to the **1024** scaling discussion, but they are not the same mechanism.

---

# 25. Thread Memory

The notes compare OS threads with goroutines.

### OS thread

The material gives:

```text
8 MB
```

as a default stack reservation example.

Therefore:

```text
10,000 threads × 8 MB
≈ 80 GB
```

of address space before application data.

### Goroutine

The notes give:

```text
2 KB
```

as an initial stack, which grows on demand.

The Go runtime multiplexes goroutines onto OS threads using a scheduler and netpoller built on `epoll`.

### MCQ trap

Goroutines are lightweight primarily because the runtime multiplexes many of them onto fewer OS threads and uses small, dynamically growing stacks.

---

# 26. Syscalls and Kernel Boundary

Every system call crosses the user/kernel boundary.

The notes emphasize that:

```text
read()
write()
```

are not free.

At very high operation rates, syscall overhead becomes significant.

---

# 27. `sendfile()`

Normal approach:

```text
disk
 ↓
kernel
 ↓
user-space buffer
 ↓
kernel
 ↓
socket
```

With:

```c
sendfile(sock_fd, file_fd, NULL, n);
```

the file can move:

```text
file → socket
```

without the application copying the bytes through a user-space buffer.

### MCQ

**Why is `sendfile()` useful?**

To reduce unnecessary user-space copying/round trips when serving files.

---

# 28. `io_uring`

The notes present `io_uring` as a way to:

> Submit a batch of operations through a shared ring rather than making one syscall per operation.

The larger idea:

```text
make fewer syscall transitions
```

---

# 29. Little's Law / Concurrency

The key formula:

```text
connections in flight
=
requests per second × seconds per request
```

or:

```text
L = λW
```

where:

- `L` = average number of items/connections in flight
- `λ` = throughput (requests/sec)
- `W` = average time per request (seconds)

---

# 30. Math Examples

## Example 1

Given:

```text
1000 requests/sec
100 ms/request
```

Convert:

```text
100 ms = 0.1 sec
```

Then:

```text
concurrency = 1000 × 0.1
            = 100
```

### Answer

**100 connections in flight**

---

## Example 2

Given:

```text
1000 requests/sec
1 sec/request
```

Then:

```text
1000 × 1 = 1000
```

### Answer

**1000 connections in flight**

---

## Example 3

Given:

```text
10,000 requests/sec
0.5 sec/request
```

Then:

```text
10,000 × 0.5 = 5,000
```

### Answer

**5,000 connections in flight**

---

## Example 4

Given:

```text
5,000 requests/sec
200 ms/request
```

Convert:

```text
200 ms = 0.2 sec
```

Then:

```text
5,000 × 0.2 = 1,000
```

### Answer

**1,000 connections in flight**

---

## Reverse calculation

If:

```text
concurrency = 2,000
throughput = 500 req/sec
```

Then:

```text
latency = concurrency / throughput
        = 2000 / 500
        = 4 sec
```

### Answer

**4 seconds/request**

---

# 31. Why Latency Is Dangerous

For identical traffic:

```text
higher latency
      ↓
more connections in flight
      ↓
more memory/resource usage
      ↓
larger queue
```

This is why the notes describe:

> **Slowness as a memory problem.**

High-latency work can be moved off the request path asynchronously so the queue can drain.

---

# 32. Apache

Apache evolved through:

```text
prefork → worker → event
```

## Prefork MPM

- Pool of processes.
- One connection per process.
- Strong isolation.
- Expensive per connection.

## Worker MPM

- Processes run multiple threads.
- Cheaper than process-per-connection.
- A thread can remain pinned to an idle keep-alive connection.

## Event MPM

A listener mechanism can park idle keep-alive sockets on `epoll`.

Only sockets with actual work are handed to worker threads.

### Key idea

```text
idle connection
      ↓
don't waste a worker thread
      ↓
event/epoll waits for activity
```

---

# 33. HAProxy

HAProxy is a **load balancer / TCP/HTTP proxy**.

### Timeline from the notes

- Arrived in **2001**.
- `epoll` appeared in Linux 2.5.44 in late **2002**.
- Therefore HAProxy was initially largely based on `select`.
- Later moved to `epoll`.
- Became multi-threaded in version **1.8 (2017)**.

### Important idea

A load balancer can maintain upstream connection pools.

This reduces connection churn between the load balancer and backend services.

---

# 34. Varnish

Varnish is a **caching reverse proxy / HTTP accelerator**.

### Direction matters

The notes contrast:

```text
Squid
client-side caching proxy
```

with:

```text
Varnish
cache in front of origin
```

### Concurrency model

- Acceptor thread per listening point/pool.
- Worker thread per active connection.
- Idle descriptors go to a waiter.
- Linux waiter uses `epoll`.
- BSD uses `kqueue`.

### Core pattern

> **Threads for work, event loop for waiting.**

---

# 35. nginx

## Architecture

```text
master
  ↓
worker 1 → epoll loop
worker 2 → epoll loop
worker 3 → epoll loop
...
```

The notes describe:

- One master.
- One worker per CPU core.
- Each worker is single-threaded.
- Each worker runs one `epoll` loop.
- Each worker can handle thousands of connections.

---

# 36. nginx and `sendfile()`

nginx uses `sendfile()` to serve static files efficiently.

Instead of:

```text
file → user buffer → socket
```

it can use:

```text
file → socket
```

through the kernel path.

### MCQ connection

**nginx + static files + avoiding unnecessary copying = `sendfile()`**

---

# 37. nginx Reverse Proxy and Other Roles

The notes describe nginx being used for:

- Reverse proxy
- Caching
- Load balancing
- TLS termination
- Rate limiting

It also has a thread pool for blocking disk reads.

### Why?

Even an event loop can be hurt by blocking disk operations.

A page fault/cache miss or blocking disk read can stall other work assigned to that worker.

---

# 38. All Four Servers — Quick Comparison

| Server | Main concurrency model | Key association |
|---|---|---|
| **Apache** | prefork → worker → event | Event MPM + epoll |
| **HAProxy** | select → epoll, later multi-threaded | Load balancer |
| **Varnish** | worker per active connection + epoll waiter | Caching reverse proxy |
| **nginx** | master + worker per core + epoll | `sendfile`, reverse proxy |

### Biggest pattern

Despite different histories:

> **They all ended up using an event loop plus a relatively small worker pool.**

---

# 39. Four Ideas to Remember

These are the final four ideas of the teacher's session.

## 1. Amortise the setup

Pay expensive setup **once at startup**, not once per request.

Examples:

- Pre-fork
- Pre-thread
- FastCGI
- Connection pools
- Servlets

---

## 2. Nothing is free per connection

Each connection consumes resources such as:

- Descriptor
- Stack
- Buffer
- Scheduler/resource state

Multiply even small costs by thousands.

---

## 3. Framing is length or delimiter

Two fundamental approaches:

```text
Length → exactly how many bytes?
Delimiter → where does the stream end?
```

FastCGI uses both:

- Record length.
- Empty record as stream delimiter.

---

## 4. Concurrency = Throughput × Latency

```text
concurrency
=
requests/sec × seconds/request
```

Faster requests reduce the number of connections held in flight for the same throughput.

---

# 40. High-Yield MCQ Traps

### Trap 1
**`select()` watches only 5 connections if only 5 are active.**

❌ Wrong.

It can watch many descriptors, but scans the watched set.

---

### Trap 2
**`epoll` is stateless while `select` is stateful.**

❌ Wrong terminology.

Use:

```text
select → repeatedly supplies/scans set
epoll → persistent registration + ready events
```

---

### Trap 3
**CGI request body goes to stdout.**

❌ Wrong.

```text
stdin → request/body
stdout → response
stderr → error log
```

---

### Trap 4
**FastCGI creates a new PHP process for every request.**

❌ Wrong.

FastCGI keeps long-lived application workers.

---

### Trap 5
**Empty `FCGI_PARAMS` contains no parameters, so it is useless.**

❌ Wrong.

It is a structural marker:

> parameters are finished.

---

### Trap 6
**HTTP statelessness means applications cannot have sessions.**

❌ Wrong.

It means HTTP itself does not require per-request connection state.

---

### Trap 7
**C10K means 10,000 requests/sec.**

❌ Wrong.

It refers to approximately:

> **10,000 simultaneous persistent connections.**

---

### Trap 8
**Increasing latency does not change concurrency if throughput stays constant.**

❌ Wrong.

From Little's Law:

```text
concurrency = throughput × latency
```

---

### Trap 9
**nginx uses `sendfile()` because it needs to copy files into user space faster.**

❌ Wrong.

The point is to **avoid the unnecessary user-space copy**.

---

### Trap 10
**A thread per connection has no meaningful memory cost.**

❌ Wrong.

Every thread requires stack/resource memory.

---

# 41. MCQ-Style Practice

## Single Correct

### Q1
What is the main scalability problem with `select()`?

A. It cannot monitor sockets  
B. It creates one process per socket  
C. It repeatedly scans the watched descriptor set  
D. It requires HTTP

**Answer: C**

---

### Q2
What does `NULL` as the final argument to `select()` mean?

A. Return immediately  
B. 1-second timeout  
C. Infinite timeout  
D. Disable reads

**Answer: C**

---

### Q3
What carries a CGI request body?

A. stdout  
B. stdin  
C. stderr  
D. envp only

**Answer: B**

---

### Q4
What is the main advantage of FastCGI over CGI?

A. It removes HTTP  
B. It eliminates sockets  
C. It reuses long-lived application workers  
D. It requires one thread per client

**Answer: C**

---

### Q5
Which record signals that FastCGI parameters are finished?

A. `FCGI_END_REQUEST`  
B. Empty `FCGI_PARAMS`  
C. Empty `FCGI_STDOUT`  
D. `FCGI_BEGIN_REQUEST`

**Answer: B**

---

### Q6
Which formula represents Little's Law as taught here?

A. `latency / throughput`  
B. `throughput / latency`  
C. `throughput × latency`  
D. `throughput + latency`

**Answer: C**

---

### Q7
What does C10K refer to?

A. 10,000 requests/sec  
B. 10,000 servers  
C. 10,000 simultaneous persistent connections  
D. 10,000 threads exactly

**Answer: C**

---

### Q8
Which nginx feature avoids unnecessary user-space copying when serving files?

A. `fork()`  
B. `select()`  
C. `sendfile()`  
D. `FCGI_PARAMS`

**Answer: C**

---

### Q9
Which is nginx's broad concurrency model in the notes?

A. One process per request  
B. Master + workers, each with an event loop  
C. One thread per HTTP request only  
D. One CGI process per file

**Answer: B**

---

### Q10
Which server is primarily associated with load balancing?

A. Varnish  
B. HAProxy  
C. php-fpm  
D. CGI

**Answer: B**

---

# 42. Multi-Select Practice

### Q11
Which are costs of a connection?

A. File descriptor  
B. Memory/buffers  
C. Scheduler/resource state  
D. Nothing if idle

**Answers: A, B, C**

---

### Q12
Which statements about FastCGI are correct?

A. Workers can be long-lived  
B. It communicates over Unix/TCP sockets  
C. It uses typed records  
D. It requires a new process for every request

**Answers: A, B, C**

---

### Q13
Which can be non-HTTP protocols/services from the notes?

A. Redis RESP  
B. MySQL binary protocol  
C. MQTT  
D. Kafka native protocol

**Answers: A, B, C, D**

---

### Q14
Which are examples of amortizing setup?

A. Pre-forking  
B. Pre-threading  
C. FastCGI  
D. Connection pooling

**Answers: A, B, C, D**

---

# 43. Math MCQs

### Q15
A server handles 2,000 requests/sec and average request latency is 50 ms. Approximate connections in flight?

A. 10  
B. 50  
C. 100  
D. 1,000

**Solution:**

```text
50 ms = 0.05 sec

2000 × 0.05 = 100
```

**Answer: C**

---

### Q16
A service handles 500 requests/sec with 2 seconds average latency. Approximate concurrency?

A. 250  
B. 500  
C. 1,000  
D. 2,000

```text
500 × 2 = 1000
```

**Answer: C**

---

### Q17
A service has 3,000 connections in flight and processes 750 requests/sec. Average request latency?

A. 0.25 sec  
B. 1 sec  
C. 2 sec  
D. 4 sec

```text
latency = 3000 / 750 = 4 sec
```

**Answer: D**

---

### Q18
A system's throughput remains 10,000 req/sec. Latency rises from 100 ms to 500 ms. What happens to concurrency?

Initial:

```text
10,000 × 0.1 = 1,000
```

New:

```text
10,000 × 0.5 = 5,000
```

**Answer:** concurrency increases **5×**.

---

# 44. Last-Minute Revision Sheet

```text
fork/client
→ process per connection
→ expensive

thread/client
→ cheaper than process
→ still stack/memory cost

select
→ rebuild fd set
→ scan all
→ O(n)
→ FD_SETSIZE = 1024
→ NULL timeout = wait forever

epoll
→ register once
→ epoll_wait()
→ ready events
→ O(ready)

CGI
→ process per request
→ stdin = request
→ stdout = response
→ stderr = error log
→ headers → env variables

FastCGI
→ long-lived workers
→ Unix/TCP socket
→ typed records
→ empty PARAMS = params finished
→ empty STDIN = body finished
→ STDOUT = response
→ STDERR = log
→ END_REQUEST = completion

HTTP
→ stateless
→ easier horizontal scaling

DB
→ persistent connections
→ avoid repeated TCP/handshake/auth/session setup

Limits
→ FD
→ memory
→ syscalls/context switches
→ queueing

C10K
→ 10,000 connections

C10M
→ 10,000,000 connections

Little's Law
→ concurrency = throughput × latency

sendfile
→ file → socket
→ avoid user-space copy

Apache
→ prefork → worker → event

HAProxy
→ load balancer
→ select → epoll

Varnish
→ caching reverse proxy
→ worker + epoll waiter

nginx
→ master + worker/core
→ epoll
→ sendfile
→ reverse proxy/cache/LB/TLS/rate limiting
```

---

# 45. The 10 Things I Would Memorize Before the Exam

1. **`select()` = O(n) scan; `epoll` = ready-event model.**
2. **`FD_SETSIZE = 1024`** in the teacher's `select()` discussion.
3. **`NULL` timeout = block forever.**
4. **CGI = new process per request.**
5. **CGI: stdin=request, stdout=response, stderr=error log.**
6. **FastCGI = long-lived workers + socket + typed records.**
7. **Empty FastCGI record = stream finished/delimiter.**
8. **C10K = 10,000 persistent connections.**
9. **Little's Law = throughput × latency = concurrency.**
10. **nginx = master/workers + epoll + `sendfile()`.**

---

# 46. Teacher's Core Message

The entire session can be reduced to this:

> **The loop is the architecture.**

Servers scale by moving expensive setup away from the request path, avoiding unnecessary per-connection costs, waiting efficiently for readiness, and keeping concurrency under control.

The four final ideas:

```text
1. Amortise the setup.
2. Nothing is free per connection.
3. Framing is length or delimiter.
4. Concurrency = throughput × latency.
```

