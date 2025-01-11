# Итоговая работа по предмету "Основы виртуализации"
За основу был взят проект "Taski" 
## Taski
Это приложение создано для планирования своих задач.

### Работали на платформах:
* Docker;
* Kubernetes.

## Реализация проекта:

### 1. Клонируем репозиторий проекта "Taski"

``` git clone git@github.com:yandex-praktikum/taski.git ```

### 2. Создаём Dockerfile для frontend и backend

* В папке backend создаём Dockefile — файл для описания сборки Docker-образа:
``` FROM python:3.12-alpine ``` - базовый образ для сборки, аlpine - для облегчения веса образа, multi-stage build
``` WORKDIR /app ``` - устанавливаем рабочую директорию внутри контейнера на /app
``` COPY backend/requirements.txt . ``` - копируем файл requirements.txt из локальной директории backend в текущую рабочую директорию /app контейнера
``` RUN pip install --no-cache-dir -r requirements.txt ``` - запускаем установку зависимостей, указанных в requirements.txt, флаг --no-cache-dir запрещает кэширование установленных пакетов, чтобы уменьшить размер образа
``` COPY backend/ /app/ ``` - копируем содержимое локальной папки backend в рабочую директорию /app внутри контейнера
``` CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"] ``` - устанавливаем команду, которая выполняется при запуске контейнера (python manage.py runserver 0.0.0.0:8000 запускает встроенный сервер разработки для Django; 0.0.0.0:8000 указывает серверу слушать запросы на порту 8000)
``` RUN python manage.py migrate ``` - выполняем миграции базы данных через Django-команду manage.py migrate


* В папке frontend создаём Dockefile:





![](ссылка на ресурс)