---
layout: post
title:  "Benefits of Event Sourcing"
---

## Fully Asynchronous Reactive System
Event sourcing belongs to a broader category of system design called Event Driven Architecture. Components in such systems don’t communicate with each other via requests and responses. Rather they communicate by passing around messages.

Components in an event sourced system are loosely coupled. They don’t need to be aware of each other. They merely need to know how to publish a command and how to process an event. And as such, they are only dependent on the message bus/message broker used by system for publication and receipt of messages.

Components work together either via choreography or via orchestration to get the work done. But there is no synchronous message passing. All messages (command or events) are passed asynchronously via the messaging middleware. This loose coupling leads to a system that is easier to understand, build, test, deploy, maintain, and evolve.

## Auditing
Often, knowing just the current state of an entity is not enough. You would also like to know how the entity got to its current state. Event souring helps you with this since it preserves a complete history of the entire system as a sequence of immutable events. This can provide you with a detailed audit trail of any entity within the system.

Did your shipment end up at a facility it wasn’t supposed to be? The log of events will tell you the path your shipment took through the logistics network to end up at the incorrect facility. Did you not get the expected price on your purchase of 10, 000 shares of AAPL? Events associated with your order will tell you the venues that filled your order and the price per fill.

Financial industries have strict regulatory requirements to preserve every detail about a financial transaction, at times until several years after the transaction is over. As such, they naturally tend to event sourcing. Most other industries don’t have such strict requirements. Nevertheless, they can benefit from the auditability provided by event sourcing.

## Debugging
Lost the application logs? Forgot to add lines of log in application code and so cannot figure out a bug? You can still debug your application, provided the event logs are still there (which they would be if the event logs are replicated). The event logs from production environment could be replayed in a test/local environment through the components that have a bug. You can then step through the production events using a debugger to identify the reason behind erroneous behavior. After fixing the bug you can replay event logs through the fixed component to ensure correct behavior.

## Fault Tolerance
Things fail in a distributed system all the time. Event sourced systems allow progress despite faults. Components within the system collaborate such that groups of components act as subsystems that provide disparate services offered by the system. They are shielded from the rest of the system in the resources they consume and the functionality they offer. Failure of subsystems merely leads to degraded functionality. As such, a fault in no component or subsystem leads to a  failure of the entire system.

A component crashed with an OOM error? No problem. Increase the memory allotted to the component and restart it. A machine crashed along with multiple components that were running on it? No problem. Start another machine with the same set of components (and component identifiers on it). Since components are replicated state machines, they pick up right where they left off earlier.

What might happen if an external dependency is not met? For example, what happens if an email service provider has failed. The component that is responsible for sending out emails will observe failed requests. It might retry the request with exponential backoff (or circuit break depending on how you have designed it). But it will remain stuck at that event until the service provider comes back online. Requests from the component will start succeeding at that point of time and it will continue to clear its backlog from the event stream.

Event sourced systems rely on heavy replication of the event log. Even if the entire distributed system has failed, you can rebuild it completely by replaying events from the event log.

## Traceability
Event sourcing comes with traceability built into it. As a request progresses through the system, it leaves behind a trail of events. This helps to identify what the system as a whole is doing with the request, which components of the system act on it, and the time each component takes to process the request. If a request got into an ill state, you want to know with absolute certainty how that happened. Event sourcing does not just make it possible, it also makes it easy - all you need to do is to search through the event log for events relating to the requested entity. 

## Reduction in Storage Costs
Logs are crucial to debugging any system. When you have a distributed system with a large number of instances/processes, and your are saving the application logs of each process in the system, the storage costs quickly add up. If you reduce the retention period, you loose the ability to debug problems that have gone unattended for a while. Also, you loose the ability to explain why the system got into a certain state.

With event sourcing, you can throw away application logs and still retain the ability to debug the system and explain why the system was in a certain state. All you need are the event logs for the session in consideration - just one log file for a session of an entire distributed system (or a set of files if the log is split into multiple segments). That is a massive reduction in storage costs. There is further reduction if events are implemented using space efficient binary formats.

## Lossless Updates
Each update to each entity in the system is emitted as an event. You know precisely why an entity is in a given state at any point of time. This is in contrast with CRUD applications that store the current state of an entity. Such systems apply updates directly to the database, loosing the earlier updates.

With destructive data updates in CRUD applications, recovery might be impossible if the previous state is overwritten with new incorrect state by a buggy application. This can be remediated by preserving the history of the updated entity, but traditional CRUD applications need to go out of their way to do this. And doing this for all entities in the system is a massive undertaking.

