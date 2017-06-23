- Feature Name: Change Feeds
- Status: draft
- Start Date: 2017-06-13
- Authors: Arjun Narayan and Tobias Schottdorf
- RFC PR: TBD
- Cockroach Issue: #9712, #6130

# Summary

Add a basic building block for change feeds. Namely, add a command `ChangeFeed` served by `*Replica` which, given a (sufficiently recent) HLC timestamp and a set of key spans contained in the Replica returns a stream-based connection that

1. eagerly delivers updates that affect any of the given key spans, and
1. periodically delivers "closed timestamps", i.e. tuples `(timestamp,
   key_range)` where receiving `(ts1, [a,b))` guarantees that no future update
   affecting `[a,b)` will be sent for a timestamp less than `ts1`.

The design overlaps with incremental Backup/Restore and in particular [follower reads][followerreads], but also has relevance for [streaming SQL results][streamsql]

The design touches on the higher-level primitives that abstract away the Range level, but only enough to motivate the Replica-level primitive itself.

# Motivation

Many databases have various features for propagating database updates to
external sinks eagerly. Currently, however, the only way to get data out of
CockroachDB is via SQL `SELECT` queries, `./cockroach dump`, or Backups
(enterprise only). Furthermore, `SELECT` queries at the SQL layer do not support
incremental reads, so polling through SQL is inefficient for data that is not
strictly ordered. Anecdotally, change feeds are one of the more frequently
requested features for CockroachDB.

Our motivating use cases are:

- wait for updates on individual rows or small spans in a table. For example, if
  a branch is pushed while its diff is viewed on Github, a notification will pop
  up in the browser to alert the user that the diff they're viewing has changed.
  This kind of functionality should be easy to achieve when using CockroachDB.
  Individual developers often ask for this, and it's one of the RethinkDB/etcd
  features folks really like.
- stream updates to a table or database into an external system, for example
  Kafka, with at-least-once semantics. This is also known as Change Data Capture
  (CDC). This is something companies tend to ask about.
- implement efficient incrementally updated materialized views. This is
  something we think everyone wants, but does not dare to ask.

The above use cases were key in informing the design of the basic building block
presented here: We

- initiate change feeds using a HLC timestamp since that is the right primitive
  for connecting the "initial state" (think `SELECT * FROM ...` or `INSERT ...
  RETURNING CURRENT_TIMESTAMP()`) to the stream of updates.
- chose to operate on a set of key ranges since that captures both collections
  of individual keys, sets of tables, or whole databases (CDT).
- require close notifications because often, a higher-level system needs to
  buffer updates until is knows that older data won't change any more; a simple
  example is wanting to output updates in timestamp-sorted order. More
  generally, close notifications enable check pointing for the case in which a
  change feed disconnects (which would not be uncommon with large key spans).
- emit close notifications with attached key ranges since that is a natural
  consequence of the Range-based sharding in CockroachDB, and fine-grained
  information is always preferrable. When not necessary, the key range can be
  processed away by an intermediate stage that tracks the minimum closed
  timestamp over all tracked key spans and emits that (with a global key range)
  whenever it changes.
- make the close notification threshold configurable via `ZoneConfig`s (though
  this is really a requirement we impose on [16593][followerreads]). Fast close
  notifications (which allow consumers to operate more efficiently) correspond
  to disabling long transactions, which we must not impose globally.
- aim to serve change feeds from follower replicas (hence the connection to
  [16593][followerreads]).
- make the protocol efficient enough to poll whole databases in practice.

Note that the consumers of this primitive are always going to be
CockroachDB-internal subsystems that consume raw data (similar to a stream of
`WriteBatch`). We propose here a design for the core layer primitive, and the
higher-level subsystems are out of scope.

# Detailed design

## Types of events

The events emitted are simple:

- `key` was set to `value` at `timestamp`, and
- `key` was deleted at `timestamp`.

The exact format in which they are reported are TBD. Throughout this document, we assume that we are passing along `WriteBatch`es, though that is not a hard requirement. Typical consumers will live in the SQL subsystem, so they may profit from a simplified format.

## Replica-level

