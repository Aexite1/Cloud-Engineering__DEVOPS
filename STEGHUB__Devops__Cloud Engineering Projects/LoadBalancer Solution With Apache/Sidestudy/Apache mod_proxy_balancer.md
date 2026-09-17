# Apache mod_proxy_balancer – Side Self-Study

## Introduction

`mod_proxy_balancer` is an Apache HTTP Server module that can be used to distribute requests between multiple backend servers.

Instead of having every request go directly to one server, Apache can act as a load balancer and share the traffic between different backend servers. This can help with **availability, scalability, and handling more traffic**.

In this study, I looked at how `mod_proxy_balancer` is configured, the different load-balancing methods, sticky sessions, failover and recovery, health checks, and how to monitor the balancer.

---

## 1. Enabling the Required Modules

Before configuring the load balancer, Apache needs the required proxy modules to be enabled.

Depending on the Linux distribution, the configuration file could be `httpd.conf` or `apache2.conf`.

The important modules are:

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

### What each module does

* **`mod_proxy`** – Provides Apache's proxy functionality.
* **`mod_proxy_balancer`** – Adds the load-balancing functionality.
* **`mod_proxy_http`** – Allows Apache to proxy HTTP requests to backend servers.

---

## 2. Basic Load Balancer Configuration

After the required modules are enabled, I can define the backend servers that will receive the traffic.

For example:

```apache
<Proxy "balancer://mycluster">
    BalancerMember "http://backend1.example.com:8080"
    BalancerMember "http://backend2.example.com:8080"
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass "/app" "balancer://mycluster"
ProxyPassReverse "/app" "balancer://mycluster"
```

Here, the two backend servers are grouped together under `mycluster`.

### Understanding the configuration

**`BalancerMember`**

This is used to add a backend server to the balancer.

In this example:

```apache
BalancerMember "http://backend1.example.com:8080"
BalancerMember "http://backend2.example.com:8080"
```

there are two backend servers.

**`ProxySet lbmethod`**

This determines how Apache should distribute requests between the backend servers.

Some available methods include:

* **`byrequests`** – Distributes requests across the available workers.
* **`bytraffic`** – Distributes traffic based on the amount of data being handled.
* **`bybusyness`** – Takes the number of active requests into consideration.
* **`heartbeat`** – Uses an external heartbeat mechanism when determining load.

**`ProxyPass`**

This tells Apache where to send requests that come to a particular path.

```apache
ProxyPass "/app" "balancer://mycluster"
```

So requests going to `/app` are sent to the backend cluster.

**`ProxyPassReverse`**

This helps Apache handle response headers correctly when acting as a reverse proxy.

---

# 3. Advanced Configuration

A basic load balancer is useful, but there are situations where more control is needed.

Some of the additional configurations I looked at are **sticky sessions, failover and recovery, and health checks**.

---

## Sticky Sessions

Sticky sessions, also called **session persistence**, are used when requests from the same user need to continue going to the same backend server.

This can be important when an application stores session information on a particular server.

For example:

```apache
<Proxy "balancer://mycluster">
    BalancerMember "http://backend1.example.com:8080" route=1
    BalancerMember "http://backend2.example.com:8080" route=2
    ProxySet lbmethod=byrequests stickysession=JSESSIONID
</Proxy>
```

The important parts here are:

**`route`**

Each backend server is given a route identifier.

```apache
route=1
route=2
```

**`stickysession`**

This tells Apache which session cookie should be used when maintaining session persistence.

In this example:

```apache
stickysession=JSESSIONID
```

Apache uses the `JSESSIONID` cookie to identify the session.

---

## Failover and Recovery

Another important part of load balancing is knowing what happens when one of the backend servers fails.

For example:

```apache
<Proxy "balancer://mycluster">
    BalancerMember "http://backend1.example.com:8080" route=1 retry=5
    BalancerMember "http://backend2.example.com:8080" route=2 status=+H
    ProxySet lbmethod=byrequests stickysession=JSESSIONID
</Proxy>
```

