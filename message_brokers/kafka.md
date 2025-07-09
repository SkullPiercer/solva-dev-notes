# Apache Kafka и взаимодействие микросервисов

Apache Kafka — это распределённая платформа потоковой передачи данных (streaming platform), которая используется для построения высокопроизводительных, масштабируемых и устойчивых систем.



## Зачем нужен Kafka?

- **Высокая производительность**: может обрабатывать миллионы сообщений в секунду
- **Хранение истории сообщений**: в отличие от RabbitMQ, Kafka сохраняет все сообщения на диске
- **Масштабируемость**: легко масштабируется горизонтально (partition-based)
- **Надёжность**: устойчив к сбоям, репликация сообщений
- **Подходит для stream processing**: данные можно обрабатывать в реальном времени



## Основные концепции Kafka

| Компонент          | Описание                                                           |
| ------------------ | ------------------------------------------------------------------ |
| **Producer**       | Отправляет сообщения в Kafka                                       |
| **Consumer**       | Читает сообщения из Kafka                                          |
| **Topic**          | Категория или имя потока сообщений                                 |
| **Partition**      | Раздел внутри топика, для масштабируемости                         |
| **Broker**         | Узел Kafka, обрабатывающий данные                                  |
| **Consumer Group** | Группа потребителей, совместно читающих сообщения из одного топика |



## Пример архитектуры

```
   [Producer A] --->

                   [Topic: orders] ---> [Consumer Group 1: analytics-service]
                  /
   [Producer B] --->

                         ---> [Consumer Group 2: email-service]
```



## Пример продюсера на Python (с использованием kafka-python)

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

producer.send('orders', {'order_id': 123, 'status': 'created'})
producer.flush()
```



## Пример консюмера

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers='localhost:9092',
    auto_offset_reset='earliest',
    group_id='analytics-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    print(f"Получено сообщение: {message.value}")
```



## Преимущества Kafka

- Хранение истории (log-based)
- Распределённая архитектура
- Поддержка масштабирования и отказоустойчивости
- Высокая производительность (миллионы сообщений в секунду)
- Экосистема: Kafka Streams, Kafka Connect, ksqlDB



## Недостатки

- Сложнее в развёртывании и поддержке
- Не всегда нужен, если хватает обычной очереди
- Требует отдельного подхода к обработке ошибок и повторной доставке



## Kafka vs RabbitMQ

| Характеристика     | Kafka                                    | RabbitMQ                        |
| ------------------ | ---------------------------------------- | ------------------------------- |
| Хранение сообщений | Да, на диск, с возможностью чтения позже | Нет (по умолчанию)              |
| Подход             | Streaming (log)                          | Queue-based (AMQP)              |
| Скорость обработки | Очень высокая                            | Высокая                         |
| Поддержка повторов | Да (offsets)                             | Да (acknowledgement + requeue)  |
| Типичные случаи    | Аналитика, стриминг, аудит               | Фоновая обработка задач, Celery |
