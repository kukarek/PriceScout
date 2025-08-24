# 📂 Структура Aggregator Service

---

```bash
Aggregator/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py        # REST/GraphQL ручки для общения с Gateway
│   │
│   ├── core/
│   │   ├── config.py        # Настройки (env, константы)
│   │   ├── logger.py        # Логирование
│   │   └── utils.py         # Вспомогательные утилиты
│   │
│   ├── services/
│   │   ├── dispatcher.py    # Отправка задач в очередь для скрапперов
│   │   ├── aggregator.py    # Агрегация ответов от скрапперов
│   │   └── tracker.py       # Отслеживание статуса (какие скрапперы уже ответили)
│   │
│   ├── consumers/
│   │   ├── scraper_consumer.py  # Слушает очередь с результатами от скрапперов
│   │   └── __init__.py
│   │
│   ├── models/
│   │   ├── request.py       # Pydantic-модели: SearchRequest, SearchResult
│   │   └── response.py
│   │
│   └── main.py              # Точка входа (FastAPI + consumers запуск)
│
├── tests/                   # Тесты (unit + integration)
│   ├── test_api.py
│   ├── test_dispatcher.py
│   └── test_aggregator.py
│
├── dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# ⚙️ Логика работы

1. **Gateway → Aggregator**
    
    - Приходит запрос поиска (`search: "айфон"`, `categories: ["phones"]`).
        
    - Оркестратор создаёт `request_id`.
        
2. **Dispatcher**
    
    - Кладёт задачи в очередь (например, RabbitMQ) для каждого нужного скраппера.
        
    - В задаче: `request_id`, `query`, `filters`.
        
3. **Scrapers → Aggregator (Consumer)**
    
    - Скрапперы возвращают результаты с этим `request_id`.
        
    - Consumer слушает очередь и передаёт данные в `aggregator`.
        
4. **Aggregator**
    
    - Собирает все ответы по `request_id`.
        
    - Знает список ожидаемых скрапперов (например, `ozon`, `wb`, `avito`).
        
    - Если все ответили или вышел timeout → отдаёт результат клиенту.
        
5. **Tracker**
    
    - Ведёт состояние по каждому запросу:
        
        - какие скрапперы уже ответили
            
        - сколько ещё ждём
            
        - статус (`in_progress`, `done`, `timeout`).
            

---

# 📌 Мини-ER схема в памяти

- **SearchRequest**
    
    - `request_id: str`
        
    - `query: str`
        
    - `status: str (in_progress/done/timeout)`
        
    - `expected_scrapers: list[str]`
        
    - `received_scrapers: list[str]`
        
    - `results: list[Product]`
        
- **Product**
    
    - `title: str`
        
    - `price: float`
        
    - `url: str`
        
    - `source: str (ozon/wb/avito)`
        

---

