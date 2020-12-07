---
layout: post
title:  "An Introduction to Event Sourcing"
---

Event sourcing is a way of building a distributed system wherein every change to the system is captured as an ordered sequence of events. Events are immutable and are only appended to a log. This log serves as the system of record for the whole distributed system.

The state of any entity is persisted in the system of record as a series of events (deltas). The current state of an entity is calculated by any process in the system by accumulating the series of deltas relating to the entity. This process is also known as a projection.

Consider the following example from accounting:
<table class="table">
  <tr>
    <th>Serial No.</th>
    <th>Date</th>
    <th>Transaction</th>
    <th>Amount</th>
    <th>Type</th>
  </tr>
  <tr>
    <td>1</td>
    <td>Jan 10, 2020</td>
    <td>Account Open</td>
    <td>$0</td>
    <td>NA</td>
  </tr>
  <tr>
    <td>2</td>
    <td>Jan 10, 2020</td>
    <td>Cash Deposit</td>
    <td>$100</td>
    <td>Credit</td>
  </tr>
  <tr>
    <td>3</td>
    <td>Jan 30, 2020</td>
    <td>Salary for Jan</td>
    <td>$9,500</td>
    <td>Credit</td>
  </tr>
  <tr>
    <td>4</td>
    <td>Feb 1, 2020</td>
    <td>Rent Payment</td>
    <td>$1,040</td>
    <td>Debit</td>
  </tr>
  <tr>
    <td>5</td>
    <td>Feb 2, 2020</td>
    <td>Utilities Payment</td>
    <td>$156</td>
    <td>Debit</td>
  </tr>
  <tr>
    <td>6</td>
    <td>Feb 15, 2020</td>
    <td>Check Deposit</td>
    <td>$250</td>
    <td>Credit</td>
  </tr>
</table>

You open a salary account with your favourite bank. The bank does not hold a single record for your account that captures the current state/balance of the account. Rather, the bank records every transaction that your account has gone through since its inception. The current value of your account at the end of Feb 15, 2020 then, is simply the sum total of all changes to your account since creation = $100 + $9, 500 - $1, 040 - $156 + $250 = $8, 654. When this is modelled as an event sourced system, each of those transactions will be an event relating to your account. The accounting software will calculate your account balance by applying the events to your account value at origin ($0). 

Event sourcing is not just for the financial industry. Event sourcing can be used to build any near real-time distributed system. No system, distributed or otherwise, should be built such that you cannot explain how the system (or any entity in it) got into a certain state. Unexpected things happen and when you are dealing with large scale systems, for example when you do millions or billions of transactions per day, unexpected things happen all the time. When they do, you want to be able to figure out how they did. Event sourcing does not just make this possible. It also makes it easy.

Event sourcing comes in multiple flavors. This article presents you with one of those. Throughout this article you will be provided with examples from two systems built using this paradigm:
* An Order Management System (OMS) - a system that is responsible for managing the lifecycle of financial orders.
* A Shipment Management System (SMS) - a system that is responsible for managing the lifecycle of shipments.

## What is an Event?

An event mentions a fact about the distributed system. Events usually represent actual actions in the business domain. For example, a financial instrument is available for trading, a shipment is expected at a facility, an order has been received by the system, or a shipment is in transit to another facility. Events are also used to disseminate information about various components of the system. For example, a heartbeat event informs interested parties about the fact that a component is online and fulfilling its duties. 

The data in a distributed system can be classified under two categories:
* Static data: this is data that changes infrequently. Static data events are emitted by the system to apprise all components within the system of plain facts about the system. For example:
	* In an Order Management System you might have InstrumentAdded events wherein each InstrumentAdded event provides information about an instrument that is now available for trading. Per instrument, the event might provide multiple attributes - type of instrument (equity, option, etc.), symbol of the instrument, strike price (if it is an option), etc.
	* In a Shipment Management System you might have FacilityAdded events wherein each FacilityAdded event provides information about a facility (i.e. a building) that can process a shipment. A FacilityAdded event could provide multiple pieces of information about a facility such as - facility ID, facility name, facility address, etc.
