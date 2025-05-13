# Apache Kafka

## Summary and Overview

**Apache Kafka** is an open-source distributed event streaming platform for building high-throughput, real-time data pipelines and applications. Producers (data publishers) send **messages** to named **topics**. Topics are partitioned logs, where each **partition** is an ordered, append-only sequence of messages. Every message in a partition is assigned a unique **offset** (0,1,2,…). Kafka runs on a **cluster** of broker nodes: each broker stores partitions of topics. Brokers receive data from producers, assign offsets, and write the data to disk. Consumers read messages from topics; they pull data from partitions and track their read position by offset.

* **Topic:** a feed/category of messages. Internally, a topic’s data is split into **partitions** (logs).
* **Partition:** an immutable, ordered log of messages. Each message has a sequential **offset** (its position in the log).
* **Offset:** a unique integer id for a message within its partition. Consumers use offsets to know which messages they have processed.
* **Producer:** an application that publishes (writes) messages to one or more topics. Producers can specify a partition (e.g. via a key) or rely on round-robin/partitioner.
* **Broker:** a Kafka server in the cluster. Brokers store topic partitions and handle all reads/writes. Brokers replicate partitions across the cluster for fault tolerance.
* **Consumer:** an application that subscribes to topics and reads messages. Consumers fetch messages in offset order from partitions, and commit their offsets to remember progress.
* **Consumer Group:** a set of consumers sharing the same group id. Each partition’s messages are delivered to exactly one consumer instance in the group. This allows parallel consumption: if a topic has N partitions, up to N consumers in a group can read in parallel.
* **Replication:** each partition is replicated to multiple brokers (configurable **replication factor**). One replica is the *leader* (handling reads/writes) and others are *followers* that replicate the data. If a broker fails, a follower becomes leader to avoid data loss.
* **Retention:** Kafka retains all published messages for a configurable time (or size) even after consumption. After the retention period, old messages are deleted to free space.
* **Ordering:** Kafka only guarantees message order *within* a single partition. Across different partitions of a topic, ordering is not enforced, so consumers may see messages out of the original global order.
* **Delivery semantics:** By default Kafka provides *at-least-once* delivery – every message will be delivered at least once (duplicates are possible on retries). Kafka can achieve *exactly-once* semantics using idempotent producers or transactions.

&#x20;*Figure: High-level Kafka architecture (producers write to a Kafka cluster’s topics/partitions; consumers read from them). Each broker stores replicated partitions across the cluster.*

Kafka’s **architecture** is based on a distributed commit log. Producers write records to a topic’s partitions, which the Kafka cluster (brokers) persists on disk. Each message append is extremely fast due to sequential disk writes. Consumers (often in consumer groups) pull messages at their own pace, using stored offsets to resume or replay. A controller broker (elected among the cluster) manages partition leadership and cluster metadata. Early versions used Apache ZooKeeper for coordination (leader election, config), though newer Kafka versions can run in “KRaft” mode without ZooKeeper.

Kafka is used for a variety of real-world use cases:

* **Website activity tracking:** Originally, Kafka was designed to publish real-time user activity streams (page views, clicks, etc.) to topics per event type. Downstream systems consume these feeds for analytics or storage.
* **Log aggregation and metrics:** Kafka often replaces traditional log collectors (Flume/Scribe) by treating application logs or metrics as streams of messages. This enables centralized processing with low latency and durable storage.
* **Stream processing pipelines:** It is common to build multi-stage processing where raw input is consumed and transformed into new topics. For example, processing news articles from an “articles” topic and producing cleaned data to another topic. Kafka Streams and other stream frameworks integrate tightly here.
* **Event sourcing:** Kafka can store every state change in an application as a time-ordered log of events. Since Kafka retains data for long periods, one can rebuild state by replaying the event log.
* **Commit log for distributed systems:** Kafka can act as an external distributed commit log, helping replicate data between nodes or serve as a re-synchronization mechanism.

## Limitations and Challenges

Kafka is powerful but has some trade-offs and operational considerations:

* **Partition-level ordering only:** Kafka only guarantees message order within each partition. If your topic has multiple partitions, consumers may see events in a different order than they were produced globally. Handling strict ordering across partitions requires additional design (e.g. sending all related events to the same partition).
* **Possible duplicate messages:** With *at-least-once* delivery, a broker or producer retry can cause the same message to be delivered more than once. Consumers must handle duplicates (idempotent processing or deduplication) unless exactly-once semantics are enabled.
* **Consumer scaling tied to partitions:** The maximum parallelism in a consumer group is bounded by the number of partitions. You cannot have more active consumers in a group than partitions (extras will idle). Thus planning partition count upfront is important for scaling.
* **Operational complexity:** Running Kafka requires managing a distributed cluster (brokers, controllers, ZooKeeper/KRaft, etc.). Tuning configurations (memory, I/O, topics, retention, etc.) and monitoring for lags or under-replicated partitions adds operational overhead. In large clusters, handling outages (broker failures and rebalances) can be complex.
* **Latency under heavy load:** Kafka is designed for low-latency, but high network latency or backpressure can increase delays. In extreme cases (high throughput, slow consumers), 99th-percentile end-to-end latencies can rise (engineers report p99 \~1 second under load). If producers vastly outpace consumers, consumer lags can grow, causing memory pressure or stale processing.
* **Broker failure impact:** Although replication provides fault-tolerance, a broker failure requires re-electing leaders and reassigning partitions, which briefly stalls those partitions. During this time consumption can pause or lag. Careful replication factor and monitoring are needed to minimize disruption.
* **Potential data loss scenarios:** While Kafka provides high durability, it is not immune to data loss. Misconfigurations (e.g. insufficient replication, premature retention settings) or multiple simultaneous failures can lead to lost messages. System designers must plan for worst-case scenarios (e.g. use proper `acks=all` settings, adequate replication, and monitor under-replicated partitions).

