# Celery и Celery Beat — Асинхронные фоновые задачи

Celery — это распределённая очередь задач, которая позволяет запускать фоновые задачи в приложениях. Он широко используется для выполнения асинхронной и периодической обработки, особенно в системах с микросервисной архитектурой.



## Зачем использовать Celery?

- **Асинхронность** — выполнение тяжёлых операций (например, отправка email, интеграция с API) вне основного потока приложения
- **Масштабируемость** — добавление воркеров позволяет обрабатывать больше задач
- **Повторяемость** — задачи можно перезапустить при сбоях
- **Мониторинг** — можно отслеживать состояние задач (с помощью Flower и других инструментов)
- **Гибкость** — поддержка различных брокеров (RabbitMQ, Redis, SQS)



## Архитектура Celery

```
[App] → [Broker] → [Worker] → [Result Backend]
```

- **App** — основное приложение, которое отправляет задачи
- **Broker** — очередь задач (например, Redis, RabbitMQ)
- **Worker** — процесс, исполняющий задачи
- **Result Backend** — хранилище результатов выполнения задач (например, Redis, PostgreSQL)



## Установка

```bash
pip install celery[redis]
```



## Минимальный пример

```python
# tasks.py
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task
def add(x, y):
    return x + y
```

Запуск воркера:

```bash
celery -A tasks worker --loglevel=info
```

Вызов задачи из Python:

```python
from tasks import add

result = add.delay(4, 4)
print(result.get())  # 8
```



## Celery Beat — планировщик задач

Celery Beat позволяет выполнять задачи по расписанию (например, каждый час или каждый день в 8:00).

### Установка

```bash
pip install django-celery-beat  # или просто использовать beat с celery
```

### Пример `beat_schedule`

```python
from celery.schedules import crontab

app.conf.beat_schedule = {
    'send-report-every-morning': {
        'task': 'tasks.send_report',
        'schedule': crontab(hour=8, minute=0),
    },
}
```

Запуск Beat:

```bash
celery -A tasks beat --loglevel=info
```

Можно использовать `celery -A tasks worker -B`, но это не рекомендуется для продакшена.



## Примеры задач

```python
@app.task(bind=True, max_retries=3)
def fetch_url(self, url):
    try:
        response = requests.get(url)
        return response.text
    except requests.RequestException as exc:
        raise self.retry(exc=exc, countdown=10)
```



## Мониторинг задач: Flower

Flower — это веб-интерфейс для мониторинга и управления задачами Celery.

```bash
pip install flower
celery -A tasks flower
```

Откроется интерфейс на `http://localhost:5555`



## Использование в микросервисах

- Микросервис может отправлять задачу в очередь, другой — её обрабатывать
- Использовать Redis/RabbitMQ как централизованный брокер
- Указывать `queue` для маршрутизации задач:

```python
@app.task(queue='emails')
def send_email(...):
    ...
```



## Общие рекомендации

- Используйте `acks_late=True` для надёжности
- Используйте `retry` для повторов
- Делите задачи на небольшие, атомарные
- Логируйте исключения
- Следите за зависимостями (например, при использовании `requests`/`boto3` внутри задач)



## Потенциальные проблемы

| Проблема                         | Решение                                  |
| -------------------------------- | ---------------------------------------- |
| Зависание воркеров               | Использовать таймауты и watchdog         |
| Потеря задач                     | Использовать надёжные брокеры (RabbitMQ) |
| Переиспользование больших данных | Передавать ID и доставать из БД          |



## Заключение

Celery — это фундаментальный инструмент для построения надёжной, масштабируемой и асинхронной архитектуры, как в рамках одного сервиса, так и между несколькими микросервисами.
