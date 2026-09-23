# Admin Frontend

Актуализировано 2026-09-23 по `admin/src/router/index.ts`, `admin/src/components/CameraStreamDialog.vue` и backend API.

`admin` — Vue 3 + TypeScript SPA на Vite. Интерфейс использует Element Plus, Pinia, Axios, Vue Router, vue-i18n и Socket.IO client. В Compose контейнер слушает порт `8080`; при включённом профиле `prod` Nginx публикует UI на `80/443`.

## Страницы и доступ

| Маршрут | Назначение | Доступ |
| --- | --- | --- |
| `/login` | Вход | Без авторизации |
| `/dashboard` | Сводка | Авторизованные пользователи |
| `/employees` | Сотрудники, фотографии и назначения | Авторизованные пользователи |
| `/cameras` | Камеры, обычный и распознанный поток | Авторизованные пользователи |
| `/presence` | Текущее присутствие | Авторизованные пользователи |
| `/statistics` | События, интервалы активностей, evidence и статистика | Авторизованные пользователи |
| `/labeling` | Разметка сохранённых клипов | `SUPERADMIN`, `COMPANY_ADMIN` |
| `/companies`, `/users`, `/activities` | Управление компаниями, пользователями, активностями и обучением | `SUPERADMIN` |

`/events` и `/employee-activities` перенаправляют на `/statistics`. Точные права на операции проверяет backend; наличие страницы само по себе не даёт права на запись.

## Связь с сервисами

- REST API и Socket.IO идут через backend. `VITE_API_BASE_URL` задаёт адрес API; если он пуст, используется origin страницы. Axios добавляет Bearer access token и один раз обновляет его при `401`.
- Обычный просмотр камеры: Admin запрашивает `GET /api/cameras/:id/stream-url`, получает `mjpegUrl` с токеном доступа и открывает MJPEG `/streams/:id.mjpg` через Camera Gateway.
- Просмотр с распознаванием: `<img>` открывает единый endpoint Recognition `GET /video_feed?cameraId=:id`. Адрес берётся из `VITE_RECOGNITION_STREAM_URL` либо из origin страницы. Отдельных портов `5000 + cameraId` больше нет.
- Статистика использует, среди прочего, `GET /api/activity-intervals`; сохранённые клипы и их кадры доступны через `/api/captures`. Обновления событий и интервалов приходят через Socket.IO.

## Где смотреть детали

- [Маршруты и guards](https://github.com/aikeenaikeen/admin/blob/main/src/router/index.ts)
- [Клиент API](https://github.com/aikeenaikeen/admin/blob/main/src/api/client.ts)
- [Просмотр камеры](https://github.com/aikeenaikeen/admin/blob/main/src/components/CameraStreamDialog.vue)
- [Конфигурация Admin](https://github.com/aikeenaikeen/admin/blob/main/ENV.md)
- [Backend API](BACKEND_DOCUMENTATION.md)