### `retry`

```apache
retry=5
```

This specifies how long Apache should wait before trying the failed backend server again.

The value is given in seconds.

### `status`

```apache
status=+H
```

This can be used to set a particular status for a backend server. In this example, `+H` marks the server as a **hot standby**.

The main idea is that the load balancer should be able to deal with backend failures rather than allowing one failed server to bring down the entire application.

---

# 4. Health Checks

A load balancer also needs a way to determine whether its backend servers are healthy.

One way of configuring an HTTP health check is:

```apache
<Proxy "balancer://mycluster">
    BalancerMember "http://backend1.example.com:8080" route=1 hcmethod=GET hcuri=/healthcheck
    BalancerMember "http://backend2.example.com:8080" route=2 hcmethod=GET hcuri=/healthcheck
    ProxySet lbmethod=byrequests stickysession=JSESSIONID
</Proxy>
```

### `hcmethod`

This specifies the HTTP method Apache should use when performing the health check.

Here, the method is:

```apache
hcmethod=GET
```

### `hcuri`

This specifies the URI that Apache should request when checking the backend.

```apache
hcuri=/healthcheck
```

So Apache can request the `/healthcheck` endpoint to check the backend server.

This is more useful than simply checking whether a port is open because the check can target an actual application endpoint.

---

# 5. Understanding Sticky Sessions

Sticky sessions are worth looking at separately because they are commonly needed by applications that keep session information on individual servers.

The basic idea is simple:

> A user's session stays connected to the same backend server.

For example, suppose there are two backend servers:

```text
Backend 1
Backend 2
```

A user initially gets connected to Backend 1.

With sticky sessions enabled, later requests from that same session can continue going to Backend 1 instead of randomly being sent to Backend 2.

This is normally achieved using a session cookie.

## Example

```apache
<Proxy "balancer://mycluster">
    BalancerMember "http://backend1.example.com:8080" route=1
    BalancerMember "http://backend2.example.com:8080" route=2
    ProxySet lbmethod=byrequests stickysession=JSESSIONID
</Proxy>

ProxyPass "/app" "balancer://mycluster"
ProxyPassReverse "/app" "balancer://mycluster"
```

In this configuration:

* **`route`** identifies each backend server.
* **`stickysession`** specifies the session cookie used for maintaining the session.

### When Sticky Sessions Can Be Useful

Sticky sessions can be useful when:

* **The application is stateful** and stores session information directly on a backend server.
* **User-specific information** needs to remain available from the same backend server.
* The application would otherwise need the backend servers to constantly share session information.

---

# 6. Monitoring the Load Balancer

Apache also provides a balancer manager that can be used to monitor and manage the configured backend servers.

A basic configuration looks like this:

```apache
<Location "/balancer-manager">
    SetHandler balancer-manager
    Require ip 192.168.1.0/24
</Location>
```

### Understanding the configuration

**`SetHandler balancer-manager`**

This enables Apache's balancer manager for the specified location.

**`Require ip`**

This controls who can access the balancer manager.

For example:

```apache
Require ip 192.168.1.0/24
```

restricts access to clients within the specified IP range.

This is important because the balancer manager should not normally be left openly accessible to everyone.

---

# Conclusion

`mod_proxy_balancer` gives Apache the ability to distribute requests between multiple backend servers.

From this study, the main areas I looked at were:

* Enabling the required proxy modules
* Adding backend servers using `BalancerMember`
* Choosing a load-balancing method
* Configuring sticky sessions
* Handling backend failures and recovery
* Setting up health checks
* Monitoring the balancer using `balancer-manager`

The important thing I took from this is that load balancing is not only about splitting traffic between servers. The configuration also needs to consider **what happens when a server fails, how the health of the servers is checked, and whether the application needs users to remain connected to a particular backend server**.
