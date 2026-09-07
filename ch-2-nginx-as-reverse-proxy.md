# 🔄 NGINX as a Reverse Proxy

## 🧠 What is a Reverse Proxy?

A **reverse proxy** is a server that sits between the **client** and the **backend server**.

Instead of the client directly communicating with the backend, the client sends the request to the reverse proxy.

The reverse proxy then:

1. Receives the client request
2. Checks the request
3. Decides where the request should go
4. Forwards the request to the backend
5. Receives the backend response
6. Sends the response back to the client

### Without Reverse Proxy

```text
Client
   |
   | HTTP request
   v
Node.js :3000
   |
   | Response
   v
Client
```

The client directly knows about the Node.js server.

---

### With Reverse Proxy

```text
Client
   |
   | HTTP request
   v
 NGINX
   |
   | Forward request to
   v
Node.js :3000
   |
   | Response
   v
 NGINX
   |
   | Response
   v
Client
```

The client only communicates with NGINX.

---

# 🎯 Simple Real-World Example

Imagine a company has:

```text
Frontend
Backend API
Admin API
Authentication Service
Payment Service
```

Instead of exposing every service directly:

```text
Internet
   |
   +----> Frontend :3000
   |
   +----> API :4000
   |
   +----> Auth :5000
   |
   +----> Payment :6000
```

we can put NGINX in front:

```text
                    Internet
                       |
                       v
                    NGINX
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Frontend       API          Auth
       :3000         :4000         :5000
```

Now NGINX becomes the **single entry point**.

---

# 🔀 How Does NGINX Know Where to Send Requests?

NGINX can look at different parts of the request.

For example, it can use the URL path.

```text
/api/     → API server
/app/     → App server
/auth/    → Authentication server
```

Example:

```nginx
location /api/ {
    proxy_pass http://localhost:3000;
}

location /app/ {
    proxy_pass http://localhost:4000;
}
```

Now:

```text
http://localhost/api/users
        |
        v
      NGINX
        |
        v
Node.js :3000
```

And:

```text
http://localhost/app/profile
        |
        v
      NGINX
        |
        v
Node.js :4000
```

This is called **path-based routing**.

---


# 📊 Forward Proxy vs Reverse Proxy

| Feature               | Forward Proxy     | Reverse Proxy         |
| --------------------- | ----------------- | --------------------- |
| Represents            | Client            | Server                |
| Usually configured by | Client/network    | Server/infrastructure |
| Hides                 | Client            | Backend               |
| Main direction        | Client → Internet | Client → Backend      |
| Common use            | Corporate proxy   | NGINX                 |
| Load balancing        | Usually no        | Yes                   |
| Backend routing       | No                | Yes                   |
| API gateway role      | No                | Can be                |
| SSL termination       | Possible          | Very common           |

---

# 🚀 Why Use NGINX as a Reverse Proxy?

NGINX can do much more than simply forward requests.

It can become a **control layer** between users and your backend.

```text
                    Internet
                       |
                       v
              ┌─────────────────┐
              │      NGINX      │
              │                 │
              │ Rate Limiting   │
              │ IP Filtering   │
              │ Request Limits │
              │ Timeouts       │
              │ Headers        │
              │ Routing        │
              │ Logging        │
              └────────┬────────┘
                       |
              ┌────────┴────────┐
              |                 |
              v                 v
          Node.js           Node.js
           :3000             :4000
```

---

# 🛡️ 1. Protect Backend Servers

Without NGINX:

```text
Internet
   |
   v
Node.js :3000
```

The backend is directly exposed.

With NGINX:

```text
Internet
   |
   v
 NGINX
   |
   v
Node.js :3000
```

The backend can be kept on an internal network or otherwise restricted so clients use NGINX as the public entry point.

NGINX can also reject requests before they reach Node.js.

---

# 🚦 2. Rate Limiting

A client may send too many requests:

```text
Client
 |
 +-- Request
 +-- Request
 +-- Request
 +-- Request
 +-- Request
 +-- Request
 ...
```

If every request reaches Node.js, the application has to process all of them.

NGINX can limit the request rate first.

Example:

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

location /api/ {
    limit_req zone=api_limit burst=20;

    proxy_pass http://localhost:3000;
}
```

Meaning:

```text
Client
   |
   v
 NGINX
   |
   | Rate limit check
   |
   +---- Too many requests → Reject/limit
   |
   +---- Normal request → Backend
                         |
                         v
                     Node.js
```

### Important

Rate limiting is useful for reducing:

* Request floods
* Excessive API usage
* Some automated abuse
* Accidental traffic spikes

But it is **not complete DDoS protection** and does not replace application-level anti-abuse systems.

---

# 🔌 3. Connection Limiting

Rate limiting controls:

> How many requests can be made over time?

Connection limiting controls:

> How many active connections can exist at the same time?

Example:

```nginx
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

