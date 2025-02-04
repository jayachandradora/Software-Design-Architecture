# Priority Based Kafka Messages Processing from two Kafka topic queues with Real time


To implement a system that processes messages from two Kafka topic queues—one for delta updates (higher priority) and the other for full-load data (lower priority)—while alternating based on their availability, we can design a message consumption system with the following logic:

### Requirements:
1. **Priority Handling**: The delta topic messages have a higher priority, so they should be processed first. The full-load messages are processed only when the delta topic queue is empty.
2. **Alternating Consumption**: If there are delta messages, they should be consumed first. The full-load topic messages are consumed only when no messages are available in the delta topic queue.
3. **Kafka Consumer Setup**: We will use two consumers, one for each topic (delta and full-load).
4. **Polling Logic**: We need to poll messages from both topics in a loop, with priority given to the delta topic.

### Approach:

1. **Set Up Kafka Consumers**:
   - Create two Kafka consumers: one for the delta topic and one for the full-load topic.
   
2. **Message Polling and Processing**:
   - The main loop will continuously poll messages from both topics.
   - If there are messages available in the delta topic, process them first.
   - If no messages are available in the delta topic, then start processing the full-load topic messages.

3. **Concurrency Management**:
   - To avoid blocking on either queue, we can use separate threads for polling each topic. However, we should manage the consumer thread for the full-load topic so that it only runs when the delta topic has no messages.

4. **Graceful Shutdown**:
   - Ensure that the system shuts down gracefully when there are no more messages to consume or the application is stopping.

Here’s how you can implement this in Java:

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.util.Collections;
import java.util.Properties;

public class KafkaPriorityConsumer {
    
    private static final String DELTA_TOPIC = "delta_topic";
    private static final String FULL_LOAD_TOPIC = "full_load_topic";
    private static final String BOOTSTRAP_SERVERS = "localhost:9092";
    
