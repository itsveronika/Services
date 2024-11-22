# Конвертация валют по курсу

## Импортируем библиотеки
```
import json
import time
from kafka import KafkaConsumer, KafkaProducer
import redis
```

## Настраиваем Kafka и Redis
```
KAFKA_BOOTSTRAP_SERVERS = "localhost:9092"
KAFKA_TOPIC_EXCHANGE = "currency_exchange_rates"

REDIS_HOST = "localhost"
REDIS_PORT = 6379
REDIS_DB = 0
```

## Данная функция получает курсы валют
```
def get_exchange_rates_from_external_api(api_url):

    try:
        response = requests.get(api_url)
        response.raise_for_status() 
        data = response.json()
        return data
    except requests.exceptions.RequestException as e:
        print(f"Ошибка при получении курсов валют: {e}")
        return None
    except json.JSONDecodeError as e:
        print(f"Ошибка десериализации JSON: {e}")
        return None
```

## Инициализируем Kafka Consumer и Producer
```
consumer = KafkaConsumer(
    KAFKA_TOPIC_EXCHANGE,
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    auto_offset_reset="earliest", # Начать с самого начала
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

producer = KafkaProducer(
    bootstrap_servers=KAFKA_BOOTSTRAP_SERVERS,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)
```

## Инициализируем Redis
```
redis_client = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, db=REDIS_DB)
```

## Данная функция вызывает get_exchange_rates_from_icron() для получения курсов
```
def update_exchange_rates():
    rates = get_exchange_rates_from_icron()
    redis_client.set("exchange_rates", json.dumps(rates))
    print(f"Exchange rates updated: {rates}")
    producer.send(KAFKA_TOPIC_EXCHANGE, value={'rates': rates, 'timestamp': time.time()})
```

## Данная функция получает курсы валют из Redis. Если курсы не найдены, то возвращает None. А также проверяет, существуют ли указанные валюты в полученном словаре курсов и вычисляет и возвращает конвертированную сумму
```
def convert_currency(amount, from_currency, to_currency):
    rates_str = redis_client.get("exchange_rates")
    if rates_str is None:
        return None # Курсы не загружены
    rates = json.loads(rates_str.decode('utf-8'))
    if from_currency not in rates or to_currency not in rates:
        return None # Неизвестная валюта

    return amount * (rates[to_currency] / rates[from_currency])
```

## Данный отрывок кода отвечает за то, чтобы обновления курсов запускались каждые 4 часа
```
if __name__ == "__main__":
    update_exchange_rates() 
    while True:
        time.sleep(4 * 60 * 60)
        update_exchange_rates()
```
