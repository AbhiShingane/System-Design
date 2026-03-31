# Load Balancer

## How Internet Communication Works

When a client communicates with a server over the internet — say, `linkedin.com` — the request first travels to a **DNS (Domain Name Server)** to resolve the domain into an IP address. Once the IP is returned, the client connects directly to the target server. DNS servers are typically managed by ISPs (Internet Service Providers), which can be local or global.

**Total request time = DNS resolution time + actual request time**

### DNS Caching

To reduce DNS lookup latency, IP addresses can be cached at multiple levels:

- Browser
- Operating System (OS)
- ISP
- DNS Server

---

## Why Load Balancers Are Needed

For high-traffic platforms like Facebook, Instagram, or LinkedIn, relying on a single IP address creates bottlenecks and risks. To handle this, companies deploy their services across **multiple IP addresses**. The DNS server returns a shuffled list of IPs so that traffic is distributed across servers rather than concentrated on one.

When a request is made:
1. It travels over the public network and reaches a **proxy server** (or load balancer).
2. The load balancer distributes it to one of the available **application servers**.

> **Load balancers evenly distribute incoming traffic across multiple application servers**, ensuring no single server is overwhelmed.

---

## Types of Load Balancers

### 1. Hardware Load Balancer
Dedicated physical devices designed for high-performance traffic distribution.

### 2. Software Load Balancer
Runs as software and is further categorized into:

| Type | Description |
|------|-------------|
| **L4 (Transport Layer)** | Operates on IP address and port number. Faster, but minimal request visibility. |
| **L7 (Application Layer)** | Has access to the full request data (headers, URL, cookies, etc.). More flexible but slightly slower. |

> **L4 load balancers** are generally faster since they only expose minimal information (IP + port), while **L7 load balancers** enable smarter routing decisions using full request context.

---

## Network Architecture

- The load balancer sits on the **public network** and is the entry point for all client requests.
- Application servers reside in a **private network**, hidden from the public.
- The load balancer handles **HTTPS** termination; communication between the load balancer and application servers typically uses **HTTP**.

---

## Service Discovery & Health Monitoring

The load balancer interacts with a **Service Registry**, which maintains:

- IDs and IP addresses of all registered services
- Current status of each service (`active` / `inactive`)

The load balancer continuously monitors service health using mechanisms like **heartbeats**. When a request arrives, the load balancer:

1. Performs **authentication & authorization**
2. Handles **SSL termination**
3. Queries **Service Discovery** for a list of healthy, active services
4. Routes the request based on the configured **load balancing strategy**

---

## Load Balancing Strategies

### 1. Round Robin

Requests are distributed to servers **one after another in a cyclic order**. Each server gets an equal turn, and after the last server is reached, the cycle restarts from the first.

**Advantages:**
- Simplest algorithm to implement
- Maintains fairness across servers
- Low overhead
- Quick response time for lightweight workloads

**Drawbacks:**
- Unaware of current server load
- Leads to inefficient resource utilization under uneven workloads

**Example:**  
Three servers — `S1`, `S2`, `S3`. Requests are routed as: `R1→S1`, `R2→S2`, `R3→S3`, `R4→S1`, and so on.

---

### 2. Least Connection

A more intelligent approach than Round Robin. The load balancer tracks the **number of active connections per server** and always routes the new request to the server with the **fewest active connections**.

**Advantages:**
- Smarter traffic routing
- Works well for long-lived connections (e.g., WebSockets, database connections)
- Performance improves as load patterns stabilize

**Drawbacks:**
- Does not account for task complexity — a server with fewer connections may still be heavily loaded
- Can cause an initial imbalance during startup

**Example:**  
`S1` has 5 active connections, `S2` has 3, and `S3` has 7. The next request goes to `S2`.

---

### 3. Weighted Round Robin / Weighted Least Connection

An advanced extension of Round Robin and Least Connection. Each server is assigned a **weight** based on its capacity or performance. Servers with higher weights receive proportionally more requests.

**Advantages:**
- Better resource utilization across heterogeneous servers
- Adaptable to different server capacities
- Supports priority-based routing
- Scales well in mixed-capacity environments

**Drawbacks:**
- More complex to configure and maintain
- Requires continuous resource monitoring to assign accurate weights

**Example:**  
`S1` has weight `3`, `S2` has weight `2`, `S3` has weight `1`. Out of every 6 requests: `S1` gets 3, `S2` gets 2, `S3` gets 1.

---

### 4. Hashing

