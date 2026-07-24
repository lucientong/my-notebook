# Message Queues - Asynchronous Decoupling in Backend Systems

Language: English | [中文](../后端知识库/08-消息队列.md)

---

## Table of Contents

### Fundamentals and Components
1. [Message Queue Fundamentals](#1-message-queue-fundamentals)
2. [Kafka In Depth](#2-kafka-in-depth)
3. [RabbitMQ In Depth](#3-rabbitmq-in-depth)

### Reliability and Selection
4. [Message Reliability](#4-message-reliability)
5. [Choosing a Message Queue](#5-choosing-a-message-queue)

### Practice and Interview Review
6. [Practical Cases](#6-practical-cases)
7. [Interview Self-Check](#7-interview-self-check)

---

## 1. Message Queue Fundamentals

### 1.1 Why Do We Need Message Queues?

**Core purposes**:

| Purpose | Explanation | Example |
|---------|-------------|---------|
| **Decoupling** | Producers and consumers evolve independently | Order service -> MQ -> Inventory service |
| **Asynchrony** | Non-critical work is processed asynchronously | User signup -> MQ -> Send email |
| **Traffic smoothing** | Buffer traffic spikes | Flash sale -> MQ -> Process slowly |
| **Broadcasting** | One event can be consumed by multiple downstream services | Order created -> multiple subscribers |

**Synchronous vs asynchronous processing**:

```text
Synchronous:
User signup -> Write DB (50ms) -> Send email (200ms) -> Send SMS (150ms) = 400ms

Asynchronous with MQ:
User signup -> Write DB (50ms) -> Send MQ message (5ms) = 55ms
                                      |
                                      v
                         Background processing: email and SMS
```

The main value is not "making everything asynchronous"; it is controlling coupling, latency, burst traffic, and downstream failure isolation.

### 1.2 Core Concepts

| Concept | Meaning |
|---------|---------|
| **Producer** | Sends messages |
| **Consumer** | Receives and processes messages |
| **Broker** | Message server that stores and routes messages |
| **Topic** | Logical message stream or category |
| **Partition** | Physical shard of a Kafka topic |
| **Consumer Group** | A group of consumers sharing work |
| **Offset** | Consumer position in a partition |

### 1.3 Messaging Models

**Point-to-point queue**:

```text
[Producer] -> [Queue] -> [Consumer]
```

- A message is usually consumed by one consumer instance.
- After successful consumption, the message is acknowledged or removed.

**Publish-subscribe topic**:

```text
[Producer] -> [Topic] -> [Consumer A]
                    -> [Consumer B]
                    -> [Consumer C]
```

- One message can be consumed by multiple subscriber groups.
- Messages may be retained for a configurable period, enabling replay.

---

## 2. Kafka In Depth

### 2.1 Architecture

```text
[Producer] -> [Broker 1] -> [Consumer Group A]
           -> [Broker 2] -> [Consumer Group B]
           -> [Broker 3]

ZooKeeper or KRaft: metadata management and controller election
```

**Core components**:

- **Producer**: writes records to topics.
- **Broker**: Kafka server; multiple brokers form a cluster.
- **Topic**: logical event stream.
- **Partition**: physical append-only log for a topic.
- **Consumer Group**: consumers that jointly consume partitions.

Modern Kafka can run in KRaft mode, which removes the dependency on ZooKeeper.

### 2.2 Topic and Partition

```text
Topic: orders
├─ Partition 0: [msg0, msg3, msg6, ...]
├─ Partition 1: [msg1, msg4, msg7, ...]
└─ Partition 2: [msg2, msg5, msg8, ...]
```

**Partitioning strategies**:

1. **Round robin**: distribute messages evenly.
2. **Key hash**: messages with the same key go to the same partition, preserving per-key order.
3. **Custom partitioner**: choose a partition explicitly.

**Replication model**:

```text
Partition 0
├─ Leader (Broker 1)    <- reads and writes
├─ Follower (Broker 2)  <- replication
└─ Follower (Broker 3)  <- replication
```

A producer writes to the leader. Followers replicate data from the leader. Kafka uses ISR, or In-Sync Replicas, to track replicas that are sufficiently caught up.

### 2.3 Producer

**Go example**:

```go
import "github.com/segmentio/kafka-go"

func main() {
    writer := kafka.NewWriter(kafka.WriterConfig{
        Brokers:  []string{"localhost:9092"},
        Topic:    "orders",
        Balancer: &kafka.Hash{}, // partition by key
    })
    defer writer.Close()

    err := writer.WriteMessages(context.Background(),
        kafka.Message{
            Key:   []byte("order_1001"),
            Value: []byte(`{"id":1001,"amount":100}`),
        },
    )
    if err != nil {
        log.Fatal(err)
    }
}
```

**Java example**:

```java
import org.apache.kafka.clients.producer.*;

Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all");  // wait for all ISR replicas

Producer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>(
    "orders",
    "order_1001",
    "{\"id\":1001,\"amount\":100}"
);

producer.send(record).get(); // synchronous send

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        System.out.println("Sent successfully: " + metadata.offset());
    } else {
        exception.printStackTrace();
    }
});

producer.close();
```

### 2.4 Consumer

**Go example**:

```go
func main() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        Topic:   "orders",
        GroupID: "order-service",
        MinBytes: 10e3,
        MaxBytes: 10e6,
    })
    defer reader.Close()

    for {
        msg, err := reader.ReadMessage(context.Background())
        if err != nil {
            log.Fatal(err)
        }

        fmt.Printf("Partition: %d, Offset: %d, Key: %s, Value: %s\n",
            msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))

        processOrder(msg.Value)
    }
}
```

**Java example**:

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-service");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("enable.auto.commit", "false");  // commit offset manually

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("Partition: %d, Offset: %d, Key: %s, Value: %s\n",
            record.partition(), record.offset(), record.key(), record.value());

        processOrder(record.value());
    }

    consumer.commitSync();
}
```

### 2.5 Consumer Group

```text
Topic: orders (3 partitions)

Consumer Group A:
├─ Consumer 1 -> Partition 0, 1
└─ Consumer 2 -> Partition 2

Consumer Group B:
├─ Consumer 1 -> Partition 0
├─ Consumer 2 -> Partition 1
└─ Consumer 3 -> Partition 2
```

Within one consumer group, each partition is assigned to at most one consumer at a time. Different consumer groups can independently consume the same topic.

### 2.6 Offset Management

**Auto commit**:

```java
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "1000");
```

Auto commit is simple but dangerous: if the offset is committed before business processing succeeds, the message may be lost from the consumer's perspective.

**Manual synchronous commit**:

```java
consumer.commitSync();  // block until commit succeeds
```

**Manual asynchronous commit**:

```java
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Offset commit failed", exception);
    }
});
```

**Seeking to a specific offset**:

```java
consumer.seekToBeginning(partitions);
consumer.seekToEnd(partitions);
consumer.seek(partition, offset);
```

---

## 3. RabbitMQ In Depth

### 3.1 Architecture

```text
[Producer] -> [Exchange] -> [Queue] -> [Consumer]
```

**Core components**:

- **Producer**: publishes messages.
- **Exchange**: routes messages.
- **Queue**: stores messages.
- **Binding**: connects an exchange to a queue.
- **Consumer**: receives and acknowledges messages.

### 3.2 Exchange Types

**Direct exchange**:

```text
Exchange (direct)
├─ routing_key: "order.create" -> Queue A
└─ routing_key: "order.pay"    -> Queue B
```

**Fanout exchange**:

```text
Exchange (fanout)
├─ Queue A
├─ Queue B
└─ Queue C
```

**Topic exchange**:

```text
Exchange (topic)
├─ routing_key: "order.*"      -> Queue A  (order.create, order.pay)
├─ routing_key: "order.create" -> Queue B
└─ routing_key: "#"            -> Queue C  (all messages)
```

**Headers exchange** routes messages based on header attributes.

### 3.3 Producer

```go
import "github.com/streadway/amqp"

func main() {
    conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
    defer conn.Close()

    ch, _ := conn.Channel()
    defer ch.Close()

    ch.ExchangeDeclare(
        "orders", // name
        "direct", // type
        true,     // durable
        false,    // auto-deleted
        false,    // internal
        false,    // no-wait
        nil,      // arguments
    )

    ch.Publish(
        "orders",       // exchange
        "order.create", // routing key
        false,          // mandatory
        false,          // immediate
        amqp.Publishing{
            ContentType: "application/json",
            Body:        []byte(`{"id":1001}`),
        },
    )
}
```

### 3.4 Consumer

```go
func main() {
    conn, _ := amqp.Dial("amqp://guest:guest@localhost:5672/")
    defer conn.Close()

    ch, _ := conn.Channel()
    defer ch.Close()

    q, _ := ch.QueueDeclare(
        "order_queue", // name
        true,          // durable
        false,         // delete when unused
        false,         // exclusive
        false,         // no-wait
        nil,           // arguments
    )

    ch.QueueBind(
        q.Name,         // queue name
        "order.create", // routing key
        "orders",       // exchange
        false,
        nil,
    )

    msgs, _ := ch.Consume(
        q.Name, // queue
        "",     // consumer
        false,  // auto-ack
        false,  // exclusive
        false,  // no-local
        false,  // no-wait
        nil,    // args
    )

    for msg := range msgs {
        fmt.Println("Received:", string(msg.Body))
        processOrder(msg.Body)
        msg.Ack(false)
    }
}
```

### 3.5 Message Acknowledgement

**Producer confirm mode**:

```go
ch.Confirm(false)

ch.Publish(...)

select {
case confirm := <-ch.NotifyPublish(make(chan amqp.Confirmation, 1)):
    if confirm.Ack {
        fmt.Println("Message confirmed")
    } else {
        fmt.Println("Message not confirmed")
    }
}
```

**Consumer acknowledgement**:

```go
msg.Ack(false)        // acknowledge
msg.Nack(false, true) // reject and requeue
msg.Reject(false)     // reject without requeue
```

---

## 4. Message Reliability

### 4.1 Producer Reliability

**Problem**: the message may fail to be sent after the business operation succeeds.

**Solution 1: synchronous send with retry**:

```go
func sendWithRetry(producer *kafka.Writer, msg kafka.Message, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := producer.WriteMessages(context.Background(), msg)
        if err == nil {
            return nil
        }
        time.Sleep(time.Second * time.Duration(i+1))
    }
    return errors.New("send failed")
}
```

**Solution 2: transactional outbox / local message table**:

```sql
CREATE TABLE local_messages (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    topic VARCHAR(100),
    message TEXT,
    status VARCHAR(20),  -- 'pending', 'sent', 'failed'
    retry_count INT DEFAULT 0,
    created_at TIMESTAMP
);
```

```go
// 1. Business write and local message in the same database transaction.
tx.Exec("INSERT INTO orders ...")
tx.Exec("INSERT INTO local_messages ...")
tx.Commit()

