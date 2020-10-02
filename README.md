**Kafka Theory**

**Topic** – Stream of data

- Topics are split in partitions
- Must have X partitions
- Must have X replication factor
- Possible configurations:
  - Replication factor
  - Number of partitions
  - Message size
  - Compressions level
  - Log Cleanup Policy
  - Min Insync Replicas
- A kafka broker automatically creates a topic under the following circumstances:
  - When a producer starts writing messages to the topic
  - When a consumer starts reading messages from the topic
  - When any client requests metadata for the topic
- Consumers do not directly write to the **\_\_consumer\_offsets** topic, they instead **interact with a broker** that has been elected to manage that topic, which is the **Group Coordinator broker**

**Partitions:**

- Ordered data (if no partitions are added later).
- Have to be specified at creation time.
- Offset only have meaning for a specific partition.
- Order is guaranteed only within a partition (not across partition).
- In Kafka data is kept for a limited time but offsets keep growing (default is a week).
- Data in immutable on partitions (append only).
- Data is assigned randomly to partitions unless a key is provided.
- Each partition has one leader and multiple ISR (Replicas).
- Each partition can handle X amount of throughput.
- More partitions:
  - Better parallelism, throughput
  - More consumers in a group
  - A partition at least on each broker
  - More open files on Kafka
  - More elections to perform on Zookeeper
- Partition Rebalance occurs when a new consumer is added, removed or consumer dies or partitions increased.
- Message with no keys will be stored with round-robin strategy among partitions.
- To one partition only one thread is assigned.

**Offset:**

- The position on the number of records within a partition.
- The **high watermark** is the indicated offset of messages that are **fully replicated** , while the end-of-log offset might be larger if there are newly appended records to the leader partition which are not replicated yet.
- The high watermark is an advanced Kafka concept, and is advanced once all the ISR replicates the latest offsets. A consumer can only read up to the value of the High Watermark (which can be less than the highest offset, in the case of acks=1)

**Brokers:**

- Each broker is identified with IDs
- Each broker contains certain topic partitions.
- Connecting to a broker is the same as being connected to all cluster.
- At any time only ONE broker can be leader for a given partition
- Only that leader can receive and serve data for a partition
- What decides leaders and ISR is zookeeper
- Every broker is also a bootstrap broker
  - You only need to connect to one broker to be connected to entire cluster
  - Each broker knows about all brokers, topics and partitions (metadata)

**Replication factor:**

- Number of copies of all partitions.
- Contributes to resilience.
- Increase in RF increase pressure in the brokers.
- More replication more latency.
- More storage usage.
- **min.insync.replicas** only matters if **acks=all**
- single **in-sync replica** is still readable, but not writeable if the producer using **acks=all**

**Zookeeper:**

