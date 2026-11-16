# Проектирование API

Пакет ИСР: **A2** (задача 8). Истории: **ID-03, ID-04, ID-06, ID-07**
(также затрагивает ID-01, ID-02, ID-05, ID-09, ID-10).

## 1. Общие принципы

- **Стиль:** REST поверх HTTP, обмен в формате JSON (`application/json`, UTF-8).
- **Базовый путь:** `/api`.
- **Идентификаторы ресурсов:** UUID.
- **Даты и время:** ISO-8601 с часовым поясом (`OffsetDateTime`), например
  `2026-11-20T09:00:00+03:00`. Часовой пояс парковки — `Europe/Moscow`.
- **Состояние:** приложение stateless; аутентификация — по токену в каждом запросе.
- **Разделение прав:** выполняется на сервере (см. `../requirements/roles-and-permissions.md`).

## 2. Аутентификация и авторизация

- Заголовок запроса: `Authorization: Bearer <JWT access token>`.
- Токен выдаётся при входе (`/api/auth/login`) и содержит роли пользователя.
- Правила доступа (Spring Security):

| Правило | Пути | Требование |
| --- | --- | --- |
| Публичные | `/api/auth/register`, `/api/auth/login` | без токена |
| Административные | `/api/admin/**` | роль `PARKING_ADMIN` |
| Прочие | все остальные | аутентификация (любая роль) |

- Ответы безопасности: `401` — нет/просрочен токен, `403` — недостаточно прав.

## 3. Ресурсы и эндпоинты

### 3.1. Аутентификация

| Метод | Путь | Назначение |
| --- | --- | --- |
| POST | `/api/auth/register` | Регистрация (роль `TEACHER` или `TECHNICAL_STAFF`) |
| POST | `/api/auth/login` | Вход, выдача токена |
| GET | `/api/auth/me` | Текущий пользователь и его роли |

### 3.2. Транспортные средства (владелец — текущий пользователь)

| Метод | Путь | Назначение |
| --- | --- | --- |
| GET | `/api/vehicles` | Список ТС пользователя |
| POST | `/api/vehicles` | Добавить ТС |
| PATCH | `/api/vehicles/{vehicleId}` | Изменить ТС |
| POST | `/api/vehicles/{vehicleId}/deactivate` | Деактивировать ТС |

### 3.3. Парковки и схема

| Метод | Путь | Назначение |
| --- | --- | --- |
| GET | `/api/parking-lots/available` | Парковки, доступные пользователю |
| GET | `/api/parking-lots/default/layout` | Схема парковки по умолчанию |
| GET | `/api/parking-lots/{parkingLotId}/layout` | Схема выбранной парковки |
| GET | `/api/parking/levels/{parkingLevelId}/availability` | Доступность мест уровня на период (параметры `vehicleId`, `startsAt`, `endsAt`) |

### 3.4. Бронирования

| Метод | Путь | Назначение |
| --- | --- | --- |
| GET | `/api/reservations/my/active` | Активные брони пользователя |
| GET | `/api/reservations/my/history` | История броней пользователя |
| POST | `/api/reservations` | Создать бронь (`201 Created`) |
| POST | `/api/reservations/{reservationId}/cancel` | Отменить бронь |
| PATCH | `/api/reservations/{reservationId}/period` | Перенести / продлить бронь |

### 3.5. Администрирование (`/api/admin/**`, роль `PARKING_ADMIN`)

