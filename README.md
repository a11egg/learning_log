# Learning Log

Учебный проект курса «Python & Django»: дневник обучения. Пользователь регистрируется,
заводит темы («что изучаю») и добавляет к ним записи («что узнал»), с датой и
редактированием. Каждый видит только свои темы и записи.

## Стек

| Компонент | Версия / роль |
| --- | --- |
| Python | 3.12 |
| Django | 5.2 LTS |
| База данных | SQLite (`db.sqlite3`) |
| Оформление | django-bootstrap4 (Bootstrap 4, CSS/JS с CDN) |
| Прод-сервер | Gunicorn |
| Статика | WhiteNoise |
| Запуск | Docker Compose |

## Структура проекта

```text
learning-log/
├── manage.py
├── learning_log/              # настройки проекта
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── learning_logs/             # приложение с темами и записями
│   ├── models.py              # Topic, Entry
│   ├── forms.py               # TopicForm, EntryForm
│   ├── views.py               # index, topics, topic, new_topic, new_entry, edit_entry
│   ├── urls.py
│   ├── tests.py               # 4 теста доступа к данным
│   ├── migrations/            # 0001_initial, 0002_entry, 0003_topic_owner
│   └── templates/learning_logs/
│       ├── base.html          # общий шаблон с Bootstrap
│       ├── index.html
│       ├── topics.html
│       ├── topic.html
│       ├── new_topic.html
│       ├── new_entry.html
│       └── edit_entry.html
├── users/                     # регистрация, вход, выход
│   ├── urls.py
│   ├── views.py
│   └── templates/registration/
│       ├── login.html
│       ├── logged_out.html
│       └── register.html
├── Dockerfile
├── compose.yaml
├── requirements.txt
├── .env.example
├── .dockerignore
└── .gitignore
```

## Быстрый старт (Docker)

```sh
cp .env.example .env      # Windows: Copy-Item .env.example .env
docker compose up -d --build
docker compose ps          # контейнер должен быть Up (healthy)
```

Приложение: <http://127.0.0.1:8000/>

Создать администратора для `/admin/`:

```sh
docker compose exec web python manage.py createsuperuser
```

Логи, остановка:

```sh
docker compose logs -f web
docker compose down          # база в томе learning_log_data сохраняется
docker compose down -v       # удалить и базу
```

## Запуск без Docker (виртуальное окружение)

Нужен Python 3.12.

```sh
python3.12 -m venv ll_env
source ll_env/bin/activate          # Windows: .\ll_env\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

## Тесты

```sh
docker compose exec web python manage.py test
```

Ожидаемый результат: `Ran 4 tests` и `OK`. Проверяются:

- гость перенаправляется на страницу входа;
- владелец открывает свою тему;
- другой пользователь получает 404;
- `GET /users/logout/` возвращает 405 и не завершает сеанс (в Django 5 выход выполняется POST-формой).

## Переменные окружения

Файл `.env` содержит настройки конкретного сервера и в Git не попадает. Перечень
переменных — в `.env.example`.

| Переменная | Назначение | Пример |
| --- | --- | --- |
| `DJANGO_SECRET_KEY` | секретный ключ Django | длинная случайная строка |
| `DJANGO_DEBUG` | режим отладки | `False` на сервере |
| `DJANGO_ALLOWED_HOSTS` | разрешённые адреса через запятую | `example.com,localhost,127.0.0.1` |
| `DJANGO_CSRF_TRUSTED_ORIGINS` | доверенные источники для POST | `https://example.com` |
| `DJANGO_USE_HTTPS` | `True`, если приложение открывается по HTTPS | `False` |
| `SQLITE_PATH` | путь к базе внутри контейнера | `/app/data/db.sqlite3` |
| `APP_BIND_IP` | адрес публикации порта | `127.0.0.1` за прокси, `0.0.0.0` при прямом доступе |
| `APP_PORT` | внешний порт приложения | `8200` |

Сгенерировать секретный ключ:

```sh
python -c "import secrets; print(secrets.token_urlsafe(64))"
```

## Развёртывание на Ubuntu-сервере

Требуется Docker с плагином Compose. Схема: контейнер слушает порт на хосте,
снаружи его закрывает обратный прокси (Caddy, Nginx, туннель), который и держит HTTPS.