// 2. Periodically scan and send pending messages.
func sendPendingMessages() {
    rows := db.Query("SELECT * FROM local_messages WHERE status = 'pending'")
    for rows.Next() {
        err := kafka.WriteMessages(...)
        if err == nil {
            db.Exec("UPDATE local_messages SET status = 'sent' WHERE id = ?", id)
        }
    }
}
```

The outbox pattern turns a distributed transaction into a local transaction plus eventual consistency.

### 4.2 Broker Reliability

**Kafka**:

1. Set `replication.factor=3`.
2. Use ISR to track replicas that are in sync with the leader.
3. Configure `acks` carefully:
   - `acks=0`: fastest, but messages may be lost.
   - `acks=1`: leader acknowledgement only.
   - `acks=all`: all ISR replicas acknowledge, most reliable.

**RabbitMQ**:

1. Make exchanges, queues, and messages durable.
2. Use quorum queues or mirrored queues for replication.
3. Use publisher confirms for producer-side acknowledgement.

### 4.3 Consumer Reliability

**Problems**: processing failure, duplicate consumption, and incorrect acknowledgement timing.

**Manual offset commit in Kafka**:

```go
processMessage(msg)
reader.CommitMessages(ctx, msg)
```

**Idempotent processing**:

```go
func processOrder(orderID string) {
    if db.Exists("processed:" + orderID) {
        return
    }

    doBusinessLogic()
    db.Set("processed:" + orderID, 1)
}
```

**Dead-letter queue**:

```go
if err := processMessage(msg); err != nil {
    sendToDLQ(msg)
    msg.Ack(false)
}
```

In production, the common practical guarantee is `at-least-once delivery + consumer-side idempotency`.

### 4.4 Message Ordering

**Kafka solution**:

1. Send events for the same business entity to the same partition by key.
2. Keep one consumer processing each partition.

```go
writer.WriteMessages(context.Background(),
    kafka.Message{
        Key:   []byte(orderID),
        Value: []byte(data),
    },
)
```

**RabbitMQ solution**:

1. Use a single queue for strict serial processing.
2. Maintain ordering with business IDs when concurrency is required.

Global ordering usually sacrifices throughput. Per-entity ordering is usually the better engineering trade-off.

### 4.5 Kafka Rebalance ⭐⭐

Rebalance happens when partition ownership inside a consumer group changes. It is critical to understand because it can pause consumption and cause latency spikes.

**Triggers**:

```text
1. A consumer joins the group.
2. A consumer leaves, crashes, or times out.
3. The topic partition count changes.
4. The subscribed topic set changes.
```

**Eager rebalance flow**:

```text
1. All consumers stop consuming and revoke all partitions.
2. The Group Coordinator collects member metadata.
3. One consumer is elected as the group leader.
4. The leader computes partition assignment.
5. The coordinator sends assignments to all consumers.
6. Consumers resume consumption.

