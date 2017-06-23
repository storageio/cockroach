- Feature Name: Change Feeds
- Status: draft
- Start Date: 2017-06-13
- Authors: Arjun Narayan and Tobias Schottdorf
- RFC PR: TBD
- Cockroach Issue: #9712, #6130

# Summary

Add a basic building block for change feeds which, given a (sufficiently recent) HLC timestamp and a set of key spans returns a stream-based connection that

1. eagerly delivers updates that affect any of the given key spans, and
1. periodically delivers "closed timestamps", i.e. tuples `(timestamp,
   key_range)` where receiving `(ts1, [a,b))` guarantees that no future update
   affecting `[a,b)` will be sent for a timestamp less than `ts1`.

# Motivation

Many databases have various features for propagating database updates to
external sinks eagerly. Currently, however, the only way to get data out of
CockroachDB is via SQL `SELECT` queries, `./cockroach dump`, or Backups
(enterprise only). Furthermore, `SELECT` queries at the SQL layer do not support
incremental reads, so polling through SQL is inefficient for data that is not
strictly ordered. Change feeds are one of the more frequently requested features
for CockroachDB.

Our motivating use cases are:

- wait for updates on individual rows or small spans in a table. For example, if
  a branch is pushed while its diff is viewed on Github, a notification will pop
  up in the browser to alert the user that the diff they're viewing has changed. This kind of functionality should be easy to achieve when using CockroachDB.
- stream updates to a table or database into an external system, for example Kafka, with at-least-once semantics. This is also known as Change Data Capture (CDC).
- implement efficient incrementally updated materialized views.

The above use cases were key in informing the design of the basic building block presented here: We

- initiate change feeds using a HLC timestamp since that is the right primitive for connecting the "initial state" (think `SELECT * FROM ...` or `INSERT ... RETURNING CURRENT_TIMESTAMP()`) to the stream of updates.
- chose to operate on a set of key ranges since that captures both collections of individual keys, sets of tables, or whole databases (CDT).
- require close notifications because often, a higher-level system needs to buffer updates until is knows that older data won't change any more; a simple example is wanting to output updates in timestamp-sorted order. More generally, close notifications enable check pointing for the case in which a change feed disconnects (which would not be uncommon with large key spans).
- emit close notifications with attached key ranges since that is a natural consequence of the Range-based sharding in CockroachDB, and fine-grained information is always preferrable. When not necessary, the key range can be processed away by an intermediate stage that tracks the minimum closed timestamp over all tracked key spans and emits that (with a global key range) whenever it changes.
- make the close notification threshold configurable via `ZoneConfig`s (though this is really a requirement we impose on [16593][followerreads]). Fast close notifications (which allow consumers to operate more efficiently) correspond to disabling long transactions, which we must not impose globally.
- aim to serve change feeds from follower replicas (hence the connection to [16593][followerreads]).
- make the protocol efficient enough to poll whole databases in practice.

Note that the consumers of this primitive are always going to be CockroachDB-internal subsystems that consume raw data (similar to a stream of `WriteBatch`). We propose here a design for the core layer primitive, and the higher-level subsystems are out of scope.

# Detailed design

## Replica-level [wip]

All Replicas accept a new `ChangeFeed` command which contains a timestamp and (in the usual header) a key range for which updates are to be delivered. The `ChangeFeed` command first grabs `raftMu`, notes the current applied log index, opens a RocksDB snapshot, registers itself with the raft processing goroutine and releases `raftMu` again. By registering itself, it receives all future `WriteBatch`es which apply to the replica




# Background and Related Work

