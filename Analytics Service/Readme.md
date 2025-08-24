
---

# **1. Источники данных**

Analytics Service получает данные **асинхронно через брокер сообщений (RabbitMQ/Kafka)**:

|Источник|Тип данных|Куда отправляется в Analytics|
|---|---|---|
|GraphQL Gateway|События пользователей (поиск, клик, фильтры)|очередь `analytics.events`|
|Aggregator|Логи обработки запросов, статусы скрапперов|очередь `analytics.events`|
|Scraper (опционально)|Метрики скорости и ошибок парсинга|очередь `analytics.events`|
|AI Router (опционально)|Метрики обработки AI (время, ошибки, категории)|очередь `analytics.events`|

**Важно:** сервис **не делает прямые запросы к другим сервисам**, он только **подписывается на сообщения и сохраняет/агрегирует их**.

---

# **2. Структура Analytics Service**

```bash
analytics_service/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py            # REST/GraphQL для получения отчётов
│   │
│   ├── core/
│   │   ├── config.py            # Настройки (DB, брокер, очереди)
│   │   ├── logger.py            # Логи событий и ошибок
│   │   └── utils.py             # Вспомогательные функции
│   │
│   ├── consumers/
│   │   └── events_consumer.py   # Подписка на очередь событий
│   │
│   ├── services/
│   │   ├── aggregator.py        # Агрегация и обработка событий
│   │   ├── storage.py           # Сохранение в БД (PostgreSQL)
│   │   └── reporting.py         # Генерация отчетов / метрик
│   │
│   ├── models/
│   │   ├── event.py             # Pydantic модели UserEvent, ScraperEvent
│   │   └── report.py            # Модели отчётов и метрик
│   │
│   └── main.py                  # Запуск FastAPI + consumer
│
├── tests/
│   ├── test_aggregator.py
│   ├── test_storage.py
│   └── test_integration.py
│
├── dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# **3. Поток данных в Analytics**

1. **Gateway / Aggregator / Scraper → очередь `analytics.events`**
    
    - Пример события от Gateway:
        

```json
{
  "user_id": "uuid-1234",
  "action": "search",
  "query": "айфон",
  "timestamp": "2025-08-23T16:05:00Z"
}
```

2. **Consumer в Analytics Service**
    
    - Слушает очередь
        
    - Валидирует событие
        
    - Сохраняет в PostgreSQL или Redis
        
3. **Aggregator / Reporting**
    
    - Считает статистику: топовые товары, количество запросов, ошибки скрапперов
        
    - REST/GraphQL эндпоинты для дашборда или админки
        

---

# **4. Мини-ER модели**

- **UserEvent**
    

```python
class UserEvent(BaseModel):
    user_id: str
    action: str  # "search", "click", "filter"
    query: Optional[str]
    timestamp: datetime
```

- **ScraperEvent**
    

```python
class ScraperEvent(BaseModel):
    request_id: str
    scraper: str
    status: str  # "done", "error"
    duration_ms: int
    timestamp: datetime
```

- **Report**
    

```python
class Report(BaseModel):
    top_queries: list[str]
    most_active_users: list[str]
    scraper_stats: dict
```

---

### 🔹 **Итог**

- Analytics Service **читает события из очередей** и **не зависит от прямых запросов к сервисам**.
    
- Сервис **агрегирует, сохраняет и предоставляет отчёты**.
    
- Структура позволяет легко масштабировать аналитический сервис независимо от остальной системы.
    

---