Problem: the whole consumer group stops during rebalance.
```

**Incremental rebalance with Cooperative Sticky (Kafka 2.4+)**:

```text
Improvement: partitions are not revoked all at once.
1. First round: revoke only partitions that need to move.
2. Second round: assign revoked partitions to new consumers.
Benefit: unaffected partitions can continue consuming.
```

**Reducing unnecessary rebalances**:

```bash
# Common issue: slow consumers are considered dead.
session.timeout.ms=45000
heartbeat.interval.ms=15000
max.poll.interval.ms=600000
max.poll.records=50
```

### 4.6 Delay Queue ⭐⭐

Delay queues allow messages to be consumed only after a specified delay. Common use cases include order timeout cancellation, scheduled jobs, and retry backoff.

**RabbitMQ implementation**:

```text
Approach: TTL + DLX

1. Send the message to a normal queue with TTL=30 minutes.
2. The message expires after 30 minutes.
3. The expired message is routed to a dead-letter exchange.
4. The DLX routes it to the real consumption queue.

Flow:
Producer -> [Normal Queue, TTL=30min] -> expired -> [DLX] -> [Consumption Queue] -> Consumer
```

**Kafka implementation options**:

```text
Kafka does not natively support delay queues.

Option 1: multi-level delay topics
- delay-5s, delay-30s, delay-5min, delay-30min
- A delay service forwards expired messages to the target topic.

