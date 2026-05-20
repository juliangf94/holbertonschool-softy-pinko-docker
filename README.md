# holbertonschool-softy-pinko-docker

A Docker infrastructure project built incrementally across 7 tasks. The final setup runs a Flask API backend, an Nginx frontend, and an Nginx reverse proxy with horizontal scaling via Docker Compose.

## Project Structure

```
.
├── task0/   # Hello World Docker image
├── task1/   # Flask API backend
├── task2/   # Nginx static frontend
├── task3/   # Flask API with CORS + dynamic frontend content
├── task4/   # Docker Compose: backend + frontend
├── task5/   # Docker Compose: backend + frontend + reverse proxy
└── task6/   # Horizontal scaling with --scale
```

## Tasks

### Task 0 — Ubuntu Hello World
A minimal Docker image based on `ubuntu:latest` that prints `Hello, World!` on startup.

```bash
docker build -t softy-pinko:task0 ./task0
docker run softy-pinko:task0
```

### Task 1 — Flask API
A Flask API running on port `5252` with a single endpoint `GET /api/hello`.

```bash
docker build -t softy-pinko:task1 ./task1
docker run -p 5252:5252 -it softy-pinko:task1
curl http://localhost:5252/api/hello
```

### Task 2 — Nginx Static Frontend
An Nginx container serving the Softy Pinko static site on port `9000`.

```bash
docker build -f ./task2/front-end/Dockerfile -t softy-pinko-front-end:task2 ./task2/front-end
docker run -p 9000:9000 -it softy-pinko-front-end:task2
curl http://localhost:9000/
```

### Task 3 — CORS + Dynamic Content
The Flask API gains CORS support via `flask-cors`. The frontend fetches `GET /api/hello` via AJAX and renders the response in the page.

```bash
# Backend
docker build -f ./task3/back-end/Dockerfile -t softy-pinko-back-end:task3 ./task3/back-end
docker run -p 5252:5252 -it softy-pinko-back-end:task3

# Frontend
docker build -f ./task3/front-end/Dockerfile -t softy-pinko-front-end:task3 ./task3/front-end
docker run -p 9000:9000 -it softy-pinko-front-end:task3
```

### Task 4 — Docker Compose
Both services are orchestrated with Docker Compose. The frontend depends on the backend.

```bash
docker compose -f ./task4/docker-compose.yml build
docker compose -f ./task4/docker-compose.yml up
curl http://localhost:5252/api/hello
curl http://localhost:9000/
docker compose -f ./task4/docker-compose.yml down
```

### Task 5 — Reverse Proxy
An Nginx proxy sits in front of both services on port `80`. It routes `/api` requests to the backend and all other requests to the frontend. Neither backend nor frontend expose ports directly to the host.

```bash
docker compose -f ./task5/docker-compose.yml build
docker compose -f ./task5/docker-compose.yml up
curl http://localhost:80/api/hello
curl http://localhost:80/
docker compose -f ./task5/docker-compose.yml down
```

### Task 6 — Horizontal Scaling
The backend is scaled to 2 instances using `--scale`. The Nginx proxy load-balances between them using round-robin via Docker's internal DNS.

```bash
docker compose -f ./task6/docker-compose.yml build
docker compose -f ./task6/docker-compose.yml up --scale back-end=2
curl http://localhost:80/api/hello  # run multiple times to see round-robin in logs
docker compose -f ./task6/docker-compose.yml down
```

## Requirements

- Docker Engine 20.10+
- Docker Compose v2
