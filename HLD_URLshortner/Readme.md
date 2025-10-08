# High Level System Design For URL Shortner / Tiny URL
In today’s digital age, the need for an efficient and streamlined URL management system has become increasingly important. URL shortening services, such as Bit.ly, TinyURL, and ZipZy.in, play a crucial role by transforming lengthy web addresses into compact, easily shareable links.

## Requirements for a URL Shortener Service
### 1. Functional Requirements:
- The service must be able to generate a unique, shortened alias for any given long URL.
- When a user accesses a short link, the system should seamlessly redirect them to the original URL.
- All links should have an expiration mechanism, ensuring they become inactive after a predefined time period.
### 2. Non-Functional Requirements:
- The system must be highly available, as any downtime could disrupt URL redirection for users.
-  Redirection should occur in real-time, with minimal latency, to maintain a smooth user experience.
-  The generated short URLs should be unpredictable, preventing malicious users from guessing or accessing other links.

## Capacity Estimation
Before designing a URL shortening system, it’s crucial to clarify all assumptions and constraints.

Let’s consider an example to establish a realistic scale. Suppose Twitter, with around 300 million monthly users, uses this service, and only 10% of them actively shorten URLs — that’s approximately 30 million users per month, or around 1 million users per day. Each of these users would generate multiple read and write operations daily.

For our URL shortening service, we can generate short URLs using a 7-character code. Each character is selected from a set of 62 possible characters, including uppercase letters (A–Z), lowercase letters (a–z), and digits (0–9). For instance, a shortened URL could be: https://zpzy.in/redirecting?code=G7kLp9Q. This method allows us to create a vast number of unique, compact, and easily shareable links.

For the shortened URL format, we might use a pattern like us.com/uniquecode, where the unique code is 7 characters long. To estimate how much data we’ll store, we can use a data capacity model. Each entry in the database would typically include:
- Long URL: 2 KB (assuming up to 2048 characters)
- Short URL: 17 bytes
- Created At: 7 bytes
- Expires At: 7 bytes
- This gives a total of approximately 2.031 KB per shortened URL entry.
Using this model, we can project the total data storage required for one year or more, which is essential for planning database capacity, storage scaling, and overall system architecture.

When designing the database for a URL shortening service, it’s important to carefully consider all the attributes that need to be stored for each record. Typical fields include the original long URL, the shortened URL, the creation timestamp, and the expiration timestamp. Based on the earlier assumption of a maximum URL length and data size, we can estimate the total storage requirements. 

For example, 
- 30 million users per month :  the system would consume approximately 60.7 GB of data per month
-  which translates to about 0.7 TB per year and roughly 3.6 TB over five years.
## Low Level Design for URL shortner
<img width="1408" height="736" alt="Gemini_Generated_Image_sxrfbisxrfbisxrf" src="https://github.com/user-attachments/assets/a420a1f3-4cda-466a-897e-6a2b4497e13c" />
# Choosing Database for storing URL
When choosing a database for a URL-shortening service you have to weigh consistency guarantees against operational complexity.

- **Relational databases (RDBMS)** give you **ACID transactions** and built-in uniqueness guarantees, which make it straightforward to enforce that a short code maps to exactly one long URL — but **scaling an RDBMS** horizontally requires **sharding** and adds significant **operational overhead**.
- **NoSQL systems**, on the other hand, are designed for easy **horizontal scaling and high availability**; they make it simpler to add nodes when storage or throughput grows, but many NoSQL deployments are **eventually consistent** —which makes globally enforcing uniqueness and avoiding race conditions harder unless you explicitly use their conditional-write or strong-consistency features.

That **race condition** is the real danger: 
if multiple app servers independently generate the same 7-character code and each checks the database before inserting, both may see **“not present”** and write **conflicting mappings, corrupting data** so the same short code later resolves to the wrong long URL. 

Approaches that rely purely on **random-generation** without atomic, server-side guarantees are **fragile in distributed setups**. 