Option 2: timing wheel
- Implement a timing wheel data structure.
- Push messages to Kafka when they expire.

Option 3: external component
- Store delayed messages in Redis ZSet with score=expiration_timestamp.
- Periodically scan expired messages and push them to Kafka.
```

### 4.7 Kafka Idempotence and Exactly-Once Semantics ⭐⭐

**Producer-side idempotence**:

```text
Problem: a producer retry caused by network jitter may create duplicate messages.

Enable:
enable.idempotence=true

Mechanism:
1. The producer gets a PID, or Producer ID.
2. Each message carries PID + Sequence Number.
3. The broker checks <PID, Partition, SeqNum>.
4. Duplicate records are acknowledged but not written again.

Limitations:
- Guarantees idempotence only within a single partition.
- Guarantees idempotence only within a producer session.
```

**Exactly-once semantics with transactions**:

```text
End-to-end exactly-once = idempotent producer + transactions + read_committed consumers

Transaction flow:
1. Producer begins a transaction.
2. Producer sends records to multiple topics or partitions.
3. Producer sends consumed offsets as part of the transaction.
4. Producer commits the transaction atomically.

Typical use case: Kafka Streams consume-transform-produce workflows.
Cost: roughly 20% performance overhead, depending on workload.
```

---

## 5. Choosing a Message Queue

### 5.1 Kafka vs RabbitMQ

| Dimension | Kafka | RabbitMQ |
|-----------|-------|----------|
| **Throughput** | Millions/sec | Tens of thousands/sec |
| **Latency** | Milliseconds | Microseconds to low milliseconds |
| **Persistence** | Disk, sequential append | Memory + disk |
| **Ordering** | Ordered within a partition | Ordered within a queue |
| **Routing** | Topic/partition based | Flexible exchange routing |
| **Best for** | Logs, event streams, big data, stream processing | Task queues, RPC-like workloads, complex routing |

### 5.2 Selection Guide

**Choose Kafka when**:

- You need very high throughput.
- You are building log collection, event streaming, monitoring, CDC, or stream processing.
- You need replay by offset.
- You can model ordering by partition key.

**Choose RabbitMQ when**:

- You need flexible routing with exchanges.
- You need task queue semantics, low latency, priority queues, or delay queues.
- You need explicit ack/nack semantics at the message level.

**Choose Redis Pub/Sub or Redis Streams when**:

- The scenario is lightweight.
- You already operate Redis and do not need complex MQ features.
- You can accept simpler durability and governance semantics.

---

## 6. Practical Cases

### Case 1: Asynchronous Order Processing

**Architecture**:

```text
[Order Service] -> Kafka(orders) -> [Inventory Service]
                                -> [Points Service]
                                -> [Notification Service]
