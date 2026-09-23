# Backend API

Актуализировано 2026-09-23 по `backend/src/index.ts`, маршрутам `backend/src/modules/` и `backend/prisma/schema.prisma`.

`backend` — Express + TypeScript API на порту `3000`. Prisma хранит данные в PostgreSQL, Redis/BullMQ обслуживает очереди и межсервисные сигналы, Socket.IO отправляет обновления клиентам. Загруженные фотографии, видео для обучения, модели и evidence лежат в `uploads`, а не в Git.

## Данные

- Организации и доступ: `Company`, `User`, `Employee`, `EmployeePhoto`, `Camera`.
- Посещаемость: `Observation`, `Event`, `EmployeePresence`.
- Активности: `Activity`, `CompanyActivity`, назначения сотрудникам, `ActivityResult`, `ActivityInterval`.
- Модели и обучение: `ModelVersion`, `TrainingAsset`, `TrainingAnnotation`, `TrainingJob`; также classifier groups и их версии моделей и задания.
- Разметка реальных фрагментов потока: `CaptureClip` с привязкой к компании, камере и активности.

## Основные API

| Группа | Назначение |
| --- | --- |
| `/api/auth` | Вход и обновление JWT |
| `/api/companies`, `/api/users`, `/api/employees`, `/api/cameras` | Справочники и настройки |
| `/api/observations`, `/api/events`, `/api/presence`, `/api/statistics` | Наблюдения, посещаемость и сводки |
| `/api/activities`, `/api/company-activities`, `/api/activity-results`, `/api/activity-intervals` | Активности и результаты |
| `/api/classifier-groups`, `/api/company-classifier-groups`, `/api/classifier-group-results` | Группы классификаторов |
| `/api/training-assets`, `/api/training-jobs`, `/api/classifier-group-training-jobs`, `/api/models` | Данные обучения, задания и версии моделей |
| `/api/captures` | Индекс клипов, кадры и разметка |
| `/api/recognition` | Конфигурация и задания для ML-сервиса |
| `/api/health` | Проверка состояния API |

Маршруты пользователя защищены JWT и проверками роли/компании. Вызовы Recognition к служебным маршрутам подписываются HMAC (`X-Signature`, `X-Timestamp`, `X-Request-Id`); Redis предотвращает повторное использование request ID. Конкретные методы и ограничения смотрите в файлах routes, не выводите их из названия группы.

## Потоки данных

1. Admin работает с backend через REST и Socket.IO.
2. Backend выдаёт URL потока `/streams/:id.mjpg?token=...` и вызывает Camera Gateway для проверки RTSP и перезапуска потока. Gateway сам читает камеры из PostgreSQL.
3. Recognition получает через подписанный API список камер, URL MJPEG, конфигурацию компании, сотрудников, назначения активностей и моделей.
4. Recognition отправляет наблюдения, события, интервалы и результаты в backend. Backend сохраняет их и публикует нужные realtime события в Admin.
5. Backend передаёт задания обучения отдельному `recognition-training-worker`; worker сообщает progress и загружает артефакты модели через служебный API.

## Хранение и конфигурация

Для production `BACKEND_UPLOADS_MOUNT` должен указывать на постоянный bind mount вне репозитория `backend`. База и `uploads` требуют резервной копии перед деплоем. Список переменных находится в [backend/ENV.md](https://github.com/aikeenaikeen/backend/blob/main/ENV.md) и [infra/ENV.md](https://github.com/aikeenaikeen/infra/blob/main/ENV.md).

Исходники: [регистрация маршрутов](https://github.com/aikeenaikeen/backend/blob/main/src/index.ts), [схема Prisma](https://github.com/aikeenaikeen/backend/blob/main/prisma/schema.prisma), [служебная авторизация](https://github.com/aikeenaikeen/backend/blob/main/src/middleware/serviceAuth.ts), [маршруты Recognition](https://github.com/aikeenaikeen/backend/blob/main/src/modules/recognition/recognition.routes.ts).
