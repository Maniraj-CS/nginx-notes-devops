# Introduction to NGINX

Nginx is an open-source, high-performance web server that is also used for:

* Reverse Proxy
* Load Balancer
* HTTP Cache
* Mail Proxy

It is designed for high concurrency, high performance, and low memory usage.

## Installing NGINX

### Using Docker

Start a container:

> `docker run --name nginx -p 8080:80 -d nginx`

If you want to run a terminal inside NGINX:

> `docker exec -it <container_id or container_name> bash`

If the container is stopped, use this command to run it again:

> `docker start <container_name or container_id>`

If you want to view logs:

> `docker logs <container_name or container_id>`

### Clean Up

Stop the container:

> `docker stop <container_name or container_id>`

Delete the container:

> `docker rm <container_name or container_id>`

### Using Linux/Mac

> `sudo apt update && sudo apt upgrade -y`

> `sudo apt install nginx -y`

## 📁 NGINX File Structure (Linux)

```text
File/Directory                     Purpose

/etc/nginx/nginx.conf              Main configuration file
/etc/nginx/sites-available/        Stores virtual host (server block) configs
/etc/nginx/sites-enabled/          Symlinks to active site configs
/var/www/html                      Default web root directory
/var/log/nginx/                    Contains access and error logs
```

# 📁 File Structure Recap

NGINX configuration paths depend on **how NGINX was installed**.

There is an important difference between:

* NGINX installed directly on Ubuntu/Linux
* NGINX running inside the official Docker image

---

## 🐧 NGINX Installed Directly on Ubuntu/Linux

When NGINX is installed using the Ubuntu/Debian package manager, you commonly see:

```text
/etc/nginx/
├── nginx.conf
├── sites-available/
│   └── default
├── sites-enabled/
│   └── default
├── conf.d/
├── snippets/
└── mime.types
```

### Important Paths

| Path                                 | Purpose                                   |
| ------------------------------------ | ----------------------------------------- |
| `/etc/nginx/nginx.conf`              | Main/global NGINX configuration           |
| `/etc/nginx/sites-available/default` | Site/server configuration                 |
| `/etc/nginx/sites-enabled/default`   | Enabled site configuration                |
| `/etc/nginx/conf.d/`                 | Additional `.conf` files                  |
| `/var/www/html`                      | Default web root for serving static files |
| `/var/log/nginx/access.log`          | Records client requests                   |
| `/var/log/nginx/error.log`           | Records NGINX errors and warnings         |

### `sites-available`

This directory stores server configurations that are available to use.

Example:

```text
/etc/nginx/sites-available/default
```

You can have multiple configurations:

```text
sites-available/
├── website.conf
├── api.conf
└── admin.conf
```

---

### `sites-enabled`

This directory contains the configurations that NGINX should actually use.

On Ubuntu/Debian, it commonly works using symbolic links:

```text
sites-available/default
          |
          | symbolic link
          v
sites-enabled/default
```

You can think of it as:

```text
sites-available
        |
        | "Configurations I have"
        v
sites-enabled
        |
        | "Configurations I want NGINX to use"
        v
      NGINX
```

---

# 🐳 NGINX Inside Docker

The official NGINX Docker image usually does **not** use the Ubuntu-style:

```text
/etc/nginx/sites-available/default
```

Instead, it commonly uses:

```text
/etc/nginx/
├── nginx.conf
├── conf.d/
│   └── default.conf
├── mime.types
├── fastcgi_params
├── scgi_params
└── uwsgi_params
```

### Important Docker Paths

| Path                             | Purpose                                      |
| -------------------------------- | -------------------------------------------- |
| `/etc/nginx/nginx.conf`          | Main/global NGINX configuration              |
| `/etc/nginx/conf.d/default.conf` | Default server/reverse-proxy configuration   |
| `/etc/nginx/conf.d/`             | Directory for additional NGINX `.conf` files |
| `/usr/share/nginx/html`          | Default directory for static files           |
| `/var/log/nginx/access.log`      | Request/access logs                          |
| `/var/log/nginx/error.log`       | Error logs                                   |

---


# ⚙️ `nginx.conf` vs `default.conf`

