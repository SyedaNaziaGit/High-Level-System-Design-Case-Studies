# High Level System Design of Cab Booking system [UBER] 
## 1. Requirements  For Uber System Design
#### a. Functional Requirements

The system should provide users with a seamless and intuitive cab booking experience. 
- Users must be able to view all available cabs, along with their estimated arrival times (ETA) and minimum fare.
-  Once a suitable cab is selected, users should be able to book it for their desired destination.
-  After booking, users should have real-time visibility of the driver’s location.
-  Additionally, the system should allow users to cancel their rides at any point before or during the trip, ensuring flexibility and control over their travel plans.

 #### b. Non-Functional Requirements

To deliver a robust service,
- the system must ensure high availability, so users can reliably access the platform at all times. 
- It should exhibit high reliability, guaranteeing consistent and accurate ride booking and tracking.
- The architecture should be highly scalable to handle increasing numbers of users, rides, and concurrent requests. 
- The system should maintain low latency, providing near-instant responses for search, booking, and tracking features to enhance the user experience.

## 2. Capacity Estimation
#### Assumptions:
- Number of  Active Users on application  = 5 Millionon
- Number of Drivers  = 200,000  
- Average number of Rides Daily = 1 Million
- Average number of Actions performed by one user  = 5
- Number of Requests to handle daily = 5 * 1 Million = 5 Million
- Number of  requests per second our system  = 5*10^6  / (24 * 60 *60) = approx 58 requests per second
#### Storage Estimation
- Storage on every message on average = 500 bytes
- Storage Space Required Everyday = 58 * 500 = 2.32 GB 

## Uber Low Level System Design
We are all familiar with Uber and its ride-hailing services. A user can request a ride through the application, and within minutes, a nearby driver arrives to take them to their destination.

In its early days, Uber was built using a **Monolithic** architecture. The system consisted of a single backend service, a frontend service, and one centralized database. Python and its frameworks were used for development, with SQLAlchemy serving as the ORM layer for database interactions.

This architecture worked well when Uber operated in a few cities with a relatively small number of trips. However, as Uber expanded into multiple cities, the team began encountering significant challenges. Scaling the monolithic system became increasingly difficult, and maintaining reliability and performance under growing traffic loads was a struggle.

In 2014, Uber decided to transition to a **Service-Oriented Architecture (SOA)**. This approach allowed them to break down the monolithic system into smaller, independent services that could be developed, deployed, and scaled separately. The new architecture not only improved the scalability and reliability of the ride-hailing service but also enabled Uber to expand into food delivery, cargo, and other on-demand services.

A core functionality of Uber’s platform is to efficiently match riders with available cabs. To achieve this, the system is designed around two distinct services within its architecture:

- **Supply Service**
This service manages all cab-related operations, including tracking available drivers, their locations, and their statuses.

- **Demand Service**
This service handles rider-related operations, such as receiving ride requests, managing user preferences, and tracking active trips.

By separating these responsibilities into dedicated services, Uber can independently scale and optimize the supply and demand sides of the platform, ensuring faster and more reliable matching for riders and drivers.

<img width="1344" height="768" alt="HLDimage1" src="https://github.com/user-attachments/assets/6097762a-8152-4a37-b1aa-4515b333b98d" />

### Brief OverView 
The cab booking system can be viewed as a dynamic interaction between supply and demand, where the platform continuously balances rider requests with available drivers in real time. Uber’s Dispatch System acts as a real-time marketplace that intelligently matches riders to nearby cabs, ensuring efficient allocation of resources. At the core of this system lies **DISCO (Dispatch Optimization System)**, which operates primarily on geospatial and location-based data. It leverages Google’s S2 library, a powerful tool that models the Earth’s surface as a sphere and divides it into small, hierarchical cells—each roughly 1 km in size. These cells, when combined, form a complete digital map, and every cell is assigned a unique identifier. This structure allows Uber to efficiently distribute and store location data across a distributed system. Using these unique cell IDs, the platform can quickly locate and retrieve specific data from the relevant server. Additionally, **consistent hashing** based on cell identifiers ensures balanced data distribution and quick access, while the S2 library enables precise geographical coverage and mapping for any given region or trip.

**For example**, imagine drawing a circle on a map to identify all available drivers within a specific area. Using the **Google S2 library**, we can define a radius—say 2 to 3 kilometers—and the system will automatically filter and retrieve all the map cells that fall within that circular region. Since each cell has a unique identifier, the system can quickly access the relevant data and compile a list of all active drivers or “supplies” present within those cells. Once this data is gathered, the platform can perform further computations, such as estimating the **ETA (Estimated Time of Arrival)** by calculating the shortest distance between each driver and the passenger. This approach enables the system to efficiently match riders with nearby drivers or display the number of available cars in a given region, ensuring real-time, location-aware decision-making.

