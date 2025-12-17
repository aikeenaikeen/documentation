# 📚 Recognition Service — Документация

## 🎯 Описание

**Recognition Service** — Python сервис для распознавания лиц в реальном времени. Обрабатывает видео с камер, распознает сотрудников, отправляет события в Backend.

## 🛠️ Технологический стек

- **Python 3.10+**
- **InsightFace** — распознавание лиц (buffalo_l модель)
- **OpenCV** — обработка видео
- **Flask** — HTTP сервер для видео стримов
- **NumPy** — математика
- **Requests** — HTTP клиент

---

## 🔗 Взаимодействие с другими сервисами

### 1️⃣ Backend API → Recognition Service

#### Загрузка списка сотрудников

`GET /api/employees`

**Когда:** При запуске + каждые 5 минут (настраивается)

**Ответ:**
```json
[
  {
    "id": 1,
    "name": "Иван Иванов",
    "role": "Менеджер",
    "photoUrl": "/uploads/employees/photo-123.jpg"
  }
]
```

**Что делает Recognition Service:**
1. Загружает фото сотрудников по URL
2. Извлекает face embeddings (512D вектора) через InsightFace
3. Кэширует в `face_encodings_cache.pkl` (для быстрого рестарта)
4. Использует для распознавания в реальном времени

---

#### Загрузка списка камер компании

`GET /api/cameras/public/{companySlug}/cameras`

**Когда:** При запуске + каждые 60 секунд (настраивается)

**Ответ:**
```json
[
  {
    "id": 1,
    "name": "Вход главный",
    "location": "1 этаж",
    "streamUrl": "rtsp://user:pass@192.168.1.100:554/path"
  }
]
```

**Multi-Camera режим:**
- MultiCameraManager периодически синхронизируется с Backend
- Автоматически запускает новые камеры
- Автоматически останавливает удаленные камеры
- Один поток на камеру

---

### 2️⃣ Recognition Service → Backend API

#### Отправка событий распознавания

`POST /api/events`

**Когда:** При распознавании сотрудника

**Payload:**
```json
{
  "employeeId": 123,
  "type": "IN",  // или "OUT"
  "cameraId": 1,
  "timestamp": "2025-12-17T10:30:00Z"
}
```

**Логика присутствия:**
- **IN событие:** сотрудник стабильно присутствует > 1 секунды
- **OUT событие:** сотрудник отсутствует > 10 секунд

**Anti-spam:**
- Backend дедуплицирует события (60 сек окно)
- Recognition Service также имеет cooldown логику

---

### 3️⃣ IP Камеры → Recognition Service

#### Чтение видео потоков

Recognition Service читает RTSP напрямую с IP-камер:

```python
# Подключение к RTSP
rtsp_url = "rtsp://username:password@192.168.1.100:554/ISAPI/Streaming/Channels/101"
cap = cv2.VideoCapture(rtsp_url)
```

**Параметры:**
- Frame skip: обработка каждого 3-го кадра (для производительности)
- RTSP transport: TCP (для стабильности)
- Автоматический реконнект при обрыве соединения

**Альтернатива:** Можно читать MJPEG от Camera Gateway (через streamUrl)

---

### 4️⃣ Recognition Service → Admin Frontend

#### Flask видео сервер

Каждая камера предоставляет HTTP endpoint для просмотра видео с распознаванием.

**Endpoint:** `GET /video_feed`

**Порт:** `5000 + cameraId`

**Примеры:**
- Камера ID=1 → `http://localhost:5001/video_feed`
- Камера ID=2 → `http://localhost:5002/video_feed`
- Камера ID=3 → `http://localhost:5003/video_feed`

**Что отображается:**
- Bounding boxes вокруг лиц
- Имена распознанных сотрудников
- Confidence score
- FPS счетчик

**Формат:** MJPEG stream (`multipart/x-mixed-replace`)

**Использование в Admin:**
```html
<img :src="`http://localhost:${5000 + cameraId}/video_feed`" />
```

---

## 📊 Диаграмма взаимодействия

```
┌──────────────────────────────────────────────┐
│   Recognition Service (Python, порт 5000+)   │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  MultiCameraManager                    │ │
│  │  • Синхронизация камер с Backend       │ │
│  │  • Один поток на камеру                │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  Video Loop (per camera)               │ │
│  │  1. Чтение кадра (OpenCV)              │ │
│  │  2. Детекция лиц (InsightFace)         │ │
│  │  3. Извлечение embeddings              │ │
│  │  4. Matching с базой сотрудников       │ │
│  │  5. Tracking (FaceTracker)             │ │
│  │  6. Presence logic (IN/OUT)            │ │
│  │  7. Отправка событий в Backend         │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  Flask Server (per camera)             │ │
│  │  GET /video_feed → MJPEG с overlays    │ │
│  └────────────────────────────────────────┘ │
└──────┬───────────────┬───────────────────────┘
       │               │
       │               │ GET /api/employees
       │               │ GET /api/cameras/public/{slug}/cameras
       │               │ POST /api/events
       │               │
       ▼               ▼
┌─────────────┐  ┌──────────┐
│  IP Камеры  │  │ Backend  │
│             │  │   API    │
│ RTSP поток  │  │          │
└─────────────┘  └──────────┘
       │               │
       │               │ MJPEG stream (видео + overlays)
       │               ▼
       │        ┌──────────────┐
       │        │    Admin     │
       │        │  Frontend    │
       └────────┴──────────────┘
```

---

## 🧠 Pipeline распознавания

**Шаг 1:** Чтение кадра (OpenCV)  
**Шаг 2:** Детекция лиц (InsightFace, buffalo_l)  
**Шаг 3:** Извлечение embeddings (512D векторы)  
**Шаг 4:** Matching с базой (cosine similarity, threshold 0.2)  
**Шаг 5:** Tracking (FaceTracker, IoU matching)  
**Шаг 6:** Presence logic:
- IN: стабильное присутствие > 1 сек
- OUT: отсутствие > 10 сек

**Шаг 7:** Отправка событий в Backend (POST /api/events)

---

## 💾 Кэширование

**Файл:** `face_encodings_cache.pkl`

**Содержит:**
- Face embeddings всех сотрудников
- Employee IDs
- Hash списка сотрудников (для валидации)
- Timestamp

**Логика:**
- При запуске проверяет кэш
- Если hash совпадает → использует кэш (быстро)
- Если hash изменился → перезагружает все фото
- При добавлении/удалении сотрудника → кэш инвалидируется

---

## 📋 Требуемые зависимости

| Сервис | Обязательный |
|--------|--------------|
| **Backend API** | ✅ Да |
| **IP Камеры (RTSP)** | ✅ Да |
| **InsightFace модели** | ✅ Да (авто-загрузка) |

---

*Документация актуальна на: декабрь 2025*

