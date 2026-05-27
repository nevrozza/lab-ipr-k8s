# Сетевая связность и структура баз данных

В данном разделе подробно описаны механизмы сетевого взаимодействия микросервисов, организация схемы хранения данных и процесс устранения коллизий при автоматических миграциях.

## 1. Сетевое взаимодействие

Все компоненты приложения обмениваются трафиком внутри изолированного пространства имен `messager-app`.

| Имя службы | Порт | Назначение |
| :--- | :--- | :--- |
| `frontend` | `80` | Обслуживание статических файлов веб-интерфейса |
| `bff` | `8080` | API-Gateway (шлюз), маршрутизирующий запросы к бэкенду |
| `user-service` | `8081` | Микросервис управления пользователями и авторизации |
| `message-service` | `8082` | Микросервис отправки сообщений, чатов и работы с файлами |
| `postgres` | `5432` | Централизованная база данных PostgreSQL |
| `minio` | `9000` | S3-совместимое объектное хранилище |

---

## 2. Структура таблиц базы данных `messager`

База данных инициализируется автоматически при запуске кластера с помощью Job-контейнеров на базе утилиты `goose`. В ходе отладки структуры таблиц были приведены в соответствие с ожиданиями кода приложений:

### Таблица `users`
```sql
CREATE TABLE users (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Таблица `messages`
```sql
CREATE TABLE messages (
    id VARCHAR(255) PRIMARY KEY,
    sender_id VARCHAR(255) NOT NULL,
    receiver_id VARCHAR(255) NOT NULL,
    text TEXT,
    file_id VARCHAR(255),
    file_name VARCHAR(255),
    edited BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Таблица `files`
```sql
CREATE TABLE files (
    id VARCHAR(255) PRIMARY KEY,
    orig_name VARCHAR(255) NOT NULL,
    content_type VARCHAR(255) NOT NULL,
    path VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Отладка и решение проблем

### Коллизия таблиц миграций Goose
* **Проблема:** Оба микросервиса (`user-service` и `message-service`) делят одну базу данных `messager`. По умолчанию утилита `goose` создает одну таблицу `goose_db_version` для отслеживания версий. При запуске джоба `migrate-messages` записывала версию `1` в таблицу. Запускавшаяся следом джоба `migrate-users` видела версию `1` уже примененной и пропускала создание таблицы `users`, из-за чего сервис пользователей падал с ошибкой `500`.
* **Решение:** В конфигурацию Job-ов была добавлена переменная окружения `GOOSE_TABLE`. Это позволило разделить таблицы истории миграций на `goose_users_version` и `goose_messages_version`, полностью изолировав процессы обновления баз данных друг от друга.