Change feeds are related to, but not exactly the same
as,
[Database Triggers](https://www.postgresql.org/docs/9.1/static/sql-createtrigger.html).

Database triggers are arbitrary stored procedures that "trigger" upon
the execution of some commands (e.g. writes to a specific database
table). However, triggers do not usually give any consistency
guarantees (a crash after commit, but before a trigger has run, for
example, could result in the trigger never running), or atomicity
guarantees. Triggers thus typically provide "at most once" semantics.

In contrast, change feeds typically provide "at least once"
semantics. This requires that change feeds publish an ordered stream
of updates, that feeds into a queue with a sliding window (since
subscribers might lag the publisher). Importantly, publishers must be
resilient to crashes (up to a reasonable downtime), and be able to
recover where they left off, ensuring that no updates are missed.

"Exactly once" delivery is impossible for a plain message queue, but
recoverable with deduplication at the consumer level with a space
overhead. Exactly once message application is required to maintain
correctness on incrementally updating materialized views, and thus,
some space overhead is unavoidable. The key to a good design remains
in minimizing this space overhead (and gracefully decommissioning the
changefeed/materialized views if the overhead grows too large and we
can no longer deliver the semantics).

Change feeds are typically propagated through Streaming Frameworks
like [Apache Kafka](https://kafka.apache.org)
and
[Google Cloud PubSub](https://cloud.google.com/pubsub/docs/overview),
or to simple Message Queues like [RabbitMQ](https://www.rabbitmq.com/)
and [Apache ActiveMQ](http://activemq.apache.org/).


# Scope

A change feed should be registerable against all writes to a single
database or a single table. The change feed should persist across
chaos, rolling cluster upgrades, and large tables/databases split
across many ranges, delivering at-least-once message semantics. The
change feed should be registerable as an Apache Kafka publisher.

Out of scope are filters on change feeds, or combining multiple feeds
into a single feed. We leave that for future work on materialized
views, which will consume these whole-table/whole-database change
feeds.

It should be possible to create change feeds transactionally with
other read statements: for instance, a common use case is to perform a
full table read along with registering a change feed for all
subsequent changes to that table: this way, the reader/consumer can
build some initial state from the initial read, and be sure that they
have not missed any message in between the read and the first message
received on the feed.

Finally, a design for change feeds should be forward compatible with
incrementally updated materialized views using the timely+differential
dataflow design.

# Related Work

[RethinkDB change feeds](https://rethinkdb.com/docs/changefeeds/ruby/)
were a much appreciated feature by the industry community. RethinkDB
change feeds returned a `cursor`, which was possibly blocking. A
cursor could be consumed, which would deliver updates, blocking when
there were no new updates remaining. Change feeds in RethinkDB could
be configured to `filter` only a subset of changes, rather than all
updates to a table/database.

The
[Kafka Producer API](http://docs.confluent.io/current/clients/confluent-kafka-go/index.html#hdr-Producer) works
as follows: producers produce "messages", which are sent
asynchronously. A stream of acknowledgement "events" are sent back. It
is the producers responsibility to resend messages that are not
acknowledged, but it is acceptable for a producer to send a given
message multiple times. Kafka nodes maintain a "sliding window" of
messages, and consumers read them with a cursor that blocks once it
reaches the head of the stream. While in the worst case a producer
that has lost all state can give up, and restart

# Strawman Solutions

An important consideration in implementing change feeds is preserving
the ordering semantics provided by MVCC timestamps: we wish for the
feed as a whole to maintain the same ordering as transaction timestamp
ordering.

## Approach 1: Polling

The first approach is to periodically poll the underlying table,
performing a full table `Scan`, looking for any changes. This would be
very computationally expensive.

A second approach is to take advantage of the incremental backup's
`MVCCIncrementalIterator`: this iterator is given _two_ ranges: a
start and end key range, and a start and end timestamp. It iterates
over keys in the key range that have changes between the start and end
timestamps. If the timestamps are configured correctly, this would
then allow us to sweep over all the changes since the previous scan,
in effect running incremental backups and sending the changes.

If this is done on a per-range basis, each incremental sweep is
limited to the 64mb range size, keeping it to a reasonable maximum
size. Furthermore, this workload can be performed on a follower
replica, removing the pressure on leaseholder replicas. While every
store has some leader replicas, in the presence of a skewed write
workload, this would relieve some IO pressure on the leaders of
rapidly mutating ranges.

These per-range updates, however, do have to be aggregated (and
ordered) by a single coordinator, which must be fault tolerant.

The biggest downside to this approach is the continuous reads that are
performed  regardless of actual changes. Even a passive range with
minimal or no updates would have frequent scans performed, which
combined with the read amplification, result in lots of IO
overhead. This thus scales with the amount of data stored, not the
volume of updates.

## Approach 2: Triggers on commit with rising timestamp watermarks

A second approach is for ranges to eagerly push changes on commit,
directly when changes are committed. Ranges would eagerly push
updates, along with the timestamp of the update to a coordinator. The
biggest complication here comes from the design
post-[Proposer-Evaluated KV](proposer_evaluated_kv.md). WriteBatches
are now opaque KV changes after Raft replication. Taking those
WriteBatches and reconstructing the MVCC intents from them is a
difficult task, one best avoided. Furthermore, intents are resolved at
the range level on a "best-effort" basis, and cannot be used as the
basis of a consistent change-feed.

### Interlude: MVCC Refresher

As a quick refresher on CockroachDB's MVCC model, Consider how a
multi-key transaction executes:

1. A write transaction `TR_1` is initiated, which writes three keys:
   `A=a`, `B=b`, and `C=c`. These keys reside on three different ranges,
   which we denote `R_A`, `R_B`, and `R_C` respectively. The
   transaction is assigned a timestamp `t_1`. One of the keys is
   chosen as the leader for this transaction, lets say `A`.

2. The writes are first written down as _intents_, as follows: The
   following three keys are written:
    `A=a intent for TR_1 at t_1 LEADER`
    `B=b intent for TR_1 at t_1 leader on A`
    `C=c intent for TR_1 at t_1 leader on A`.

3. Each of `R_B` and `R_C` does the following: it sends an `ABORT` (if
   it couldn't successfully write the intent key) or `STAGE` (if it
   did) back to `R_A`.

4. While these intents are "live", `B` and `C` cannot provide reads to
   other transactions for `B` and `C` at `t>=t_1`. They have to relay
   these transactions through `R_A`, as the leader intent on `A` is
   the final arbiter of whether the write happens or not.

5. `R_A` waits until it receives unanimous consent. If it receives a
   single abort, it atomically deletes the intent key. It then sends
   asynchronous cancellations to `B` and `C`, to clean up their
   intents. If these cancellations fail to send, then some future
   reader that performs step 4 will find no leader intent, and presume
   the intent did not succeed, relaying back to `B` or `C` to clean up
   their intent.

6. If `R_A` receives unanimous consent, it atomically deletes the
   intent key and writes value `A=a at t_1`. It then sends
   asynchronous commits to `B` and `C`, to clean up their intents. If
   they receive their intents, they remove their intents, writing `B=b
   at t_1` and `C=c at t_1`. Do note that if these messages do not
   make it due to a crash at A, this is not a consistency violation:
   Reads on `B` and `C` will have to route through `A`, and will stall
   until they find the updated successful write on `A`.

However, the final step is not transactionally consistent. We cannot
use those cleanups to trigger change feeds, as they may not happen
until much later (e.g. in the case of a crash immediately after `TR_1`
atomically commits at `A`, before the cleanup messages are sent, after
`A` recovers, the soft state of the cleanup messages is not recovered,
and is only lazily evaluated when `B` or `C` are read). Using these
intent resolutions would result in out-of-order message delivery.

Using the atomic commit on `R_A` as the trigger poses a different
problem: now, an update to a key can come from _any other range on
which a multikey transaction could legally take place._ While in SQL
this is any other table in the same database, in practice in
CockroachDB this means effectively any other range. Thus every range
(or rather, every store, since much of this work can be aggregated at
the store level) must be aware of every changefeed on the database,
since it might have to push a change from a transaction commit that
involves a write on a "watched" key.

A compromise is for changefeeds to only depend on those ranges that
hold keys that the changefeed depends on. However, a range (say `R_B`)
are responsible for registering dependencies on other ranges when
there is an intent created, and that intent lives on another range
(say `R_A`). This tells the changefeed that it now depends on
`R_A`, and it registers a callback with `R_A`.

# Challenges

## Fault tolerance/recovery

Approach 1 is very resilient to failures: an incremental scan needs to
only durably commit the latest timestamp that was acknowledged, and
use that timestamp as the basis for the next MVCC incremental
iteration.

Approach 2, however, is more complex if a message is not
acknowledged. The range leader must keep a durable log of all
unacknowledged change messages, otherwise recovery is not possible . This
adds significant write IO for supporting change feeds.

## In order message delivery aggregated across ranges

Combining messages from multiple ranges on a single "coordinator" is
required to order messages from multiple ranges. Scaling beyond a
single coordinator requires partitioning change feeds, or giving up
in-order message delivery.

Ranges thus send messages to a changefeed coordinator, which is
implemented as a DistSQL processor.

## Scalability

Scalability requires that the coordinator not have to accumulate too
many buffered messages, before sending them out. In order to deliver a
message with timestamp `t_1`, the coordinator needs to be sure that no
other range could send a message with a write at a previous timestamp
`t_2 <= t1`.

## Latency minimization


##


## Proposal: watermarked triggers on commit with incremental scans for
failover recovery

We propose combining the two approaches above.



1. Ranges eagerly push updates that are committed on their keyspace.
2. When a range registers an intent, it pushes a "dependency"




# updates notes

- make a kv operation similar to `Scan`, except with an additional
  "BaseTimestamp" field. When that Scan executes, its internal
  MVCCScan uses a `NewTimeBoundIter(BaseTimestamp, Timestamp)` instead
  of a regular iter. That means it only returns that which has changed
  since the latest such poll. That operation also "closes out"
  Timestamp, i.e. for the affected key range, nothing can write under
  `Timestamp` any more. Ideally Timestamp will always trail real time
  a few seconds, but it does not have to. We could let it run with low
  priority. Potential issues when monitoring a huge key space, but we
  should somehow use DistSQL anyway to shard this fire hose. By
  reduction we may assume we're watching a single range.
- We can already implement ghetto-changefeeds by doing the polling
  internally and advancing: BaseTimestamp := OldTimestamp, Timestamp
  := Now().
- Real changefeeds will want to be notified of new committed
  changes. Seeing how the base polling operation above works, it
  doesn't really help to do this upstream of Raft (and doing so loses
  capability of reading from a follower). This is the really tricky
  part - we need to decide whether we can afford dealing only with the
  case in which you monitor a sufficiently small key range (so that we
  can assume we're only looking at a small handful of ranges). If
  that's enough, we can hook into the Raft log appropriately and could
  actually uses that to stream commands unless we fall behind and a
  truncation kicks us out. For larger swathes of keyspace, we need to
  add something to the Replica state so that whatever the leader is,
  it knows to send changes somewhere. We're going to need a big
  whiteboard for this one! Also think about backpressure etc.




# Pushing updates from the transactional layer

So far, we have discussed processors that find efficiencies in high
latency/deep dataflow settings. This creates (potentially) a more
efficient dataflow execution model for query execution. For
materialized views, however, we need to stream *change notifications*
from ranges:

* Each processor registers a change notification intent to all range
  leaseholders that are inputs to its dataflow.

* Ranges send the triple of <change notification(+/- tuple),
  CockroachDB transaction timestamp, RangeID> to the leaf processor.

* Ranges also keep track of a "low watermark" of their transaction
  timestamps --- this is the lowest transaction timestamp that a Range
  might assign to a CockroachDB write.

* A range keeps track of all the triples that it has sent out, ordered
  by transaction timestamp. When the low watermark rises to or above
  the timestamp of a triple it has sent out, it emits a close
  notification for that timestamp.

* A range also occasionally (~1s) heartbeats close notifications for
  timestamps at its node low watermark, if no write notification has
  been sent in that duration. Thus, processors always receive some
  monotonically increasing timestamp.

You will note that the triple includes a RangeID. This is because
ranges on different nodes can feed into a single processor, and that
processor might want to order messages from multiple nodes into a
single stream. However, nodes might be under different contention
loads, and thus, might be closing their watermarks differently. Thus,
a close watermark is only valid for all messages with earlier
timestamps *from the same node*. This thus forms a (different) partial
ordering over the <timestamp, rangeID> pair. Once two ranges close
timestamps *t_1, n_1* and *t_2, n_2*, where *t_1 < t_2*, we can
eliminate the range numbers, recovering the total ordering.

Similarly, if a logical processor node is replicated into multiple
physical processor nodes operating on parallel streams, this
introduces another dimension into the Timelystamp partial order

<work through an example here>

Thus, timelystamps are composed of a sequence of (Timestamp, NodeID)`
pairs, with the special partial order above defined internally on a
pair, and the standard cartesian product partial order on tuples of
pairs.

<definitely work through an example here>

# Unresolved Questions

## Licensing

We will have to figure out what's CCL and what's OSS. Intuitively CDT sounds like it could be an enterprise feature and single-column watches should be OSS. However, there's a lot in between, and the primitive presented here is shared between all of them.


[followerreads]: https://github.com/cockroachdb/cockroach/issues/16593