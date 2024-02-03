# Distributed Email Service
We'll design a distributed email service, similar to gmail in this chapter.

In 2020, gmail had 1.8bil active users, while Outlook had 400mil users worldwide.

# Step 1 - Understand the Problem and Establish Design Scope
 * C: How many users use the system?
 * I: 1bil users
 * C: I think following features are important - auth, send/receive email, fetch email, filter emails by read/unread status, search email by subject sender and body, anti-spam and anti-virus protection.
 * I: Good list. Don't worry about auth for now.
 * C: How do users connect with email servers?
 * I: Typically, email clients connect via SMTP, POP, IMAP, but we'll use HTTP for client server communication.
 * C: Can emails have attachments?
 * I: Yes

## Non-functional requirements
 * Reliability - we shouldn't lose email data
 * Availability - We should use replication to ensure availability and prevent single points of failure. We should also tolerate partial system failures. In CAP, we select availability and partitioning over consistency.
 * Scalability - As userbase grows, our system should be able to handle them.Performance should not degrade.
 * Flexibility and extensibility - system should be flexible and easy to extend with new features. One of the reasons we chose HTTP over POP/IMAP/other traditional mail protocols.

## Back-of-the-envelope estimation
 * 1bil users
 * Assuming one person sends 10 emails per day -> 100k emails per second (10^9 x 10/10^5).
 * Assuming one person receives 40 emails per day and each email on average has 50kb metadata -> 730pb storage per year.(1 bil users per year x 365 days per year x 40 emails per day x 50 kB per email)
 * Assuming 20% of emails have storage attachments and average size is 500kb -> 1,460pb per year. (1 bil users per year x 365 days per year x 40 emails per day x 500 kB per attachment x 20%)

# Step 2 - Propose High-Level Design and Get Buy-In
## Email knowledge 101
There are various protocols used for sending and receiving emails:
 * SMTP (Simple Mail Transfer Protocol) - standard protocol for sending emails from one server to another.
 * POP (Post Office Protocol)- standard protocol for receiving and downloading emails from a remote mail server to a local client. Once retrieved, emails are deleted from remote server.
 * IMAP (Internet Mail Access Protocol) - similar to POP, it is used for receiving and downloading emails from a remote server, but it keeps the emails on the server-side allowing for access from multiple devices. Widely used for individual accounts and works well when connection is slow because only email header is downloaded till it is opened.
 * HTTPS - not technically an email protocol, but it can be used for web-based email clients (eg Microsoft Outlook uses a HTTPS based protocol called ActiveSync).

Apart from the mailing protocol, there are some DNS records we need to configure for our email server - the MX (mail exchanger) records. The priority numbers indicate preferences, where the mail server with a lower priority number is more preferred. Here, the sending mail server will try to connect to server with priority 5 and if it fails it goes to server with next lower priority, priority 10:
![dns-lookup](images/dns-lookup.png)

Email attachments are sent base64-encoded and there is usually a size limit of 25mb on most mail services.
This is configurable and varies from individual to corporate accounts.MIME(Multipurpose Internet Mail Extension) protocol is used for sending attachments.

## Traditional mail servers
Traditional mail servers work well when there are a limited number of users, connected to a single server.
![traditional-mail-server](images/traditional-mail-server.png)
 * Alice logs into her Outlook email and presses "send". Email is sent to Outlook mail server. Communication is via SMTP.
 * Outlook server queries DNS to find MX record for gmail.com and transfers the email to their servers. Communication is via SMTP.
 * Gmail server stores the email and makes it available to Bob when he logs in.
 * Bob fetches emails from his gmail server via IMAP/POP.

In traditional mail servers, emails were stored on the local file system. Every email was a separate file. Eg:MailDir
![local-dir-storage](images/local-dir-storage.png)

As the scale grew, disk I/O became a bottleneck. Also, it doesn't satisfy our high availability and reliability requirements.
Disks can be damaged and server can go down.