This is an important concept.

## `/etc/nginx/nginx.conf`

This is the **main/global configuration**.

It controls things such as:

* Worker processes
* Events
* HTTP-level configuration
* Logging
* Global settings
* Loading other configuration files

For example, it commonly contains:

```nginx
http {
    include /etc/nginx/mime.types;

    include /etc/nginx/conf.d/*.conf;
}
```

That `include` is important.

It tells NGINX:

> "Load the `.conf` files from `/etc/nginx/conf.d/`."

Therefore:

```text
nginx.conf
    |
    | include
    v
conf.d/*.conf
    |
    v
default.conf
```

---

# 📄 `default.conf`

This is where you normally define your **server blocks**.

For example:

```nginx
server {
    listen 80;

    location /api/ {
        proxy_pass http://host.docker.internal:3000;
    }

    location /app/ {
        proxy_pass http://host.docker.internal:4000;
    }
}
```

---

# 🌐 `/usr/share/nginx/html`

This directory is normally used when NGINX is serving **static files**.

For example:

```text
/usr/share/nginx/html/
├── index.html
├── style.css
└── app.js
```

NGINX can serve these files directly.

```text
Browser
   |
   v
NGINX
   |
   v
/usr/share/nginx/html/index.html
```
---

# 📊 NGINX Logs

## Access Log

```text
/var/log/nginx/access.log
```

This records incoming requests.

For example:

```text
GET /api/users
GET /app/profile
POST /api/login
```

You can inspect it inside the container:

```bash
cat /var/log/nginx/access.log
```

Or follow it live:

```bash
tail -f /var/log/nginx/access.log
```

---

## Error Log

```text
/var/log/nginx/error.log
```

This contains NGINX errors and warnings.

For example:

```text
502 Bad Gateway
connection refused
configuration errors
upstream errors
```

You can inspect it:

```bash
cat /var/log/nginx/error.log
```

Or:

```bash
tail -f /var/log/nginx/error.log
```

---

# 🧪 Useful NGINX Commands

### Check configuration syntax

```bash
nginx -t
```

If successful:

```text
syntax is ok
test is successful
```

---

### Reload configuration

```bash
nginx -s reload
```

This reloads the configuration without completely stopping NGINX.

With your Docker container:

```bash
docker exec <container_id> nginx -t
```

Then:

```bash
docker exec <container_id> nginx -s reload
```

Or:

```bash
docker exec <container_id> sh -c 'nginx -t && nginx -s reload'
```

---

### Open a shell inside the NGINX container

```bash
docker exec -it <container_id> bash
```

If `bash` isn't available:

```bash
docker exec -it <container_id> sh
```

Then you can inspect:

```bash
ls /etc/nginx/
```

and:

```bash
ls /etc/nginx/conf.d/
```

---

# 🧠 Quick Comparison

| Purpose              | Ubuntu/Linux NGINX                   | Docker NGINX                     |
| -------------------- | ------------------------------------ | -------------------------------- |
| Main config          | `/etc/nginx/nginx.conf`              | `/etc/nginx/nginx.conf`          |
| Common server config | `/etc/nginx/sites-available/default` | `/etc/nginx/conf.d/default.conf` |
| Enabled sites        | `/etc/nginx/sites-enabled/`          | Usually not used                 |
| Static files         | `/var/www/html`                      | `/usr/share/nginx/html`          |
| Access log           | `/var/log/nginx/access.log`          | `/var/log/nginx/access.log`      |
| Error log            | `/var/log/nginx/error.log`           | `/var/log/nginx/error.log`       |

---

# ⚠️ Important Note

Don't assume that:

```text
/etc/nginx/sites-available/default
```

must exist on every NGINX installation.

It depends on the **installation/package layout**.

For your Docker setup, the important file is:

```text
/etc/nginx/conf.d/default.conf
```

So when you're following an Ubuntu NGINX tutorial and it says:

```text
/etc/nginx/sites-available/default
```

you should translate that concept to your Docker setup as:

```text
/etc/nginx/conf.d/default.conf
```

The exact path is different, but the purpose is similar: **it contains the server configuration that handles your incoming requests.**
