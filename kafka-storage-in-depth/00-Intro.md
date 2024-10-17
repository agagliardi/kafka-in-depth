# Kafka

## Preface
This presentation assumes a basic knowledge of Kafka's main concepts, especially topic, partition and segment.

![Kafka Components](../resources/10-KafkaBasicConcepts.svg)


## Introduction
Kafka is sort of a _dumb pipe_ for storage. It is designed for _event storming_. That is, handling large volumes of small messages with near real-time stream processing needs.

When an event is produced to Kafka, it is simply appended to the _topic_. When an event is consumed, it is simply retrieved at the next offset. 

It has been implemented to work with _immutable records_ as fast as close to the storage I/O limits with a small and mostly constant overhead. Optimised storage handling is the key feature of Kafka.

---
##### Note

- Event: Mostly just refers to the concept of recording something that has happened. It is used as a synonym for message.
- Message: It is the payload of data transmitted between producers and consumers.
- Record: It is the element stored in Kafka. It is used as a data format for storage and includes key, value and metadata.

---

The Kafka architecture is built on two layers, the _Control Plane_ (CP) and the _Data Plane_ (DP).  The CP is responsible for managing the metadata of a cluster. It is essentially the Zookeeper or KRaft part of the job. The DP handles the actual data management, client request processing and replication. Storage is in charge to the DP. 

There are other key concepts that will be part of other presentations:

- Replication
- Consumer Group Protocol (partition allocation and offset, rebalancing)
- Idempotent producer (durability, availability, message ordering)

There are also ancillary concepts that could be part of future presentation like:

- Transactions 
- Tiered storage

## Data layer
Kafka was not designed to be fast in memory with eventual persistence. Instead, it was designed to take advantage of the sequential performance of very inexpensive disk. It writes to disk immediately, without flushing, relying on pagecache and avoiding garbage collectors as much as possible. In fact, the garbage collector is the kryptonite of Kafka.

Kafka storage leverages the linear cost of sequential reads and writes typical of magnetic disks. A good old magnetic disk in RAID could be as fast as a state-of-the-art NVME because seek time is not as important.

There are two other common I/O drawbacks that Kafka addresses. The small and frequent I/O operations and byte copying. We will see the first addressed by Record Batch and the second by Standardised Record.

Kafka storage uses the concepts of _Record_ and _Record Batch_, not _Message_ (as per _Message Exchange Patterns_). Kafka is designed for fast persistence rather than fast transfer. It is more `affine` to a database than to a message broker. In fact, Kafka Message has no built-in routing capabilities (e.g. reply address, hop counter, etc.), but it does have, for example, compression.

### Record Batch

![Record Batch](../resources/20-RecordBatch.svg)

Records are always handled in batches by the producer. Record batches and records have their own headers. Record batches carry a fixed size (61 bytes) of metadata that includes compression configuration, producer details and timestamps. Grouping messages into a batch, i.e. large network packets, large sequential disk operations with large contiguous blocks of storage, has many benefits for all I/O:
- Allows network requests to group messages together
- reduce network round trips
- the server appends chunks of messages to its log in one go
- the consumer fetches large linear chunks at a time. 
- linear writes

It is at the record batch level that messages are compressed. They are compressed all together, not one at a time. They are also stored and transmitted compressed. 

To make it easy to understand, the producer has a big responsibility in message handling, including part of the persistence. Producer not only selects the partition to write to, but also prepares the RecordBatch, compressed if necessary. It will pack some RecordBatchs in _Producer Request_, which will be sent to the partition leader's broker.


### Message Storage Round Trip

The producer request reaching the broker is parked in a _Socket Receive Buffer_. A network thread picks it up and adds it to the request queue. Then an I/O thread pops the message, does some validation (possibly decompressing) before appending it to the _commit log_. A commit log, as in many databases, is a set of segments, each consisting of two files. The _.log_, which stores the data, and the _.index_, which contains the record offsets. But this is not the end of the story.

Because disk flushes are not synchronous (by default), the broker waits for the replication to complete before giving the thumbs up to the producer (ack). Meanwhile, requests are parked in a map theatrically called _purgatory_. 
After replication, the broker pops a request from the purgatory map and drops a response in the response queue. It then makes the return trip using the network threads and the _Socket Send Buffer_.

