FROM node:20.9.0-alpine as build

WORKDIR /app

COPY frontend/package*.json ./

RUN npm install

COPY . .

RUN npm run build
