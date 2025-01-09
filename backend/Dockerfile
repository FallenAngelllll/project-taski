FROM python:3.12-alpine

WORKDIR /app

COPY backend/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY backend /app

# Запуск миграций перед стартом приложения
CMD ["sh", "-c", "python manage.py migrate && gunicorn taski_backend.wsgi:application --bind 0.0.0.0:8000"]
