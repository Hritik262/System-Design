# System-Design Reference Notes

![System Design Overview](images/jj3A5N8.png)

## Table of Contents
- [Domain Name System](#domain-name-system)
- [Content Delivery Networks](#content-delivery-networks)
- [Load Balancer](#load-balancer)
- [Horizontal Scaling](#horizontal-scaling)
- [Application Layer](#application-layer-separation-from-web-layer)

## Domain Name System
A Domain Name System (DNS) translates a domain name such as www.example.com to an IP address.

DNS is hierarchical, with a few authoritative servers at the top level. Your router or ISP provides information about which DNS server(s) to contact when doing a lookup. Lower level DNS servers cache mappings, which could become stale due to DNS propagation delays. DNS results can also be cached by your browser or OS for a certain period of time, determined by the time to live (TTL).

- **NS record (name server)** - Specifies the DNS servers for your domain/subdomain.
- **MX record (mail exchange)** - Specifies the mail servers for accepting messages.
- **A record (address)** - Points a name to an IP address.
- **CNAME (canonical)** - Points a name to another name or CNAME (example.com to www.example.com) or to an A record.

Services such as CloudFlare and Route53 provide managed DNS services.

## Content Delivery Networks
A content delivery network (CDN) is a globally distributed network of proxy servers, serving content from locations closer to the user. Generally, static files such as HTML/CSS/JS, photos, and videos are served from CDN, although some CDNs such as Amazon's CloudFront support dynamic content. The site's DNS resolution will tell clients which server to contact.

Serving content from CDNs can significantly improve performance in two ways:
- Users receive content from data centers close to them
- Your servers do not have to serve requests that the CDN fulfills

### Pull CDNs
Pull CDNs grab new content from your server when the first user requests the content. You leave the content on your server and rewrite URLs to point to the CDN. This results in a slower request until the content is cached on the CDN.

A time-to-live (TTL) determines how long content is cached. Pull CDNs minimize storage space on the CDN, but can create redundant traffic if files expire and are pulled before they have actually changed. Sites with heavy traffic work well with pull CDNs, as traffic is spread out more evenly with only recently-requested content remaining on the CDN.

### Push CDNs
Push CDNs receive new content whenever changes occur on your server. You take full responsibility for providing content, uploading directly to the CDN and rewriting URLs to point to the CDN. You can configure when content expires and when it is updated. Content is uploaded only when it is new or changed, minimizing traffic, but maximizing storage.

Sites with a small amount of traffic or sites with content that isn't often updated work well with push CDNs. Content is placed on the CDNs once, instead of being re-pulled at regular intervals.

## Load Balancer
Load balancers distribute incoming client requests to computing resources such as application servers and databases. In each case, the load balancer returns the response from the computing resource to the appropriate client.

### Benefits
Load balancers are effective at:
- Preventing requests from going to unhealthy servers
- Preventing overloading resources
- Helping to eliminate a single point of failure

### Implementation
Load balancers can be implemented with:
- Hardware (expensive)
- Software such as HAProxy

### Load Balancing Algorithms
A load balancer is a software or hardware device that keeps any one server from becoming overloaded. A load balancing algorithm is the logic that a load balancer uses to distribute network traffic between servers (an algorithm is a set of predefined rules).

There are two primary approaches to load balancing:
- **Dynamic load balancing** uses algorithms that take into account the current state of each server and distribute traffic accordingly. 
- **Static load balancing** distributes traffic without making these adjustments. Some static algorithms send an equal amount of traffic to each server in a group, either in a specified order or at random.

Additional benefits include:
- **SSL termination** - Decrypt incoming requests and encrypt server responses so backend servers do not have to perform these potentially expensive operations
- **X.509 certificates** - Removes the need to install certificates on each server
- **Session persistence** - Issue cookies and route a specific client's requests to same instance if the web apps do not keep track of sessions

### Disadvantages of Load Balancers
- The load balancer can become a performance bottleneck if it does not have enough resources or if it is not configured properly.
- Introducing a load balancer to help eliminate a single point of failure results in increased complexity.
- A single load balancer is a single point of failure, configuring multiple load balancers further increases complexity.

### Load Balancer vs Reverse Proxy

#### Use Cases
- **Load Balancer**: Useful when you have multiple servers. Often, load balancers route traffic to a set of servers serving the same function.
- **Reverse Proxy**: Can be useful even with just one web server or application server, opening up the benefits described in the previous section.

#### Common Solutions
Solutions such as NGINX and HAProxy can support both layer 7 reverse proxying and load balancing.

#### Disadvantages of Reverse Proxy
- Introducing a reverse proxy results in increased complexity.
- A single reverse proxy is a single point of failure, configuring multiple reverse proxies (ie a failover) further increases complexity.

### Layer 4 Load Balancing (Transport Layer)

#### What it Looks At:
- IP addresses (source/destination)
- Port numbers (e.g., 80, 443)
- Does NOT inspect packet content

#### How it Works:
- Passes through the connection from client to server
- The load balancer itself does not terminate the connection
- Performs Network Address Translation (NAT)

#### Pros:
- **Fast**: Requires less processing
- **Low Resource Usage**: Doesn't consume much computing power
- **Basic Load Balancing**: Good for simple traffic distribution

#### Cons:
- **Less Intelligent**: Cannot make application-specific decisions
- **No Content-Based Routing**: Cannot route traffic based on URL or cookies

**Example**: Route all traffic on port 80 to servers A, B, C.

### Layer 7 Load Balancing (Application Layer)

#### What it Looks At:
- IP addresses, Port numbers, plus...
- HTTP headers
- URL path
- Cookies
- Actual message content

#### How it Works:
- Terminates the connection from the client (SSL termination often happens here)
- Reads and parses the full request message
- Establishes a new connection to the selected target server

#### Pros:
- **More Intelligent**: Can perform application-specific routing (URL, cookie-based)
- **Advanced Features**: Enables SSL termination, caching, content compression, URL rewriting, Web Application Firewall (WAF), etc.
- **Efficient Resource Utilization**: Directs traffic to the most appropriate server pools

#### Cons:
- **Can be Slower**: Deeper packet inspection and managing two connections adds overhead
- **Higher Resource Usage**: Requires more computing power

**Example**: If the request path is /images, send it to server A; if it's /api, send it to server B.

#### Key Difference:
- **Layer 4**: Handles traffic at the "connection" level. Focuses on "where it came from, where it's going" (IP/Port).
- **Layer 7**: Handles traffic at the "request content" level. Focuses on "what is being asked for" (URL, headers, cookies).

## Horizontal Scaling

### Definition
Adding more machines/servers to a system to distribute the workload and increase capacity. Often facilitated by load balancers.

### Compared to Vertical Scaling (Scaling Up)
- **Vertical Scaling**: Increasing resources (CPU, RAM, disk) of a single existing server. Limited by the maximum capacity of a single machine.
- **Horizontal Scaling**: Adding more instances of servers. Potentially limitless.

### Advantages
- **Cost-Efficient**: Uses cheaper, commodity hardware instead of expensive, specialized large machines.
- **Higher Availability/Resilience**: If one server fails, others continue operating, preventing service downtime.
- **Easier Talent Acquisition**: Easier to find engineers familiar with commodity hardware setups.

### Example
**E-commerce website**: Instead of upgrading one super-powerful server, you add multiple smaller, identical web servers behind a load balancer to handle increased user traffic. If one web server goes down, the load balancer directs traffic to the remaining ones.

### Disadvantages & Considerations

#### Increased Complexity
- Managing multiple server instances (deployment, configuration, monitoring) is more complex than managing one.
- Requires server cloning and consistent environment setup.

#### Stateless Servers Required
- Application servers must be stateless (not store any user-specific data like session info, user profiles locally).
- Reason: A user's subsequent request might go to a different server, which wouldn't have the necessary context if data was stored locally.

#### Centralized State Management
- User-related state (sessions, shopping cart data) must be stored in a centralized, shared data store.
- Examples: Databases (SQL, NoSQL), or persistent caches (Redis, Memcached).

#### Downstream Scaling
- As upstream application servers scale horizontally, downstream services (like databases, caches, message queues) must also be able to handle the increased number of simultaneous connections and requests. They might also need to scale horizontally or vertically.

## Application Layer (Separation from Web Layer)

### Concept
This involves separating the Web Layer (responsible for user interface, serving static assets like HTML/CSS, handling browser requests) from the Application Layer (where the core business logic, data processing, and API endpoints reside).

The Application Layer is sometimes also referred to as the "Platform Layer."

### Advantages

#### Independent Scaling & Configuration
- Allows the web and application layers to be scaled and configured independently.
- **Benefit**: If the website UI experiences high traffic (e.g., many users browsing), you can add more web servers without needing to scale the application servers. Conversely, if your APIs are heavily used for complex processing, you can scale only the application servers. This leads to more efficient resource utilization.

#### Modular Development & Growth
- Adding new API functionality often only requires adding or modifying application servers, without necessarily impacting the web servers.
- **Single Responsibility Principle**: This architecture encourages breaking down a large system into smaller, autonomous services (often called microservices), where each service has a single, well-defined responsibility.
- **Rapid Growth**: Smaller teams working on smaller, independent services can plan and execute changes more aggressively, leading to faster development and growth.

### Disadvantages & Considerations

#### Increased Complexity (Architectural & Operational)
- Moving from a monolithic system (where all functionality is in one large application) to an architecture with a separate Application Layer (especially with loosely coupled services like microservices) introduces significant complexity.
- Requires a different approach to architecture, operations, and development processes.

#### Deployment & Operations Challenges
- Managing, deploying, and monitoring multiple independent services (microservices) is inherently more complex than managing a single monolithic application.
- This often necessitates robust automation tools (e.g., CI/CD pipelines, container orchestration platforms like Kubernetes) for effective management.

### Microservices
Related to the "Application Layer" discussion are microservices, which can be described as a suite of independently deployable, small, modular services. Each service runs a unique process and communicates through a well-defined, lightweight mechanism to serve a business goal.

**Example**: Pinterest could have the following microservices: user profile, follower, feed, search, photo upload, etc.

### Service Discovery

#### Purpose
In distributed systems, services need to find and communicate with each other dynamically. Service Discovery solves this by helping services locate one another.

#### How it Works
- **Registration**: Services, when they start, register their names, IP addresses, and ports with a Service Discovery system.
- **Discovery**: Other services query this system to find the network location of the services they need to interact with.

#### Why it's Needed
Modern applications have dynamic services (IPs change, instances come and go), making manual configuration impractical.

#### Common Systems
- Consul
- Etcd (often used with Kubernetes)
- Zookeeper

#### Health Checks
- Service Discovery systems perform regular health checks on registered services to verify their integrity and availability.
- Typically done via an HTTP endpoint (e.g., /health).
- Unhealthy services are automatically removed from the discovery pool to prevent traffic from being routed to them.

#### Key-Value Store (Optional)
- Some systems like Consul and Etcd include a built-in key-value store.
- Useful for storing configuration values (e.g., database credentials, feature flags) and other shared application data.
