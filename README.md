<div align="center">

#  High-Level System Design Case Studies

**Worked-out HLD designs for popular real-world systems, each with its own detailed README.**

![Focus](https://img.shields.io/badge/Focus-High%20Level%20Design-blue)
![Case Studies](https://img.shields.io/badge/Case%20Studies-4-brightgreen)
![Status](https://img.shields.io/badge/Status-Growing-orange)

</div>

---

##  About

This repository is my collection of **High-Level Design (HLD) case studies**. Each folder takes one well-known system, breaks the problem down, and walks through how I would design it to be scalable, reliable and fast.

>  Open any case study folder below to read its full write-up.

---

##  Case Studies

| # | Case Study | What the system does | Folder |
| :-: | --- | --- | --- |
| 1 |  **Cab Booking System** | Ride-hailing platform: rider requests, driver matching, live location and trip lifecycle | [`HLD_CabBookingSystem`](./HLD_CabBookingSystem) |
| 2 |  **Dropbox** | Cloud file storage and sync across devices | [`HLD_Dropbox`](./HLD_Dropbox) |
| 3 |  **URL Shortener** | Turns long URLs into short links and redirects them at scale | [`HLD_URLshortner`](./HLD_URLshortner) |
| 4 |  **WhatsApp** | Real-time messaging with delivery status and media sharing | [`HLD_Whatsapp`](./HLD_Whatsapp) |

---

##  How Each Case Study Is Structured

Every design follows the same approach, so they are easy to compare:

1. **Problem statement**: what we are building and why
2. **Requirements**: functional and non-functional (scale, latency, availability)
3. **Capacity estimation**: users, traffic, storage and bandwidth
4. **API design**: the key endpoints
5. **Data model**: entities, schema and database choice
6. **High-level architecture**: components and how they talk to each other
7. **Deep dives**: the hard parts of that particular system
8. **Trade-offs and bottlenecks**: what I chose, what I gave up, and why

>  Adjust this list to match the sections your case study READMEs actually use.

---

##  Core Concepts You'll See Across These Designs

| Concept | Why it matters |
| --- | --- |
| **Load balancing** | Spread traffic across servers and avoid single points of failure |
| **Caching** | Cut latency and database load for read-heavy paths |
| **Database choice** (SQL vs NoSQL) | Pick storage that fits the access pattern |
| **Sharding and replication** | Scale storage and stay available under failure |
| **Message queues** | Decouple services and handle spikes asynchronously |
| **CDN and object storage** | Serve files and media efficiently |
| **WebSockets / real-time delivery** | Push updates to clients instantly |
| **Consistency vs availability (CAP)** | Make deliberate trade-offs per feature |

---

##  Repository Structure

```
High-Level-System-Design-Case-Studies/
├── HLD_CabBookingSystem/   # Ride-hailing design + README
├── HLD_Dropbox/            # Cloud storage and sync design + README
├── HLD_URLshortner/        # URL shortening service design + README
├── HLD_Whatsapp/           # Messaging system design + README
└── README.md               # You are here
```

---

##  Feedback

Spotted something that could be designed better, or want to suggest a system to cover next? Feel free to open an [issue](../../issues). Constructive discussion on trade-offs is always welcome.

---


