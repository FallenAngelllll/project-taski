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

* В директории backend создаём Dockefile — файл для описания сборки Docker-образа:<br />

``` FROM python:3.12-alpine ``` - базовый образ для сборки, аlpine - для облегчения веса образа, multi-stage build<br />
``` WORKDIR /app ``` - устанавливаем рабочую директорию внутри контейнера на /app<br />
``` COPY backend/requirements.txt . ``` - копируем файл requirements.txt из директории backend в текущую рабочую директорию /app контейнера<br />
``` RUN pip install --no-cache-dir -r requirements.txt ``` - запускаем установку зависимостей, указанных в requirements.txt, флаг --no-cache-dir запрещает кэширование установленных пакетов, чтобы уменьшить размер образа<br />
``` COPY backend/ /app/ ``` - копируем содержимое папки backend в рабочую директорию /app внутри контейнера<br />
``` CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"] ``` - устанавливаем команду, которая выполняется при запуске контейнера (python manage.py runserver 0.0.0.0:8000 запускает встроенный сервер разработки для Django; 0.0.0.0:8000 указывает серверу слушать запросы на порту 8000)<br />
``` RUN python manage.py migrate ``` - выполняем миграции базы данных через Django-команду manage.py migrate<br />


* В директории frontend создаём Dockefile — файл для описания сборки Docker-образа:<br />

``` FROM node:20.9.0-alpine as build ``` - первый этап сборки. Базовый образ для сборки, аlpine - для облегчения веса образа, multi-stage build<br />
``` WORKDIR /app ``` - устанавливаем рабочую директорию внутри контейнера на /app<br />
``` COPY frontend/package*.json ./ ``` - копируем файлы package.json и package-lock.json из директории frontend в текущую рабочую директорию (/app) внутри контейнера<br />
``` RUN npm install ``` - выполняем команду npm install внутри контейнера<br />
``` COPY frontend/public ./public ``` - копируем содержимое папки frontend/public в /app/public внутри контейнера<br />
``` COPY frontend/src ./src ``` - копируем содержимое папки frontend/src в /app/src внутри контейнера<br />
``` RUN npm run build ``` - запускаем скрипт build, определенный в package.json<br />
``` FROM nginx:1.25-alpine ``` - второй этап сборки. Базовый образ nginx:1.25-alpine.<br />
``` RUN rm /etc/nginx/conf.d/default.conf ``` - удаляем файл конфигурации Nginx по умолчанию<br />
``` COPY nginx/nginx.conf /etc/nginx/conf.d/ ``` - копируем файл nginx.conf из локальной папки nginx в директорию конфигурации Nginx внутри контейнера<br />
``` COPY --from=build /app/build /usr/share/nginx/html ``` - копируем папку /app/build, созданную на этапе build, в директорию /usr/share/nginx/html внутри контейнера. Эта директория, откуда Nginx обслуживает статические файлы. Таким образом, приложение, собранное на первом этапе, становится доступным через веб-сервер Nginx<br /> 





![](ссылка на ресурс)
