# ⚖️ NGINX as a Load Balancer

## 🎯 Goal

Use **NGINX to distribute incoming traffic across multiple backend servers**.

This helps applications achieve:

* Better availability
* Better reliability
* Horizontal scalability
* Reduced load on individual servers
* Basic fault tolerance

---

# 🧠 1. What is Load Balancing?

Imagine your application has only one backend:

```text
             Users
               |
               v
        ┌──────────────┐
        │   Backend    │
        │   Server     │
        │    :3000     │
        └──────────────┘
```

Suppose 10,000 users send requests.

Every request goes to the same server.

Eventually:

```text
Backend Server
     ↓
CPU:     🔥🔥🔥🔥🔥
RAM:     🔥🔥🔥🔥
Requests: ███████████████
```

The server can become overloaded.

Instead, we can run multiple backend servers:

```text
                 Users
                   |
                   v
              ┌─────────┐
              │  NGINX  │
              │  :80    │
              └────┬────┘
                   |
          ┌────────┼────────┐
          ↓        ↓        ↓
       Server 1 Server 2 Server 3
        :3001     :3002     :3003
```

NGINX distributes incoming requests between these servers.

This process is called:

> **Load balancing**

---

# 🔥 2. Why Do We Need Multiple Backend Servers?

There are three major reasons.

## 2.1 Prevent Server Overload

Instead of:

```text
1000 requests
      ↓
   Server 1
```

we can have:

```text
1000 requests
      ↓
    NGINX
      ↓
 ┌────┼────┐
 ↓    ↓    ↓
S1   S2   S3
333  333  334
```

Each server handles a portion of the traffic.

---

## 2.2 High Availability

Suppose:

```text
Server 1 → ❌ DOWN
Server 2 → ✅
Server 3 → ✅
```

NGINX can continue sending requests to the available servers.

Therefore, the entire application does not necessarily go down because one backend failed.

---

## 2.3 Horizontal Scaling

Instead of making one server extremely powerful:

```text
Server
64 CPU
256 GB RAM
```

we can add more servers:

```text
Server 1
Server 2
Server 3
Server 4
```

This is called:

> **Horizontal scaling**

Adding more machines/instances is horizontal scaling.

Adding more CPU/RAM to the same machine is:

> **Vertical scaling**

---

# 🧩 3. Where Does NGINX Fit?

The architecture becomes:

```text
                    Internet
                       |
                       v
                ┌─────────────┐
                │    NGINX    │
                │ Load Balancer│
                └──────┬──────┘
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
     Backend 1    Backend 2    Backend 3
       :3001        :3002        :3003
```

The client does NOT need to know about:

```text
:3001
:3002
:3003
```

The client only communicates with:

```text
NGINX :80
```

NGINX decides where the request should go.

---

# 🧠 4. Reverse Proxy vs Load Balancer

These concepts are closely related.

## Reverse Proxy

NGINX receives the request and forwards it to a backend.

```text
Client
  ↓
NGINX
  ↓
Backend
```

Example:

```text
/api/users → Backend
/app       → Frontend
```

---

## Load Balancer

NGINX receives the request and chooses between multiple backend servers.

```text
Client
  ↓
NGINX
  ↓
 ┌───────┬───────┐
 ↓       ↓       ↓
S1      S2      S3
```

Therefore:

> A load balancer can also be a reverse proxy.

In NGINX:

```text
Reverse Proxy:
NGINX → Backend

Load Balancer:
NGINX → Backend 1
     → Backend 2
     → Backend 3
```

---

# 🏗️ 5. The `upstream` Block

This is the most important concept for NGINX load balancing.

Example:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

We are creating a group called:

```text
backend_app
```

Inside that group:

```text
backend_app
     |
     ├── 127.0.0.1:3001
     └── 127.0.0.1:3002
```

You can think of `upstream` as:

> "These are the backend servers that NGINX can send requests to."

---

# ⚙️ 6. Basic Load Balancer Configuration

For Ubuntu/Linux NGINX installed directly:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_app;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

The important part is:

```nginx
proxy_pass http://backend_app;
```

Notice that we are NOT writing:

```nginx
proxy_pass http://127.0.0.1:3001;
```

Instead:

```nginx
proxy_pass http://backend_app;
```

NGINX sees that `backend_app` is an `upstream` group and chooses one of its servers.

---