Since event sourcing has preservation of history built into it, it helps you identify the correct old state and then helps you fix your mistake by adding a corrective event.

Event sourcing does not loose any information. Most systems cannot boast of that.

## Materialized Views
Relational data stores allow you to create materialized views which are simply tables that store the results of database queries. This is done primarily for two reasons:
* Write representation of data is often very different from read representation of data. Data is normalized when it is persisted in a relational store. But when that data is consumed by an application, it denormalizes it via joins to give the data an expected shape. Materialized view is the read-optimized denormalized shape of that data.
* Frequently running a query against multiple tables in a database can be very expensive. Materialized views act as caches that reduce read loads.

Using event sourcing, you can produce as many read-oriented views of data as you need. A component can produce a materialized view simply by folding all events relating to the entities the component is interested in. Such views can then be used to serve read queries on the system.

Views in event sourced systems provide you with current data. Whenever an event indicating a change in state of a relevant entity is produced, it will be processed by the component that generates the view and will be immediately applied to the view. The data provided by a view will be out-of-date only by the amount the component generating the view is lagging behind the event stream. This would typically be a couple of milliseconds (or even a couple of microseconds if your system is fast enough). This is in contrast with relational data stores where the freshness of data served by the view depends upon how frequently the view is updated. If the view is updated every time one of the underlying tables is updated, it will affect the performance of the data store. If the view is updated manually or on a defined schedule, applications that use it will have to deal with the possibility of the data being stale.

Materialized views are great at generating real time reports such as: get me the number of large shipments delivered today; get me the  number of shipments worth more than $1000 delivered today; get me the number of shipments that were expected to be delivered by today but are still not delivered. Such views keep updating in real time providing you with analytical insights into your system.

## Reduced Load on the Network
A component in a distributed system might rely upon multiple other components to get its job done. For example, a component that is responsible for accepting equity orders from clients might depend upon a component that provides information about the equity instruments to validate the client’s order. It might also rely upon another component to check whether the client has enough credit for the transaction.

There are multiple ways to go about this. In the microservices world, this is accomplished by a service making requests to multiple other services to retrieve all information it needs to do its job. This is done per request received by the service, putting a tremendous load on the network. To minimize this network round-trip, caches are often employed. But then you run into issues like cache coherency and invalidation.

Event sourced systems solve this problem elegantly. A component caches all information that it needs to do its job knowing that whenever relevant information changes, events will be emitted to inform it about the change. This causes an enormous reduction in the amount of information that is passed around leading to optimum network usage.

## Cache Coherency
Distributed systems typically have a single source of truth for a piece of data and multiple copies of it that contain identical records or derived data. This is done to reduce load on the source of truth and to provide better read performance. For example, an Order Management System might maintain a cache with all orders from a client meant to serve read requests from the client.

Such systems need to deal with consistency problems. If data is updated in the source of truth, the update needs to be propagated to all components within the system that hold the data. One way of solving this problem is via dual writes - its the responsibility of the writer to update data in each component of the system that holds it.

Dual writes lead to two kinds of problems:
* **Lost updates**: if the data was written successfully to only a few of the places what is the writer supposed to do? Retry the write at those places where it did not get a confirmation from? What if the write is not idempotent, the request went through, but the response could not get back to the writer? If the write is idempotent but the store is temporarily unavailable (may be it failed or is unavailable due to a netsplit), what is the writer supposed to do? Wait indefinitely? There are no easy answers here.
* **Race conditions**: if two writers update a piece of data in the source of truth, they need to propagate these to all other places in the system that hold the data. These two writes could arrive in different order at different points in the system causing them to disagree with each other. There are intelligent ways of dealing with such race conditions, but they are not as elegant as an event based approach.

Another way of solving this problem is via deletion or cache invalidation. After a writer has updated data in the source of truth, it sends a delete to all caches that hold this data. Updated data will be brought into the cache when it is read next. This solution cannot easily cope with derived data though and is still not as elegant as an event based approach.

Event sourced systems solve the problem of dual writes via sequencing the writes. The updates are emitted as sequenced events and are applied in the same order at every place in the system preventing any race condition. If a write/event could not be applied to a node in the system due to its unavailability, it will be when the node is back online and picks up where it left off in the event stream. At any rate, a cache can always be rebuilt from scratch by consuming the events from the log starting at the beginning of the current session.

## Liberty from Data Stores
Data modelling is an important activity during the design of any system. While modelling the entities of a domain, you want to make sure that you consider all attributes of the entities without any regard to how they might be persisted. This leads to distilled domain objects that are a pleasure to work with in any object oriented programming language.

