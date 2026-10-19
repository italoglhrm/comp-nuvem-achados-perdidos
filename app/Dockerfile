FROM python:3.12-slim 

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd --system --no-create-home appuser \
    && mkdir -p /app/uploads \
    && chown -R appuser:appuser /app/uploads 

USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]