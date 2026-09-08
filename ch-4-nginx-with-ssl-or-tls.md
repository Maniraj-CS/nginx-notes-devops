# 🔒 SSL/TLS Setup in NGINX

## 🎯 Goal

Configure NGINX to serve an application over **HTTPS** using a **self-signed certificate**.

This is mainly useful for:

* Local development
* Internal applications
* Testing
* Learning HTTPS
* Testing production-like frontend/API behavior

For a public production website, you normally use a certificate issued by a trusted Certificate Authority (CA), such as Let's Encrypt.

---

# 🧠 1. HTTP vs HTTPS

Before understanding SSL/TLS, understand the problem with normal HTTP.

With HTTP:

```text
Client
   |
   | HTTP
   | GET /login
   ↓
NGINX
   |
   ↓
Backend
```

HTTP does not provide encryption by itself.

HTTPS adds TLS:

```text
Client
   |
   | 🔐 HTTPS
   ↓
NGINX
   |
   ↓
Backend
```

TLS protects the connection between the client and the HTTPS server.

---

# 🔐 2. What Does HTTPS Actually Give Us?

HTTPS/TLS provides three major security properties.

## 1. Confidentiality

Other parties should not be able to simply read the encrypted traffic.

Example:

```text
Client
  |
  | 🔐 encrypted
  | password=...
  ↓
Server
```

---

## 2. Integrity

TLS helps detect if data was modified while traveling between the endpoints.

Conceptually:

```text
Client
   |
   | "transfer $100"
   ↓
Network
   |
   | ❌ modified
   ↓
Server
```

TLS is designed to detect tampering with the protected connection.

---

## 3. Authentication

A certificate helps the browser verify that it is talking to the intended server/domain, assuming the certificate chains to a trusted CA.

This is where **certificates** become important.

---

# 🧠 3. SSL vs TLS

You will often hear:

```text
SSL
TLS
SSL certificate
TLS certificate
```

Technically:

> Modern HTTPS uses TLS, not the old SSL protocols.

"SSL certificate" is still commonly used as an informal term for a TLS certificate.

So when we say:

```text
SSL certificate
```

in these notes, understand it as:

```text
TLS certificate
```

---

# 🔑 4. Certificate + Private Key

HTTPS generally needs two important cryptographic objects:

```text
Certificate
     +
Private Key
```

Example:

```text
nginx-selfsigned.crt
nginx-selfsigned.key
```

Think of them as:

```text
Certificate
   ↓
Public information about the server
   +
identity information
```

and:

```text
Private Key
   ↓
Secret cryptographic key
```

---

# 🚨 5. Never Expose the Private Key

The private key is extremely sensitive.

For example:

```text
/etc/ssl/private/nginx-selfsigned.key
```

should be protected.

Do NOT:

```text
❌ commit it to GitHub
❌ put it in a public repository
❌ send it to users
❌ expose it through your web server
```

The certificate can be distributed publicly.

The private key must remain secret.

---

# 🛠️ 6. Creating a Self-Signed Certificate

The example uses OpenSSL.

Command:

```bash
sudo openssl req -x509 -nodes -days 365 \
 -newkey rsa:2048 \
 -keyout /etc/ssl/private/nginx-selfsigned.key \
 -out /etc/ssl/certs/nginx-selfsigned.crt
```

Let's break it down.

---

# 🔍 7. Understanding the OpenSSL Command

## `openssl`

```bash
openssl
```

OpenSSL is a cryptographic toolkit.

It can be used for things such as:

* Creating keys
* Creating certificates
* Inspecting certificates
* Testing TLS
* Cryptographic operations

---

## `req`

```bash
openssl req
```

The `req` command deals with certificate requests and certificate-related operations.

---

## `-x509`

```bash
-x509
```

This tells OpenSSL to create a self-signed X.509 certificate rather than just generating a Certificate Signing Request.

---

# 🧠 8. What Is X.509?

X.509 is a standard format used for digital certificates.

A certificate can contain information such as:

```text
Subject
Issuer
Validity period
Public key
Signature
Extensions
```

Conceptually:

