# System-Design Reference Notes

![System Design Overview](images/jj3A5N8.png)

## Table of Contents
- [Domain Name System](#domain-name-system)
- [Content Delivery Networks](#content-delivery-networks)
- [Load Balancer](#load-balancer)
- [Reverse Proxy Explained](#reverse-proxy-explained)
- [Service Discovery](#service-discovery-brief-notes)
- [Horizontal Scaling](#horizontal-scaling)
- [Application Layer](#application-layer-separation-from-web-layer)
- [Databases](#databases)
- [Caching](#caching)

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

## Reverse Proxy Explained

### Real-world Analogy: The Restaurant Maître d' (Head Waiter)

Imagine a large, popular restaurant with many different kitchen stations (Italian, Chinese, Indian, etc.).

- **You (Client)**: The customer placing an order.
- **Maître d' (Reverse Proxy)**: You don't go directly to the kitchens. You give your order to the maître d'.
  - The maître d' takes your order.
  - Decides which kitchen station (backend server) can best handle it (e.g., Italian kitchen for an Italian dish).
  - Forwards your order to that specific kitchen.
  - Receives the prepared dish from the kitchen.
  - Serves the dish to you.
- **Kitchen Stations (Backend Servers)**: These are the actual servers that process the request (cook the food).

### Benefits of the Maître d' (Reverse Proxy) in this analogy:
- **Security**: Hides the internal structure of the kitchen (backend servers) from the customer.
- **Load Balancing**: Distributes orders among different kitchens to prevent any single kitchen from being overloaded.
- **Efficiency/Caching**: If a "special dish" is ordered, the maître d' might already know its details and can provide a quick response without asking the kitchen (serves cached content).
- **Simplicity/Abstraction**: Provides a single entry point (the maître d') despite a complex internal setup (multiple kitchens).
- **SSL Termination**: Can handle encrypted communication from the customer and then talk to the kitchen normally.

### Technical Explanation: Reverse Proxy

A Reverse Proxy is a server that sits in front of one or more web servers (backend servers) and forwards client requests to them. The client makes requests to the reverse proxy, which then acts as an intermediary, directing traffic to the appropriate backend server.

#### How it works:
1. **Client Request**: A user's browser sends a request (e.g., www.example.com).
2. **Request to Reverse Proxy**: This request first arrives at the reverse proxy (which is configured for www.example.com).
3. **Reverse Proxy Decision**: The reverse proxy receives the request and, based on its configuration and rules, decides which backend server should handle it. This decision can be based on:
   - **Load Balancing**: Which backend server has the least load.
   - **Request Path/URL**: Directing /api/users to a user service, and /images to an image service.
   - **Health Checks**: Ensuring the chosen backend server is healthy and operational.
4. **Forwarding Request**: The reverse proxy forwards the client's request to the selected backend server.
5. **Backend Response**: The backend server processes the request and sends the response back to the reverse proxy.
6. **Sending Response to Client**: The reverse proxy then sends this response back to the original client. The client remains unaware of which specific backend server processed its request.

#### Key Benefits of a Reverse Proxy (Technical):
- **Load Balancing**: Distributes incoming client requests across multiple backend servers to ensure no single server is overwhelmed, improving performance and availability.
- **Security**: Hides the IP addresses and internal architecture of backend servers, protecting them from direct attacks. It can also help mitigate DDoS attacks.
- **SSL/TLS Termination**: Handles the encryption/decryption of client connections (HTTPS). This offloads the computational burden of SSL from backend servers.
- **Caching**: Can store frequently accessed content, reducing the need for backend servers to regenerate it and speeding up response times for clients.
- **Compression**: Can compress responses before sending them to clients, saving bandwidth and improving page load times.
- **URL Rewriting**: Allows modifying incoming URLs before forwarding them to backend servers, offering flexibility in URL management.
- **Centralized Logging and Monitoring**: All traffic passes through the reverse proxy, making it an ideal point for collecting access logs and monitoring system health.

#### Common Reverse Proxy Software:
- Nginx
- Apache (using mod_proxy)
- HAProxy
- Cloudflare (also acts as a Content Delivery Network)

#### Flow Clarification:
The correct sequence is:

**Client -> Reverse Proxy -> Backend Server**

Load Balancing is a function or feature that the Reverse Proxy performs (if configured and if multiple backend servers are available). It's not a separate component that comes after the reverse proxy in the direct request path.

So, a more detailed understanding of the flow is:

**Client -> Reverse Proxy (where Load Balancing decisions are made, if applicable) -> Backend Server(s)**

In essence, a reverse proxy acts as a sophisticated gateway, improving the security, performance, and scalability of web applications.

## Service Discovery (Brief Notes)

### What it is:
- A mechanism for services in a distributed system to find each other dynamically without hardcoding addresses.
- Acts like a "phone book" or "directory" for services.

### Why it's needed:
- **Dynamic Environments**: Service IPs/ports change frequently (e.g., in microservices, cloud deployments).
- **Avoids Hardcoding**: Eliminates the need to manually update configurations when services scale or move.
- **Resilience**: Helps avoid unhealthy service instances.

### How it works (Core components):
1. **Service Registration**: When a service starts, it registers its address (IP:Port) and name with a Service Discovery System (SDS).
2. **Health Checks**: The SDS periodically checks registered services (e.g., via HTTP endpoint) to verify they are alive and healthy. Unhealthy services are removed from the available list.
3. **Service Lookup/Discovery**: When a client service needs to communicate with another service, it queries the SDS for a healthy instance of that service.
4. **SDS Response**: The SDS provides the address of an available, healthy service instance to the client.

### Key Benefits:
- **Flexibility**: Services can be deployed, scaled, and moved without manual configuration changes.
- **Resilience/Fault Tolerance**: Unhealthy instances are automatically detected and removed.
- **Decoupling**: Services don't need to know the physical location of other services.

### Common Tools:
- Consul
- Etcd
- Zookeeper

### Summary of the Flow (with Load Balancing):
1. Client -> Requests Reverse Proxy (entry point).
2. Reverse Proxy -> Uses Service Discovery info to find available Service A instances.
3. Reverse Proxy -> (Load Balances) and forwards request to one Service A instance.
4. Service A -> Uses Service Discovery to find available Service B instances.
5. Service A -> (Load Balances) and requests one Service B instance.
6. Service B -> Responds to Service A.
7. Service A -> Responds to Reverse Proxy.
8. Reverse Proxy -> Responds to Client.

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

## Databases

### Importance of Database Selection
Picking the right database for a system is an important decision, as it can have a significant impact on the performance, scalability, and overall success of the system. Some of the key reasons why it's important to pick the right database include:

- **Performance**: Different databases have different performance characteristics, and choosing the wrong one can lead to poor performance and slow response times.

- **Scalability**: As the system grows and the volume of data increases, the database needs to be able to scale accordingly. Some databases are better suited for handling large amounts of data than others.

- **Data Modeling**: Different databases have different data modeling capabilities and choosing the right one can help to keep the data consistent and organized.

- **Data Integrity**: Different databases have different capabilities for maintaining data integrity, such as enforcing constraints, and can have different levels of data security.

- **Support and maintenance**: Some databases have more active communities and better documentation, making it easier to find help and resources.

### SQL vs NoSQL Databases Notes

#### 1. SQL Databases (Examples: MySQL, PostgreSQL, Oracle, SQL Server)

**Best Suited For:**
- **Structured, Relational Data**: Data that adheres to a predefined schema and has well-defined relationships between different data entities.
- **Examples**: E-commerce platforms (customer, order, product relationships), banking systems, financial records, inventory management, traditional CRM systems.

**Key Characteristics:**
- **Fixed/Rigid Schema**: Data must conform to a strict, predefined structure (tables, columns, data types) before it can be stored. Schema changes typically require altering the table structure, which can be complex and time-consuming for large datasets.
  - **Example**: Defining a Users table with id (INT PRIMARY KEY), name (VARCHAR(100)), email (VARCHAR(100) UNIQUE).
- **ACID Transactions** (Atomicity, Consistency, Isolation, Durability): SQL databases strongly guarantee ACID properties, which are crucial for data integrity and reliability.
  - **Atomicity**: All operations within a transaction either complete successfully or none of them do.
  - **Consistency**: A transaction brings the database from one valid state to another.
  - **Isolation**: Concurrent transactions execute independently without interfering with each other.
  - **Durability**: Once a transaction is committed, its changes are permanent, even in the event of system failures.
  - **Example**: A bank transfer where money is debited from one account and credited to another; both actions must succeed or neither.
- **Complex Queries & Joins**: They excel at performing complex queries and efficient JOIN operations across multiple related tables. SQL (Structured Query Language) is a powerful language for data manipulation and retrieval.
  - **Example**: Joining Customers, Orders, and Products tables to find out which customer bought which product on what date.

**Advantages**: Strong data consistency, excellent for managing complex relationships, high data integrity, mature technology with wide support.

**Disadvantages**: Can be challenging to scale horizontally (scaling typically involves vertical scaling initially), less flexible schema can hinder rapid development or evolution with changing data requirements.

#### 2. NoSQL Databases (Examples: MongoDB, Cassandra, Redis, Neo4j)

**Best Suited For:**
- **Unstructured, Non-Relational Data**: Data that doesn't have a fixed schema, or whose structure changes frequently. Often categorized by data model (e.g., Document, Key-Value, Column-Family, Graph).
- **Examples**: Social media feeds, user activity logs, real-time analytics, IoT sensor data, content management systems, user profiles in web/mobile apps.

**Key Characteristics:**
- **Flexible/Dynamic Schema (Schema-less)**: No predefined schema is required. Each record (e.g., a "document" in MongoDB) can have its own unique structure. This allows for rapid development and easy adaptation to evolving data models.
  - **Example (MongoDB Document)**:
  ```json
  {
    "name": "Alice",
    "age": 30,
    "city": "New York"
  }
  // Another document in the same collection
  {
    "name": "Bob",
    "email": "bob@example.com",
    "hobbies": ["coding", "gaming"]
  }
  ```
  Notice how Alice has city and Bob has email and hobbies, without a rigid table structure.
- **High Scalability & Performance**: Designed for horizontal scaling (distributing data across many servers or clusters), making them ideal for handling massive amounts of data and high throughput. They can offer high performance for specific access patterns.
- **Big Data and Real-time Web Applications**: Commonly used in environments where large volumes of data need to be stored and accessed quickly, such as real-time recommendation engines or user activity streams.

**Advantages**: Handles unstructured data perfectly, high horizontal scalability, high performance for specific data access patterns, rapid development due to schema flexibility.

**Disadvantages**: Generally do not provide full ACID guarantees (often opting for "eventual consistency" for better availability/performance), complex joins are difficult or must be handled at the application layer, data consistency is not as strong as in SQL.

#### 3. Choosing Between SQL and NoSQL (Depends on Use Case):

**Choose SQL if:**
- You require structured data with complex relationships.
- Strong data integrity and ACID transactions are paramount (e.g., financial systems).
- You anticipate needing complex queries and joins.
- Your data model is stable and unlikely to change frequently.

**Choose NoSQL if:**
- You need to store large amounts of unstructured or semi-structured data.
- High scalability (especially horizontal) and performance are your top priorities.
- Your data model is flexible or frequently evolving.
- You are building big data or real-time web applications.

**Conclusion:**
Neither SQL nor NoSQL databases are universally "better." Both have their distinct strengths and weaknesses. The appropriate choice hinges on the specific needs of your project, the nature of your data, scalability requirements, and consistency demands. Many modern applications adopt a hybrid approach (polyglot persistence), utilizing both SQL and NoSQL databases for different parts of their data storage strategy.

### NoSQL Databases

#### Key Value Store
A key-value store generally allows for O(1) reads and writes and is often backed by memory or SSD. Data stores can maintain keys in lexicographic order, allowing efficient retrieval of key ranges. Key-value stores can allow for storing of metadata with a value.

Key-value stores provide high performance and are often used for simple data models or for rapidly-changing data, such as an in-memory cache layer. Since they offer only a limited set of operations, complexity is shifted to the application layer if additional operations are needed.

#### Document Store (Concise Notes)
- **Definition**: Document-centric database, stores whole objects as documents (JSON, XML, BSON).
- **Structure**: Each document is self-contained with its own internal structure.
- **Querying**: Rich APIs/languages to query within the document's structure.
- **Grouping**: Documents organized into collections, tags, etc.
- **Schema Flexibility**: Crucial! No fixed schema; documents in same group can have different fields.
- **Overlap**: Blurring lines with Key-Value stores having metadata features.

#### Graph Databases: Comprehensive Notes

##### 1. Introduction and Core Principles
**What is it?** A Graph Database is a type of NoSQL (non-relational) database that stores data in the form of nodes (entities) and edges (relationships). It is fundamentally designed to emphasize and prioritize the connections between data points.

**Core Components:**
- **Nodes (Vertices):**
  - These are your data records. Think of them as real-world entities like a person, a place, a thing, or an event.
  - Each node has a unique identity.
  - They can possess properties (stored as key-value pairs). For example, a Person node might have properties like name: "Alice" and age: 30.
- **Edges (Relationships/Arcs):**
  - These represent the connections between two nodes. This is where the true power of graph databases lies.
  - Each edge has a direction (e.g., A -> B means a relationship from A to B).
  - Each edge has a type that describes the nature of the relationship (e.g., FRIENDS_WITH, LIKES, WORKS_FOR).
  - Edges can also have their own properties. For instance, a FRIENDS_WITH edge might have a property since: 2018.

**Purpose:** Graph databases are optimized for data models where complex relationships, numerous foreign keys, or many-to-many relationships are paramount. They eliminate the need for costly and complex JOIN operations often found in relational databases.

##### 2. Characteristics of Graph Databases
- **Relationship-Centric:** Relationships are treated as "first-class citizens," not derived from data. This makes traversing (navigating through) chains of relationships extremely fast.
- **Schema-Flexible:** Most graph databases are schema-less or schema-flexible. You can add new nodes and edges, and modify their properties "on the fly" without rigid pre-definition.
- **Query Performance:** When data exhibits a high degree of interconnectedness, graph queries (for pattern matching, shortest path, recommendations, etc.) offer superior performance compared to relational databases, which would require multiple, expensive JOINs for similar tasks.
- **Intuitive Modeling:** Graph models naturally reflect real-world scenarios, making it easier for developers to understand and represent complex data structures.

##### 3. Use Cases
- **Social Networks:** Ideal for storing and querying Users (nodes) and their FRIENDS_WITH, FOLLOWS, LIKES (edges) relationships.
  - Example: "Who are Alice's friends' friends?" or "Which people share common interests?"
- **Recommendation Engines:** Generate recommendations based on relationships between Users (nodes), LIKES, PURCHASED, VIEWED (edges), and Products (nodes).
  - Example: "What other items did users who bought this item also purchase?"
- **Fraud Detection:** Identify irregular patterns (anomalies) in transaction networks. Analyze Accounts, Transactions, IP Addresses (nodes) and INITIATED_FROM, RECEIVED_BY (edges).
  - Example: "Did this transaction occur between two accounts that were previously involved in fraudulent activity?"
- **Knowledge Graphs:** Represent entities (concepts, places, people) and their semantic relationships within a large domain. The Google Knowledge Graph is a well-known example.
- **Network and IT Operations:** Track network topology, dependencies, and configuration changes within complex IT infrastructures.
- **Identity and Access Management:** Manage access relationships between users, roles, permissions, and resources.

##### 4. Advantages and Challenges
**Advantages:**
- **High Performance for Connected Data:** Avoids complex joins, leading to faster execution of deep and wide queries.
- **Flexibility:** Easy to evolve the schema as data requirements change.
- **Intuitive Data Modeling:** Directly models real-world networks.
- **Agile Development:** Easier to incorporate new features and data types.

**Challenges:**
- **Maturity:** Newer than relational databases; the talent pool and tooling, while growing, might be less established (though this is rapidly changing).
- **Learning Curve:** Graph-specific query languages (like Cypher, Gremlin) may require initial learning effort.
- **Appropriate Use Case:** Not the best solution for all types of data. If your data is largely "non-relational" but consists of a large collection of simple, isolated documents, a document database might be more suitable.
- **Tooling and Ecosystem:** The ecosystem is still evolving compared to the mature SQL environment.
- **Scalability for Massively Dense Graphs:** Managing and distributing extremely large and dense graphs can present unique challenges (though modern graph DBs are making significant progress here).

##### 5. Query Languages
Graph databases utilize specialized query languages:
- **Cypher:** Neo4j's proprietary declarative query language, known for its readability and expressive power.
- **Gremlin:** The traversal language of the Apache TinkerPop framework, which is imperative and JVM-based. Many graph databases support it.
- **SPARQL:** A query language for RDF (Resource Description Framework) data, used in the semantic web.

##### Conclusion
Graph databases excel when the relationships between entities are the most critical aspect of your data. They effectively solve the "joining problem" inherent in relational databases and provide high performance and flexibility for a wide array of complex scenarios, from social networks to fraud detection.

#### Wide Column Store: Visualization Notes (with Example)
- **Analogy**: Imagine a very large, flexible spreadsheet.

**Basic Structure:**
- **Rows**: Identified by a unique Row Key.
- **Columns**: Grouped into Column Families.
- Think of Column Families as sections/categories within the spreadsheet (e.g., "User Profile", "Order Details").
- Each column has a full name: Column Family:Column Qualifier (e.g., User_Profile:name).

**Sparsity (Emptiness):**
- Unlike traditional spreadsheets, a row doesn't need to have data for every column in every column family.
- Only columns with stored values actually exist for that row. Many cells can be "empty" or non-existent.

**Example:**
- Row Key "user_1" might have User_Profile:name, User_Profile:age, User_Profile:city.
- Row Key "user_2" might only have User_Profile:name, User_Profile:email (no age or city for this user).

**Timestamping:**
- Every value in a column (each "cell") has an associated, hidden timestamp.
- Used for versioning (tracking changes over time) and conflict resolution).
- Example: If User_Profile:name for user_1 changes from "Alice" to "Alicia", a new entry with the new name and a newer timestamp is stored for that column.

**Data Access:**
- Access data using Row Key to retrieve an entire row.
- Or, use Row Key + Column Family:Column Qualifier to fetch a specific column's value.
- Example: To get user_1's age, you would query for Row Key "user_1" and Column "User_Profile:age".

**Flexibility:**
The sparse nature and lack of a strict schema per row make it highly flexible for varying data structures across different entries.

### SQL Databases

#### Replication
Replication is the process of copying data from one database to another. Replication is used to increase availability and scalability of databases. There are two types of replication: master-slave and master-master.

**Master-slave Replication:**
The master serves reads and writes, replicating writes to one or more slaves, which serve only reads. Slaves can also replicate to additional slaves in a tree-like fashion. If the master goes offline, the system can continue to operate in read-only mode until a slave is promoted to a master or a new master is provisioned.

**Master-master Replication:**
Both masters serve reads and writes and coordinate with each other on writes. If either master goes down, the system can continue to operate with both reads and writes.

#### Sharding: Comprehensive Notes

##### 1. What is Sharding?
**Core Idea:** Sharding is a database scaling technique where data is distributed across multiple smaller, separate, and independent databases. Each of these smaller databases (called a shard) manages only a subset of the total data.

**Why is it necessary?** When your data volume or transaction load becomes too large for a single database server (no matter how powerful it is) to handle efficiently, Sharding comes into play. It's a method of horizontal scaling, meaning you add more machines to distribute the load, rather than upgrading a single, more powerful machine.

##### 2. How Does Sharding Work?
- **Data Distribution:** Data is partitioned and distributed to different shards based on a shard key. The shard key is a specific column (e.g., user_id, product_id) whose value determines which shard a particular piece of data will reside on.
  - Example: If user_id is the shard key, a hash value or range of the user_id will decide which shard the user's data goes to.
- **Independence:** Each shard is a complete, independent database instance. It has its own CPU, memory, and storage. This means that if one shard fails, the others continue to operate.
- **Cluster:** These individual shards collectively form a larger database cluster that manages the entire dataset.
- **Shard Router/Query Coordinator:** An additional layer exists to intercept incoming queries and intelligently redirect them to the correct shard. Clients typically interact with this router, not directly with the individual shards.

##### 3. Visualization and Example: "Users Database" (E-commerce Platform)
Consider a large e-commerce website with millions of users. Managing all user data (profiles, orders, preferences) on a single database server becomes challenging. Query performance and write throughput become significant bottlenecks.

**Without Sharding:**
```
                  +--------------------------------+
                  | Main Database Server (Monolith)|
                  | (Handles ALL Users' Data)      |
                  | CPU, RAM, Storage (Can become  |
                  |     a bottleneck at scale)     |
                  +---------------+----------------+
                                  |
                                  | High Traffic
                  +---------------+----------------+
                  | Website/Application Servers    |
                  +--------------------------------+
```

**Problem:** As the number of users grows, this "Main Database Server" will become a bottleneck. Queries will slow down, write latencies will increase, and the risk of server failure due to overload will rise.

**With Sharding:**
We partition the user data based on user_id across multiple shards. Let's assume we've set up 3 shards:

```
                  +------------------------------------------------+
                  |           Shard Router / Query Coordinator     |
                  | (Intelligently directs requests to the correct |
                  |  shard based on the shard key)                 |
                  +--------------------+---------------------------+
                                       |
                                       |
           +---------------------------+---------------------------+
           |                           |                           |
  +--------v--------+       +----------v----------+       +--------v--------+
  |    Shard 1      |       |      Shard 2        |       |    Shard 3      |
  | (Users 1-1M)    |       | (Users 1M-2M)       |       | (Users 2M-3M)   |
  | DB Server A     |       | DB Server B         |       | DB Server C     |
  | (Only a subset  |       | (Only a subset      |       | (Only a subset  |
  |  of the data)   |       |  of the data)       |       |  of the data)   |
  +-----------------+       +---------------------+       +-----------------+
```

**Shard Key Example (user_id as Shard Key):**
- If user_id is our chosen shard key:
  - Data for User IDs from 1 to 1 Million is stored on Shard 1.
  - Data for User IDs from 1 Million 1 to 2 Million is stored on Shard 2.
  - Data for User IDs from 2 Million 1 to 3 Million is stored on Shard 3.
- This distribution strategy is often based on range (as shown above) or hashing (e.g., user_id % num_shards).

**Query Example:**
- When an application requests data for user_id = 500,000, the Shard Router will direct this request only to Shard 1.
- When a request comes for user_id = 1,500,000, it is routed only to Shard 2.
- This way, each shard only needs to handle queries and writes for its specific subset of data, dramatically improving performance and reducing the load on individual servers.

##### 4. Advantages of Sharding:
- **Reduced Read and Write Traffic:** Each shard handles only a fraction of the total data, significantly reducing the I/O load and processing for read and write operations.
- **Increased Cache Hits:** Because each shard's dataset is smaller, it's more likely that frequently accessed data will reside in the server's cache, leading to faster data retrieval.
- **Reduced Index Size:** Each shard maintains an index only for its portion of the data. Smaller indexes are faster to search and manage, boosting query performance.
- **Improved Overall Performance:** All the above factors combine to drastically improve query execution times and overall system responsiveness.
- **Enhanced Fault Tolerance (Fault Isolation):** If one shard fails (e.g., due to hardware malfunction), only the data on that specific shard becomes unavailable. The rest of the system (other shards) continues to operate normally, preventing a complete system outage.
  - (Note: To prevent data loss on a failed shard, some form of replication within each shard's cluster is crucial.)
- **Increased Write Throughput (Parallel Writes):** Similar to federation, there's no single central master serializing all writes. Each shard can process writes for its data partition independently and in parallel, leading to a much higher overall write throughput.
- **Scalability (Horizontal Scaling):** As the number of users or data grows, you can easily add more shards to the cluster, distributing the load further. This allows the system to scale almost indefinitely without upgrading existing hardware to larger, more expensive machines (vertical scaling).

##### 5. Challenges of Sharding:
- **Complexity:** Implementing, managing, and maintaining a sharded database system is significantly more complex than a single monolithic database.
- **Shard Key Selection:** Choosing the right shard key is critical. A poor choice can lead to:
  - Hotspots: Uneven data distribution where one shard receives a disproportionately large amount of traffic, becoming a bottleneck.
  - Data Imbalance: Some shards having much more data than others.
- **Resharding:** As data grows or access patterns change, you might need to re-distribute data across new or existing shards. This process (resharding or rebalancing) can be complex, time-consuming, and may require downtime.
- **Distributed Transactions:** If a single transaction needs to modify data that spans across multiple shards, managing transaction atomicity and consistency (ACID properties) becomes much harder and requires specialized solutions.
- **Joins Across Shards:** Queries that require joining data from different shards can be complex and less efficient than joins within a single database.
- **Operational Overhead:** Backup, recovery, and monitoring across multiple independent shards require more sophisticated tools and procedures.

##### Conclusion:
Sharding is a powerful and essential technique for handling very large datasets and high-traffic applications. It enables horizontal scaling, dramatically improves performance, and provides increased fault tolerance. However, its implementation comes with significant operational complexities and requires careful planning, especially regarding shard key selection and managing distributed transactions.

#### Federation (Functional Partitioning): Comprehensive Notes

##### 1. What is Federation?
**Core Idea:** Federation, also known as Functional Partitioning, is a database scaling technique where you split (divide) your database system into multiple separate databases based on its functionality or service. This means each database handles a specific type of data or an application feature.

**Distinct from Sharding:** In sharding, we divide a single type of data (like users) across multiple databases. In federation, we place different types of data (like users, products, orders) into entirely separate databases. It's a form of vertical scaling where data is distributed according to its category.

##### 2. How Does Federation Work?
- **Service-Oriented Approach:** Each database is dedicated to a specific microservice or application module. The application's logic dictates which request goes to which database.
- **Logical Separation:** There is a logical separation of data. User data resides only in the users database, product data only in the products database, and order data only in the orders database.
- **Independent Databases:** Each federated database is an independent entity. It can have its own server, its own schema, and its own configuration. It can be optimized specifically for its functionality.
- **Decoupling:** Different functional areas (modules) become decoupled from each other.

##### 3. Visualization and Example: E-commerce Website
Imagine a large e-commerce website with many functionalities: managing users, displaying product listings, and running forum discussions.

**Without Federation:**
```
                  +--------------------------------+
                  | Main Database Server (Monolith)|
                  | (All User, Product, Forum Data)|
                  | CPU, RAM, Storage (Overloaded) |
                  +---------------+----------------+
                                  |
                                  | All Traffic (User logins, product views, forum posts)
                  +---------------+----------------+
                  | Website/Application Servers    |
                  +--------------------------------+
```

**Problem:** If everything resides in a single database, every activity—from user login to new product uploads and forum posts—will hit this single database. This will lead to an overwhelming load, performance degradation, and difficult maintenance.

**With Federation:**
We've split the database system into three separate databases based on functionality: Users, Products, and Forums.

```
                  +------------------------------------------------+
                  |           Application Backend (Logic)          |
                  | (Routes requests to the appropriate database)  |
                  +--------------------+---------------------------+
                                       |
                                       |
           +---------------------------+---------------------------+
           |                           |                           |
  +--------v--------+       +----------v----------+       +--------v--------+
  |  Users Database |       | Products Database   |       | Forums Database |
  | (User Profiles, |       | (Product Info,      |       | (Forum Posts,   |
  |  Auth Data)     |       |  Inventory)         |       |  User Threads)  |
  | DB Server A     |       | DB Server B         |       | DB Server C     |
  +-----------------+       +---------------------+       +-----------------+
```

**Example Scenarios:**
- **User Login:** When a user logs in, the application's backend logic will direct that request only to the Users Database.
- **Product Browsing:** When a user browses products, the request will go only to the Products Database.
- **New Forum Post:** When a user creates a new post in the forum, the request will go only to the Forums Database.
- **User's Past Order History:** If a user wants to view their order history, the application will need to fetch the user ID from the Users Database and then retrieve orders for that user ID from the Orders Database. This requires coordination at the application layer.

##### 4. Advantages of Federation:
- **Reduced Read and Write Traffic:** Each database only has to handle queries and writes related to its specific functionality. This significantly reduces the load on each database and increases overall throughput.
- **Less Replication Lag:** If you are using replication for high availability (creating copies of data), smaller databases mean that changes take less time to apply. This keeps replica databases (the copies) more up-to-date.
- **More Cache Hits:** Each database's dataset is specific to its functionality and is a smaller portion of the overall data. Smaller datasets can fit more easily into memory, which improves cache locality and leads to more cache hits (meaning a higher chance of finding requested data in the cache). This results in faster queries.
- **Improved Overall Performance:** All the points mentioned above combine to significantly boost query performance within each functional area.
- **Parallel Writes:** In federation, there isn't a single central master database serializing all writes. Each functional database handles writes for its data independently, enabling parallel writes and increasing the overall throughput (the number of operations that can be performed in a given time).
- **Enhanced Fault Tolerance (Failure Isolation):** This is a significant advantage. If, for any reason, the Forums Database crashes, the Users and Products functionalities will still be fully available. The entire system will not go down; only the forum-related functionality will be affected.
- **Easier Maintenance and Optimization:** It's easier to optimize each database for its specific functionality and access patterns. Schema changes or maintenance tasks only impact that specific functional area, not the entire system.
- **Technology Flexibility:** You can choose the most suitable database technology for each functional database (e.g., a highly relational SQL DB for users, a Document DB for a product catalog, a Columnar DB for real-time analytics).

##### 5. Challenges of Federation:
- **Data Consistency:** If a single operation needs to update data across multiple federated databases (e.g., when a user places an order, user info needs updating and order info needs saving), maintaining distributed transactions and data consistency becomes complex. Atomicity (all or nothing) becomes difficult.
- **Complex Joins/Cross-Database Queries:** Queries that need to fetch and join data from multiple functional databases can be very complex and inefficient. The application layer might have to manually fetch data from different databases and combine it, which increases latency.
- **Increased Network Latency:** Storing data on separate servers increases the number of inter-service communications (requests between different databases), which can slightly increase overall latency, especially if servers are geographically distant.
- **Operational Complexity:** Managing, monitoring, backing up, and restoring multiple separate databases is significantly more complex than doing so for a single database. It requires specialized tools and expertise.
- **Application Logic Complexity:** The application layer now needs to know which data resides in which database and route requests accordingly. This routing logic must be developed within the application code.

##### Conclusion:
Federation is a very effective strategy for scaling large and feature-rich applications. It improves performance in specific functional areas, provides fault isolation, and simplifies maintenance. It's a natural fit for microservices architecture. However, distributed data management comes with its own complexities that need careful consideration, especially when cross-functional queries or transactions are required.

#### Denormalization
Denormalization attempts to improve read performance at the expense of some write performance. Redundant copies of the data are written in multiple tables to avoid expensive joins. Some RDBMS such as PostgreSQL and Oracle support materialized views which handle the work of storing redundant information and keeping redundant copies consistent.

Once data becomes distributed with techniques such as federation and sharding, managing joins across data centers further increases complexity. Denormalization might circumvent the need for such complex joins.

#### SQL Tuning Notes

##### 1. What is SQL Tuning?
- **Definition**: SQL tuning is the systematic process of identifying, diagnosing, and optimizing SQL statements (queries) that do not meet desired performance standards.
- **Objective**: To make database operations faster, more efficient, and more reliable, thereby improving the overall responsiveness and scalability of an application.
- **Methodology**: Involves analyzing queries, understanding their execution plans, and making targeted changes to reduce resource consumption (CPU, memory, disk I/O) and execution time.

##### 2. Breadth of the Topic:
- SQL tuning is a vast and complex subject, with numerous books and resources dedicated to it.
- **Reasons for Complexity**:
  - Varies significantly across different database systems (e.g., MySQL, PostgreSQL, Oracle, SQL Server).
  - Optimization strategies depend heavily on specific application requirements and data access patterns.
  - Encompasses various aspects, including database schema design, indexing strategies, hardware resources, and even operating system configurations.

##### 3. Identifying Performance Bottlenecks:
The initial crucial step in SQL tuning is to effectively pinpoint where performance issues lie. This is primarily done through two methods:

**A. Benchmarking**:
- **Purpose**: To simulate and evaluate how the database system performs under high-load situations (heavy traffic or numerous concurrent requests).
- **Benefit**: Uncovers system limitations, bottlenecks, and potential performance degradation points that might not be apparent under normal operating conditions.
- **Tools**: ApacheBench (ab), JMeter, Gatling, k6.
- **Example Usage**: Running `ab -n 1000 -c 100 http://yourwebsite.com/api/products` to simulate 100 concurrent users making 1000 requests, and observing if the database slows down.

**B. Profiling**:
- **Purpose**: To track and analyze specific performance issues within a live or test database environment. It helps identify which queries are consuming the most time or resources.
- **Benefit**: Pinpoints the exact problematic queries or operations.
- **Tools & Techniques**:
  - **Slow Query Log** (available in MySQL, PostgreSQL, SQL Server): A database feature that automatically logs queries exceeding a predefined execution time limit (e.g., 2 seconds).
    - Example (MySQL): Checking `long_query_time` variable to see the threshold.
  - **Execution Plans** (e.g., EXPLAIN in PostgreSQL/MySQL, SHOWPLAN in SQL Server): Shows how the database engine plans to execute a query. This reveals information like index usage, number of rows scanned, and table join order.
    - Example: `EXPLAIN SELECT * FROM orders WHERE order_date < '2023-01-01';`
  - **Database Monitoring Tools**: Such as Prometheus, Grafana, DataDog, New Relic, which track real-time performance metrics (CPU usage, memory, disk I/O, active connections).

##### 4. Common Optimizations Following Benchmarking and Profiling:
The insights gained from benchmarking and profiling often lead to the following types of optimizations:

**A. Indexing**:
- **Key Optimization**: Often the most effective. Missing or inappropriate indexes can severely degrade query performance.
- **Scenario**: If a WHERE clause (e.g., `WHERE order_date < '2023-01-01'`) is used on an unindexed column, the database may perform a full table scan. Adding an index allows direct access to relevant rows.

**B. Query Rewriting/Optimization**:
- **Technique**: Modifying queries to be more efficient.
- **Examples**:
  - Selecting only necessary columns (e.g., `SELECT name, email` instead of `SELECT *`).
  - Converting subqueries to JOINs or vice versa based on performance characteristics.
  - Effective use of LIMIT clauses for pagination.

**C. Database Schema Changes**:
- **Addressing Structural Issues**: Sometimes performance problems stem from the database's design.
- **Examples**:
  - Denormalization (as discussed previously) to avoid expensive joins.
  - Correcting inappropriate data types (e.g., using INT instead of VARCHAR for numeric IDs).

**D. Hardware Upgrades**:
- **Physical Resource Enhancement**: If software-level tuning has been exhausted, the system might require more robust hardware (e.g., more CPU, RAM, faster storage like SSDs).

**E. Database Configuration Tuning**:
- **Server Parameter Adjustment**: Fine-tuning internal database server parameters (e.g., buffer pool size, cache sizes, connection limits) to optimize resource allocation and behavior.

**F. Application-Level Caching**:
- **Reducing Database Load**: Implementing caching at the application layer for frequently accessed data reduces the number of direct database queries.

##### Conclusion:
SQL tuning is an iterative and ongoing process. By leveraging benchmarking and profiling tools to identify bottlenecks, and then applying optimizations such as indexing, query rewriting, schema adjustments, and hardware upgrades, the ultimate goal is to enhance user experience and ensure efficient utilization of system resources.

## Caching

Caching is the process of storing frequently accessed data in a temporary storage location, called a cache, in order to quickly retrieve it without the need to query the original data source. This can improve the performance of an application by reducing the number of times a data source must be accessed.

### Caching Strategies

#### Refresh Ahead
The cache automatically refreshes any recently accessed cache entry prior to its expiration. This strategy can help avoid cache misses, but may lead to unnecessary cache updates if data is not accessed again before expiration.

##### Refresh-ahead Caching Strategy Notes

**1. What is Refresh-ahead?**
- **Core Concept**: A caching strategy where a cache entry is automatically updated or refreshed before it actually expires.
- **Mechanism**: You configure the cache system to proactively fetch fresh data from the original data source for recently accessed cache entries (items that have been queried or used recently) just prior to their configured expiration time.
- **Objective**: To ensure that fresh data is always available in the cache when requested by a client, minimizing perceived latency.

**2. Advantage of Refresh-ahead:**
- **Reduced Latency vs. Read-through/Cache-Aside**:
  - In Read-through or Cache-Aside strategies, when a cache entry expires or is not found (cache miss), the application must query the original data source. This introduces latency as the user waits for the data to be fetched from the slower source.
  - With Refresh-ahead, if the cache can accurately predict which items are likely to be needed in the future (i.e., those recently accessed), it refreshes them asynchronously in the background before they are requested.
  - **Result**: When a user or application requests the data, it finds a fresh copy already in the cache, leading to virtually zero perceived latency from a cache miss, thus significantly improving response times.
  - **Example**: For frequently accessed data like stock prices or news headlines, the cache can refresh this data just before it expires. When a user checks the latest prices/news, the data is already up-to-date in the cache.

**3. Disadvantage of Refresh-ahead:**
- **Risk of Inaccurate Prediction leading to Reduced Performance**:
  - The primary drawback of Refresh-ahead arises when the cache fails to accurately predict which items will actually be needed in the future.
  - If the cache refreshes data that is not subsequently requested by a client, it can lead to a reduction in overall performance rather than an improvement.
  - **Reasons for Performance Reduction**:
    - **Wasteful Resource Usage**: The system expends resources (network bandwidth, database CPU/I/O, cache server processing) making unnecessary calls to the original data source to refresh data that nobody will use.
    - **Increased Load on Source**: The original data source (e.g., database) experiences higher load from these proactive, potentially unneeded refresh operations.
    - **Stale Data Risk (in specific scenarios)**: If data is refreshed but not used, and then the original data changes again shortly after the proactive refresh, the cache might still hold outdated data if it's not accessed again until much later. (Though less common than the resource waste problem).

**Analogy**:
- Imagine a buffet where food is prepared (data fetched).
- **Standard Buffet (Read-through/Cache-Aside)**: A dish runs out (cache expires/misses). The chef only starts cooking a new batch after a customer asks for it. The customer waits.
- **Refresh-ahead (with good prediction)**: The chef notices that Pasta is always popular. He sees a dish of Pasta getting low, and before it's completely empty, he starts cooking a new batch. When the next customer asks for Pasta, it's already ready. No wait.
- **Refresh-ahead (with bad prediction)**: The chef thinks the Fish dish will be popular and starts cooking a new batch before the old one is empty. But no one asks for Fish. Resources (ingredients, cooking time) were wasted on a dish that wasn't eaten, potentially at the expense of cooking another popular dish faster.

**Conclusion**:
Refresh-ahead is a powerful strategy for improving user-perceived latency, especially for predictable access patterns. However, its effectiveness heavily relies on the ability to accurately predict future data access. If predictions are poor, the overhead of unnecessary refreshes can negate performance benefits and increase resource consumption.

#### Write-Behind
In this strategy, the application writes data to the cache, which then asynchronously writes the data back to the data source. This can improve write performance as the application doesn't need to wait for the data to be written to the data source.

##### How Write-behind Works:
- Add/update entry in cache
- Asynchronously write entry to the data store, improving write performance

![Write-Behind Caching](images/writeBehind.png)

##### Disadvantages of Write-behind:
- There could be data loss if the cache goes down prior to its contents hitting the data store.
- It is more complex to implement write-behind than it is to implement cache-aside or write-through.

#### Write-through
The application writes data to both the cache and the data source simultaneously. This ensures data consistency between the cache and the data source, but may result in slower write operations.

#### Cache Aside
The application is responsible for reading and writing from the data source and the cache. When reading data, the application first checks the cache; if the data is not found (a cache miss), it reads from the data source and then writes to the cache. When writing data, the application writes directly to the data source and invalidates the corresponding cache entry.

### Caching Locations

#### Client Caching
Caching that occurs on the client side, such as browser caching. This can include HTML pages, JavaScript files, stylesheets, and images.

#### CDN Caching
Content Delivery Networks cache content at various points of presence (PoPs) around the world, reducing latency for users by serving content from a location closer to them.

#### Web Server Caching
Web servers can cache dynamic content to reduce the load on application servers and databases. This can include full page caching, fragment caching, and object caching.

#### Database Caching
Databases often include their own caching mechanisms to improve performance, such as query caches and buffer pools.

#### Application Caching
Application-level caching involves storing the results of expensive operations, such as complex database queries or API calls, in memory for quick access. This can be implemented using tools like Redis or Memcached.