location /api/ {
    limit_conn conn_limit 20;

    proxy_pass http://localhost:3000;
}
```

This limits the number of simultaneous connections from one IP.

---

# 📦 4. Request Size Limiting

A client could send a very large request body.

Example:

```text
POST /upload

Request body = 500 MB
```

NGINX can reject requests that are too large:

```nginx
client_max_body_size 2M;
```

Now:

```text
Request <= 2 MB
      |
      v
   Backend
```

But:

```text
Request > 2 MB
      |
      v
   NGINX rejects
```

This prevents unnecessarily large requests from reaching your application.

---

# ⏱️ 5. Timeouts

Sometimes a backend takes too long to respond.

NGINX can define timeouts.

```nginx
proxy_connect_timeout 5s;
proxy_read_timeout 30s;
proxy_send_timeout 30s;
```

### `proxy_connect_timeout`

Maximum time NGINX waits to connect to the backend.

```text
NGINX ----connect----> Node.js
          |
          | 5 seconds
          |
          X
```

---

### `proxy_read_timeout`

How long NGINX waits for data from the backend.

```text
NGINX → Node.js

       waiting for response
              |
              | 30 seconds
              |
              X
```

---

### `proxy_send_timeout`

How long NGINX allows sending the request to the backend.

---

# 🚫 6. IP Blocking

NGINX can block specific IP addresses.

```nginx
location / {
    deny 192.168.1.100;
    allow all;

    proxy_pass http://localhost:3000;
}
```

Flow:

```text
Blocked IP
    |
    v
  NGINX
    |
    X
 Reject
```

Allowed IP:

```text
Allowed IP
    |
    v
  NGINX
    |
    v
Backend
```

---

# 🔐 7. Access Control

NGINX can restrict certain paths.

For example:

```text
/api/       → Public
/admin/     → Restricted
/internal/  → Restricted
```

Example:

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    deny all;

    proxy_pass http://localhost:3000;
}
```

---

# 🧹 8. Request and Response Headers

NGINX can modify headers before sending requests to the backend.

Example:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

These headers allow the backend to understand information about the original request.

For example:

```text
Client
IP = 192.168.1.10

        |
        v

      NGINX
        |
        | X-Real-IP: 192.168.1.10
        v

     Node.js
```

Without appropriate proxy headers, the backend may see the proxy/container address instead of the original client IP.

---

# 📊 9. Logging

NGINX can record requests and errors.

For example:

```text
GET /api/users
POST /api/login
GET /app/dashboard
```

Logs are useful for:

* Debugging
* Monitoring
* Finding errors
* Understanding traffic
* Investigating suspicious traffic

Common NGINX logs:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

---

# 🛡️ 10. Security Headers

NGINX can add security-related response headers.

Example:

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
```

These tell browsers to apply safer behavior.

---

# 🔀 11. Path-Based Routing

One NGINX server can send different paths to different applications.

```nginx
location /api/ {
    proxy_pass http://host.docker.internal:3000;
}

location /app/ {
    proxy_pass http://host.docker.internal:4000;
}
```

Architecture:

```text
                    NGINX
                      |
          +-----------+-----------+
          |                       |
          v                       v
       /api/                    /app/
          |                       |
          v                       v
 Node.js :3000              Node.js :4000
```

---


# 🔥 Request Flow

Suppose the browser sends:

```text
GET /api/users
```

The request goes through several steps.

```text
1. Browser
      |
      v

2. NGINX
      |
      v

3. Check request size
      |
      v

4. Check rate limit
      |
      v

5. Check connection limit
      |
      v

6. Apply proxy headers
      |
      v

7. Forward request
      |
      v

8. Node.js :3000
      |
      v

9. Node.js creates response
      |
      v

10. NGINX receives response
      |
      v

11. Browser receives response
```

So NGINX is not just a simple "forwarder".

It can be the **first layer that controls traffic before your application receives it**.

---

# 🎯 Main Benefits of Reverse Proxy

## 1. Single Entry Point

Instead of exposing many backend services:

```text
Internet
 |
 +-- API
 +-- Auth
 +-- App
 +-- Admin
```

you expose:

```text
Internet
 |
 v
NGINX
 |
 +-- API
 +-- Auth
 +-- App
 +-- Admin
```

---

## 2. Backend Protection

NGINX can reject bad or excessive requests before they reach your application.

```text
Internet
   |
   v
 NGINX
   |
   +-- Bad request → Reject
   |
   +-- Too many requests → Limit
   |
   +-- Too large → Reject
   |
   +-- Valid request → Backend
```

---

## 3. Centralized Configuration

Instead of implementing everything separately in every Node.js server:

```text
Node.js #1 → rate limiting
Node.js #2 → rate limiting
Node.js #3 → rate limiting
```

you can apply some common traffic rules at NGINX.

```text
                 NGINX
                   |
          Common traffic rules
                   |
        +----------+----------+
        |          |          |
      Node #1    Node #2    Node #3
