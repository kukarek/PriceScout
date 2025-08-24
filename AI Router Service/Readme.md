
---

## **1. Логика работы AI Router как маршрутизатора**

1. **Gateway → Orchestrator → AI Router**
    
    - Приходит запрос пользователя: `"айфон"`
        
    - Orchestrator передаёт его в AI Router с `request_id` и контекстом (категории, фильтры).
        
2. **AI Router принимает решение о скрапперах**
    
    - На основе запроса (или ML/правил) формирует список сервисов:
        

```json
{
  "request_id": "uuid-1234",
  "scrapers": ["WB", "Ozon", "Avito"]
}
```

3. **Отправка результата обратно в Orchestrator**
    
    - Orchestrator получает список скрапперов, которые должны выполнить поиск.
        
    - Orchestrator кладёт задачи для этих скрапперов в очередь.
        

---

## **2. Структура AI Router для маршрутизации**

```bash
ai_router/
├── src/
│   ├── services/
│   │   ├── router.py          # Основной модуль маршрутизации
│   │   ├── ai_client.py       # (опционально) для классификации/ML
│   │   └── result_sender.py   # Отправка списка скрапперов в Orchestrator
│   │
│   ├── consumers/
│   │   └── orchestrator_consumer.py  # Слушает новые запросы поиска
│   │
│   ├── models/
│   │   ├── task.py             # TaskRequest, TaskResult
│   │   └── scraper_route.py    # ScraperRouteResponse
│   │
│   └── main.py                 # FastAPI + consumer
```

- **TaskRequest**: приходит от Orchestrator
    

```python
class TaskRequest(BaseModel):
    request_id: str
    query: str
    categories: list[str]
```

- **ScraperRouteResponse**: возвращается в Orchestrator
    

```python
class ScraperRouteResponse(BaseModel):
    request_id: str
    scrapers: list[str]  # ["WB", "Ozon", "Avito"]
```

---

## **3. Поток данных**

```
Client → Gateway → Orchestrator → AI Router
AI Router: решает, какие скрапперы нужны
AI Router → Orchestrator: ["WB", "Ozon", "Avito"]
Orchestrator → очередь → Scrapers (каждый получает request_id)
Scrapers → Orchestrator → Aggregator → Gateway → Client
```

---

## **4. Дополнительно**

- Можно добавить **приоритет скрапперов**, если какие-то источники важнее.
    
- Можно использовать **ML или правила**, чтобы не дергать все скрапперы, а только релевантные.
    

---
