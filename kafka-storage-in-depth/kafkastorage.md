# TODO............



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



## min.cleanable.dirty.ratio
  - https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html
  - https://medium.com/apache-kafka-from-zero-to-hero/apache-kafka-guide-23-log-cleanup-compact-3f62751e4acb
  - [ADVANCE] https://zendesk.engineering/an-investigation-into-kafka-log-compaction-5e520f4291f0




## Replication


Apache Kafka is a commit-log system. The records are appended at the end of each Partition, and each Partition is also split into segments. Segments help delete older records through Compaction, improve performance, and much more.

Apache Kafka behaves as a commit-log when it comes to dealing with storing records. 

Records are appended at the end of each log one after the other and each log is also split in segments.
Segments help with deletion of older records, improving performance, and much more. 

So, the log is a logical sequence of records that’s composed of segments (files) and segments store a sub-sequence of records.

Broker configuration allows you to tweak parameters related to logs. You can use configuration to control the rolling of segments, record retention, and so on.

Not everyone is aware of how these parameters have an impact on broker behavior. 

For instance, they determine how long records are stored and made available to consumers. 



Kafka is typically referred to as a Distributed, Replicated Messaging Queue, which although technically true, usually leads to some confusion depending on your definition of a messaging queue. Instead, I prefer to call it a Distributed, Replicated Commit Log. This, I think, clearly represents what Kafka does, as all of us understand how logs are written to disk. And in this case, it is the messages pushed into Kafka that are stored to disk.
