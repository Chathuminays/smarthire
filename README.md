# SmartHire - Job Platform Built with Microservices

A full-stack job platform where applicants find and apply for jobs, employers post openings, and admins manage the system. Built as a distributed microservices architecture using Spring Boot, Spring Cloud, Kafka, and PostgreSQL.

This project demonstrates production-grade patterns: JWT authentication with refresh token rotation, event-driven communication, API gateway routing, service discovery, circuit breaking, Redis caching, and containerized deployment with Docker Compose.

## Architecture

```
                         ┌──────────────────────┐
                         │      Client App       │
                         │   (React + TypeScript) │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     API Gateway       │
                         │  (Spring Cloud Gateway)│
                         │     Port: 8080        │
                         └──────────┬───────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                   │
                 ▼                  ▼                   ▼
        ┌────────────────┐ ┌────────────────┐ ┌─────────────────┐
        │  User Service  │ │  Job Service   │ │  Notification   │
        │  Port: 8081    │ │  Port: 8082    │ │  Service        │
        │                │ │                │ │  Port: 8083     │
        │  • Auth (JWT)  │ │  • Job CRUD    │ │                 │
        │  • Profiles    │ │  • Applications│ │  • Email/SMS    │
        │  • Roles       │ │  • Search      │ │  • Event logs   │
        └───────┬────────┘ └───────┬────────┘ └────────┬────────┘
                │                  │                    │
                ▼                  ▼                    │
        ┌──────────────┐  ┌──────────────┐             │
        │ PostgreSQL   │  │ PostgreSQL   │             │
        │ (userdb)     │  │ (jobdb)      │             │
        │ Port: 5432   │  │ Port: 5433   │             │
        └──────────────┘  └──────────────┘             │
                │                  │                    │
                │          ┌──────────────┐            │
                │          │    Redis     │            │
                │          │  Port: 6379  │            │
                │          └──────────────┘            │
                │                  │                    │
                └──────────────────┼────────────────────┘
                                   │
                            ┌──────▼──────┐
                            │ Apache Kafka │
                            │ Port: 9092   │
                            └──────┬──────┘
                                   │
                            ┌──────▼──────┐
                            │  Zookeeper  │
                            │ Port: 2181   │
                            └─────────────┘

        ┌──────────────────────────────────────────────┐
        │           Eureka Service Discovery           │
        │                Port: 8761                    │
        └──────────────────────────────────────────────┘
```

## Tech Stack

**Backend:** Java 21, Spring Boot 3.x, Spring Security, Spring Data JPA, Spring Cloud Gateway, Spring Kafka

**Infrastructure:** PostgreSQL 16, Apache Kafka, Redis 7, Netflix Eureka, Resilience4j

**DevOps:** Docker, Docker Compose

**Frontend:** React 18, TypeScript, Tailwind CSS

## Microservices Breakdown

### User Service (Port 8081)

Handles authentication and user profile management.

- JWT access tokens (15 min expiry) and refresh tokens (7 day expiry) with rotation
- BCrypt password hashing
- Role-based access control: `APPLICANT`, `EMPLOYER`, `ADMIN`
- User profile with skills (stored as JSON in PostgreSQL), experience, education, resume URL
- Internal endpoint for other services to validate tokens
- Publishes `USER_REGISTERED` and `USER_UPDATED` events to Kafka

