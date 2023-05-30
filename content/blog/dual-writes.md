+++
title = "Let's Talk About Dual Writes"
date = "2023-05-27"
in_search_index = true
+++
Most web applications need to store data _somewhere_ to handle persistent
state between requests. Usually this is an OLTP (online transaction processing) database.
This might be a relational database like PostgreSQL but could just as easily be a
document oriented database like Mongo. Depending on your use case this initial storage may
take you pretty far. Databases these days often offer some degree of full text search,
indexes for optimizing geospacial queries, and pub/sub mechanisms. 
At some point though, it is likely you'll end up with additional data stores.

Additional data stores can be obvious such as an OLAP (online analytical processing)
database, Elasticsearch, or caches. They may also be somewhat less obviously databases such as
[Sidekiq](https://github.com/sidekiq/sidekiq) (which is a job queue for Ruby that uses Redis)
or a remote queue system like RabbitMQ or Kafka. I mention this
other category separately because I've noticed that on many teams there is a lot more
care taken to ensure writes to something like Elasticsearch are working than with a
job back end like Sidekiq which "just works".

This post is going to use the P modeling language to describe a common, but flawed approach
to managing multiple storage systems as well as a couple of the available alternatives.
You can find the P source for this on [GitHub](https://github.com/mlh758/blog_dual_write_spec).

If you want more details I recommend all of the following:

* Enjoy the many [Jepsen audits](https://aphyr.com/tags/jepsen) showing all the exciting ways
systems can fail.
* [This blog post](https://www.confluent.io/blog/using-logs-to-build-a-solid-data-infrastructure-or-why-dual-writes-are-a-bad-idea/) from the author of _Designing Data Intensive Applications_ about using logs to replicate changes.
* Just read _Designing Data Intensive Applications_.
* [This video on event sourcing](https://www.youtube.com/watch?v=MYD4rrIqDhA). I like it because it's a good summary of what event sourcing is, what problems it can solve for you, and what tools offer similar solutions for simpler scenarios.

## Choosing Dual Writes

Once you have one or more secondary data stores you have to decide how to keep it in sync with
the primary database. The obvious choice, and one I see chosen frequently, is to write
a value to the primary storage and then write a value to the secondary storage.
If you have several additional storage mechanisms you might write to the primary and then
enqueue a job (itself a write to a remote system) to make the updates to those additional systems.
I've also seen what at first glance appears to be a more sophisticated approach where a write
is done against the primary database then then an event enqueued to notify additional systems
to make their updates. In reality however, this is equivalent to the job mechanism I described
earlier as both rely on a write to a secondary system to succeed.

In Rails this kind of thing is _very_ common even among large community gems. Sidekiq is a
popular job runner that uses Redis to store a job queue. `ActiveRecord` provides lifecycle
callbacks on your data models so that tasks can be run `after_commit`. Gems like [sunspot](https://github.com/sunspot/sunspot)
use that to trigger updates to a search index after a model is updated and the transaction
commits.

The most insidious thing about this approach is that most of the time _it works_.

## The Problem With Dual Writes

When things go wrong with this mechanism it's usually rare, sometimes subtle, and often difficult to
reproduce outside of production where infrastructure is more complicated and traffic volumes are
higher.

Let's assume you have two servers, A and B. Your primary database is PostgreSQL and you
synchronize some updates to a search index. You do this by enqueuing an async job after an
update to the primary database commits.

Here's a couple of things that could go wrong:

1. You write the record to the primary and your job queue is unavailable. Redis might be
out of memory, the network is down, who knows. The search index never gets that update.
1. An update comes to server A to set X = 2 and shortly after an update comes to server B to
set X = 3. These database transactions run and in the primary X = 3. However after the commit
server A pauses for garbage collection. Server B enqueues the task to update X = 3 in the
index and _then_ server A enqueues the task to update X = 2. Now the primary says X = 3
and the search index says X = 2.

You could mitigate some ordering problems with [Lamport Timestamps](https://en.wikipedia.org/wiki/Lamport_timestamp) created in the primary database but only if the updates cause no side effects in the secondary systems. It may also not be clear where your transaction boundaries
are in sufficiently messy code. You may end up eunqueueing jobs in the middle of a large
transaction that gets rolled back. Now you have emitted events for an operation that
never occurred. Even if you get the transaction boundaries right, the potential for
data loss just gets bigger if that job with the accumulated events then falls on the floor.

## Modeling the problem

We'll start by modeling a single database just to verify the base case of our specification
and then move through a couple of problem scenarios and solutions.

A reminder that you can view the full code for this post [on GitHub](https://github.com/mlh758/blog_dual_write_spec) to explore it in more detail.
P has been updated since my last post which simplifies the commands a little. `p compile` builds
a project and `p check -tc <test case> -i <iteration count>` runs the model checker.

### Verifying Consistencey

In all our tests we want to verify that eventually it will always be the case that all
databases recorded the same events in the same order. With P we do that with a liveness property
using `hot` and `cold` states. `AllMatch` is checking that the events which _all_ databases
have recorded match. It will consider `1, 2, 3` and `1, 2` a match and won't verify the third
slot until the other database catches up.

```
cold state WaitForEvents {
    on eWriteRecorded do (write: tWriteRecorded) {
        UpdateDatabase(write);
        dbs = values(databaseStates);
        if (!SameSize()) {
            goto WaitForSequencesToMatch;
        }
        assert AllMatch(), "Sequences are out of order";
    }
}

hot state WaitForSequencesToMatch {
    on eWriteRecorded do (write: tWriteRecorded) {
        UpdateDatabase(write);
        dbs = values(databaseStates);
        assert AllMatch(), "Sequences are out of order";
        if (SameSize()) {
            goto WaitForEvents;
        }
    }
}
```

Databases will announce when they see a value and the spec machine will record that in a
sequence for each database. We know if the databases have a different number of events they
can't be consistent so we transition to a hot state. If they have the same number of events
and yet the events in each slot don't match we've hit a correctness violation.

If we're in a hot state and the databases become consistent we can fall back to a cold state.

### A Single Database

This is just to check that the spec we wrote functions correctly. A single database should always
be consistent with itself. Here's our first pass at a database:

```
machine Database
{
    // store our name so the spec machine can track our state correctly
    var name: string;
    start state Init {
        entry (payload: string) {
            name = payload;
            goto WaitForRequests;
        }
        defer eWriteReq;
    }
    // announce the commands we recieve
    state WaitForRequests {
        on eWriteReq do (val: int) {
            announce eWriteRecorded, (name, val);
        }
    }
}
```

Below is our first test driver. We declare a database, `announce` an init event
to start our spec machine, and then fire a few commands at the database.

```
machine SingleDatabase {
    var database: Database;
    var iter: int;
    var instances: seq[string];
    start state Init {
        entry {
            instances += (0, "Primary");
            announce eMonitor_ConsistencyInit, instances;
            database = new Database(instances[0]);
            while (iter < 10) {
                iter = iter + 1;
                send database, eWriteReq, iter;
            }
        }
    }
}
```

After running `p compile` and `p check -tc tcSingleDatabase -i 100` we should see:

```
..... Found 0 bugs.
```

Hooray.

### Dual Writes, Single Server, Magic Uptime

First, we introduce the server machine. Servers act as the client to the database
and send it a sequence of messages. Each server gets initialized with a primary
database and a potentially empty set of followers. In dual write scenarios it will
write first to the primary and then to the followers. We initialize each server
with a non-overlapping sequence of messages just to keep the sequence of integers
unique. Here is an excerpt of the server machine. I omitted the variable declarations
and init function since they're boring:

```
type tStartServer = (primary: Database, followers: set[Database], startAt: int, messageCount: int);

machine Server {
    // for each message in our range, send it to the primary
    // and then any followers we're aware of to simulate dual writes
    var follower: Database;
    state SendMessages {
        entry {
            while (counter < messageCount) {
                send primary, eWriteReq, startAt + counter;
                foreach (follower in followers) {
                    send follower, eWriteReq, startAt + counter;
                }
                counter = counter + 1;
            }
        }
    }
}
```

Our test case for this scenario is similarly straightforward. `BuildDatabases` just turns
the sequence of names into the `(Database, set[Database])` tuple we need for initializing
servers.

```
machine SingleServerPerfectDualWrites {
    // declarations omitted
    start state Init {
        entry {
            instances += (0, "Primary");
            instances += (1, "Secondary");
            announce eMonitor_ConsistencyInit, instances;
            dbs = BuildDatabases(instances);
            server = new Server((primary = dbs.0, followers = dbs.1, startAt = 0, messageCount = 10));
        }
    }
}
```

Running `p check -tc tcPerfectDualWrites -i 100` provides another `.... Found 0 bugs.`. This makes sense. With no
concurrency or errors this shouldn't fail. This is also why you rarely detect issues with dual writes in development
or staging environments unless you perform load testing and then run scripts to verify consistency of your data.

### Our First Bug

Time to try it with multiple servers writing to the databases. I'm going to omit the test case machine since it looks
just like the last one except we initialize two servers with two different `startAt` values. Running with `-tc tcTwoServerDualWrites`
should find a bug pretty fast. You'll see something like the following:

```
...A bunch of stuff about all the state machines booting up and maybe some initial messages
<SendLog> 'Server(6)' in state 'SendMessages' sent event 'eWriteReq with payload (20)' to 'Database(3)'.
<DequeueLog> 'Database(3)' dequeued event 'eWriteReq with payload (20)' in state 'WaitForRequests'.
<AnnounceLog> 'Database(3)' announced event 'eWriteRecorded' with payload <Primary,20,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Primary,20,>)' in state 'WaitForSequencesToMatch'.
<SendLog> 'Server(5)' in state 'SendMessages' sent event 'eWriteReq with payload (1)' to 'Database(3)'.
<SendLog> 'Server(5)' in state 'SendMessages' sent event 'eWriteReq with payload (1)' to 'Database(4)'.
<DequeueLog> 'Database(3)' dequeued event 'eWriteReq with payload (1)' in state 'WaitForRequests'.
<AnnounceLog> 'Database(3)' announced event 'eWriteRecorded' with payload <Primary,1,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Primary,1,>)' in state 'WaitForSequencesToMatch'.
<DequeueLog> 'Database(4)' dequeued event 'eWriteReq with payload (0)' in state 'WaitForRequests'.
<AnnounceLog> 'Database(4)' announced event 'eWriteRecorded' with payload <Secondary,0,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Secondary,0,>)' in state 'WaitForSequencesToMatch'.
<DequeueLog> 'Database(4)' dequeued event 'eWriteReq with payload (1)' in state 'WaitForRequests'.
<AnnounceLog> 'Database(4)' announced event 'eWriteRecorded' with payload <Secondary,1,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Secondary,1,>)' in state 'WaitForSequencesToMatch'.
<ErrorLog> Assertion Failed: Sequences are out of order
```

Server B sent its payload `20` to the primary, has not yet sent it to the secondary. Server A slips it's payload `1` over to both
the primary and the secondary. In this run the primary saw `0, 20, 1` and the secondary saw `0, 1` before the assertion tripped.

### Our Second Bug

We assume the writes to the primary won't fail. In a live system these might happen within a transaction boundary so if the request
fails it gets rolled back and we return an error to the user. We care about the fallibility of writing to the secondary more
because its harder to fix problems there. If that write fails it probably isn't trivial to undo the transaction that just
completed, and you may not want to return an error to the user since their write _did_ complete on the primary.

We can model this failure with the simpler single server scenario. We just need to make the write to the secondary fallible
in our model. One way to do that is with `$` which tells the model checker to branch. This is the update to the `Server` machine:

```
state SendMessages {
    entry {
        while (counter < messageCount) {
            send primary, eWriteReq, startAt + counter;
            foreach (follower in followers) {
                // now sometimes the secondary won't receive the message
                if ($) {
                    send follower, eWriteReq, startAt + counter;
                }
            }
            counter = counter + 1;
        }
    }
}
```

If we re-run the now imperfectly named `tcPerfectDualWrites` we will either see an output very similar to the above, or a
liveness violation as the sequences fail to converge. If you're following along and want to force a liveness violation you
can remove the assertion in `Consistency.p` or move it so it only gets checked when the sequences have the same length. Some
writes to the secondary are getting lost, the system never converges on a consistent state.

## Doing Better

When you have the server perform dual writes to update followers you are essentially working with leaderless replication.
You have all the problems of such a system, such as write conflicts and asynchronicity, with none of the tools to help
you get to a consistent state like quorum reads/writes. You also don't get any of the performance or additional uptime
benefits that a leaderless system implies.

All of the solutions I describe below are essentially bringing you back to a single-leader style of replication. _One_
data store operates as the source of truth and all the follower databases asynchronously update themselves from a
consistent view of that database. All of these are asynchronous and rely on a log to arrive on a consistent state.

All of these mechanisms also provide _at least once_ delivery semantics. This means that every event in the log will make
it to a follower at least one time. Followers, or the gateways in front of them, will need to account for duplicate
delivery. As far as I'm aware this is unavoidable. If the update mechanism uses something like RabbitMQ the consumer
might commit the update for the event and then crash before sending the `ACK`. Similarly, a Kafka consumer might handle
a message before comitting the updated offset with the broker.

### Change Data Capture

This is essentially how transactional databases keep their followers up to date. After the leader database commits its change
it writes that change set to a log. Followers consume that log to update their own state. [Debezium](https://debezium.io/) is
a tool that does this for some transactional databases. It consumes the WAL of the database and then uses Kafka to create
a log that consumers can pull from so that many secondary systems can update their own state.

Mongo DB provides [change streams](https://www.mongodb.com/docs/manual/changeStreams/) which is a similar concept but logical
replication is provided as a service directly by the database. If you wrote those change events to a durable log like Kafka
you would have a very similar system to Debezium. The [resume token](https://www.mongodb.com/docs/manual/changeStreams/#resume-a-change-stream) provides a mechanism for recovering from failure. The recipient system can commit a token, such
as the offset or logical timestamp, in the same transaction it commits the results of processing the event so that it
does not process duplicates. The [Kafka docs](https://docs.confluent.io/kafka/design/delivery-semantics.html#consumer-receipt)
describe something like this.

If your database provides a CDC mechanism or is supported by a CDC tool like Debezium this is a pretty good bet.

### Transactional Outbox

Typically a microservices pattern, [transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) is
also useful for performing logical replication. This pattern relies on your primary database having the ability to update
multiple tables/collections within a single transaction. Most relational databases provide this. _Some_ document databases
also provide this but often come with a substantial performance penalty.

The idea here is that within the same transaction as your other updates, you write events to an outbox collection. If the
transaction fails the events never become visible outside that transaction. You might use an additional service to pull from
that outbox and write to a log that other systems pull from, or expose an API that allows other services to pull events with a
checkpoint value that they track internally.

If you're already using a transactional database, this can be an easy pattern to get started with and help close the gaps
in your existing tools without having to stand up new infrastructure.

### Event Sourcing

This flips the replication mechanism on its head. All events are recorded in an append-only data store and other databases
derive a view of that data by pulling events from that log. There are some more details [here](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) or the oodles of blogs and videos describing the pattern. For the purpose of consistency
alone it may be overkill but there are some additional advantages.

* Your application is built from an audit log. You know your log is complete because all visible state is derived from it.
* Writes are effectively serialized around the log which can mitigate some concurrency issues.
* Appending events to storage is very fast.

There are some headaches you need to be aware of though:

* The system is async all the way through. If a user expects to read their own writes immediately after they hit submit you'll
need to delay their read until the follower you're interested in catches up.
* It becomes harder to set constraints on your system because you usually check those conditions against the state of the database.
That database is probably behind. You'll need to think about compensating transactions to get your system back into a state that is
considered valid for your business rules.
* The oft-touted ability to replay all events in history becomes a lot more complicated when the handlers for those events might
interact with third party systems. You may also need to log the responses of all external systems so they can be replayed as well.

You might be able to use a transactional database as an event store. This would allow you to commit the events and process them on
_one of_ your views within the same transaction. Users would be able to read their own writes and you could do some consistency
checks. Other systems would update from the event log as normal.
However, it would be very easy to write to the derived tables directly and not actually use the event log which could
undermine its integrity. This could also make your primary datastore a throughput bottleneck if you were looking at event
sourcing for performance reasons.

## Modeling a Solution

The model we're creating would work for either change data capture or transactional outbox. They're essentially different
implementations of the same approach.

### Simplified, but Incomplete

We're going to start with a simplified version of our replication log. It gets us to the state of having a leader
database with followers and passes writes to followers through the leader instead of the server.

A database can now be either leading or following:

```
state WaitForRequests {
    on eWriteReq do (val: int) {
        txId = txId + 1;
        announce eWriteRecorded, (name, val);
        foreach (replica in replicas) {
            send replica, eUpdateFollower, (txId = txId, value = val);
        }
    }
}

state Following {
    on eUpdateFollower do (payload: tFollowerMsg) {
        announce eWriteRecorded, (name, payload.value);
    }
}
```

The replicator machine sits between the leader database and its follower. This is your Kafka consumer pulling events
off the log to store updates or whatever mechanism you choose to update the log into the follower. We could just as
easily model this system without the `Replicator` but it's going to have a little more logic soon and I didn't want
to bloat the `Database` machine too much. For the moment it's just proxying update events to a follower:

```
machine Replicator {
    state SyncingFollower {
        on eUpdateFollower do (payload: tFollowerMsg) {
            send target, eUpdateFollower, payload;
        }
    }
}
```

We then set up the test case:

```
machine LeaderFollower {
    ... declarations
    start state Init {
        entry {
            instances += (0, "Leader");
            instances += (1, "Follower");
            announce eMonitor_ConsistencyInit, instances;
            // initialize the follower first and assign it to the replicator
            follower = new Database((name = "Follower", replicas = default(set[Replicator]), isLeader = false));
            replicators += (new Replicator(follower));
            // leader knows about its replicas, servers only talk to the leader
            leader = new Database((name = "Leader", replicas = replicators, isLeader = true));
            servers += (new Server((primary = leader, followers = default(set[Database]), startAt = 0, messageCount = 10)));
            servers += (new Server((primary = leader, followers = default(set[Database]), startAt = 20, messageCount = 10)));
        }
    }
}
```

Running this shows no bugs, but we know this isn't quite complete. Let's talk about at least once delivery again.

### Errors updating the follower

First off, we're going to assume writes to a follower _eventually_ succeed. If the follower crashes and never
recovers we clearly cannot achieve consistency. We know that a follower might crash before sending an `ack` or
that the response can get lost on the network. We should model what that looks like and account for it.

We'll model the `Replicator` like it's a consumer from the log writing into the database. This is a synchronous
action so it will wait for a reply confirming the write. If it fails it will retry until it succeeds. This is
how we maintain event ordering but it's also why we can't do it inside the request handler process. Replication
lag is acceptable to some extend in a background process, not at all so within a request handler in the server.

On the database side we'll just model the "reply never comes" problem. Errors are handled the same way so it would
just be noise in the model.

```
state Following {
    on eUpdateFollower do (payload: tFollowerMsg) {
        announce eWriteRecorded, (name, payload.message.value);
        if ($) {
            send payload.replyTo, eAckWrite;
        }
    }
}
```

The event payload has been updated to contain the machine to reply to and the actual event payload is under the
`message` property of the event. Because of the `$` sometimes we'll just drop the reply on the floor.

The replicator implementation is now:

```
state SyncingFollower {
    on eReplicateWrite do (payload: tReplicateMsg) {
        writeSucceeded = false;
        while (!writeSucceeded) {
            send target, eUpdateFollower, (replyTo = this, message = payload);
            receive {
                case eAckWrite: {
                    writeSucceeded = true;
                }
            }
        }
        
    }
}
```

We keep trying to send the message to the follower until it replies before moving on to the next message. If
we run the spec now we should get an error like the following:

```
<SendLog> 'Replicator(4)' in state 'SyncingFollower' sent event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' to 'Database(3)'.
<ReceiveLog> Replicator(4) is waiting to dequeue an event of type 'eAckWrite' or 'PHalt' in state 'SyncingFollower'.
<DequeueLog> 'Database(3)' dequeued event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' in state 'Following'.
<AnnounceLog> 'Database(3)' announced event 'eWriteRecorded' with payload <Follower,0,>.<MonitorLog> PImplementation.
ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Follower,0,>)' in state 'WaitForSequencesToMatch'.
// notice the lack of ack
...
<ErrorLog> Deadlock detected. Replicator(4) is waiting to receive an event, but no other controlled tasks are enabled.
```

That one makes sense, we're not timing out our wait so the replicator is effectively dead waiting on a network
reply that never comes. I pulled in a `Timer` module from the P lang's example repo and added that to the
replicator. The receive block now waits on an `ack` or times out at which point it will retry the message.

```
receive {
    case eAckWrite: {
        writeSucceeded = true;
        CancelTimer(timer);
    }
    case eTimeOut: {
        print format("Timed out with tx: {0}, retrying", payload.txId);
        StartTimer(timer);
    }
}
```

You probably know where this is going.

```
<SendLog> 'Replicator(4)' in state 'SyncingFollower' sent event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' to 'Database(3)'.
<ReceiveLog> 'Replicator(4)' dequeued event 'eTimeOut' in state 'SyncingFollower'.
<PrintLog> Timed out with tx: 1, retrying
<DequeueLog> 'Database(3)' dequeued event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' in state 'Following'.
<AnnounceLog> 'Database(3)' announced event 'eWriteRecorded' with payload <Follower,0,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Follower,0,>)' in state 'WaitForSequencesToMatch'.
<SendLog> 'Replicator(4)' in state 'SyncingFollower' sent event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' to 'Database(3)'.
<ReceiveLog> Replicator(4) is waiting to dequeue an event of type 'eAckWrite', 'eTimeOut' or 'PHalt' in state 'SyncingFollower'.
<DequeueLog> 'Database(3)' dequeued event 'eUpdateFollower with payload (<replyTo:Replicator(4), message:<txId:1, value:0, >, >)' in state 'Following'.
<AnnounceLog> 'Database(3)' announced event 'eWriteRecorded' with payload <Follower,0,>.<MonitorLog> PImplementation.ConsistencyInvariant is processing event 'eWriteRecorded with payload (<Follower,0,>)' in state 'WaitForSequencesToMatch'.
<ErrorLog> Assertion Failed: Sequences are out of order
<StrategyLog> Found bug using 'random' strategy.
```

This isn't what I expected, but it also makes sense. In this case the replicator timed out waiting on the
send to the follower so it retried. However, both messages actually made it to the follower _eventually_, we just
didn't wait long enough for the `ack`. Of course in the real world it's hard to tell how long is long enough so
you need to account for such things.

### Successful Replication

We're already passing around a transaction ID generated in the leader so we have the information we need
to handle duplicate messages that come in on the follower.

```
state Following {
    on eUpdateFollower do (payload: tFollowerMsg) {
        if (payload.message.txId > txId) {
            announce eWriteRecorded, (name, payload.message.value);
            txId = payload.message.txId;
        }
        
        if ($) {
            send payload.replyTo, eAckWrite;
        }
    }
}
```

We only process (store) the command if the transaction ID is greater than our last seen transaction. Because the replicator
will keep trying indefinitely until it knows the message was stored, these duplicates are still in processing order. Once
the replicator knows the command was saved it can move on to the next one. _Eventually_ our databases become consistent.
If you look over the checker output you'll see that on some schedules it takes quite a while to get there between timeouts
and dropped `ack` replies but we get there in the end.

Mission Accomplished.

## Reflecting on P

The P language, like TLA+ is a relatively niche project. Microsoft has been supporting it well and 2.0 is a step towards
being easier to use with simplified installation and commands. I have a lot of hope for this project.

The main pain points right now, at least for me, are:

* Compiler output is pretty simplistic. Better compiler errors would help, but I always managed to get there eventually.
* There's not really any syntax highlighting available. It's mentioned on the site but at least in vscode it doesn't seem
to work. This makes the compiler errors more important. I found myself in more of a code/compile/fix/repeat loop than in
other development I do.
* I used the wrong iterator variable once and figuring out why I had an index out of range error was _tough_ since I got
more stack trace from the internals than from the P spec itself. I had to add a bunch of print statements since I only
knew which state in which machine rather than which line was causing the problem.

P programs are supposed to be high level models rather than full fledged implementations so some of these issues
are tolerable in a modeling language while they would be a deal breaker in a full fledged implementation language.

## Distributed Transactions

Instead of asynchronous replication you _could_ attempt a distributed transaction protocol like Two Phase Commit.
That's usually not a viable option. 2PC reduces throughput, concurrency, and reliability. You may also be using a
database without a concept of a rollback meaning you'll have to build a complex protocol over your storage to
serialize writes. There's a pretty good post [here](https://dbmsmusings.blogspot.com/2019/01/its-time-to-move-on-from-two-phase.html)
but the limitations of 2PC are written about in many places.