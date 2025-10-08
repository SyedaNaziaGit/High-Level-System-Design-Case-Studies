# High Level System Design For DropBox or Google Drive System Design 

Basically Dropbox or Google Drive are cloud storage services – they let you save  your files or data (photos , videos, documents, code)  online  


Dropbox has around 500+ million unique users, 500Million requests/day, High writes and reads
## 1.	Requirement Gathering  for Dropbox System Design :
### a.	Functional Requirements / Features Required :
- The user can upload or download files or photos or media from computer or phone
- The user should be able to create or delete the directory on drive
- The drive synchronize the data between the drive and the users
- The user should be able  to sync update, delete, write, read, rewrite the data 
- The user should be able to look at the history of updates [when was the last update made with time stamp]
- The user can access anytime anywhere from any device
- The user can share them easily  via a link
- The user can backup safely even when laptop data has crashed
### b.	Non Functional Requirements
#### ACID properties:
Atomicity Consistency Integrity and Durability – all file operations should follow these properties 
#### Availability :
 It basically means what percentage of times the system is available to the users request. Requirement is that the system should be  available to users request all the time
#### Durability :
Our system should ensure the file uploaded or changes made  to the storage should be permanent without any data loss
#### Reliability: 
It means as long as the input remains same the output should also be same i.e, its reliable for the requests made by the user
#### Scalability : 
Our system should  be able to handle the increasing traffic and number  of users as well as requests.

## 2.	Capacity estimation for System Design :
#### Storage Estimation Assumptions:
The total number of users = 10 Million

The Total number of daily active users = 1 Million

The average files stored by each user = 100

The average file size of each file = 100KB

Total number of connections per minute = 10K

#### Storage Estimation =
Total number of files = 10 Million * 100 = 1GB

Total storage Required = 1GB * 100KB = 1PB
## Problem:
When using Amazon S3 as a cloud storage service, uploading a 20MB file is straightforward. However, if we later need to update the same file, we may first need to download it from S3, make the edits, and then upload it again. If we want to maintain a history of all file versions, we must keep copies of each update. For small changes, like minor character corrections, or even larger edits, this approach can quickly become inefficient. For example, if the file is 20MB and we make three updates, we would consume 60MB of bandwidth just to store these versions. This not only increases cloud storage costs but can also slow down the upload and download process, making frequent updates costly and time-consuming.
There are several challenges when managing file updates on cloud storage like Amazon S3. First, upload and concurrency can be problematic. 

1.	For instance, if we want to automate file monitoring and uploads using a Python script, implementing multithreading or multiprocessing to sync files faster is difficult, making concurrent uploads and downloads hard to achieve. 

2.	latency becomes an issue: if uploading or downloading 1MB takes 1 second, a 20MB file would require around 20 seconds, and this delay cannot be easily optimized. 

3.	Bandwidth consumption is inefficient because even a minor change, like a single character correction, requires uploading the entire 20MB file, consuming the same bandwidth as the full file.

4.	History and Storage Management are costly: to maintain different versions of the file, we must store the full 20MB file for each update, even for tiny edits, which is inefficient and expensive.

<img width="975" height="563" alt="HLDimage1" src="https://github.com/user-attachments/assets/d34ddc31-6b83-44e6-bb40-2123f949f90c" />

To address the problems of bandwidth, latency, concurrency, and storage when updating large files on cloud storage, we can adopt a chunk-based approach instead of treating the file as a single unit. In this design, a file is divided into smaller chunks. When a small change is made—such as editing a single character—only the affected chunk needs to be updated and synced with the cloud. For example, if a file is split into 10 chunks and the third chunk is modified, only that 2MB chunk is uploaded instead of the entire 20MB file. This reduces bandwidth usage from 20MB to 2MB and decreases storage overhead, as we only store the updated chunk instead of maintaining a full copy of the file for each change.


This chunk-based approach also solves concurrency issues. By using multithreaded or multiprocess scripts in Python, multiple chunks can be uploaded concurrently. If the third chunk is being updated by one thread, other threads can simultaneously handle other chunks. As a result, the effective upload time is reduced—for instance, a 20-second upload distributed across five threads may complete in just 4 seconds. Similarly, if a seventh chunk is later updated, only the new chunk is synced, further optimizing bandwidth and storage.

To manage and reconstruct files efficiently, a metadata file is maintained, which stores references (hashes) and indexes for all chunks. This allows downloading all chunks concurrently and then stitching them back together to recreate the full file. This design is the foundation of modern distributed file systems and cloud file-sharing services. Systems like HDFS (Hadoop Distributed File System) follow a similar approach: they divide large files into 64MB chunks, store multiple copies for redundancy, and manage chunk-based operations internally, enabling efficient updates, high concurrency, and optimized bandwidth and storage.

## Low Level System Design Of DropBox

<img width="753" height="502" alt="image" src="https://github.com/user-attachments/assets/5d4cc4cf-49b2-46a5-b0e8-b9f075772aae" />
The fundamentals of designing a file-uploading service, such as Dropbox or Google Drive, revolve around creating a seamless experience for users across multiple devices and platforms.