```text
┌────────────────────────────┐
│       Certificate          │
├────────────────────────────┤
│ Subject: localhost         │
│ Issuer: localhost          │
│ Public Key                 │
│ Valid From                 │
│ Valid Until                │
│ Extensions                 │
│ CA Signature / Signature   │
└────────────────────────────┘
```

---

# 🔢 9. `-days 365`

```bash
-days 365
```

The certificate will be valid for approximately 365 days.

After expiration, the certificate needs to be replaced/renewed.

---

# 🔑 10. `-newkey rsa:2048`

```bash
-newkey rsa:2048
```

Generate a new RSA private key with a 2048-bit key size.

The important idea:

```text
RSA
 ↓
Asymmetric cryptography
```

There are two related keys:

```text
Public Key
Private Key
```

The private key must remain secret.

---

# 🚫 11. What Does `-nodes` Mean?

```bash
-nodes
```

In this context, it means the generated private key is not encrypted with an additional passphrase.

Why?

NGINX needs to start/reload without someone manually typing a password for the key.

This is convenient for local development.

For production, key-management practices should be chosen carefully rather than blindly copying this command.

---

# 📁 12. `-keyout`

```bash
-keyout /etc/ssl/private/nginx-selfsigned.key
```

This specifies where OpenSSL writes the private key.

Result:

```text
/etc/ssl/private/
└── nginx-selfsigned.key
```

---

# 📜 13. `-out`

```bash
-out /etc/ssl/certs/nginx-selfsigned.crt
```

This specifies where OpenSSL writes the certificate.

Result:

```text
/etc/ssl/certs/
└── nginx-selfsigned.crt
```

---

# 🏷️ 14. Common Name (CN)

During certificate creation, OpenSSL asks questions.

One important field is:

```text
Common Name (CN)
```

For local testing:

```text
localhost
```

can be used.

If you're testing another hostname, the certificate should be created with the appropriate hostname information.

---

# ⚠️ 15. Modern Certificates and SAN

This is an important advanced concept.

You may hear:

```text
CN = Common Name
SAN = Subject Alternative Name
```

Modern clients primarily use **SAN** to determine whether the certificate matches the hostname.

For example:

```text
SAN:
DNS:localhost
```

or:

```text
SAN:
DNS:example.com
DNS:www.example.com
```

Therefore, simply putting a hostname into CN is not the whole story for modern certificate validation.

For serious certificate generation, make sure the appropriate SAN entries are included.

---

# 🔐 16. Configure NGINX for HTTPS

For an Ubuntu package installation:

```bash
sudo nano /etc/nginx/sites-available/default
```

