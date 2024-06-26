# [System Design Interview - An Insider's Guide (vol 1 & 2)](https://bytebytego.com/courses/system-design-interview)
These notes are based on the System Design Interview books - [vol 1](https://www.goodreads.com/book/show/54109255-system-design-interview-an-insider-s-guide) and [vol 2](https://www.goodreads.com/book/show/60631342-system-design-interview-an-insider-s-guide).

[Links](https://github.com/alex-xu-system/bytebytego/blob/main/system_design_links.md)

blog.bytebytego.c o m

Instead of getting the physical books, I've bought the online course, so there might be some mismatch with the physical book's chapter indices. In addition to that, there could be some content updates for the online course, but not the physical books.

**Note:** These notes are a work in progress. I'll remove this remark once I go through the whole book.

 * [Chapter 2 - Scale From Zero To Millions Of Users](./chapter02)
 * [Chapter 3 - Back-of-the-envelope Estimation](./chapter03)
 * [Chapter 4 - A Framework For System Design Interviews](./chapter04)
 * [Chapter 5 - Design A Rate Limiter](./chapter05)
 * [Chapter 6 - Design Consistent Hashing](./chapter06)
 * [Chapter 7 - Design A Key-Value Store](./chapter07)
 * [Chapter 8 - Design A Unique ID Generator In Distributed Systems](./chapter08)
 * [Chapter 9 - Design A URL Shortener](./chapter09)
 * [Chapter 10 - Design A Web Crawler](./chapter10)
 * [Chapter 11 - Design A Notification System](./chapter11)
 * [Chapter 12 - Design A News Feed System](./chapter12)
 * [Chapter 13 - Design A Chat System](./chapter13)
 * [Chapter 14 - Design A Search Autocomplete System](./chapter14)
 * [Chapter 15 - Design YouTube](./chapter15)
 * [Chapter 16 - Design Google Drive](./chapter16)
 * [Chapter 17 - Proximity Service](./chapter17)
 * [Chapter 18 - Nearby Friends](./chapter18)
 * [Chapter 19 - Google Maps](./chapter19)
 * [Chapter 20 - Distributed Message Queue](./chapter20)
 * [Chapter 21 - Metrics Monitoring And Alerting System](./chapter21)
 * [Chapter 22 - Ad Click Event Aggregation](./chapter22)
 * [Chapter 23 - Hotel Reservation System](./chapter23)
 * [Chapter 24 - Distributed Email Service](./chapter24)
 * [Chapter 25 - S3-like Object Storage](./chapter25)
 * [Chapter 26 - Real-time Gaming Leaderboard](./chapter26)
 * [Chapter 27 - Payment System](./chapter27)
 * [Chapter 28 - Digital Wallet](./chapter28)
 * [Chapter 29 - Stock Exchange](./chapter29)

System Design LifeCycle(SDLC)

1. Planning Stage
2. Feasibility Study Stage
3. System Design Stage
4. Implementation Stage
5. Testing Stage
6. Deployment Stage
7. Maintenance and Support

Different types of Software Architecture Patterns include:

Layered Pattern
Client-Server Pattern
Event-Driven Pattern
Microkernel Pattern
Microservices Pattern

There are many different types of load balancers, and the specific type that is used in a given system will depend on the specific requirements of the system. Some common types of load balancers include:

* Layer 4 load balancers operate at the network layer of the OSI model and distribute requests based on the source and destination IP addresses and port numbers of the requests.
* Layer 7 load balancers operate at the application layer of the OSI model and distribute requests based on the content of the requests, such as the URL or the type of HTTP method used.
* Global load balancers are used in distributed systems to distribute requests among multiple servers located in different geographic regions.
* Application load balancers are specialized load balancers that are designed to work with specific types of applications or protocols, such as HTTP or HTTPS.
