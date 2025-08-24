Задачи:

- Принимать запросы от фронта / мобильного клиента.
    
- Авторизация + проверка подписки.
    
- Отправка запроса в оркестратор.
    
- Возврат агрегированных результатов пользователю.
    

---

# 📂 Структура Gateway Service

```bash
gateway/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   └── routes.py         # FastAPI эндпоинты: /login, /search, /profile
│   │
│   ├── core/
│   │   ├── config.py         # Настройки (env, порты, урлы сервисов)
│   │   ├── logger.py         # Логирование
│   │   └── security.py       # JWT, OAuth2, проверка подписки
│   │
│   ├── services/
│   │   ├── Aggregator_client.py  # gRPC/HTTP клиент к оркестратору
│   │   └── user_service_client.py  # клиент для сервиса пользователей/биллинга
│   │
│   ├── models/
│   │   ├── user.py           # Pydantic-схемы: User, AuthRequest, AuthResponse
│   │   ├── search.py         # SearchRequest, SearchResponse
│   │   └── subscription.py   # SubscriptionStatus
│   │
│   └── main.py               # Точка входа (FastAPI приложение)
│
├── tests/
│   ├── test_auth.py
│   ├── test_search.py
│   └── test_integration.py
│
├── dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

# ⚙️ Логика работы

1. **Фронт / Мобильный клиент → Gateway**
    
    - Запрос `/search?query=iphone` с JWT токеном.
        
2. **Gateway → Auth/Users Service**
    
    - Проверяет токен, подписку и лимиты.
        
3. **Gateway → Aggregator**
    
    - Пробрасывает запрос поиска (HTTP/gRPC).
        
4. **Aggregator → Gateway → Клиент**
    
    - Возвращает агрегированные результаты.
        

---

# 📌 Мини-ER модели Gateway

- **User**
    
    - `id: uuid`
        
    - `email: str`
        
    - `subscription: str (free/premium)`
        
    - `expires_at: datetime`
        
- **SearchRequest**
    
    - `query: str`
        
    - `categories: list[str]`
        
- **SearchResponse**
    
    - `request_id: str`
        
    - `results: list[Product]`
        

---

👉 Важно: Gateway **не хранит данные** (только кэшировать может), он всего лишь **проверяет доступ и маршрутизирует**.

---
