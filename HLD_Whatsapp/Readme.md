# High Level System Design For Whatsapp Messenger / Chat application
WhatsApp handles billions of messages every day, requiring a highly scalable and efficient system. At its core, the platform must manage vast amounts of data while ensuring messages are delivered in real time. To achieve this, WhatsApp employs distributed servers, message queues, and optimized storage mechanisms that allow millions of concurrent connections.

## Requirements for WhatsApp Messenger System Design

### Functional Requirements:
- The system must support both one-on-one and group conversations, allowing users to communicate seamlessly.
- It should provide message delivery acknowledgments, such as sent, delivered, and read, so users are always informed of message status.
- WhatsApp should also enable sharing of media files, including images, videos, and audio.
- Chat messages must be persistently stored when a user is offline, ensuring successful delivery once the user comes online.
- Push notifications should alert offline users of new messages as soon as they become available.

### Non-Functional Requirements:
- The system must deliver messages with low latency, ensuring near real-time communication.
- Message consistency is crucial, so messages should be received in the order they were sent.
-  High availability is required, though in certain cases, consistency may take priority over availability.
-  Security is a top priority, with end-to-end encryption guaranteeing that only the communicating parties can access the content of messages, keeping WhatsApp itself blind to the message contents.
-  The system must be highly scalable to accommodate a growing user base and an increasing volume of messages daily.

## High Level Design of Whatsapp Messenger
<img width="1408" height="736" alt="HLdimg1" src="https://github.com/user-attachments/assets/52244581-4ed4-4237-b4c9-ee06e1f70195" />

Designing a real-time messaging system like WhatsApp requires careful consideration of scalability, reliability, and user experience. At its core, the system enables seamless communication between two clients, say Client A and Client B, through a messaging server that acts as an intermediary. The server ensures that messages are delivered reliably, even under high load, and maintains the order and integrity of the conversation.

When designing such a system, several key features and requirements must be considered. First, understanding the scale is crucial—knowing the number of active users helps determine how many servers and resources are needed. Essential features such as “last seen” status, support for media messages, encryption for security, and even audio/video calling need to be accounted for in the architecture.

Given a user base in the billions, a single server cannot handle all requests. Therefore, multiple messaging servers are deployed in a horizontally distributed cluster to share the load. However, clients cannot connect randomly to any server. This is where a load balancer becomes critical: it directs each client to the appropriate messaging server based on factors such as current server load or the server the client was previously connected to.

Additionally, a database is required to store certain message states, user metadata, and other transient information. This ensures the system can reliably manage ongoing conversations and maintain a consistent experience across devices. Overall, the design emphasizes high availability, fault tolerance, and low-latency communication to provide users with a real-time messaging experience that is both robust and scalable.

In a real-time messaging system like WhatsApp, the connection between the client and the server is known as a **duplex connection**, meaning it is bidirectional. Once Client A establishes a connection to the messaging server, messages can flow both ways—from the client to the server and from the server to the client. There are different types of network connections, such as TCP, UDP, WebSocket, polling, and long polling, but TCP is the most commonly used, as it provides reliable, ordered delivery between client and server.

For example, when Client A wants to send a message to Client B, the process begins with Client A pushing the message to the messaging server. The server then identifies the appropriate process or connection to forward the message to Client B. If Client B is offline or disconnected from the server, the message is temporarily stored in the database. Once Client B reconnects, the server retrieves the message from the database and delivers it using the established connection. Importantly, only the client initiates the connection—the server does not know the client’s address directly.

Message acknowledgements form an essential part of this system, supporting features like WhatsApp’s “single tick,” “double tick,” and “blue tick” indicators. When Client A sends a message, the server first acknowledges receipt back to Client A, which is reflected as a single tick. Once the server successfully delivers the message to Client B and receives confirmation, Client A sees two ticks indicating delivery. When Client B opens the chat and reads the message, the server is notified, which then informs Client A, resulting in the blue ticks that indicate the message has been read. Each message is associated with a unique identifier to ensure that acknowledgements are correctly mapped to the corresponding message.

This system ensures reliable, real-time communication while maintaining accurate delivery and read statuses, even under varying network conditions

