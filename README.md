# Итоговая работа по предмету<br /> "Основы виртуализации" 
За основу был взят проект "Taski" 
## Taski
Это приложение создано для планирования своих задач.

### Работали на платформах:
* Docker;
* Kubernetes;
* Yandex.Cloud.

## Реализация проекта:

### 1. Клонируем репозиторий проекта "Taski" 

``` git clone git@github.com:yandex-praktikum/taski.git ```

### 2. Разворачиваем проект через Docker compose

* В директории backend создаём **Dockefile** — файл для описания сборки Docker-образа:<br />

``` FROM python:3.12-alpine ``` - базовый образ для сборки, аlpine - для облегчения веса образа, multi-stage build<br />
``` WORKDIR /app ``` - устанавливаем рабочую директорию внутри контейнера на /app<br />
``` COPY backend/requirements.txt . ``` - копируем файл requirements.txt из директории backend в текущую рабочую директорию /app контейнера<br />
``` RUN pip install --no-cache-dir -r requirements.txt ``` - запускаем установку зависимостей, указанных в requirements.txt, флаг --no-cache-dir запрещает кэширование установленных пакетов, чтобы уменьшить размер образа<br />
``` COPY backend/ /app/ ``` - копируем содержимое папки backend в рабочую директорию /app внутри контейнера<br />
``` CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"] ``` - устанавливаем команду, которая выполняется при запуске контейнера (python manage.py runserver 0.0.0.0:8000 запускает встроенный сервер разработки для Django; 0.0.0.0:8000 указывает серверу слушать запросы на порту 8000)<br />
``` RUN python manage.py migrate ``` - выполняем миграции базы данных через Django-команду manage.py migrate<br />


* В директории frontend создаём **Dockefile** — файл для описания сборки Docker-образа:<br />

``` FROM node:20.9.0-alpine as build ``` - первый этап сборки. Базовый образ для сборки, аlpine - для облегчения веса образа, multi-stage build<br />
``` WORKDIR /app ``` - устанавливаем рабочую директорию внутри контейнера на /app<br />
``` COPY frontend/package*.json ./ ``` - копируем файлы package.json и package-lock.json из директории frontend в текущую рабочую директорию (/app) внутри контейнера<br />
``` RUN npm install ``` - выполняем команду npm install внутри контейнера<br />
``` COPY frontend/public ./public ``` - копируем содержимое папки frontend/public в /app/public внутри контейнера<br />
``` COPY frontend/src ./src ``` - копируем содержимое папки frontend/src в /app/src внутри контейнера<br />
``` RUN npm run build ``` - запускаем скрипт build, определенный в package.json<br />
``` FROM nginx:1.25-alpine ``` - второй этап сборки. Базовый образ nginx:1.25-alpine<br />
``` RUN rm /etc/nginx/conf.d/default.conf ``` - удаляем файл конфигурации Nginx по умолчанию<br />
``` COPY nginx/nginx.conf /etc/nginx/conf.d/ ``` - копируем файл nginx.conf из локальной папки nginx в директорию конфигурации Nginx внутри контейнера<br />
``` COPY --from=build /app/build /usr/share/nginx/html ``` - копируем папку /app/build, созданную на этапе build, в директорию /usr/share/nginx/html внутри контейнера. Эта директория, откуда Nginx обслуживает статические файлы. Таким образом, приложение, собранное на первом этапе, становится доступным через веб-сервер Nginx<br /> 


* Создадим **директорию nginx** и в ней файл конфигурации **nginx.conf**:<br />

