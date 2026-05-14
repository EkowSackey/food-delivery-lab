# Food Delivery — Microservices System

A microservices-based food delivery platform migrated from a monolithic architecture. The system is composed of six independently deployable services orchestrated via Docker Compose, communicating through a REST API gateway and a RabbitMQ message broker.

---

## Architecture overview

See [`docs/architecture.md`](docs/architecture.md) for the full service diagram and message flow documentation.

**Services:**

| Service | Port | Responsibility |
|---|---|---|
| API Gateway | 8080 | Single entry point — JWT validation, routing |
| Customer Service | 8081 | Registration, login, profile management |
| Restaurant Service | 8082 | Restaurant and menu management |
| Order Service | 8083 | Order lifecycle, event publishing |
| Delivery Service | 8084 | Delivery tracking, event consumption |
| Discovery Server | 8761 | Eureka service registry |

---

## Prerequisites

- Docker and Docker Compose (v2)
- Java 21 + Maven (only if building locally outside Docker)
- A local RabbitMQ **must not** be running on ports 5672 / 15672, or the port remappings in `docker-compose.yml` must be adjusted

---

## Quick start

### 1. Clone with submodules

```bash
git clone --recurse-submodules git@github-work:EkowSackey/food-delivery-lab.git
cd food-delivery-lab
```

If you already cloned without `--recurse-submodules`:
```bash
git submodule update --init --recursive
```

### 2. Create the environment file

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```
DB_USERNAME=postgres
DB_PASSWORD=yourpassword
JWT_SECRET=your-256-bit-secret
INTERNAL_SERVICE_SECRET=your-internal-secret
RABBITMQ_USERNAME=guest
RABBITMQ_PASSWORD=guest
```

### 3. Start all services

```bash
docker compose up --build
```

First build takes several minutes (Maven downloads dependencies inside Docker). Subsequent starts are fast.

Wait until all services are registered in Eureka before sending requests:

```
http://localhost:8761
```

All six services should appear as UP.

### 4. Verify the system is healthy

```bash
# Should return a list of restaurants (empty on first run)
http GET localhost:8080/api/restaurants/search/all
```

---

## Port remappings

The host ports below differ from the standard defaults to avoid conflicts with locally running services:

| Service | Container port | Host port |
|---|---|---|
| PostgreSQL | 5432 | **5433** |
| RabbitMQ AMQP | 5672 | **5673** |
| RabbitMQ Management UI | 15672 | **15673** |

RabbitMQ Management UI: [http://localhost:15673](http://localhost:15673) (default credentials: `guest` / `guest`)

---

## Running the test plan

See [`TEST_PLAN.md`](TEST_PLAN.md) for a step-by-step httpie test plan covering all endpoints.

For Postman users, import [`food-delivery.postman_collection.json`](food-delivery.postman_collection.json). The collection auto-captures tokens and IDs into variables — run the folders in order.

---

## Troubleshooting

**Gateway returns 500 after a restart:**
Stale Eureka registrations from the previous container run can persist. Restart the discovery server to clear them:
```bash
docker compose restart discovery-server
```

**Services fail to start — port already in use:**
Check that no local PostgreSQL or RabbitMQ instance is using the remapped ports (5433, 5673, 15673).

**A service is not appearing in Eureka:**
Allow 30–60 seconds after container start. Eureka registration is asynchronous.

---

## Updating a submodule

After making changes inside a service directory, commit and push in that service repo first, then update the pointer in this root repo:

```bash
# Inside the service directory
git add . && git commit -m "your message" && git push

# Back in the root
git add <service-dir>
git commit -m "chore: update <service> submodule pointer"
git push
```
