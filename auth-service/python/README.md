# Сервис авторизации (Python / FastAPI)

## Описание

REST API для регистрации и аутентификации пользователей. Использует **FastAPI**, **SQLAlchemy (async)** и **PostgreSQL**. Токены — **JWT (HS256)**.

## API

| Метод | Путь        | Описание                              | Авторизация |
|-------|-------------|---------------------------------------|-------------|
| GET   | `/health`   | Проверка работоспособности сервиса    | Нет         |
| POST  | `/register` | Регистрация нового пользователя       | Нет         |
| POST  | `/login`    | Вход, получение JWT access_token      | Нет         |
| GET   | `/me`       | Данные текущего пользователя по токену| Bearer JWT  |

### Примеры запросов

```bash
# Регистрация
curl -X POST http://auth.42.sirius/register \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "secret123", "email": "alice@example.com"}'

# Вход
curl -X POST http://auth.42.sirius/login \
  -F "username=alice" \
  -F "password=secret123"
# Ответ: {"access_token": "eyJ...", "token_type": "bearer"}

# Проверка токена
curl http://auth.42.sirius/me \
  -H "Authorization: Bearer eyJ..."
```

## Сборка и запуск

### Локально (для разработки)

```bash
cd auth-service/python

# Создание виртуального окружения
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Переменные окружения (или создайте .env файл)
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=authdb
export DB_USER=authuser
export DB_PASSWORD=password
export SECRET_KEY=my-dev-secret-key

uvicorn app:app --reload --port 8000
```

Документация Swagger: http://localhost:8000/docs

### Сборка Docker-образа

```bash
cd auth-service/python

docker build -t your-registry/auth-service:latest .

# Тест с локальным PostgreSQL
docker run -p 8000:8000 \
  -e DB_HOST=host.docker.internal \
  -e DB_USER=authuser \
  -e DB_PASSWORD=password \
  -e DB_NAME=authdb \
  -e SECRET_KEY=test-secret \
  your-registry/auth-service:latest
```

### Публикация в реестр

```bash
docker push your-registry/auth-service:latest
```

После этого укажите полное имя образа в `kubernetes/manifests/auth-service/deployment.yaml`.
