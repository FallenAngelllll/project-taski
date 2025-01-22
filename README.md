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


* В корневой директории создадим **директорию nginx** и в ней файл конфигурации **nginx.conf**:<br />

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

* Запустим Docker Desktop, откроем терминал и перейдем в корневую директорию проекта

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

* В корневой директории создаём файл **k8s.yaml** — файл описывает ресурс Kubernetes, который управляет развертыванием проекта; он определяет, как и какие контейнеры запускать в кластере:<br />

``` apiVersion: apps/v1 ``` - API-версия Kubernetes<br />
``` kind: Deployment ``` - тут указано, что мы описываем ресурс вида Deployment<br />
``` metadata: ```  <br />
  ``` name: taski-deployment ```<br />
``` spec: ``` - блок спецификации развертывания<br />
  ``` replicas: 1 ``` - свойство объекта спецификаций развёртывания, которое задаёт то, сколько экземпляров (реплик) подов нужно запустить<br />
  ``` selector: ``` - указываем селектор для поиска подов, которые принадлежат этому развертыванию<br />
    ``` matchLabels: ```<br />
      ``` app: taski ``` - метка для подов<br />
  ``` template: ``` - этот объект задаёт шаблон пода, который описываемый ресурс Deployment будет использовать для создания новых подов<br />
    ``` metadata: ```<br />
      ``` labels: ```<br />
        ``` app: taski ```<br />
    ``` spec: ```<br />
      ``` containers: ```<br />
      ``` - name: backend ```<br />
        ``` image: taski-backend:v1 ```<br />
        ``` ports: ```<br />
        ``` - containerPort: 8000 ```<br />
      ``` - name: frontend ```<br />
        ``` image: taski-frontend:v1 ```<br />
        ``` ports: ```<br />
        ``` - containerPort: 3000 ```<br />
      ``` - name: nginx ```<br />
        ``` image: nginx:1.25-alpine ```<br />
        ``` ports: ```<br />
        ``` - containerPort: 80 ```<br />

* В корневой директории создаём файл **service.yaml** — файл описывает ресурс Kubernetes, который управляет развертыванием проекта; он определяет, как и какие контейнеры запускать в кластере:<br />

``` apiVersion: v1 ```<br />
``` kind: Service ``` - тут указано, что мы описываем ресурс вида Service<br />
``` metadata: ```<br />
  ``` name: taski-service ```<br />
``` spec: ```<br />
  ``` selector: ```<br />
    ``` app: taski ```<br />
  ``` ports: ```<br />
  ``` - name: nginx ```<br />
    ``` port: 8080 ```<br />
    ``` targetPort: 8080 ```<br />
  ``` - name: frontend ```<br />
    ``` port: 3000 ```<br />
    ``` targetPort: 3000 ```<br />
  ``` - name: backend ```<br />
    ``` port: 8000 ```<br />
    ``` targetPort: 8000 ```<br />

* Немного изменим файл конфигурации **nginx.conf** (изменения выделены жирным):<br />

``` server { ```<br />
  **``` listen 8080; ```<br />**
  ``` root /usr/share/nginx/html/; ```<br />

  ``` location /api/ { ```<br />
    ``` proxy_set_header Host $http_host; ```<br />
    **``` proxy_pass http://taski-service:8000/api/;  ```<br />**
  ``` } ```<br />

  ``` location /api/tasks/ { ```<br />
    ``` proxy_set_header Host $http_host; ```<br />
    **``` proxy_pass http://taski-service:8000/api/tasks/; ```<br />**
  ``` } ```<br />

  ``` location /admin/ { ```<br />
    ``` proxy_set_header Host $http_host; ```<br />
    **``` proxy_pass http://taski-service:8000/admin/; ```<br />**
  ``` } ```<br />

  ``` location / { ```<br />
    ``` root /usr/share/nginx/html; ```<br />
    ``` index index.html; ```<br />
    ``` try_files $uri /index.html; ```<br />
  ``` } ```<br />
