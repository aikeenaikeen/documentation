# Camera Gateway

Актуализировано 2026-09-23 по `camera-gateway/src/` и конфигурации Compose.

`camera-gateway` — Node.js + TypeScript + Express сервис на порту `4000`. Он читает настройки камеры через Prisma из PostgreSQL, расшифровывает RTSP credentials и через FFmpeg отдаёт браузеру и Recognition MJPEG. Распознавание использует переданный backend URL потока Gateway; утверждение, что оно всегда читает RTSP напрямую, устарело.

## API

| Метод и путь | Назначение |
| --- | --- |
| `GET /streams/:cameraId.mjpg` | MJPEG поток |
| `GET /api/cameras`, `GET /api/cameras/:id` | Камеры и безопасные RTSP URL без пароля |
| `POST /api/cameras/:id/preview` | Проверка получения кадра через FFmpeg |
| `POST /api/streams/:id/reload` | Перезапуск активного потока после изменения камеры |
| `GET /api/health` | Health check |

Все маршруты, кроме health, требуют `CAMERA_GATEWAY_ACCESS_TOKEN` в query `token` либо заголовке `x-gateway-token`/`x-stream-token`. Backend выдаёт браузеру готовый `mjpegUrl` через `GET /api/cameras/:id/stream-url` и передаёт внутренний URL Recognition. Токен в URL является секретом: не публикуйте его в документации или логах.

## Жизненный цикл потока

- По первому подключению Gateway запускает FFmpeg для камеры; несколько клиентов используют один поток.
- После отключения последнего клиента процесс останавливается по `STREAM_IDLE_TIMEOUT_MS` (по умолчанию 15 секунд).
- При сбое процесс переподключается с ограничением числа рестартов. Watchdog отслеживает поток с клиентами и перезапускает FFmpeg, если кадры перестали поступать.
- Разрешение, FPS, JPEG quality, интервалы watchdog и таймауты задаются через env, а не являются фиксированным свойством сервиса.

Детали: [маршруты](https://github.com/aikeenaikeen/camera-gateway/blob/main/src/index.ts), [StreamManager](https://github.com/aikeenaikeen/camera-gateway/blob/main/src/services/streamManager.ts), [переменные](https://github.com/aikeenaikeen/camera-gateway/blob/main/ENV.md), [Backend API](BACKEND_DOCUMENTATION.md).
