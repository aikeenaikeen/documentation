# Infrastructure

Актуализировано 2026-09-23 по `infra/docker-compose*.yml` и скриптам `infra/`.

`infra` запускает весь стек: PostgreSQL 15, Redis 7, backend, Camera Gateway, Admin, Recognition и отдельный training worker. Compose также содержит подготовку общего хранилища клипов, Ollama VLM verifier (профиль `vlm`), Nginx и Portainer (профиль `prod`), а также миграции и seed (профиль `ops`). Это больше не конфигурация одной PostgreSQL.

## Топология

| Сервис | Роль и порт внутри Compose |
| --- | --- |
| `postgres` | Общая БД, `5432` внутри Docker сети; базовый Compose не публикует порт на хост |
| `redis` | Очереди и сигналы, `6379` внутри сети |
| `backend` | API и Socket.IO, `3000` |
| `camera-gateway` | RTSP → MJPEG, `4000` |
| `admin` | SPA, `8080` |
| `recognition-service` | MJPEG `5001`, management/health `7001` |
| `recognition-training-worker` | Отдельное обучение, management `7001` внутри контейнера |
| `vlm-verifier` | Ollama/Qwen3-VL, `11434`, профиль `vlm` |
| `nginx` | UI, `/api`, `/streams`, `/video_feed`, `80/443`, профиль `prod` |

Порты на хосте зависят от используемых Compose-файлов и профилей; `docker-compose.override.yml` публикует локальные порты. Production Nginx опубликован только с профилем `prod`.

## Запуск и обновление

Создайте `infra/.env` по `infra/env.example`, затем из каталога `infra`:

```bash
./pull-all.sh
./deploy.sh --local --gpu --no-seed --health-check
```

На NVIDIA ПК `--gpu` добавляет `docker-compose.gpu.yml`: `AI_DEVICE=cuda`, CUDA для training worker и запрет CPU/legacy fallbacks. `deploy.sh` проверяет Docker/Compose, GPU, свободное место, uploads mount, выполняет backup, миграции и health checks. Для одной проверки окружения используйте `./deploy.sh --local --gpu --preflight-only`.

`vlm-verifier` имеет отдельный Compose profile `vlm`; GPU override задаёт ему GPU, но сам профиль не включает. Для работы локального Qwen сервис должен быть запущен и модель должна быть доступна в Ollama. Активность VLM также определяется recognition config компании.

## Постоянные данные

- PostgreSQL живёт в `postgres-data`; Redis — в `redis-data`.
- `BACKEND_UPLOADS_MOUNT` в production должен указывать на bind mount **вне** `backend/`, например `../data/backend-uploads:/app/uploads`. Там находятся фотографии, training assets и модели.
- `capture-clips` общий для Recognition и backend: первый записывает клипы, второй индексирует и размечает их.
- Перед обновлением используйте `backup-data.sh`; он сохраняет PostgreSQL и `uploads`, но не том `capture-clips`. Если размеченные клипы нужно сохранить вне хоста, копируйте этот том отдельно. Для операций Compose используйте `safe-compose.sh`; не удаляйте volumes при обычном обновлении.
- `sync-remote.sh` исключает `.env` и `backend/uploads` при синхронизации кода.

Точные команды и переменные: [infra/README.md](https://github.com/aikeenaikeen/infra/blob/main/README.md), [infra/ENV.md](https://github.com/aikeenaikeen/infra/blob/main/ENV.md), [Compose](https://github.com/aikeenaikeen/infra/blob/main/docker-compose.yml), [GPU override](https://github.com/aikeenaikeen/infra/blob/main/docker-compose.gpu.yml).