## Distributed mail servers
Distributed mail servers are designed to support modern use-cases and solve modern scalability issues.

These servers can still support SMTP/IMAP/POP for native email clients and SMTP for mail exchange across servers.

But for rich web-based mail clients, a RESTful API over HTTP is typically used.

Example APIs:
 * `POST /v1/messages` - sends a message to recipients in To, Cc, Bcc headers.
 * `GET /v1/folders` - returns all folders of an email account

Example response:
```
[{id: string        Unique folder identifier.
  name: string      Name of the folder.
                    According to RFC6154 [9], the default folders can be one of
                    the following: All, Archive, Drafts, Flagged, Junk, Sent,
                    and Trash.
  user_id: string   Reference to the account owner
}]
```
 * `GET /v1/folders/{:folder_id}/messages` - returns all messages under a folder \w pagination
 * `GET /v1/messages/{:message_id}` - get all information about a particular message
Example response:
```
{
  user_id: string                      // Reference to the account owner.
  from: {name: string, email: string}  // <name, email> pair of the sender.
  to: [{name: string, email: string}]  // A list of <name, email> paris
  subject: string                      // Subject of an email
  body: string                         //  Message body
  is_read: boolean                     //  Indicate if a message is read or not.
}
```

Here's the high-level design of the distributed mail server:
![high-level-architecture](images/high-level-architecture.png)
 * Webmail - users use web browsers to send/receive emails
 * Web servers - public-facing request/response services used to manage login, signup, user profile, etc.
 * Real-time servers - Used for pushing new email updates to clients in real-time. We use websockets for real-time communication but fallback to long-polling for older browsers that don't support them. Eg: Apache James mail server that implements the JSON Meta Application Protocol (JMAP) over websocket.
 * Metadata db - stores email metadata such as subject, body, from, to, etc.
 * Attachment store - Object store (eg Amazon S3), suitable for storing large files. NoSQL db like Cassandra not a god fit because it can't practically store more than 1 MB attachment size and we can't use a row cache as attachments take too much memory space. 
 * Distributed cache - We can cache recent emails in Redis to improve UX.
 * Search store - distributed document store that uses data structure called inverted index that supports very fast full-text searches.

Here's what the email sending flow looks like:
![email-sending-flow](images/email-sending-flow.png)
 * User writes an email and presses "send". Email is sent to load balancer.
 * Load balancer rate limits excessive mail sends and routes to one of the web servers.
 * Web servers do basic email validation (eg email size) and short-circuits outbound flow if domain is same as sender and inserts email into sender's sent folder and receiver's inbox folder. But does spam and virus check first.
 * If basic validation passes, email is sent to message queue (attachment is referenced from object store)
 * If basic validation fails, email is sent to error queue
 * SMTP outgoing workers pull messages from outgoing queue, do spam/virus checks and route to recipient mail server.
 * Email is stored in the "Sent Emails" folder of the sender.

Each message in the outgoing queue contains all the metadata required to create an email. A distributed message queue is a critical component that allows asynchronous mail processing. By decoupling SMTP outgoing workers from web servers, we can scale them both independently. 

We need to also monitor size of outgoing message queue. Growing too large might indicate a problem:
 * Recipient's mail server is unavailable. We can retry sending the email at a later time using exponential backoff.
 * Not enough consumers to handle the load, we might have to scale the consumers to reduce processing time.

Here's the email receiving flow:
![email-receiving-flow](images/email-receiving-flkow.png)
 * Incoming emails arrive at the SMTP load balancer. Mails are distributed to SMTP servers by load balancer, where email acceptance policy is configured and applied at the SMTP connection level (eg invalid emails are directly discarded).
 * If attachment of email is too large, we can put it in attachment/object store (s3).
 * Emails are put in message queue which acts as buffer in case of email volume surges.
 * Mail processing workers do preliminary checks, after which mails are forwarded to storage, cache, object store and real-time servers (websocket servers that allow clients to get email in real time) in case user is online.
 * Offline users get their new emails once they come back online via HTTP API. It is stored in storage layer till then. Once they come online, Web servers pull new emails from storage layer and return them to the client.

