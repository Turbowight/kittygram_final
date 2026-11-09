![Main Kittygram workflow](https://github.com/Turbowight/kittygram_final/actions/workflows/main.yml/badge.svg)

# Kittygram

Kittygram — социальная сеть для любителей котиков. Пользователи регистрируются,
добавляют своих котов с фотографией, цветом, годом рождения и достижениями,
смотрят котиков других пользователей.

Проект запускается в Docker-контейнерах, а тестирование, сборка образов,
деплой на сервер и уведомление в Telegram выполняются автоматически
через GitHub Actions при каждом пуше в ветку `main`.

## Технологии

- Python 3.12, Django 5.1, Django REST Framework, Djoser
- PostgreSQL 13
- Gunicorn
- Nginx
- React (фронтенд)
- Docker, Docker Compose
- GitHub Actions (CI/CD), Docker Hub

## Архитектура

| Контейнер  | Образ                 | Назначение                                              |
|------------|-----------------------|---------------------------------------------------------|
| `db`       | `postgres:13`         | База данных                                             |
| `backend`  | `kittygram_backend`   | Django + Gunicorn, порт 8000                            |
| `frontend` | `kittygram_frontend`  | Собирает React и копирует сборку в volume `static`      |
| `gateway`  | `kittygram_gateway`   | Nginx: раздаёт статику и медиа, проксирует `/api/` и `/admin/` |

Volumes:

- `pg_data` — данные PostgreSQL;
- `static` — статика фронтенда и бэкенда (админка, DRF);
- `media` — файлы, загруженные пользователями.

## Локальный запуск в контейнерах

1. Клонируйте репозиторий:

   ```bash
   git clone https://github.com/Turbowight/kittygram_final.git
   cd kittygram_final
   ```

2. Создайте `.env` по образцу `.env.example` и заполните значения:

   ```bash
   cp .env.example .env
   ```

3. Соберите и запустите контейнеры:

   ```bash
   docker compose up --build
   ```

4. В другом терминале выполните миграции и соберите статику бэкенда:

   ```bash
   docker compose exec backend python manage.py migrate
   docker compose exec backend python manage.py collectstatic --noinput
   docker compose exec backend cp -r /app/collected_static/. /backend_static/static/
   ```

5. Откройте http://localhost:9000. Создать администратора:
   `docker compose exec backend python manage.py createsuperuser`.

## Деплой на сервер

1. Установите на сервер Docker и Docker Compose.
2. Создайте на сервере папку `~/kittygram` и файл `~/kittygram/.env`
   (образец — `.env.example`). Обязательно укажите `DOCKERHUB_USERNAME`,
   `ALLOWED_HOSTS` с доменом и IP сервера и `CSRF_TRUSTED_ORIGINS`.
3. Настройте внешний Nginx на сервере, чтобы запросы к домену Kittygram
   шли в контейнер `gateway` (порт 9000):

   ```nginx
   server {
       server_name kittygram.example.com;
       client_max_body_size 20M;

       location / {
           proxy_set_header Host $http_host;
           proxy_pass http://127.0.0.1:9000;
       }
   }
   ```

   Затем проверьте конфигурацию, перезапустите Nginx и получите
   SSL-сертификат:

   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   sudo certbot --nginx
   ```

4. Дальше всё делает GitHub Actions: при пуше в `main` файл
   `docker-compose.production.yml` копируется на сервер, образы обновляются,
   выполняются миграции и сбор статики.

## CI/CD (GitHub Actions)

Workflow `.github/workflows/main.yml` (его копия — `kittygram_workflow.yml`):

1. `tests` — проверка кода бэкенда линтером ruff и тесты Django на PostgreSQL;
2. `frontend_tests` — тесты фронтенда;
3. `build_*_and_push_to_docker_hub` — сборка и публикация образов
   `kittygram_backend`, `kittygram_frontend`, `kittygram_gateway`;
4. `deploy` — обновление проекта на сервере;
5. `send_message` — уведомление в Telegram.

Секреты репозитория (Settings → Secrets and variables → Actions):

| Секрет            | Значение                                   |
|-------------------|--------------------------------------------|
| `DOCKER_USERNAME` | логин на Docker Hub                        |
| `DOCKER_PASSWORD` | пароль или access token Docker Hub         |
| `HOST`            | IP-адрес сервера                           |
| `USER`            | имя пользователя на сервере                |
| `SSH_KEY`         | закрытый SSH-ключ для доступа к серверу    |
| `SSH_PASSPHRASE`  | пароль от SSH-ключа (если есть)            |
| `TELEGRAM_TO`     | ID вашего Telegram-аккаунта                |
| `TELEGRAM_TOKEN`  | токен Telegram-бота                        |

## Автотесты Яндекс Практикума

1. Заполните `tests.yml` в корне проекта своими данными
   (`repo_owner`, `kittygram_domain`, `taski_domain`, `dockerhub_username`, `vm_name`).
2. Создайте виртуальное окружение, установите зависимости и запустите тесты:

   ```bash
   python -m venv venv
   source venv/bin/activate        # Windows: source venv/Scripts/activate
   pip install -r backend/requirements.txt
   pytest
   ```

## Автор

[Turbowight](https://github.com/Turbowight)