* Dynamic data: this is data that changes often. Dynamic data events are typically events relating to the entities that the distributed system is managing. For instance:
	* In an OMS, few types of such events might be:
		* OrderPending - a client has sent an order that needs to be processed by a component in the system.
		* OrderAccepted - a pending order has been found to be valid and has been accepted for processing.
		* OrderRejected - a pending order has been found to be invalid and has been rejected.
		* OrderExecuted - an order has been partially/completely filled.
	* In an SMS, few types of such events might be:
		* ShipmentExpected - a shipment is expected at a facility.
		* ShipmentArrived - a shipment has arrived at a facility.
		* ShipmentAccepted/ShipmentRejected - a shipment that arrived at a facility has either been accepted or rejected after validations.
		* ShipmentInTransit - shipment is in transit to another facility.
		* OutForDelivery - shipment is out for delivery to the recipient.

You might not find this distinction between static and dynamic data in most places. But I think this is an  important distinction to make because processing of dynamic data is often contingent upon the presence of static data and the two come from different places. For example, you will not be able to ship goods to a certain address if you did not know whether the address is serviceable by your system or the facility in your logistics system that serves as the destination facility for the given address. The serviceability of addresses is more or less static and is inherent to your system. A client will only provide you with the shipping address for a shipment which will serve as dynamic data for your system.

Events are immutable and are only appended to a log. Immutability of events preserves the history/timeline of the system and is crucial for it to function correctly (more on that later). This log is often called an event log (a physical manifestation of an event stream) and is very different from application logs produced by components in the system.
Event log acts as the system of record for the whole system. Each component in the system might have its own data store, but the component will rely upon the event log as the source of truth, applying the events to its data store. 

## Ordering of Events

Event sourced systems rely upon a FIFO total order broadcast of events. FIFO implies that if event e1 was sent before event e2 by a component, then whenever both events are delivered to any component in the system, e1 is delivered before e2. Total order broadcast implies that the same set of events is delivered to all components in the system and in the same order. This enables the components in the system to behave as replicated state machines.

The event handler in any component in the system is supposed to be deterministic. This implies that given a state of a component, an event causes a deterministic state transition in the component. Components in the system start with no state or fixed initial state, and have the same sequence of events applied to their event handler. This implies that you can have as many replicas of a component as necessary. Hence, the term replicated state machines.

Replicated state machines make event sourced systems fault tolerant -  if a component crashes, restart it (fixing the reason behind the crash, for example an OutOfMemoryError); if a machine fails, spin up a new machine as a replacement with all components on the crashed machine. The restarted component or its replica picks up where the crashed one left (its slight more involved than that but more on that later). 

A component in the system is assigned the responsibility of ordering/sequencing the events in the system. Such a component is often called a sequencer. Events in a session of the system start at sequence number 1 with each new event having a sequence number that is one greater than the sequence number of the last event.

## Stateful  Processing

Components in an event sourced system typically start with no state. They build their state by reading from the event log, starting with the first event, and applying the events one after the other to their existing state. Since the application state is derivable from the event log, it can be thrown away at any point of time and rebuilt from the event log. This makes local state fault tolerant.

There are multiple advantages of this form of processing:
* A component does not need to bother itself with keeping its state in a persistent data store. If the component crashes, it can simply be restarted and it will rebuild its state by processing the events in the log from the beginning. And since event handling is deterministic, it will end up with the exact same state before it crashed.
* The entire state of a component can be held in its memory. This assumes that the state of the component is small enough to fit in its memory. If not, either some of it could be offloaded to its disk, or the data processing could be partitioned so that the state of each component fits in its memory.
* This yields higher performance. You no longer need to make a round trip to the data store to fetch latest data. You have it in memory of the component.
* There is no dependence on a translation layer that converts between component state and database state. You end up with very clean code.

This form of stateful processing is in contrast with the stateless processing popularized by microservices architecture. Statefulness in this case brings about co-location of data with its processing leading to orders of magnitude better performance than is possible with external data stores.

## Sessions and Snapshots

So you know that the state of any component can be built from scratch by playing all the events from the beginning of the log. But do we really need to do this? What if the event log grows too big? Would this not take too much time?