``` } ```<br />

В Docker имя хоста taski_backend указывает на контейнер с бэкендом. Docker автоматически создает общую сеть, где контейнеры могут взаимодействовать друг с другом по своим именам. 
В Kubernetes вместо контейнеров напрямую используются сервисы для общения между компонентами. В Kubernetes нельзя напрямую обращаться к подам или контейнерам, как в Docker. 
Вместо этого используется сервис как точка входа. Поэтому мы заменили taski_backend (контейнер) на taski-service.

* Установка и запуск Minikube

Для установки Minikube следуйте указаниям, которые можно найти в [документации](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download "Ссылка"). В процессе установки Minikube вы также установите Kubectl. 
Это — клиент, который позволяет выполнять запросы к API-серверу Kubernetes.<br />

Для запуска Minikube выполните команду: ` minikube start `<br />

![шаг1](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%201.png?raw=true)

* Загрузим образы в кластер, чтобы их можно было использовать в Kubernetes:

Команда для **Windows (PowerShell)**: ` & minikube -p minikube docker-env --shell powershell | Invoke-Expression `<br />
Команда для **Linux/MacOS**: ` eval $(minikube -p minikube docker-env) `<br />

Создание образа для backend: ` docker build -t taski-backend:v1 -f backend/Dockerfile . `<br />

![шаг2](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%202.png?raw=true)

Создание образа для frontend: ` docker build -t taski-frontend:v1 -f frontend/Dockerfile . `<br />

![шаг3](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%203.png?raw=true)

* Применение манифестов для развертывания приложения:

Разворачиваем поды с бэкендом, фронтендом и Nginx с помощью объекта Deployment: ` kubectl apply -f k8s.yaml `<br />
Настраиваем сервис для маршрутизации запросов к развернутым подам: ` kubectl apply -f service.yaml `<br />

![шаг4](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%204.png?raw=true)

* Проверка состояния развернутых объектов Kubernetes

Посмотрим список всех подов в кластере, их текущий статус и состояние (Running, Pending или CrashLoopBackOff): ` kubectl get pods `<br />
Посмотрим список всех сервисов в кластере и их настройки (порты и внешние IP-адреса): ` kubectl get svc `<br />

![шаг5](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%205.png?raw=true)

* Открытие сервиса в браузере через Minikube

Введите команду: ` minikube service taski-service `<br />

![шаг6](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/step%206.png?raw=true)

* После успешного развёртывания можем проверить работу:

- [x] по адресу (порт будет меняться): http://127.0.0.1:57915

![1](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/1.png?raw=true)

![2](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/2.png?raw=true)

- [x] также создадим еще задачу и проверим, по адресам (порт будет меняться): http://127.0.0.1:57915 и http://127.0.0.1:57917/api/tasks

![3](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/3.png?raw=true)

![4](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/4.png?raw=true)