The client can take various forms—desktop applications, mobile apps, or web browser interfaces—allowing users to upload or synchronize files effortlessly. Each client is designed to ensure smooth file operations, including uploading, syncing, and managing changes, and multiple clients can exist for a single user. 

<img width="681" height="341" alt="image" src="https://github.com/user-attachments/assets/6a482c32-0e5a-446e-9949-e1dd0945d3de" />
At a high level, the architecture comprises four key components: the client, which tracks file changes within designated folders; cloud storage, typically leveraging services like Amazon S3 or other blob storage solutions to store the actual files; a messaging system, such as RabbitMQ or Kafka, to coordinate updates and notifications between components; and a database with caching, which maintains metadata about the files to ensure fast access and efficient synchronization. Together, these components form a robust ecosystem that enables reliable and consistent file management across devices.

### Folder watcher 
The Folder watcher continuously monitors designated folders for any changes. As soon as new files are detected, it notifies the chunker and indexer, passing along the file paths. 

### Chunker
The Chunker then splits these files into smaller chunks, computes a unique hash for each chunk, and uploads them to cloud storage, such as Amazon S3. Each uploaded chunk’s hash and cloud URL are returned to the indexer, which maps the hashes to their respective URLs and updates the internal database with the file metadata. This metadata not only aids in conflict resolution and offline access but also ensures consistent tracking of file changes. 

### Indexer
The Indexer also communicates with the messaging service to propagate updates, which are queued by the synchronization service. 

### Syncronization Service/Sync service
The Sync service then ensures that metadata and file updates are consistently synchronized across all devices, maintaining a reliable and consistent state in the central MySQL database. The synchronization service plays a crucial role in ensuring consistency across multiple devices. It sends messages back to the messaging queue, which are then broadcasted to all other clients belonging to the same user. Each client, having a similar folder structure, allows its local indexer to retrieve the corresponding file chunks from the cloud. The indexer then reconstructs the files and synchronizes them into the client’s local folder, ensuring that all devices have an up-to-date and consistent view of the user’s files.

###  Messaging Service 
The Messaging Service acts as the communication backbone of the system, managing the flow of updates between different components. When file changes occur, they are not sent directly to the synchronization service; instead, they are routed through a message queue. This design enables asynchronous communication, ensuring that updates are processed efficiently without overloading any single component or blocking operations. By decoupling the client and sync services, the system becomes more scalable, fault-tolerant, and responsive. Each client can have its own dedicated queue—or a set of queues—allowing parallel handling of multiple updates and ensuring that all devices stay in sync, even under heavy load or network delays.

<img width="646" height="455" alt="image" src="https://github.com/user-attachments/assets/2bed4e29-9f21-4fb0-968b-73380f12bc0e" />

The system employs two types of message queues  a request queue and a response queue to handle file synchronization in an asynchronous and reliable manner. When a client detects a file change (for example, Client 1), it uploads the updated file chunks to the cloud and gathers all associated metadata, including chunk hashes, file versions, and paths. This metadata is then posted to the request queue instead of being sent directly to the synchronization service. The use of asynchronous queues is crucial because clients may not always be online or connected to the internet. By buffering updates in a queue, the system ensures that no messages are lost, even if a client temporarily disconnects.

The synchronization service consumes messages from the request queue, processes them, and updates the metadata stored in the internal database and cache. Once the metadata has been updated, the sync service broadcasts notifications to all other registered clients of the same user through the response queues (e.g., Response Queue 1, 2, 3, etc.). This design guarantees that all devices eventually receive consistent updates, ensuring smooth synchronization across multiple clients.

The metadata database serves as the backbone for maintaining the state of files and their relationships. It stores information such as file chunks, hashes, versions, user identifiers, and workspace or device associations. 

The architectural flow can be summarized as:
### Clients → Edge Wrapper → Engine → Cache → MySQL. 
Dropbox, for instance, uses a similar architecture with HStore as its metadata layer.

Consistency in metadata storage is essential, as multiple clients may be modifying or syncing files simultaneously. While both RDBMS and NoSQL databases can be used, RDBMS systems like MySQL are preferred for their strong consistency guarantees. NoSQL databases, though scalable, often provide eventual consistency, which can lead to synchronization errors or missing chunks — an unacceptable risk in file storage systems. However, consistency layers can be added on top of NoSQL systems if scalability is a primary concern. Dropbox successfully uses MySQL to handle metadata for millions of users and files, proving that relational databases can scale effectively when designed and optimized properly.

<img width="767" height="559" alt="image" src="https://github.com/user-attachments/assets/b8e3cd4a-5dc6-4e3c-9f71-3b594f78264f" />

### Internal Architecture of Metadata