Replicas (whether they're lease holder or not) accept a new `ChangeFeed` command
which contains a base HLC timestamp and (in the usual header) a key range for
which updates are to be delivered. The `ChangeFeed` command first grabs
`raftMu`, opens a RocksDB snapshot, registers itself with the raft processing
goroutine and releases `raftMu` again. By registering itself, it receives all
future `WriteBatch`es which apply on the Replica, and sends them on the stream
to the caller (after suitably sanitizing to account only for the updates which
are relevant to the span the caller has supplied).

The remaining difficulty is that additionally, we must retrieve all updates made at or after the given base HLC timestamp, and this is what the engine snapshot is for. We invoke a new MVCC operation (specified below) that, given a base timestamp and a set of key ranges, synthesizes ChangeFeed notifications from the snapshot (this is possible if the base timestamp does not violate the GC threshold).

Once these synthesized events have been fed to the client, we begin relaying close notifications. We assume that close notifications are driven by [follower reads][followerreads] and can be observed periodically, so that we are really only relaying them to the stream. Close notifications are per-Range, so all the key ranges (which are contained in the range) will be affected equally.

When the range splits or merges, of if the Replica gets removed, the stream
terminates in an orderly fashion. The caller (who has knowledge of Replica
distribution) will retry accordingly. For example, when the range splits, the
caller will open two individual ChangeFeed streams to Replicas on both sides of
the post-split Ranges, using the highest close notification timestamp it
received before the stream disconnected. If the Replica gets removed, it will simply reconnect to another Replica, again using its most recently received close notification.

Initially, streams may also disconnect if they find themselves unable to keep up with write traffic on the Range. Later, we can consider adding backpressure, but this is out of scope for now.

## MVCC-level

We need a command that, given a snapshot, a key span and a base timestamp,
synthesizes a set of `WriteBatch`es with the following property: If all MVCC
key-value pairs (whether insertions or deletions) with a timestamp larger than
or equal to the base timestamp were removed from the snapshot, applying the
emitted `WriteBatch`es (in any order) would restore the initial snapshot.

Note that if a key is deleted at `ts1` and then a new value written at `ts2 >
ts1`, we don't have to emit the delete before the write; we give no ordering
guarantees (doing so would require a scan in increasing timestamp order, which
is at odds with how we lay out data on-disk).

However, when so requested, we vow to emit events in ascending timestamp order
for each individual key, which is easy: Different versions of a key are grouped
together in descending timestamp order, so once we've created `N` `WriteBatch`es
for this key, we simply emit these `N` items in reverse order once we're done
with the key. This mode of operation would be requested by `ChangeFeed`s which
listen for on an individual key, and which don't want to wait for the next close
notification (which may take ~10s to arrive) until they can emit sorted updates
to the client (see the Github example).

Note: There is significant overlap with `NewMVCCIncrementalIterator`, which is
used in incremental Backup/Restore.

## DistSender-level

The design is hand-wavy in this section because it overlaps with [follower
reads][followerreads] significantly. When reads can be served from followers,
`DistSender` or the distributed SQL framework need to be able to leverage this
fact. That means that they need to be able to (mostly correctly) guess which, if
any, follower Replica is able to serve reads at a given timestamp. Reads can be
served from a Replica if it has been notified of that fact, i.e. if a higher
timestamp has been closed.

Assuming such a mechanism in place, whether in DistSender or distributed SQL, it
should be straightforward to entity which abstracts away the Range level. For
example, given the key span of a whole table, that entity splits it along range
boundaries, and opens individual `ChangeFeed` streams to suitable members of
each range. When individual streams disconnect (for instance due to a split) new
streams are initiated (using the last known closed timestamp for the lost
stream) and any higher-level consumers notified that they need to drop their
buffers for the affected key range.

## Implementation suggestions

There is a straightforward path to implement small bits and pieces of this
functionality without embarking on an overwhelming endeavour all at once:

First, implement the basic `ChangeFeed` command, but without the base timestamp
or close notifications. That is, once registered with `Raft`, updates are
streamed, but there is no information on which updates were missed and what
timestamps to expect. In turn, the MVCC work is not necessary yet, and neither
are [follower reads][followerreads].

Next, add an experimental `./cockroach debug changefeed <key>` command which
launches a single `ChangeFeed` request through `DistSender`. It immediately
prints results to the command line without buffering and simply retries the
`ChangeFeed` command until the client disconnects. It may thus both produce
duplicates and miss updates.

This is a toy implementation that already makes for a good demo and allows a
similar toy implementation that uses [streaming SQL][streamsql] once it's
available to explore the SQL API associated to this feature.

A logical next step is adding the MVCC work to also emit the events from the
snapshot. That with some simple buffering at the consumer gives at-least-once
delivery.

Once follower reads are available, it should be straightforward to send close
notifications from `ChangeFeed`.

At that point, core-level work should be nearly complete and the focus shifts to
implementing distributed SQL processors which provide real changefeeds by
implementing the routing layer, buffering based on close notifications, and to
expose that through SQL in a full-fledged API. This should allow watching simple
`SELECT` statements (anything that boils down to simple table-readers with not
too much aggregation going on), and then, of course, materialized views.

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

# Concerns

## Performance

Performance concerns exist mostly in two areas: keeping the follower reads
active (i.e. proactively closing out timestamps) and the MVCC scan to recover
the base timestamp.

We defer discussion of the former into its own RFC. The concern here is that
when one is receiving events from, say, a giant database which is mostly
inactive, we don't want to force continuous Range activity  even if there aren't
any writes on almost all of the Ranges. Naively, this is necessary to be able to
close out timestamps.

The second concern is the case in which there is a high rate of `ChangeFeed`
requests with a base timestamp and large key ranges, so that a lot of work is
spent on scanning data from snapshots which is not relevant to the feed. This
shouldn't be an issue for small (think single-key) watchers, and may ultimately
be a non-issue. Change Data Capture-type change feeds are very likely to be
singletons and long-lived. Thus, it's possible that all that's needed here are
good diagnostic tools and the option to avoid the snapshot catch-up operation
(when missing a few updates is OK).

## Ranged updates

Note that the existing single-key events immediately make the notification for, say, `DeleteRange(a,z)` very expensive since that would have to report each individual deletion. This is similar to the problem of large Raft proposals due to proposer-evaluated KV for these operations.

Pending the [Revert RFC][revertrfc], we may be able to track range deletions more efficiently as well, but a straightforward general approach is to attach to such proposals enough information about the original batch to synthesize a ranged event downstream of Raft.

# Unresolved Questions

## CDC

This design aims at killing all birds with one stone: short-lived watchers,
materialized views, change data capture. It's worth discussing whether there are
specialized solutions for, say, CDC, that are strictly superior and worth having
two separate subsystems, or that may even replace this proposed one.

## Licensing

We will have to figure out what's CCL and what's OSS. Intuitively CDT sounds like it could be an enterprise feature and single-column watches should be OSS. However, there's a lot in between, and the primitive presented here is shared between all of them.


[followerreads]: https://github.com/cockroachdb/cockroach/issues/16593
[revertrfc]: https://github.com/cockroachdb/cockroach/pull/16294
[streamsql]: https://github.com/cockroachdb/cockroach/pull/16626