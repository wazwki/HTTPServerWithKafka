# **How to Set Up Apache Kafka in a Golang HTTP Server**

# 1. Overview of the Sarama Library for Apache Kafka in Go

`Sarama` is a popular Go library for working with Apache Kafka. It provides a full set of features for interacting with Kafka, including creating producers, consumers, consumer groups, and managing a Kafka cluster. In this overview, we will go over Sarama's main features and components and show how to use them.

## 1. Installing Sarama

First, install the library:

```bash
go get github.com/IBM/sarama
```

```go
import (
    "github.com/IBM/sarama"
)
```

## 2. Logging

`Sarama` supports built-in logging, which can be configured as follows:

```go
sarama.Logger = log.New(os.Stdout, "[Sarama] ", log.LstdFlags)
```

# 2. Main Components of Sarama

## 1. Configuration

Configuration is a key part of working with Sarama. It allows you to set the behavior of the producer, consumer, and other components.

```go
config := sarama.NewConfig()
config.Version = sarama.V2_6_0_0 // Specify the Kafka protocol version
config.ClientID = "example-client" // Set the client ID
```

## 2. Producer Configuration

Key parameters:

- `Producer.RequiredAcks`: The level of acknowledgment for sent messages. Can be:
    - `NoResponse`: The producer does not expect an acknowledgment.
    - `WaitForLocal`: Only leader partition acknowledgment is required.
    - `WaitForAll`: Acknowledgment is required from all replicas.
- `Producer.Retry.Max`: The number of retry attempts for sending a message in case of failure.
- `Producer.Return.Successes`: If `true`, the producer will return successful sends.

## 3. Consumer Configuration

Key parameters:

- `Consumer.Offsets.Initial`: Specifies where to start reading if the offset is not saved. Can be `OffsetNewest` or `OffsetOldest`.
- `Consumer.Group.Rebalance.Strategy`: Defines the strategy for rebalancing consumer groups. Main strategies:
    - `BalanceStrategyRange`: Distributes partitions by range.
    - `BalanceStrategyRoundRobin`: Distributes partitions evenly among consumers.

# 3. Working with a Producer

A producer is a component that sends messages to Kafka.

## 1. Example of Using a Synchronous Producer

A synchronous producer sends a message and waits for confirmation of the send:

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

    log.Printf("Message sent to partition %d with offset %d\\n", partition, offset)
}
```

## 2. Asynchronous Producer

An asynchronous producer sends messages without waiting for confirmation, increasing throughput:

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
                log.Printf("Message sent to partition %d with offset %d\\n", success.Partition, success.Offset)
            }
        }
    }()

    msg := &sarama.ProducerMessage{
        Topic: "example-topic",
        Value: sarama.StringEncoder("Asynchronous message"),
    }

    producer.Input() <- msg

    // Wait before program exit to ensure messages are sent
    select {}
}
```

# 4. Working with a Consumer

A consumer is a component that reads messages from Kafka.

## 1. Example of Using a Consumer

Simple message reading from a partition:

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
        log.Printf("Received message: %s\\n", string(message.Value))
    }
}
```

## 2. Consumer Groups

Consumer groups allow messages to be processed in parallel by multiple consumer instances, distributing the load among them.

### Example of Using a Consumer Group

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
                log.Fatalf("Failed to consume: %v", err)
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
        log.Printf("Received message: %s\\n", string(message.Value))
        session.MarkMessage(message, "")
    }
    return nil
}
```

# 5. Kafka Management

Besides creating producers and consumers, `Sarama` also provides an API for managing Kafka clusters, such as creating and deleting topics, modifying configurations, and more. For this, the `ClusterAdmin` is used.

### Example of Using ClusterAdmin

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