Even hashing the long URL (for example **MD5** and taking the first 7 characters) reduces but does not eliminate collisions; you still need a deterministic, atomic way to detect and resolve conflicts so two different long URLs never permanently claim the same short code.

In practice you pick a pattern that matches your scale and risk tolerance. For strong correctness and simpler logic, use a datastore with atomic **“insert-if-not-exists”** semantics or enforce a unique constraint inside a transactional RDBMS so collisions are rejected at the database level and the application can retry deterministically. For very large scale, use a distributed unique-id generator (auto-increment shards, Snowflake-style IDs, or centralized counters) and encode those IDs in Base62 to produce compact, collision-free short codes; combine this with consistent hashing and load balancing to spread traffic and queries. If you opt for NoSQL, rely on its conditional write/CAS features or add a lightweight coordination layer to guarantee uniqueness. Whatever route you choose, design the flow so the database — not racing application servers — is the ultimate arbiter of uniqueness, and build a simple collision-handling retry strategy for the rare conflicts that still occur.

To generate **unique short URLs** efficiently, we can use a **Base62 encoding algorithm**, 
- which converts a numeric ID into a short alphanumeric string using digits (0–9), lowercase letters (a–z), and uppercase letters (A–Z).
- This approach ensures compact, URL-friendly codes while maintaining uniqueness. For instance, the following sample function demonstrates this logic:
<img width="974" height="353" alt="image" src="https://github.com/user-attachments/assets/2dff5f94-a77d-47cb-a801-bc72f3fee5f7" />
This function takes a decimal number and returns its Base62-encoded form, which can then be used as the unique identifier in shortened URLs.

To design a truly **scalable and efficient URL-shortening system**, we can adopt a **counter-based** approach instead of relying on random numbers or hashing. In this technique, a **global counter variable** is maintained, which increments by one each time a new short URL is created. Because this counter is synchronized and unique, every new request automatically receives a distinct numeric value. This number is then passed through a **Base62 encoding function** to generate a compact, **alphanumeric identifier**, which becomes the **short URL suffix**.

The key advantage of this approach is that it eliminates the need to check the database for **duplicate entries**, as the counter inherently ensures uniqueness. It also simplifies **concurrency management, reduces latency, and supports effortless horizontal scalability** — making it far more reliable and efficient than random or hash-based techniques in high-traffic systems.

While using a counter-based approach for generating unique short URLs is simple and scalable, it introduces a **critical challenge** when the system expands horizontally. In a single-server setup, maintaining one centralized counter works well. However, as soon as **multiple application servers** come into play, relying on a single counter creates a **Single Point of Failure (SPOF)**. If the server hosting the counter or database goes down, the entire system loses its ability to generate new short URLs. Attempting to maintain multiple independent counters across regions—such as assigning ranges like Counter1 for IDs 0–1,000,000 and Counter2 for 1,000,001–100,000,000—adds **complexity and synchronization issues**.

**What happens if one counter reaches its maximum range or fails? Who monitors and resets it safely?**

Managing this manually quickly becomes unreliable and error-prone.
To overcome these limitations, we can use **Apache Zookeeper**, a **distributed coordination service** that efficiently **manages synchronization** across multiple servers. 

**Zookeeper** acts as a centralized controller, ensuring that distributed systems remain consistent and coordinated. 
#### Analogy
Think of it as a class teacher managing multiple students (servers) — the teacher keeps track of which students are present, assigns unique IDs to each, and maintains an updated record of all activities. 

Similarly, Zookeeper helps servers register, coordinate, and communicate with one another. It ensures that each server can generate unique counters, detect failures, and maintain consistency without human intervention. By delegating coordination to Zookeeper, developers can focus on business logic while ensuring **high availability, fault tolerance, and smooth scalability** across distributed environments.

