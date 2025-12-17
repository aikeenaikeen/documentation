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

#### 🔐 Аутентификация
- `POST /api/auth/login` — Вход (получение JWT токенов)
- `POST /api/auth/refresh` — Обновление access токена

**Механизм:**
- Request interceptor автоматически добавляет `Authorization: Bearer ${token}` к каждому запросу
- Response interceptor перехватывает 401 и автоматически обновляет токен через refresh endpoint
- Токены хранятся в localStorage

#### 👥 Сотрудники
- `GET /api/employees` — Список всех сотрудников
- `POST /api/employees` — Создать сотрудника (с загрузкой фото через FormData)
- `PUT /api/employees/:id` — Обновить сотрудника
- `DELETE /api/employees/:id` — Удалить сотрудника

#### 📹 Камеры
- `GET /api/cameras` — Список камер
- `POST /api/cameras` — Создать камеру (с RTSP данными)
- `PUT /api/cameras/:id` — Обновить камеру
- `DELETE /api/cameras/:id` — Удалить камеру
- `POST /api/cameras/:id/test` — Тест подключения к камере
- `GET /api/cameras/:id/stream-url` — Получить URL MJPEG потока

#### 📊 Данные
- `GET /api/presence` — Текущий список присутствующих
- `GET /api/events` — История событий распознавания (с фильтрами)
- `GET /api/statistics` — Статистика и аналитика

#### 🏢 Компании (только SUPERADMIN)
- `GET /api/companies` — Список компаний
- `POST /api/companies` — Создать компанию
- `PUT /api/companies/:id` — Обновить
- `DELETE /api/companies/:id` — Удалить

---

### 2️⃣ Camera Gateway

**Назначение:** Proxy для получения видео потоков с IP-камер

#### Получение MJPEG потока

```typescript
// 1. Запрашиваем URL потока у Backend
const response = await apiClient.get(`/api/cameras/${cameraId}/stream-url`)

// 2. Backend возвращает URL Camera Gateway
// { mjpegUrl: "http://localhost:4000/stream/1" }

// 3. Отображаем поток в <img>
<img :src="response.data.mjpegUrl" />
```

**Порядок работы:**
1. Admin запрашивает stream URL у Backend API
2. Backend возвращает URL Camera Gateway
3. Camera Gateway проксирует RTSP поток с камеры в MJPEG
4. Admin отображает MJPEG поток в браузере

---

### 3️⃣ Recognition Service

**Назначение:** Видео поток с наложением результатов распознавания (рамки, имена)

#### Получение потока с распознаванием

```typescript
// Прямое подключение к Recognition Service
// Порт = 5000 + cameraId
const streamUrl = `http://localhost:${5000 + cameraId}/video_feed`

<img :src="streamUrl" />
```

**Особенности:**
- Каждая камера имеет свой экземпляр Recognition Service на отдельном порту
- Порт вычисляется как: `5000 + cameraId`
- Поток уже содержит overlays с результатами распознавания
- Если сервис недоступен — показывается ошибка

**Пример:**
- Камера ID=1 → `http://localhost:5001/video_feed`
- Камера ID=2 → `http://localhost:5002/video_feed`
- Камера ID=3 → `http://localhost:5003/video_feed`

---

### 4️⃣ WebSocket (Real-time обновления)

**Подключение:** Socket.IO к Backend

```typescript
import { io } from 'socket.io-client'

const socket = io('http://localhost:3000')

// События от Backend
socket.on('newEvent', (event) => {
  // Новое событие распознавания
  // Автоматически добавляется в таблицу Events
})

socket.on('presenceUpdate', (data) => {
  // Изменение статуса присутствия сотрудника
  // Обновляется страница Presence
})

socket.on('cameraStatus', (data) => {
  // Изменение статуса камеры (вкл/выкл)
})
```

**Что обновляется в real-time:**
- ✅ Новые события распознавания
- ✅ Изменение присутствия сотрудников
- ✅ Статусы камер
- ✅ Счетчики на Dashboard

---

## ⚙️ Конфигурация

### Переменные окружения (.env)

```bash
VITE_API_BASE_URL=http://localhost:3000
```

### Vite Proxy (для разработки)

```typescript
server: {
  port: 8080,
  proxy: {
    '/api': { target: 'http://localhost:3000' },
    '/uploads': { target: 'http://localhost:3000' }
  }
}
```

---

## 🚀 Запуск

```bash
# Development
npm run dev
# → http://localhost:8080

# Production build
npm run build
```

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

*Документация актуальна на: декабрь 2024*
