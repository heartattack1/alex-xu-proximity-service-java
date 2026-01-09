# Proximity Service (Nearby Search)

## Overview

Proximity Service — это backend-сервис для поиска ближайших объектов (businesses) по географическим координатам.
Типичный use case — поиск ресторанов, магазинов или сервисов поблизости, аналогично Yelp или Google Maps Nearby.

Система предоставляет:
- поиск ближайших business по координатам и радиусу;
- CRUD-операции для business;
- масштабируемую архитектуру, оптимизированную под read-heavy нагрузку.

Проект предназначен для генерации Java-приложения (Spring Boot) на основе данного README.

---

## Functional Requirements

### Nearby Search
- Входные параметры:
  - latitude (double)
  - longitude (double)
  - radius (int, метры, optional, default = 5000, max = 20000)
- Выход:
  - список business в пределах радиуса
  - расстояние до каждого business
  - общее количество найденных объектов

### Business Management
- Создание business
- Обновление business
- Удаление business
- Получение деталей business по ID

---

## Non-Functional Requirements

- **Latency**: p95 < 200 ms для nearby search
- **Availability**: 99.9%
- **Scalability**: горизонтальное масштабирование stateless-сервисов
- **Consistency**: eventual consistency между business data и geo-index
- **Privacy**: координаты пользователей не сохраняются

---

## High-Level Architecture

### Components

1. **API Gateway / Load Balancer**
   - маршрутизация HTTP-запросов
   - rate limiting и базовая валидация

2. **LBS (Location-Based Service)**
   - обработка nearby search
   - read-heavy, stateless
   - горизонтально масштабируется

3. **Business Service**
   - CRUD-операции для business
   - управление бизнес-данными

4. **Storage**
   - Primary DB — запись данных
   - Read Replicas — чтение
   - Geo Index Storage — пространственный индекс

5. **Cache (Redis)**
   - кэш business details
   - опциональный кэш geo-index

### Request Flow (Nearby Search)

1. Клиент отправляет запрос с координатами
2. API Gateway маршрутизирует запрос в LBS
3. LBS вычисляет geohash и соседние клетки
4. Из geo-index извлекаются кандидаты
5. Выполняется точная фильтрация по расстоянию
6. Подгружаются данные business
7. Формируется ответ

---

## API Specification

### Nearby Search

```
GET /v1/search/nearby?latitude=37.7749&longitude=-122.4194&radius=5000
```

Response:
```
{
  "total": 2,
  "businesses": [
    {
      "id": "uuid",
      "name": "Coffee Shop",
      "latitude": 37.775,
      "longitude": -122.418,
      "distanceMeters": 120
    }
  ]
}
```

### Business APIs

- `GET /v1/businesses/{id}`
- `POST /v1/businesses`
- `PUT /v1/businesses/{id}`
- `DELETE /v1/businesses/{id}`

---

## Data Model

### business

| Field        | Type    | Description |
|-------------|---------|-------------|
| business_id | UUID PK | Business ID |
| name        | String  | Name |
| latitude    | double  | Latitude |
| longitude   | double  | Longitude |
| address     | String  | Address |
| updated_at  | time    | Last update |

### geo_index

| Field        | Type    | Description |
|-------------|---------|-------------|
| business_id | UUID PK | Business ID |
| geohash     | String  | Geohash prefix |
| latitude    | double  | Latitude |
| longitude   | double  | Longitude |

Index:
- B-tree index on `geohash`

---

## Geo Indexing Strategy

### Geohash

- Используется Geohash для разбиения пространства
- Длина geohash выбирается на основе радиуса поиска

| Radius (km) | Geohash Length |
|------------|----------------|
| <= 1       | 6 |
| <= 5       | 5 |
| <= 20      | 4 |

### Boundary Handling

- Для каждой geohash-клетки вычисляются 8 соседних
- Поиск выполняется по основной и соседним клеткам
- Итоговая фильтрация выполняется через Haversine distance

---

## Consistency Model

- Business data — source of truth
- Geo-index обновляется:
  - MVP: синхронно при записи
  - Target: через outbox + event consumer
- Допускается eventual consistency

---

## Scalability Considerations

- Stateless сервисы
- Горизонтальное масштабирование
- Read replicas
- Batch loading business data
- Redis cache для business details

---

## Project Structure (Recommended)

```
.
├── src/main/java
│   ├── adapter        # REST controllers, DTO
│   ├── application    # use cases
│   ├── domain         # entities, value objects
│   └── infrastructure # JPA, Redis, messaging
├── docker-compose.yml
└── README.md
```

---

## Milestones

### M1
- CRUD business
- DB schema and migrations

### M2
- Geo-index
- Nearby search endpoint

### M3
- Performance optimizations
- Caching

### M4
- Outbox pattern
- Service separation

---

## Local Development

- Java 17
- Spring Boot 3
- PostgreSQL
- Redis
- Docker Compose

---

## Notes

Данный README является единственным источником требований и архитектуры.
Проект должен быть сгенерирован строго в соответствии с данным описанием.
