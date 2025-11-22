

# Ticket Selling Microservices 🎫

A microservices-based **ticket booking backend** built with **Spring Boot 3**, **Kafka**, **MySQL**, **Spring Cloud Gateway (MVC)**, **Keycloak** for security, and **Resilience4j** for fault tolerance.

This project models a simplified ticketing system with:
- **Inventory Service** for venues/events and capacities
- **Booking Service** for creating bookings
- **Order Service** for consuming booking events and finalizing orders
- **API Gateway** for routing, auth, and aggregated API docs


---

## 🧠 High-Level Overview

When a user books tickets:

1. **Client → API Gateway**  
   Request goes to the gateway on `localhost:8090`.

2. **Gateway → Booking Service (8081)**  
   Booking service:
   - Validates the request
   - Calls **Inventory Service** to fetch event capacity & ticket price
   - Creates a **Customer** record
   - Builds a `BookingEvent` payload

3. **Booking Service → Kafka**  
   Booking service publishes a `BookingEvent` to the `booking` topic.

4. **Order Service (8082) → Kafka**  
   Order service consumes `BookingEvent`:
   - Persists an `Order` entity
   - Calls **Inventory Service** to reduce available capacity.

5. **Inventory Service (8080)**  
   Updates the remaining capacity for the booked event.

All calls go through the gateway in a real deployment; direct service ports are mostly for local dev/testing.

---

## 🏗️ Architecture