# 🔄 7. Request Flow

Suppose we have:

```text
Backend 1 → :3001
Backend 2 → :3002
```

A client sends:

```http
GET /
```

Flow:

```text
Client
  |
  | GET /
  ↓
NGINX :80
  |
  | chooses backend
  ↓
Backend 1 :3001
  |
  | response
  ↓
NGINX
  |
  ↓
Client
```

Next request might go:

```text
Client
  ↓
NGINX
  ↓
Backend 2 :3002
```

---

# 🔄 8. Load Balancing Algorithms

NGINX supports different methods for deciding which backend receives a request.

The important ones to learn first are:

1. Round Robin
2. Least Connections
3. IP Hash

---

# 🔁 9. Round Robin

Round Robin is the default method.

Configuration:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

No algorithm needs to be specified.

NGINX distributes requests in sequence.

Example:

```text
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 1
Request 4 → Server 2
Request 5 → Server 1
Request 6 → Server 2
```

Visualization:

```text
             NGINX
               |
        ┌──────┴──────┐
        ↓             ↓
     Server 1      Server 2
        ↑             ↑
       R1,R3,R5     R2,R4,R6
```

### When is it useful?

Round Robin works well when:

* Servers have similar capacity
* Requests have roughly similar processing costs
* You want simple traffic distribution

---

# ⚖️ 10. Least Connections

Configuration:

```nginx
upstream backend_app {
    least_conn;

    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

Instead of simply rotating servers, NGINX considers the number of active connections.

Example:

```text
Server 1 → 10 active connections
Server 2 → 3 active connections
```

A new request is more likely to go to:

```text
Server 2
```

because it currently has fewer connections.

Visualization:

```text
                  New Request
                       |
                       v
                     NGINX
                       |
              Which server has
              fewer connections?
                       |
             ┌─────────┴─────────┐
             ↓                   ↓
        Server 1             Server 2
        10 conns              3 conns
                                  ↑
                             Request goes here
```

### When is it useful?

`least_conn` can be useful when:

* Requests take different amounts of time
* Some connections stay open for a long time
* Backends don't finish requests at the same speed

---

# 📌 11. Round Robin vs Least Connections

Suppose:

```text
Server 1 → 100 active connections
Server 2 → 5 active connections
```

### Round Robin

It may still alternate:

```text
Request → Server 1
Request → Server 2
Request → Server 1
Request → Server 2
```

### Least Connections

It considers current active connections:

```text
100 connections
       ↓
   Server 1

5 connections
       ↓
   Server 2 ← new request
