# Образ kittygram_backend: Django + Gunicorn
FROM python:3.12-slim

# Не создавать .pyc и сразу выводить логи (без буферизации)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Сначала копируем только зависимости: этот слой кешируется
# и пересобирается лишь при изменении requirements.txt
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install -r requirements.txt --no-cache-dir

# Затем копируем код проекта
COPY . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "kittygram_backend.wsgi"]
