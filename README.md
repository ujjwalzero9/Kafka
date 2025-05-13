# Apache Kafka Interview Preparation Notes (SDE-1, Backend)

## 1. Summary and Overview

### What is Kafka?

Apache Kafka is a distributed event streaming platform used to build real-time data pipelines and streaming applications. It enables publishing, storing, and processing streams of records in a fault-tolerant way.

### How Kafka Works

* **Producer:** Sends data (messages) to Kafka topics.
* **Topic:** A category/feed to which records are sent. Topics are split into **partitions**.
* **Partition:** An ordered, immutable sequence of messages. Each message has a unique **offset**.
* **Consumer:** Reads messages from partitions.
* **Broker:** A Kafka server storing data and handling requests.
* **Consumer Group:** A group of consumers sharing the load for a topic.
* **Offset:** Position of a message in a partition.
* **Replication:** Each partition can be copied to multiple brokers.
* **Retention:** Kafka retains messages for a set duration, regardless of whether they're read.

### Core Concepts

* Kafka ensures **at-least-once** delivery by default.
* Message **ordering** is guaranteed only within a partition.
* **Consumer offset** management is key to processing guarantees.

### Real-World Use Cases

* Log aggregation
* Website activity tracking
* Event sourcing
* Data pipeline staging
* Metrics collection

### Architecture

```
[Producer] ---> [Kafka Broker Cluster (Topics/Partitions)] ---> [Consumer Group]
```

Each topic is divided into partitions which are distributed across brokers. Consumers in a group divide partitions among themselves.

---

## 2. Limitations and Challenges

* **Ordering only within a partition**
* **Duplicate messages** possible (requires idempotency)
* **Scaling limited by partition count**
* **Operational complexity** (e.g., broker failure handling, monitoring lags)
* **Retention-based data loss** if consumers fall too far behind
* **Latency spikes** under high throughput or load

---

## 3. Important Interview & Situational Questions

### Technical Questions

1. How do Kafka offsets work?
2. How does Kafka ensure message durability?
3. What’s the difference between Kafka and RabbitMQ?
4. How does Kafka provide fault tolerance?
5. What are Kafka consumer groups and why are they used?
6. How does replication work in Kafka?
7. What delivery guarantees does Kafka provide?
8. How is message ordering preserved in Kafka?
9. What’s the difference between at-least-once and exactly-once semantics in Kafka?
10. How does Kafka handle message retention?

### Situational Questions

1. A consumer is lagging—how would you debug and fix it?
2. How would you implement a retry mechanism for message processing failures?
3. What would you do if Kafka messages are arriving out of order?
4. How would you design a system that ensures idempotent processing of Kafka messages?
5. Kafka vs REST: when would you use which in a microservices setup?
6. How would you handle backpressure from downstream systems reading from Kafka?

### Trade-off Questions

1. Kafka vs Redis Streams
2. Kafka vs REST for service communication
3. Kafka vs traditional message queues

---

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