```

So:

> Round Robin cares about sequence.

> Least Connections cares about current connection count.

---

# 📍 12. IP Hash

Configuration:

```nginx
upstream backend_app {
    ip_hash;

    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

NGINX uses the client's IP address to determine the backend.

Conceptually:

```text
Client A
   ↓
NGINX
   ↓
Server 1
```

Repeated requests from the same client IP will generally be directed to the same backend while the backend set remains suitable for that mapping.

Another client:

```text
Client B
   ↓
NGINX
   ↓
Server 2
```

---

# 🧠 13. Why Would We Want IP Hash?

Consider an application that stores temporary session information in server memory.

Example:

```text
User
 ↓
Server 1
 ↓
Session stored in Server 1 memory
```

If the next request goes to Server 2:

```text
User
 ↓
Server 2
 ↓
Session doesn't exist ❌
```

IP-based affinity can help keep a client mapped to the same backend.

However, modern distributed applications often avoid depending on local server memory.

Instead, session/state can be stored in something shared such as:

```text
Redis
Database
Distributed session store
```

Then any backend can handle the request.

---

# ⚠️ 14. Important Problem With IP Hash

IP hash does NOT mean:

> "One user = one server forever."

There are limitations.

For example, many users may appear to come from the same public IP because of:

* NAT
* Corporate networks
* Mobile networks
* Proxies

Also, a client's IP can change.

Therefore, IP hash is not always the best solution for session persistence.

---

# 🧪 15. Demo: Two Node.js Backend Servers

Let's create two simple servers.

Create a directory:

```bash
mkdir ~/load-test
cd ~/load-test
```

---

## Server 1

Create:

```text
server1.js
```

```javascript
require('http').createServer((req, res) => {
  res.end('Response from Server 1');
}).listen(3001);
```

---

## Server 2

Create:

```text
server2.js
```

```javascript
require('http').createServer((req, res) => {
  res.end('Response from Server 2');
}).listen(3002);
```

Now we have:

```text
Node Server 1 → localhost:3001
Node Server 2 → localhost:3002
```

---

# ▶️ 16. Run Both Servers

Run:

```bash
node server1.js &
node server2.js &
```

Check:

```bash
ss -ltnp | grep -E '3001|3002'
```

You should see both ports listening.

---

# ⚙️ 17. Configure NGINX

Ubuntu installation:

```bash
sudo nano /etc/nginx/sites-available/default
```

Configuration:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://backend_app;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# 🧪 18. Test NGINX Configuration

Always test before reloading:

```bash
sudo nginx -t
```

Expected:

```text
syntax is ok
test is successful
```

Then:

```bash
sudo systemctl reload nginx
```

---

# 🧪 19. Test the Load Balancer

Run:

```bash
curl http://localhost
```

Run it multiple times:

```bash
curl http://localhost
curl http://localhost
curl http://localhost
curl http://localhost
```

You should see responses coming from the backend servers.

For a simple two-server setup, you may observe something like:

```text
Response from Server 1
Response from Server 2
Response from Server 1
Response from Server 2
```

The exact observed sequence can depend on connection behavior, so don't treat the output as a guaranteed strict alternation.

---

# 🧠 20. What Actually Happens?

The client sends:

```text
GET /
```

to:

```text
localhost:80
```

NGINX receives it.

Then NGINX looks at:

```nginx
upstream backend_app
```

and finds:

```text
127.0.0.1:3001
127.0.0.1:3002
```

NGINX selects a backend according to the configured load-balancing method.

Then:

```text
Client
  |
  | GET /
  ↓
NGINX :80
  |
  | proxy
  ↓
Backend :3001
  |
  | response
  ↓
NGINX
  |
  ↓
Client
```

The client only sees NGINX.

---

# 🐳 21. NGINX Load Balancing With Docker

If NGINX and your backend servers are separate Docker containers, you generally shouldn't use:

```nginx
127.0.0.1:3001
```

to refer to another container.

Why?

Inside a container:

```text
127.0.0.1
```

means:

> This same container.

So:

```text
NGINX container
127.0.0.1:3001
```

means:

```text
NGINX container itself
```

not another backend container.

---

# 🐳 22. Docker Compose Architecture

Example:

```text
                    Client
                       |
                       v
                 ┌──────────┐
                 │  NGINX   │
                 │  :80     │
                 └────┬─────┘
                      |
             Docker Network
                      |
          ┌───────────┴───────────┐
          ↓                       ↓
    ┌────────────┐          ┌────────────┐
    │ backend1   │          │ backend2   │
    │ :3000      │          │ :3000      │
    └────────────┘          └────────────┘
```

Notice both backend containers can use port `3000`.

They have different container names:

```text
backend1
backend2
```

NGINX can use those names.

---

# ⚙️ 23. Docker NGINX Configuration

```nginx
upstream backend_app {
    server backend1:3000;
    server backend2:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_app;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Here:

```text
backend1
backend2
```

are Docker service/container names resolved through the Docker network.

---

# 🏥 24. Backend Failure

One of the biggest advantages of load balancing is handling backend failures.

Suppose:

```text
NGINX
  |
  ├── Server 1 → ✅
  └── Server 2 → ❌
```

NGINX can avoid sending traffic to an unavailable backend under its upstream failure handling.

So:

```text
Requests
   ↓
 NGINX
   ↓
Server 1
```

The application may continue working even though Server 2 is down.

---

# ⚠️ 25. Passive Failure Detection

NGINX can detect certain upstream failures while processing requests.

For example:

```text
Request
   ↓
NGINX
   ↓
Backend 1
   ↓
Connection fails
```

NGINX can temporarily consider that backend unavailable and try another suitable upstream server, depending on the configuration and failure type.

This is often called:

> Passive health checking / passive failure detection

It happens based on real request traffic.

---

# ❤️ 26. Active Health Checks

There is an important distinction.

### Passive checking

NGINX discovers problems while serving real traffic.

```text
Real request
     ↓
Backend fails
     ↓
NGINX notices
```

### Active health checking

A system periodically sends dedicated health-check requests:

```text
NGINX
  |
  | health check
  ↓
Backend
  |
  ↓
OK
```

Advanced active health-check functionality depends on the NGINX edition/module you are using.

For standard open-source NGINX, don't assume every commercial/third-party health-check feature is available.

---

# ⚙️ 27. Backend Weights

You can give servers different weights.

Example:

```nginx
upstream backend_app {
    server 127.0.0.1:3001 weight=3;
    server 127.0.0.1:3002 weight=1;
}
```

Conceptually:

```text
Server 1 → weight 3
Server 2 → weight 1
```

Server 1 receives more traffic.

Think:

```text
Server 1: █████████
Server 2: ███
```

This is useful when:

```text
Server 1 = powerful
Server 2 = less powerful
```

---

# 🚫 28. Temporarily Disable a Backend

You can mark a server as unavailable:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002 down;
}
```

Now:

```text
Server 1 → active
Server 2 → disabled
```

This can be useful during maintenance.

---

# 🧪 29. Backup Server

NGINX upstreams can also use a backup server.

Example:

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003 backup;
}
```

Conceptually:

```text
Primary servers:
    Server 1
    Server 2