```text
             [ Client ]
                 │
                 ▼
        ┌────────────────┐
        │  API Gateway   │  Port: 8090
        │  (Spring Cloud │
        │   Gateway MVC) │
        └───────┬────────┘
                │
     ┌──────────┼───────────┐
     ▼          ▼           ▼
[Booking]   [Inventory]  [Order]
 Service     Service      Service
 Port 8081   Port 8080   Port 8082

Booking <──Kafka topic "booking"──┐
                                  ▼
                               Order

MySQL "ticketing" DB shared across services
Keycloak for JWT auth (used at the gateway)
Resilience4j at the gateway for circuit-breaking & retries
OpenAPI/Swagger UI aggregated via the gateway

✨ Features

Create ticket bookings for an event

Validate available capacity before booking

Maintain customer, event, venue, and order data

Asynchronous order processing using Kafka (booking topic)

Inventory update after order creation

API Gateway:

Routes traffic to booking & inventory services

Aggregates Swagger docs for both services

Secures endpoints using Keycloak JWT

Circuit breaker, retry, timeout via Resilience4j

MySQL persistence via Spring Data JPA

Clean Controller–Service–Repository layering

🧰 Tech Stack

Core

Java 17+

Spring Boot 3.4.x

Spring Web / Spring MVC

Spring Data JPA

Spring Cloud Gateway (MVC)

Spring Security OAuth2 Resource Server

Apache Kafka

Resilience4j

Springdoc OpenAPI (Swagger)

MySQL

Security

Keycloak (JWT, realms, roles)

Gateway-level auth via JWT using the JWK set URI

Build

Maven

🧩 Services Breakdown
1. API Gateway (apigateway)

Port: 8090
Responsibilities:

Acts as the single entry point

Uses Spring Cloud Gateway MVC routing functions

Validates JWTs against Keycloak (via JWK set URI)

Configures circuit breakers, retries, and timeouts using Resilience4j

Aggregates Swagger docs for:

Inventory Service: /docs/inventoryservice/v3/api-docs

Booking Service: /docs/bookingservice/v3/api-docs

Exposes a unified Swagger UI at:
http://localhost:8090/swagger-ui.html

Key config (excerpt) – apigateway/src/main/resources/application.properties:

server.port=8090

keycloak.auth.jwk-set-uri=http://localhost:8091/realms/ticketing-security-realm/protocol/openid-connect/certs

swagger-ui + doc URLs

security.excluded.urls to allow docs without auth

Resilience4j circuit breaker, retry, and timelimiter defaults

2. Inventory Service (inventoryservice)

Port: 8080
Base Path (by convention): /api/v1/inventory
(confirmed by inventory.service.url in booking/order services)

Responsibilities:

Manages Venue and Event inventory

Exposes APIs for:

Getting inventory for all venues/events

Getting inventory for a single event

Updating event capacity after a booking

Key pieces:

InventoryController

GET /api/v1/inventory/event/{eventId}
→ returns EventInventoryResponse

PUT /api/v1/inventory/event/{eventId}/capacity/{capacity}
→ subtracts capacity from leftCapacity of the event

InventoryService

Uses EventRepository and VenueRepository

Converts entities into EventInventoryResponse / VenueInventoryResponse

Entities:

Event – includes fields like id, name, address, totalCapacity, leftCapacity, ticketPrice, venue

Venue – id, name, address, totalCapacity

Config: inventoryservice/src/main/resources/application.properties

server.port=8080

spring.datasource.url=jdbc:mysql://localhost:3306/ticketing

spring.jpa.hibernate.ddl-auto=none

OpenAPI paths for Swagger

Note: DDL is not auto-generated. You must create tables manually – entities include SQL comments as hints.

3. Booking Service (bookingservice)

Port: 8081
Base Path: /api/v1

Responsibilities:

Handles booking requests

Talks to Inventory Service to validate event capacity and price

Creates Customer record

Constructs a BookingEvent and publishes it to Kafka

Key components:

BookingController

POST /api/v1/booking
Request body (BookingRequest):

{
  "userId": 1,
  "eventId": 100,
  "ticketCount": 2
}


Response (BookingResponse):

{
  "userId": 1,
  "eventId": 100,
  "ticketCount": 2,
  "totalPrice": 500.00,
  "message": "Booking created successfully"
}


(Fields inferred from BookingResponse structure.)

BookingService

Calls InventoryServiceClient → GET {inventory.service.url}/event/{eventId}

Maps InventoryResponse (capacity, ticketPrice, venue)

Persists a Customer via CustomerRepository

Publishes BookingEvent to Kafka using KafkaTemplate<String, BookingEvent>

InventoryServiceClient

Uses inventory.service.url from config:

inventory.service.url=http://localhost:8080/api/v1/inventory


Fetches InventoryResponse for an event

BookingEvent (Kafka payload)

public class BookingEvent {
    private Long userId;
    private Long eventId;
    private Long ticketCount;
    private BigDecimal totalPrice;
}


Customer entity

id, name, email, address

Mapped to customer table (SQL blueprint is in the comment)

Config: bookingservice/src/main/resources/application.properties

server.port=8081

MySQL connection to ticketing DB

Kafka:

spring.kafka.bootstrap-servers=localhost:9092

spring.kafka.template.default-topic=booking

OpenAPI paths for Swagger

4. Order Service (orderservice)

Port: 8082

Responsibilities:

Consumes BookingEvent messages from Kafka

Creates and stores Order entities

Calls Inventory Service to reduce remaining capacity

Key components:

OrderService

Annotated with @KafkaListener on the booking topic (configured in properties)

On each BookingEvent:

Maps it to an Order entity

Saves via OrderRepository

Calls InventoryServiceClient.updateInventory(eventId, ticketCount)

Logs inventory updates

Order entity (simplified)

public class Order {
    private Long id;
    private BigDecimal totalPrice;
    private Long ticketCount;
    private LocalDateTime placedAt;
    private Long customerId;
    private Long eventId;
}


InventoryServiceClient

Reads inventory.service.url from config:

inventory.service.url=http://localhost:8080/api/v1/inventory


PUT {inventory.service.url}/event/{eventId}/capacity/{ticketCount}

Uses RestTemplate internally

Config: orderservice/src/main/resources/application.properties

server.port=8082

MySQL ticketing database

Kafka consumer:

spring.kafka.bootstrap-servers=localhost:9092

spring.kafka.consumer.group-id=order-service

JSON deserializer for BookingEvent

inventory.service.url=http://localhost:8080/api/v1/inventory

🚀 Getting Started
1. Prerequisites

Java 17+

Maven

MySQL running with a database named ticketing

Kafka + Zookeeper running locally on localhost:9092

Keycloak running (for JWT validation at the gateway)

Tables created according to your entities:

customer, order, event, venue, etc.
(Entities contain comments with sample DDL.)

2. Clone the repository
git clone https://github.com/mokshagna27/ticketselling-microservices.git
cd ticketselling-microservices

3. Configure MySQL

Update the spring.datasource.* properties in these files if needed:

bookingservice/src/main/resources/application.properties

inventoryservice/src/main/resources/application.properties

orderservice/src/main/resources/application.properties

Example:

spring.datasource.url=jdbc:mysql://localhost:3306/ticketing
spring.datasource.username=root
spring.datasource.password=your_password
spring.jpa.hibernate.ddl-auto=none


Make sure the ticketing database and required tables exist. JPA will not auto-create them (ddl-auto=none).

4. Configure Kafka

Ensure Kafka is running on:

spring.kafka.bootstrap-servers=localhost:9092


by default in booking & order services.

5. Configure Keycloak (Gateway)

In apigateway/src/main/resources/application.properties:

keycloak.auth.jwk-set-uri=http://localhost:8091/realms/ticketing-security-realm/protocol/openid-connect/certs


Set this to your Keycloak realm's JWK set URI.

6. Run the services

From the project root, you can run each service:

cd apigateway
mvn spring-boot:run

cd ../inventoryservice
mvn spring-boot:run

cd ../bookingservice
mvn spring-boot:run

cd ../orderservice
mvn spring-boot:run


Order doesn’t strictly matter, but a good flow is:
Inventory → Booking → Order → Gateway.

🔍 Testing the Flow (Happy Path)

Ensure inventory exists
Insert venue and event rows into the ticketing DB with some left_capacity and ticket_price.

Call Inventory API (direct or via gateway)

GET http://localhost:8080/api/v1/inventory/event/{eventId}
to confirm event capacity.

Create a booking

POST http://localhost:8081/api/v1/booking
or via gateway route once configured.

Example body:

{
  "userId": 1,
  "eventId": 100,
  "ticketCount": 2
}


Observe Kafka + Order Service

Booking service publishes BookingEvent to Kafka.

Order service consumes it, creates an order, logs inventory update.

Verify DB

Check order table for a new order row.

Check event.left_capacity is reduced accordingly.