# Step 3 - Design Deep Dive
Let's now go deeper into some of the components.

## Metadata database
Here are some of the characteristics of email metadata:
 * headers are usually small and frequently accessed
 * Body size ranges from small to big, but is infrequently accessed, typically read once
 * Most mail operations are isolated to a single user - eg fetching email, marking as read, searching. No sharing between two or more users.
 * Data recency impacts data usage. Users typically read only recent emails
 * Data has high-reliability requirements. Data loss is unacceptable.

At gmail/outlook scale, the database is typically custom made to reduce input/output operations per second (IOPS).

Let's consider what database options we have:
 * Relational database (MySQL or PostgreSQL) - we can build indexes for headers and body, but these DBs are typically optimized for small chunks of data.
 * Distributed object store (Amazon S3)- this can be a good option for backup storage, but can't efficiently support searching/marking as read/etc.
 * NoSQL (Cassandra) - Google BigTable is used by gmail, but it's not open-sourced.

Based on above analysis, very few existing solutions seems to fit our needs perfectly.
In an interview setting, it's infeasible to design a new distributed database solution, but important to mention characteristics:
 * Single column can be a single-digit MB
 * Strong data consistency
 * Designed to reduce disk I/O
 * Highly available and fault tolerant
 * Should be easy to create incremental backups

**Data Model**
In order to partition the data, we can use the `user_id` as a partition key, so that one user's data is stored on a single shard.
This prohibits us from sharing an email with multiple users, but this is not a requirement for this interview.

Let's define the tables:
 * Primary key consists of partition key (data distribution across nodes) and clustering key (sorting data within a partition)
 * Queries we need to support - get all folders for a user, display all emails for a folder, create/get/delete an email, fetch read/unread email, get conversation threads (bonus)

Legend for tables to follow:
![legend](images/legend.png)

Here is the folders table:
![folders-table](images/folders-table.png)

emails table:
![emails-table](images/emails-table.png)
 * email_id is timeuuid which allows sorting based on timestamp when email was created

Attachments are stored in a separate table, identified by filename:
![attachments](images/attachments.png)

Supporting fetchin read/unread emails is easy in a traditional relational database, but not in Cassandra, since filtering on non-partition/clustering key is prohibited.
One workaround to fetch all emails in a folder and filter in-memory, but that doesn't work well for a big-enough application.

What we can do is denormalize the emails table into read/unread emails tables:
![read-unread-emails](images/read-unread-emails.png)

In order to support conversation threads (eg. using JWZ algorithm), we can include some headers, which mail clients interpret and use to reconstruct a conversation thread:
```
{
  "headers" {
     "Message-Id": "<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>",
     "In-Reply-To": "<CAEWTXuPfN=LzECjDJtgY9Vu03kgFvJnJUSHTt6TW@gmail.com>",
     "References": ["<7BA04B2A-430C-4D12-8B57-862103C34501@gmail.com>"]
  }
}
```

Finally, we'll trade availability for consistency for our distributed database, since it is a hard requirement for this problem.

Hence, in the event of a failover or network parititon, sync/update actions will be briefly unavailable to impacted users.

## Email deliverability
It is easy to setup a server to send emails, but getting the email to a receiver's inbox is hard, due to spam-protection algorithms.

If we just setup a new mail server and start sending mails through it, our emails will probably end up in the spam folder.

