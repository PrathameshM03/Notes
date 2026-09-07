# Spring Framework — 11: Messaging with Kafka & RabbitMQ
### New topic. Full depth, story format.

---

## Story 1: The order service that went down and took checkout with it

**The situation:** An e-commerce `OrderService` calls three other services synchronously, directly, over HTTP, the moment an order is placed:
```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryClient.reserveStock(order);      // HTTP call
        notificationClient.sendConfirmation(order); // HTTP call
        analyticsClient.recordPurchase(order);      // HTTP call
    }
}
```
**Where it breaks:** the `analyticsClient` service — which just logs purchase data for a dashboard nobody's even looking at right now — has a bad deploy and starts timing out. Because `placeOrder` calls it **synchronously**, every single checkout on the entire site now hangs waiting for a timeout from the *least important* service in the chain, and eventually **checkout itself goes down** — taking real revenue with it — because of a failure in a completely non-critical, "nice to have" analytics service.

**Why direct HTTP coupling is the actual root cause:** this is the distributed-systems version of the tight-coupling story from the very first scenario doc — `OrderService` is tightly coupled to the **availability** of three other services, even though logically, "an order was placed" is a fact that inventory, notifications, and analytics should each get to react to **independently**, on their own schedule, without blocking each other or the original checkout flow.

---

## Story 2: The message broker as decoupling — the core idea, before any code

**The situation:** Instead of `OrderService` calling each downstream service directly, it publishes **one event** — "an order was placed" — to a **message broker**, and each interested service **subscribes** to that event independently:
```
OrderService --publishes--> [ Message Broker ] --delivers to--> InventoryService
                                                --delivers to--> NotificationService
                                                --delivers to--> AnalyticsService
```
**Why this fixes Story 1's exact failure mode:** `OrderService` now only depends on the broker being available (a much simpler, more reliable single dependency) — not on all three downstream services being simultaneously healthy. If `AnalyticsService` is down, the message just **waits in the broker** until it recovers — checkout is completely unaffected.

**Two message broker philosophies, both worth understanding — not just "pick one":**

### RabbitMQ — a traditional message queue
Built around the idea of a **queue**: a message is put in, and (typically) **one consumer** takes it out and processes it. Good for **task distribution** — "here's a job, whichever worker picks it up next should do it, exactly once."
```java
@RabbitListener(queues = "inventory.reserve")
public void handleOrderPlaced(OrderPlacedEvent event) {
    inventoryService.reserveStock(event.getOrderId());
}
```

### Kafka — a distributed log
Built around the idea of a **log/stream**: messages are appended to a topic and **retained** (not deleted on consumption) for a configured period, and **multiple independent consumer groups** can each read the *same* stream of messages, each tracking their own position. Good for **event streaming** — "many different services need to know this happened, each in their own way, and we might want to replay history later."
```java
@KafkaListener(topics = "order-events", groupId = "inventory-service")
public void handleOrderPlaced(OrderPlacedEvent event) {
    inventoryService.reserveStock(event.getOrderId());
}
```

**Choosing between them — the real distinguishing question:** *"does exactly one consumer need to handle each message (a task queue), or do multiple independent services each need their own full view of every event (a stream)?"* RabbitMQ fits the former naturally; Kafka fits the latter naturally (though both can be configured to approximate the other's behavior — this is about which one requires less fighting the tool's natural design for your use case).

---

## Story 3: Publishing an event with Spring Kafka — full working example

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```
```properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.group-id=inventory-service
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.properties.spring.json.trusted.packages=*
```
```java
public record OrderPlacedEvent(Long orderId, Long customerId, List<OrderItem> items, Instant placedAt) { }
```
```java
@Service
public class OrderService {
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;
    private final OrderRepository orderRepository;

    public void placeOrder(Order order) {
        orderRepository.save(order);
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(), order.getCustomerId(), order.getItems(), Instant.now());
        kafkaTemplate.send("order-events", order.getId().toString(), event);
        // returns almost immediately — publishing to Kafka is fast; consumers process independently
    }
}
```
```java
@Component
public class InventoryEventListener {
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        inventoryService.reserveStock(event.orderId(), event.items());
    }
}

