# 📧 NGINX as Mail Proxy

NGINX can also work as a **mail proxy**.

Instead of HTTP traffic, NGINX can proxy email protocols such as:

* IMAP
* POP3
* SMTP

NGINX can provide a single endpoint in front of multiple mail servers.

---

# 🔄 How Mail Proxy Works

Without NGINX:

```text
Email Client
     |
     +--------> Mail Server 1
     |
     +--------> Mail Server 2
```

With NGINX:

```text
                ┌──────────────┐
Email Client ──>│    NGINX     │
                │  Mail Proxy  │
                └──────┬───────┘
                       |
              ┌────────┼────────┐
              |        |        |
              v        v        v
           Mail 1    Mail 2   Mail 3
```

NGINX becomes the entry point for the mail clients.

---

# 📮 Supported Mail Protocols

NGINX can proxy:

### IMAP

Used by email clients to access and manage email stored on a mail server.

```text
Email Client
     |
     v
   NGINX
     |
     v
   IMAP Server
```

---

### POP3

Another protocol used for retrieving email.

```text
Email Client
     |
     v
   NGINX
     |
     v
   POP3 Server
```

---

### SMTP

Used for sending email.

```text
Email Client
     |
     v
   NGINX
     |
     v
   SMTP Server
```

---

# 🔐 SSL/TLS with Mail Proxy

Mail traffic can also use SSL/TLS.

Conceptually:

```text
Email Client
     |
   TLS
     |
     v
   NGINX
     |
   TLS
     |
     v
Mail Server
```

NGINX supports mail SSL/TLS when the appropriate mail SSL module is available.

---

# ⚙️ Mail Module

Mail proxy configuration uses the `mail` context.

A simplified example:

```nginx
mail {

    server {
        listen 110;
        protocol pop3;
        proxy on;
    }

    server {
        listen 143;
        protocol imap;
        proxy on;
    }
}
```

The exact configuration depends on the mail protocol, authentication setup, upstream mail servers, and security requirements.

---

# 🧩 Important Mail Proxy Directives

Some directives provided by the NGINX mail proxy module include:

```text
proxy_buffer
proxy_pass_error_message
proxy_protocol
proxy_smtp_auth
proxy_timeout
xclient
```

These control different aspects of proxying mail traffic.

---

# 🏗️ Why Use NGINX as Mail Proxy?

A mail proxy can provide:

### 1. Single Entry Point

Clients can connect to one endpoint instead of directly connecting to multiple mail servers.

### 2. Load Distribution

Mail traffic can be distributed across multiple mail servers.

### 3. Scaling

More mail servers can be added behind the proxy.

### 4. Routing

NGINX can help route clients toward appropriate backend mail servers.

NGINX's documentation specifically describes using the mail proxy to scale mail servers, choose servers according to rules such as client IP, and distribute load.

---

# 🔐 Mail Proxy vs HTTP Reverse Proxy

The concepts are similar, but the protocols are different.

### HTTP Reverse Proxy

```text
Client
  |
  | HTTP / HTTPS
  v
NGINX
  |
  v
Web Server / API
```

### Mail Proxy

```text
Email Client
  |
  | IMAP / POP3 / SMTP
  v
NGINX
  |
  v
Mail Server
```

---

# ⚠️ Important

Mail proxying is **not the same thing as an email server**.

NGINX does not become your complete mail system.

It acts as a proxy between email clients and mail servers.

```text
NGINX
  ↓
Proxy

Mail Server
  ↓
Actually handles email
```

---

# 🎯 What a DevOps Engineer Should Know

For normal DevOps roles, you usually don't need deep expertise in NGINX mail proxying.

Understand:

1. What a mail proxy is
2. IMAP
3. POP3
4. SMTP
5. Why a proxy can sit in front of mail servers
6. Basic mail proxy architecture
7. TLS/SSL concept
8. Basic NGINX mail configuration

You should spend **much more time on HTTP reverse proxy, load balancing, TLS, and HTTP caching** than on mail proxying.

---

# 🧠 NGINX Feature Summary

```text
                 NGINX
                   |
       ┌───────────┼───────────┐
       |           |           |
       v           v           v
  Web Server   Reverse Proxy  Load Balancer
                               |
                               v
                         Multiple Backends

       ┌────────────────────────────┐
       |                            |
       v                            v
  HTTP Cache                    Mail Proxy
                                   |
                          ┌────────┼────────┐
                          v        v        v
                        IMAP     POP3     SMTP
```

For DevOps:

```text
Web Server       → Deep
Reverse Proxy    → Deep
Load Balancer    → Deep
SSL/TLS          → Deep
HTTP Cache       → Medium
Mail Proxy       → Basic
```
