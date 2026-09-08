# 🚀 NGINX as HTTP Cache

NGINX can work as an **HTTP reverse proxy cache**.

Instead of sending every request to the backend server, NGINX can store responses and return the cached response for future requests.

This can:

* Reduce backend load
* Reduce response time
* Reduce network traffic
* Improve application performance

---

# 🔄 How HTTP Caching Works

Without caching:

```text
Client
   |
   v
 NGINX
   |
   v
Backend
   |
   v
Response
   |
   v
Client
```

Every request reaches the backend.

With NGINX caching:

```text
                ┌───────────────┐
Client ────────>│     NGINX     │
                │               │
                │     Cache     │
                └───────┬───────┘
                        |
                 Cache HIT? ─── Yes ──> Response
                        |
                       No
                        |
                        v
                     Backend
                        |
                        v
                     Response
                        |
                        v
                   Store in Cache
```

---

# 🎯 Cache HIT vs Cache MISS

## Cache HIT

The requested response already exists in the cache.

```text
Client
  |
  v
NGINX
  |
  v
Cache
  |
  └── HIT
       |
       v
    Response
```

The backend does not need to process the request.

---

## Cache MISS

The response is not available in the cache.

```text
Client
  |
  v
NGINX
  |
  v
Cache
  |
  └── MISS
       |
       v
    Backend
       |
       v
    Response
       |
       v
    NGINX
       |
       ├── Store response in cache
       |
       └── Send response to client
```

---

# ⚙️ Basic Cache Configuration

First define a cache zone using `proxy_cache_path`.

```nginx
http {

    proxy_cache_path /var/cache/nginx
        levels=1:2
        keys_zone=my_cache:10m
        max_size=1g
        inactive=60m;

    server {

        listen 80;

        location / {

            proxy_pass http://backend;

            proxy_cache my_cache;

            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;
        }
    }
}
```

---

# 📁 `proxy_cache_path`

```nginx
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=my_cache:10m
    max_size=1g
    inactive=60m;
```

Important options:

### `/var/cache/nginx`

Directory where cached responses are stored.

### `levels=1:2`

Defines the directory structure used for cache files.

### `keys_zone=my_cache:10m`

Creates a shared memory zone called `my_cache`.

NGINX uses this zone to store information about cached objects.

### `max_size=1g`

Maximum cache size.

### `inactive=60m`

If a cached response is not accessed for 60 minutes, it can be removed.

---

# 🗂️ `proxy_cache`

```nginx
proxy_cache my_cache;
```

This enables the cache zone named `my_cache` for the location.

---

# ⏱️ `proxy_cache_valid`

```nginx
proxy_cache_valid 200 10m;
proxy_cache_valid 404 1m;
```

This defines how long different responses should be cached.

For example:

```text
200 → 10 minutes
404 → 1 minute
```

---

# 🔑 Cache Key

NGINX needs a way to identify cached responses.

The `proxy_cache_key` directive defines this key.

Example:

```nginx
proxy_cache_key "$scheme$request_method$host$request_uri";
```

A simple example could be:

```text
GET + example.com + /api/users
```

Different cache keys represent different cached responses.

---

# 📦 What Should Be Cached?

Caching is generally useful for responses that don't change frequently.

Examples:

```text
Static files
Images
CSS
JavaScript
Public API responses
Product/catalog data
Documentation
```

Be careful when caching:

```text
User-specific data
Private responses
Authentication-related responses
Frequently changing data
```

---

# 🚫 Disable Cache for Certain Requests

You can use `proxy_no_cache`.

Example:

```nginx
location /api/ {

    proxy_pass http://backend;

    proxy_cache my_cache;

    proxy_no_cache $http_authorization;
}
```

This can prevent responses associated with authenticated requests from being cached based on the configured condition.

---

# 📊 Checking Cache Status

You can expose the cache status using a response header:

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

Then you may see:

```text
X-Cache-Status: HIT
```

or:

```text
X-Cache-Status: MISS
```

Other useful states include:

```text
HIT
MISS
BYPASS
EXPIRED
STALE
UPDATING
```

---

# 🧠 Important Directives

| Directive               | Purpose                                                              |
| ----------------------- | -------------------------------------------------------------------- |
| `proxy_cache_path`      | Defines cache storage                                                |
| `proxy_cache`           | Enables a cache zone                                                 |
| `proxy_cache_key`       | Defines cache key                                                    |
| `proxy_cache_valid`     | Defines cache lifetime                                               |
| `proxy_cache_methods`   | Defines cacheable request methods                                    |
| `proxy_no_cache`        | Prevents storing certain responses                                   |
| `proxy_cache_bypass`    | Prevents using cache under conditions                                |
| `proxy_cache_use_stale` | Allows stale cached responses in certain cases                       |
| `proxy_cache_lock`      | Prevents many requests from simultaneously populating the same cache |

NGINX documents `GET` and `HEAD` as the default cache methods.

---

# 🏗️ Real-World Architecture

```text
                    ┌──────────────┐
                    │   Clients    │
                    └──────┬───────┘
                           |
                           v
                    ┌──────────────┐
                    │    NGINX     │
                    │              │
                    │ Load Balancer│
                    │     +        │
                    │ HTTP Cache   │
                    └──────┬───────┘
                           |
             ┌─────────────┼─────────────┐
             |             |             |
             v             v             v
         Backend 1     Backend 2     Backend 3
```

NGINX can therefore act as both:

```text
Reverse Proxy
      +
Load Balancer
      +
HTTP Cache
```

---

# 🎯 What I Need to Know

For DevOps, you don't need to memorize every caching directive.

Understand:

1. What HTTP caching is
2. Cache HIT vs MISS
3. `proxy_cache_path`
4. `proxy_cache`
5. `proxy_cache_valid`
6. Cache keys
7. What should/shouldn't be cached
8. Cache invalidation
9. Basic cache debugging
10. How caching reduces backend load

---

# 🧪 Simple Practice

Run:

```text
Client → NGINX → Backend
```

Then enable:

```nginx
proxy_cache my_cache;
```

Send the same request multiple times.

Observe:

```text
First request  → MISS → Backend
Second request → HIT  → Cache
Third request  → HIT  → Cache
```

This is the most important concept to understand.
