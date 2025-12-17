# 📚 Infrastructure — Документация

## 🎯 Описание

**Infra** — Docker Compose конфигурация для запуска PostgreSQL базы данных.

## 🛠️ Технологический стек

- **Docker + Docker Compose**
- **PostgreSQL 15** (Alpine)

---

## 🔗 Взаимодействие с другими сервисами

### PostgreSQL Database (порт 5432)

Единая база данных для всей системы.

#### 1️⃣ Backend API → PostgreSQL

**Направление:** Backend → PostgreSQL (read/write)

- Prisma ORM
- Миграции + seed данных
- Управление всеми таблицами (Company, User, Employee, Camera, Event)

---

#### 2️⃣ Camera Gateway → PostgreSQL

**Направление:** Camera Gateway → PostgreSQL (read-only)

- Читает таблицу `Camera`
- Расшифровывает пароли для RTSP
- Синхронизированная Prisma схема (копия из Backend)

---

#### 3️⃣ Recognition Service → PostgreSQL

❌ **Не подключается напрямую** — работает через Backend API

---

## ⚙️ Connection String

```bash
postgresql://attendance_user:password@localhost:5432/attendance
```

**Порт:** 5432  
**База:** attendance  
**Пользователь:** attendance_user

---

## 📊 Диаграмма взаимодействия

```
┌─────────────────────────────────────────┐
│       PostgreSQL (порт 5432)            │
│         Database: attendance            │
│                                         │
│  Tables:                                │
│  • Company                              │
│  • User                                 │
│  • Employee (+ фото)                    │
│  • Camera (пароли AES-256)              │
│  • Event (IN/OUT)                       │
└──────────┬─────────────┬────────────────┘
           │             │
           │ read/write  │ read-only
           │             │
           ▼             ▼
    ┌──────────┐  ┌──────────────┐
    │ Backend  │  │   Camera     │
    │   API    │  │   Gateway    │
    └──────────┘  └──────────────┘
           │
           │ REST API
           │
           ▼
    ┌──────────────┐
    │ Recognition  │
    │   Service    │
    └──────────────┘
```

---

## 📋 Требуемые зависимости

| Сервис | Обязательный |
|--------|--------------|
| **Docker + Docker Compose** | ✅ Да |

---

*Документация актуальна на: декабрь 2025*

