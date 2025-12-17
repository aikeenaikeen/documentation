# 📚 Backend API — Документация

## 🎯 Описание

**Backend API** — центральный сервер системы. Хранит данные (PostgreSQL), предоставляет REST API и WebSocket.

## 🛠️ Технологический стек

- **Node.js + TypeScript + Express + Prisma ORM**
- **JWT** (access + refresh), **bcrypt**, **AES-256**
- **Socket.IO** — real-time
- **Multer + Sharp** — загрузка фото

## 📊 База данных

| Таблица | Описание |
|---------|----------|
| **Company** | Компании |
| **User** | Пользователи (SUPERADMIN, COMPANY_ADMIN, USER) |
| **Employee** | Сотрудники + фото |
| **Camera** | IP-камеры (RTSP, пароли AES-256) |
| **Event** | События распознавания (IN/OUT) |

---

## 🔗 Взаимодействие с другими сервисами

### 1️⃣ Admin Frontend → Backend

#### Аутентификация
- `POST /api/auth/login` → `{ accessToken, refreshToken, user }`
- `POST /api/auth/refresh` → обновление access токена

**JWT:** Access (15 мин), Refresh (7 дней)

#### Сотрудники
- `GET /api/employees` — список
- `POST /api/employees` — создать (multipart/form-data + фото)
- `PUT /api/employees/:id` — обновить
- `DELETE /api/employees/:id` — удалить

**Фото:** Sharp обработка → `uploads/employees/`

#### Камеры
- `GET /api/cameras` — список
- `POST /api/cameras` — создать (пароль шифруется AES-256)
- `PUT /api/cameras/:id` — обновить
- `DELETE /api/cameras/:id` — удалить
- `POST /api/cameras/:id/test` — тест подключения
- `GET /api/cameras/:id/stream-url` → `{ mjpegUrl: "http://localhost:4000/streams/1.mjpg" }`

#### Данные
- `GET /api/presence` — присутствие
- `GET /api/events` — история (фильтры + пагинация)
- `GET /api/statistics` — статистика

#### Компании (SUPERADMIN only)
- `GET /api/companies`, `POST /api/companies`, `PUT /api/companies/:id`, `DELETE /api/companies/:id`

---

### 2️⃣ Recognition Service → Backend

#### Отправка событий распознавания

`POST /api/events`

```json
{
  "employeeId": 123,
  "type": "IN",  // или "OUT"
  "cameraId": 1,
  "timestamp": "2024-12-17T10:30:00Z"
}
```

**Обработка:**
1. Проверка существования сотрудника
2. Дедупликация (если то же событие < 60 сек назад → пропуск)
3. Сохранение в БД
4. WebSocket broadcast → Admin Frontend
5. Возврат `{ ok: true }`

#### Загрузка данных

- `GET /api/employees` — список сотрудников с фото (для кэша лиц)
- `GET /api/cameras/company/:slug` — список камер компании

---

### 3️⃣ Backend → Camera Gateway

#### Тест подключения к камере

Admin → `POST /api/cameras/:id/test` → Backend:
1. Расшифровывает пароль камеры (AES-256)
2. Формирует RTSP URL: `rtsp://user:pass@ip:554/path`
3. Отправляет в Camera Gateway: `POST /api/cameras/:id/preview`
4. Возвращает результат Admin

**Ответ:** `{ ok: true, latencyMs: 245 }`

---

### 4️⃣ WebSocket (Backend → Admin)

**События Socket.IO:**

- `event:created` — новое событие распознавания
- `employee:created` — новый сотрудник
- `employee:updated` — обновление сотрудника

**Обновляется в real-time:** Dashboard, Events, Presence

---

## ⚙️ Конфигурация

### Переменные окружения (.env)

```bash
# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/attendance

# JWT
JWT_ACCESS_SECRET=change-me-access-secret
JWT_REFRESH_SECRET=change-me-refresh-secret
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# Encryption (для паролей камер)
ENCRYPTION_KEY=change-me-32-char-encryption-key

# File uploads
UPLOADS_DIR=./uploads
MAX_FILE_SIZE=10485760  # 10MB

# CORS
CORS_ORIGIN=*

# Socket.IO
SOCKET_PATH=/ws

# Camera Gateway
CAMERA_GATEWAY_PUBLIC_URL=http://localhost:4000
CAMERA_GATEWAY_INTERNAL_URL=http://camera-gateway:4000

# Event deduplication
EVENT_DEDUPLICATION_WINDOW_MS=60000  # 60 секунд
```

---

## 🚀 Запуск

```bash
# Development
npm run dev
# → http://localhost:3000

# Production build
npm run build
npm start

# Prisma migrations
npm run prisma:migrate

# Database seed (тестовые данные)
npm run prisma:seed
```

---

## 📊 Диаграмма взаимодействия

```
                   ┌─────────────────┐
                   │  Admin Frontend │ (порт 8080)
                   └────────┬────────┘
                            │
                  REST API + WebSocket
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │      Backend API (порт 3000)          │
        │   Express + Prisma ORM + Socket.IO    │
        │                                        │
        │  • JWT Auth                            │
        │  • Events (дедупликация)               │
        │  • Employees (фото обработка)          │
        │  • Cameras (AES-256 шифрование)        │
        └────┬──────────┬──────────┬─────────────┘
             │          │          │
             │          │          │ POST /api/events
             │          │          │ GET /api/employees
             │          │          │
             ▼          │          ▼
     ┌──────────────┐  │  ┌──────────────────┐
     │  PostgreSQL  │  │  │  Recognition     │
     │   Database   │  │  │    Service       │
     │ (порт 5432)  │  │  │   (Python)       │
     └──────────────┘  │  └──────────────────┘
                       │          │
                       │          │ Читает RTSP
                       │          │
                       │          ▼
                       │    [IP Камеры]
                       │          │
                       │          │ RTSP поток
                       │          │
                       ▼          ▼
              ┌──────────────────────┐
              │   Camera Gateway     │ (порт 4000)
              │   RTSP → MJPEG       │
              └──────────────────────┘
                       │
                       │ MJPEG streams
                       ▼
              Admin Frontend (просмотр видео)
```

---

## 📋 Требуемые сервисы

| Сервис | Порт | Обязательный |
|--------|------|--------------|
| **PostgreSQL** | 5432 | ✅ Да |
| **Camera Gateway** | 4000 | ⚠️ Для просмотра потоков |
| **Recognition Service** | - | ⚠️ Для распознавания |

---

*Документация актуальна на: декабрь 2024*