**Endpoints:**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/register` | Public | Register a new user |
| POST | `/auth/login` | Public | Login and receive tokens |
| POST | `/auth/refresh` | Public | Exchange refresh token for new token pair |
| POST | `/auth/logout` | Required | Invalidate refresh token |
| GET | `/users/me` | Required | Get current user's profile |
| GET | `/users/{id}/profile` | Required | Get any user's profile |
| PUT | `/users/{id}/profile` | Owner only | Update profile |
| GET | `/users/admin/all` | Admin only | List all users |
| POST | `/internal/validate-token` | Internal | Validate JWT from other services |
| GET | `/internal/users/{id}` | Internal | Get user by ID (inter-service) |

### Job Service (Port 8082)

Manages job postings, applications, and job search.

- Full CRUD for job listings (employers only)
- Job application workflow with status tracking (APPLIED → REVIEWED → SHORTLISTED → REJECTED / ACCEPTED)
- Dynamic filtering with JPA Specifications (location, salary range, skills, job type)
- Paginated search results
- Redis caching for frequently accessed job listings
- Validates user tokens via User Service
- Publishes `JOB_POSTED` and `APPLICATION_STATUS_CHANGED` events to Kafka

### Notification Service (Port 8083)

Consumes events from Kafka and handles notifications.

- Listens to all domain events from User Service and Job Service
- Simulates email and SMS dispatch (logs to console, extensible to real providers)
- Event-driven architecture - fully decoupled from other services

### Service Discovery (Port 8761)

Netflix Eureka server. All services register themselves on startup and discover each other dynamically - no hardcoded URLs between services.

### API Gateway (Port 8080)

Single entry point for all client requests. Routes traffic to the correct service based on URL path.

- `/api/users/**` → User Service
- `/api/jobs/**` → Job Service
- `/api/notifications/**` → Notification Service
- Rate limiting to prevent abuse
- Centralized CORS configuration
- Circuit breaker with Resilience4j for fault tolerance

## Getting Started

### Prerequisites

- Java 21 ([Adoptium Temurin](https://adoptium.net/))
- Docker Desktop ([download](https://www.docker.com/products/docker-desktop/))
- IntelliJ IDEA Community Edition ([download](https://www.jetbrains.com/idea/download/))
- Git

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/smarthire.git
cd smarthire
```

### 2. Start Infrastructure

This starts PostgreSQL (two instances), Redis, Kafka, and Zookeeper in Docker containers:

```bash
docker compose up -d
```

Verify everything is running:

```bash
docker compose ps
```

### 3. Run the Services

Open the project in IntelliJ IDEA. Start the services in this order:

1. **Service Discovery** - `DiscoveryApplication.java` → verify at http://localhost:8761
2. **User Service** - `UserServiceApplication.java`
3. **Job Service** - `JobServiceApplication.java`
4. **Notification Service** - `NotificationServiceApplication.java`
5. **API Gateway** - `GatewayApplication.java`

All services should appear as registered on the Eureka dashboard.

### 4. Test the API

Register a user:

```bash
curl -X POST http://localhost:8080/api/users/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jane",
    "lastName": "Silva",
    "email": "jane@example.com",
    "password": "securePass123",
    "role": "APPLICANT"
  }'
```

Login:

```bash
curl -X POST http://localhost:8080/api/users/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "jane@example.com",
    "password": "securePass123"
  }'
```

Use the returned `accessToken` in subsequent requests:

```bash
curl http://localhost:8080/api/users/users/me \
  -H "Authorization: Bearer <access-token>"
```

### API Documentation

Swagger UI is available for each service when running:

- User Service: http://localhost:8081/swagger-ui.html
- Job Service: http://localhost:8082/swagger-ui.html

## Project Structure

```
smarthire/
├── docker-compose.yml
├── pom.xml                          # Parent POM linking all modules
├── README.md
│
├── service-discovery/               # Eureka Server
├── api-gateway/                     # Spring Cloud Gateway
│
├── user-service/
│   └── src/main/java/com/smarthire/user/
│       ├── controller/              # REST endpoints
│       ├── service/                 # Business logic
│       ├── repository/              # Data access (Spring Data JPA)
│       ├── model/                   # JPA entities
│       ├── dto/                     # Request/response objects
│       ├── security/                # JWT filter, config, token service
│       ├── config/                  # App and Kafka configuration
│       └── exception/               # Global error handling
│
├── job-service/
│   └── src/main/java/com/smarthire/job/
│       ├── controller/
│       ├── service/
│       ├── repository/
│       ├── model/
│       ├── dto/
│       ├── config/
│       └── exception/
│
└── notification-service/
    └── src/main/java/com/smarthire/notification/
        ├── consumer/                # Kafka event listeners
        ├── service/
        └── config/
```

## Key Design Decisions

**Database per service** - User Service and Job Service each have their own PostgreSQL instance. Services never share a database directly, ensuring loose coupling. Data is shared through APIs and events.

**Event-driven communication** - Kafka handles asynchronous events (user registered, job posted, application status changed). Services don't need to know about each other to react to domain events.

**JWT with refresh token rotation** - Access tokens are short-lived (15 minutes). Refresh tokens are single-use; each refresh issues a new pair, preventing token replay attacks.

**Internal API endpoints** - The `/internal/*` routes on User Service allow other services to validate tokens and look up user data without duplicating authentication logic.

**Specification pattern for search** - Job filtering uses JPA Specifications to build dynamic queries at runtime, avoiding brittle query-per-filter approaches.

## Environment Variables

All configuration is in each service's `application.yml`. Key values:

| Variable | Default | Description |
|----------|---------|-------------|
| PostgreSQL (User) | `localhost:5432/userdb` | User Service database |
| PostgreSQL (Job) | `localhost:5433/jobdb` | Job Service database |
| Redis | `localhost:6379` | Job listing cache |
| Kafka | `localhost:9092` | Event broker |
| Eureka | `localhost:8761` | Service registry |
| JWT Secret | Configured in `application.yml` | Token signing key |

## License

This project is built for educational and portfolio purposes.