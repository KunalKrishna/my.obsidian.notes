**Vertical Scaling** (scaling up) : Increases a **single database server**'s capacity (CPU, RAM, storage), providing simple, high-performance, consistent, but finite, and costly scaling with potential downtime.
**Horizontal Scaling** (scaling out) : adds **more servers** to a cluster, allowing nearly infinite, cost-effective scalability and higher reliability through distributed architecture, but introduces complexity and data consistency challenges.

|                    | Vertical                                                                                                                   | Horizontal                                                                                                                    |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **Approach:**      | adds resources/power (CPU, RAM) to a single node                                                                           | adds multiple nodes to a network                                                                                              |
| **Performance:**   | ideal for low-latency/smaller workloads                                                                                    | shines in high-traffic/distributed applications                                                                               |
| **Cost:**          | often cheaper initially but expensive at high scale                                                                        | often has higher upfront costs but better long-term cost-efficiency                                                           |
| **Downtime:**      | often requires downtime for upgrades                                                                                       | typically enables scaling without downtime                                                                                    |
| **Examples:**      | MySQL on Amazon RDS                                                                                                        | Cassandra, MongoDB, Google Cloud Spanner                                                                                      |
| **Ideal Use Case** | Best for smaller, legacy applications with limited budgets and strict data consistency needs where downtime is acceptable. | Best for high-growth, modern applications requiring high availability, fault tolerance, and massive, real-time data handling. |
![[dbms.scaling1.png]]
**Scaling vertically** =  One big hulk will do all the work for you.
**Scaling horizontally** =  Thousands of minions will do the work together for you.

![[dbms.scaling2.png]]
[Horizontal vs. Vertical Scaling – How to Scale a Database](https://www.freecodecamp.org/news/horizontal-vs-vertical-scaling-in-database/)

### Replication
= redundancy 
= high availability 

[Sharding](https://planetscale.com/docs/vitess/sharding/sharding-quickstart) and partitioning are techniques to divide and scale large databases.
Sharding distributes data across multiple servers, while partitioning splits tables within one server.
### Partition

### Sharding
sharding involves **splitting** the data rather than making copies


[Sharding vs. partitioning: What’s the difference? — PlanetScale](https://planetscale.com/blog/sharding-vs-partitioning-whats-the-difference)