```

**Order service producer**:

```go
func CreateOrder(order *Order) error {
    tx, _ := db.Begin()
    tx.Exec("INSERT INTO orders ...")
    tx.Exec("INSERT INTO local_messages (topic, message) VALUES ('orders', ?)", order.JSON())
    tx.Commit()

    go sendToKafka("orders", order)
    return nil
}
```

**Inventory service consumer**:

```go
func main() {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers: []string{"localhost:9092"},
        Topic:   "orders",
        GroupID: "inventory-service",
    })

    for {
        msg, _ := reader.ReadMessage(context.Background())

        if processed(msg.Key) {
            continue
        }

        err := deductInventory(msg.Value)
        if err != nil {
            sendToDLQ(msg)
        }

        markProcessed(msg.Key)
    }
}
```

Key production points:

- Use an outbox to avoid losing messages after the order transaction commits.
- Make downstream consumers idempotent.
- Monitor lag, failure rate, retry count, and DLQ growth.
- Provide a controlled replay mechanism.

### Case 2: Traffic Smoothing for Flash Sales

**Architecture**:

```text
[Flash Sale Request] -> Redis pre-deduct stock -> MQ -> [Order Service processes gradually]
```

**Flash sale API**:

```go
func Seckill(userID, productID int64) error {
    if !limiter.Allow() {
        return ErrTooManyRequests
    }

    stock := redis.Decr(fmt.Sprintf("stock:%d", productID)).Val()
    if stock < 0 {
        redis.Incr(fmt.Sprintf("stock:%d", productID))
        return ErrOutOfStock
    }

    kafka.WriteMessages(context.Background(), kafka.Message{
        Key:   []byte(fmt.Sprintf("%d", userID)),
        Value: []byte(fmt.Sprintf(`{"user_id":%d,"product_id":%d}`, userID, productID)),
    })

    return nil
}
```

**Order service consumer**:

```go
func processOrder(msg kafka.Message) {
    order := parseMessage(msg.Value)
    db.Exec("INSERT INTO orders ...")
    db.Exec("UPDATE inventory SET stock = stock - 1 WHERE product_id = ? AND stock > 0", order.ProductID)
    sendNotification(order.UserID)
}
```

The key is to protect the core order service from a traffic spike while keeping compensation and reconciliation available.

---

## Summary

Message queues are mainly about:

1. **Fundamentals**: decoupling, asynchrony, traffic smoothing, and broadcasting.
2. **Kafka**: high throughput, partitions, consumer groups, replay, and stream processing.
3. **RabbitMQ**: flexible routing, task queue semantics, low latency, and explicit acknowledgements.
4. **Reliability**: producer outbox, broker replication, consumer idempotency, retries, DLQ, and replay.
5. **Selection**: Kafka for event streams and big data; RabbitMQ for task queues and routing-heavy workloads.

---

## 7. Interview Self-Check

### Message Governance Notes

- MQ is not a magic solution for asynchronous processing. It converts synchronous complexity into message semantics, retry semantics, and operational governance.
- A production-grade MQ design must define idempotency, retry budget, dead-letter handling, backlog response, replay, and observability.
- Without lag, failure rate, retry count, and DLQ monitoring, a message system can become a failure amplifier.

### Quick Questions

### Q1: What roles do Producer, Broker, and Consumer play?

**Answer:** The producer creates messages, the broker stores and routes messages, and the consumer receives messages and executes business logic. The question is simple, but the key point is responsibility separation: the producer should not depend on the consumer's runtime state.

### Q2: What is the difference between a Topic and a Queue?

**Answer:** A queue usually represents point-to-point semantics where one message is processed by one consumer instance or one consumer group. A topic usually represents publish-subscribe semantics where multiple subscriber groups can consume the same event stream independently.

### Q3: What is the difference between push and pull consumption?

**Answer:** Push means the broker actively sends messages to consumers, which is easy to use but harder for backpressure. Pull means consumers fetch messages at their own pace, which gives better flow control and is a better fit for Kafka's high-throughput log model.

### Q4: What does ACK mean? What happens if a consumer does not ACK?

**Answer:** ACK means the consumer confirms that a message has been processed successfully. If the message is not acknowledged, the broker usually keeps it uncommitted, redelivers it, or triggers retry logic. The timing of ACK determines the trade-off between message loss and duplicate processing.

### Q5: What are at-most-once, at-least-once, and exactly-once semantics?

**Answer:** At-most-once means messages may be lost but are not duplicated. At-least-once means messages are not easily lost but may be duplicated. Exactly-once means no loss and no duplicates under a defined scope, but it is expensive and complex. Most production systems use at-least-once plus idempotent consumers.

### Q6: When should a Kafka consumer commit its offset?

**Answer:** Usually after business processing succeeds. Committing before processing can turn a failure into message loss. Committing after processing may cause duplicate consumption after a crash, so idempotency is required.

### Q7: What is the difference between a retry queue and a DLQ?

**Answer:** A retry queue is for failures that may recover, usually with backoff. A DLQ is for messages that still fail after controlled retries and need investigation or manual handling. Mixing them makes retry policy and incident analysis unclear.

### Q8: Why can increasing Kafka partitions break ordering or key routing?

**Answer:** Many producers route by a formula like `hash(key) % partition_count`. When the partition count changes, the same key may map to a different partition, so the previous ordering boundary may be broken.

### Q9: Why can "just add MQ" increase system complexity?

**Answer:** Once MQ is introduced, the system must handle at-least-once delivery, idempotency, retries, DLQ, replay, backlog monitoring, and failure visibility. MQ decouples services only when the team can operate the message semantics it introduces.

### Q10: Why should retries have a budget?

**Answer:** Unlimited retries can amplify downstream failures into retry storms. Retries should have a maximum count, exponential backoff, jitter, retryable error classification, and integration with circuit breaking, rate limiting, and DLQ handling.

### Deep-Dive Questions

### Q11: What are the core purposes of a message queue? When should you introduce one?

**Answer:** The core purposes are decoupling, asynchronous processing, traffic smoothing, and broadcasting. Common scenarios include sending email after registration, notifying multiple downstream services after order creation, buffering flash-sale traffic, and collecting large-scale logs. You should introduce MQ when the asynchronous boundary and failure semantics are explicitly designed.

### Q12: What is the relationship between Kafka Topic and Partition? Why do we need partitions?

**Answer:** A topic is a logical stream, while a partition is the physical append-only log. Partitions improve throughput through parallel reads and writes, enable horizontal scaling, and preserve ordering within a partition. A partition can be selected by round robin, key hash, or a custom partitioner.

### Q13: How does Kafka Consumer Group work? What is the problem with rebalance?

**Answer:** Consumers in the same group share topic partitions, and each partition is assigned to only one consumer in that group. Rebalance happens when membership or partition metadata changes. Eager rebalance can stop the whole group temporarily, causing latency spikes. Cooperative Sticky assignment reduces this by moving only affected partitions.

### Q14: Why can Kafka reach very high throughput?

**Answer:** Kafka uses sequential disk writes, page cache, batching, zero-copy transfer, and partition-level parallelism. Sequential append makes disk IO efficient, batching reduces network round trips, `sendfile` avoids unnecessary user-space copies, and partitions allow producers and consumers to scale horizontally.

### Q15: How do you prevent message loss across Producer, Broker, and Consumer?

**Answer:** On the producer side, use synchronous send with retry and an outbox table for transactional consistency. On the broker side, configure replication, `acks=all`, and sufficient `min.insync.replicas`. On the consumer side, process first and commit/ack after success, with idempotent handling and DLQ fallback.

### Q16: How do you handle duplicate consumption?

**Answer:** Duplicates cannot be fully avoided in distributed systems, so the consumer must be idempotent. Common methods include a unique message ID with Redis `SET NX` or a database unique index, business state machines, optimistic locking, and making idempotency records and business writes part of the same transaction.

### Q17: How do you guarantee ordering? What is the difference between global ordering and partition ordering?

**Answer:** Global ordering requires a single partition and a single consumer, which sacrifices concurrency. Partition ordering maps the same business key, such as order ID, to the same partition. It preserves per-entity order while allowing different entities to be processed in parallel.

### Q18: How do Kafka and RabbitMQ differ? How would you choose between them?

**Answer:** Kafka is optimized for high-throughput event streams, replay, sequential disk writes, and partition-level scaling. RabbitMQ is optimized for flexible routing, task queues, explicit ack/nack, low latency, and message-level control. Choose Kafka for logs, CDC, analytics, and streaming; choose RabbitMQ for task queues, complex routing, and RPC-like asynchronous workflows.

### Q19: How would you implement a delay queue?

**Answer:** In RabbitMQ, use TTL plus DLX or a delayed-message plugin. In Kafka, use delay topics, a timing wheel, or an external store such as Redis ZSet where the score is the expiration timestamp. The key trade-off is between native simplicity, delay precision, operational cost, and replayability.

### Q20: How does Kafka implement exactly-once semantics? What is the cost?

**Answer:** Kafka combines idempotent producers, transactions, and `read_committed` consumers. A transaction can atomically write records to multiple partitions and commit consumed offsets. This is most useful for consume-transform-produce workflows such as Kafka Streams. The cost is transaction coordination overhead and lower throughput.

### Q21: What is the outbox pattern and what problem does it solve?

**Answer:** The outbox pattern solves the problem where a business transaction succeeds but message sending fails. The business write and outbox record are committed in the same local database transaction. A background dispatcher scans pending messages, sends them to MQ, and updates status. It provides eventual consistency without a distributed transaction.

### Q22: Why is Kafka not ideal as a traditional task queue?

**Answer:** Kafka is designed as a high-throughput log stream, not as a system where each task is picked and acknowledged individually. It lacks natural message-level ack/nack, priority, delayed redelivery, and fine-grained DLQ semantics. It is excellent for event streams, logs, CDC, and analytics, but complex task orchestration is usually a better fit for RabbitMQ or a workflow engine.

### Q23: Why is consumer rebalance often a production instability source?

**Answer:** Rebalance revokes and reassigns partitions, which may pause consumption, reset local state, and create lag spikes. Common triggers include frequent restarts, GC pauses, small `max.poll.interval.ms`, slow message handling, and aggressive autoscaling. Mitigation includes Cooperative Sticky assignment, stable consumers, controlled rolling deployment, asynchronous processing, and proper timeout tuning.

### Q24: How should a DLQ be designed?

**Answer:** A DLQ should define entry conditions, retry history, retention, ownership, alerting, and replay workflow. It should not receive every temporary failure. A good design separates retryable errors from poison messages and gives operators a controlled way to inspect, fix, and replay messages.

### Q25: If an order must eventually send a coupon after checkout succeeds, how would you design it?

**Answer:** This is an eventual consistency problem. I would use an outbox or transactional message pattern: after the order transaction commits, a pending coupon event is persisted, asynchronously delivered to MQ, and consumed idempotently by the coupon service. I would add retry, alerting, reconciliation, and controlled replay to guarantee eventual delivery without double issuing coupons.

### Q26: How do you design replay capability?

**Answer:** Replay requires stable business keys, event versions, searchable message metadata, idempotent consumers, and a controlled replay channel with throttling, auditing, and rollback strategy. Without replay, production recovery often degenerates into manual scripts and risky data patches.

### Q27: How should you choose a Kafka partition key?

**Answer:** Choose a key by balancing ordering and load distribution. If events for the same entity must be ordered, use the entity ID. If strict ordering is not required, prefer a high-cardinality and evenly distributed key. Also consider future partition expansion, because changing partition count may change routing results.

### Q28: How do you ensure "business success means consumption success"?

**Answer:** Process the business transaction first, then acknowledge the message or commit the offset. Kafka typically commits offsets after successful processing; RabbitMQ ACKs after successful processing. Idempotency is still required because crashes between business success and acknowledgement can cause duplicates.

### Open-Ended Design Questions

### D1: Design an order message system that guarantees strict per-order ordering and no message loss.

**Reference approach:**

- Use Kafka with `OrderID` as the partition key.
- Preserve order within each partition and process each partition serially.
- Use producer `acks=all`, retries, `replication.factor>=3`, and `min.insync.replicas>=2`.
- Commit offsets manually after successful business processing.
- Make consumers idempotent with `OrderID + operation type` or a business state machine.
- Use DLQ after bounded retries, with SLA, owner, and replay workflow.
- Accept that different orders can be processed in parallel, while events for the same order remain ordered.

### D2: Kafka consumer lag keeps increasing. How would you troubleshoot it?

**Reference approach:**

- Confirm lag with `kafka-consumer-groups.sh --describe`.
- Check consumer processing time, thread blocking, external dependency latency, and frequent rebalances.
- Check broker disk IO, network bandwidth, partition leader distribution, and page cache pressure.
- Scale consumers up to the partition count, tune `max.poll.records`, and batch or async-process downstream writes.
- If needed, split traffic temporarily, degrade non-critical work, and prioritize business-critical topics.