@Component
public class NotificationEventListener {
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        notificationService.sendConfirmation(event.customerId(), event.orderId());
    }
}
```
**Why the message key matters (`order.getId().toString()` above):** Kafka guarantees **ordering only within a single partition**, and messages with the **same key** always land in the same partition — so using the order ID as the key ensures all events for the *same* order are processed in the order they were published, even though events for *different* orders might be processed out of order relative to each other across partitions (which is usually fine — different orders are independent).

---

## Story 4: The message that got processed twice — at-least-once delivery and idempotent consumers

**The situation:** `InventoryEventListener` processes an `OrderPlacedEvent`, successfully reserves stock, but then the app crashes **before** it can acknowledge to Kafka that the message was processed. On restart, Kafka — having never received an acknowledgment — **redelivers the same message**. Stock gets reserved **twice** for one order.

**Why this isn't a Kafka bug — it's an inherent trade-off of "at-least-once delivery":** message brokers generally offer one of three delivery guarantees:
- **At-most-once** — a message might be lost if something fails, but never duplicated.
- **At-least-once** — a message is never lost, but might be delivered more than once (this scenario).
- **Exactly-once** — genuinely hard to guarantee end-to-end across a distributed system; Kafka has support for it in specific configurations, but it adds real complexity and doesn't eliminate the need for careful consumer design.

**Most real systems choose at-least-once and handle duplicates at the application level** — this is precisely the **idempotency** theme from the CRUD scenario doc's Story 1, showing up again at the messaging layer:
```java
@Component
public class InventoryEventListener {
    private final ProcessedEventRepository processedEventRepository;

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    @Transactional
    public void handleOrderPlaced(OrderPlacedEvent event) {
        String eventKey = "order-placed-" + event.orderId();
        if (processedEventRepository.existsByEventKey(eventKey)) {
            return; // already handled this exact event — skip, don't double-reserve stock
        }
        inventoryService.reserveStock(event.orderId(), event.items());
        processedEventRepository.save(new ProcessedEvent(eventKey));
    }
}
```
**The core lesson, stated plainly:** *"design every message consumer to be safely re-runnable with the same message, because it eventually will be."* This isn't a defensive-programming nicety — it's a structural requirement of building on top of at-least-once messaging, the same way retry-safety is a structural requirement of any distributed HTTP API (Story 1 of the CRUD scenario doc).

**Interview angle:** *"How do you handle duplicate message delivery in a Kafka consumer?"* — naming the idempotent-consumer pattern (a processed-events table, or a natural idempotency check in the business logic itself) and explaining *why* duplicates are an inherent property of at-least-once delivery (not a bug to "fix" at the broker level) shows real distributed-systems understanding.

---

## Story 5: The poison message — one bad payload that blocked an entire queue

**The situation:** A malformed event (a bug in the producer serialized a field incorrectly) lands in the `order-events` topic. `InventoryEventListener` throws a deserialization exception trying to process it. Without special handling, Kafka's default consumer behavior can end up **retrying the same failing message indefinitely**, and depending on configuration, this can **block all subsequent messages** in that partition from being processed at all, since ordering-within-partition means later messages wait behind the stuck one.

**The fix — a Dead Letter Topic (DLT), so one bad message doesn't halt everything:**
```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    return new DefaultErrorHandler(recoverer, new FixedBackOff(1000L, 3)); // retry 3 times, then give up
}
```
```java
@KafkaListener(topics = "order-events.DLT", groupId = "inventory-service-dlt")
public void handleFailedEvent(OrderPlacedEvent event) {
    log.error("Order event permanently failed processing: {}", event);
    alertingService.notifyOnCallEngineer("Poison message in order-events", event);
}
```
**The pattern, explained:** after a configured number of retry attempts, a failing message is automatically routed to a **separate topic** (`order-events.DLT` — "dead letter topic") instead of blocking the main queue forever. The main flow keeps processing subsequent (good) messages; the problematic one is set aside for a human (or a dedicated recovery process) to investigate — turning "everything stops" into "one message needs manual attention, everything else keeps working."

**Interview angle:** *"What happens if a Kafka consumer keeps failing to process a specific message?"* — naming the dead-letter-topic pattern, and explaining *why* it exists (preventing one bad message from blocking an entire partition's worth of legitimate, unrelated messages) is the expected depth for a messaging-focused interview question.

---

## Quick-fire: messaging concepts worth knowing exist

| Concept | What it is | Why it matters |
|---|---|---|
| **Consumer group** | A set of consumer instances sharing the work of reading a topic's partitions | Scaling consumers horizontally — Kafka automatically spreads partitions across group members, so adding more instances increases throughput |
| **Partition count** | How many parallel "lanes" a Kafka topic is split into | Sets the **ceiling on parallelism** — you can never have more *actively working* consumers in one group than partitions, so under-provisioning partitions caps your scale-out ability |
| **RabbitMQ exchange types (direct/topic/fanout)** | Different routing strategies for how a published message reaches queues | `fanout` broadcasts to every bound queue (like Kafka's multi-consumer-group model); `direct`/`topic` route selectively based on a routing key — picking the wrong type is a common RabbitMQ design mistake |
| **Outbox pattern** | Writing the "event to publish" into the **same database transaction** as the business change, then a separate process reliably publishes it to Kafka afterward | Solves the subtle problem where `orderRepository.save()` succeeds but the subsequent `kafkaTemplate.send()` fails (or vice versa) — without it, the DB and the message broker can silently disagree about what actually happened |

---
*Next: `12-microservices-spring-cloud.md` — one app becomes ten, and why "just call the other service's URL" stops working at that scale.*
