# Nginx HTTP Load Balancing: Methods and Features

## Overview

Nginx is a high-performance HTTP server and reverse proxy, well known for handling huge numbers of simultaneous connections without breaking a sweat. One of its standout capabilities is HTTP load balancing — spreading incoming traffic across multiple backend servers so that your web application stays available, scales cleanly, and behaves reliably under pressure.

This document walks through the load balancing methods Nginx supports, along with the more advanced features that come with them.

## Load Balancing Methods

### 1. Round Robin

This is Nginx's default. Requests are handed out to each backend server in turn, cycling through the pool evenly.

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

### 2. Least Connections

Traffic is routed to whichever server currently has the fewest active connections. This works especially well when request durations vary, since it prevents one server from getting buried while others sit idle.

```nginx
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 3. IP Hash

With this method, the client's IP address determines which backend handles the request. The same client always lands on the same server, which gives you a simple form of session persistence.

```nginx
upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 4. Generic Hash

Here you supply your own key — for example, a URL parameter — and Nginx uses it to decide which backend gets the request.

```nginx
upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 5. Random with Two Choices

Nginx picks two servers at random, then sends the request to whichever of the two has fewer connections. It's a middle ground between pure randomness and connection-based balancing.

```nginx
upstream backend {
    random two least_conn;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 6. Least Time

This method favors the server with the lowest average response time, factoring in active connections as well.

```nginx
upstream backend {
    least_time header;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

## Advanced Features

### 1. Health Checks

Nginx can actively probe backend servers and stop sending traffic to any that fail. Only healthy servers receive requests.

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;

    health_check interval=5s fails=3 passes=2;
}
```

### 2. Session Persistence (Sticky Sessions)

Sticky sessions keep a given client tied to the same backend for the life of the session.

```nginx
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

### 3. SSL/TLS Termination

Nginx can terminate SSL/TLS itself, so backend servers don't have to spend CPU cycles on encryption and decryption.

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/key.key;

    location / {
        proxy_pass http://backend;
    }
}
```

### 4. HTTP/2 and WebSocket Support

Nginx handles both HTTP/2 and WebSockets, which matters for modern web apps that rely on persistent connections.

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/key.key;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 5. Custom Load Balancing Logic

You can bend Nginx's balancing behavior to fit specific scenarios using variables and custom logic inside the upstream block.

### 6. Dynamic Configuration with NGINX Plus

The commercial NGINX Plus build adds features like reconfiguring upstreams without a reload, active health checks, and richer monitoring.

```nginx
upstream backend {
    zone backend 64k;
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
}
```

## Setting It Up

### 1. Basic Setup

At its simplest, load balancing means defining an `upstream` block and pointing a `server` block at it.

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

### 2. Switching Methods

Changing methods is a one-line change — just add the relevant directive inside the `upstream` block.

### 3. Turning On Health Checks

Health checks are enabled with a single directive, which is worth doing on any production setup.

### 4. Adding Session Persistence

Sticky sessions handle persistence when you need a client to keep hitting the same backend.

### 5. Terminating SSL/TLS

For HTTPS, configure a `server` block that listens on port 443 and points at your certificate and key.

## Best Practices

- Enable health checks so unhealthy backends drop out of rotation automatically.
- Terminate SSL/TLS at the load balancer to reduce backend load and centralize certificate management.
- Use sticky sessions where user session consistency is required.
- Take advantage of dynamic configuration (NGINX Plus) if you need to change upstreams without downtime.
- Keep Nginx updated and monitor it regularly for both performance and security.

## Wrapping Up

Nginx gives you a solid, flexible set of HTTP load balancing tools. Whether you need something as simple as round robin or as nuanced as least-time with health checks, the same upstream/server pattern covers it. Picking the right method and layering on the features you need is what keeps a web service available, scalable, and dependable.

Reference: [NGINX Plus – HTTP Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
