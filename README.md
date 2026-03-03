# Demo Web + CI/CD (GitLab)

Небольшой Flask-сервис с полным CI/CD пайплайном в GitLab:
lint -> test -> build -> scan -> push -> deploy (staging).

## Что делает сервис

- `GET /` возвращает JSON с именем сервиса и версией:
  `{"service":"demo-web","version":"..."}`
- `GET /health` возвращает `ok`

`version` берется из переменной окружения `APP_VERSION` (по умолчанию `dev`).

## Стек

- Python 3.12
- Flask
- Pytest + Flake8
- Docker / Docker Compose
- GitLab CI/CD
- Trivy (SCA и image scan)

## Структура проекта

```text
.
├── app.py                    # Flask приложение
├── requirements.txt
├── Dockerfile
├── .gitlab-ci.yml            # CI/CD пайплайн
├── deploy/
│   ├── docker-compose.yml    # запуск контейнера на staging
│   └── remote-deploy.sh      # скрипт деплоя на удаленной VM
└── tests/
    ├── conftest.py
    └── test_app.py
```

## Локальный запуск

### Вариант 1: напрямую через Python

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --no-cache-dir -r requirements.txt
python app.py
```

Приложение стартует на `http://localhost:8080`.

### Вариант 2: через Docker

```bash
docker build -t demo-web:local --build-arg APP_VERSION=local .
docker run --rm -p 8080:8080 demo-web:local
```

## Проверки качества локально

```bash
pip install --no-cache-dir -r requirements.txt
flake8 app.py tests
python -m pytest -q
```

## CI/CD пайплайн

Пайплайн в `.gitlab-ci.yml` выполняет:

1. `lint` - проверка кода `flake8`
2. `test` - unit-тесты `pytest`
3. `build_image` - сборка Docker-образа с тегами:
   - `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA`
   - `$CI_REGISTRY_IMAGE:staging`
4. `trivy_scan` - скан репозитория и Docker-образа на HIGH/CRITICAL
5. `push_image` - публикация образов в GitLab Container Registry
6. `deploy_staging` - деплой на staging по SSH (только для ветки `main`)

## Обязательные CI/CD переменные (GitLab)

Настройте в `Settings -> CI/CD -> Variables`:

- `STAGING_HOST` - хост staging VM (например `1.2.3.4`)
- `STAGING_SSH_PORT` - SSH-порт (например `22`)
- `STAGING_USER` - пользователь SSH (например `ubuntu`)
- `STAGING_PATH` - директория проекта на VM (например `/opt/demo-web`)
- `STAGING_URL` - URL стенда (например `http://1.2.3.4:8080`)
- `SSH_PRIVATE_KEY` - приватный SSH-ключ для доступа к VM
- `REGISTRY_USER` - логин Deploy Token для Container Registry
- `REGISTRY_PASSWORD` - пароль Deploy Token для Container Registry

## Требования к staging VM

- Установлены Docker и Docker Compose
- Пользователь `STAGING_USER` имеет права запускать Docker
- Открыт порт `8080` (или нужный порт для доступа к сервису)

## Как работает деплой

При пуше в `main` job `deploy_staging`:

1. Подключается к VM по SSH
2. Копирует `deploy/docker-compose.yml` и `deploy/remote-deploy.sh`
3. Генерирует `.env` с `IMAGE=$CI_REGISTRY_IMAGE:staging`
4. На VM выполняет:
   - `docker login` в GitLab Registry
   - `docker compose pull`
   - `docker compose up -d --remove-orphans`
   - `docker image prune -f`
