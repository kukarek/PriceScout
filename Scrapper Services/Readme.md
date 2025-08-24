
---

# 📂 Структура Scraper Service

```bash
scraper_service/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py           # Health check, metrics (опционально)
│   │
│   ├── core/
│   │   ├── config.py           # Настройки API ключей, очередей, таймаутов
│   │   ├── logger.py           # Логи работы скраппера
│   │   └── utils.py            # Вспомогательные функции
│   │
│   ├── services/
│   │   ├── fetcher.py          # Основной модуль для получения данных с маркетплейсов
│   │   ├── parser.py           # Парсинг HTML/API ответа в модель продукта
│   │   └── result_sender.py    # Отправка результата в Aggregator через очередь
│   │
│   ├── consumers/
│   │   └── Aggregator_consumer.py  # Слушает задачи поиска с Aggregator
│   │
│   ├── models/
│   │   ├── product.py          # Pydantic модели Product, ScraperResult
│   │   └── task.py             # TaskRequest, TaskStatus
│   │
│   └── main.py                 # Точка входа: запуск FastAPI + consumer
│
├── tests/
│   ├── test_fetcher.py
│   ├── test_parser.py
│   └── test_integration.py
│
├── dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# ⚙️ Логика работы Scraper

1. **Aggregator → Scraper**
    
    - Aggregator отправляет задачу через очередь (`request_id`, `query`, `filters`)
        
2. **Consumer получает задачу**
    
    - Слушает очередь `scraper.tasks`
        
    - Определяет, какой источник (WB, Ozon, Avito) нужно обрабатывать
        
3. **Fetcher → парсинг**
    
    - Делает запрос к API/HTML страницы
        
    - Parser преобразует данные в **модель продукта**
        
4. **Result Sender → Aggregator**
    
    - Отправляет результаты в очередь `scraper.results` с `request_id`
        
    - После завершения отправляет **TaskStatus: done**
        
5. **Логирование для аналитики**
    
    - Каждое событие (начало, ошибки, время обработки) публикуется в очередь `analytics.events`
        

---

# 📌 Мини-ER модели Scraper

- **TaskRequest**
    

```python
class TaskRequest(BaseModel):
    request_id: str
    query: str
    marketplace: str  # "WB", "Ozon", "Avito"
    filters: dict
```

- **ScraperResult**
    

```python
class ScraperResult(BaseModel):
    request_id: str
    marketplace: str
    products: list[Product]
    timestamp: datetime
```

- **Product**
    

```python
class Product(BaseModel):
    sku: str
    title: str
    price: float
    availability: str
    url: str
```

- **TaskStatus**
    

```python
class TaskStatus(BaseModel):
    request_id: str
    marketplace: str
    status: str  # "in_progress", "done", "error"
    duration_ms: int
```

---

# 🔹 Поток данных

```
Aggregator → scraper.tasks (queue)
         │
         ▼
     Scraper Consumer
         │
     Fetcher → Parser → Product
         │
         ▼
   scraper.results (queue) → Aggregator
         │
   analytics.events (queue) → Analytics Service
```

---