1. Склонировать репозиторий:

   ```sh
   git clone https://github.com/<ваш-логин>/learning_log.git ~/learning-log
   cd ~/learning-log
   ```

2. Создать `.env` рядом с `compose.yaml`:

   ```dotenv
   DJANGO_SECRET_KEY=свой-длинный-ключ
   DJANGO_DEBUG=False
   DJANGO_ALLOWED_HOSTS=example.com,localhost,127.0.0.1
   DJANGO_CSRF_TRUSTED_ORIGINS=https://example.com
   DJANGO_USE_HTTPS=True
   SQLITE_PATH=/app/data/db.sqlite3
   APP_BIND_IP=127.0.0.1
   APP_PORT=8200
   ```

   `APP_BIND_IP=127.0.0.1` — когда порт наружу публикует прокси на том же хосте.
   `APP_BIND_IP=0.0.0.0` — когда домен пробрасывается на порт извне (NAT, VPN, туннель).

3. Запустить и проверить:

   ```sh
   docker compose up -d --build
   docker compose ps
   docker compose logs --tail=50 web
   curl -I http://127.0.0.1:8200/
   docker compose exec web python manage.py createsuperuser
   ```

4. Обновление после `git push`:

   ```sh
   git pull
   docker compose up -d --build
   ```

   Миграции и `collectstatic` выполняются автоматически при старте контейнера.

### Домен и HTTPS

Если приложение открывается по HTTPS через прокси:

- `DJANGO_ALLOWED_HOSTS` — домен и локальные адреса;
- `DJANGO_CSRF_TRUSTED_ORIGINS` — полный адрес со схемой, `https://example.com`;
- `DJANGO_USE_HTTPS=True` — включает `Secure` для cookie сессии и CSRF;
- прокси должен передавать `X-Forwarded-Proto: https` (настройка
  `SECURE_PROXY_SSL_HEADER` в `settings.py` уже включена).

После правки `.env` примените настройки: `docker compose up -d`.

### Учебный сервер курса (`newproject`)

Если деплой идёт на общий сервер с командой `newproject`, последовательность такая:

```sh
newproject <фамилия>-learning-log     # печатает каталог и первый порт диапазона
cd ~/projects/<фамилия>-learning-log
git init -q
git remote add origin https://github.com/<логин>/learning_log.git
git pull origin main
```

Дальше заполните существующий `.env` (строки, созданные `newproject`, не удаляйте),
укажите `APP_BIND_IP=0.0.0.0` и первый порт диапазона, затем
`docker compose up -d --build`.

На общем сервере оставьте один процесс Gunicorn. В `compose.yaml`:

```yaml
    command:
      - sh
      - -c
      - |
        set -e
        python manage.py migrate --noinput
        python manage.py collectstatic --noinput
        exec gunicorn learning_log.wsgi:application \
          --bind 0.0.0.0:8000 \
          --workers 1 \
          --threads 4 \
          --timeout 60
```

и в `Dockerfile`:

```dockerfile
CMD ["gunicorn", "learning_log.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "1", "--threads", "4", "--timeout", "60"]
```

## Частые проблемы

| Симптом | Причина и решение |
| --- | --- |
| `port is already allocated` | занят `APP_PORT`; выбрать свободный порт |
| `Bad Request (400)` | адреса нет в `DJANGO_ALLOWED_HOSTS` — добавьте домен и IP |
| `CSRF verification failed` | в `DJANGO_CSRF_TRUSTED_ORIGINS` нужен полный адрес со схемой и портом |
| не выполняется вход по HTTPS | `DJANGO_USE_HTTPS=True` и заголовок `X-Forwarded-Proto` от прокси |
| `no such table: ...` | не применены миграции: `docker compose exec web python manage.py showmigrations` |
| `No module named 'bootstrap4'` | пакета нет в `requirements.txt`; пересобрать образ |
| контейнер `unhealthy` | смотреть `docker compose logs web` |
| изменения не видны | код копируется при сборке: `docker compose up -d --build` |

## Что не попадает в репозиторий

`.env`, `db.sqlite3`, `ll_env/`, `__pycache__/`, `staticfiles/`.
Список переменных для настройки сервера — в `.env.example` и в разделе выше.