#### Scaling relational databases (RDBMS)
Scaling relational databases (RDBMS) is often challenging, and Dropbox faced similar difficulties when managing its vast metadata store. To address this, Dropbox implemented database sharding in MySQL, distributing metadata across multiple databases to balance load and improve performance. This sharding technique is part of their Edge Store system. However, working with multiple shards introduces significant complexity for developers — they must manage schema consistency, rebalance data when databases fail, and ensure continuous availability. Each time a shard reaches capacity or a database node fails, new shards must be added and rebalanced, making maintenance both time-consuming and error-prone
#### Edge Wrapper
To simplify this process, Dropbox built an Edge Wrapper, an abstraction layer that sits between clients and the underlying sharded metadata databases. Instead of interacting directly with the databases, clients communicate through this wrapper using an Object Relational Mapper (ORM) interface, typically written in Go or Python. The ORM handles data access, while an internal Engine translates ORM calls into optimized database queries. Between the wrapper and the database lies an internal cache, which improves performance by reducing direct database lookups. 

When a client requests information, the wrapper first checks the cache; if the data isn’t found, it fetches it from the database via the engine and then updates the cache automatically.

This layered architecture offers multiple advantages. Developers are shielded from the complexities of sharding and database schema management, as the Edge Wrapper abstracts those details completely. Moreover, the Edge Store provides transaction isolation natively, ensuring that concurrent database operations remain consistent without requiring developers to manually handle locks or transactions.

### Serving Data at Scale
With over 10 million users and hundreds of millions of files, Dropbox needed to serve metadata and file data at lightning-fast speeds. Since Amazon S3, used for file storage, is globally distributed across multiple regions, Dropbox adopted a data locality strategy for metadata as well. 

The system ensures that metadata is stored close to the user’s geographic region — for example, a user in Singapore would have their metadata stored in a nearby data center. This minimizes latency and speeds up file access.

Without metadata, no client can retrieve or reconstruct a file from S3, as metadata holds essential information about file chunks, hashes, versions, and locations. Therefore, 

Dropbox also replicated its metadata infrastructure closer to users. Using techniques like clustering algorithms (e.g., K-Means), Dropbox identified user groups and strategically deployed Edge Store servers and CDNs in regions with dense user populations. This design ensures that both metadata and file data are served efficiently, enabling users to experience fast, reliable file synchronization and downloads worldwide.

## High Level Design of Dropbox System Design :
<img width="986" height="471" alt="image" src="https://github.com/user-attachments/assets/b6cc8e7e-a3f6-4d73-b5f5-282ae833ea63" />

### Internal Architecture: File Upload and Download Flow
The file management system enables seamless upload and download operations through a well-orchestrated architecture involving multiple coordinated components.
#### 1. User Uploading:
Users interact with a web or client application to initiate file uploads. The client communicates with the Upload Service on the server side, which manages the entire upload workflow. For large files, the client may split the data into smaller chunks to ensure efficient and reliable transfer.
#### 2. Upload Service:
The Upload Service is responsible for handling all incoming upload requests. It generates Presigned URLs that allow clients to upload directly to Amazon S3, thereby reducing server load and improving performance. The service oversees the upload process to ensure data integrity and completeness. Once an upload is successful, it updates the Metadata Database with essential file details such as name, size, and owner. For very large files, it also coordinates the chunked upload process.
#### 3. Getting Presigned URL:
When a client needs to upload a file, it first requests a Presigned URL from the Upload Service. The service communicates with the S3 backend to generate a unique, time-limited URL that grants secure permission to upload a specific file to a designated bucket. This mechanism allows the client to interact directly with S3 for the upload, bypassing the server in the actual file transfer phase.
#### 4. S3 Bucket:
Amazon S3 serves as the primary storage layer, providing durability, scalability, and high availability. With Presigned URLs, clients can upload files directly to S3 without passing through the application servers. The bucket structure is often organized based on user accounts or metadata categories, ensuring structured and efficient data management.
#### 5. Metadata Database:
The Metadata Database maintains structured information about each uploaded file. It stores attributes such as file name, size, owner, access permissions, and timestamps. This enables the system to quickly retrieve file details and enforce access controls without needing to query S3 directly. It also ensures synchronization between metadata and the actual stored content.
#### 6. Uploading to S3 Using Presigned URLs and Metadata:
Once the Presigned URL is obtained, the client uploads the file directly to S3. During this process, associated metadata—such as file name, size, and owner details—is transmitted to keep the S3 data and the Metadata Database synchronized. This integration guarantees that metadata accurately reflects the uploaded content.
#### 7. Role of Task Runner:
After the upload completes, a Task Runner process is triggered to handle post-upload operations. This may include updating file status in the Metadata Database, triggering indexing for search functionality, or sending notifications to users. The Task Runner ensures that background tasks are executed reliably without blocking the main upload flow.
#### 8. Downloading Services:
For file downloads, clients send requests via the client application. The Download Service retrieves file metadata from the database, verifying permissions and ownership. Based on this information, it can generate a Presigned URL for secure direct download from S3. This ensures an efficient and secure download experience while maintaining data consistency and access control.

