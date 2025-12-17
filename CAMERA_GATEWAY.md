# 📚 Camera Gateway — Документация

## 🎯 Описание

**Camera Gateway** — сервис для конвертации RTSP потоков с IP-камер в MJPEG формат для отображения в браузере.

## 🛠️ Технологический стек

- **Node.js + TypeScript + Express**
- **FFmpeg** — конвертация RTSP → MJPEG
- **Prisma ORM** — чтение данных камер из PostgreSQL (read-only)
- **AES-256** — расшифровка паролей камер

---

## 🔗 Взаимодействие с другими сервисами

### 1️⃣ Admin Frontend → Camera Gateway

#### Получение MJPEG потока

`GET /streams/:cameraId.mjpg`

**Использование:**
```html
<img src="http://localhost:4000/streams/1.mjpg" />
```

**Ответ:** Бесконечный MJPEG stream (`multipart/x-mixed-replace`)

---

### 2️⃣ Backend API → Camera Gateway

#### Тест подключения к камере

`POST /api/cameras/:id/preview`

**Процесс:**
1. Читает данные камеры из БД
2. Расшифровывает пароль (AES-256)
3. Пытается подключиться через FFmpeg
4. Возвращает `{ ok: true/false, latencyMs }`

---

### 3️⃣ Recognition Service → Camera Gateway

❌ **Не используется** — Recognition Service читает RTSP напрямую с камер, а не через Gateway.

---

### 4️⃣ Camera Gateway → IP Камеры (RTSP)

#### Процесс конвертации RTSP → MJPEG

1. Читает данные камеры из PostgreSQL (Prisma)
2. Расшифровывает пароль (AES-256)
3. Формирует RTSP URL: `rtsp://user:pass@ip:554/path`
4. Запускает FFmpeg (RTSP → MJPEG, 1280x720, 10 FPS)
5. Стримит клиентам (один FFmpeg на камеру)

**StreamManager:**
- Запуск при подключении клиента
- Авто-остановка через 15 сек без клиентов

---

## 📊 Диаграмма взаимодействия

```
┌─────────────────┐
│  Admin Frontend │
└────────┬────────┘
         │
         │ GET /streams/1.mjpg
         │ (MJPEG поток)
         │
         ▼
┌─────────────────────────────────────┐
│    Camera Gateway (порт 4000)       │
│                                     │
│  ┌───────────────────────────────┐ │
│  │     StreamManager             │ │
│  │  • Управление FFmpeg          │ │
│  │  • Один процесс на камеру     │ │
│  │  • Авто-остановка (idle)      │ │
│  └───────────────────────────────┘ │
│                                     │
│  ┌───────────────────────────────┐ │
│  │   FFmpeg Process (per camera) │ │
│  │   RTSP → MJPEG конвертация    │ │
│  └───────────────────────────────┘ │
└──────────┬──────────┬───────────────┘
           │          │
           │          │ Читает данные камер
           │          ▼
           │  ┌──────────────┐
           │  │  PostgreSQL  │ (read-only)
           │  │   Database   │
           │  └──────────────┘
           │
           │ RTSP подключение
           │ rtsp://user:pass@ip:554/path
           │
           ▼
    [IP Камера 1]  [IP Камера 2]  [IP Камера N]
    Hikvision      Dahua          ...
```

---

## 📋 Требуемые зависимости

| Сервис | Обязательный |
|--------|--------------|
| **PostgreSQL** | ✅ Да |
| **FFmpeg** | ✅ Да |

---

*Документация актуальна на: декабрь 2025*