``` server { ``` - блок, описывающий настройки конкретного сервера<br />
  ``` listen 80; ``` - HTTP-запросы на порту 80<br />
  ``` root /usr/share/nginx/html/; ``` - корневая директория для статических файлов<br />
  
  ``` location /api/ { ``` - блок настроек для запросов, начинающихся с /api/<br />
    ``` proxy_set_header Host $http_host; ``` - устанавливаем заголовок Host для запроса<br />
    ``` proxy_pass http://taski_backend:8000/api/; ``` - запросы на /api/ будут перенаправляться на backend<br />
  ``` } ```<br />
  
  ``` location /api/tasks/ { ``` - блок настроек для запросов, начинающихся с /api/tasks/<br />
    ``` proxy_set_header Host $http_host; ```<br />
    ``` proxy_pass http://taski_backend:8000/api/tasks/; ``` - запросы на /api/tasks/ будут перенаправляться на backend<br />
  ``` } ```<br />
  
  ``` location /admin/ { ``` - блок настроек для запросов, начинающихся с /admin/<br />
    ``` proxy_set_header Host $http_host; ```<br />
    ``` proxy_pass http://taski_backend:8000/admin/; ``` - запросы на /admin/ будут перенаправляться на backend<br />
  ``` } ```<br />
  
  ``` location / { ``` - блок для всех остальных запросов, обрабатывает запросы к маршрутам фронтенда<br />
    ``` root /usr/share/nginx/html; ``` <br />
    ``` index index.html; ``` - устанавливаем файл по умолчанию (index.html), который будет раздаваться при обращении к корневому маршруту /<br />
    ``` try_files $uri /index.html; ``` - проверяем, существует ли запрашиваемый файл ($uri). Если файл не найден, перенаправляет запрос на index.html<br />
  ``` } ```<br />
``` } ```<br />


* В корневой директории проекта создадим файл **docker-compose.yml** (файл описывает многоконтейнерное приложение, состоящее из трех сервисов):<br />

``` services: ```<br />  
  ``` backend: ```<br />
    ``` build: ```<br />
      ``` context: . ``` - контекст сборки, используется текущая директория<br />
      ``` dockerfile: ./backend/Dockerfile ``` - путь к Dockerfile для сборки образа backend<br />
    ``` container_name: taski_backend ``` - имя контейнера<br />
    ``` command: ["python", "manage.py", "runserver", "0.0.0.0:8000"] ``` - команда, которая будет выполнена при запуске контейнера<br />
    ``` ports: ```<br />
      ``` - "8000:8000" ```<br />
    ``` environment: ``` - список переменных окружения, которые передаются в контейнер<br />
      ``` - DEBUG=True ``` - включает режим отладки в Django<br />
      ``` - ALLOWED_HOSTS=* ``` - разрешает доступ с любого хоста<br />

  ``` frontend: ```<br />
    ``` build: ```<br />
      ``` context: . ```<br />
      ``` dockerfile: ./frontend/Dockerfile ``` <br />
    ``` container_name: taski_frontend ```<br />
    ``` ports: ```<br />
      ``` - "3000:80" ```<br />

  ``` nginx: ```<br />
    ``` image: nginx:1.25-alpine ``` - указываем образ для Nginx<br />
    ``` container_name: taski_nginx ```<br />
    ``` ports: ```<br />
      ``` - "80:80" ```<br />
    ``` depends_on: ``` - указываем зависимости для сервиса<br />
      ``` - backend ```<br />
      ``` - frontend ```<br />

* Запустим Docker Desktop, откроем терминал и перейдем в корневую директора проекта

* Собёрем проект командой: ` docker-compose up --build `

* После успешной сборки можем проверить работу:

- [x] на порте 3000: http://localhost:3000

![1](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/1.png?raw=true)

![2](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/2.png?raw=true)

- [x] на порте 8000: http://localhost:8000/api/tasks

![3](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/3.png?raw=true)

- [x] также создадим еще задачу и проверим: http://localhost:3000 и http://localhost:8000/api/tasks

![4](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/4.png?raw=true)

![5](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/5.png?raw=true)

![6](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/6.png?raw=true)

* Вес образов:

![вес образов](https://github.com/FallenAngelllll/project-taski/blob/main/image/docker%20compose/%D0%B2%D0%B5%D1%81%20%D0%BE%D0%B1%D1%80%D0%B0%D0%B7%D0%BE%D0%B2.png?raw=true)

* Чтобы остановить запущенные контейнеры введите команду: ` docker-compose down `

### 3. Разворачиваем проект через Kubernetes (K8s)