Well, if you really write your code in an efficient way, you should be able to play through few million events per second. Remember that we are dealing with clean and efficient code that relies upon in memory application state. Such a processing should be really fast.

Assuming that the event log has seen 15 billion events and that a component can play through 10 million events per second (while rewinding with events being consumed in batches), if the component is started past the 15 billionth event mark, it will take the component 25 minutes to catch up with the latest events. That is not too bad. But as the system runs for a while we will see trillions, quadrillions, quintillions of events. Playing events starting at the beginning of the log will clearly not scale.

This is the reason why event sourced systems deal with sessions and snapshots. When the whole distributed system is started/rebooted it is said to enter a new session. Events in this session will start at sequence number 1 and will go upwards from there. Each component in the system will see the events starting at sequence 1 of the current session and will keep progressing their individual states as new events are read from the log.

Normally during the course of a session, components only go through the new events in the session. If a component crashes, or a new component is spun up in the middle of a session, it will have to go though all events of the session starting with sequence number 1. Such an operation would typically take seconds or minutes depending upon the volume of events the distributed system has seen and the rate of event processing of the component.

Snapshots provide continuity between sessions. If a dynamic entity remains in a non-terminal state at the end of one session, we need to ensure the entity is available in the next session as well for processing. For example, if a shipment could not be delivered in one session, someone might be interested in working on it in the next session as well. This is achieved via snapshots. A snapshot contains the final state for all non-terminal dynamic entities at the end of a session. For example, if you still had a million active shipments in the current session, you will write the final state of these shipments to a snapshot. The snapshot of one session is used to initiate the next session of the system. For example, if a shipment was in transit to facility x at the end of current session, the start of the next session will bear an event that says the shipment is in transit to facility x.

When a new session of the system is started, the first set of events you add to the system are the static data events - events that inform all components within the system of data that is more or less static for the system and that the components within the system rely upon to get their job done. So for an Order Management System you will add events about the instruments in the system. For a Shipment Management System you will add events about the logistics network (the facilities that exist, the routes within the logistics network, etc.). The next set of events to add to the system are events that provide dynamic data read from a snapshot of the previous session. After this preparatory stage, your current session is ready to do its job.

How long should a session be? The length of a session is guided by various factors:
* Some distributed systems naturally have the concept of a session. For example, an equities trading system would normally have sessions that start in the morning and end in the evening to mirror the daily trading sessions of the stock exchanges.
* Other systems might need to be available round the clock to ensure continuous processing. For example, a logistics system needs to be available 24 * 7 and any downtime would not be acceptable. In such systems, we need smooth transition from one system to the next with zero downtime. The duration of a session in such a system will be directed by the volume of events seen by the system and the amount of time a component needs to rebuild its state post a crash. This implies a monthly, weekly, or even a daily session provided a component can comfortably rebuild its state rewinding from the start of the event log in a couple of minutes (if not seconds).

<img src="/assets/img/CommandsAndEvents.png" alt="Commands and Events"/>

## Commands and Events

Event sourcing comes in multiple flavors and this article presents you with one of those. In this flavor, we rely upon a FIFO total order broadcast of events in the system. This is done by a component in the system called the sequencer. All events of the entire distributed system are emitted by the sequencer which orders the events by attaching a linearly increasing sequence number to them.

If no other component in the system is allowed to emit events, how do they notify other components in the system of a state change? This is achieved via commands. A command is a request to the sequencer to emit an event, after it has been sequenced, to inform other components in the system of a state change. 

All messages passed around in an event sourced system are either commands or events. All components in the system, except the sequencer, consume events and produce commands. The sequencer consumes commands and produces events.

Thus, such systems have two kinds of logs/streams - a stream of all events produced by the sequencer and a stream of all commands produced by disparate components in the system. The event stream is sequenced while the command  stream is not. The entire command stream need not be sequenced even though the commands emitted by a single component in the system need to be (so that the sequencer can deal with duplicates in case of retries).