From an architectural point of view, a data store is merely a detail. A mere tool that is employed by the system to durably persist its entities. The data models used by the system should not be coupled with this tool. But this is often what happens when relational data stores are used.  An Object–Relational Mapping (ORM) tool is brought in to reduce the object-relational impedance mismatch and very soon you find your code littered with references to the ORM tool. Domain entities that are supposed to be sacrosanct, are married to the ORM framework leading to a very ugly dependency. And if you need to migrate to a different data store in the future (may be a non-relational one), it becomes a behemoth task.

NoSQL data stores alleviate the impedance mismatch to a certain extent because they don’t enforce schema on write. But unless the in-memory representation and the data store representation are the same, there will always be impedance mismatch.

Event sourcing allows you to circumvent this problem in a majority of components of the system by letting them work with the domain entities in their object representation (POJOs if you are using Java as the programming language).

The system of record in an event sourced system is the event log. All changes to the entire system are preserved in the event log as a sequence of immutable events. The current state of the entities in the system can be projected to external data stores if needed. Only a small number of components in the system are assigned the responsibility of interfacing with external data stores and any translation between object representation and data store representation of the domain entities is performed here. Event sourcing puts data stores where they belong in the design of a system - a mere implementation detail that does not constrain system performance.

Event sourcing allows you to easily migrate from one data store to another. If you feel that the SQL data store that you have been using is not performant enough, and that you would like to evaluate a NoSQL data store in its stead, you can easily do this by adding another component to the system that interfaces with the NoSQL data store performing the necessary translation while reading from or writing to the data store. Data stores merely hang off of event stream spewing their data as events on the stream and recording projections of entities on the stream.

When event sourcing, you are not limited by the performance of the data stores you employ. Most components in the system don’t write to or read from any external data store. Since components hold state locally in their memory or offload it to disk, in the worst case they put extra strain on the server that they run on without fear of interrupting any other component in the system. You are, however, limited by the performance of the message delivery mechanism you use for example the message broker that is used to pass messages around. Brokers like Apache Kafka are known to regularly do trillions of small sized messages per day, so horizontal scaling via partitioning the event log is achievable.

Event sourcing helps to extract stable performance of your data stores. With an increase in the amount and/or rate of writes to your distributed system, the subsystem that takes a direct hit is the messaging middleware. Writes to the messaging middleware are append only and that scales much better than random writes to any data store. Writes made to the data stores happen at a steady rate by components hanging off the event stream at a pace the data store is comfortable with. When the amount and/or rate of reads on the distributed system increases, it puts an extra burden on those components/caches in the system that supply the reads which in turn puts extra load on the servers those components run on. There is no pressure on the data stores used by the system. This way you could even put a traditional relational data store in the OLTP pathway of a high transactional volume distributed system and not have to worry about the performance of the relational data store.

## Monitoring and Alerting
Observability is crucial to successfully running a system in production. When you distribute a system over a host of servers, you increase the surface area of where things could go wrong.

Event driven architectures are a blessing when it comes to monitoring and alerting. In any system you would like to monitor two things - the processes and the servers. Per process there are multiple things to monitor - liveness, percentage of allotted memory used (JVM heap memory for example), CPU usage, etc. You can create a telemetry log and make each process in the system send their recordings as events/heartbeats at a set frequency. You could run a process per server with the sole responsibility of sending server statistics events into the telemetry log with information such as load average, disk usage percentage, memory usage percentage, etc. The telemetry log can be consumed by alerting components that fire text and email based alerts when processes have not sent heartbeats for a configured duration or when resource usage has crossed a configured threshold. The telemetry data can also be projected onto a live dashboard that reveals the health of various components in the system.

Health checks in such systems are inexpensive. Components broadcast health information of their own accord. If they fail to do so within a configured timeout period, they are deemed as dead by the alerting components which would then send out alerts to inform the team on call. Push based health checks puts less burden on the infrastructure and reduce the possibility of false alarms being generated if a process is caught up in a lengthy computation.

If you need it, you can push the telemetry data to a time series database. You can also preserve the telemetry log for a session for later use, long after the session is over.

Event sourcing makes generating application metrics easy. All changes are emitted as events in the system which are timestamped by the sequencer. This allows you to easily calculate metrics such as rate of request, latency of servicing a request, error rate, etc. This data can again be projected on a dashboard for live monitoring.

## Independent Buildability and Deployability
Each component of an event sourced system can be independently built and deployed. This keeps the release velocity high in such systems.