- Manages brokers (keeps list of them)
- Performs leader election for partitions
- Sends notifications to Kafka in case of change
- Has leaders and followers
- Dynamic topic configurations are maintained in Zookeeper.
- **ACLs are stored in Zookeeper node /kafka-acls/** by default.
- 2181 - client port,
- 2888 - peer port,
- 3888 - leader port
- tickTime = value in ms
- initLimit = **Timeout** of zookeeper is **initLimit x tickTime**
- SyncLimit = Followers Sync timeout of zookeeper is **SyncLimit x tickTime**

**Kafka Guarantees:**

- Messages are appended to a topic-partition in order they are sent
- Consumer read messages in order stored.
- With replication factor of N, producers and consumers can tolerate up to N-1 brokers being down.
- If number of partitions remains constant for a topic the same key will always go to the same partition.
- There is only one controller (broker) in a cluster at all times.
- If more than one similar config is specified, the **smaller unit size will take precedence**.
- Kafka transfers data with **zero copy** and **no transformation**. **Any transformation (including compression) is the responsibility of clients.**
- In Kafka small heap size is needed. The rest goes for page cache.

**Default Ports:**

- Zookeeper: **2181**
- Zookeeper Leader Port **2888**
- Zookeeper Election Port (Peer port) **3888**
- Broker: **9092**
- REST Proxy: **8082**
- Schema Registry: **8081**
- KSQL: **8088**

**Partitions and Segments**

- Topics are made of partitions
- Keys are necessary if you require strong ordering or grouping for messages that share the same key. If you require that messages with the same key are always seen in the correct order, attaching a key to messages will ensure messages with the same key always go to the same partition in a topic. Kafka guarantees order within a partition, but not across partitions in a topic, so alternatively not providing a key - which will result in round-robin distribution across partitions - will not maintain such order.
- Partitions are made of segment (files).

-----------------------------------------------------------------------

| Segment 0 | Segment 1 | Segment 2 |Segment 3 (Active)|

|----------------|----------------|----------------|------------------|

| Offset 0-999 |Offset 1000-1999|Offset 2000-2999| Offset 3000-? |

-----------------------------------------------------------------------

- Only on segment is active (data being written to)
- Two segment settings:
  - log.segment.bytes: max size of a single segment
  - log.segment.ms: time Kafka will wait before committing the segment if not full
- Segments come with two indexes (files):
  - An offset position index: allows Kafka where to read to find a message
  - A timestamp to offset index: allows Kafka to find messages with timestamp

-----------------------------------------------------------------------

| Segment 0 | Segment 1 | Segment 2 |Segment 3 (Active)|

|----------------|----------------|----------------|------------------|

| Offset 0-999 |Offset 1000-1999|Offset 2000-2999| Offset 3000-? |

-----------------------------------------------------------------------

| Pos. index 0 | Pos. index 1 | Pos. index 2 | Pos. index 3 |

|----------------|----------------|----------------|------------------|

| timestp idx 0 | timestp idx 1 | timestp idx 2 | timestp idx 3 |

-----------------------------------------------------------------------

- Therefore, Kafka knows where to find data in a **constant time.**
- Segments exist in ../data/topic/
  - -rw-r--r-- 1 root root 10M Jul 8 14:45 00000000000000000000.index
  - -rw-r--r-- 1 root root 182M Jul 8 14:47 00000000000000000000.log
  - -rw-r--r-- 1 root root 10M Jul 8 14:45 00000000000000000000.timeindex

- A smaller log.segment.bytes (size, default 1GB):
  - More segments per partition
  - Log compaction happens more often
  - Kafka has to keep more files opened.
- A smaller log.segment.ms (time, default 1 week):
  - Set the frequency for log compaction
- Log compaction happens every time a segment is committed.
  - Smaller / More segments means that log cleanup will happen more often.
  - The cleaner checks for work every 15 seconds (log.cleaner.backoff.ms).

**Log Cleanup Policies**

- Kafka cluster expire data according to a policy
- The concept is called log cleanup.

  - Policy 1: **log.cleanup.policy=delete** (default)
    - Deletes data based on age (1 week default)
    - Deletes based on max size of log(default is -1 infinite)
    - Log.retention.hours (default is 168 – one week)
    - Log.retention.bytes (default -1 infinite)

  - Policy 2: **log.cleanuo.policy=compact** (default for \_\_consumer\_offsets)
    - Log compaction ensures to contain at least last know value
    - Useful if we just require a snapshot instead of full history (such as database table)
    - Keep most recent value (update)

- Most current data will exist for any key:value
- Ordering is kept, only removes some messages, do not re-order them.
- The offset is immutable
- Delete records can still be seen by consumers for delete.retention.ms (default is 24h)

**Kafka Consumers/Producers**

**Producers:**

- Write data to topics
- Know to which broker and partition to write to.
- In case Broker fails, the producer will automatically recover.
- Tree send modes:
  - **acks=0** – Won&#39;t wait for acknowledgment (possible data loss)
  - **acks=1** – Will wait for leader acknowledgement (limited data loss)
  - **acks=all** – Leader + Replicas acknowledgment (no data loss)
- Producer can choose to send a **key** with the message
  - If no key ( **key=null** ) data is sent round robin
  - If key is sent all messages will always go to same partition
  - Key is sent if you need message ordering (key hashing)
- Stateless means processing of each message depends only on the message, so converting from JSON to Avro or filtering a stream are both stateless operations.
- a **producer.send()** does not block and returns a **Future\&lt;RecordMetadata\&gt; object**
- Sending a null value is called a **tombstone (****delete marker for that key)** will be deleted upon compaction.
- Producer idempotence helps prevent the network introduced duplicates.
- Some messages may require multiple retries. If there are more than 1 requests in flight, it may result in messages received out of order. Note an exception to this rule is if you enable the producer setting: **enable.idempotence=true** which takes care of the out of ordering case on its own.
- **linger.ms** forces the producer to **wait before sending messages** , hence increasing the chance of creating batches that can be heavily compressed.
- **KafkaProducer is Thread-safe**

**Consumers:**

- Read data from topic
- In case Broker fails, the consumer will automatically recover.
- Data is **read in order within each partition**
- Consumers read data in **consumer groups**
  - Each consumer within a group reads from exclusive partitions.
  - If you have more consumers than partitions, some consumer are inactive.
  - As many consumers and partitions.
- **assign()** can be used for manual assignment of a partition to a consumer, in which case **subscribe()** must not be used.
- **assign()** takes TopicPartition object as an argument
- On a consumer multiple topic can be passed as a list or regex pattern
- The consumer will crash if the offsets it&#39;s recovering from have been deleted from Kafka.
- **subscribe()** and **assign()** cannot be called by the same consumer, **subscribe() is used to leverage the consumer group** mechanism, while **assign() is used to manually control partition assignment and reads assignment**.
- Consumers can only consumer messages up to the high watermark.
- You cannot have more consumers per groups than the number of partitions.
- Calling **close()** on consumer immediately **triggers a partition rebalance** as the consumer will not be available anymore.
- **KafkaConsumer is not Thread-safe**

**Consumer Offsets:**

- Stores offsets for each consumer group in Kafka (not zookeeper)
- Offsets committed live in Kafka to topic **\_\_consumer\_offset**
- From should were start the read (commit)
- Consumers choose when to commit offsets
  - At most once
    - Offsets are committed as soon as the message is received
    - If goes wrong the message will be lost (it won&#39;t read again)
  - At least once
    - Offsets are committed after message is processed.
    - If goes wrong message will be read again.
    - Can result in duplicate processing.
  - Exactly once:
    - Only from **Kafka =\&gt; Kafka**

**Kafka CLI**

**Create topic**

kafka-topics --zookeeper zk-1:2181 --topic first-topic --create --partitions 3 --replication-factor 2

**List topics**

kafka-topics --zookeeper zk-1:2181 –list

**Describe topics**

kafka-topics --zookeeper zk-1:2181 --topic first-topic –describe

**Describe partitions whose leader is not available**

kafka-topics --zookeeper zk-1:2181 --describe --topic new-topic --unavailable-partitions

**Describe partitions under replicated**

kafka-topics --zookeeper zk-1:2181 --describe --topic new-topic --under-replicated-partitions

**Delete topics**

kafka-topics --zookeeper zk-1:2181 --topic second-topic --delete

**List advanced topic configurations:**

kafka-configs --zookeeper zk-1:2181 --entity-type topics --entity-name config-topic --describe

**Add advanced topic configurations:**

kafka-configs --zookeeper zk-1:2181 --entity-type topics --entity-name config-topic --alter --add-config min.insync.replicas=1

**Delete advanced topic configurations:**

kafka-configs --zookeeper zk-1:2181 --entity-type topics --entity-name config-topic --alter --delete-config min.insync.replicas

**Console producer**

kafka-console-producer --broker-list kafka-1:9092 --topic first-topic

**Console producer with properties**

kafka-console-producer --broker-list kafka-1:9092 --topic first-topic --producer-property acks=all

**Console consumer (only from now on)**

kafka-console-consumer --bootstrap-server kafka-1:9092 --topic first-topic

**Console consumer (from beginning)**

kafka-console-consumer --bootstrap-server kafka-1:9092 --topic first-topic --from-beginning

**Console consumer with group**

kafka-console-consumer --bootstrap-server kafka-2:9092 --topic home-topic --group app-team

- The numbers of consumers should ideally be the same as number of partitions.
- If there are more consumers than topics some will not consume.
- If one consumer crashes other will automatically assume the role.
- In this scenario they will be consumed in round-robin.
- Consuming with group can only do --from-beginning once.
- Consuming with group even if not --from-beginning will consume past non consumed messages.

**Kafka consumer groups**

kafka-consumer-groups --bootstrap-server kafka-3:9092 --list

- Lists the consumer groups
- For each console consumer that does not specify a consumer group it will generate a random number suffix.

kafka-consumer-groups --bootstrap-server kafka-3:9092 --describe --group app-team

- You can see the offsets, the consumer and the lag

kafka-consumer-groups --bootstrap-server kafka-3:9092 --group team2 --execute --reset-offsets --to-earliest --topic home-topic

- You reset the offset by different criteria

**Console consumer/Producer with Key**

- kafka-console-producer --broker-list kafka-1:9092 --topic new-topic --property parse.key=true --property key.separator=:
- kafka-console-consumer --bootstrap-server kafka-1:9092 --topic new-topic --property print.key=true --property key.separator=:

**Confluent Schema Registry**

**Schema Registry**

- Schema registry is a separate application that provides RESTful interface for storing and retrieving Avro schemas.
- Clients can interact with the schema registry using the HTTP or HTTPS interface.
- The Schema Registry stores all the schemas in the **\_schemas** Kafka topic.
- First local cache is checked for the message schema. In case of cache miss, schema is pulled from the schema registry.
- An exception will be thrown in the Schema Registry does not have the schema.
- The Confluent Schema Registry is your safeguard against incompatible schema changes and will be the component that ensures no breaking schema evolution will be possible.
- It does not do any data verification.
- Decrease the size of the payload.
- Schema Registry stores all schemas in a Kafka topic defined by kafkastore.config= **\_schemas** (default) which is a single partition topic with log compacted.
- The default response media type **application/vnd.schemaregistry.v1+json** , **application/vnd.schemaregistry+json** , **application/json** are used in response header.
- **HTTP and HTTPS** client protocol are supported for schema registry.
- Prefix to apply to metric names for the default JMX reporter **kafka.schema.registry**
- Default port for listener is **8081**
- Confluent support primitive types of null, Boolean, Integer, Long, Float, Double, String, byte[], and complex type of IndexedRecord.
- Sending data of other types to KafkaAvroSerializer will cause a SerializationException
- The Producer
  - sends Avro data do Kafka
  - sends schema to Schema registry
- The Consumer
  - Receives Avro content from Kafka
  - Gets schema from schema Registry

**Avro and Schema evolution**

The Schema Registry server can enforce certain compatibility rules when new schemas are registered in a subject. These are the compatibility types:

**Backward**

- Is backward compatible when a **new** schema can be used to read **old** data.
- If a new field is **added** and exists a **default** for that field in the new schema.
- If a field is **removed without default** is not forward compatible.

**Forward**

- Is forward compatible when an **old** schema can be used to read **new** data.
- If a **new field is added** (without default) in the new schema and it does not exist in the old schema.
- New fields will be ignored.

**Full**

- Is fully compatible when is both **backward** and **forward**.
- Where in the new schema are **added** or **removed** fields with **defaults**.

**Breaking**

- **Add** or **remove** fields from **enum**.
- **Change type** of a field.
- **Rename** a field.

**Avro Data Types**

**Primitives types:**

- **null** : no value
- **boolean** : a binary value
- **int** : 32-bit signed integer
- **long** : 64-bit signed integer
- **float** : single precision (32-bit) IEEE 754 floating-point number
- **double** : double precision (64-bit) IEEE 754 floating-point number
- **bytes** : sequence of 8-bit unsigned bytes
- **string** : Unicode character sequence

**Complex Types** :

- **enums** : for list of defined size. Can&#39;t be change while maintaining compatibility.
- **arrays** : for a list of undefined size.
- **maps** : for a list of a key: values.
- **unions** : Allow for different datatypes.
- **Fixed**
- **record**

**Records:**

- **name**
- **type =\&gt; [enum, arrays, fixed, record, union, map]**
- **namespace**
- **doc (optional)**
- **aliases (optional)**
- **fields**
  - **name**
  - **type**
  - **doc (optional)**
  - **default (optional)**
  - **order (optional)**
  - **aliases (optional)**

**Logical Types** :

- **decimals**
- **date**
- **time-millis**
- **timestamp-millis**

**Kafka Rest Proxy**

- **Protocol buffers** isn&#39;t a natively supported type for the Confluent REST Proxy, but you may use the binary format instead
- **HTTP or HTTPS**

**Kafka Connect**

**Kafka Connect**

- **Simplify and improve getting data in and out of Kafka**
- Kafka Connect Source (Source =\&gt; Kafka) – Producer API
- Kafka Connect Sink (Kafka =\&gt; Sink) – Consumer API
- Runs as a cluster
- Without relying on external libs
- Exists a lot of **connectors**
- Part of ETL Pipeline with easy, no code setup.
- Run as a daemon on Kafka
- connect-standalone [-daemon] connect-standalone.properties
- Internal Kafka Connect topics:
  - **connect-configs** stores configurations,
  - **connect-status** helps to elect leaders for connect,
  - **connect-offsets** store source offsets for source connectors
- **JDBC** connector allows **one task per table**.

**Kafka Streams**

**Kafka Streams is a library for building streaming applications, specifically applications that transform input Kafka topics into output Kafka topics**

**Kafka Streams**

- **Data processing and transformations library**
- Standard java Application
- No need for separate cluster
- Highly scalable, elastic and fault tolerant
- Exactly once
- One record at a time
- Works for applications of any size
- May create internal intermediary topic:
  - Repartitioning topics
  - Changelog topics
- Internal topics:
  - **Prefixed by application.id**

**Kafka Streams Concepts**

- **Stream:** a sequence of immutable data records, fully ordered.
- **Stream Processor:** Is a node (graph) that transforms incoming streams, record by record and create a new **stream.**
- **Source Processor:** special processor that takes data from a Kafka topic. It doesn&#39;t transform the data.
- **Sink Processor:** Does not have children, Stream data directly to Kafka.
- Every streams application implements and executes at least one _topology_. Topology (also called DAG, or directed acyclic graph, in other stream-processing frameworks) is a set of operations and transitions that every event moves through from input to output
- The Streams engine parallelizes execution of a topology by splitting it into tasks
- It uses **RocksDB** but can use any other key, value store.
  - Producer metrics
  - Consumer metrics
  - stream-metrics
  - stream-rocksdb-state-metrics
  - stream-rocksdb-window-metrics

**Kafka Streams Configurations:**

- bootstrap.servers
- auto.offset.reset.config
- application.id
  - Consumer group.id = application.id
  - Default client.id prefix
- Default.[key|value].serde (serialization and Deserialization of data)

**Kafka Streams and KTables**

**KStream:**

- All **INSERTS** ,
- Similar to log
- Infinite
- Unbounded data streams
- **Reading from a topic that&#39;s not compacted**
- **If data is partial information**

**KTable:**

- **UPSERTS on non-null values**
- **DELETES on null values**
- Similar to a table
- Parallel with log compacted topics.
- **Reading from a topic that&#39;s log-compacted (aggregations)**
- **If you need structure (like a table)**
- Shards the data between all running Kafka Streams instances.

**GlobalKTable:**

- Has a full copy of all data on each instance.
- Needs more memory.
- Can do a KStream-GlobalKtable join without the need for having same number of partitions.

**KGroupedTable:**

- KGroupTable is a result of a groupBy done on a KTable

**Windowed joins:**

- In case of KStream-KStream join, both need to be co-partitioned.
- This restriction is not applicable in case of join with GlobalKTable.
- The output of KStream-KTable join is another KStream.

**Stateless vs Stateful**

- **Stateless** :
  - Only depends on data-point you process.
  - the processing of a message is independent from the processing of other messages.
  - Examples are when you only need to transform one message at a time, or filter out messages based on some condition.
  - No need memory of the past.
  - 1 x 2 = 2

- **Stateful** :
  - Also depends on external information.
  - Is stateful whenever, for example, it needs to  **join** ,  **aggregate** , or  **window**  its input data.
  - First operation depends of the firsts or older.
  - Depends of what happened on the past.
  - Count all values

**Stateless vs Stateful Operators**

- Stateless:
  - Branch
  - filter
  - inverseFilter
  - flatMap
  - flatMapValues
  - foreach
  - groupByKey
  - groupBy
  - map
  - mapValues
  - Peek
  - Print
  - SelectKey
  - TableToStream
- Stateful:
  - Join
  - Count
  - Aggregate
  - Reduce
  - windowing

**Windowing**

Windowing lets you control how to _group records that have the same key_ for stateful operations such as aggregations or joins into so-called _windows_. Windows are tracked per record key.

| **Name** | **Behaviour** | **Description** |
| --- | --- | --- |
| Tumbling time window | Time-based | Fixed-size, non-overlapping, gap-less windowsFor e.g. if window-size=5min and advance-interval =5min then it looks like [0-5min] [5min-10min] [10min-15min] … |
| Hopping time window | Time-based | Time-basedif widow-size=5min and advance-interval=3min then it looks like [0-5min] [3min-8min] [6min-11min] … |
| Sliding time window | Time-based | Fixed-size, overlapping windows that work on differences between record timestampsOnly used in Joins |
| Session window | Session-based | Dynamically-sized, non-overlapping, data-driven windowsUsed to aggregate key based events into session |

**Kafka KSQL**

**KSQL is based on Kafka Streams** and allows you to express transformations in the SQL language that get automatically converted to a Kafka Streams program in the backend.

In databases  **change data capture**  (CDC) is a set of software design patterns used to determine (and track) the data that has changed so that action can be taken using the changed data. CDC is also an approach to data integration that is based on the identification, capture and delivery of the changes made to data sources. CDC is generally better as it is more direct, efficient, and correctly captures things like deletes that the JDBC connector has no way to capture. Basically, sync data from a source database to new system.

- KSQL is **not ANSI SQL compliant** , for now there are no defined standards on streaming SQL languages.
- KSQL queries streams continuously and **must be stopped explicitly****.**

[https://docs.confluent.io/current/ksql/docs/tutorials/index.html#ksql-tutorials](https://docs.confluent.io/current/ksql/docs/tutorials/index.html#ksql-tutorials)

- CREATE STREAM new\_topic\_stream (text varchar) WITH (kafka\_topic=&#39;new-topic&#39;, value\_format=&#39;DELIMITED&#39;);

**Data modelling:**

- **Static data** should be **modelled as a table**
- **Business Transactions** should be **modelled as a Stream**.

SHOW STREAMS

EXPLAIN \&lt;query\&gt; statements run against the KSQL

**Kafka Monitoring**

- expose metrics through JMX
- Important metrics:
  - Under replicated partitions
  - Request handlers
  - Request timing

**Kafka Operations**

- Rolling Restart of Brokers
- Updating configurations
- Rebalance partitions
- Increasing replication factor
- Adding Broker
- Replacing a Broker
- Removing a Broker
- Upgrading a cluster with zero downtime

**Kafka Security**

**Encryption:**

- Data between exchanged clients and brokers should be secret
- SSL

**Authentication**

- Identity confirmation
- Similar to login/pass
- SSL certificates
  - SASL
  - Kerberos
  - SCRAM

**Authorisation**

- What user can or can&#39;t do.
- ACL

**Kafka Cluster Management**

**Cluster Replication**

- Mirror Maker
- Active =\&gt; Passive
- Active =\&gt; Active
- Does not preserve offsets, just data.

**Performance**

**max.in.flight.requests.per.connection**

- This controls how many messages the producer will send to the server without receiving responses.
- Setting this high can increase memory usage while improving throughput but setting it too high can reduce throughput as batching becomes less efficient.
- Setting this to 1 **will guarantee that messages will be written to the broker in the order** in which they were sent, even when retries occur.

**linger.ms**

- linger.ms controls the amount of time to wait for additional messages before sending the current batch.
- KafkaProducer sends a batch of messages either when the current batch is full or when the linger.ms limit is reached.
- By default, the producer will send messages as soon as there is a sender thread available to send them, even if there&#39;s just one message in the batch.
- By setting linger.ms higher than 0, we instruct the producer to wait a few milliseconds to add additional messages to the batch before sending it to the brokers.
- This increases latency but also increases throughput (because we send more messages at once, there is less overhead per message).

**batch.size**

- Controls how many bytes of **data to collect before sending** messages to the Kafka broker.
- Set this as high as possible, without exceeding available memory.
- Enabling compression can also help make more compact batches and increase the throughput of your producer.

**enable.idempotence**

- **Prevents network induced duplicates**.

**max.pool.intervall.ms**

- To **give the processing thread more time** to process a message.

**unclean.leader.election.enable**

- To have availability over consistency.

**Session.timeout.ms**

- Specifies the amount of time within which the broker needs to get at least one heart beat signal from the consumer.
- Otherwise it will mark the consumer as dead (default value is 10000 ms = 10 seconds)

**Heartbeat.interval.ms**

- Specifies the frequency of sending heart beat signal by the consumer (default is 3000 ms = 3 seconds)