An event sourced system might have other streams as well, but it needs to have at least these two. You might want to have a heartbeat stream where every component periodically puts a heartbeat event out to let an alerting component know that it is still alive. You might want to have an administrative stream wherein you put administrative events (via command line or graphical tools) to instruct components within the system to do certain things - such as shutdown, go inactive/active etc. The sequencer is responsible for providing an order only to the stream of domain events to encode causality between them. Other event streams need not have a total order, even though events emitted by any individual component will still be ordered.

## Components

Each process in a distributed system, with a cohesive set of responsibilities, is called a component. For example, the sequencer is a component that is responsible for assigning sequence numbers to the events in the system. 

The source code of each component in the system is usually a module by itself. This is done to ensure independent development and deployment of the component. 

For instance, in an Order Management System some of the components that you might find are:
* A client proxy - an entry point into the OMS used by clients to send orders.
* A gateway - an exit point of the OMS that is used to send orders to exchanges, other OMSs, etc.
* An order router - a component that determines the exchanges to route a client order to.

## Message Delivery

Messages in event sourced systems are of two kinds - commands and events. The messages are passed around via some sort of message-oriented middleware. 

High throughput and low latency trading systems frequently rely upon UDP multicast for delivery. Since all components in such systems sit in a single data center with a very reliable network, the overhead that TCP incurs for reliable delivery is deemed unworthy. The interaction protocols in such systems have safeguards built into them to deal with out-of-order delivery or message loss. These systems also need an event store to hold the events so that any process that has crashed or has been started in the middle of a session can rewind and catch up with the event stream.

UDP as a transport mechanism is suitable for systems that need to be extremely low latency. For others, TCP is good enough. Such systems rely upon a message bus or a message broker as their middleware. Brokers such as Apache Kafka come with an additional advantage - they act as a replicated store of messages. You no longer need to invest resources building or purchasing an event store. Kafka provides it to you for free. 

Event sourced systems also distinguish between two modes of message delivery - push and pull. In push based systems (such as ones that use a UDP multicast based middleware), events are pushed to components. While this can reduce the latency of such systems, it has the potential to overwhelm components that do a lot in their event handling code or interface with slow external consumers (such as writing to a relational data store). This implies that such components will have events queued up for processing with possible data loss. This is mitigated by putting a big queue between such systems and the messaging middleware. In pull based systems, the components read events off of the message broker. Latency in such systems tend to be higher but they have the advantage that a component consumes events at a pace it is comfortable with. 

## Rewinding from the Stream

Any component in an event sourced system that is started in the middle of a session, either for the first time or after a crash (and hopefully after fixing the reason behind crash), needs to start processing events starting at the beginning of the event log. This is called rewinding from the event stream. When the process has seen the latest event on the stream, it is said to have caught up.

A component typically exists in two states:
* Passive: the component is reading from the event stream and is applying the events to its internal state. It does not send any commands out in this state and if it is supposed to interface with external systems, it will not send messages out to them either. A component in usually in a passive state under two circumstances:
	* When it is rewinding from the event stream.
	* When it acts as a hot standby replica for another component.
* Active: the component is processing the latest events from the event stream, is sending commands out, and if it is supposed to interface with external systems, it sends messages out to them as well.

## Record and Replay

The event log is an append-only log of immutable events. You can save the event logs for each session of your system for later use. 

The event log for a session could be played through a single component in the system or, if needed,  through the entire system. This could be done for various purposes such as reproduction of bugs or  regression tests. 

Record and replay provides you with a time machine - it allows you to reconstruct the state of an entire  distributed system at any point in time of its existence. Just like Git allows you to check out a given commit of your repository, resetting the repository to the given historical commit, event sourcing allows you to set the entire system at a given point of its history so that you can inspect how the system looked in the moment.


## Wrapping Up

Event sourcing is not a new idea although the name relatively is. Multiple systems that we use daily, for example version control systems like Git, database replication systems like Write-Ahead Log (WAL) shipping, do something to the same effect.

In my opinion, event sourcing is the best way of building near real-time distributed systems. The financial industry in America trusts billions of its dollars every day with event sourcing. Don’t you think this way of building applications would be good enough for you? Event sourcing is simple at its core. And simplicity scales while complexity does not. Even if you don’t go with event sourcing, fully asynchronous event driven architectures are the way forward.