## Testing
**Functional tests** - these are of two types:
* **Positive tests**: this is when you need to verify that a given action triggered a specific result. Since event sourced components are reactive in nature, this is easy to verify. The given action would be represented by an event on the event stream. The component under consideration must react to this event leading to another event on the stream or sending a message out to an external system.
* **Negative tests**: this is when you need to verify that a given action did not trigger any result. This is again easy to verify. The component upon seeing the event will not take any action (no event in response and no message to another system). If you follow up with a positive test that is verified by the component producing a specific result, you can be absolutely sure that the negative tests pass.

**Regression tests** - event sourcing makes regression tests easy. Suppose you want to do regression tests for a component that sends messages out to an external system. You can take the event log for an entire session from production and play it through two versions of the component - one that ran in production and the one that needs to be regression tested. You write the messages sent by the components into two separate log files while they are playing through production events. Then you can diff the messages produced by the two versions of the component to find out whether there is any regression. In case the component is not supposed to send messages out to an external system, you can write down the commands the components would send into separate log files and then diff them.

## Concurrency
Multithreaded programming is fundamentally hard and is best left to experts who do this on a regular basis. With multithreaded programming you have to deal with synchronizing access to mutable shared data, deadlocks, starvation, and livelocks. Defects in multithreaded applications are often timing related and very hard to reproduce. Multithreaded applications may exhibit non-deterministic behavior if synchronization is not done correctly, depending on the order in which their threads are scheduled for execution and the order in which the instruction streams of the threads are interleaved. This makes debugging such applications very difficult. Extracting good performance out of multithreaded code is hard:
* Coarse grained locking reduces concurrency.
* Fine grained locking increases complexity and reduces performance.
* Context switches waste time and lead to cache misses.

Multithreaded programming makes sense in case of high-end server applications where a single application would utilize most of a server's resources. For example databases, and message brokers. You would not want to colocate a database with another important process in your distributed system on the same machine. This allows the database to utilize all of the cores, disk, and memory on the server that it runs on. To extract maximum performance of that server, it makes sense for the database code to be multithreaded.

For simple server-side applications, single-threaded event-driven programming is better. It gives you code that is easy to reason about, easy to write, easy to test, and easy to debug. Your code will not suffer from the common pitfalls of concurrent programming - race conditions, deadlocks, starvation, and livelocks.

Event sourced applications prefer single threaded components where all event handling is done in a single thread of execution. You can tie the event handling thread of your application to a single core of the underlying server by setting the processor affinity of its process (for example by using Linux's taskset command). If you do this for all of the components that run on a server (i.e. bind each to a separate core of the server), you can max out the CPU utilization. Since processes are tied to cores, you save time on context switches. This also improves the odds of CPU caches being warm leading to an increased number of cache hits and better application response times.

## Application Integration
Events are great not just within a system but also at the boundaries of a system integrating it with other systems in the organization. A system would emit events to inform other systems of appropriate business decisions. For example, in an e-commerce setup, an order management system would emit an event informing a fulfillment system that an order has been created and that the order needs to be fulfilled. Such application integration is done by a component that is often called a Gateway. A Gateway is responsible for translating interesting events within the system to messages sent to external systems and vice versa. Thus events provide a useful mechanism for stitching disparate applications within an organization.

## Operations
Software does not just needs to support its customers, it also needs to support the operations team that keep it running. Great software and great operations go hand in hand. If a member of the operations team is on the phone with a customer and they cannot verify what the customer did with their order or what has happened to their order, the customer will have to wait for answers. If the inputs from a customer or the actions on a customer's order have been thrown away, for example because you only persist the latest state of entities, you loose the ability to easily answer such questions. Under such circumstances, the customer will be disappointed, members of operations will be demoralised, and such issues might need to be escalated to the engineering team. This wastes everyone's time and the company's reputation. Event souring helps with this because it will preserve all actions by a customer and every action that the system takes on the customer's order as events. Tools can be built to query the event log and display all events relating to the customer's order in the sequence in which they happened. Since all events are timestamped by the sequencer, you also know at which point in time an event happened. Such tools can be used by the operations team to answer the customer's questions in time.

Mistakes are often made in production, either by people or by software or by both. When mistakes have been made, you need your production support team to be able to fix them easily. But how do you do this? Do you allow them to make direct updates to the source of truth? What if they make further mistakes? What if those mistakes lead to corrupt data stores? Event sourcing helps you with this by allowing the members of support team to apply corrective actions. These are in the form of events that are appended to the log providing you with a perfect record of the corrective action that was taken, the person who took it, and the time when the action was taken. When these events are projected to the data stores, the entities end up in correct states without a user having to make direct data stores updates. This way your production support team can confidently fix mistakes without having to worry about accidentally corrupting your data store.



