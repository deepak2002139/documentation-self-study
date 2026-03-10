# ☕ Kafka Consumer in Java — The Complete Guide

> Everything you need to know about building Kafka Consumers in Java — from basic concepts to advanced patterns, with full working sample code.

---

## 📌 Table of Contents

1. [What is a Kafka Consumer?](#1-what-is-a-kafka-consumer)
2. [How a Kafka Consumer Works — Step by Step](#2-how-a-kafka-consumer-works--step-by-step)
3. [Consumer Lifecycle — Visual Flow](#3-consumer-lifecycle--visual-flow)
4. [Project Setup (Maven)](#4-project-setup-maven)
5. [Basic Consumer — Full Working Code](#5-basic-consumer--full-working-code)
6. [What the Output Looks Like](#6-what-the-output-looks-like)
7. [Consumer Configuration — All Key Properties Explained](#7-consumer-configuration--all-key-properties-explained)
8. [Consumer with Manual Offset Commit](#8-consumer-with-manual-offset-commit)
9. [Consumer with JSON Deserialization](#9-consumer-with-json-deserialization)
10. [Consumer Groups — How They Work](#10-consumer-groups--how-they-work)
11. [Rebalance Listener — Detect Partition Changes](#11-rebalance-listener--detect-partition-changes)
12. [Consuming from Specific Partitions & Offsets](#12-consuming-from-specific-partitions--offsets)
13. [Multi-Threaded Consumer](#13-multi-threaded-consumer)
14. [Graceful Shutdown](#14-graceful-shutdown)
15. [Error Handling & Retry Patterns](#15-error-handling--retry-patterns)
16. [Dead Letter Queue (DLQ) Pattern](#16-dead-letter-queue-dlq-pattern)
17. [Exactly-Once Consumer with Transactions](#17-exactly-once-consumer-with-transactions)
18. [Spring Boot Kafka Consumer (Bonus)](#18-spring-boot-kafka-consumer-bonus)
19. [Consumer Lag — What It Is & How to Monitor](#19-consumer-lag--what-it-is--how-to-monitor)
20. [Common Errors & Troubleshooting](#20-common-errors--troubleshooting)
21. [Best Practices Checklist](#21-best-practices-checklist)
22. [Quick Reference Cheat Sheet](#22-quick-reference-cheat-sheet)
23. [Summary](#23-summary)

---

## 1. What is a Kafka Consumer?

A **Kafka Consumer** is an application that **reads (consumes) messages/events** from Kafka topics.

> Think of it like a **subscriber** who reads articles from a newspaper (topic) at their own pace and can even re-read old editions.

### Key Characteristics

| Feature               | Description                                                        |
| --------------------- | ------------------------------------------------------------------ |
| **Pull-based**        | Consumer **pulls** data from Kafka (Kafka doesn't push)            |
| **Offset tracking**   | Consumer remembers where it left off (via offsets)                 |
| **Replayable**        | Can re-read old messages by resetting offsets                      |
| **Grouped**           | Multiple consumers can form a **Consumer Group** for parallelism   |
| **Fault-tolerant**    | If a consumer dies, another one in the group takes over            |

---

## 2. How a Kafka Consumer Works — Step by Step

```
Step 1: CONFIGURE  → Set broker address, deserializers, group ID
Step 2: CREATE     → Instantiate KafkaConsumer object
Step 3: SUBSCRIBE  → Subscribe to one or more topics
Step 4: POLL       → Continuously poll for new messages in a loop
Step 5: PROCESS    → Process each message (your business logic)
Step 6: COMMIT     → Commit offset (auto or manual) to track progress
Step 7: CLOSE      → Gracefully shut down the consumer
```

### What happens during `poll()`?

```
Consumer calls poll()
       │
       ▼
┌──────────────────────────────────────────────┐
│  1. Connect to Kafka Broker                  │
│  2. Join Consumer Group (if first poll)      │
│  3. Get assigned partitions                  │
│  4. Fetch messages from assigned partitions  │
│  5. Return batch of ConsumerRecords          │
│  6. Auto-commit previous offsets (if enabled)│
└──────────────────────────────────────────────┘
       │
       ▼
Consumer processes records
```

---

## 3. Consumer Lifecycle — Visual Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     KAFKA CONSUMER LIFECYCLE                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐     ┌───────────┐     ┌──────────┐     ┌──────────┐ │
│  │ Configure│────▶│ Subscribe │────▶│  Poll()  │────▶│ Process  │ │
│  │ Consumer │     │ to Topics │     │ Messages │     │ Messages │ │
│  └──────────┘     └───────────┘     └──────────┘     └──────────┘ │
│                                          │                  │      │
│                                          │                  │      │
│                                          │    ┌──────────┐  │      │
│                                          │◀───│  Commit  │◀─┘      │
│                                          │    │  Offset  │         │
│                                          │    └──────────┘         │
│                                          │                         │
│                                     ┌──────────┐                   │
│                                     │  Close   │                   │
│                                     │ Consumer │                   │
│                                     └──────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

### What is an Offset?

```
Partition 0:
 ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
 │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │  ← offsets
 └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                          ▲                    ▲
                          │                    │
                   Last committed         Latest message
                   offset (3)             offset (7)

 Consumer will read offsets 4, 5, 6, 7 on next poll()
```

- **Offset** = a sequential number assigned to each message in a partition.
- **Committed offset** = the last offset the consumer has confirmed processing.
- **Consumer lag** = latest offset − committed offset.

---

## 4. Project Setup (Maven)

```xml
<!-- pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>kafka-consumer-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>17</java.version>
        <kafka.version>3.7.0</kafka.version>
        <jackson.version>2.17.0</jackson.version>
    </properties>

    <dependencies>
        <!-- Kafka Client -->
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>${kafka.version}</version>
        </dependency>

        <!-- JSON Processing (for JSON events) -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- SLF4J Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>2.0.12</version>
        </dependency>
    </dependencies>
</project>
```

### Gradle Alternative

```groovy
// build.gradle
dependencies {
    implementation 'org.apache.kafka:kafka-clients:3.7.0'
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
    implementation 'org.slf4j:slf4j-simple:2.0.12'
}
```

---

## 5. Basic Consumer — Full Working Code

```java
// ===== BasicConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.header.Header;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.Collections;
import java.util.Properties;

public class BasicConsumer {

    public static void main(String[] args) {

        // ═══════════════════════════════════════════════
        // STEP 1: CONFIGURE THE CONSUMER
        // ═══════════════════════════════════════════════
        Properties props = new Properties();

        // Where to find Kafka brokers
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // How to convert bytes back to key & value (deserialization)
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class.getName());

        // Consumer group ID — consumers with the same group ID
        // share the work of reading partitions
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-first-consumer-group");

        // Where to start reading if no previous offset exists:
        //   "earliest" = from the very beginning
        //   "latest"   = only new messages from now on
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // Automatically commit offsets every 5 seconds (default)
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "5000");

        // Max records returned per poll() call
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "100");

        // ═══════════════════════════════════════════════
        // STEP 2: CREATE THE CONSUMER
        // ═══════════════════════════════════════════════
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        DateTimeFormatter formatter = DateTimeFormatter
            .ofPattern("yyyy-MM-dd HH:mm:ss")
            .withZone(ZoneId.systemDefault());

        try {
            // ═══════════════════════════════════════════════
            // STEP 3: SUBSCRIBE TO TOPIC(S)
            // ═══════════════════════════════════════════════
            consumer.subscribe(Collections.singletonList("order-events"));
            // For multiple topics:
            // consumer.subscribe(Arrays.asList("order-events", "payment-events"));

            System.out.println("🎧 Consumer started — listening on topic 'order-events'...");
            System.out.println("   Group ID: my-first-consumer-group");
            System.out.println("   Waiting for messages...\n");

            // ═══════════════════════════════════════════════
            // STEP 4: POLL FOR MESSAGES (Infinite Loop)
            // ═══════════════════════════════════════════════
            while (true) {
                // poll() fetches messages from Kafka
                // Duration = how long to wait if no messages are available
                ConsumerRecords<String, String> records = consumer.poll(
                    Duration.ofMillis(1000)
                );

                // If no messages, the loop continues (non-blocking)
                if (records.isEmpty()) {
                    continue;
                }

                System.out.printf("📬 Received %d message(s) in this batch%n%n",
                    records.count());

                // ═══════════════════════════════════════════════
                // STEP 5: PROCESS EACH MESSAGE
                // ═══════════════════════════════════════════════
                for (ConsumerRecord<String, String> record : records) {

                    System.out.println("═══════════════════════════════════════════");
                    System.out.println("📨 MESSAGE DETAILS:");
                    System.out.println("───────────────────────────────────────────");
                    System.out.printf("  Topic       : %s%n", record.topic());
                    System.out.printf("  Partition   : %d%n", record.partition());
                    System.out.printf("  Offset      : %d%n", record.offset());
                    System.out.printf("  Timestamp   : %s%n",
                        formatter.format(Instant.ofEpochMilli(record.timestamp())));
                    System.out.printf("  Key         : %s%n", record.key());
                    System.out.printf("  Value       : %s%n", record.value());

                    // Print headers (metadata attached to the message)
                    if (record.headers().toArray().length > 0) {
                        System.out.println("  Headers     :");
                        for (Header header : record.headers()) {
                            System.out.printf("    %s = %s%n",
                                header.key(),
                                new String(header.value()));
                        }
                    }
                    System.out.println("═══════════════════════════════════════════\n");

                    // ── Your Business Logic Here ──
                    processMessage(record.key(), record.value());
                }
            }

            // STEP 6: Offsets are auto-committed (every 5 seconds)
            //         No manual code needed when auto-commit is ON.

        } catch (Exception e) {
            System.err.println("❌ Error in consumer: " + e.getMessage());
            e.printStackTrace();
        } finally {
            // ═══════════════════════════════════════════════
            // STEP 7: CLOSE THE CONSUMER (Graceful Shutdown)
            // ═══════════════════════════════════════════════
            consumer.close();
            System.out.println("👋 Consumer closed.");
        }
    }

    private static void processMessage(String key, String value) {
        System.out.printf("  🔄 Processing: key=%s → %s%n%n", key, value);
        // Add your business logic:
        // - Save to database
        // - Call another service
        // - Update cache
        // - Trigger notification
    }
}
```

---

## 6. What the Output Looks Like

When you run the consumer and a producer sends messages, you'll see:

```
🎧 Consumer started — listening on topic 'order-events'...
   Group ID: my-first-consumer-group
   Waiting for messages...

📬 Received 3 message(s) in this batch

═══════════════════════════════════════════
📨 MESSAGE DETAILS:
───────────────────────────────────────────
  Topic       : order-events
  Partition   : 0
  Offset      : 0
  Timestamp   : 2026-03-10 14:30:00
  Key         : order-a1b2c3d4
  Value       : {"eventType":"ORDER_PLACED","data":{"orderId":"order-a1b2c3d4","userId":"user-123","product":"Mechanical Keyboard","amount":129.99,"status":"PENDING"}}
  Headers     :
    event-type = ORDER_PLACED
    source = order-service
═══════════════════════════════════════════

  🔄 Processing: key=order-a1b2c3d4 → {"eventType":"ORDER_PLACED",...}

═══════════════════════════════════════════
📨 MESSAGE DETAILS:
───────────────────────────────────────────
  Topic       : order-events
  Partition   : 1
  Offset      : 0
  Timestamp   : 2026-03-10 14:30:01
  Key         : order-e5f6g7h8
  Value       : {"eventType":"ORDER_PLACED","data":{"orderId":"order-e5f6g7h8","userId":"user-456","product":"Gaming Mouse","amount":79.99,"status":"PENDING"}}
═══════════════════════════════════════════

  🔄 Processing: key=order-e5f6g7h8 → {"eventType":"ORDER_PLACED",...}

═══════════════════════════════════════════
📨 MESSAGE DETAILS:
───────────────────────────────────────────
  Topic       : order-events
  Partition   : 2
  Offset      : 0
  Timestamp   : 2026-03-10 14:30:02
  Key         : order-i9j0k1l2
  Value       : {"eventType":"ORDER_PLACED","data":{"orderId":"order-i9j0k1l2","userId":"user-789","product":"USB-C Hub","amount":49.99,"status":"PENDING"}}
═══════════════════════════════════════════

  🔄 Processing: key=order-i9j0k1l2 → {"eventType":"ORDER_PLACED",...}
```

---

## 7. Consumer Configuration — All Key Properties Explained

### Essential Properties

| Property                   | What It Does                                                          | Example Value                  |
| -------------------------- | --------------------------------------------------------------------- | ------------------------------ |
| `bootstrap.servers`        | Kafka broker address(es)                                              | `localhost:9092`               |
| `key.deserializer`         | Converts message key bytes → Java object                              | `StringDeserializer`           |
| `value.deserializer`       | Converts message value bytes → Java object                            | `StringDeserializer`           |
| `group.id`                 | Consumer group name (required for subscribe)                          | `my-consumer-group`            |
| `auto.offset.reset`        | Where to start if no committed offset exists                          | `earliest` / `latest`          |

### Offset Management

| Property                      | What It Does                                            | Default  |
| ----------------------------- | ------------------------------------------------------- | -------- |
| `enable.auto.commit`          | Auto-commit offsets periodically?                       | `true`   |
| `auto.commit.interval.ms`     | How often to auto-commit (ms)                           | `5000`   |

### Performance Tuning

| Property                    | What It Does                                              | Default     |
| --------------------------- | --------------------------------------------------------- | ----------- |
| `max.poll.records`           | Max records per poll() call                              | `500`       |
| `max.poll.interval.ms`      | Max time between poll() calls before considered dead      | `300000`    |
| `fetch.min.bytes`           | Minimum data to return per fetch (waits until available)  | `1`         |
| `fetch.max.wait.ms`         | Max time broker waits to fill `fetch.min.bytes`           | `500`       |
| `fetch.max.bytes`           | Max data per fetch request                                | `52428800`  |
| `max.partition.fetch.bytes` | Max data per partition per fetch                          | `1048576`   |

### Session & Heartbeat

| Property                   | What It Does                                               | Default  |
| -------------------------- | ---------------------------------------------------------- | -------- |
| `session.timeout.ms`       | Time before consumer considered dead (no heartbeat)        | `45000`  |
| `heartbeat.interval.ms`    | How often consumer sends heartbeat to broker               | `3000`   |

### `auto.offset.reset` Explained

```
Scenario: Consumer starts for the FIRST TIME (no committed offset)

auto.offset.reset = "earliest"
  → Read ALL existing messages from the very beginning
  → Use when: You don't want to miss any data

  Partition: [msg0] [msg1] [msg2] [msg3] [msg4]
                ▲
                └── Start reading here

auto.offset.reset = "latest"
  → Skip all existing messages, only read NEW ones
  → Use when: You only care about new incoming data

  Partition: [msg0] [msg1] [msg2] [msg3] [msg4]
                                            ▲
                                            └── Start reading here

auto.offset.reset = "none"
  → Throw an exception if no committed offset found
  → Use when: You want strict control
```

---

## 8. Consumer with Manual Offset Commit

When you need **more control** over when offsets are committed (e.g., commit only after successfully processing).

### Why Manual Commit?

```
AUTO-COMMIT PROBLEM:
  1. poll() returns messages
  2. Auto-commit happens (offset = 5)    ← committed BEFORE processing!
  3. Processing message at offset 5
  4. ❌ Application crashes!
  5. Consumer restarts → starts at offset 6
  6. ⚠️ Message at offset 5 is LOST (never processed but committed)

MANUAL COMMIT SOLUTION:
  1. poll() returns messages
  2. Processing message at offset 5
  3. ✅ Processing successful
  4. commitSync() (offset = 5)           ← committed AFTER processing!
  5. ❌ If crash before commit → restart from offset 5 → no data loss!
```

### Synchronous Commit (Safe)

```java
// ===== ManualCommitConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class ManualCommitConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "manual-commit-group");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // ⚡ DISABLE auto-commit
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("order-events"));

        System.out.println("🎧 Manual Commit Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    try {
                        // Process the message
                        System.out.printf("📨 Processing: partition=%d, offset=%d, key=%s%n",
                            record.partition(), record.offset(), record.key());

                        processMessage(record);

                        System.out.printf("  ✅ Successfully processed offset %d%n%n",
                            record.offset());

                    } catch (Exception e) {
                        System.err.printf("  ❌ Failed to process offset %d: %s%n",
                            record.offset(), e.getMessage());
                        // Handle the error (retry, send to DLQ, etc.)
                    }
                }

                // Commit AFTER all records in the batch are processed
                if (!records.isEmpty()) {
                    consumer.commitSync();  // Blocks until committed
                    System.out.println("📌 Offsets committed (sync)\n");
                }
            }
        } finally {
            consumer.close();
        }
    }

    private static void processMessage(ConsumerRecord<String, String> record) {
        // Simulate business logic
        System.out.printf("  🔄 Business logic for: %s%n", record.value());
    }
}
```

### Asynchronous Commit (Faster)

```java
// ===== AsyncCommitConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class AsyncCommitConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "async-commit-group");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("order-events"));

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    processMessage(record);
                }

                // Async commit — doesn't block, uses callback
                if (!records.isEmpty()) {
                    consumer.commitAsync((offsets, exception) -> {
                        if (exception != null) {
                            System.err.println("❌ Async commit failed: " + exception.getMessage());
                        } else {
                            System.out.println("📌 Offsets committed (async)");
                        }
                    });
                }
            }
        } finally {
            // Final sync commit before closing (to ensure last offsets are saved)
            consumer.commitSync();
            consumer.close();
        }
    }

    private static void processMessage(ConsumerRecord<String, String> record) {
        System.out.printf("📨 Processed: partition=%d, offset=%d, value=%s%n",
            record.partition(), record.offset(), record.value());
    }
}
```

### Per-Partition Commit (Most Granular)

```java
// Commit offsets for each partition separately
for (var partitionRecords : records.partitions()) {
    // Get all records for this partition
    List<ConsumerRecord<String, String>> partRecords = records.records(partitionRecords);

    for (ConsumerRecord<String, String> record : partRecords) {
        processMessage(record);
    }

    // Commit only THIS partition's offset
    long lastOffset = partRecords.get(partRecords.size() - 1).offset();
    consumer.commitSync(Collections.singletonMap(
        partitionRecords,
        new OffsetAndMetadata(lastOffset + 1)  // +1 because we commit the NEXT offset
    ));

    System.out.printf("📌 Committed partition %d up to offset %d%n",
        partitionRecords.partition(), lastOffset);
}
```

### Commit Strategies Comparison

| Strategy          | Pros                                 | Cons                                    | When to Use               |
| ----------------- | ------------------------------------ | --------------------------------------- | ------------------------- |
| **Auto Commit**   | Simplest, no extra code              | Risk of data loss or duplicates         | Non-critical data         |
| **Sync Commit**   | Guarantees commit success            | Slower (blocks the thread)              | Critical data             |
| **Async Commit**  | Non-blocking, faster                 | May fail silently                       | High throughput needed    |
| **Per-Partition** | Most granular control                | Most complex code                       | Maximum reliability       |

---

## 9. Consumer with JSON Deserialization

Most real-world Kafka messages are **JSON**. Here's how to consume and parse them.

### The Event Classes

```java
// ===== OrderEvent.java =====

public class OrderEvent {
    private String eventType;
    private String eventId;
    private String timestamp;
    private OrderData data;

    // Default constructor (required for Jackson)
    public OrderEvent() {}

    // Getters and Setters
    public String getEventType() { return eventType; }
    public void setEventType(String eventType) { this.eventType = eventType; }

    public String getEventId() { return eventId; }
    public void setEventId(String eventId) { this.eventId = eventId; }

    public String getTimestamp() { return timestamp; }
    public void setTimestamp(String timestamp) { this.timestamp = timestamp; }

    public OrderData getData() { return data; }
    public void setData(OrderData data) { this.data = data; }

    @Override
    public String toString() {
        return String.format("OrderEvent{type='%s', id='%s', data=%s}",
            eventType, eventId, data);
    }
}
```

```java
// ===== OrderData.java =====

public class OrderData {
    private String orderId;
    private String userId;
    private String product;
    private double amount;
    private String status;

    public OrderData() {}

    // Getters and Setters
    public String getOrderId() { return orderId; }
    public void setOrderId(String orderId) { this.orderId = orderId; }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }

    public String getProduct() { return product; }
    public void setProduct(String product) { this.product = product; }

    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    @Override
    public String toString() {
        return String.format("OrderData{orderId='%s', userId='%s', product='%s', amount=%.2f}",
            orderId, userId, product, amount);
    }
}
```

### JSON Consumer — Full Code

```java
// ===== JsonConsumer.java =====

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class JsonConsumer {

    // Jackson ObjectMapper for JSON → Java object conversion
    private static final ObjectMapper objectMapper = new ObjectMapper();

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "json-consumer-group");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("order-events"));

        System.out.println("🎧 JSON Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    try {
                        // ── Parse JSON string → Java Object ──
                        OrderEvent event = objectMapper.readValue(
                            record.value(),
                            OrderEvent.class
                        );

                        System.out.println("═══════════════════════════════════════════");
                        System.out.println("📨 ORDER EVENT RECEIVED:");
                        System.out.println("───────────────────────────────────────────");
                        System.out.printf("  Event Type : %s%n", event.getEventType());
                        System.out.printf("  Event ID   : %s%n", event.getEventId());
                        System.out.printf("  Timestamp  : %s%n", event.getTimestamp());
                        System.out.printf("  Order ID   : %s%n", event.getData().getOrderId());
                        System.out.printf("  User ID    : %s%n", event.getData().getUserId());
                        System.out.printf("  Product    : %s%n", event.getData().getProduct());
                        System.out.printf("  Amount     : $%.2f%n", event.getData().getAmount());
                        System.out.printf("  Status     : %s%n", event.getData().getStatus());
                        System.out.printf("  [Partition=%d, Offset=%d]%n",
                            record.partition(), record.offset());
                        System.out.println("═══════════════════════════════════════════\n");

                        // Route based on event type
                        switch (event.getEventType()) {
                            case "ORDER_PLACED":
                                handleOrderPlaced(event);
                                break;
                            case "ORDER_CANCELLED":
                                handleOrderCancelled(event);
                                break;
                            case "ORDER_SHIPPED":
                                handleOrderShipped(event);
                                break;
                            default:
                                System.out.println("⚠️  Unknown event type: " + event.getEventType());
                        }

                    } catch (Exception e) {
                        System.err.printf("❌ Failed to parse message at offset %d: %s%n",
                            record.offset(), e.getMessage());
                    }
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        } finally {
            consumer.close();
        }
    }

    private static void handleOrderPlaced(OrderEvent event) {
        System.out.printf("🛒 New order! %s ordered '%s' for $%.2f%n%n",
            event.getData().getUserId(),
            event.getData().getProduct(),
            event.getData().getAmount());
    }

    private static void handleOrderCancelled(OrderEvent event) {
        System.out.printf("🚫 Order %s has been cancelled.%n%n",
            event.getData().getOrderId());
    }

    private static void handleOrderShipped(OrderEvent event) {
        System.out.printf("📦 Order %s has been shipped!%n%n",
            event.getData().getOrderId());
    }
}
```

### Sample JSON Message Being Consumed

```json
{
  "eventType": "ORDER_PLACED",
  "eventId": "evt-a1b2c3d4",
  "timestamp": "2026-03-10T14:30:00Z",
  "data": {
    "orderId": "order-12345",
    "userId": "user-123",
    "product": "Mechanical Keyboard",
    "amount": 129.99,
    "status": "PENDING"
  }
}
```

### Output

```
═══════════════════════════════════════════
📨 ORDER EVENT RECEIVED:
───────────────────────────────────────────
  Event Type : ORDER_PLACED
  Event ID   : evt-a1b2c3d4
  Timestamp  : 2026-03-10T14:30:00Z
  Order ID   : order-12345
  User ID    : user-123
  Product    : Mechanical Keyboard
  Amount     : $129.99
  Status     : PENDING
  [Partition=0, Offset=42]
═══════════════════════════════════════════

🛒 New order! user-123 ordered 'Mechanical Keyboard' for $129.99
```

---

## 10. Consumer Groups — How They Work

### Visual Explanation

```
Topic "order-events" has 4 partitions: P0, P1, P2, P3

═══ SCENARIO 1: 1 Consumer in Group ═══

Consumer Group "order-processors":
  Consumer A → P0, P1, P2, P3   (one consumer reads ALL partitions)

═══ SCENARIO 2: 2 Consumers in Group ═══

Consumer Group "order-processors":
  Consumer A → P0, P1            (partitions are SPLIT)
  Consumer B → P2, P3

═══ SCENARIO 3: 4 Consumers in Group ═══

Consumer Group "order-processors":
  Consumer A → P0                (one partition each — IDEAL!)
  Consumer B → P1
  Consumer C → P2
  Consumer D → P3

═══ SCENARIO 4: 5 Consumers in Group (more consumers than partitions) ═══

Consumer Group "order-processors":
  Consumer A → P0
  Consumer B → P1
  Consumer C → P2
  Consumer D → P3
  Consumer E → (idle, nothing to read!)  ← WASTED!

═══ SCENARIO 5: Two DIFFERENT Groups ═══

Consumer Group "order-processors":
  Consumer A → P0, P1
  Consumer B → P2, P3

Consumer Group "analytics-service":     ← INDEPENDENT group
  Consumer X → P0, P1, P2, P3          ← Reads ALL messages too!

Both groups read ALL messages independently.
```

### Running Multiple Consumers in Same Group

```java
// Just run the SAME consumer code in multiple terminals/processes
// with the SAME group.id — Kafka will automatically distribute partitions

// Terminal 1:
// java BasicConsumer   → gets P0, P1

// Terminal 2:
// java BasicConsumer   → gets P2, P3

// Kafka handles the distribution automatically!
```

---

## 11. Rebalance Listener — Detect Partition Changes

When consumers join or leave a group, Kafka **rebalances** partitions. You can listen for these events.

```java
// ===== RebalanceListenerConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class RebalanceListenerConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "rebalance-demo-group");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        // Subscribe WITH a rebalance listener
        consumer.subscribe(
            Collections.singletonList("order-events"),
            new ConsumerRebalanceListener() {

                @Override
                public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
                    // Called BEFORE rebalance — partitions are about to be taken away
                    System.out.println("⚠️  PARTITIONS REVOKED (being taken away):");
                    for (TopicPartition tp : partitions) {
                        System.out.printf("    - %s partition %d%n", tp.topic(), tp.partition());
                    }
                    // IMPORTANT: Commit offsets here to avoid re-processing
                    consumer.commitSync();
                    System.out.println("    📌 Offsets committed before rebalance\n");
                }

                @Override
                public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
                    // Called AFTER rebalance — new partitions have been assigned
                    System.out.println("✅ PARTITIONS ASSIGNED (new assignment):");
                    for (TopicPartition tp : partitions) {
                        System.out.printf("    + %s partition %d%n", tp.topic(), tp.partition());
                    }
                    System.out.println();
                }
            }
        );

        System.out.println("🎧 Rebalance-Aware Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("📨 [P%d|O%d] %s%n",
                        record.partition(), record.offset(), record.value());
                }
                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```

### What Happens During Rebalance

```
                    Consumer A         Consumer B (new)
                        │                    │
Time ──────────────────────────────────────────────────▶
        │               │                    │
        │  Reading P0,  │                    │
        │  P1, P2, P3   │                    │
        │               │                    │
        ├───────────── REBALANCE TRIGGERED ──┤
        │               │                    │
        │  onPartitionsRevoked(P0,P1,P2,P3)  │
        │  → commitSync()                    │
        │               │                    │
        │  onPartitionsAssigned(P0, P1)      │
        │               │  onPartitionsAssigned(P2, P3)
        │               │                    │
        │  Reading P0,  │  Reading P2,       │
        │  P1           │  P3                │
```

---

## 12. Consuming from Specific Partitions & Offsets

Sometimes you want to read **specific partitions** or **start from a specific offset**.

```java
// ===== SpecificPartitionConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class SpecificPartitionConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        // NOTE: No group.id needed when using assign() instead of subscribe()

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        // ── Option 1: Assign specific partitions ──
        TopicPartition partition0 = new TopicPartition("order-events", 0);
        TopicPartition partition2 = new TopicPartition("order-events", 2);
        consumer.assign(Arrays.asList(partition0, partition2));

        // ── Option 2: Seek to a specific offset ──
        consumer.seek(partition0, 10);    // Start from offset 10 on partition 0
        consumer.seek(partition2, 5);     // Start from offset 5 on partition 2

        // ── Option 3: Seek to beginning or end ──
        // consumer.seekToBeginning(Arrays.asList(partition0));
        // consumer.seekToEnd(Arrays.asList(partition2));

        System.out.println("🎯 Reading P0 from offset 10, P2 from offset 5...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("📨 [P%d|O%d] key=%s value=%s%n",
                        record.partition(), record.offset(), record.key(), record.value());
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```

---

## 13. Multi-Threaded Consumer

Run multiple consumers in **parallel threads** within a single application.

```java
// ===== MultiThreadedConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

public class MultiThreadedConsumer {

    private final AtomicBoolean running = new AtomicBoolean(true);
    private final ExecutorService executor;
    private final List<ConsumerWorker> workers = new ArrayList<>();

    public MultiThreadedConsumer(int numThreads) {
        this.executor = Executors.newFixedThreadPool(numThreads);

        for (int i = 0; i < numThreads; i++) {
            ConsumerWorker worker = new ConsumerWorker(i, running);
            workers.add(worker);
            executor.submit(worker);
        }
    }

    public void shutdown() {
        running.set(false);
        executor.shutdown();
        try {
            executor.awaitTermination(10, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("👋 All consumer threads stopped.");
    }

    // ── Inner class: Each thread runs its own consumer ──
    static class ConsumerWorker implements Runnable {
        private final int workerId;
        private final AtomicBoolean running;

        ConsumerWorker(int workerId, AtomicBoolean running) {
            this.workerId = workerId;
            this.running = running;
        }

        @Override
        public void run() {
            Properties props = new Properties();
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
            props.put(ConsumerConfig.GROUP_ID_CONFIG, "multi-threaded-group");
            props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
            props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

            KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
            consumer.subscribe(Collections.singletonList("order-events"));

            System.out.printf("🧵 Worker-%d started%n", workerId);

            try {
                while (running.get()) {
                    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
                    for (ConsumerRecord<String, String> record : records) {
                        System.out.printf("🧵 Worker-%d | P%d | O%d | %s%n",
                            workerId, record.partition(), record.offset(), record.value());
                    }
                    if (!records.isEmpty()) {
                        consumer.commitSync();
                    }
                }
            } finally {
                consumer.close();
                System.out.printf("🧵 Worker-%d stopped%n", workerId);
            }
        }
    }

    // ── Main ──
    public static void main(String[] args) {
        int threadCount = 3;  // Match this to your partition count
        MultiThreadedConsumer multiConsumer = new MultiThreadedConsumer(threadCount);

        // Shutdown hook for clean exit
        Runtime.getRuntime().addShutdownHook(new Thread(multiConsumer::shutdown));

        System.out.printf("🚀 Started %d consumer threads. Press Ctrl+C to stop.%n%n", threadCount);
    }
}
```

---

## 14. Graceful Shutdown

Always shut down consumers cleanly to avoid rebalancing delays.

```java
// ===== GracefulShutdownConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.errors.WakeupException;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class GracefulShutdownConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "graceful-shutdown-group");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        // ── Register shutdown hook ──
        final Thread mainThread = Thread.currentThread();
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("\n🛑 Shutdown signal received...");
            consumer.wakeup();  // Causes poll() to throw WakeupException
            try {
                mainThread.join();  // Wait for main thread to finish
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }));

        try {
            consumer.subscribe(Collections.singletonList("order-events"));
            System.out.println("🎧 Consumer started (with graceful shutdown). Press Ctrl+C to stop.\n");

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("📨 [P%d|O%d] %s%n",
                        record.partition(), record.offset(), record.value());
                }
                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }

        } catch (WakeupException e) {
            // Expected when shutdown hook calls consumer.wakeup()
            System.out.println("⏹️  Consumer wakeup triggered (expected on shutdown)");

        } finally {
            // Commit final offsets and close cleanly
            try {
                consumer.commitSync();
                System.out.println("📌 Final offsets committed");
            } catch (Exception e) {
                System.err.println("⚠️  Could not commit final offsets: " + e.getMessage());
            }
            consumer.close();
            System.out.println("👋 Consumer closed gracefully. Goodbye!");
        }
    }
}
```

### Shutdown Flow

```
User presses Ctrl+C
       │
       ▼
Shutdown Hook runs
       │
       ▼
consumer.wakeup() called
       │
       ▼
poll() throws WakeupException
       │
       ▼
Caught in catch block
       │
       ▼
finally block runs:
  → commitSync() (save final offsets)
  → consumer.close() (leave group cleanly)
       │
       ▼
Consumer removed from group → triggers fast rebalance
```

---

## 15. Error Handling & Retry Patterns

```java
// ===== RetryConsumer.java =====

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.*;

public class RetryConsumer {

    private static final ObjectMapper mapper = new ObjectMapper();
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 1000;

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "retry-consumer-group");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("order-events"));

        System.out.println("🎧 Retry Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    boolean success = false;

                    for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
                        try {
                            System.out.printf("📨 [Attempt %d/%d] Processing offset %d...%n",
                                attempt, MAX_RETRIES, record.offset());

                            processRecord(record);
                            success = true;
                            System.out.printf("  ✅ Success on attempt %d%n%n", attempt);
                            break;  // Exit retry loop on success

                        } catch (Exception e) {
                            System.err.printf("  ❌ Attempt %d failed: %s%n", attempt, e.getMessage());

                            if (attempt < MAX_RETRIES) {
                                long delay = RETRY_DELAY_MS * attempt;  // Exponential-ish backoff
                                System.out.printf("  ⏳ Retrying in %dms...%n", delay);
                                Thread.sleep(delay);
                            }
                        }
                    }

                    if (!success) {
                        // All retries exhausted → send to Dead Letter Queue
                        System.err.printf("  💀 All %d retries failed for offset %d → Sending to DLQ%n%n",
                            MAX_RETRIES, record.offset());
                        sendToDeadLetterQueue(record);
                    }
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            consumer.close();
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) throws Exception {
        // Your business logic here
        // Simulating random failures for demonstration
        if (Math.random() < 0.3) {
            throw new RuntimeException("Simulated processing error");
        }
        System.out.printf("  🔄 Processed: %s%n", record.value());
    }

    private static void sendToDeadLetterQueue(ConsumerRecord<String, String> record) {
        // In production: send to a "order-events.DLQ" topic using a producer
        System.out.printf("  📤 DLQ: partition=%d, offset=%d, value=%s%n",
            record.partition(), record.offset(), record.value());
    }
}
```

---

## 16. Dead Letter Queue (DLQ) Pattern

```java
// ===== DLQConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import java.time.Duration;
import java.util.*;

public class DLQConsumer {

    private static final String SOURCE_TOPIC = "order-events";
    private static final String DLQ_TOPIC = "order-events.DLQ";

    public static void main(String[] args) {
        // Consumer setup
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "dlq-consumer-group");
        consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        consumerProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // DLQ Producer setup
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
        KafkaProducer<String, String> dlqProducer = new KafkaProducer<>(producerProps);

        consumer.subscribe(Collections.singletonList(SOURCE_TOPIC));
        System.out.println("🎧 DLQ-Enabled Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, String> record : records) {
                    try {
                        processRecord(record);
                    } catch (Exception e) {
                        // ── Send failed message to DLQ ──
                        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
                            DLQ_TOPIC, record.key(), record.value()
                        );
                        // Attach metadata as headers
                        dlqRecord.headers()
                            .add("original-topic", SOURCE_TOPIC.getBytes())
                            .add("original-partition", String.valueOf(record.partition()).getBytes())
                            .add("original-offset", String.valueOf(record.offset()).getBytes())
                            .add("error-message", e.getMessage().getBytes())
                            .add("error-timestamp", java.time.Instant.now().toString().getBytes());

                        dlqProducer.send(dlqRecord);
                        System.out.printf("  💀 Sent to DLQ: offset=%d, error=%s%n%n",
                            record.offset(), e.getMessage());
                    }
                }

                if (!records.isEmpty()) {
                    consumer.commitSync();
                }
            }
        } finally {
            consumer.close();
            dlqProducer.close();
        }
    }

    private static void processRecord(ConsumerRecord<String, String> record) throws Exception {
        System.out.printf("📨 Processing: [P%d|O%d] %s%n", record.partition(), record.offset(), record.value());
        // Your business logic here...
    }
}
```

### DLQ Flow Diagram

```
                          ┌─────────────────┐
  order-events ──────────▶│    Consumer     │
                          │  (processes     │
                          │   messages)     │
                          └────────┬────────┘
                                   │
                          ┌────────┴────────┐
                          │                 │
                     ✅ Success         ❌ Failure
                          │                 │
                    Continue to         Send to DLQ
                    next message             │
                                             ▼
                                   ┌─────────────────┐
                                   │ order-events.DLQ│
                                   │ (failed msgs +  │
                                   │  error metadata) │
                                   └─────────────────┘
                                             │
                                    (Later: investigate,
                                     fix, and reprocess)
```

---

## 17. Exactly-Once Consumer with Transactions

Read from one topic, process, and write to another topic — all **atomically**.

```java
// ===== ExactlyOnceConsumer.java =====

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import java.time.Duration;
import java.util.*;

public class ExactlyOnceConsumer {

    public static void main(String[] args) {
        // Consumer config
        Properties consumerProps = new Properties();
        consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        consumerProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, "exactly-once-group");
        consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");  // Only read committed msgs

        // Producer config (transactional)
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producerProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-transactional-id");
        producerProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps);
        KafkaProducer<String, String> producer = new KafkaProducer<>(producerProps);

        producer.initTransactions();
        consumer.subscribe(Collections.singletonList("order-events"));

        System.out.println("🔒 Exactly-Once Consumer started...\n");

        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));

                if (!records.isEmpty()) {
                    // Begin atomic transaction
                    producer.beginTransaction();

                    try {
                        for (ConsumerRecord<String, String> record : records) {
                            // Transform and write to output topic
                            String transformedValue = "PROCESSED: " + record.value();
                            producer.send(new ProducerRecord<>(
                                "processed-orders", record.key(), transformedValue));

                            System.out.printf("🔄 [P%d|O%d] Transformed & sent to output topic%n",
                                record.partition(), record.offset());
                        }

                        // Send consumer offsets as part of the transaction
                        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
                        for (TopicPartition partition : records.partitions()) {
                            List<ConsumerRecord<String, String>> partRecords = records.records(partition);
                            long lastOffset = partRecords.get(partRecords.size() - 1).offset();
                            offsets.put(partition, new OffsetAndMetadata(lastOffset + 1));
                        }
                        producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

                        // Commit transaction — ALL or NOTHING
                        producer.commitTransaction();
                        System.out.println("✅ Transaction committed (exactly-once)\n");

                    } catch (Exception e) {
                        // Abort transaction — nothing is committed
                        producer.abortTransaction();
                        System.err.println("❌ Transaction aborted: " + e.getMessage());
                    }
                }
            }
        } finally {
            consumer.close();
            producer.close();
        }
    }
}
```

---

## 18. Spring Boot Kafka Consumer (Bonus)

### application.yml

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: spring-order-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      enable-auto-commit: false
    listener:
      ack-mode: manual  # For manual offset commit
      concurrency: 3    # Number of consumer threads
```

### Simple Listener

```java
// ===== OrderEventListener.java =====

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class OrderEventListener {

    // ── Basic Listener ──
    @KafkaListener(topics = "order-events", groupId = "spring-order-group")
    public void handleOrderEvent(
            @Payload String message,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
            Acknowledgment acknowledgment) {

        System.out.println("═══════════════════════════════════════");
        System.out.printf("📨 Spring Consumer received:%n");
        System.out.printf("   Partition : %d%n", partition);
        System.out.printf("   Offset    : %d%n", offset);
        System.out.printf("   Timestamp : %d%n", timestamp);
        System.out.printf("   Payload   : %s%n", message);
        System.out.println("═══════════════════════════════════════\n");

        // Process...
        processOrder(message);

        // Manual acknowledge (commit offset)
        acknowledgment.acknowledge();
        System.out.println("📌 Offset committed\n");
    }

    private void processOrder(String message) {
        System.out.println("🔄 Processing order...");
    }
}
```

### Listener with JSON Deserialization

```java
// ===== JsonOrderListener.java =====

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.TopicPartition;
import org.springframework.kafka.annotation.PartitionOffset;
import org.springframework.stereotype.Component;

@Component
public class JsonOrderListener {

    private final ObjectMapper mapper = new ObjectMapper();

    // Listen to ALL partitions
    @KafkaListener(topics = "order-events", groupId = "spring-json-group")
    public void onOrderEvent(String message) {
        try {
            JsonNode event = mapper.readTree(message);

            String eventType = event.get("eventType").asText();
            JsonNode data = event.get("data");

            switch (eventType) {
                case "ORDER_PLACED":
                    System.out.printf("🛒 New Order: %s — %s ($%.2f)%n",
                        data.get("orderId").asText(),
                        data.get("product").asText(),
                        data.get("amount").asDouble());
                    break;

                case "ORDER_CANCELLED":
                    System.out.printf("🚫 Cancelled: %s%n", data.get("orderId").asText());
                    break;

                default:
                    System.out.println("⚠️ Unknown event: " + eventType);
            }

        } catch (Exception e) {
            System.err.println("❌ Parse error: " + e.getMessage());
        }
    }

    // Listen to SPECIFIC partitions from SPECIFIC offsets
    @KafkaListener(
        groupId = "spring-specific-group",
        topicPartitions = @TopicPartition(
            topic = "order-events",
            partitionOffsets = {
                @PartitionOffset(partition = "0", initialOffset = "0"),
                @PartitionOffset(partition = "1", initialOffset = "0")
            }
        )
    )
    public void onSpecificPartitions(String message) {
        System.out.println("🎯 Specific partition message: " + message);
    }
}
```

---

## 19. Consumer Lag — What It Is & How to Monitor

```
Consumer Lag = Latest Offset (in partition) − Consumer's Committed Offset

Partition 0:
  Latest message offset:     1000
  Consumer committed offset:  950
  Consumer LAG:                50  messages behind!

High lag = Consumer can't keep up with producers
```

### Check Lag via CLI

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group
```

**Output:**

```
GROUP                TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-consumer-group    order-events  0          950             1000            50
my-consumer-group    order-events  1          1200            1200            0
my-consumer-group    order-events  2          800             875             75
```

### How to Reduce Lag

| Strategy                         | How                                                 |
| -------------------------------- | --------------------------------------------------- |
| **Add more consumers**           | Must be ≤ number of partitions                      |
| **Add more partitions**          | Increases parallelism                               |
| **Optimize processing**          | Make your `processMessage()` faster                 |
| **Increase `max.poll.records`**  | Process more records per poll                       |
| **Use async processing**         | Process records in parallel within each poll batch  |

---

## 20. Common Errors & Troubleshooting

| Error                                      | Cause                                      | Solution                                               |
| ------------------------------------------ | ------------------------------------------ | ------------------------------------------------------ |
| `Connection to node -1 could not be established` | Kafka broker not reachable           | Check broker is running, verify `bootstrap.servers`    |
| `No current assignment for partition`      | Consumer not subscribed/assigned           | Call `subscribe()` or `assign()` before `poll()`       |
| `Offset out of range`                      | Requested offset doesn't exist             | Set `auto.offset.reset=earliest`                       |
| `CommitFailedException`                    | Rebalance happened during commit           | Process faster or increase `max.poll.interval.ms`      |
| `Consumer poll timeout expired`            | Processing took too long between polls     | Increase `max.poll.interval.ms` or reduce batch size   |
| `Deserialization error`                    | Message format doesn't match deserializer  | Check producer's serializer matches consumer's         |
| `Group coordinator not available`          | Broker is starting up                      | Wait and retry                                         |
| `Rebalancing in progress`                  | Consumer joining/leaving group             | Normal — wait for it to complete                       |

---

## 21. Best Practices Checklist

```
✅ ALWAYS use a meaningful group.id
✅ ALWAYS handle exceptions inside the poll loop (don't let them kill the consumer)
✅ ALWAYS implement graceful shutdown (consumer.wakeup() + shutdown hook)
✅ USE manual commit for critical data
✅ MATCH consumer thread count to partition count
✅ SET appropriate max.poll.interval.ms for slow processing
✅ MONITOR consumer lag
✅ USE Dead Letter Queues for unprocessable messages
✅ LOG partition and offset for every processed message (aids debugging)
✅ USE idempotent processing (handle duplicates gracefully)

❌ DON'T ignore exceptions in the processing loop
❌ DON'T have more consumers than partitions (extras will be idle)
❌ DON'T do heavy processing inside poll() without tuning max.poll.interval.ms
❌ DON'T forget to close the consumer in a finally block
❌ DON'T use auto-commit for critical financial data
```

---

## 22. Quick Reference Cheat Sheet

### Consumer Creation (Minimal)

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "my-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("my-topic"));

while (true) {
    var records = consumer.poll(Duration.ofMillis(1000));
    records.forEach(r -> System.out.println(r.value()));
}
```

### CLI Commands

```bash
# Consume from beginning
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic order-events --from-beginning

# Consume with keys and headers
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic order-events --from-beginning \
  --property print.key=true \
  --property print.headers=true \
  --property print.timestamp=true

# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# Reset offsets to earliest
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --to-earliest --execute

# Reset offsets to latest
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --to-latest --execute

# Reset to specific offset
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events:0 \
  --reset-offsets --to-offset 100 --execute
```

---

## 23. Summary

```
┌──────────────────────────────────────────────────────────────────┐
│              KAFKA CONSUMER IN JAVA — SUMMARY                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WHAT: Application that reads messages from Kafka topics         │
│                                                                  │
│  HOW IT WORKS:                                                   │
│    1. Configure (broker, deserializers, group ID)                │
│    2. Subscribe to topic(s)                                      │
│    3. Poll in a loop → get batch of records                      │
│    4. Process each record                                        │
│    5. Commit offset (auto or manual)                             │
│    6. Repeat!                                                    │
│                                                                  │
│  KEY CONCEPTS:                                                   │
│    • Consumer Group — share work across consumers                │
│    • Offset — tracks reading position per partition              │
│    • Rebalancing — redistribution when consumers change          │
│    • Commit — save progress (auto, sync, async)                  │
│                                                                  │
│  PATTERNS COVERED:                                               │
│    ✅ Basic Consumer                                              │
│    ✅ Manual Offset Commit (Sync, Async, Per-Partition)           │
│    ✅ JSON Deserialization                                        │
│    ✅ Consumer Groups                                             │
│    ✅ Rebalance Listener                                          │
│    ✅ Specific Partitions & Offsets                               │
│    ✅ Multi-Threaded Consumer                                     │
│    ✅ Graceful Shutdown                                           │
│    ✅ Error Handling & Retries                                    │
│    ✅ Dead Letter Queue (DLQ)                                     │
│    ✅ Exactly-Once Transactions                                   │
│    ✅ Spring Boot Integration                                     │
│                                                                  │
└───────────────────────────────────────────────��──────────────────┘
```

---

> _"A well-designed Kafka consumer is the backbone of any real-time data pipeline."_

**Happy Consuming! 🚀**