Here's what we can do to prevent that:
 * Dedicated IPs - use dedicated IPs for sending emails, otherwise, recipient servers will not trust you.
 * Classify emails - avoid sending marketing emails from the same servers to prevent more important email to be classified as spam
 * Warm up your IP address slowly to build a good reputation with big email providers. It takes 2 to 6 weeks to warm up a new IP
 * Ban spammers quickly to not deteriorate your reputation
 * Feedback processing - setup a feedback loop with ISPs to keep track of complaint rate and ban spam accounts quickly.If an email fails to deliver, it can be either due to hard bounce (recipient's email address is invalid), soft bounce (temporary issue, like ISP being too busy, can retry) or complaint (recipient clicks 'report spam' button). Use separate queues for each so they can be managed accordingly.
 * Email authentication - use common techniques to combat phishing such as Sender Policy Framework (SPF), DomainKeys Identified Mail(DKIM), Domain based Message Authenticaito, Reporting and Conformance (DMARC).

You don't need to remember all of this. Just know that building a good mail server requires a lot of domain knowledge AND good relationships with ISPs.

## Search
Searching includes doing a full-text search based on email contents or more advanced queries based on from, to, subject, unread, etc filters.

One characteristic of email search is that it is local to the user and it has more writes than reads, because we need to re-index it on each operation, but users rarely use the search tab.

Let's compare google search with email search:
|               | Scope                | Sorting                               | Accuracy                                          |
|---------------|----------------------|---------------------------------------|---------------------------------------------------|
| Google search | The whole internet   | Sort by relevance                     | Indexing takes some time, so not instant results. |
| Email search  | User’s own email box | Sort by attributes eg time, has attachment, date within, is unread, etc | Indexing should be quick and results accurate.    |

To achieve this search functionality, one option is to use an Elasticsearch cluster. We can use `user_id` as the partition key to group data under the same node:
![elasticsearch](images/elasticsearch.png)

Mutating operations are async via Kafka in order to decouple services that trigger indexing from the services that perform reindexing.
Actually searching for data happens synchronously.

Elasticsearch is one of the most popular search-engine databases and supports full-text search for emails very well. One challenge is to keep our primary email store in sync with ElasticSearch.

Alternatively, we can attempt to develop our own custom search solution to meet our specific requirements.

Designing such a system is out of scope. One of the core challenges when building it is to optimize it for write-heavy workloads.

To achieve that, we can use Log-Structured Merge-Trees (LSM) to structure the index data on disk. Write path is optimized for sequential writes only. 
This technique is used in Cassandra, BigTable and RocksDB.

Its core idea is to store data in-memory until a predefined threshold is reached, after which it is merged in the next layer (disk). When a new email arrives, it is first added to level 0 in-memory cache, and when data size in memory reaches the predefined threshold, data is merged to the next level. We can also separate data changes frequently (folder data) from data that does not change frequently (email data).:
![lsm-tree](images/lsm-tree.png)

Main trade-offs between the two approaches:
 * Elasticsearch scales to some extent, whereas a custom search engine can be fine-tuned for the email use-case, allowing it to scale further.
 * Elasticsearch is a separate service we need to maintain, alongside the metadata store. A custom solution can be the datastore itself. So system complexity is higher for elasticsearch and data consistency is lower.
 * Data loss is not possible in either. For Elasticsearch, the index can be rebuilt from the primary storage, in case of failure.
 * Elasticsearch is an off-the-shelf solution and is easy to integrate, whereas the custom search engine would require significant engineering effort to build.
 * Elasticsearch better for smaller email services. A team of elasticsearch experts are needed for large scale email services. For Gmail and Outlook scale, search embedded in database is better as opposed to separate indexing approach.

## Scalability and availability
Since individual user operations don't collide with other users, most components can be independently scaled.

To ensure high availability, we can also use a multi-datacenter setup with leader-folower failover in case of failures:
![multi-dc-example](images/multi-dc-example.png)

# Step 4 - Wrap up
Additional talking points:
 * Fault tolerance - Many parts of the system could fail. It is worthwhile how we'd handle node failures.
 * Compliance - PII needs to be stored in a reasonable way, given Europe's GDPR laws.
 * Security - email encryption, phishing protection, safe browsing, etc.
 * Optimizations - eg preventing duplication of the same attachments, sent multiple times by different users.