Backup:
    Server 3
```

Server 3 is used when the primary servers are unavailable according to the upstream behavior.

---

# ⏱️ 30. Connection and Proxy Timeouts

Load balancing isn't only about choosing a server.

NGINX also controls how long it waits.

Example:

```nginx
proxy_connect_timeout 5s;
proxy_read_timeout 30s;
proxy_send_timeout 30s;
```

### `proxy_connect_timeout`

How long NGINX waits while establishing a connection to the backend.

### `proxy_read_timeout`

How long NGINX waits while reading a response from the backend.

### `proxy_send_timeout`

How long NGINX waits while sending data to the backend.

Example:

```text
NGINX
  |
  | connect
  ↓
Backend
```

If the backend doesn't respond within the configured timeout, NGINX can terminate the attempt.

---

# 📦 31. Request Headers

When NGINX proxies a request, it is useful to forward information about the original client.

Common configuration:

```nginx
proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;
```

These headers help the backend understand:

```text
What hostname did the client request?
What was the original client IP?
What proxy chain did the request pass through?
Was the original request HTTP or HTTPS?
```

This becomes particularly important when you have:

```text
Client
  ↓
CDN
  ↓
NGINX
  ↓
Backend
```

---

# 🧠 32. Important Concept: Stateless Backends

Load balancing works best when backend servers are **stateless**.

Stateless means:

> The server does not depend on its own local memory to remember important user state between requests.

Bad architecture:

```text
Request 1
   ↓
Server 1
   ↓
Session stored only in RAM
```

Then:

```text
Request 2
   ↓
Server 2
   ↓
Session missing ❌
```

Better architecture:

```text
              ┌─────────────┐
              │    Redis    │
              │ Shared State│
              └──────┬──────┘
                     |
          ┌──────────┴──────────┐
          ↓                     ↓
      Server 1              Server 2
```

Both servers can access the shared state.

This allows NGINX to distribute requests more freely.

---

# 🧠 33. Load Balancing ≠ Scaling Everything

Adding NGINX does not automatically solve every scaling problem.

You may have:

```text
              NGINX
                |
       ┌────────┼────────┐
       ↓        ↓        ↓
      API1     API2     API3
       |        |        |
       └────────┼────────┘
                ↓
             Database
```

Now the database may become the bottleneck.

So scaling an application often requires thinking about:

```text
NGINX
  ↓
Application servers
  ↓
Cache / Redis
  ↓
Database
```

Each layer can become a bottleneck.

---

# 🔥 34. Real-World Architecture

A simplified production architecture might look like:

```text
                    Internet
                       |
                       ↓
                 Load Balancer
                  /   NGINX
                       |
             ┌─────────┴─────────┐
             ↓         ↓         ↓
          API-1      API-2      API-3
             |         |         |
             └────┬────┴────┬────┘
                  ↓         ↓
               Redis     Database
```

For larger systems, there may also be:

```text
Internet
   ↓
CDN
   ↓
Cloud Load Balancer
   ↓
NGINX
   ↓
Application Servers
   ↓
