
# Ticket Selling Microservices 🎫

A microservices-based **ticket booking backend** built with **Spring Boot 3**, **Kafka**, **MySQL**, **Spring Cloud Gateway (MVC)**, **Keycloak** for authentication, and **Resilience4j** for fault tolerance. The system models a complete event ticketing workflow: users create bookings, bookings generate events to Kafka, the order service consumes these events to create orders, and inventory is updated accordingly.

---

## 🧠 High-Level Overview

```

Client → API Gateway (8090)
│
├── Booking Service (8081)
│        │
│        └── calls Inventory Service (8080)
│
└── Order Service (8082) ← consumes Kafka topic "booking"
│
└── updates Inventory Service (8080)

```

Core responsibilities:
- **Booking Service**: validates capacity, creates customers, publishes `BookingEvent` to Kafka  
- **Order Service**: consumes events, creates orders, reduces event capacity  
- **Inventory Service**: manages venues, events, capacities  
- **API Gateway**: authentication, routing, Swagger aggregation, circuit breaking  

The project uses a single MySQL database (`ticketing`) but isolated service layers.

---

## ✨ Features

- Create/update/manage event bookings  
- Capacity validation against Inventory Service  
- Event-driven communication via Kafka (`booking` topic)  
- Order creation upon booking events  
- Capacity decrement for events  
- JWT authentication via Keycloak at API Gateway  
- Circuit breaker, retry, timeout (Resilience4j)  
- Swagger/OpenAPI documentation aggregated at the gateway  
- Clean Controller → Service → Repository architecture  

---

## 🧰 Tech Stack

**Backend**  
- Java 17  
- Spring Boot 3.4.x  
- Spring Web / MVC  
- Spring Data JPA  

**Infrastructure**  
- Apache Kafka  
- MySQL  
- Spring Cloud Gateway (MVC)  
- Resilience4j  
- Springdoc OpenAPI  

**Security**  
- Keycloak (JWT validation via JWK)  
- Spring Security OAuth2 Resource Server  

---

## 🧩 Microservices Overview

### 🔹 API Gateway (`apigateway`)
**Port:** 8090  
**Responsibilities:**
- Central entry point for all traffic  
- Validates JWT tokens using Keycloak JWK  
- Routes requests to booking/inventory/order services  
- Aggregates Swagger docs:
  - `http://localhost:8090/swagger-ui.html`
- Circuit breaker, retry, timeout configuration via Resilience4j  

Key config:
```

server.port=8090
keycloak.auth.jwk-set-uri=[http://localhost:8091/realms/ticketing-security-realm/protocol/openid-connect/certs](http://localhost:8091/realms/ticketing-security-realm/protocol/openid-connect/certs)
springdoc.swagger-ui.path=/swagger-ui.html

````

---

### 🔹 Inventory Service (`inventoryservice`)
**Port:** 8080  
**Base Path:** `/api/v1/inventory`

**Responsibilities:**
- Manages events + venue capacity  
- Provides event details, capacity, ticket price  
- Reduces capacity after orders  

APIs:
- `GET /event/{eventId}` → get capacity + price  
- `PUT /event/{eventId}/capacity/{count}` → reduce capacity  

Entities:
- `Event` (id, name, address, totalCapacity, leftCapacity, ticketPrice, venue)  
- `Venue` (id, name, address, totalCapacity)  

---

### 🔹 Booking Service (`bookingservice`)
**Port:** 8081  
**Base Path:** `/api/v1/booking`

**Responsibilities:**
- Accept user booking requests  
- Fetch event info from Inventory Service  
- Validate capacity  
- Create Customer entry  
- Publish `BookingEvent` to Kafka  

`BookingEvent` contains:
```java
{
  userId,
  eventId,
  ticketCount,
  totalPrice
}
````

Kafka config:

```
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.template.default-topic=booking
```

Example request:

```json
{
  "userId": 1,
  "eventId": 100,
  "ticketCount": 2
}
```

---

### 🔹 Order Service (`orderservice`)

**Port:** 8082

**Responsibilities:**

* Consumes `BookingEvent` from Kafka topic `booking`
* Creates an `Order` entity
* Calls Inventory Service to reduce event capacity

Order entity fields:

* id
* ticketCount
* totalPrice
* placedAt
* customerId
* eventId

Kafka consumer:

```
spring.kafka.consumer.group-id=order-service
spring.kafka.consumer.value-deserializer=JsonDeserializer
```

---

## 🚀 Getting Started

### 1️⃣ Prerequisites

* Java 17
* Maven
* MySQL running with a database named `ticketing`
* Kafka + Zookeeper running on `localhost:9092`
* Keycloak running (for JWT auth)

### 2️⃣ Clone the Repository

```bash
git clone https://github.com/mokshagna27/ticketselling-microservices.git
cd ticketselling-microservices
```

### 3️⃣ Configure MySQL

In each service:

```
spring.datasource.url=jdbc:mysql://localhost:3306/ticketing
spring.datasource.username=root
spring.datasource.password=your_password
spring.jpa.hibernate.ddl-auto=none
```

Make sure required tables exist (`event`, `venue`, `customer`, `order`).
Entities include SQL schema inside comments.

### 4️⃣ Start Services

In order:

```bash
cd inventoryservice
mvn spring-boot:run

cd ../bookingservice
mvn spring-boot:run

cd ../orderservice
mvn spring-boot:run

cd ../apigateway
mvn spring-boot:run
```

---

## 🔍 Testing the Flow

### Step 1: Check event inventory

```
GET http://localhost:8080/api/v1/inventory/event/{eventId}
```

### Step 2: Create a booking

```
POST http://localhost:8081/api/v1/booking
```

Body:

```json
{
  "userId": 1,
  "eventId": 100,
  "ticketCount": 2
}
```

### Step 3: Kafka event fired

`BookingService` → topic `booking`

### Step 4: Order created

`OrderService` consumes the message → stores order in DB.

### Step 5: Inventory updated

`OrderService` → Inventory → reduce capacity