To retrieve messages, a consumer client sends a fetch request to the broker with details such as topic, partition and offset. It makes the same trip to the I/O thread. It checks that the fetch request matches the _.index_, then loads data from the _.log_ into a response, and so on.

To prevent clients from hammering the broker when there are no new messages, requests are parked in _purgatory_ while waiting for them.


### Files

1. `.log`
Batch records are written to files called log files in append-only mode. Deletion is a separate task that _should_ not affect read/write operations. Log files contain some metadata and sequences of Record Batch as they have been received. Then the same when read, record batch are transferred to the consumer as it is in the file.
2. `.idx`.
Contains offsets and physical file positions for messages in the order they were appended to the log. 
3. `.timeindex`. 
This indexes messages by their timestamp for time-based message retrieval. It contains a timestamp and an offset of the index.
4. `.txnindex`. 
Contains information about aborted transactions for each partition segment. It only exists if a transactional producer has been used with the topic. It allows Kafka to properly handle and recover transactions.
5. `.snapshot`. 
It maps unique producer IDs (PIDs) to message sequence numbers. It exists only when an idempotent producer publishes to a topic partition (only once). 
6. `.leader_epoch_checkpoint`. 
This is where Kafka checkpoints the last recorded leader epoch and the leader's last offset upon becoming leader. Leader epochs are used to track leadership changes for a partition, ensuring that each leader has a unique identifier. By checkpointing the latest leader epoch and offset, Kafka can pick up where it left off in the event of failures or restarts, maintaining data consistency and availability.


### Performance Key Points

#### Standardised message

Messages is a fairly minimal structure designed to fit the memory. In fact, the message format is the on-disk format. In addition, the same standardised message format is used throughout the Kafka layer, including network transfers. This avoids the byte-copying bottleneck. Obviously, the structure starts with the length, which is useful for buffer allocation, then attributes, but mostly as a delta from the record batch, followed by key and value details. At the end are the headers, since they are not a zero-day feature (as of 0.11).

![Record](../resources/30-Record.svg)


*Kafka's standardised messages allow it to take advantage of PageCache and Zero-copy to achieve blazingly fast performance.

#### Zero-copy

