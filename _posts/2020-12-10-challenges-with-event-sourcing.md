---
layout: post
title:  "Challenges with Event Sourcing"
category: "eventsourcing"
---

## Unfamiliar Paradigm
Event sourcing offers an unfamiliar programming model. There aren’t many systems around that use this model. Collaboration via asynchronous message passing is very different from the usual request-response based protocols that most programmers are aware of. As such, not enough engineers are available who know how to program this way. It might take a new engineer in the team a couple of weeks or a even a couple of months to learn. But once they have scaled the learning curve, the rewards are enormous.

## Eventual Consistency
Event sourcing only supports eventual consistency. Writes are made first to the event log which acts as the system-of-record and are then replicated to other components in the system as they process events from the log. The delay between the publication and the consumption of an event is typically few milliseconds but new events relating to the entity under consideration might have been published during this while making the data slightly outdated. Most distributed systems are okay with this sort of eventual consistency. But if your system has stronger consistency requirements, such as linearizability, you would be better off using a distributed data store that offers them (for example Google's Spanner).

## Historical Data
Once an entity has reached the end of its lifecycle, there is no reason to carry it forward to the next session of your system. As such, you loose the ability to answer historical queries with an event sourced system. Such systems are primarily supposed to be used in online transactional pathways. For analytical or historical queries, a data warehouse is a much better option.

## Single Point of Failure
As mentioned earlier, this article presents you with one flavor of event sourcing - one that relies upon a sequencer for atomic broadcast. This makes the sequencer a candidate for single point of failure in the system. It does not necessarily need to be so. You can have multiple replicated sequencers with one of them acting as the leader of the group while the others passively replicate from the event stream. Leader is elected via consensus and whenever the leader is no longer available, the group elects a new one in its place. Automatic failovers via consensus are the new norm. If you don't want to go to the extent of implementing a consensus algorithm yourself, you could rely upon an external software such as Zookeeper. If you want to keep it even simpler, you could rely upon manual failover of the sequencer but then the whole system will not be available until the failover happens.

## Determinism
Event handlers in all components in the system must be deterministic. This ensures that if a component crashes and is restarted, or if a passively replicating component takes over after an active component has failed, the replacement component would be in the exact same state as the failed component was in before failure. If event handling code is non-deterministic, the replacement component will be in a different state and would do wrong things when it takes over.


