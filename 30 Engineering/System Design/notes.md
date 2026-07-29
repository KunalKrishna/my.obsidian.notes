![[sysD.png]]
- Delivery Framework
	- Requirement
		- Functional Requirements
		- Non-FR : (SCALE For Cloud Design-S)
			- Scalability
			- CAP
			- Latency
			- Environment
			- Fault Tolerance
			- Compliance(Law)
			- Durability
			- Security
		- Capacity Estimations
	- Core Entities
	- API or System interface
	- (Optional) Data Flow 
	- HLD
	- Deep Dive

![[Delivery Framework.png]]

- Core Concepts
	- Networking Essentials
	- API Design
	- Data Modeling
	- Database Indexing
	- Caching
	- Sharding
	- Consistent Hashing
	- CAP Theorem
	- Numbers to Know

- Key Technologies : 
	- Core Database
		- Relational Database
		- NoSQL DB
	- Blob Storage
	- Search Optimized Db
	- API Gateway
	- Load Balancer
	- Queue
	- Streams/ Event Sourcing
	- Distributed Lock
	- Distributed Cache
	- CDN

- Common Patterns 
	- Pushing Realtime Updates
	- Managing Long-Running Tasks
	- Dealing with Contention
	- Scaling Reads
	- Scaling Writes 
	- Handling Large Blobs
	- Multi-Step Processes
	- Proximity-Based Services

Core Concepts : 
- Networking Essentials : OSI Model :  PDNTSPA
	- Layer 7 : Application Layer
	- Layer 4 : Transport Layer - TCP (reliable) vs UDP (fast)
	- Layer 3 : Network Layer - IP Address
	* Network Layer Protocol  : IP protocol
	- Transport Layer Protocol : 
		- UDP : Fast but Unreliable
		- TCP : Reliable but with overhead
	- Application Layer Protocol
		- **HTTP/HTTPS** : The Web's Foundation ( request-response model)
		- **REST** : Simple and Flexible
		- **GraphQL** : Flexible Data Fetching
		- **gRPC** : Efficient Inter-Service Communication
		- **SSE** - Server-Sent Events : Real-Time Push Communication
			-  a great way to push from the server to client
			- e.g. for keeping bidders up-to-date on the current price of an auction.
		- **WebSockets**: Real-Time Bidirectional Communication
		- **WebRTC** : Peer-to-Peer Communication 
- Load Balancing (Scaling)
	- Client-Side Load Balancing
	- 
- Data Modelling
- Caching
- Sharding
- Consistenet Hashing
- CAP Theorem
- Db Indexing
- NtK


Scaling : two options
- **Vertical** : Bigger server
- **Horizontal** : add more servers to handle the load 
### Communication Models Comparison

| Protocol      | Model            | Direction     | Persistence      | Use Case                             |
| ------------- | ---------------- | ------------- | ---------------- | ------------------------------------ |
| **HTTP**      | Request-Response | Client→Server | No (stateless)   | Web pages, REST APIs                 |
| **GraphQL**   | Request-Response | Client→Server | No (stateless)   | Flexible API queries                 |
| **gRPC**      | RPC + Streaming  | Bidirectional | Yes (HTTP/2)     | Microservices, high-performance APIs |
| **SSE**       | Server Push      | Server→Client | Yes (persistent) | Live updates, notifications          |
| **WebSocket** | Full-Duplex      | Bidirectional | Yes (persistent) | Real-time chat, live data            |
| **WebRTC**    | Peer-to-Peer     | P2P direct    | Yes (persistent) | Video calls, file sharing            |
![[Pasted image 20260513095621.png]]