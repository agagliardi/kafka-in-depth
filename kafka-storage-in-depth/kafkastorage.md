## Log Segments:

Detail structure https://strimzi.io/blog/2021/12/17/kafka-segment-retention/

In Kafka's storage architecture, log segments represent a fundamental unit for storing data. Here's a detailed breakdown of how log segments work:

### Sequential Storage:

Log segments are sequentially appended files where Kafka stores its data on disk.
Each log segment contains a contiguous sequence of records, ordered by their offset, which represents the position of the record in the log.
### Immutability:

Once a log segment is written, it becomes immutable, meaning its contents cannot be modified.
Immutability ensures data integrity and simplifies replication, as replicas can be created by simply copying the segment without worrying about concurrent writes.
### Segment Rolling:

As data is continuously appended to a segment, it grows until it reaches a certain configurable size or time threshold.
When a segment reaches this threshold, it is closed for writing, and a new segment is opened to continue storing incoming records.
This process is known as segment rolling and ensures that segments remain manageable in size, facilitating efficient storage management and retrieval.
### Retention and Deletion:

Kafka supports configurable retention policies, allowing administrators to specify how long data should be retained in the log.
Once data exceeds the retention period, it becomes eligible for deletion during log cleanup processes.
Kafka's retention policies enable organizations to control storage costs and comply with data retention regulations by automatically removing old data.
### Segment Indexing:

Each log segment includes an index file that maps offsets to physical file positions, enabling efficient record retrieval based on offset.
The index allows Kafka to quickly locate records within a segment without having to read through the entire file sequentially.

#### extra: What about changing segment size?
"Impact of increasing/decreasing the segment size"
https://www.conduktor.io/blog/understanding-kafkas-internal-storage-and-log-retention/

## Write-ahead Log (WAL):

~ NVM SSD disks friendly, mechanical symphaty

Write-Ahead Log (WAL) is a crucial component of Kafka's storage mechanism, providing durability and fault tolerance. Here's a detailed look at how WAL is implemented:

### Durability Guarantee:

When a producer sends a message to Kafka, it first writes the message to a log segment on disk before acknowledging the write to the producer.
This ensures that the message is durably stored on disk before being considered successfully written, providing a strong durability guarantee.
### Sequential Write Operations:

Kafka employs sequential write operations to the log, appending new messages to the end of the log segment.
Sequential writes are more efficient than random access writes, especially for spinning disk drives, and help optimize disk I/O performance.

Write aplification, Wear leveling and Garbage Collection
https://en.wikipedia.org/wiki/Write_amplification


### Recovery Mechanism:

In the event of a broker failure or crash, Kafka can recover data using the WAL.
Upon restart, Kafka reads the WAL to reconstruct the state of the log and resumes normal operation from the last known consistent state.
### Transaction Support:

Kafka's WAL also supports transactional writes, allowing producers to atomically write multiple messages as part of a transaction.
Transactional support ensures that either all messages within a transaction are successfully written to the log, or none of them are, preserving data consistency.
### Integration with Log Segments:

WAL seamlessly integrates with Kafka's log segment mechanism, where each log segment corresponds to a portion of the WAL.
Once a log segment is filled, it is closed, and its contents are flushed to disk, ensuring that data written to the WAL is eventually persisted to stable storage.


## Retention Policies:

Kafka supports configurable retention policies for managing data retention.
Data can be retained based on time or size constraints, allowing organizations to control storage costs and meet compliance requirements.
Retention policies ensure that Kafka can handle high volumes of data without overwhelming the storage resources.

"Log retention - The records may persist longer than the retention time"
https://www.conduktor.io/blog/understanding-kafkas-internal-storage-and-log-retention/

"How long my records are retained? Longer than you expect!"
Detail structure https://strimzi.io/blog/2021/12/17/kafka-segment-retention/

### Log compaction
https://www.conduktor.io/kafka/kafka-topic-configuration-log-compaction/

"Removing log data with cleanup policies"
https://strimzi.io/blog/2021/06/08/broker-tuning/

Tombstone
https://medium.com/@damienthomlutz/deleting-records-in-kafka-aka-tombstones-651114655a16

~ min.cleanable.dirty.ratio
https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html
https://medium.com/apache-kafka-from-zero-to-hero/apache-kafka-guide-23-log-cleanup-compact-3f62751e4acb
[ADVANCE] https://zendesk.engineering/an-investigation-into-kafka-log-compaction-5e520f4291f0


## Partitioning and Replication

Topics in Kafka are partitioned, allowing for parallelism and scalability.
Each partition has its own log, enabling independent storage and retrieval of data.
Partitioning enables efficient distribution of data across brokers in a Kafka cluster.

Replication in Kafka ensures fault tolerance and high availability.
Each partition has multiple replicas distributed across brokers.
Replicas are kept in sync using leader-follower replication, ensuring data consistency and resilience against failures.


~ Co-locating messages with the same key within the same partition can ensure consistent behavior during log compaction. 


## Large Message

https://www.conduktor.io/kafka/how-to-send-large-messages-in-apache-kafka/

## Tired Storage (in dev, Kafka 3.6)

https://cwiki.apache.org/confluence/display/KAFKA/KIP-405%3A+Kafka+Tiered+Storage

https://developers.redhat.com/articles/2023/11/22/getting-started-tiered-storage-apache-kafka#set_up_a_kafka_cluster

https://github.com/strimzi/strimzi-kafka-operator/pull/9727


##  Kafka Static Quota

https://access.redhat.com/documentation/en-us/red_hat_amq_streams/2.6/html-single/release_notes_for_amq_streams_2.6_on_openshift/index#kafka_static_quota_plugin_configuration

https://github.com/fvaleri/strimzi-debugging/tree/main/sessions/006



