# Обработка денежных переводов/платежей

## Импортируем необходимые библиотеки
```
import json
import time
import redis
from kafka import KafkaProducer
from kafka.errors import KafkaError
```

## Задаем конфигурационные параметры для подключения к брокеру сообщений Kafka и базе данных Redis, которые настраивают подключение к внешним сервисам, необходимым для работы приложения 
```
KAFKA_BOOTSTRAP_SERVERS = 'localhost: 9092' 
REDIS_HOST = 'localhost'
REDIS_PORT = 6379
```

## Настраиваем Kafka Producer
```
producer = KafkaProducer(bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
                         value_serializer=lambda v: json.dumps(v).encode('utf-8'))
```

## Настраиваем Redis client
```
redis_client = redis.StrictRedis(host=REDIS_HOST, port=REDIS_PORT, db=0)
```

## Создаем основную функция обработки платежа, которая проверяет, содержит ли payment_data ключи sender_id, recipient_id и amount. Добавляет JSON-представление payment_data в список payment_queue в Redis. Обрабатывет ошибки, выводя сообщение об ошибке и возвращая False 
```
def handle_payment(payment_data):
    try:
        if not all(key in payment_data for key in ['sender_id', 'recipient_id', 'amount']):
            print("Недостаточно данных платежа!")
            return False

        redis_client.rpush("payment_queue", json.dumps(payment_data))

        producer.send('payment_requests', payment_data)
        print(f"Платеж успешно отправлен: {payment_data}")
        return True

    except Exception as e:
        print(f"Ошибка при обработке платежа: {e}")
        return False
```

## Функция main() - создает пример данных платежа payment_data, вызывает handle_payment() для обработки этих данных, выводит сообщение об успешной обработке
```
def main():

    payment_data = {
        'sender_id': 'user123',
        'recipient_id': 'user456',
        'amount': 100,
        'currency': 'USD'
    }

    if handle_payment(payment_data):
      print("Платеж обработан и отправлен в Kafka")
```

## Определяем, как был запущен скрипт,как основная программа или как модуль в другой скрипт 
```
if __name__ == "__main__":
    main()
```
