FROM node:20.9.0-alpine as build 

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

CMD ["npx", "-y", "http-server", "-p", "3000", "/app/build"]

EXPOSE 3000
