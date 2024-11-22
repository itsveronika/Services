# Сервис, который проводит транзакцию и сохраняет версионность

## Импортируем библиотеки
```
import json
import uuid
from kafka import KafkaProducer, KafkaConsumer
import redis
```

## Настраиваем Kafka и Redis
```
KAFKA_BOOTSTRAP_SERVERS = "localhost:9092" 
KAFKA_TOPIC_TRANSFERS = "money_transfers"

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0
```

## Инициализация Kafka Producer
```
producer = KafkaProducer(
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
```

## Инициализация Redis
```
redis_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, db=REDIS_DB)
```

## Функция initiate_transfer(sender_id, receiver_id, amount) - отвечает за новый перевод денег, сохраняет данные в Redis, используя `transfer_id` в качестве ключа, отправляет сообщение с данными о переводе и возвращает transfer_id
```
def initiate_transfer(sender_id, receiver_id, amount):
    """Инициализирует перевод денег."""
    transfer_id = str(uuid.uuid4())
    transfer_data = {
        "transfer_id": transfer_id,
        "sender_id": sender_id,
        "receiver_id": receiver_id,
        "amount": amount,
        "status": "initiated",
        "version": 1
    }

    redis_client.set(f"transfer:{transfer_id}", json.dumps(transfer_data))

    producer.send(KAFKA_TOPIC_TRANSFERS, value=transfer_data)
    print(f"Transfer initiated: {transfer_data}")
    return transfer_id
```

## Функция обрабатывает данные о переводе, полученные из Kafka. Сравнивает полученную version с версией в Redis. Если версии совпадают, то функция увеличивает version, изменяет status, сохраняет обновленные данные обратно в Redis
```
def process_transfer(transfer_data):
    transfer_id = transfer_data["transfer_id"]
    version = transfer_data["version"]

    existing_data_str = redis_client.get(f"transfer:{transfer_id}")
    if existing_data_str:
        existing_data = json.loads(existing_data_str)
        if existing_data["version"] != version:
            print(f"Conflict detected for transfer {transfer_id}. Version mismatch.")
            return

    transfer_data["version"] += 1
    transfer_data["status"] = "completed" # Симулируем успешное завершение

    redis_client.set(f"transfer:{transfer_id}", json.dumps(transfer_data))
    print(f"Transfer completed: {transfer_data}")