In a distributed URL-shortening system, Apache Zookeeper plays a vital role in maintaining coordination and ensuring smooth operation across multiple servers. Zookeeper assigns each server a **unique ID**, selects a **master node**, and stores configuration information that all hosts can access. It also provides a **coordinated, database-like file system**, where servers can read and write essential metadata in a synchronized manner.

Previously, when using a counter-based system to generate unique short URLs, we faced challenges such as deciding the counter range for each server, handling hardware failures, and managing new server additions. These issues made counter management complex and prone to errors. Zookeeper effectively solves this by offering a **centralized coordination service** that automatically manages configuration, synchronization, and node tracking. Whenever a server is added or removed, Zookeeper updates the system state accordingly, ensuring no overlap or collisions in counter allocation. By integrating Zookeeper, the distributed system achieves high availability, consistent coordination, and fault tolerance, allowing developers to focus on core functionality rather than low-level synchronization problems.

To achieve a **high-performance, highly scalable, and collision-free architecture**, the system must be designed to efficiently manage load distribution, counter allocation, and data consistency. A **load balancer (LB)** —either hardware or software—is used to evenly distribute incoming traffic across multiple application servers. Each app server is responsible for generating short URLs, and to ensure uniqueness, every server maintains a dedicated counter range for ID generation.

When new application servers are added, they immediately **register with Apache Zookeeper**, which assigns each a unique ID and marks them as active in the system. Zookeeper is pre-configured with batches of counter ranges (for example, 1–2 lakh, 2–3 lakh, etc.) derived from the total possible 7-character Base62 combinations (≈3.5 trillion unique codes). As each server joins, it requests a counter range from Zookeeper, which allocates an **unused range** from its bucket. The server then converts these counter values to Base62 strings to generate unique short URLs—completely eliminating the need to check for duplicates in the database. This results in **zero collision probability** and significantly improved **system performance**.

When users access short URLs, the application first checks the cache (such as Redis or Memcached) to resolve the mapping instantly. If the cache entry has expired, only then does it query the database, ensuring **low latency and high throughput**. If any app server fails mid-operation (for example, between the 3L–4L counter range), Zookeeper is notified and safely **discards the remaining counter range**, preventing reuse and maintaining data integrity. Similarly, when a server exhausts its assigned range, it requests a new range from Zookeeper, which assigns the next available block (e.g., 5L–6L).

This approach guarantees that no two servers ever generate overlapping IDs, ensuring collision-free, distributed operation. With the help of **Zookeeper coordination, load balancing, caching, and either NoSQL or RDBMS databases**, the system achieves strong consistency, high availability, and seamless scalability, capable of supporting billions of short URL mappings efficiently.

## APIs :
To operate a URL-shortening service effectively, we need to design a set of core APIs that handle the main functionalities. The two fundamental APIs are:

### 1. Create Tiny URL  
This API takes a long URL as input, generates a unique short code (using techniques like Base62 encoding or counter-based generation), and stores the mapping in the database. It then returns the shortened URL to the user.

### 2. Get Long URL (via Short URL) 
This API retrieves the original long URL when a user accesses the short link, enabling seamless redirection to the intended destination.

In addition to these, it’s important to track **analytics and system performance** to understand how users are interacting with the service. This can include metrics such as the number of clicks, user locations, popular links, and system health indicators. For this, we can integrate Google Analytics, set up our own analytics service, or use **Elasticsearch and related open-source tools** to collect and visualize data. Incorporating analytics not only helps monitor application performance and user engagement but also provides valuable insights for scaling, optimizing system resources, and enhancing the overall user experience.
## High Level Design for URL Shortner
<img width="1472" height="704" alt="Gemini_Generated_Image_ytdl6nytdl6nytdl" src="https://github.com/user-attachments/assets/ef1f2a5d-c04a-4d4f-96a2-c8092e6c65ed" />





This function takes a decimal number and returns its Base62-encoded form, which can then be used as the unique identifier in shortened URLs.
