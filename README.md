## API Gateway (практическая реализация)

### Описание решения

Реализован API Gateway на базе **NGINX**, который обеспечивает:

- Маршрутизацию запросов к соответствующим микросервисам
- Проверку JWT токенов через сервис аутентификации
- Проксирование запросов к MinIO для скачивания файлов
- Единую точку входа для всех сервисов

### Архитектура

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│    NGINX    │ ← API Gateway (порт 80)
│  (gateway)  │
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  security   │ │  uploader   │ │   storage   │
│   (Flask)   │ │   (Node.js) │ │   (MinIO)   │
│   порт 3000 │ │   порт 3000 │ │   порт 9000 │
└─────────────┘ └─────────────┘ └─────────────┘
```

### Технологии

| Компонент    | Технология        | Назначение                          |
|--------------|-------------------|-------------------------------------|
| Gateway      | NGINX 1.29        | Маршрутизация, проверка токенов     |
| Security     | Flask + JWT       | Аутентификация, выдача/проверка JWT |
| Uploader     | Node.js + MinIO   | Приём и сохранение изображений      |
| Storage      | MinIO             | S3-совместимое хранилище            |
| Хранение     | Docker volumes    | Персистентность данных              |

### Запуск проекта

#### 1. Клонирование репозитория

```bash
git clone <repository-url>
cd micros-gateway
```

#### 2. Структура проекта

```
micros-gateway/
├── docker-compose.yml
├── gateway/
│   └── nginx.conf
├── uploader/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   └── src/
│       └── server.js
├── security/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── src/
│       └── server.py
└── .env (опционально)
```

#### 3. Запуск контейнеров

```bash
docker compose up --build
```

Все сервисы автоматически поднимутся с правильными зависимостями.

### Проверка работоспособности

#### 1. Получение JWT токена

```bash
curl -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/v1/token
```

**Ответ:** строка с JWT токеном

```json
"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I"
```

#### 2. Загрузка изображения

```bash
curl -X POST \
  -H "Authorization: Bearer <TOKEN>" \
  -H 'Content-Type: octet/stream' \
  --data-binary @image.jpg \
  http://localhost/v1/upload
```

**Ответ:** JSON с именем сохранённого файла

```json
{"filename":"550e8400-e29b-41d4-a716-446655440000.jpg"}
```

#### 3. Скачивание изображения

```bash
curl -X GET \
  -H "Authorization: Bearer <TOKEN>" \
  http://localhost/v1/user/550e8400-e29b-41d4-a716-446655440000.jpg \
  --output downloaded.jpg
```

**Результат:** файл `downloaded.jpg` успешно скачан

```bash
$ file downloaded.jpg
downloaded.jpg: JPEG image data, JFIF standard 1.01
```

#### 4. Проверка отказа без токена

```bash
curl -X POST -H 'Content-Type: octet/stream' \
  --data-binary @image.jpg \
  http://localhost/v1/upload
```

**Ответ:** 401 Unauthorized

```json
{"error":"Unauthorized"}
```

### Логирование и мониторинг

```bash
# Просмотр логов всех сервисов
docker compose logs -f

# Логи конкретного сервиса
docker compose logs gateway
docker compose logs security
docker compose logs uploader
docker compose logs storage
```

### Управление проектом

```bash
# Остановка всех контейнеров
docker compose down

# Остановка с удалением томов (очистка данных)
docker compose down -v

# Перезапуск конкретного сервиса
docker compose restart gateway

# Просмотр статуса
docker compose ps
```
### Переменные окружения (.env)

```env
Storage_AccessKey=minioadmin
Storage_Secret=minioadmin
Storage_Bucket=data
```

### Результаты тестирования

| Тест | Ожидаемый код | Результат |
|------|---------------|-----------|
| POST /v1/token с валидными данными | 200 | ✅ JWT получен |
| POST /v1/upload с валидным токеном | 200 | ✅ Файл сохранён |
| POST /v1/upload без токена | 401 | ✅ Ошибка авторизации |
| GET /v1/user/{filename} с токеном | 200 | ✅ Файл скачан |
| GET /v1/user/{filename} без токена | 401 | ✅ Ошибка авторизации |
| Content-Type загружаемого файла | image/* | ✅ Проверка MIME |