![5](https://github.com/FallenAngelllll/project-taski/blob/main/image/Kubernetes%20(K8s)/5.png?raw=true)

* Остановка и удаление кластера Minikube:

Остановим запущенный кластер Minikube: ` minikube stop `<br />
Полностью удалим кластер Minikube: ` minikube delete `

### 3. Автоматизируем развертку docker через CI/CD

* На облачной платформе Yandex.Cloud создадим виртуальную машину, а также реестр, где будут храниться docker-образы

![1](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/%D0%B2%D0%BC.png?raw=true)

* Установим docker внутри виртуальной машины:

Введите команду: ` sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin `<br />

* В корневой директории создаём папку **.github/workflows** и в ней файл **deploy.yml**:<br />

``` name: CI/CD Deploy Taski ``` - название workflow<br />
 
``` on: ```<br />
  ``` push: ```<br />
    ``` branches: ```<br />
    ```   - main ``` - настраиваем автоматический запуск CI/CD при пуше в ветку main<br />
    
``` jobs: ```<br />
  ``` build: ```<br />
    ``` runs-on: ubuntu-latest ``` - определяем окружение, где будут выполняться шаги workflow<br />
    ``` steps: ```<br />
      ``` - uses: actions/checkout@v2 ``` - загружаем файлы репозитория в рабочую директорию для дальнейшей работы<br />
      ``` - name: Login to Yandex.Cloud Container Registry ``` - необходимо для того, чтобы пушить образы Docker в YCR<br />
      ```   id: login-cr ```<br />
      ```   uses: yc-actions/yc-cr-login@v1 ```<br />
      ```   with: ```<br />
        ```   yc-sa-json-credentials: ${{ secrets.AUTHORIZATION }} ``` <br />

      ``` - name: Build and push Docker image to Yandex.Cloud Container Registry ``` - публикация Docker-образов в реестре, чтобы их можно было использовать при развёртывании на сервере<br />
        ``` env: ```<br />
        ```   CR_REGISTRY: ${{ secrets.CR_REGISTRY }} ```<br />
        ```   IMAGE_TAG: ${{ github.sha }} ```<br />
      ```   run: | ```<br />
        ```   docker build -t cr.yandex/$CR_REGISTRY/frontend:$IMAGE_TAG ./frontend/ ```<br />
        ```   docker push cr.yandex/$CR_REGISTRY/frontend:$IMAGE_TAG ```<br />
        ```   docker build -t cr.yandex/$CR_REGISTRY/backend:$IMAGE_TAG ./backend/ ```<br />
        ```   docker push cr.yandex/$CR_REGISTRY/backend:$IMAGE_TAG ```<br />
        ```   docker build -t cr.yandex/$CR_REGISTRY/nginx:$IMAGE_TAG ./nginx/ ```<br />
        ```   docker push cr.yandex/$CR_REGISTRY/nginx:$IMAGE_TAG ``` <br />
      
      ``` - name: Connect to VPC-Server ``` - - подключение к удалённому серверу<br />
      ```   uses: appleboy/ssh-action@master ```<br />
      ```   env: ```<br />
        ```   OAUTH_TOKEN: ${{ secrets.TOKEN }} ```<br />
        ```   CR_REGISTRY: ${{ secrets.CR_REGISTRY }} ```<br />
        ```   IMAGE_TAG: ${{ github.sha }} ```<br />
        ``` with: ```<br />
        ```   host: 84.201.171.27 ```<br />
        ```   username: julie ```<br />
        ```   key: ${{ secrets.SSH_KEY }} ```<br />
        ```   port: 22 ``` <br />
        ```   script: |  ``` - выполнение команд на сервере<br />
          ```   echo ${{ secrets.TOKEN }}|docker login --username oauth --password-stdin cr.yandex ```<br />
          ```   docker kill $(docker ps -q) &> /dev/null ```<br />
          ```   docker rmi -f $(docker images -qa) ```<br />
          ```   docker system prune --all --force ```<br />
          ```   docker run -p 3000:3000 -d cr.yandex/${{ secrets.CR_REGISTRY }}/frontend:${{ github.sha }} ```<br />
          ```   docker run -p 8000:8000 -d cr.yandex/${{ secrets.CR_REGISTRY }}/backend:${{ github.sha }} ```<br />
          ```   docker run -p 80:80 --net=host -d cr.yandex/${{ secrets.CR_REGISTRY }}/nginx:${{ github.sha }} ```<br />

* После успешного развёртывания можем проверить работу:

- [x] по адресу: http://84.201.171.27

![1](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/1.png?raw=true)

![2](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/2.png?raw=true)

- [x] по адресу: http://84.201.171.27/api/tasks

![3](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/3.png?raw=true)

- [x] также создадим еще задачу и проверим, по адресам: http://84.201.171.27 и http://84.201.171.27/api/tasks

![4](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/4.png?raw=true)

![5](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/5.png?raw=true)

![6](https://github.com/FallenAngelllll/project-taski/blob/main/image/CICD/6.png?raw=true)

* После развертывания проекта остановим виртуальную машину и удалим docker-образы из реестра (YCR).