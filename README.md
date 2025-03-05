# Kafka-Learning

# Scenario:

## Q- You have a Kafka-based data pipeline where events must be processed exactly once. However, sometimes duplicate messages are processed due to retries.
## How do you ensure idempotency?

Solution:-

Enable idempotent producers:

```
enable.idempotence=true
```
This prevents duplicate messages due to retries.

Use transactional producers:

```
transactional.id= "unique-producer-id"
```
Ensures all messages within a transaction are either fully committed or fully aborted.
Deduplicate at the consumer side using unique message keys and a database table/cache (Redis) to track processed messages.

2. Handling Out-of-Order Messages
Scenario:
You have multiple partitions for a topic, but events are consumed out of order in some cases. How can you preserve message order?

Follow-up Questions:
How does Kafka guarantee ordering within partitions?
What happens when partitions increase dynamically?
Solution:
Use a single partition per key to maintain order:

java
Copy
Edit
producer.send(new ProducerRecord<>("topic", "key", "message"));
Messages with the same key always go to the same partition.
Use a timestamp-based ordering system at the consumer side if events from multiple partitions need ordering.

3. Kafka Consumer Lag Issues
Scenario:
You notice that some consumers in a Kafka consumer group are lagging, causing real-time processing delays. How do you troubleshoot and fix this?

Follow-up Questions:
What tools can you use to monitor consumer lag?
How do you optimize consumer processing speed?
Solution:
Monitor consumer lag using:

shell
Copy
Edit
kafka-consumer-groups --bootstrap-server <kafka-broker> --group <consumer-group> --describe
This command shows lag per partition.
Scale consumers horizontally by adding more consumer instances in the group to distribute the load.

Optimize consumer processing:

Use batch processing instead of processing messages one by one.
Store offsets after processing to avoid reprocessing failures.
java
Copy
Edit
consumer.commitAsync();
Tune fetch.min.bytes and fetch.max.wait.ms to balance latency vs. throughput.
4. Data Loss Prevention in Kafka
Scenario:
Your Kafka application is experiencing data loss, where some messages are missing. How do you ensure zero data loss?

Follow-up Questions:
What is the impact of setting acks=0, acks=1, or acks=all?
How do replication and ISR (In-Sync Replicas) help in preventing data loss?
Solution:
Set acks=all to ensure data is committed to all in-sync replicas before acknowledging the producer.

properties
Copy
Edit
acks=all
Ensures data is not lost if a broker crashes.
Set min.insync.replicas > 1 to avoid sending messages to unhealthy brokers.

properties
Copy
Edit
min.insync.replicas=2
Store consumer offsets manually after successful message processing.

java
Copy
Edit
consumer.commitSync();
5. Kafka Topic Partitioning Strategy
Scenario:
Your system is experiencing uneven load distribution across Kafka partitions, causing some consumers to process more messages than others. How do you fix this?

Follow-up Questions:
How does Kafka partitioning strategy work?
What are different partitioning techniques?
Solution:
Use a custom partitioner to balance partition assignments:

java
Copy
Edit
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        return key.hashCode() % numPartitions; // Distributes evenly
    }
}
Increase partitions dynamically to distribute the load better:

shell
Copy
Edit
kafka-topics --alter --topic my-topic --partitions 10 --bootstrap-server localhost:9092
Note: Increasing partitions does not redistribute existing messages.
6. Retrying Failed Messages
Scenario:
Some messages fail during processing due to transient issues (e.g., database connection failure). How do you implement a retry mechanism in Kafka consumers?

Follow-up Questions:
How does Kafka handle failed messages?
What happens when a consumer crashes mid-processing?
Solution:
Use Dead Letter Queue (DLQ) to store failed messages for later reprocessing.

java
Copy
Edit
producer.send(new ProducerRecord<>("DLQ-topic", failedMessage));
Implement exponential backoff retry:

java
Copy
Edit
while (retryCount < MAX_RETRIES) {
    try {
        processMessage(message);
        break;
    } catch (Exception e) {
        Thread.sleep((long) Math.pow(2, retryCount) * 1000); // Exponential backoff
        retryCount++;
    }
}
7. Kafka High Availability Setup
Scenario:
You need to deploy Kafka in production and ensure high availability (HA). How do you design your architecture to avoid downtime?

Follow-up Questions:
How does Kafka handle broker failures?
What is the role of Zookeeper in Kafka HA?
Solution:
Use multiple brokers (e.g., at least 3 brokers) to distribute partitions and avoid a single point of failure.

Enable replication (default is 3) to ensure data availability:

properties
Copy
Edit
replication.factor=3
Set up ISR (In-Sync Replicas) monitoring to detect unhealthy brokers.

Use a load balancer (e.g., Kafka MirrorMaker) for cross-cluster replication.

8. Cross-Data Center Replication
Scenario:
Your company wants to set up Kafka replication across multiple data centers to ensure disaster recovery. How do you implement this?

Follow-up Questions:
What is the role of Kafka MirrorMaker in cross-cluster replication?
What are the trade-offs of synchronous vs. asynchronous replication?
Solution:
Use Kafka MirrorMaker to replicate messages from one cluster to another:

shell
Copy
Edit
bin/mirror-maker.sh --consumer.config consumer.properties --producer.config producer.properties --whitelist "topic-to-replicate"
Ensures disaster recovery and geo-redundancy.
Choose asynchronous replication for better performance but possible message lag.