Every four seconds, each cab continuously transmits its live location data to **Kafka REST APIs**, forming the backbone of Uber’s real-time tracking and dispatch system. This data flow begins with the cab sending its location through a **Web Application Firewall (WAF)**, which acts as the first line of defense by filtering out malicious traffic — such as requests from blocked IPs, bots, or regions where Uber services are not yet available. From there, the request passes through a **Load Balancer (LB)** that distributes incoming traffic efficiently across multiple servers. Depending on the configuration, load balancing can operate at different layers — for example, Layer 3 (IP-based routing), Layer 4 (transport-level routing), or Layer 7 (application-level routing). Once the request reaches Kafka, the cab’s location is updated every four seconds, and a copy of this data is simultaneously stored in a **Database** and sent to **DISCO (Dispatch Optimization System)** to maintain an accurate view of all available drivers.

Given that thousands of cabs are operating in a city, this results in a massive stream of location updates being ingested by Kafka every few seconds. Kafka acts as a **High-Throughput Buffer**, allowing other components to consume this continuous flow of data efficiently — one consumer being the database for persistence, and another being DISCO, which uses the live data to optimize rider-driver matching.

To enable seamless two-way communication between apps and servers, Uber relies on **WebSockets**, which maintain a persistent, asynchronous connection between the client (cab app or user app) and the backend. This ensures that updates — such as cab movements, ETA changes, or new ride requests — can be instantly pushed without needing repeated HTTP requests. WebSockets, along with the **Node.js** framework, form the foundation of Uber’s event-driven, non-blocking architecture — ideal for handling high-frequency, asynchronous communication at scale.

The **DISCO (Dispatch Optimization Component)** itself is built using Node.js, leveraging its asynchronous and event-driven capabilities to handle real-time events efficiently. To serve users across multiple regions (region1, region2, region3, etc.), DISCO employs consistent hashing, which distributes workloads evenly across servers arranged in a **Ring structure**. This ensures scalability and fault tolerance — when a new server is added or removed, the load automatically redistributes with minimal disruption. DISCO also utilizes **RPC (Remote Procedure Calls)** for inter-server communication within the ring and implements the **SWIM (gossip) protocol**, allowing each server to be aware of others’ responsibilities and states. This decentralized awareness enables smooth scaling and quick recovery — if one server fails or a new one joins, others automatically adjust, ensuring Uber’s dispatch system remains highly available, resilient, and efficient.

### Working
When a user requests a ride in real time, the process begins seamlessly through Uber’s highly distributed architecture. As soon as the user initiates the request in the app, it is sent via a **WebSocket connection** , which ensures instant, bi-directional communication with the server. The request first reaches the Demand Service within DISCO (Dispatch Optimization System). The **Demand Service** interprets the user’s ride preferences—such as vehicle type (Mini, SUV, Pool, or full cab) and seating requirements—and then forwards these specifications to the **Supply Service**, which manages the available driver network.

At this point, the Supply Service uses the Google S2 library to determine the precise cell ID corresponding to the user’s pickup location. Each cell represents a small geographical area on the map, allowing Uber to efficiently manage and query location-based data. Based on the cell ID, the Supply Service identifies the correct server within the consistent hashing ring—a distributed cluster of servers, each responsible for a set of cells or regions. For instance, if the user’s location corresponds to cell ID 5, and that cell falls under the responsibility of the server in Region 6, the request is directed to that specific server.

The Region 6 server then analyzes the map to list all available drivers (supplies) within that area. It calculates the Estimated Time of Arrival (ETA) for each nearby driver, sorts the list to prioritize the closest options, and sends this data back to the client via the WebSocket connection. The system then dispatches ride requests to a few of the nearest drivers simultaneously. As soon as one of them accepts, the ride is confirmed and assigned to the user in real time.

If no suitable cab is found within the immediate cell, the Supply Service intelligently expands the search to nearby cells by leveraging RPC (Remote Procedure Calls) between neighboring servers within the ring. This collaborative inter-server communication ensures that the user gets the fastest possible match, maintaining Uber’s commitment to real-time responsiveness and reliability, even at massive global scale.

The **Supply Service** plays a crucial role in connecting available drivers to matching ride requests from the Demand Service. Once potential matches are identified, the Supply Service sends notifications to nearby drivers whose availability, vehicle type, and proximity meet the rider’s requirements. As Uber expands to new cities or regions, the system must dynamically handle increasing traffic and data volume. To achieve this, Uber’s **ring-based architecture (Ring BOB)** allows seamless scaling by adding new servers to the ring. When new servers are introduced, Ring BOB automatically recognizes newly created **cell IDs** generated from the mapping data and redistributes responsibilities among all servers, ensuring balanced load distribution. Similarly, when a server is removed or fails, the system intelligently reshuffles or reassigns cell responsibilities to other available servers—maintaining uninterrupted service and computation efficiency.

Uber’s **geospatial design** is powered by the **Google S2 library**, which divides the Earth’s surface into small, hierarchical cells. This enables the platform to efficiently locate cabs near a user’s pickup point by referencing specific cell IDs rather than scanning large map areas. For route and ETA calculations, Uber initially relied on Mapbox and later integrated Google’s **mapping APIs**, which provide precise distance, traffic, and travel time estimates between pickup and destination points.