Redis / Database / Queue
```

---

# 🧪 35. Testing Different Algorithms

## Round Robin

```nginx
upstream backend_app {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

---

## Least Connections

```nginx
upstream backend_app {
    least_conn;

    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

---

## IP Hash

```nginx
upstream backend_app {
    ip_hash;

    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}
```

After changing configuration:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

# 🧠 36. Mental Model

Remember this:

```text
                CLIENTS
                   |
                   ↓
                NGINX
            Load Balancer
                   |
          ┌────────┼────────┐
          ↓        ↓        ↓
       Backend  Backend  Backend
          1        2        3
```

NGINX's job:

> **Receive traffic → choose backend → forward request → return response**

The algorithm decides:

> **Which backend should receive this request?**

---

# 📚 37. Important NGINX Load Balancing Concepts

| Concept                 | Meaning                                       |
| ----------------------- | --------------------------------------------- |
| `upstream`              | Defines a group of backend servers            |
| `server`                | Defines one backend inside the upstream       |
| Round Robin             | Distributes requests sequentially             |
| `least_conn`            | Prefers backend with fewer active connections |
| `ip_hash`               | Maps clients based on IP                      |
| `weight`                | Gives a backend more/less traffic             |
| `down`                  | Marks a backend unavailable                   |
| `backup`                | Uses backend as a fallback                    |
| `proxy_pass`            | Sends request to upstream/backend             |
| `proxy_connect_timeout` | Backend connection timeout                    |
| `proxy_read_timeout`    | Backend response-read timeout                 |
| `proxy_set_header`      | Controls headers sent to backend              |

---

# 🎯 38. Interview-Level Understanding

### Q: What is load balancing?

Distributing incoming traffic across multiple backend servers to improve scalability, availability, and reliability.

### Q: What is an NGINX `upstream`?

An `upstream` defines a group of backend servers that NGINX can proxy requests to.

### Q: What is the default NGINX load-balancing algorithm?

Round Robin.

### Q: When would you use `least_conn`?

When requests or connections can have different durations and distributing based on active connections is more appropriate.

### Q: What does `ip_hash` do?

It uses the client IP to consistently map requests to a backend, providing a form of session affinity.

### Q: Why use multiple backend servers?

To handle more traffic, improve availability, and avoid depending on a single application server.

### Q: What is horizontal scaling?

Adding more application instances/servers instead of only increasing the resources of one server.

### Q: Why are stateless applications easier to load balance?

Because any backend can handle any request without depending on local in-memory session/state.

---

# ⚠️ 39. Common Beginner Mistakes

## Mistake 1

Using:

```nginx
proxy_pass http://127.0.0.1:3001;
```

when the backend is actually in another Docker container.

Remember:

```text
127.0.0.1 inside NGINX container
        =
NGINX container itself
```

For Docker Compose, use:

```nginx
proxy_pass http://backend_app;
```

with:

```nginx
upstream backend_app {
    server backend1:3000;
    server backend2:3000;
}
```

---

## Mistake 2

Forgetting to test configuration.

Always:

```bash
sudo nginx -t
```

before:

```bash
sudo systemctl reload nginx
```

---

## Mistake 3

Assuming Round Robin means perfectly equal traffic.

Real traffic distribution can be affected by connection behavior, failures, weights, keepalive behavior, and other configuration details.

---

## Mistake 4

Using IP Hash as the only session solution.

A better modern architecture is often:

```text
Backend 1 ──┐
            ├── Redis / shared state
Backend 2 ──┘
```

rather than relying on one particular backend.

---

# 🧠 40. Final Mental Model

Think of NGINX as a **traffic manager**.

Without load balancing:

```text
             Users
                |
                ↓
           Backend 1
```

With NGINX:

```text
             Users
                |
                ↓
             NGINX
                |
        ┌───────┼───────┐
        ↓       ↓       ↓
      App 1   App 2   App 3
```

NGINX decides:

```text
"Which backend should handle this request?"
```

based on the configured load-balancing strategy.

---

# 🚀 What You Should Know Before Moving On

For a strong understanding of NGINX load balancing, make sure you understand these in this order:

```text
1. What load balancing means
        ↓
2. Reverse proxy vs load balancer
        ↓
3. upstream
        ↓
4. Round Robin
        ↓
5. least_conn
        ↓
6. ip_hash
        ↓
7. weight / backup / down
        ↓
8. Backend failure
        ↓
9. Stateless applications
        ↓
10. Docker + NGINX load balancing
```

If you understand these, you have the foundation needed to move from basic NGINX configuration toward **production-style traffic architecture**.
