# **Pulse: AI-Powered Incident Monitoring Platform**

**Pulse** ingests logs and metrics from your services, detects error spikes, groups them into incidents, and uses an **LLM** to explain what went wrong and suggest a likely cause. A **React** dashboard shows service health and incidents in real time.

> **Status:** In development. Phase 0 (foundation) and Phase 1 (requirements) are complete. Follow the progress on the project board.

---

## **Table of Contents**

1. [Why Pulse](#why-pulse)
2. [Key Features](#key-features)
3. [Architecture](#architecture)
4. [Tech Stack](#tech-stack)
5. [Getting Started](#getting-started)
6. [API Overview](#api-overview)
7. [Rate Limiting](#rate-limiting)
8. [Caching Strategy](#caching-strategy)
9. [AI Integration](#ai-integration)
10. [Testing](#testing)
11. [Performance Results](#performance-results)
12. [Project Structure](#project-structure)
13. [Documentation](#documentation)
14. [Roadmap](#roadmap)
15. [Author](#author)

---

## **Why Pulse**

Small teams run several services and collect a flood of logs and alerts. When something breaks, engineers search through raw logs to find out **what happened**, **how serious it is**, and **where to start**. Pulse shortens that process by turning raw events into **incidents with plain-English explanations**, while avoiding repeated AI calls for errors it has already seen.

---

## **Key Features**

- **Secure access:** user registration and login with **JWT** authentication
- **Service registration:** hashed, revocable **API keys** for every service
- **Ingestion API:** accepts batches of logs and metrics and processes them asynchronously
- **Rate limiting:** per-API-key **token bucket** implemented with **Redis** and a Lua script
- **Spike detection:** configurable threshold and time window per service
- **Incident management:** automatic grouping, status tracking (open, acknowledged, resolved)
- **AI insights:** LLM-generated summary and likely cause for each incident
- **Smart caching:** AI summaries cached by normalized error pattern to cut LLM calls
- **Dashboard:** service health, incident list, and incident detail with timeline
- **Notifications:** alert when a new incident opens

---

## **Architecture**

```
Client Service
      |
      v
 API Gateway  (JWT / API key auth + Redis rate limiter)
      |
      v
Ingestion Service ---> RabbitMQ ---> Analysis Service (spike detection)
                                           |
                                           v
                                  AI Insight Service ---> LLM API
                                     (Redis cache)
                                           |
                                           v
                                         MySQL <--- Notification Service
                                           ^
                                           |
                                    React Dashboard
```

> An architecture diagram image will be added in `docs/` after Phase 2.

| **Service** | **Responsibility** |
|---|---|
| **Gateway** | Routing, authentication, per-key rate limiting |
| **Auth Service** | Users, JWT, services and API keys |
| **Ingestion Service** | Validates and queues incoming events |
| **Analysis Service** | Detects spikes, creates and groups incidents |
| **AI Insight Service** | Builds prompts, calls the LLM, caches results |
| **Notification Service** | Sends alerts for new incidents |

---

## **Tech Stack**

| **Layer** | **Technology** |
|---|---|
| **Backend** | Java 21, Spring Boot 3, Spring Cloud Gateway |
| **Frontend** | React, Vite |
| **Database** | MySQL 8, Flyway migrations |
| **Cache and rate limiter** | Redis (Lua scripts) |
| **Messaging** | RabbitMQ |
| **Resilience** | Resilience4j (timeout, retry, circuit breaker) |
| **Testing** | JUnit 5, Mockito, Testcontainers, k6 |
| **Observability** | Spring Actuator, Prometheus, Grafana |
| **DevOps** | Docker, Docker Compose, GitHub Actions |

---

## **Getting Started**

### **Prerequisites**

- Java 21
- Node.js (LTS)
- Docker and Docker Compose
- An LLM API key

### **Run with Docker Compose**

```bash
# 1. Clone the repository
git clone https://github.com/shivarajns/Pulse-AI-Powered-Incident-Monitoring-Platform.git
cd Pulse-AI-Powered-Incident-Monitoring-Platform

# 2. Create your environment file
cp .env.example .env
# then add your LLM API key and database passwords in .env

# 3. Start everything
docker compose up --build
```

| **Component** | **URL** |
|---|---|
| **Dashboard** | `http://localhost:3000` |
| **API Gateway** | `http://localhost:8080` |

> Exact ports and commands will be confirmed when the services are built.

---

## **API Overview**

| **Method** | **Endpoint** | **Description** |
|---|---|---|
| `POST` | `/auth/register` | Create an account |
| `POST` | `/auth/login` | Log in and receive a JWT |
| `POST` | `/services` | Register a service and receive an API key |
| `POST` | `/ingest` | Send a batch of logs and metrics |
| `GET` | `/incidents` | List incidents (filter by service and status) |
| `GET` | `/incidents/{id}` | Incident detail with AI summary |
| `PATCH` | `/incidents/{id}` | Update incident status |

Full contracts will be published as an **OpenAPI** specification in `docs/api/`.

---

## **Rate Limiting**

Pulse uses a **token bucket** per API key. Each request spends a token, and tokens refill at a steady rate. The check-and-spend step runs as a single **Redis Lua script**, so it stays correct when several gateway instances run at once. Requests over the limit receive **`429 Too Many Requests`** with a **`Retry-After`** header.

---

## **Caching Strategy**

| **Cache** | **Key** | **Purpose** |
|---|---|---|
| **AI summary cache** | Hash of the normalized error pattern | Avoid calling the LLM again for the same error |
| **Dashboard cache** | Query parameters | Speed up frequently requested views |

---

## **AI Integration**

The AI Insight Service sends a **scrubbed sample** of grouped errors to an LLM and parses a structured response containing a **summary** and a **likely cause**. It is wrapped in a **timeout**, **retry with backoff**, and a **circuit breaker**. If the AI is unavailable, the incident still appears with a fallback message and the summary is retried later. Sensitive values such as emails and tokens are removed before prompts are built.

---

## **Testing**

- **Unit tests:** JUnit 5 and Mockito
- **Integration tests:** Testcontainers (MySQL, Redis, RabbitMQ)
- **Load tests:** k6 scripts in `load-tests/`
- **CI:** GitHub Actions runs the test suite on every pull request

---

## **Performance Results**

> Results will be filled in after load testing in Phase 5. Only **measured** numbers will be published here.

| **Metric** | **Target** | **Measured** |
|---|---|---|
| **Ingestion throughput** | 500 requests per second | _pending_ |
| **p95 latency (ingestion)** | under 200 ms | _pending_ |
| **AI cache hit rate** | above 60% | _pending_ |

---

## **Project Structure**

```
pulse/
  docs/               requirements, design, ADRs, test plan
  services/
    gateway/
    auth-service/
    ingestion-service/
    analysis-service/
    ai-insight-service/
    notification-service/
  common/             shared DTOs and event classes
  frontend/           React dashboard
  deploy/             docker-compose and monitoring config
  load-tests/         k6 scripts
  .github/workflows/  CI pipelines
```

---

## **Documentation**

| **Document** | **Description** |
|---|---|
| [`docs/00-project-foundation.md`](docs/00-project-foundation.md) | Goals, tooling, workflow, timeline |
| [`docs/01-requirements.md`](docs/01-requirements.md) | User stories, functional and non-functional requirements, risks |
| `docs/02-architecture.md` | System design (Phase 2) |
| `docs/03-detailed-design.md` | Schema, API contracts, event design (Phase 3) |
| `docs/04-test-plan.md` | Test strategy (Phase 5) |
| `docs/adr/` | Architecture decision records |

---

## **Roadmap**

- [x] **Phase 0:** Project foundation
- [x] **Phase 1:** Planning and requirements
- [ ] **Phase 2:** System design (HLD)
- [ ] **Phase 3:** Detailed design (LLD)
- [ ] **Phase 4:** Implementation
- [ ] **Phase 5:** Testing and load testing
- [ ] **Phase 6:** CI/CD and deployment
- [ ] **Phase 7:** Monitoring and maintenance

---

## **Author**

**Shivaraj**

- GitHub: [`shivarajns`](https://github.com/shivarajns)
- LinkedIn: [`Shivaraju N S`](https://www.linkedin.com/in/shivaraju-ns-0642a6291)
- Portfolio: [`Shivaraju NS`](https://github.com/shivarajns)
