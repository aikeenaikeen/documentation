# 📚 Admin Frontend — Документация

## 🎯 Описание

**Admin Frontend** — веб-интерфейс для управления системой распознавания лиц. Построен на Vue.js 3 + TypeScript.

## 🛠️ Технологический стек

- **Vue.js 3** + TypeScript + Vite
- **Element Plus** — UI компоненты
- **Pinia** — State management
- **Axios** — HTTP клиент
- **Socket.IO Client** — Real-time обновления

---

## 📁 Основные страницы

- `/login` — Авторизация
- `/dashboard` — Дашборд со статистикой
- `/employees` — Управление сотрудниками
- `/cameras` — Управление камерами
- `/presence` — Текущее присутствие
- `/live` — Просмотр видео потоков
- `/events` — История событий
- `/statistics` — Статистика и отчеты
- `/companies` — Управление компаниями (только SUPERADMIN)

---

## 🔐 Роли пользователей

| Роль | Доступ |
|------|--------|
| **SUPERADMIN** | Все + управление компаниями |
| **COMPANY_ADMIN** | Все в рамках своей компании |
| **USER** | Только просмотр (read-only) |

---

## 🔗 Взаимодействие с другими сервисами

### 1️⃣ Backend API (порт 3000)

**Базовый URL:** `http://localhost:3000`

Admin Frontend взаимодействует с Backend через REST API:

#### API эндпоинты

**Аутентификация:**
- `POST /api/auth/login` → JWT токены
- `POST /api/auth/refresh` → обновление токена

**CRUD операции:**
- `/api/employees` — сотрудники (GET, POST, PUT, DELETE)
- `/api/cameras` — камеры (GET, POST, PUT, DELETE)
- `/api/companies` — компании (GET, POST, PUT, DELETE) — только SUPERADMIN

**Данные:**
- `GET /api/presence` — присутствие
- `GET /api/events` — события (фильтры + пагинация)
- `GET /api/statistics` — статистика

**Особое:**
- `POST /api/cameras/:id/test` — тест подключения
- `GET /api/cameras/:id/stream-url` → URL MJPEG потока

**Механизм:** Axios interceptors (авто-добавление токена, авто-refresh при 401)

---

### 2️⃣ Camera Gateway (порт 4000)

**Получение MJPEG потока:**

1. Admin → Backend: `GET /api/cameras/:id/stream-url`
2. Backend → Admin: `{ mjpegUrl: "http://localhost:4000/streams/1.mjpg" }`
3. Admin отображает: `<img :src="mjpegUrl" />`

---

### 3️⃣ Recognition Service (порт 5000+)

**Поток с распознаванием:**

Прямое подключение: `http://localhost:${5000 + cameraId}/video_feed`

**Пример:**
- Камера ID=1 → порт 5001
- Камера ID=2 → порт 5002
- Камера ID=3 → порт 5003

**Отображение:** `<img :src="streamUrl" />`

---

### 4️⃣ WebSocket (Backend → Admin)

**Socket.IO события:**
- `event:created` — новое событие распознавания
- `employee:created` — новый сотрудник
- `employee:updated` — обновление сотрудника

**Обновляется в real-time:** Dashboard, Events, Presence

---

## 📊 Диаграмма взаимодействия

```
┌─────────────────┐
│  Admin Frontend │  (порт 8080)
│   (Vue.js SPA)  │
└────────┬────────┘
         │
         │ REST API + WebSocket
         │
         ▼
┌─────────────────┐
│   Backend API   │  (порт 3000)
│   (Node.js)     │
└────────┬────────┘
         │
         │ Управление данными (PostgreSQL)
         │
         ├──────────────────┬──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
┌────────────────┐  ┌──────────────┐  ┌─────────────────┐
│ Camera Gateway │  │ Recognition  │  │    Database     │
│  (порт 4000)   │  │   Service    │  │  (PostgreSQL)   │
│                │  │ (порт 5000+) │  │                 │
└────────────────┘  └──────────────┘  └─────────────────┘
         │                  │
         │                  │
         ▼                  ▼
    [IP Камеры]       [IP Камеры]
    MJPEG поток     Поток + распознавание
```

---

## 📋 Требуемые сервисы для работы

| Сервис | Порт | Обязательный | Назначение |
|--------|------|--------------|------------|
| **Backend API** | 3000 | ✅ Да | Основной API, БД, авторизация |
| **Camera Gateway** | 4000 | ⚠️ Для просмотра потоков | Проксирование RTSP → MJPEG |
| **Recognition Service** | 5000+ | ⚠️ Для распознавания | Видео + распознавание лиц |
| **PostgreSQL** | 5432 | ✅ Да | База данных |

---

*Документация актуальна на: декабрь 2025*
