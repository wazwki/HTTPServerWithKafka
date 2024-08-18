# How to config Apache Kafka in golang HTTP-server


### Detailed Overview of the Sarama Library for Apache Kafka in Go

`Sarama` is a popular Go library for working with Apache Kafka. It provides a full suite of functionalities for interacting with Kafka, including creating producers, consumers, consumer groups, and tools for managing the Kafka cluster. In this overview, we will dive deep into the main features and components of `Sarama` and demonstrate how to use them.

### Installing Sarama

To get started, install the library with the following command:

```bash
go get github.com/IBM/sarama
```
```go
import (
    "github.com/IBM/sarama"
)
```

### Logging

`Sarama` supports built-in logging, which can be configured as follows:

```go
sarama.Logger = log.New(os.Stdout, "[Sarama] ", log.LstdFlags)
```

### Key Components of Sarama

#### 1. Configuration

Configuration is a crucial part of working with Sarama. It allows you to set up the behavior of the producer, consumer, and other components.

```go
config := sarama.NewConfig()
config.Version = sarama.V2_6_0_0 // Specify Kafka protocol version
config.ClientID = "example-client" // Set client ID
```

##### Producer Configuration

Key parameters:

- `Producer.RequiredAcks`: The level of acknowledgment required for sending messages. It can be:
  - `NoResponse`: The producer does not wait for acknowledgment.
  - `WaitForLocal`: Waits for acknowledgment only from the partition leader.
  - `WaitForAll`: Waits for acknowledgment from all replicas.
  
- `Producer.Retry.Max`: The maximum number of retry attempts in case of an error.
- `Producer.Return.Successes`: If `true`, the producer will return successful sends.

##### Consumer Configuration

Key parameters:

- `Consumer.Offsets.Initial`: Indicates where to start reading if the offset is not stored yet. It can be `OffsetNewest` or `OffsetOldest`.
- `Consumer.Group.Rebalance.Strategy`: Determines the strategy for rebalancing consumer groups. Main strategies:
  - `BalanceStrategyRange`: Distributes partitions by range.
  - `BalanceStrategyRoundRobin`: Evenly distributes partitions across consumers.

### Working with Producers

A producer is a component that sends messages to Kafka.

#### Example of a Synchronous Producer

A synchronous producer sends a message and waits for the acknowledgment:

```go
package main

import (
    "log"
    "github.com/Shopify/sarama"
)

func main() {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll
    config.Producer.Retry.Max = 5
    config.Producer.Return.Successes = true

    producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }
    defer producer.Close()

    msg := &sarama.ProducerMessage{
        Topic: "example-topic",
        Value: sarama.StringEncoder("Hello, Kafka!"),
    }

    partition, offset, err := producer.SendMessage(msg)
    if err != nil {
        log.Fatalf("Failed to send message: %v", err)
    }

    log.Printf("Message sent to partition %d with offset %d\n", partition, offset)
}
```

#### Asynchronous Producer

An asynchronous producer allows you to send messages without waiting for acknowledgment, increasing throughput:

```go
package main

import (
    "log"
    "github.com/Shopify/sarama"
)

func main() {
    config := sarama.NewConfig()
    config.Producer.RequiredAcks = sarama.WaitForAll
    config.Producer.Retry.Max = 5

    producer, err := sarama.NewAsyncProducer([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatalf("Failed to create producer: %v", err)
    }
    defer producer.Close()

    go func() {
        for {
            select {
            case err := <-producer.Errors():
                log.Printf("Failed to send message: %v", err)
            case success := <-producer.Successes():
                log.Printf("Message sent to partition %d with offset %d\n", success.Partition, success.Offset)
            }
        }
    }()

    msg := &sarama.ProducerMessage{
        Topic: "example-topic",
        Value: sarama.StringEncoder("Asynchronous message"),
    }

    producer.Input() <- msg

    // Wait before exiting to ensure messages are sent
    select {}
}
```

### Working with Consumers

A consumer is a component that reads messages from Kafka.

#### Example of a Consumer

Simple message consumption from a partition:

```go
package main

import (
    "log"
    "github.com/Shopify/sarama"
)

func main() {
    config := sarama.NewConfig()
    config.Consumer.Offsets.Initial = sarama.OffsetNewest

    consumer, err := sarama.NewConsumer([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatalf("Failed to create consumer: %v", err)
    }
    defer consumer.Close()

    partitionConsumer, err := consumer.ConsumePartition("example-topic", 0, sarama.OffsetNewest)
    if err != nil {
        log.Fatalf("Failed to subscribe to partition: %v", err)
    }
    defer partitionConsumer.Close()

    for message := range partitionConsumer.Messages() {
        log.Printf("Received message: %s\n", string(message.Value))
    }
}
```

### Consumer Groups

Consumer groups allow you to process messages in parallel with multiple consumer instances, distributing the load among them.

#### Example of a Consumer Group

```go
package main

import (
    "context"
    "log"
    "os"
    "os/signal"
    "github.com/Shopify/sarama"
)

func main() {
    config := sarama.NewConfig()
    config.Version = sarama.V2_1_0_0
    config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
    config.Consumer.Offsets.Initial = sarama.OffsetNewest

    consumerGroup, err := sarama.NewConsumerGroup([]string{"localhost:9092"}, "example-group", config)
    if err != nil {
        log.Fatalf("Failed to create consumer group: %v", err)
    }
    defer consumerGroup.Close()

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go func() {
        for {
            if err := consumerGroup.Consume(ctx, []string{"example-topic"}, &consumerGroupHandler{}); err != nil {
                log.Fatalf("Error in consumption: %v", err)
            }
            if ctx.Err() != nil {
                return
            }
        }
    }()

    sigterm := make(chan os.Signal, 1)
    signal.Notify(sigterm, os.Interrupt)
    <-sigterm
}

type consumerGroupHandler struct{}

func (h *consumerGroupHandler) Setup(sarama.ConsumerGroupSession) error {
    return nil
}

func (h *consumerGroupHandler) Cleanup(sarama.ConsumerGroupSession) error {
    return nil
}

func (h *consumerGroupHandler) ConsumeClaim(session sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for message := range claim.Messages() {
        log.Printf("Received message: %s\n", string(message.Value))
        session.MarkMessage(message, "")
    }
    return nil
}
```

### Kafka Management

In addition to creating producers and consumers, `Sarama` also provides an API for managing Kafka clusters, such as creating and deleting topics, altering configurations, and more. This is done using the `ClusterAdmin`.

#### Example of using ClusterAdmin

```go
package main

import (
    "log"
    "github.com/Shopify/sarama"
)

func main() {
    config := sarama.NewConfig()
    config.Version = sarama.V2_6_0_0

    admin, err := sarama.NewClusterAdmin([]string{"localhost:9092"}, config)
    if err != nil {
        log.Fatalf("Failed to create ClusterAdmin: %v", err)
    }
    defer admin.Close()

    err = admin.CreateTopic("new-topic", &sarama.TopicDetail{
        NumPartitions:     1,
        ReplicationFactor: 1,
    }, false)
    if err != nil {
        log.Fatalf("Failed to create topic: %v", err)
    }

    log.Println("Topic successfully created")
}
```

### Conclusion

`Sarama` is a powerful and flexible library for working with Apache Kafka in Go. It supports a wide range of features, from creating simple producers and consumers to managing Kafka clusters. With `Sarama`, you can build high-performance, scalable applications that process large volumes of data in real time.