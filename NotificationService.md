# Cервис уведомлений для платежной системы

## Импортируем библиотеки
```
import json
from kafka import KafkaConsumer
import redis
import time
import uuid
```

## Настраиваем Kafka и Redis
```
KAFKA_BOOTSTRAP_SERVERS = "localhost:9092" # Замените на ваши настройки
KAFKA_TOPIC_TRANSACTIONS = "payment_transactions"

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0
```

## Инициализацируем Redis и Kafka Consumer
```
redis_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, db=REDIS_DB)

consumer = KafkaConsumer(
    KAFKA_TOPIC_TRANSACTIONS,
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    auto_offset_reset='earliest',
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)
```

## Функция - сохраняет информацию об уведомлении в Redis. А с помощью библиотеки uuid генерируется уникальный идентификатор для уведомления 
```
def send_notification(user_id, message):
    notification_id = str(uuid.uuid4())

    notification = {
        'notification_id': notification_id,
        'user_id': user_id,
        'message': message,
        'status': 'sent'
    }

    redis_client.rpush(f'notifications:{user_id}', json.dumps(notification))
    print(f"Notification sent: {notification}")
```

## Функция - обрабатывает данные о транзакции и вызывает send_notification для отправки уведомления пользователю
```
def process_transaction(transaction):
    try:
        user_id = transaction['user_id']
        status = transaction['status']
        transaction_id = transaction['transaction_id']
        message = f"Transaction {transaction_id} is {status}"
        send_notification(user_id, message)

    except KeyError as e:
        print(f"Error processing transaction: Missing key {e}")

    except Exception as e:
        print(f"Error processing transaction: {e}")
```

## В данном цикле обрабатываются полученные сообщения и вызывается функция process_transaction для обработки данных транзакции
```
if __name__ == "__main__":
    print("Notification service started")

    for message in consumer:
        transaction = message.value
        process_transaction(transaction)
```
