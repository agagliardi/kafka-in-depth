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

There are other key concepts not covered in this presentation such as

- Consumer Group Protocol (partition allocation and offset, rebalancing)
- Idempotent producer (durability, availability, message ordering)

There are also ancillary concepts that have emerged over time, such as
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

##### `.log`
Batch records are written to files called log files in append-only mode. Deletion is a separate task that _should_ not affect read/write operations. Log files contain some metadata and sequences of Record Batch as they have been received. Then the same when read, record batch are transferred to the consumer as it is in the file.
##### `.idx`.
Contains offsets and physical file positions for messages in the order they were appended to the log. 
##### `.timeindex`. 
This indexes messages by their timestamp for time-based message retrieval.
##### `.txnindex`. 
Contains information about aborted transactions for each partition segment. It only exists if a transactional producer has been used with the topic. It allows Kafka to properly handle and recover transactions.
##### `.snapshot`. 
It maps unique producer IDs (PIDs) to message sequence numbers. It exists only when an idempotent producer publishes to a topic partition (only once). 
##### `.leader_epoch_checkpoint`. 
This is where Kafka checkpoints the last recorded leader epoch and the leader's last offset upon becoming leader. Leader epochs are used to track leadership changes for a partition, ensuring that each leader has a unique identifier. By checkpointing the latest leader epoch and offset, Kafka can pick up where it left off in the event of failures or restarts, maintaining data consistency and availability.


### Performance Key Points

#### Standardised Message

Messages is a fairly minimal structure designed to fit the memory. In fact, the message format is the on-disk format. In addition, the same standardised message format is used throughout the Kafka layer, including network transfers. This avoids the byte copying bottleneck. Obviously, the structure starts with the length, which is useful to allocate the buffer, then attributes, but mostly as a delta from the record batch, followed by key and value details. At the end are the headers, because they are not a zero-day feature (since 0.11).

![Record](../resources/30-Record.svg)


*Kafka's standardised messages allow it to take advantage of PageCache and Zero-copy to achieve blazingly fast performance.

#### Zero-copy

Zero-copy is the kernel feature (sys call `sendfile()` mapped by Java `FileChannel#transferTo`) to transfer data directly from read buffer to socket buffer in kernel space. Using encryption aka SSL prevents zero-copy (`SSL_sendfile` [is currently not supported](https://kafka.apache.org/documentation/#maximizingefficiency)).

- Transfer with SSL (without zero-copy)

Disk → Kernel buffer → User space buffer (SSL) → Kernel network buffer → Network

- Zero copy transfer

Disk → Kernel buffer → Network

#### PageCache

PageCache is a kernel cache for disk `pages' implemented with paging memory management. The operating system caches pages in unused portions of RAM transparently to applications. This is why it is so important to have lots of free memory on a broker's machine.




---------------------------
# TO CONTINUE

replication


Apache Kafka is a commit-log system. The records are appended at the end of each Partition, and each Partition is also split into segments. Segments help delete older records through Compaction, improve performance, and much more.

Apache Kafka behaves as a commit-log when it comes to dealing with storing records. 

Records are appended at the end of each log one after the other and each log is also split in segments.
Segments help with deletion of older records, improving performance, and much more. 

So, the log is a logical sequence of records that’s composed of segments (files) and segments store a sub-sequence of records.

Broker configuration allows you to tweak parameters related to logs. You can use configuration to control the rolling of segments, record retention, and so on.

Not everyone is aware of how these parameters have an impact on broker behavior. 

For instance, they determine how long records are stored and made available to consumers. 



Kafka is typically referred to as a Distributed, Replicated Messaging Queue, which although technically true, usually leads to some confusion depending on your definition of a messaging queue. Instead, I prefer to call it a Distributed, Replicated Commit Log. This, I think, clearly represents what Kafka does, as all of us understand how logs are written to disk. And in this case, it is the messages pushed into Kafka that are stored to disk.







Userfuyl link:
- https://www.confluent.io/blog/apache-kafka-architecture-and-internals-by-jun-rao/ 
- https://developer.confluent.io/courses/architecture/broker/
- https://andriymz.github.io/kafka/kafka-disk-write-performance/#zero-copy-data-transfer
- https://kafka.apache.org/documentation/#messageformat



NOTE: Compaction and Immutability
NOTE: Record Batch and producer
NOTE: The importance of lingering

