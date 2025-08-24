# 🎯 Цель проекта

Учебная **платформа-агрегатор товаров**, которая:

- ищет товары по названию или артикулу на разных площадках,
    
- использует **AI (GPT API)** для выбора площадок под запрос,
    
- поддерживает **авторизацию и подписки**,
    
- ведёт аналитику запросов пользователей,
    
- построена на **микросервисной архитектуре**,
    
- деплоится через **Docker/Kubernetes**.
    

---

# 🏗 Архитектура (упрощённая)

```
Client (CLI/Web/Postman)
        │
        ▼
 [GraphQL Gateway] ───► [Auth & Billing] ─► PG (users, plans)
        │
        ├──► [Search Orchestrator] ──► [AI Router (GPT API)]
        │                  │
        │                  ├─► [Scraper: WB]
        │                  ├─► [Scraper: Ozon]
        │                  └─► [Scraper: Avito] ...
        │
        └──► [Analytics Service] ◄─(events)
        
Storage: PostgreSQL (offers, users, events)  
Cache:   Redis  
Queue:   RabbitMQ/Kafka  
```

---

# 🔩 Основные сервисы

1. **GraphQL Gateway**
    
    - Принимает запросы (`search`, `login`, `me`).
        
    - Проверяет JWT у Auth-сервиса.
        
    - Делает запросы в Orchestrator.
        
    - Отправляет события в Analytics.
        
2. **Auth & Billing**
    
    - Регистрация, логин, JWT.
        
    - Подписки (free/pro).
        
    - Квоты на количество поисков.
        
    - Таблицы: `users`, `plans`, `subscriptions`.
        
3. **Search Orchestrator**
    
    - Получает запрос от Gateway.
        
    - Спрашивает **AI Router (GPT API)**: какие площадки использовать.
        
    - Параллельно дергает соответствующие Scraper-сервисы.
        
    - Объединяет офферы, убирает дубли, сортирует.
        
    - Сохраняет результат в PG.
        
4. **Scraper-сервисы** (по одному на площадку)
    
    - gRPC API: `Search(query, sku) -> [Offer]`.
        
    - У каждого своя логика поиска.
        
    - Возвращают нормализованные данные.
        
5. **Analytics Service**
    
    - Принимает события (`search.requested`, `search.result_size`).
        
    - Сохраняет в PG (таблица `events`).
        
    - Считает базовые метрики (популярные запросы, активность пользователей).
        

---

# 🔐 Авторизация и подписки

- **JWT** (access + refresh).
    
- Тарифы:
    
    - `free`: 10 поисков в день, ограниченные площадки.
        
    - `pro`: безлимит.
        
- Реализация платежей — сначала mock, позже можно интегрировать.
    

---

# 🗃 Минимальные таблицы

**users**: `id`, `email`, `password_hash`, `plan`, `quota_left`  
**products**: `id`, `canonical_name`, `sku`, `category`  
**offers**: `id`, `product_id`, `marketplace`, `price`, `url`, `title`  
**events**: `id`, `user_id`, `event_type`, `payload(jsonb)`, `ts`

---

# ⚙️ Технологии

- **Python 3.11**, FastAPI, Pydantic
    
- **GraphQL**: Strawberry
    
- **gRPC**: betterproto/grpclib
    
- **PostgreSQL** + SQLAlchemy
    
- **Redis** (кэш, лимиты)
    
- **RabbitMQ/Kafka** (события)
    
- **Celery** (фоновая аналитика)
    
- **Docker + K8s** (деплой)
    


---

## 🔗 Взаимодействие (пример)

1. Клиент → Gateway → Search API (`"найти iPhone 14"`)
    
2. Search API → AI Routing (`куда идти`)
    
3. AI Routing → выбрал: Ozon, M.Video, Ситилинк.
    
4. Gateway → запрос в Aggregator.
    
5. Aggregator → пушит события в Kafka: `search.requested`.
    
6. Коннекторы маркетплейсов поднимаются в K8s (HPA автоскейл).
    
7. Коннекторы находят товары → пишут в Kafka: `product.found`.
    
8. Aggregator собирает все результаты, чистит дубликаты.
    
9. Результат → Gateway → клиент.

___ 
