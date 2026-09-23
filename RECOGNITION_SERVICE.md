# Recognition Service

Актуализировано 2026-09-23 по `recognition_service/main.py`, `video_loop.py`, `streaming_api.py`, `management_api.py` и конфигурации Compose.

`recognition_service` — Python ML-сервис для всех компаний и камер. Он получает через backend конфигурацию и MJPEG URL Camera Gateway, определяет сотрудников и их активности, а результаты отправляет обратно в backend. Training запущен в отдельном контейнере `recognition-training-worker` (`TRAINING_ONLY=1`).

## Runtime pipeline

1. `GlobalCameraManager` получает активные камеры и конфигурацию через HMAC API backend, запускает camera workers и обновляет данные.
2. Worker читает MJPEG поток Gateway. InsightFace распознаёт лицо; YOLO находит человека, связывает лицо с person track и готовит crop сотрудника.
3. По назначенным сотруднику активностям action model обрабатывает буфер person crops. Object cues мягко корректируют score; затем применяются сглаживание и разрешение конфликтующих кандидатов.
4. Кандидат, прошедший `startThreshold`, может проверяться Qwen3-VL через локальный Ollama. Политика для неопределённого ответа и ошибки задаётся recognition config; при отключённом verifier этот шаг пропускается.
5. Подтверждённые результаты попадают в `ActivityIntervalAggregator`, затем в backend как интервалы и evidence. При `ACTIVITY_SHADOW_MODE=1` модели и capture работают, но интервалы и события наружу не публикуются.

Обучение VideoMAE и inference используют person crops, а не полный кадр. Загруженные training assets и ONNX модели хранятся через backend. При `CAPTURE_ENABLED=1` сервис также сохраняет клипы с потока в общий `capture-clips`; Admin размечает их через backend.

## HTTP интерфейсы

| Порт | Маршруты | Назначение |
| --- | --- | --- |
| `5001` (`PORT`) | `/video_feed?cameraId=:id`, `/cameras/:id/video_feed`, `/health`, `/cameras` | Единый streaming API для всех камер |
| `7001` (`MANAGEMENT_PORT`) | `/health`, `/metrics`, `/object-classes` | Состояние inference и GPU |
| `7001` training worker | `/train`, `/preview-training-samples`, `/health` | Служебное обучение с подписанными запросами |

Порт `5000 + cameraId` и обязательный `COMPANY_SLUG` относятся к старой схеме. Для диагностики inference используйте management `/health`: он показывает режим, состояние camera manager и AI device.

## Конфигурация и зависимости

- Для NVIDIA production применяйте GPU override: `AI_DEVICE=cuda`, `TRAINING_PERSON_DET_DEVICE=cuda`, `ALLOW_PERSON_DET_CPU_FALLBACK=0`, `ALLOW_OBJECT_CUE_CPU_FALLBACK=0`, `ALLOW_FACE_CPU_FALLBACK=0`, `ALLOW_LEGACY_FACE_ONLY_MODE=0`.
- Пороги активностей, назначения и VLM политика приходят из company/activity config backend. Env задаёт инфраструктурные параметры: backend URL, HMAC secret, Redis, GPU, адрес/модель Ollama, capture.
- Qwen3-VL через Ollama — дополнительный verifier после action threshold. Compose сервис `vlm-verifier` находится в профиле `vlm`; наличие GPU override само по себе его не запускает.

Исходники и настройки: [main.py](https://github.com/aikeenaikeen/recognition_service/blob/main/main.py), [video_loop.py](https://github.com/aikeenaikeen/recognition_service/blob/main/video_loop.py), [streaming_api.py](https://github.com/aikeenaikeen/recognition_service/blob/main/streaming_api.py), [management_api.py](https://github.com/aikeenaikeen/recognition_service/blob/main/management_api.py), [ENV.md](https://github.com/aikeenaikeen/recognition_service/blob/main/ENV.md).