Requests are mapped to servers using a **hash function** applied to request attributes (e.g., client IP, session ID, URL). The hash output determines which server handles the request, ensuring consistent routing for the same input.

**Common Hashing Methods:**

| Method | Description |
|--------|-------------|
| Division Remainder | `hash = key % number_of_servers` |
| Multiplicative Hash Function | Uses a multiplier constant for distribution |
| Universal Hashing | Randomized approach to reduce collision probability |
| Cryptographic Hashing | Uses MD5, SHA-256, etc., to resist targeted attacks |

**Advantages:**
- Enables session persistence (sticky sessions)
- Consistent mapping of requests to servers
- Simple to implement with division-based methods

**Drawbacks:**
- Hash **collisions** can route multiple keys to the same server
- Hash functions are **non-invertible**
- Limited security in non-cryptographic variants
- Does not maintain order

**Example:**  
Client IP `192.168.1.10` is hashed and mapped to `S2`. All subsequent requests from that IP consistently go to `S2`.

---

### 5. Random Allocation

Incoming requests are distributed **randomly** across available servers with no predefined pattern or tracking.

**Trade-offs:**
- **Simplicity vs. Efficiency** — Easy to implement but may cause uneven load distribution
- **Uniformity Over Time** — Statistically balanced over a large number of requests
- **No State Tracking Required** — Zero overhead for maintaining connection or load data

**Drawbacks:**
- Unpredictable load distribution in the short term
- No awareness of current server load
- Not suitable for **sticky sessions**

**Example:**  
Five requests arrive — they are randomly assigned: `R1→S3`, `R2→S1`, `R3→S3`, `R4→S2`, `R5→S1`.

---

### 6. Geolocation-Based Load Balancing

Traffic is routed based on the **geographic location** of the client. Requests are directed to the nearest or most appropriate server or data center to minimize latency.

**Trade-offs:**
- **Latency vs. Complexity** — Reduces latency significantly but adds routing complexity
- **Accuracy vs. Overhead** — More accurate geo-detection requires more processing
- **Cost vs. Performance** — Regional deployments improve performance but increase infrastructure cost

**Drawbacks:**
- **Geo-IP inaccuracy** — IP-based location detection can sometimes be incorrect
- **Mobile users** — Frequent location changes make routing less predictable
- **Increased complexity** — Requires maintaining regional infrastructure

**Example:**  
A user in Mumbai is routed to a data center in India (`DC-IN`), while a user in New York is routed to a US-based data center (`DC-US`).

---

### 7. Resource-Based Load Balancing

Routes requests based on the **real-time resource availability** of each server (e.g., CPU usage, memory, disk I/O). This ensures requests go to the server best equipped to handle them at that moment.

**Trade-offs:**
- **Efficiency vs. Complexity** — Highly optimized routing but requires sophisticated monitoring
- **Resource Type vs. Balance** — Balancing across multiple resource types (CPU, RAM, etc.) is non-trivial
- **Cost vs. Performance** — Advanced monitoring tools add infrastructure cost

**Drawbacks:**
- Requires **complex real-time monitoring** of each server
- The monitoring itself introduces **resource overhead**

**Example:**  
`S1` is at 90% CPU, `S2` is at 40% CPU, and `S3` is at 60% CPU. The next request is routed to `S2`.

---

### 8. Rate Limiting

> Rate limiting is not strictly a load balancing strategy, but it plays an important role in **managing traffic load** across application servers.

It limits the number of requests a client or service can make within a defined time window. It is commonly used to:

- Protect server resources from overload
- Prevent abuse (e.g., DDoS attacks, brute force)
- Ensure fair usage among all clients

**Example:**  
An API allows a maximum of `100 requests per minute` per client. Once exceeded, the client receives a `429 Too Many Requests` response until the window resets.

---

## Summary Table

| Strategy | Best For | Key Limitation |
|----------|----------|----------------|
| Round Robin | Uniform, stateless requests | Ignores server load |
| Least Connection | Long-lived connections | Ignores task complexity |
| Weighted Round/Least | Mixed-capacity servers | Requires weight tuning |
| Hashing | Session persistence | Hash collisions |
| Random | Simple, low-overhead systems | Unpredictable short-term balance |
| Geolocation | Global, latency-sensitive apps | Geo-IP inaccuracy |
| Resource-Based | CPU/memory-intensive workloads | Complex monitoring |
| Rate Limiting | Abuse prevention, fair usage | Not a distribution strategy |
