FROM node:20.9.0-alpine as build

WORKDIR /app

COPY frontend/package*.json ./

RUN npm install

COPY frontend /app
RUN npm run build

FROM nginx:1.25-alpine

RUN rm /etc/nginx/conf.d/default.conf

COPY nginx/nginx.conf /etc/nginx/conf.d/

COPY --from=build /app/build /usr/share/nginx/html