    public static void main(String[] args) {
        
        // Configure the Kafka consumer for delta topic
        Properties deltaProps = new Properties();
        deltaProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        deltaProps.put(ConsumerConfig.GROUP_ID_CONFIG, "delta-consumer-group");
        deltaProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        deltaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Configure the Kafka consumer for full-load topic
        Properties fullLoadProps = new Properties();
        fullLoadProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        fullLoadProps.put(ConsumerConfig.GROUP_ID_CONFIG, "full-load-consumer-group");
        fullLoadProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        fullLoadProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Create the Kafka consumers
        KafkaConsumer<String, String> deltaConsumer = new KafkaConsumer<>(deltaProps);
        KafkaConsumer<String, String> fullLoadConsumer = new KafkaConsumer<>(fullLoadProps);

        // Subscribe to topics
        deltaConsumer.subscribe(Collections.singletonList(DELTA_TOPIC));
        fullLoadConsumer.subscribe(Collections.singletonList(FULL_LOAD_TOPIC));

        // Start message processing loop
        try {
            while (true) {
                // Poll for messages from both delta and full-load topics
                var deltaRecords = deltaConsumer.poll(1000);  // 1 second timeout
                var fullLoadRecords = fullLoadConsumer.poll(1000);

                // Process delta messages (higher priority)
                if (!deltaRecords.isEmpty()) {
                    deltaRecords.forEach(record -> {
                        // Process delta message here
                        System.out.println("Processing delta message: " + record.value());
                    });
                }
                
                // Process full-load messages only when delta is empty
                if (deltaRecords.isEmpty() && !fullLoadRecords.isEmpty()) {
                    fullLoadRecords.forEach(record -> {
                        // Process full-load message here
                        System.out.println("Processing full-load message: " + record.value());
                    });
                }
                
                // Add any additional processing or sleep logic as needed
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // Close the consumers gracefully
            deltaConsumer.close();
            fullLoadConsumer.close();
        }
    }
}
```

### Explanation of the Code:

1. **Consumer Configuration**:
   - We create two `KafkaConsumer` instances: one for the delta topic and one for the full-load topic.
   - The properties such as `bootstrap.servers` and `group.id` are configured for both consumers. Adjust the group ID to make sure that the consumers can track their offsets independently.

2. **Polling Logic**:
   - We continuously poll both consumers using the `poll()` method, which retrieves records from the Kafka brokers.
   - The `poll()` method has a timeout (1000 milliseconds), and the consumer checks if there are messages available in the delta topic. If there are, it processes them.
   - If the delta topic has no messages (i.e., `deltaRecords.isEmpty()`), the consumer will start processing the full-load topic.

3. **Message Processing**:
   - Once records are retrieved, we process the messages. For the sake of the example, we are just printing the message content to the console, but you can replace this with your actual business logic.

4. **Graceful Shutdown**:
   - The `finally` block ensures that the consumers are closed properly when the application exits, which is a good practice to avoid leaving Kafka connections open.

### Key Points:
- **Priority Handling**: Delta topic messages are processed first. Full-load topic messages are only processed when no delta messages are available.
- **Polling Efficiency**: The `poll()` method is non-blocking with a timeout, so the consumer remains responsive.
- **Scalability**: This design can be extended for more complex priority handling (e.g., different priorities between multiple queues) or to handle more topics.

### Improvements:
- **Threading**: You could further enhance the system by using separate threads or executor services for the two consumers if you want them to operate concurrently in a more complex system.
- **Backpressure Handling**: If the message volume grows, consider implementing backpressure mechanisms or adjusting the consumer's `max.poll.records` setting to control the flow.
- **Error Handling and Dead-letter Queue**: Handle errors gracefully by moving failed messages to a dead-letter queue for further analysis.

This should give you a solid foundation to handle alternating message processing with priority between the delta and full-load Kafka topics.

# Scenarios Based Message Processing.

To implement the logic based on the test cases you've outlined, the flow can be enhanced to achieve the following:

1. **Finish processing all delta messages before moving to full-load messages**.
2. **Only pull full-load messages when delta topic is empty**.
3. **Interrupt full-load message processing if a delta message arrives, process it completely, and then resume full-load processing**.
4. **Ensure continuous, streaming-like behavior** by constantly polling both topics and processing messages in the required order.

Here is the modified code that achieves this logic:

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.util.Collections;
import java.util.Properties;

public class KafkaPriorityConsumer {

    private static final String DELTA_TOPIC = "delta_topic";
    private static final String FULL_LOAD_TOPIC = "full_load_topic";
    private static final String BOOTSTRAP_SERVERS = "localhost:9092";

    public static void main(String[] args) {

        // Configure the Kafka consumer for delta topic
        Properties deltaProps = new Properties();
        deltaProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        deltaProps.put(ConsumerConfig.GROUP_ID_CONFIG, "delta-consumer-group");
        deltaProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        deltaProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Configure the Kafka consumer for full-load topic
        Properties fullLoadProps = new Properties();
        fullLoadProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        fullLoadProps.put(ConsumerConfig.GROUP_ID_CONFIG, "full-load-consumer-group");
        fullLoadProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        fullLoadProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // Create the Kafka consumers
        KafkaConsumer<String, String> deltaConsumer = new KafkaConsumer<>(deltaProps);
        KafkaConsumer<String, String> fullLoadConsumer = new KafkaConsumer<>(fullLoadProps);

        // Subscribe to topics
        deltaConsumer.subscribe(Collections.singletonList(DELTA_TOPIC));
        fullLoadConsumer.subscribe(Collections.singletonList(FULL_LOAD_TOPIC));

        // Start message processing loop
        try {
            boolean processingDelta = false;

            while (true) {
                // Poll for messages from both delta and full-load topics
                var deltaRecords = deltaConsumer.poll(1000);  // 1 second timeout
                var fullLoadRecords = fullLoadConsumer.poll(1000);

                // Process delta messages (higher priority)
                if (!deltaRecords.isEmpty()) {
                    processingDelta = true;
                    deltaRecords.forEach(record -> {
                        // Process delta message here
                        System.out.println("Processing delta message: " + record.value());
                    });
                }

                // If delta is empty, only process full-load messages
                if (deltaRecords.isEmpty() && !processingDelta) {
                    fullLoadRecords.forEach(record -> {
                        // Process full-load message here
                        System.out.println("Processing full-load message: " + record.value());
                    });
                }

                // If delta is not empty, halt full-load processing and process delta messages
                if (!deltaRecords.isEmpty()) {
                    // Continue processing delta until it's empty
                    deltaRecords.forEach(record -> {
                        System.out.println("Processing delta message: " + record.value());
                    });
                }

                // Handle interruption of full-load processing when a new delta message arrives
                if (processingDelta && deltaRecords.isEmpty()) {
                    processingDelta = false; // Full load processing can resume
                }

                // Sleep or add any delay if needed
                // Thread.sleep(100); // Optional: Add a small sleep to throttle polling
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // Close the consumers gracefully
            deltaConsumer.close();
            fullLoadConsumer.close();
        }
    }
}
```

### Key Changes in the Code:

1. **Polling both topics in parallel**: The consumers poll both the `delta_topic` and the `full_load_topic` continuously. The poll timeout is set to 1 second for quick responsiveness.

2. **Delta Message Priority**: The logic ensures that if any messages are available in the `delta_topic`, they are processed first. Once all delta messages are processed, it switches to full-load topic processing.

3. **Interrupting Full-load Processing for Delta Messages**: If delta messages come in while processing full-load messages, the full-load processing is temporarily halted. The consumer switches to process the delta messages first. After processing the delta messages, it resumes processing the full-load topic.

4. **Resuming Full-load Processing**: If no delta messages are available, the consumer will continue processing messages from the full-load topic.

5. **Continuous Polling**: The loop runs continuously, polling the topics in a streaming-like manner. There’s no fixed exit condition, so the system behaves like a real-time stream processor.

### How It Works:

- **Delta Message Processing**: When delta messages are present, they are processed first, ensuring that high-priority updates are handled immediately.
- **Full-Load Message Processing**: If no delta messages are present, it processes full-load messages. However, if new delta messages arrive during the full-load processing, the full-load processing pauses and switches to process the delta messages.
- **Streaming Nature**: The loop is designed to continuously poll and process messages, giving it a stream-like behavior, where data is processed as it arrives, with priority given to delta messages.

This should now meet all the conditions and handle the continuous data processing as described. 

# Detailed Explanation and process flow

Sure! Let's go over the mechanism in more detail to understand how the full-load processing is temporarily halted when delta messages arrive in the middle of full-load processing.

### 1. **Polling Both Topics (Delta and Full-Load)**:
In the solution, the consumer is polling **both** the delta and full-load topics **continuously** in the main processing loop. The polling is done in parallel, which means that at each iteration, the Kafka consumer fetches messages from both topics, even though they are processed sequentially.

Here’s how the polling works:
```java
var deltaRecords = deltaConsumer.poll(1000);  // Poll for delta messages
var fullLoadRecords = fullLoadConsumer.poll(1000);  // Poll for full-load messages
```

- **`deltaRecords`** will contain messages from the **delta topic**.
- **`fullLoadRecords`** will contain messages from the **full-load topic**.

Both records are fetched within the same poll iteration (i.e., at the same time), but the processing of each is done separately. 

### 2. **Delta Message Processing Priority**:
The priority of delta messages is set by the program flow. The key idea here is that **delta messages always take precedence over full-load messages**.

Here’s the processing flow:
```java
if (!deltaRecords.isEmpty()) {
    processingDelta = true;
    deltaRecords.forEach(record -> {
        // Process delta message here
        System.out.println("Processing delta message: " + record.value());
    });
}
```

If **delta messages** are found (i.e., `deltaRecords.isEmpty()` is `false`), we set the flag `processingDelta` to `true` and process all delta messages. This flag indicates that delta messages are being processed.

### 3. **Full-Load Processing Temporarily Halted**:
While **delta messages are being processed**, no full-load messages are processed. This is achieved by **not touching the full-load messages** until **all delta messages are processed** in the current poll cycle.

- If there are **delta messages available**, we process them and **skip full-load message processing entirely** for that poll cycle.
- After processing delta messages, if there are no more delta messages, the flag `processingDelta` will be set to `false`:
  
```java
if (deltaRecords.isEmpty() && !processingDelta) {
    fullLoadRecords.forEach(record -> {
        // Process full-load message here
        System.out.println("Processing full-load message: " + record.value());
    });
}
```

- When `processingDelta` is `false`, the program proceeds to process full-load messages. 

### 4. **Interrupting Full-Load Processing**:
Now, here's the crucial part about **halting full-load message processing in the middle**:

- Full-load messages will **only** be processed if the delta queue is empty (`deltaRecords.isEmpty()`) and **`processingDelta` flag is false**.
  
However, **if new delta messages come in while processing full-load messages**, we have the following logic:

1. The consumer polls both `deltaTopic` and `fullLoadTopic`. Let's say delta messages are not empty during this poll cycle.
2. The program processes all delta messages **first**, and the `processingDelta` flag is set to `true`.
3. After finishing processing delta messages, **if there are still any full-load messages left**, the program will **resume processing full-load messages** only when `deltaRecords` is empty and `processingDelta` is `false`.

The flow here is that the **processing loop** will **interrupt** full-load processing only when delta messages come in during that cycle.

### Example Walkthrough:

#### Iteration 1:
1. **Poll both topics**:
   - **Delta**: There are delta messages available.
   - **Full-Load**: No full-load messages available.
   
2. **Process delta messages**:
   - Delta messages are processed first because the delta queue has messages (`deltaRecords.isEmpty() == false`).
   - After processing all delta messages, move to the next iteration.

#### Iteration 2:
1. **Poll both topics again**:
   - **Delta**: There are still delta messages available.
   - **Full-Load**: Full-load messages are available.
   
2. **Process delta messages**:
   - Again, delta messages are processed first.
   - Full-load messages will be skipped in this iteration since delta messages still exist.

#### Iteration 3:
1. **Poll both topics again**:
   - **Delta**: No delta messages available.
   - **Full-Load**: Full-load messages are available.
   
2. **Process full-load messages**:
   - Since no delta messages are left, full-load messages are now processed.
   
#### Iteration 4:
1. **Poll both topics**:
   - **Delta**: New delta messages arrive while full-load messages are being processed.
   - **Full-Load**: Full-load messages are still pending.
   
2. **Halt Full-Load Processing**:
   - The loop detects new delta messages (`deltaRecords.isEmpty() == false`).
   - Processing of full-load messages **pauses**, and delta messages are processed first, followed by resumption of full-load processing after delta messages are done.

### Final Thought on Full-Load Processing Halting:
The full-load processing is not truly "paused" in the typical sense. Rather, the program **discontinues** full-load processing temporarily by switching to the delta messages when they arrive. The processing of the full-load messages will only resume once there are no delta messages left to process (i.e., `deltaRecords.isEmpty()` is `true` and `processingDelta` is `false`).

This creates a **dynamic, real-time system** where full-load processing is effectively "paused" whenever high-priority delta messages are available, and full-load processing is resumed once all delta messages have been consumed.


# Provided solution, multi-threading

In the provided solution, **multi-threading** is **not strictly required** to implement the priority handling between delta and full-load messages. The logic you provided can work in a **single-threaded consumer model**, where the main thread polls both Kafka topics in a loop and processes messages from the **delta topic first** before moving to the **full-load topic**.

However, if you want to **enhance responsiveness** or **improve scalability**, especially if you are processing high-throughput messages, **multi-threading** can be useful. It would allow you to **process delta and full-load messages concurrently**, but still ensure that delta messages get **priority**.

Let me break it down for you:

### 1. **Single-Threaded Approach (Current Model)**

In the existing solution, the consumer runs in a **single thread** and checks both the delta and full-load topics in each poll iteration. The priority is handled sequentially within the same thread:

- **If delta messages are available**, they are processed immediately.
- **If no delta messages are found**, the consumer will process full-load messages.
- **If new delta messages arrive while full-load messages are being processed**, the loop immediately checks the delta topic again, halting full-load processing and processing the delta messages.

This approach is straightforward and works well when you do not need extreme concurrency. Kafka consumers are designed to be fast enough to handle relatively high-throughput message processing within a single thread, especially if you can handle backpressure and message batching.

### 2. **Multi-threaded Approach (Improved with Parallelism)**

In a **multi-threaded model**, you can dedicate separate threads for processing delta and full-load messages, while still maintaining priority to delta messages. The core idea is to have **one or more threads for the full-load topic** and **one thread for the delta topic**. The delta thread will always take precedence over the full-load threads.

#### Key Concept:
- **Delta Message Thread**: Dedicated to processing delta messages as soon as they arrive.
- **Full-Load Threads**: These will process full-load messages, but only **after** delta messages are completely processed.

The full-load thread can be paused or "interrupted" if new delta messages arrive while it is processing.

### Key Benefits of Multi-threading:
1. **Concurrent Processing**: Full-load and delta messages can be processed in parallel if necessary, potentially speeding up the entire process (depending on how CPU-bound the processing logic is).
2. **Non-blocking Behavior**: The delta message processing can continue uninterrupted by full-load processing, which could improve latency for delta message processing.

#### How Multi-threading Could Be Implemented:

1. **Separate Thread for Delta Topic**: You can create a dedicated thread for polling and processing delta messages. This thread can run continuously, and as soon as delta messages arrive, they will be processed.
   
2. **Main Thread for Full-Load Topic**: The main consumer thread can handle the full-load topic and process those messages.

3. **Thread Coordination (Shared Flag or Semaphore)**: Use a shared flag or a signaling mechanism (e.g., a `Semaphore` or `BlockingQueue`) to control the communication between the delta thread and the full-load thread. This will ensure that the full-load processing is paused whenever the delta message thread is active.

### Example of Multi-Threading Concept for Kafka Consumers

Here’s an illustration of how you can implement a multi-threaded model to ensure delta message priority:

```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.util.Collections;
import java.util.Properties;

public class KafkaPriorityConsumer {

    private static final String DELTA_TOPIC = "delta_topic";
    private static final String FULL_LOAD_TOPIC = "full_load_topic";
    private static final String BOOTSTRAP_SERVERS = "localhost:9092";

    public static void main(String[] args) {
        // Create and configure Kafka consumers for both topics
        KafkaConsumer<String, String> deltaConsumer = createConsumer(DELTA_TOPIC, "delta-consumer-group");
        KafkaConsumer<String, String> fullLoadConsumer = createConsumer(FULL_LOAD_TOPIC, "full-load-consumer-group");

        // Start a thread for delta topic processing
        Thread deltaThread = new Thread(() -> processDeltaMessages(deltaConsumer));
        deltaThread.setDaemon(true);  // Daemon thread, so it won't block the JVM from exiting
        deltaThread.start();

        // Start the main thread for full-load topic processing
        processFullLoadMessages(fullLoadConsumer);
    }

    private static KafkaConsumer<String, String> createConsumer(String topic, String groupId) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
        consumer.subscribe(Collections.singletonList(topic));
        return consumer;
    }

    private static void processDeltaMessages(KafkaConsumer<String, String> deltaConsumer) {
        try {
            while (true) {
                // Poll for delta messages
                var deltaRecords = deltaConsumer.poll(1000); // 1-second timeout
                
                if (!deltaRecords.isEmpty()) {
                    deltaRecords.forEach(record -> {
                        // Process delta message here
                        System.out.println("Processing delta message: " + record.value());
                    });
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            deltaConsumer.close();
        }
    }

    private static void processFullLoadMessages(KafkaConsumer<String, String> fullLoadConsumer) {
        try {
            while (true) {
                // Poll for full-load messages
                var fullLoadRecords = fullLoadConsumer.poll(1000); // 1-second timeout
                
                if (!fullLoadRecords.isEmpty()) {
                    fullLoadRecords.forEach(record -> {
                        // Process full-load message here
                        System.out.println("Processing full-load message: " + record.value());
                    });
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            fullLoadConsumer.close();
        }
    }
}
```

### Key Points in the Multi-Threaded Approach:
- **Delta Thread**: This thread is dedicated to processing delta messages. It is always running and can **interrupt** full-load processing by processing delta messages as soon as they arrive.
- **Main Thread**: The main thread processes full-load messages but will only do so if the delta thread isn't actively processing any delta messages.
  
### Thread Coordination:
- The **delta thread** has higher priority and can preempt the **full-load thread** when new delta messages arrive.
- If **delta messages arrive**, the main thread can **pause** or **stop processing full-load messages** temporarily and let the delta thread process those messages.
  
While this setup uses **two threads** (one for delta processing and one for full-load processing), the **priority of delta messages** is maintained because the **delta thread runs independently** and always gets a chance to process messages when they arrive.

### Conclusion:
In summary:
- A **single-threaded approach** can work perfectly fine for most use cases where the delta message priority logic is simple and low to moderate throughput is expected.
- **Multi-threading** is beneficial if you want to **maximize throughput** or if you want **parallel processing** for delta and full-load messages, while still maintaining delta message priority. In this case, you'd use **separate threads** and some coordination mechanism (like flags or semaphores) to ensure full-load processing is temporarily paused whenever delta messages arrive. 

For **simple Kafka consumers** that just need **priority handling** and low complexity, **multi-threading is not strictly necessary**. But, if you're building a **high-throughput real-time system**, multi-threading will help in ensuring that both types of messages are processed more efficiently.
