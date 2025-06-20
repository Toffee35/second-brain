# Запрос
Помоги мне составить докер композ для всего этого... (подразумеваеться что все будет запускаться пока что на одной машине)
Postgres + PostGIS
ClickHouse
Redis + AOF
Nginx
Frontend на NodeJS сервере
CoreRESTfullAPI на FastAPI
TelegramBotGateway на Teloxide
RabbitMQ
TimerScheduler на Celery Beat
TaskWorker на Celery Worker

Со всеми хорошими практиками (допустим зависимости для запуска контенеров и профили, как пример прод профиль нужен сразу для запуска уже рабочего, а дев будет дополнительно содержать adminer для посмотра postgres, а так же лучше блокировать версии для всего)
# Ответ
Отличная задача! Создание грамотного `docker-compose.yml` — это фундамент для стабильной и удобной разработки и эксплуатации. Я подготовил для вас полный набор файлов с учетом всех ваших пожеланий и лучших практик.

### 1. Структура проекта

Для начала, вот рекомендуемая структура папок. Это поможет держать все в порядке.

```
/my-project
├── docker-compose.yml
├── .env
├── core-api/
│   └── Dockerfile
├── telegram-gateway/
│   └── Dockerfile
├── frontend/
│   └── Dockerfile
├── worker/
│   └── Dockerfile
└── nginx/
    ├── nginx.conf
    └── certs/
        ├── dummy.crt  (для локальной разработки)
        └── dummy.key
```