To enhance the user experience further, Uber also identifies preferred **access points—locations** that riders frequently use to begin or end trips, such as entry or exit gates in large campuses, malls, or airports. By learning from repeated booking patterns, the system automatically suggests these access points for smoother pickups. The **ETA (Estimated Time of Arrival)** is calculated by drawing a circular zone around the pickup area, analyzing nearby drivers’ routes through real road networks, and factoring in travel time, traffic conditions, and cost. This combination of geospatial intelligence, load balancing, and predictive learning ensures Uber delivers accurate, efficient, and scalable real-time ride-matching worldwide.

## Database Scaling
In its early days, Uber relied on **PostgreSQL**, a traditional relational database, to store critical information such as GPS coordinates and ride details. While this setup worked initially, the system eventually faced scalability challenges due to the massive increase in data volume and the need for high-frequency updates—such as receiving location data from every cab every four seconds. To overcome these limitations, Uber transitioned to a **NoSQL database** architecture built on top of MySQL, designed to be **schemaless, horizontally scalable**, and capable of handling distributed data efficiently across multiple regions.

This evolution allowed Uber to support millions of concurrent read and write operations happening in real time without sacrificing performance or availability. Given the nature of Uber’s workload—where ride requests, driver updates, and trip tracking occur continuously—the database must remain highly readable and writable, ensuring zero downtime even under heavy traffic. The schemaless design offers flexibility in handling dynamic and rapidly changing data structures, while horizontal scaling ensures that data can be distributed across regional data centers closer to users. This proximity reduces latency and enhances performance, enabling a seamless experience for both riders and drivers. In essence, Uber’s database system is optimized for **speed, scalability, and availability**, supporting the demanding real-time operations of a global ride-hailing platform.

## Analytics
Analytics plays a pivotal role in Uber’s ecosystem, transforming vast streams of raw operational data into meaningful insights that help improve efficiency, reduce costs, and enhance both driver and rider satisfaction. By analyzing customer patterns, driver behavior, and trip performance, Uber can continuously optimize its dispatch algorithms, pricing models, and overall system performance to deliver a smoother and more reliable experience.

To support this, Uber uses a sophisticated **data analytics pipeline** that integrates multiple technologies. Initially, data is stored across **databases (RDBMS)** for transactional purposes and then periodically transferred to large-scale data processing platforms like **Hadoop, Hive, and HDFS (Hadoop Distributed File System)**. These tools allow for batch analytics on massive datasets, enabling the company to extract insights such as regional demand trends or driver performance metrics. For real-time analytics, Uber consumes data directly from **Kafka**, ensuring that time-sensitive metrics—such as surge pricing triggers or supply-demand fluctuations—can be acted upon instantly.

Uber’s analytics capabilities also extend to its **maps and ETA (Estimated Time of Arrival)** calculations. By combining **historical map data** with **real-time telemetry**, Uber refines its routing algorithms, traffic predictions, and speed estimations. This hybrid approach helps build more accurate and dynamic maps that account for real-world driving conditions. Uber also employs AI-based **simulations** to predict ETA more accurately and efficiently during dispatch operations.

Another critical area of analytics is **fraud detection**, where Uber leverages **machine learning models** to identify anomalies in payment patterns, incentive usage, and suspicious trip activities. These models use historical and behavioral data to detect issues like fake rides, account compromises, or incentive abuse, ensuring platform integrity and trust.

### Logging And Monitoring

For logging and monitoring, Uber follows a **service-oriented architecture**, where every microservice maintains its own logs. These logs are streamed to centralized systems through Kafka and then visualized using the **ELK stack (Elasticsearch, Logstash, Kibana)** or tools like Grafana for real-time dashboards. This setup enables Uber’s engineering teams to monitor application health, detect system errors, and perform analytics on infrastructure performance.

Overall, Uber’s analytics ecosystem—powered by **Hadoop, Hive, Kafka, Spark, and Storm frameworks**—enables the company to turn trillions of data points into actionable intelligence, driving continuous innovation, operational excellence, and customer satisfaction at global scale.

Although a **complete data center failure** is an extremely rare event, Uber’s system architecture is designed to handle such scenarios gracefully without disrupting ongoing trips. To achieve this, Uber maintains a **backup data center** that mirrors all the critical components required to keep rides operational, ensuring business continuity even during major outages. However, instead of continuously copying all data from the primary center—which would be inefficient and costly—Uber employs a smarter mechanism where the **driver app itself acts as a temporary data server** during an outage.

Every time a transaction or API call occurs between the driver app and the data center, the app stores a state digest, a lightweight but unique identifier that tracks the current state of the trip and the data exchanged. This state digest ensures that the app always knows how much information it has synchronized with the central system.

In the unlikely event that the main data center goes offline, the driver app continues to operate using its locally cached state. Once the system detects the failure, all communication is automatically rerouted to the **backup data center**. When connectivity is restored, the app syncs its stored state digest with the backup center, updating any pending trip or ride data seamlessly.

This distributed and fault-tolerant design ensures that even during large-scale infrastructure failures, **ongoing rides are not interrupted**, data consistency is maintained, and both drivers and riders experience minimal service disruption — demonstrating Uber’s strong commitment to **resilience and high availability** in its global platform.