## Important Interview Questions

* **How do Kafka offsets work?** Each partition has its own sequence of offsets for messages. Consumers track an offset per partition to know which message to read next. Offsets are stored (by default) in Kafka’s internal `__consumer_offsets` topic or can be managed externally. If a consumer fails, it can resume from the last committed offset. For example, Kafka brokers assign sequential offsets to records as they arrive, and consumers periodically commit their current offsets.

* **How does Kafka ensure message durability?** Kafka writes all messages to disk in a commit log and replicates each partition to multiple brokers. When a producer writes with acks=all, it waits for the leader and enough followers to acknowledge the write. Once a message is *committed* to the log, it will not be lost as long as one replica is alive. In practice, even if a broker crashes, another replica can immediately take over without data loss. (Kafka also offers a "log compaction" feature to retain the latest update per key if long-term storage of all messages is needed.)

* **Why use Kafka instead of a traditional broker like RabbitMQ?** Kafka is built for high-throughput, distributed streaming with persisted storage. Unlike RabbitMQ (which is a message queue/broker with push semantics), Kafka provides durable logs that can serve many consumers independently. Kafka excels when you need to buffer large streams of data, replay old messages, or broadcast to multiple independent consumers. RabbitMQ might be simpler for low-latency RPC or complex routing, but Kafka is preferred for scalable data pipelines and event streaming at large scale.

* **What happens if a consumer is lagging behind?** First, identify why it’s slow (e.g. heavy processing, GC pauses, slow consumer group rebalances). You can scale by adding more partitions (and consumers) or by increasing consumer throughput (e.g. batch processing, increase `max.poll.records`). Throttling producers or allocating more resources to the consumer can help. Monitor consumer lag using Kafka’s metrics. If lag is unavoidable, ensure retention is long enough; Kafka will keep unconsumed messages until retention expires. In extreme cases, you might use a faster lane (dedicated topic) for backlogged data or skip unneeded messages (by moving offset forward).

* **How would you design a retry mechanism for failed messages?** A common pattern is to not commit the offset if processing fails, so Kafka re-delivers that message. However, to avoid infinite retry loops, many designs use a separate *retry* or *dead-letter* topic. For example, on failure you can produce the message (with headers or count) to a “retry” topic with a delay, and only commit the original offset. After a certain number of retries, send it to a dead-letter queue for manual inspection. Alternatively, use Kafka Streams or consumer client configuration to control retries and backoff. The key is that Kafka lets you reset or skip offsets manually, giving flexibility for retry logic.

* **Kafka vs Redis Streams (trade-off):** Both Kafka and Redis Streams can serve as message brokers, but Kafka is designed for high durability and throughput with long-term storage, while Redis Streams are in-memory (with optional persistence) and simpler to set up. Redis may give lower latency and simpler single-server usage, but Kafka scales to much larger data volumes, has richer replication/ack semantics, and a bigger ecosystem. Use Redis Streams for lightweight streaming in smaller setups; use Kafka when you need a robust, replicated log with durable replay capabilities.

* **Kafka vs REST for event-driven communication:** REST (HTTP APIs) are synchronous request-response calls, whereas Kafka is an asynchronous, push-based event bus. Kafka allows decoupling producers and consumers (an event can be consumed by multiple systems at its own pace), supports high throughput, and can buffer messages if consumers are slow. REST is simpler for point-to-point queries but does not natively support scalable fan-out or durable queuing. In an event-driven architecture, Kafka is often chosen for its persistence, scalability, and multiple consumer support, while REST is used for direct service-to-service calls.

## Example Snippets

## 4. Python Pseudocode Examples

### Producing Messages

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

message = {"user": "alice", "action": "purchase"}
producer.send('my-topic', value=message)
producer.flush()  # Ensure delivery before closing
producer.close()
```

### Consuming Messages

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'my-topic',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',  # or 'latest'
    enable_auto_commit=False,
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    group_id='my-consumer-group'
)

for message in consumer:
    try:
        print(f"Received: {message.value}")
        # Process the message here
        consumer.commit()  # Commit offset after successful processing
    except Exception as e:
        print("Processing failed, message will be retried.", e)
```

### Retry Logic Pseudocode (Simple)

```python
def process_message(msg):
    for attempt in range(3):
        try:
            # your_processing_logic(msg)
            return True
        except Exception as e:
            print(f"Attempt {attempt} failed")
    # Send to dead-letter topic after 3 failures
    producer.send('dead-letter-topic', value=msg)
    return False
```

---

This structured guide gives you both theoretical understanding and practical readiness for Kafka-related interviews.
