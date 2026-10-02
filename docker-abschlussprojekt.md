api/app.py
```
import os
import psycopg2
from flask import Flask

app = Flask(__name__)

def connect():
    return psycopg2.connect(
        host=os.environ["DB_HOST"],
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASSWORD"],
    )

with connect() as conn, conn.cursor() as cur:
    cur.execute("""CREATE TABLE IF NOT EXISTS besuche (
        id serial PRIMARY KEY,
        zeit timestamptz DEFAULT now())""")

@app.get("/")
def index():
    with connect() as conn, conn.cursor() as cur:
        cur.execute("INSERT INTO besuche DEFAULT VALUES")
        cur.execute("SELECT count(*) FROM besuche")
        return f"Besuche: {cur.fetchone()[0]}\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

api/requirements.txt
```
flask==3.0.3
psycopg2-binary==2.9.9
```

api/Dockerfile
```
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
RUN useradd -m appuser
USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
```

api/.dockerignore
```
__pycache__/
.env
```

docker-compose.yaml
```
services:
  api:
    build: ./api
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      DB_NAME: ${POSTGRES_DB}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  adminer:
    image: adminer:4
    ports:
      - "8081:8080"
    depends_on:
      - db
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
      
volumes:
  pgdata:
```

.env
```
POSTGRES_DB=workshop
POSTGRES_USER=workshop
POSTGRES_PASSWORD=bitte-aendern
```
