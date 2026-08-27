# EventSphere Backend Services

Business microservices layer for the EventSphere platform. This directory contains the core domain services that implement event booking, user management, and review functionality.

## Services

### User Service
**Port:** 8081

Identity and access management microservice built with Spring Web MVC and Spring Data JPA.

Responsibilities:
- User registration and authentication
- BCrypt password hashing
- Role-based access control (`ORGANIZER`, `ATTENDEE`)
- User profile CRUD operations

Tech stack:
- Spring Boot 4.1.0, Spring Web MVC
- Spring Data JPA / Hibernate
- PostgreSQL (`userdb`)
- Lombok, Hibernate Validator
- Spring Cloud Config + Netflix Eureka Client

See [user-service/README.md](user-service/README.md) for full documentation.

### Event Booking Service
**Port:** 8082

Event and ticket booking microservice built with Spring Web MVC and Spring Data JPA.

Responsibilities:
- Venue management
- Event lifecycle management
- Ticket type and pricing management
- Seat inventory enforcement
- Booking creation, status updates, and cancellation

Concurrency controls:
- Pessimistic locking (`SELECT ... FOR UPDATE`) on ticket types during booking transitions
- Optimistic locking (`@Version`) on `ticket_types` as a secondary safeguard

Tech stack:
- Spring Boot 4.1.0, Spring Web MVC
- Spring Data JPA / Hibernate
- PostgreSQL (`userdb`)
- BigDecimal for pricing precision
- Lombok, Hibernate Validator
- Spring Cloud Config + Netflix Eureka Client

See [event-booking-service/README.md](event-booking-service/README.md) for full documentation.

### Review & Notification Service
**Port:** 8083

Review, media, and notification archive microservice built with Spring Web MVC and Google Cloud NoSQL services.

Responsibilities:
- Event review CRUD operations
- Multipart media upload to Cloud Storage
- Notification archive persistence

Tech stack:
- Spring Boot 4.1.0, Spring Web MVC
- Google Cloud Firestore (document database)
- Google Cloud Storage (object storage)
- GCP SDK: `google-cloud-firestore`, `google-cloud-storage`
- Lombok, Hibernate Validator
- Spring Cloud Config + Netflix Eureka Client

See [review-notification-service/README.md](review-notification-service/README.md) for full documentation.

## Data Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL (userdb)                       │
│  ┌────────────────┐  ┌──────────────────┐  ┌────────────┐ │
│  │    users       │  │      venues      │  │   events   │ │
│  └────────────────┘  └──────────────────┘  └────────────┘ │
│  ┌────────────────┐  ┌──────────────────┐                   │
│  │ ticket_types   │  │    bookings      │                   │
│  └────────────────┘  └──────────────────┘                   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              Google Cloud Firestore                          │
│  ┌────────────────┐  ┌──────────────────────┐               │
│  │   reviews      │  │  notifications_archive│              │
│  └────────────────┘  └──────────────────────┘               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│           Google Cloud Storage (eventsphere-504909-media)    │
│  reviews/{reviewId}/{uuid}-{filename}                       │
└─────────────────────────────────────────────────────────────┘
```

## Inter-Service Communication

Services communicate through the API Gateway. Direct service-to-service REST calls are minimized; where cross-service references are needed (e.g., `organizerId` referencing `user-service`), validation is performed via REST through the gateway.

```
Frontend → API Gateway → Microservices
                               ↓
                         Eureka Registry
                               ↓
                         Config Server
                               ↓
                         Config Repo (Git)
```

## Service Ports

| Service | Port | Database | Storage |
|---------|------|----------|---------|
| User Service | 8081 | PostgreSQL | — |
| Event Booking Service | 8082 | PostgreSQL | — |
| Review & Notification Service | 8083 | Google Firestore | Google Cloud Storage |

## Running Locally

### Prerequisites

- Java 25
- Maven 3.9+
- PostgreSQL running on port 5432
- GCP project with Firestore and Cloud Storage enabled (for review-notification-service)
- Application default credentials configured (`gcloud auth application-default login`)
- Eureka Server and Config Server running

### Start Order

1. Start platform services (`backend-platform`) first:
   - Eureka Server
   - Config Server
   - API Gateway

2. Start business services (`backend-services`) after platform is ready:
   - User Service
   - Event Booking Service
   - Review & Notification Service

### Steps

```bash
# Build all services
cd user-service && ./mvnw clean package
cd ../event-booking-service && ./mvnw clean package
cd ../review-notification-service && ./mvnw clean package

# Run each service
cd user-service && ./mvnw spring-boot:run
cd event-booking-service && ./mvnw spring-boot:run
cd review-notification-service && ./mvnw spring-boot:run
```

## Environment Variables

| Variable | Service | Description |
|----------|---------|-------------|
| `DB_PASSWORD` | User Service, Event Booking Service | PostgreSQL database password |
| `spring.cloud.gcp.project-id` | Review & Notification Service | GCP project ID |
| `app.gcs.bucket-name` | Review & Notification Service | Cloud Storage bucket name |
