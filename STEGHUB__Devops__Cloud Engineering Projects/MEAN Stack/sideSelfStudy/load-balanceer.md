# Load Balancing: Distribution, Methods, and Cloud Use

## 1. Overview

Load balancing is a technique used to spread incoming requests across multiple servers or network paths. Rather than allowing one machine to handle all traffic, a load balancer distributes the workload so that available resources are used more effectively.

This is useful for improving responsiveness, availability, and the ability of an application to handle increasing traffic.

## 2. Main Forms of Load Balancers

Load balancers can broadly be implemented as either **hardware appliances** or **software-based solutions**.

### Hardware-Based Load Balancing

Hardware load balancers are dedicated physical appliances designed specifically for traffic management. They are commonly found in environments where high throughput and specialized networking capabilities are required.

**Strengths**
- High performance and reliability
- Dedicated processing resources
- Advanced security and SSL offloading capabilities

**Limitations**
- Higher purchase and maintenance costs
- Less adaptable than software-based alternatives

### Software-Based Load Balancing

A software load balancer runs on a conventional server or virtual machine and uses software to make traffic-routing decisions. This approach is common in cloud platforms, virtualized infrastructure, and data centers.

**Strengths**
- Lower cost
- Easy to scale
- Flexible deployment
- Fits well with DevOps and cloud workflows

**Limitations**
- Performance can depend on the host system
- Shared computing resources may affect throughput

## 3. Traffic Distribution Strategies

Load balancing methods can be grouped into approaches based on predefined rules and approaches that react to current system conditions.

### Static Approaches

Static methods use predetermined rules rather than continuously evaluating the current state of every server.

#### Round Robin

Requests are assigned to servers in sequence.

**Advantage:** Simple and predictable distribution.

**Limitation:** It does not consider whether one server is currently busier or more powerful than another.

#### Weighted Round Robin

Each server receives a weight that represents its relative capacity. Servers with higher weights receive a larger proportion of requests.

**Advantage:** Better suited to environments containing servers with different capabilities.

**Limitation:** The weights must be configured and maintained appropriately.

#### Least Connections

New requests are directed toward the server currently handling the fewest active connections.

**Advantage:** Takes current connection counts into account.

**Limitation:** Connection count alone does not always represent CPU usage or processing load.

#### IP Hash

The client's IP address is used to determine which server receives the request.

**Advantage:** Can help maintain session persistence for clients.

**Limitation:** Uneven client IP distribution can produce an uneven workload.

## 4. Dynamic Load Balancing

Dynamic approaches use current server or network conditions when making routing decisions.

### Least Response Time

Traffic is directed toward servers responding most quickly.

**Benefit:** Can reduce user-perceived latency.

**Trade-off:** The load balancer must continually monitor response times.

### Resource-Based Distribution

This method considers resources such as CPU utilization, memory consumption, and available network bandwidth.

**Benefit:** Can make better use of available server capacity.

**Trade-off:** Monitoring multiple resources makes the implementation more complex.

### Adaptive Load Balancing

Adaptive systems can use live metrics and, in more advanced implementations, analytical or machine-learning techniques to predict changing traffic conditions.

**Benefit:** Can respond to changing workloads dynamically.

**Trade-off:** Requires additional infrastructure, monitoring, and implementation complexity.

## 5. Load Balancing by Network Layer

Different load-balancing solutions operate at different levels of the network stack.

### HTTP/HTTPS

This approach distributes web requests between application servers. Because the request contents can be inspected, features such as SSL termination, URL-based routing, and session persistence can be supported.

### Layer 4

Layer 4 balancing operates around TCP or UDP connections. Routing decisions can be based on information such as source/destination IP addresses and port numbers without examining application-level content.

### Layer 7

Layer 7 balancing works at the application layer and can inspect application information. For HTTP traffic, this makes routing based on headers, cookies, URLs, or other request attributes possible.

## 6. Load Balancing in Cloud Platforms

Cloud infrastructure changes dynamically as workloads grow or shrink, making traffic distribution particularly important.

Managed cloud options include:

- **AWS Elastic Load Balancer (ELB):** Provides managed application and network load-balancing capabilities within AWS.
- **Azure Load Balancer:** Provides load distribution and high-availability capabilities within Microsoft Azure.

## Summary

Load balancing helps applications use multiple servers efficiently instead of depending on a single machine. The appropriate method depends on factors such as server capacity, traffic behaviour, session requirements, network layer, and deployment environment.