*   **worker/**: Это папка для кода Celery Worker и Celery Beat. Они часто используют общую кодовую базу (определения задач), поэтому логично держать их вместе.

### 2. Файл `.env` (Секреты и конфигурация)

Создайте этот файл в корне проекта. **Никогда не добавляйте его в Git!** Вместо него можно добавить `.env.template` с пустыми значениями.

```ini
# Идентификатор проекта для docker-compose
COMPOSE_PROJECT_NAME=my_project

# Настройки PostgreSQL
POSTGRES_DB=main_db
POSTGRES_USER=user
POSTGRES_PASSWORD=strongpassword

# Настройки RabbitMQ
RABBITMQ_DEFAULT_USER=user
RABBITMQ_DEFAULT_PASS=strongpassword

# Настройки Redis (пароль опционален, но рекомендуется)
REDIS_PASSWORD=strongpassword

# URL для подключения Celery к брокеру
CELERY_BROKER_URL=amqp://user:strongpassword@rabbitmq:5672//

# URL для бэкенда Celery (хранение результатов, опционально)
# Можно использовать Redis или Postgres
CELERY_RESULT_BACKEND=redis://:strongpassword@redis:6379/0

# Настройки вашего API и Бота
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
# ... другие секреты
```

### 3. Файл `docker-compose.yml`

Вот "сердце" вашей системы. Я добавил комментарии, объясняющие каждый шаг.

```yaml
version: '3.9'

# Имя проекта, чтобы контейнеры имели префикс my_project_*
name: ${COMPOSE_PROJECT_NAME:-my_project}

services:
  # === БАЗЫ ДАННЫХ И ХРАНИЛИЩА ===
  postgres:
    image: postgis/postgis:15-3.4 # Версия с PostGIS
    container_name: postgres
    profiles: ["prod"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - default

  clickhouse:
    image: clickhouse/clickhouse-server:23.8-alpine # Легковесная версия Alpine
    container_name: clickhouse
    profiles: ["prod"]
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    environment:
      # Пароль для пользователя default
      - CLICKHOUSE_DEFAULT_USER=user
      - CLICKHOUSE_DEFAULT_PASSWORD=${POSTGRES_PASSWORD} # Можно использовать свой
      - CLICKHOUSE_DB=logs_db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123/ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - default

  redis:
    image: redis:7.2-alpine
    container_name: redis
    profiles: ["prod"]
    # Включаем AOF-персистентность для надежности
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - default

  # === БРОКЕР СООБЩЕНИЙ ===
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    profiles: ["prod"]
    ports:
      # 15672 - для веб-интерфейса управления (доступен на http://localhost:15672)
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=${RABBITMQ_DEFAULT_USER}
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_DEFAULT_PASS}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running", "-q"]
      interval: 30s
      timeout: 10s
      retries: 5
    restart: unless-stopped
    networks:
      - default

  # === БЭКЕНД И ЛОГИКА ===
  core-api:
    build: ./core-api
    container_name: core-api
    profiles: ["prod"]
    env_file: .env
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      clickhouse: { condition: service_healthy }
    restart: unless-stopped
    networks:
      - default

  telegram-gateway:
    build: ./telegram-gateway
    container_name: telegram-gateway
    profiles: ["prod"]
    env_file: .env
    depends_on:
      core-api: { condition: service_started } # Зависит от API
    restart: unless-stopped
    networks:
      - default

  # === ФОНОВЫЕ ЗАДАЧИ (CELERY) ===
  worker:
    build: ./worker
    container_name: task-worker
    profiles: ["prod"]
    command: celery -A app.main worker --loglevel=info
    env_file: .env
    volumes:
      - ./worker:/app # Для удобной разработки (hot-reload)
    depends_on:
      rabbitmq: { condition: service_healthy }
      postgres: { condition: service_healthy }
      clickhouse: { condition: service_healthy }
      redis: { condition: service_healthy }
    restart: unless-stopped
    networks:
      - default

  scheduler:
    build: ./worker # Используем тот же образ, что и для worker
    container_name: timer-scheduler
    profiles: ["prod"]
    command: celery -A app.main beat --loglevel=info
    env_file: .env
    volumes:
      - ./worker:/app
    depends_on:
      rabbitmq: { condition: service_healthy }
    restart: unless-stopped
    networks:
      - default

  # === ФРОНТЕНД И ШЛЮЗ ===
  frontend:
    build: ./frontend
    container_name: frontend
    profiles: ["prod"]
    restart: unless-stopped
    networks:
      - default

  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    profiles: ["prod"]
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - core-api
      - frontend
    restart: always
    networks:
      - default

  # === ИНСТРУМЕНТЫ ДЛЯ РАЗРАБОТКИ ===
  adminer:
    image: adminer:4.8.1
    container_name: adminer
    profiles: ["dev"] # Этот сервис запустится только с профилем 'dev'
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    restart: unless-stopped
    networks:
      - default

# Именованные тома для хранения данных
volumes:
  postgres_data:
  clickhouse_data:
  redis_data:
  rabbitmq_data:

# Изолированная сеть для нашего приложения
networks:
  default:
    name: ${COMPOSE_PROJECT_NAME}_network
```

### 4. Примерные конфигурации

#### `nginx/nginx.conf`
Это базовый пример для перенаправления запросов.

```nginx
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    sendfile on;

    upstream frontend_server {
        server frontend:3000; # Порт, на котором работает ваш NodeJS сервер
    }

    upstream api_server {
        server core-api:8000; # Порт, на котором работает FastAPI
    }

    server {
        listen 80;
        server_name localhost;

        # Перенаправляем все запросы к API на core-api
        location /api/ {
            proxy_pass http://api_server/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Все остальные запросы отдаем фронтенду
        location / {
            proxy_pass http://frontend_server;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }
    }
}
```

#### Dockerfile для `worker/` (примерный)

```dockerfile
# Используйте тот же базовый образ, что и для core-api для консистентности
FROM python:3.11-slim

WORKDIR /app

# Устанавливаем зависимости
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем исходный код
COPY . .

# Команда для запуска будет передана из docker-compose.yml
```

### 5. Как это использовать

1.  **Заполните `.env`**: Скопируйте содержимое из шаблона выше в файл `.env` и впишите свои реальные секреты.
2.  **Создайте Dockerfile'ы**: Положите соответствующие Dockerfile в папки `core-api`, `telegram-gateway`, `frontend`, `worker`.
3.  **Запуск для продакшена (prod):**
    ```bash
    # Эта команда запустит все сервисы, у которых указан профиль "prod"
    docker compose --profile prod up --build -d
    ```
4.  **Запуск для разработки (dev):**
    ```bash
    # Эта команда запустит ВСЕ сервисы: и "prod", и "dev"
    docker compose --profile "*" up --build -d
    # Или более явно
    # docker compose --profile prod --profile dev up --build -d
    ```
    После этого Adminer будет доступен на `http://localhost:8080`, где вы сможете подключиться к вашей базе `postgres`.

### Ключевые моменты и лучшие практики, которые здесь учтены:

*   **Блокировка версий**: Все образы (`postgres`, `redis` и т.д.) имеют четко указанную версию. Это предотвращает поломку сборки после обновления образа в Docker Hub.
*   **Профили (`profiles`)**: Четкое разделение на `prod` и `dev`. Вы не запускаете лишние инструменты (Adminer) в проде.
*   **Зависимости (`depends_on`)**: Используется "здоровая" зависимость (`condition: service_healthy`). Это значит, что, например, `core-api` не запустится, пока `postgres` не будет полностью готов принимать подключения, а не просто запущен.
*   **`.env` файл**: Все секреты и переменные окружения вынесены из `docker-compose.yml`. Это безопасно и удобно.
*   **Именованные тома (`volumes`)**: Данные баз хранятся в именованных томах Docker, а не на хост-машине напрямую. Это стандартный и более управляемый способ.
*   **Сеть (`networks`)**: Все контейнеры работают в одной изолированной сети, что повышает безопасность и позволяет им общаться по именам сервисов (`postgres`, `redis` и т.д.).
*   **Healthchecks**: Для ключевых сервисов прописаны проверки состояния. Это критически важно для `depends_on`.
*   **Разделение Celery**: Beat (планировщик) и Worker (исполнитель) вынесены в разные сервисы. Это правильная архитектура, позволяющая масштабировать их независимо.
# Запрос
Скажи как мне разделить сервисы на репозитории и как сделать планировщик (я ранее не работал и не знаю что такое celery)
# Ответ
Отлично, давайте разберем эти два важных вопроса по шагам. Это ключевые моменты для построения масштабируемой и удобной в поддержке системы.

### Часть 1: Разделение сервисов на репозитории

Есть два основных подхода: **монорепозиторий** и **мультирепозиторий (полирепозиторий)**.

#### Подход 1: Мультирепозиторий (Poly-repo) — один сервис = один репозиторий

Это самый классический и интуитивно понятный подход.

*   `repo-core-api`
*   `repo-telegram-gateway`
*   `repo-frontend`
*   `repo-worker` (для Celery Worker и Beat)
*   `repo-infrastructure` (или `repo-ops`, `repo-deployment`)

В последнем репозитории (`repo-infrastructure`) будет лежать ваш `docker-compose.yml`, `.env.template`, конфигурации `nginx` и т.д. — всё, что описывает, **как** вся система собирается и запускается вместе.

**Как это работает:**

1.  Каждая команда разработчиков работает в своем репозитории.
2.  Для каждого сервиса настроен свой CI/CD пайплайн, который по коммиту в `main` ветку:
    *   Тестирует код.
    *   Собирает Docker-образ (`my-company/core-api:v1.2.3`).
    *   Пушит образ в Docker Registry (например, Docker Hub, GitLab Registry, AWS ECR).
3.  Когда нужно обновить сервис на продакшене, вы идете в репозиторий `repo-infrastructure`, меняете в `docker-compose.yml` версию образа (`image: my-company/core-api:v1.2.4`) и выкатываете изменения.

**Плюсы:**
*   **Изоляция:** Четкие границы ответственности. Проблемы в одном репозитории не затрагивают другие.
*   **Независимое версионирование:** `core-api` может быть версии `2.5`, а `frontend` — `1.8`.
*   **Простота CI/CD:** Пайплайны для каждого сервиса просты и независимы.

**Минусы:**
*   **Сложность управления зависимостями:** Если вы меняете API в `core-api`, вам нужно пойти в `repo-frontend` и `repo-telegram-gateway` и обновить их код, чтобы он соответствовал новому API. Это может быть сложно координировать.
*   **Дублирование кода:** Общие библиотеки или модели данных придется выносить в отдельные пакеты и подключать как зависимости, что добавляет сложности.

#### Подход 2: Монорепозиторий (Monorepo) — все сервисы в одном репозитории

Этот подход набирает огромную популярность, особенно с инструментами типа Lerna (для JS) или Bazel/Pants (для больших систем).

Ваша структура проекта, которую я описал ранее, уже является заготовкой для монорепозитория.

```
/my-project (один Git репозиторий)
├── core-api/
├── telegram-gateway/
├── frontend/
├── worker/
└── deployment/
    ├── docker-compose.yml
    └── nginx/
```

**Плюсы:**
*   **Атомарные изменения:** Вы можете в одном коммите поменять и сервер (`core-api`), и клиента (`frontend`), который его использует. Все тесты запускаются вместе, и вы уверены, что ничего не сломали. Это **главное преимущество**.
*   **Простота переиспользования кода:** Общие библиотеки и типы данных можно просто импортировать из соседней папки.
*   **Единый CI/CD:** Один пайплайн, который может быть настроен так, чтобы пересобирать только измененные сервисы.

**Минусы:**
*   **Сложность настройки CI/CD:** Настроить пайплайн, который понимает, что изменилось, и пересобирает только нужные части, сложнее.
*   **Размер репозитория:** Со временем репозиторий и история Git могут стать очень большими.
*   **Требует дисциплины:** Все разработчики работают в одном месте, что требует более строгих правил именования веток и код-ревью.

**Рекомендация для вас:**
Начните с **монорепозитория**. Для команды небольшого и среднего размера его преимущества (особенно атомарные изменения) значительно перевешивают недостатки. Вы будете двигаться гораздо быстрее. Когда (и если) проект вырастет до огромных размеров, вы сможете подумать о разделении.

---

### Часть 2: Что такое Celery и как сделать планировщик (простыми словами)

Представьте, что у вас есть ресторан.

*   **Ваш `CoreAPI`** — это официант, который принимает заказы от клиентов.
*   **Кухня** — это место, где выполняются тяжелые задачи (готовятся блюда).

Если официант будет сам готовить каждое блюдо, он не сможет принимать новые заказы, и клиенты будут долго ждать. Это плохая система.

**Celery — это система управления кухней.**

Она состоит из трёх частей:

1.  **Брокер сообщений (RabbitMQ)** — это **система заказов на кухне (типа планшета или чеков на прищепках)**. Официант (`CoreAPI`) не идет на кухню сам. Он просто вбивает заказ в систему (`"Приготовить стейк"`) и сразу же возвращается к клиентам. Система заказов надежно хранит этот чек.

2.  **Воркер (Celery Worker)** — это **повар**. Он стоит на кухне, смотрит на систему заказов (RabbitMQ) и ждет новый чек. Как только чек появляется, повар берет его, готовит блюдо (выполняет вашу Python-функцию) и, когда закончит, отмечает чек как выполненный. Если поваров много, они могут разбирать чеки параллельно.

3.  **Планировщик (Celery Beat)** — это **менеджер ресторана**. У него есть блокнот с расписанием: "Каждый день в 12:00 заказать свежие овощи", "Каждый вечер в 23:00 провести инвентаризацию". Он сам не выполняет эту работу. Когда наступает время, он просто создает новый чек ("Провести инвентаризацию") и вешает его в общую систему заказов (RabbitMQ). А уже свободный повар (Worker) возьмет этот чек и выполнит его.

#### Как это выглядит в коде (концептуально)

Вы создаете папку `worker/`. В ней будет файл, например `tasks.py`.

```python
# worker/tasks.py

from celery import Celery
from celery.schedules import crontab
import time

# 1. Настраиваем подключение к RabbitMQ
#    URL берется из переменных окружения, как в docker-compose
app = Celery('tasks', broker=os.environ.get('CELERY_BROKER_URL'))


# 2. Описываем "рецепты" (задачи, которые может выполнять повар)

@app.task  # Эта декоратор превращает функцию в "задачу" для Celery
def process_log(log_data):
    # Здесь логика записи лога в ClickHouse
    print(f"Записываю лог в ClickHouse: {log_data}")
    time.sleep(1) # имитация работы
    return "Лог записан"

@app.task
def update_postgres_data():
    # Здесь логика подключения к Postgres и обновления данных
    print("Обновляю данные в PostgreSQL...")
    time.sleep(5) # имитация долгой задачи
    return "Данные в PG обновлены"

@app.task
def rewrite_redis_aof():
    print("Запускаю очистку Redis AOF...")
    return "Команда отправлена в Redis"


# 3. Описываем расписание для "менеджера" (Celery Beat)
#    Это расписание говорит Celery Beat, какие задачи создавать и когда.

app.conf.beat_schedule = {
    # Имя задачи, должно быть уникальным
    'update-postgres-daily': {
        'task': 'tasks.update_postgres_data', # Какую функцию-задачу вызывать
        'schedule': crontab(hour=0, minute=0), # Когда: каждый день в 00:00
    },
    'rewrite-redis-aof-daily': {
        'task': 'tasks.rewrite_redis_aof',
        'schedule': crontab(hour=0, minute=5), # Когда: каждый день в 00:05
    },
}
```

**Как это всё связывается:**

*   **Из `CoreAPI` вы вызываете задачу с логами:**
    ```python
    # В коде вашего FastAPI
    from worker.tasks import process_log

    # Вместо того чтобы выполнять функцию напрямую...
    # process_log({"message": "User logged in"}) 
    
    # ...вы просите Celery сделать это за вас.
    # Это моментально создает "чек" в RabbitMQ и не тормозит API.
    process_log.delay({"message": "User logged in"}) 
    ```

*   **Docker-compose запускает двух "сотрудников":**
    *   `celery ... worker`: Запускает **повара** (Worker), который ждет задачи в RabbitMQ. Он будет выполнять и `process_log`, и `update_postgres_data`, когда они появятся.
    *   `celery ... beat`: Запускает **менеджера** (Beat), который следит за временем и по расписанию кладет задачи `update_postgres_data` и `rewrite_redis_aof` в RabbitMQ.

Вот и всё! Вы разделили принятие решений (`CoreAPI`, `Celery Beat`) от выполнения тяжелой работы (`Celery Worker`), используя `RabbitMQ` как надежного посредника.
# Запрос
Как правильно делать докер образ для rust - teloxide, python - fastapi, celery worker, celery beat? Обучи по быстрому.
# Ответ
Конечно! Обучение созданию правильных Docker-образов — один из самых ценных навыков в современной разработке. Давайте сделаем это быстро и по делу, с упором на лучшие практики.

### Главные принципы хорошего Docker-образа

Неважно, какой язык вы используете, эти 4 принципа — основа всего:

1.  **Минимальный размер (Small Size):** Меньше образ -> быстрее скачивается, меньше места занимает, меньше уязвимостей. Достигается через:
    *   **Многостадийные сборки (Multi-stage builds):** Критически важно для компилируемых языков (Rust). Мы используем один "большой" контейнер для сборки, а потом копируем *только результат* (один файл) в крошечный финальный контейнер.
    *   **Правильный базовый образ:** Вместо `python:3.11` используйте `python:3.11-slim`. Вместо `ubuntu` — `debian:bookworm-slim`. Они содержат только необходимое для запуска.
    *   **Файл `.dockerignore`:** Не копировать в образ мусор (виртуальные окружения, кэш, файлы IDE).

2.  **Кэширование слоев (Layer Caching):** Docker собирает образ по слоям. Если вы меняете только свой код, Docker не должен каждый раз переустанавливать все зависимости. Структурируйте Dockerfile так, чтобы самые редко изменяемые инструкции были вверху.

3.  **Безопасность (Security):** Никогда не запускайте контейнер от `root`! Всегда создавайте отдельного пользователя с ограниченными правами.

4.  **Воспроизводимость (Reproducibility):** Закрепляйте версии базовых образов и зависимостей. Это предотвратит поломку сборки "на ровном месте".

---

### Файл `.dockerignore` (Общий для всех)

Создайте этот файл в корне вашего проекта. Он будет использоваться при сборке всех образов.

```
# Git
.git
.gitignore

# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.venv/
venv/
env/

# Rust
/target/

# IDE/OS specific
.idea/
.vscode/
*.swp
.DS_Store

# Environment files
.env
*.env
```

---

### 1. Docker-образ для Rust (Teloxide)

Здесь мы используем **многостадийную сборку**.

**`telegram-gateway/Dockerfile`**

```dockerfile
# --- СТАДИЯ 1: СБОРЩИК (Builder) ---
# Используем полный образ Rust для компиляции нашего приложения
FROM rust:1.78-slim as builder

WORKDIR /app

# Копируем только файлы зависимостей
COPY Cargo.toml Cargo.lock ./

# Секретный трюк для кэширования: создаем фиктивный проект и собираем
# только зависимости. Этот слой будет переиспользоваться, если вы меняете
# только код, а не Cargo.toml.
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release

# Теперь копируем остальной исходный код
COPY src ./src

# Собираем наше приложение, используя уже скачанные зависимости
RUN cargo build --release

# --- СТАДИЯ 2: ФИНАЛЬНЫЙ ОБРАЗ (Final Image) ---
# Используем минимальный образ Debian. Он содержит базовые библиотеки (libc),
# необходимые для запуска Rust-приложения.
FROM debian:bookworm-slim

# Создаем пользователя без прав root для безопасности
RUN useradd -m appuser

WORKDIR /home/appuser

# Копируем ТОЛЬКО скомпилированный бинарный файл из стадии сборщика
# Замените `telegram-gateway` на имя вашего исполняемого файла из Cargo.toml
COPY --from=builder /app/target/release/telegram-gateway .

# Убеждаемся, что наш пользователь является владельцем файла
RUN chown appuser:appuser telegram-gateway

# Переключаемся на нашего пользователя
USER appuser

# Команда для запуска нашего приложения
CMD ["./telegram-gateway"]
```

**Что мы сделали:**
*   Собрали проект в большом образе `rust:1.78-slim`.
*   Создали крошечный образ `debian:bookworm-slim`.
*   Скопировали в него один-единственный файл — результат компиляции.
*   Запускаем от имени `appuser`.

---

### 2. Docker-образ для Python (FastAPI)

Принцип очень похож, но без явной компиляции. Главное — кэширование `pip` зависимостей.

**`core-api/Dockerfile`**

```dockerfile
# Используем slim-версию Python для минимизации размера
FROM python:3.11-slim

# Устанавливаем переменные окружения
# 1. Говорит Python выводить логи сразу, а не буферизировать
ENV PYTHONUNBUFFERED=1
# 2. Запрещает Python создавать .pyc файлы
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

# Создаем пользователя без прав root
RUN addgroup --system appgroup && adduser --system --group appuser

# Копируем файл с зависимостями ОТДЕЛЬНО
COPY requirements.txt .

# Устанавливаем зависимости. Этот слой будет закэширован,
# если requirements.txt не меняется.
# --no-cache-dir уменьшает размер слоя.
RUN pip install --no-cache-dir -r requirements.txt

# Копируем весь остальной код приложения.
# Только если эти файлы изменятся, Docker начнет пересборку отсюда.
COPY . .

# Меняем владельца всех файлов на нашего пользователя
RUN chown -R appuser:appgroup /app

# Переключаемся на нашего пользователя
USER appuser

# Команда для запуска FastAPI через Uvicorn.
# --host 0.0.0.0 делает сервер доступным извне контейнера.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Что мы сделали:**
*   Правильно расположили `COPY` и `RUN` для максимального кэширования `pip install`.
*   Запускаем от имени `appuser`.
*   Используем `slim` образ.

---

### 3. Docker-образ для Celery (Worker и Beat)

**Хорошая новость:** Вам нужен **только один Dockerfile**. И Worker, и Beat используют одну и ту же кодовую базу. Они — просто разные "режимы запуска" одного и того же приложения. Разница будет только в команде запуска в `docker-compose.yml`.

Dockerfile будет почти идентичен образу для FastAPI.

**`worker/Dockerfile`**

```dockerfile
# Используем тот же базовый образ, что и для API, для консистентности
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

RUN addgroup --system appgroup && adduser --system --group appuser

# Здесь будут зависимости и для вашего кода, и для Celery
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем код воркера
COPY . .

RUN chown -R appuser:appgroup /app
USER appuser

# КОМАНДУ ЗАПУСКА (CMD) ЗДЕСЬ МОЖНО НЕ УКАЗЫВАТЬ.
# Мы всё равно переопределим её в docker-compose.yml для каждого сервиса (worker и scheduler).
# Если всё же указать, это будет команда по умолчанию.
# CMD ["celery", "-A", "app.main", "worker", "--loglevel=info"]
```

### Как это связывается в `docker-compose.yml`

Теперь вы видите, как элегантно это работает. В `docker-compose.yml` вы просто ссылаетесь на один и тот же образ, но с разными командами:

```yaml
...
services:
  # ... (api, db, и т.д.)

  worker: # Это Celery Worker
    build: ./worker
    # Эта команда переопределит любую CMD в Dockerfile
    command: celery -A app.main worker --loglevel=info
    # ... (env_file, depends_on, и т.д.)

  scheduler: # Это Celery Beat
    build: ./worker # Используем ТОТ ЖЕ образ!
    # А команда другая!
    command: celery -A app.main beat --loglevel=info
    # ... (env_file, depends_on, и т.д.)
...
```

Теперь у вас есть набор профессиональных, оптимизированных и безопасных Docker-образов для всего вашего стека.