```

---

# ⚠️ Important: NGINX Is Not Your Application

NGINX should not replace your backend's business logic.

For example:

NGINX can do:

```text
Rate limiting
Request size
Routing
Connection limits
Timeouts
IP rules
Headers
Logging
```

But your Node.js application should handle:

```text
Authentication
Authorization
Database operations
Business logic
User validation
Payment logic
Application-specific abuse detection
```

Think of it like:

```text
             NGINX
        Traffic Control
              |
              v
          Node.js
       Business Logic
              |
              v
          Database
```

---


### One-line definition

> **NGINX as a reverse proxy receives client requests, applies traffic and access rules, forwards valid requests to backend services, and returns their responses to the client.**



### This is /etc/nginx/conf.d/default file in docker nginx container or /etc/nginx/sites-available/default in Ubuntu/Linux for use as a reverse proxy.


```bash
# ============================================================
# RATE LIMITING
# ============================================================

# Create a rate-limit zone based on the client's IP address.
#
# $binary_remote_addr = client's IP address
# zone=api_limit:10m  = reserve 10 MB memory for tracking IPs
# rate=10r/s          = allow 10 requests per second per IP
#
# If a client sends requests too quickly, Nginx will limit them
# before the request reaches the backend server.
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

# Separate rate limit for the /app/ service.
# Here we allow 20 requests per second per IP.
limit_req_zone $binary_remote_addr zone=app_limit:10m rate=20r/s;


# ============================================================
# CONNECTION LIMITING
# ============================================================

# Create a zone to track how many active connections
# each client IP currently has.
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;


# ============================================================
# SERVER
# ============================================================

server {
    listen 80;
    listen [::]:80;

    server_name localhost;


    # ========================================================
    # GENERAL REQUEST LIMITS
    # ========================================================

    # Maximum size of the request body.
    #
    # Example:
    # POST request with a body larger than 2 MB
    # will be rejected by Nginx.
    #
    # This helps prevent clients from sending extremely
    # large requests to the backend.
    client_max_body_size 2M;


    # ========================================================
    # TIMEOUTS
    # ========================================================

    # How long Nginx waits when connecting to the backend.
    # If backend does not accept the connection within 5s,
    # Nginx returns an error.
    proxy_connect_timeout 5s;

    # How long Nginx waits for a response from the backend.
    proxy_read_timeout 30s;

    # How long Nginx allows sending the request to the backend.
    proxy_send_timeout 30s;


    # ========================================================
    # API BACKEND
    # ========================================================

    location /api/ {

        # ----------------------------------------------------
        # RATE LIMIT
        # ----------------------------------------------------

        # Allow approximately 10 requests/second per IP.
        #
        # burst=20 means Nginx can temporarily allow
        # some extra requests instead of rejecting them
        # immediately.
        limit_req zone=api_limit burst=20;


        # ----------------------------------------------------
        # CONNECTION LIMIT
        # ----------------------------------------------------

        # Maximum 20 simultaneous connections from one IP.
        #
        # This is different from rate limiting:
        #
        # Rate limit       -> requests per second
        # Connection limit -> active connections at one time
        limit_conn conn_limit 20;


        # ----------------------------------------------------
        # REVERSE PROXY
        # ----------------------------------------------------

        # Forward /api/ requests to the Node.js server
        # running on the host machine at port 3000.
        proxy_pass http://host.docker.internal:3000;


        # ----------------------------------------------------
        # PROXY HEADERS
        # ----------------------------------------------------

        # Send the original Host header to Node.js.
        proxy_set_header Host $host;

        # Send the original client's IP address.
        proxy_set_header X-Real-IP $remote_addr;

        # Keep the complete chain of client/proxy IPs.
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Tell the backend which protocol the client used.
        # In your current setup this will normally be "http".
        proxy_set_header X-Forwarded-Proto $scheme;
    }


    # ========================================================
    # APP BACKEND
    # ========================================================

    location /app/ {

        # Allow approximately 20 requests/second per IP.
        limit_req zone=app_limit burst=40;

        # Maximum 20 active connections from one IP.
        limit_conn conn_limit 20;


        # Forward /app/ requests to the second Node.js server.
        proxy_pass http://host.docker.internal:4000;


        # Forward useful client information to the backend.
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }


    # ========================================================
    # BASIC SECURITY HEADERS
    # ========================================================

    # Prevent browsers from guessing the content type.
    add_header X-Content-Type-Options "nosniff" always;

    # Prevent your pages from being embedded in an iframe
    # by another site.
    add_header X-Frame-Options "SAMEORIGIN" always;


    # ========================================================
    # ERROR PAGE
    # ========================================================

    # If backend gives 500/502/503/504,
    # Nginx will show /50x.html.
    error_page 500 502 503 504 /50x.html;

    location = /50x.html {
        root /usr/share/nginx/html;
    }


    # ========================================================
    # BLOCK SOME HIDDEN FILES
    # ========================================================

    # Prevent access to hidden files such as:
    #
    # .env
    # .git
    # .htaccess
    #
    # This is useful because these files may contain
    # sensitive information.
    location ~ /\. {
        deny all;
    }
}

```