Internally, a real-time messaging system like WhatsApp relies on dedicated processes or threads to manage client connections. Every time a client establishes a connection to the messaging server, a long-running thread or process is created specifically for that client. This thread is responsible for handling all message transfers, acknowledgements, and related operations for that particular connection. Each thread has a unique identifier (PID or thread ID) and is associated with its own queue, which acts as a buffer for incoming or outgoing messages.

For instance, when Client A connects, a thread (say PID1) is created along with a queue. If Client B establishes a connection, a separate thread (PID2) and queue are created for B. When Client A sends a message to Client B, the server process handling A’s connection receives the message, records the PID and user ID in the database, and routes the message to the queue corresponding to B’s process. The process handling B then retrieves the message from the queue and delivers it to Client B. This back-and-forth mechanism ensures smooth, reliable message delivery.

If Client B is offline, no thread or process exists for that user, so the message is saved in a database with the recipient’s user ID. When B reconnects, a new process (say PID3) is created, the database entry is updated with the new PID, and the queued messages are delivered to Client B. This mechanism ensures that messages sent while a user is offline are reliably delivered once they come online.

The “last seen” feature is implemented using a heartbeat mechanism. The client periodically sends heartbeat signals (for example, every 5 seconds) to indicate activity. The server records these timestamps in a table mapping user IDs to the last heartbeat, which allows the system to display accurate last seen information.

For media support, lightweight connections are maintained for messaging, while media content like images, audio, or video is handled separately via an HTTP server. Clients upload media directly to this server or a CDN, which returns a unique hash or resource ID. This hash is then sent via the messaging system to the recipient, who can download the media from the same server using the hash. This design separates heavy media transfers from real-time messaging, keeping the system responsive and scalable.

**Encryption** in a messaging system is essential for ensuring privacy and security. There are generally two types of encryption. The first uses a **shared key** between the two communicating parties, where both use the same key to encrypt and decrypt messages. The second approach employs **asymmetric encryption**, which uses a pair of keys—a private key and a public key. In this method, the sender encrypts the message with the recipient’s public key, and only the recipient’s private key can decrypt it. This ensures that even if the message is intercepted, it cannot be read by unauthorized parties.

For **telephony services** such as voice and video calls, WhatsApp is designed to handle high performance and scalability. It relies on efficient protocols (for example, REL 9) that allow updates to be applied dynamically and maintain lightweight connections. In practice, a single messaging server cannot realistically manage millions of simultaneous connections, so the system is architected to distribute connections across multiple servers. Technologies like FreeBSD, key-value pair databases, and cloud infrastructure such as AWS web servers are used to maximize performance and reliability, ensuring that millions of users can connect and communicate in real time without delays or interruptions.

## Low Level Design For Whatsapp Messenger
<img width="1408" height="736" alt="hldimage2" src="https://github.com/user-attachments/assets/b6a582f7-61b6-4353-b453-308d6d550b6e" />

WhatsApp’s architecture is designed to support real-time messaging, media sharing, and group communication at massive scale while maintaining reliability and low latency. At the core of this system is the WebSocket Server, which establishes persistent connections with each active user’s device. This enables instant message delivery, with the WebSocket Manager using Redis to map users to assigned ports, ensuring that messages are routed efficiently to the correct recipient.

The Message Service handles the sending, storage, and retrieval of messages. It guarantees that messages are delivered in order, even when recipients are offline, by using Kafka queues for sequential processing and Mnesia as persistent storage. Messages are stored when users are offline and delivered once they reconnect, with support for delivery acknowledgments such as sent, delivered, and read.

For media sharing, the Asset Service manages uploading, storage, and retrieval of images, videos, and audio files. Media is compressed and encrypted on the device, stored in Blob Storage with unique IDs and hashes to prevent duplication, and delivered efficiently using CDNs when needed. The media ID is then sent through the Message Service for the recipient to download securely.

WhatsApp also supports group messaging through the Group Service, which maintains group metadata, including member lists and statuses, in MySQL and Redis for fast access. Each group is represented as a Kafka topic, allowing messages to be published and delivered sequentially to all group members. This ensures consistency and low-latency delivery, even for large groups.

Together, these components—WebSocket servers, message queues, databases, media services, and caching layers—work in harmony to provide a seamless, secure, and highly scalable messaging experience for millions of users worldwide.

