
# Ticket Selling System 🎟️

A scalable **ticket booking backend** built with **Spring Boot** and a **microservices architecture**.  
It handles bookings, orders, and inventory with **API Gateway**, **Kafka**, **MySQL**, **Flyway**, and **Keycloak** authentication.  
All services are containerized using **Docker** for seamless deployment.

---

## 📚 Table of Contents
- [Architecture](#architecture)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Microservices](#microservices)
- [Getting Started](#getting-started)
- [API Overview](#api-overview)
- [Database & Migrations](#database--migrations)
- [Kafka Topics](#kafka-topics)
- [Security](#security)
- [Future Improvements](#future-improvements)


---

## Architecture

```

Client → API Gateway → Booking Service → Inventory Service
↓
Kafka Queue
↓
Order Service

All services → MySQL DB
Auth handled via Keycloak

````

---

## Features

- 🎟️ Book tickets for events  
- 📦 Inventory reservation and release  
- 📑 Order creation & status tracking  
- 🔐 Authentication & authorization with Keycloak  
- 📩 Asynchronous communication using Kafka topics  
- 🗄️ MySQL with Flyway migrations  
- 🐳 Fully containerized using Docker  
- 🧱 Clean microservice separation  

---

## Tech Stack

**Languages & Frameworks**
- Java 17+, Spring Boot  
- Spring Web, Spring Data JPA  
- Spring Cloud (Gateway, Config)

**Infrastructure**
- MySQL  
- Apache Kafka  
- Docker & Docker Compose  
- Flyway  
- Keycloak  

**Tools**
- Git  
- IntelliJ / VS Code  

---

## Microservices

### API Gateway
- Entry point for all external requests  
- Does auth token validation  
- Routes traffic to respective services  

### Booking Service
- Creates/cancels bookings  
- Calls Inventory Service to check/reserve seats  
- Publishes booking/order events to Kafka  

### Inventory Service
- Manages seats/tickets availability  
- Reserves or releases inventory  
- Persists data in MySQL  

### Order Service
- Listens to Kafka events  
- Creates and updates order records  
- Manages order lifecycle  

### Auth (Keycloak)
- User & role management  
- Issues JWT tokens  
- Integrated with Gateway & services  

---

## Getting Started

### Prerequisites
- Java 17+  
- Docker & Docker Compose  
- MySQL  
- Kafka  

### Clone Repo

```bash
git clone https://github.com/<your-username>/ticket-selling-system.git
cd ticket-selling-system
````

### Configuration

#### MySQL

```
spring.datasource.url=jdbc:mysql://localhost:3306/ticketdb
spring.datasource.username=root
spring.datasource.password=your_password
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
```

#### Kafka

```
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=ticket-group
```

#### Keycloak

```
keycloak.auth-server-url=http://localhost:8080/auth
keycloak.realm=ticket-system
keycloak.resource=ticket-api
keycloak.public-client=true
```

### Run with Docker Compose

```bash
docker-compose up --build
```

### Run Locally (Without Docker)

```bash
cd api-gateway && mvn spring-boot:run
cd booking-service && mvn spring-boot:run
cd inventory-service && mvn spring-boot:run
cd order-service && mvn spring-boot:run
```

---

## API Overview

### Booking Service

**POST /api/bookings**

```
{
  "eventId": 1,
  "userId": 123,
  "quantity": 2
}
```

**GET /api/bookings/{id}**
Fetch booking info

**DELETE /api/bookings/{id}**
Cancel booking

### Inventory Service

**GET /api/inventory/{eventId}**
Available seats

**POST /api/inventory/reserve**
Reserve seats

**POST /api/inventory/release**
Release seats

### Order Service

**GET /api/orders/{id}**
Order details

**GET /api/orders/user/{userId}**
User orders

---

## Database & Migrations

* Managed using **Flyway**
* Migrations stored in:

```
src/main/resources/db/migration
```

* Naming:

```
V1__init.sql
V2__add_order_table.sql
```

---

## Kafka Topics

* **booking-events** → published by Booking Service
* **order-events** → consumed by Order Service

---

## Security

* Keycloak handles users/roles
* Clients send JWT token:

```
Authorization: Bearer <token>
```

* Role-based access using Spring Security:

```java
@PreAuthorize("hasRole('ROLE_USER')")
```

---

## Future Improvements

* Integrate payment gateway
* Add rate limiting, retries, circuit breaker (Resilience4j)
* Add monitoring + tracing (Prometheus, Grafana, Zipkin)
* Full test suite
* GitHub Actions CI/CD