Zero-copy is the kernel feature (sys call `sendfile()` mapped by Java `FileChannel#transferTo`) to transfer data directly from the read buffer to the socket buffer in kernel space. Using encryption aka SSL prevents zero-copy (`SSL_sendfile` [is currently not supported](https://kafka.apache.org/documentation/#maximizingefficiency)).

- Transfer with SSL (without zero-copy)

Disk → Kernel Buffer → User Space Buffer (SSL) → Kernel Network Buffer → Network

- Zero Copy Transfer

Disk → Kernel Buffer → Network

#### PageCache

PageCache is a kernel cache for disk `pages' implemented using paging memory management. The operating system caches pages in unused portions of RAM, transparently to applications. This is why it is so important to have lots of free memory on a broker's machine.

---
 DEMO TIME sendfile/SSL

---

## Storage internals

### Segment

Kafka uses an implementation of _write-ahead logging_ called _commit-log_ to persist records. This is quite similar to the persistence of most databases. Again, Kafka is more like a database than a message broker.

*Note that in the Kafka documentation, names are often ambiguous. 

Records are appended to the end of a _log_ implemented as a _partition_, which in Kafka is just a folder with the name of the _topic_ plus a progressive number.Each log is divided into _segments_. In Kafka, segments are _.log_ files. The _segment_ stores a sequence of records.

---
##### Note
Segment storage is highly configurable, but only at the broker level. It is not possible to configure it per part or per topic. It is also not possible to use a different mount point because Kafka only uses the `log.dir' folder. The workaround is to use different brokers for different storage and assign partitions accordingly. 

---

#### Segment characteristics

1. Sequentiality

Log segments are sequentially appended files in which Kafka stores its data on disk. Each log segment contains a contiguous sequence of records, ordered by their offset, which represents the position of the record in the log.

2. Immutability

Once a log segment is written, it becomes immutable, meaning that its contents cannot be changed. Immutability ensures data integrity and simplifies replication, as replicas can be created by simply copying the segment without worrying about concurrent writes.

3. Rolling

Data is continuously added to a segment. It grows until it reaches a configurable size or time threshold.
When a segment reaches this threshold, it is closed for writing and a new segment is opened to continue storing incoming records.
This process is known as segment rolling and ensures that segments remain manageable in size, facilitating efficient storage management and retrieval.

4. Retention and Deletion

Kafka has topic retention policies that specify how long data should be retained in the log file. Once data exceeds the retention threshold (time and size), it becomes eligible for deletion during log cleanup processes. This means that only the consumer will not receive any records from it. The log file will be deleted at any time in the future based on the LogCleaner settings.

5. Compacting

Log compaction is a deletion strategy that ensures that at least the most recent value for each key in a topic is retained by removing old values. It is not enabled by default and has some notable limitations:

- The record must have a key
- compacted records are still subject to the segment retention policy
- Compacting only occurs when the segment is closed, because old or large enough active segments are not compacted.
- Compacting is done by LogCleaner starting from the most dirty (i.e. how many records to delete) closed segment.

6. Tombstone
A tombstone is a special message with a null value that is used to 'suggest' a record for deletion and is deleted during the compacting process (if enabled). It is only a suggestion because the null value record is still visible to other consumers.


#### Segment file size

A small size is better for compacting. With compaction enabled, if the segment is filled slowly, it will take a lot of time to be compacted. In this scenario it is better to reduce the segment size to allow compaction to run sufficiently.

Large size is better for saving resources. Fewer open files means less memory and more partitions per single broker (if disk space allows). Especially with fast producers, using large segments reduces the effort to keep up with them because fewer file operations are required.


#### SSD write amplification

Kafka's commit log design can have a significant impact on SSD write amplification.

Flash memory works by reading and writing data in pages (typically 4KB), but erasing data at the block level (typically 128KB-256KB).

However, *blocks must be erased before pages can be rewritten*. This means that even if you need to overwrite a small piece of data, the entire block must be erased and rewritten, causing _write amplification_.

Write amplification is caused by

1.  Overwriting
Unlike HDDs, SSDs can't overwrite data in place. Data must be erased before it can be rewritten. When you update a file, SSDs typically write the new data to an unused block and mark the old block for deletion. This results in both the new data being written and the old data being erased and eventually consolidated (resulting in more writes).

2. Garbage collection
SSDs use garbage collection to reclaim space from blocks that contain deleted or obsolete data. During this process, valid data is moved from partially filled blocks to new blocks, freeing up old blocks for deletion. However, this movement of data causes additional writes, contributing to write amplification.

3. Wear leveling
SSDs implement wear leveling to ensure that all blocks receive an even number of write/erase cycles (to prevent certain blocks from wearing out faster than others). This process involves moving data around, causing additional writes.

Kafka commit logs, like most WAL implementations, can cause SSD write amplification. Write amplification occurs when more data is written to the SSD than the logical writes requested by the application. Some Kafka features increase the effect of write amplification.

Kafka appends records sequentially and is SSD friendly because SSDs perform better with sequential writes than random writes. However, Kafka log management features such as compaction and deletion create SSD write amplification. 

When Kafka deletes old segments due to retention policies or deletes tombstone markers in compaction, this results in more write cycles to the SSD. Because SSDs can't modify any part of a block, even small changes require the entire block to be rewritten, causing write amplification.

This mismatch between logical writes and physical erase size is one of the key drivers of write amplification.

Larger log segments result in fewer segment rolls and less frequent erasures, reducing write amplification.


Retention policies also play a role in write amplification. When logs are deleted, entire segments are removed, potentially triggering SSD block erasures and writes.

Tune retention and compaction settings to balance storage utilisation and performance to reduce unnecessary writes.

SSDs implement internal garbage collection to manage free space and ensure efficient writes. Frequent Kafka erasures could trigger the SSD's GC too much, leading to write amplification.


Userfuyl link:
- https://www.confluent.io/blog/apache-kafka-architecture-and-internals-by-jun-rao/ 
- https://developer.confluent.io/courses/architecture/broker/
- https://andriymz.github.io/kafka/kafka-disk-write-performance/#zero-copy-data-transfer
- https://kafka.apache.org/documentation/#messageformat
- https://strimzi.io/blog/2021/12/17/kafka-segment-retention/
- https://conduktor.io/blog/understanding-kafka-s-internal-storage-and-log-retention
- https://medium.com/@damienthomlutz/deleting-records-in-kafka-aka-tombstones-651114655a16
