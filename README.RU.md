# Как настроить Apache Kafka в golang HTTP-сервере


### Подробный обзор библиотеки Sarama для Apache Kafka на Go

`Sarama` — это популярная библиотека на Go для работы с Apache Kafka. Она предоставляет полный набор функций для взаимодействия с Kafka, включая создание производителей (Producers), потребителей (Consumers), групп потребителей (Consumer Groups), а также возможности для управления кластером Kafka. В этом обзоре мы подробно рассмотрим основные возможности и компоненты `Sarama`, а также покажем, как их использовать.

### Установка Sarama

Для начала установим библиотеку:

```bash
go get github.com/IBM/sarama
```
```go
import (
    "github.com/IBM/sarama"
)
```

### Логирование

`Sarama` поддерживает встроенное логирование, которое можно настроить следующим образом:

```go
sarama.Logger = log.New(os.Stdout, "[Sarama] ", log.LstdFlags)
```

### Основные компоненты Sarama

#### 1. Конфигурация (Configuration)

Конфигурация — это ключевая часть работы с Sarama. Она позволяет настроить поведение производителя, потребителя и других компонентов.

```go
config := sarama.NewConfig()
config.Version = sarama.V2_6_0_0 // Указание версии протокола Kafka
config.ClientID = "example-client" // Установка идентификатора клиента
```

##### Конфигурация Производителя (Producer Configuration)

Основные параметры:

- `Producer.RequiredAcks`: Уровень подтверждения отправки сообщений. Может быть:
  - `NoResponse`: Производитель не ожидает подтверждения.
  - `WaitForLocal`: Ожидается подтверждение только от лидера партиции.
  - `WaitForAll`: Ожидается подтверждение от всех реплик.
  
- `Producer.Retry.Max`: Количество попыток повторной отправки сообщения в случае ошибки.
- `Producer.Return.Successes`: Если `true`, производитель будет возвращать успешные отправки.

##### Конфигурация Потребителя (Consumer Configuration)

Основные параметры:

- `Consumer.Offsets.Initial`: Указывает, с какого оффсета начинать чтение, если он еще не сохранен. Может быть `OffsetNewest` или `OffsetOldest`.
- `Consumer.Group.Rebalance.Strategy`: Определяет стратегию ребалансировки групп потребителей. Основные стратегии:
  - `BalanceStrategyRange`: Распределяет партиции по диапазону.
  - `BalanceStrategyRoundRobin`: Распределяет партиции равномерно по потребителям.

### Работа с Производителем (Producer)

Производитель — это компонент, который отправляет сообщения в Kafka.

#### Пример использования синхронного производителя

Синхронный производитель отправляет сообщение и ожидает подтверждения отправки:

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
        log.Fatalf("Ошибка создания производителя: %v", err)
    }
    defer producer.Close()

    msg := &sarama.ProducerMessage{
        Topic: "example-topic",
        Value: sarama.StringEncoder("Привет, Kafka!"),
    }

    partition, offset, err := producer.SendMessage(msg)
    if err != nil {
        log.Fatalf("Ошибка отправки сообщения: %v", err)
    }

    log.Printf("Сообщение отправлено в партицию %d с оффсетом %d\n", partition, offset)
}
```

#### Асинхронный производитель

Асинхронный производитель позволяет отправлять сообщения без ожидания подтверждения, что увеличивает пропускную способность:

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
        log.Fatalf("Ошибка создания производителя: %v", err)
    }
    defer producer.Close()

    go func() {
        for {
            select {
            case err := <-producer.Errors():
                log.Printf("Ошибка отправки сообщения: %v", err)
            case success := <-producer.Successes():
                log.Printf("Сообщение отправлено в партицию %d с оффсетом %d\n", success.Partition, success.Offset)
            }
        }
    }()

    msg := &sarama.ProducerMessage{
        Topic: "example-topic",
        Value: sarama.StringEncoder("Асинхронное сообщение"),
    }

    producer.Input() <- msg

    // Ожидание перед завершением программы, чтобы сообщения были отправлены
    select {}
}
```

### Работа с Потребителем (Consumer)

Потребитель — это компонент, который читает сообщения из Kafka.

#### Пример использования потребителя

Простое чтение сообщений из партиции:

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
        log.Fatalf("Ошибка создания потребителя: %v", err)
    }
    defer consumer.Close()

    partitionConsumer, err := consumer.ConsumePartition("example-topic", 0, sarama.OffsetNewest)
    if err != nil {
        log.Fatalf("Ошибка подписки на партицию: %v", err)
    }
    defer partitionConsumer.Close()

    for message := range partitionConsumer.Messages() {
        log.Printf("Получено сообщение: %s\n", string(message.Value))
    }
}
```

### Группы потребителей (Consumer Groups)

Группы потребителей позволяют обрабатывать сообщения параллельно несколькими экземплярами потребителей, распределяя нагрузку между ними.

#### Пример использования группы потребителей

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
        log.Fatalf("Ошибка создания группы потребителей: %v", err)
    }
    defer consumerGroup.Close()

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go func() {
        for {
            if err := consumerGroup.Consume(ctx, []string{"example-topic"}, &consumerGroupHandler{}); err != nil {
                log.Fatalf("Ошибка потребления: %v", err)
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
        log.Printf("Получено сообщение: %s\n", string(message.Value))
        session.MarkMessage(message, "")
    }
    return nil
}
```

### Управление Kafka

Помимо создания производителей и потребителей, `Sarama` также предоставляет API для управления кластером Kafka, например, создание и удаление топиков, изменение конфигураций и многое другое. Для этого используется `ClusterAdmin`.

#### Пример использования ClusterAdmin

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
        log.Fatalf("Ошибка создания ClusterAdmin: %v", err)
    }
    defer admin.Close()

    err = admin.CreateTopic("new-topic", &sarama.TopicDetail{
        NumPartitions:     1,
        ReplicationFactor: 1,
    }, false)
    if err != nil {
        log.Fatalf("Ошибка создания топика: %v", err)
    }

    log.Println("Топик успешно создан")
}
```

### Заключение

`Sarama` — это мощная и гибкая библиотека для работы с Apache Kafka на языке Go. Она поддерживает широкий спектр функций, от создания простых производителей и потребителей до управления кластерами Kafka. С помощью `Sarama` можно создавать высокопроизводительные и масштабируемые приложения, обрабатывающие большие объемы данных в режиме реального времени.