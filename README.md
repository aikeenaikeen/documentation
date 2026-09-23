# Aikeen: обзорная документация

Сверено с локальными репозиториями 2026-09-23. Этот каталог описывает связи сервисов; точные параметры запуска и API всегда проверяйте в исходниках и `README.md`/`ENV.md` соответствующего сервиса.

| Раздел | Файл |
| --- | --- |
| Схема взаимодействия сервисов | [SERVICE_INTERACTIONS.md](SERVICE_INTERACTIONS.md) |
| Admin Frontend | [ADMIN_DOCUMENTATION.md](ADMIN_DOCUMENTATION.md) |
| Backend API | [BACKEND_DOCUMENTATION.md](BACKEND_DOCUMENTATION.md) |
| Camera Gateway | [CAMERA_GATEWAY.md](CAMERA_GATEWAY.md) |
| Recognition Service | [RECOGNITION_SERVICE.md](RECOGNITION_SERVICE.md) |
| Infrastructure | [INFRA.md](INFRA.md) |

Основной поток: **камера → Camera Gateway → Recognition → Backend → Admin**. Backend хранит конфигурацию, результаты и артефакты; отдельный training worker обучает модели, а Ollama/Qwen3-VL может проверять кандидаты активностей.