Example:

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location / {
        proxy_pass http://localhost:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

The important parts are:

```nginx
listen 443 ssl;
```

and:

```nginx
ssl_certificate ...
ssl_certificate_key ...
```

---

# 🚪 17. Why Port 443?

HTTP normally uses:

```text
80
```

HTTPS normally uses:

```text
443
```

Therefore:

```text
HTTP
localhost:80
```

versus:

```text
HTTPS
localhost:443
```

Typical architecture:

```text
Client
   |
   | HTTPS :443
   ↓
 NGINX
   |
   | HTTP or HTTPS
   ↓
Backend
```

---

# 🔄 18. HTTPS Termination

This is an important system-design concept.

Suppose:

```text
Client
   |
   | HTTPS
   ↓
NGINX
   |
   | HTTP
   ↓
Backend
```

NGINX receives encrypted HTTPS traffic.

NGINX decrypts the request.

Then it sends the request to the backend.

This is called:

> **TLS termination**

or:

> **SSL/TLS termination**

NGINX is acting as the TLS endpoint.

---

# 🧠 19. Why Terminate TLS at NGINX?

Imagine you have:

```text
                 HTTPS
Client ───────────────────→ NGINX
                              |
                              | HTTP
                              ↓
                           Backend 1
                              |
                              ↓
                           Backend 2
```

Now backend servers don't each need to handle the public TLS connection.

NGINX handles:

```text
TLS
Certificates
HTTPS connections
```

while the backend focuses on application logic.

This becomes especially useful when NGINX is also doing:

```text
Reverse proxy
Load balancing
Rate limiting
Request filtering
```

---

# ⚠️ 20. Is HTTP Between NGINX and Backend Always Safe?

Not necessarily.

If NGINX and the backend communicate on the same trusted machine/network:

```text
NGINX
  ↓
Backend
```

HTTP may be acceptable depending on the environment.

But in a distributed production architecture:

```text
NGINX
   |
   | network
   ↓
Backend server
```

you may also use TLS between NGINX and the backend.

Then:

```text
Client
   |
   | HTTPS
   ↓
NGINX
   |
   | HTTPS
   ↓
Backend
```

This provides encryption on both network segments.

---

# 🔄 21. HTTP → HTTPS Redirect

You can redirect HTTP requests to HTTPS:

```nginx
server {
    listen 80;
    server_name localhost;

    return 301 https://$host$request_uri;
}
```

Flow:

```text
Client
  |
  | http://localhost
  ↓
NGINX :80
  |
  | 301 Redirect
  ↓
https://localhost
  |
  ↓
NGINX :443
```

---

# 📌 22. What Does `301` Mean?

```text
301
```

is an HTTP status code meaning:

> Moved Permanently

The response tells the client to use another URL.

---

# 🔍 23. `$host`

```nginx
https://$host$request_uri
```

`$host` contains the requested host.

For example:

```text
localhost
```

---

# 🔍 24. `$request_uri`

`$request_uri` contains the requested URI.

For example:

```text
/api/users?page=2
```

Therefore:

```nginx
https://$host$request_uri
```

can become:

```text
https://localhost/api/users?page=2
```

---

# 🧪 25. Test NGINX Configuration

Before reloading:

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

# 🌐 26. Test HTTPS in Browser

Open:

```text
https://localhost
```

With a self-signed certificate, your browser will usually show a certificate warning.

Why?

Because your certificate was not issued by a CA that the browser trusts.

Conceptually:

```text
Browser
   |
   | "Who signed this certificate?"
   ↓
Self-signed certificate
   |
   | "I signed myself."
   ↓
Browser
   |
   | ⚠️ Not trusted
```

This is expected for a basic self-signed certificate.

---

# 🏢 27. Self-Signed vs Trusted Certificate

## Self-Signed

```text
You
 ↓
create certificate
 ↓
sign certificate yourself
```

Browser:

```text
⚠️ Not automatically trusted
```

Good for:

```text
Local development
Testing
Internal experiments
```

---

## CA-Signed Certificate

```text
Your server
     ↓
Certificate Authority
     ↓
Signed certificate
     ↓
Browser trusts CA
```

Browser:

```text
🔒 Trusted
```

Good for:

```text
Production websites
Public APIs
Public applications
```

---

# 🏛️ 28. What Is a Certificate Authority?

A Certificate Authority (CA) is an organization/system trusted by operating systems and browsers to issue certificates.

Examples include:

```text
Let's Encrypt
DigiCert
GlobalSign
```

The browser has a collection of trusted root CAs.

Conceptually:

```text
Browser
   |
   ↓
Trusted Root CA
   |
   ↓
Certificate
   |
   ↓
example.com
```

---

# 🔐 29. Certificate Chain

In production, certificates commonly form a trust chain.

Simplified:

```text
Root CA
   ↓
Intermediate CA
   ↓
Your Website Certificate
   ↓
example.com
```

The browser uses this chain to establish trust.

This is why production TLS configuration involves more than just:

```text
certificate
+
private key
```

---

# 🧪 30. Test HTTPS With curl

You can test without a browser:

```bash
curl -k https://localhost
```

The important part is:

```text
-k
```

It tells curl to skip normal certificate verification.

This is useful because your self-signed certificate is not automatically trusted.

---

# ⚠️ 31. Why Shouldn't You Use `-k` in Normal Production Requests?

Because:

```bash
curl -k
```

basically says:

> Don't verify the server certificate in the normal way.

That removes an important security check.

Use it for local testing when you intentionally have a self-signed certificate.

For trusted production certificates, normally use:

```bash
curl https://example.com
```

without `-k`.

---

# 📁 32. SSL File Paths

Ubuntu/Linux package installation:

```text
/etc/ssl/
├── certs/
│   └── nginx-selfsigned.crt
│
└── private/
    └── nginx-selfsigned.key
```

NGINX configuration:

```text
/etc/nginx/
└── sites-available/
    └── default
```

---

# 🐳 33. Important: Docker NGINX Setup

Your current NGINX setup is different from the Ubuntu example.

You are running NGINX inside Docker.

Therefore, you normally use:

```text
/etc/nginx/conf.d/default.conf
```

instead of:

```text
/etc/nginx/sites-available/default
```

Your architecture is approximately:

```text
Host Machine
     |
     ↓
Docker
     |
     ↓
┌───────────────────────┐
│    NGINX Container    │
│                       │
│ /etc/nginx/           │
│ ├── nginx.conf        │
│ └── conf.d/           │
│     └── default.conf  │
└───────────┬───────────┘
            |
            ↓
       Backend Server
```

---

# 🐳 34. HTTPS in Docker

The certificate and private key must be available **inside the NGINX container**.

For example:

```text
NGINX container

/etc/nginx/
    conf.d/
        default.conf

/etc/ssl/
    certs/
        nginx-selfsigned.crt
    private/
        nginx-selfsigned.key
```

Then NGINX can use:

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location / {
        proxy_pass http://host.docker.internal:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# 🔐 35. Docker Port Mapping

The NGINX container must expose HTTPS.

For example:

```text
Host port 443
      ↓
Container port 443
```

Docker Compose might conceptually contain:

```yaml
ports:
  - "80:80"
  - "443:443"
```

Then:

```text
Browser
   |
   | https://localhost:443
   ↓
Host :443
   |
   ↓
Docker
   |
   ↓
NGINX Container :443
```

---

# 🧠 36. HTTPS + Reverse Proxy

Now combine what you've learned.

```text
                     Client
                        |
                        | HTTPS
                        ↓
                ┌───────────────┐
                │     NGINX     │
                │               │
                │ TLS Terminate │
                │ Reverse Proxy │
                └───────┬───────┘
                        |
                        | HTTP
                        ↓
                    Backend
                    :3000
```

NGINX performs:

```text
1. Receive HTTPS
2. TLS handshake
3. Decrypt request
4. Apply NGINX rules
5. Proxy request
6. Receive backend response
7. Send HTTPS response
```

---

# 🚀 37. HTTPS + Load Balancing

Now combine Section 4 and Section 5.

```text
                         Clients
                            |
                            | HTTPS
                            ↓
                    ┌──────────────┐
                    │    NGINX     │
                    │              │
                    │ TLS + Proxy  │
                    │ Load Balancer│
                    └──────┬───────┘
                           |
              ┌────────────┼────────────┐
              ↓            ↓            ↓
           Backend 1    Backend 2    Backend 3
             :3000        :3000        :3000
```

This is a much more realistic architecture.

NGINX can:

```text
HTTPS
 ↓
TLS termination
 ↓
Rate limiting
 ↓
Routing
 ↓
Load balancing
 ↓
Backend
```

---

# 🧠 38. TLS Handshake — High-Level Understanding

You don't need to memorize every cryptographic detail yet.

At a high level:

```text
Client
   |
   | ClientHello
   ↓
Server
   |
   | ServerHello + Certificate
   ↓
Client
   |
   | verifies certificate
   ↓
Secure key establishment
   |
   ↓
Encrypted communication
```

After the TLS handshake:

```text
Client ⇄ NGINX
      🔐 encrypted
```

The exact TLS handshake details depend on the TLS version and negotiated cryptographic algorithms.

---

# 🔑 39. Asymmetric vs Symmetric Encryption

This is important for understanding HTTPS.

## Asymmetric Cryptography

Uses:

```text
Public Key
Private Key
```

It is useful for authentication and establishing secure communication.

---

## Symmetric Cryptography

Uses:

```text
Shared Secret Key
```

Once the TLS session is established, symmetric encryption is generally used for efficiently protecting application data.

So conceptually:

```text
TLS setup
   ↓
Asymmetric/public-key mechanisms
   ↓
Establish shared session secrets
   ↓
Symmetric encryption
   ↓
Fast encrypted communication
```

You don't need to manually implement this.

TLS libraries such as OpenSSL handle it.

---

# 🛡️ 40. TLS Versions

Modern systems should use modern TLS versions.

Conceptually:

```text
Old:
SSLv2 ❌
SSLv3 ❌

Modern:
TLS 1.2 ✅
TLS 1.3 ✅
```

You should not enable obsolete SSL protocols.

For production configuration, use modern TLS settings appropriate for your NGINX/OpenSSL versions.

---

# ⚙️ 41. Useful TLS Configuration

A production-style configuration may include settings such as:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

This tells NGINX which TLS protocol versions are allowed.

For learning purposes, the most important idea is:

> Use modern TLS versions and disable obsolete protocols.

---

# ⚠️ 42. Self-Signed Certificates Are Not "Bad"

Self-signed certificates are not inherently useless.

They are useful for:

```text
Local development
Testing
Private environments
Learning TLS
```

The problem is **trust**, not the basic encryption mechanism.

For example:

```text
Self-signed certificate
        ↓
Encryption: ✅
Automatic browser trust: ❌
```

A trusted CA-issued certificate:

```text
CA-issued certificate
        ↓
Encryption: ✅
Browser trust: ✅
```

---

# 📊 43. HTTP vs HTTPS

| Feature               | HTTP | HTTPS                      |
| --------------------- | ---- | -------------------------- |
| Encryption            | ❌    | ✅                          |
| Integrity protection  | ❌    | ✅                          |
| Server authentication | ❌    | ✅ with trusted certificate |
| Default port          | 80   | 443                        |
| TLS                   | ❌    | ✅                          |

---

# 📊 44. Self-Signed vs CA-Signed

|                                 | Self-Signed | CA-Signed             |
| ------------------------------- | ----------- | --------------------- |
| Easy to create                  | ✅           | Usually more involved |
| Good for local testing          | ✅           | ✅                     |
| Browser automatically trusts it | ❌           | ✅                     |
| Public production use           | Usually ❌   | ✅                     |
| Encryption                      | ✅           | ✅                     |
| Requires CA trust chain         | ❌           | ✅                     |

---

# 📚 45. Important Terms

| Term            | Meaning                                               |
| --------------- | ----------------------------------------------------- |
| HTTP            | Unencrypted application protocol                      |
| HTTPS           | HTTP protected by TLS                                 |
| TLS             | Protocol providing secure communication               |
| Certificate     | Contains server identity/public-key information       |
| Private Key     | Secret key associated with the certificate            |
| X.509           | Common certificate standard                           |
| CA              | Certificate Authority                                 |
| Self-Signed     | Certificate signed by itself rather than a trusted CA |
| TLS Termination | NGINX handles the client's TLS connection             |
| Port 80         | Conventional HTTP port                                |
| Port 443        | Conventional HTTPS port                               |
| OpenSSL         | Cryptographic/TLS toolkit                             |
| SAN             | Subject Alternative Name                              |

---

# 🎯 46. Mental Model

Remember this:

```text
HTTP:

Client
  |
  | HTTP
  ↓
NGINX
  |
  ↓
Backend
```

HTTPS:

```text
Client
  |
  | 🔐 HTTPS
  ↓
NGINX
  |
  | HTTP/HTTPS
  ↓
Backend
```

NGINX becomes the secure front door:

```text
                 Internet
                    |
                    | HTTPS
                    ↓
             ┌──────────────┐
             │    NGINX     │
             │              │
             │ TLS          │
             │ Reverse Proxy│
             │ Load Balancer│
             └───────┬──────┘
                     |
              ┌──────┼──────┐
              ↓      ↓      ↓
             App1   App2   App3
```

---

# 🚀 47. What You Should Know Before Moving Forward

Learn these concepts in this order:

```text
1. HTTP vs HTTPS
        ↓
2. TLS
        ↓
3. Certificate
        ↓
4. Private Key
        ↓
5. Self-signed certificate
        ↓
6. Certificate Authority
        ↓
7. Port 443
        ↓
8. TLS termination
        ↓
9. HTTP → HTTPS redirect
        ↓
10. TLS handshake — high level
        ↓
11. HTTPS + Reverse Proxy
        ↓
12. HTTPS + Load Balancing
        ↓
13. Docker + NGINX + HTTPS
```

Once these are clear, you have a solid foundation for understanding how HTTPS is deployed in real backend architectures.

---

# 📝 One-Line Revision

> **NGINX can terminate TLS at the edge, use a certificate and private key to provide HTTPS, and then securely or insecurely proxy the request to backend services depending on the architecture.**