| Метод | Путь | Назначение |
| --- | --- | --- |
| GET | `/api/admin/parking-lots` | Список парковок |
| POST | `/api/admin/parking-lots` | Создать парковку |
| PATCH | `/api/admin/parking-lots/{parkingLotId}` | Изменить парковку |
| DELETE | `/api/admin/parking-lots/{parkingLotId}` | Удалить парковку |
| GET | `/api/admin/parking-lots/{parkingLotId}/layout` | Схема парковки (админ) |
| POST | `/api/admin/parking-lots/{parkingLotId}/levels` | Создать уровень |
| PATCH | `/api/admin/parking-levels/{parkingLevelId}` | Изменить уровень |
| DELETE | `/api/admin/parking-levels/{parkingLevelId}` | Удалить уровень |
| POST | `/api/admin/parking-levels/{parkingLevelId}/spots` | Создать место |
| PATCH | `/api/admin/parking-spots/{spotId}` | Изменить место |
| DELETE | `/api/admin/parking-spots/{spotId}` | Удалить место |
| POST | `/api/admin/parking-levels/{parkingLevelId}/layout-elements` | Добавить элемент схемы |
| PATCH | `/api/admin/parking-layout-elements/{elementId}` | Изменить элемент схемы |
| DELETE | `/api/admin/parking-layout-elements/{elementId}` | Удалить элемент схемы |
| GET | `/api/admin/users` | Список пользователей |
| PUT | `/api/admin/users/{userId}/parking-lots` | Заменить набор доступных парковок |
| GET | `/api/admin/users/{userId}/vehicles` | ТС пользователя |
| POST | `/api/admin/users/{userId}/vehicles` | Добавить ТС пользователю |
| PATCH | `/api/admin/users/{userId}/vehicles/{vehicleId}` | Изменить ТС пользователя |
| POST | `/api/admin/users/{userId}/vehicles/{vehicleId}/deactivate` | Деактивировать ТС |
| GET | `/api/admin/users/{userId}/reservations/active` | Активные брони пользователя |
| GET | `/api/admin/users/{userId}/reservations/history` | История броней пользователя |
| POST | `/api/admin/users/{userId}/reservations` | Специальная бронь (`201 Created`) |
| POST | `/api/admin/users/{userId}/reservations/{reservationId}/cancel` | Отменить бронь пользователя |
| PATCH | `/api/admin/users/{userId}/reservations/{reservationId}/period` | Изменить период брони пользователя |
| GET | `/api/admin/parking-spots/{parkingSpotId}/reservation` | Бронь места на период (`startsAt`, `endsAt`) |

## 4. Модели запросов и ответов

### Регистрация

```json
POST /api/auth/register
{
  "email": "user@vsu.local",
  "password": "secret123",
  "fullName": "Иванов Иван",
  "phone": "+7 (999) 123-45-67",
  "department": "Кафедра ИТ",
  "roleCode": "TEACHER"
}
```

### Создание брони

```json
POST /api/reservations
{
  "vehicleId": "uuid",
  "parkingSpotId": "uuid",
  "startsAt": "2026-11-20T09:00:00+03:00",
  "endsAt": "2026-11-20T11:00:00+03:00"
}
```

### Создание ТС

```json
POST /api/vehicles
{
  "licensePlate": "А123АА36",
  "widthMeters": 1.8,
  "lengthMeters": 4.5,
  "heightMeters": 1.6
}
```

### Выдача доступа к парковкам

```json
PUT /api/admin/users/{userId}/parking-lots
{ "parkingLotIds": ["uuid-1", "uuid-2"] }
```

## 5. Формат ошибок

Все ошибки возвращаются в едином виде:

```json
{
  "timestamp": "2026-11-20T09:00:00+03:00",
  "status": 409,
  "error": "Конфликт",
  "message": "Парковочное место уже занято на выбранный период",
  "path": "/api/reservations"
}
```

| Код | Когда |
| --- | --- |
| 400 | Некорректные данные или параметры (валидация, бизнес-правила) |
| 401 | Не авторизован (нет/просрочен токен) |
| 403 | Недостаточно прав (нет роли `PARKING_ADMIN`) |
| 404 | Ресурс не найден |
| 409 | Конфликт: пересечение броней, нарушение целостности |
| 500 | Внутренняя ошибка сервера |

## 6. Связь с диаграммами

- Порядок вызовов при создании брони — `../diagrams/Sequence создание бронирования.drawio`.
- Аутентификация и выдача токена — `../diagrams/Sequence аутентификация.drawio`.
- Сущности и связи — `../diagrams/ER-диаграмма.drawio`.
