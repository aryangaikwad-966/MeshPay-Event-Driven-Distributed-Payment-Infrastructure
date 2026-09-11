# MeshPay: Event-Driven Distributed Payment Infrastructure

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Java: 17+](https://img.shields.io/badge/Java-17+-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot: 3.3](https://img.shields.io/badge/Spring%20Boot-3.3-green.svg)](https://spring.io/projects/spring-boot)
[![Spring Cloud: 2023.0](https://img.shields.io/badge/Spring%20Cloud-2023.0-blue.svg)](https://spring.io/projects/spring-cloud)

MeshPay is a microservices-based distributed payment system demonstrating event-driven architecture, Kafka-based inter-service communication, transactional outbox pattern for reliable event publishing, and Saga orchestration with persistent state management for distributed transaction coordination.

---

## 🏗️ Architecture

### Microservices

MeshPay consists of the following microservices:

| Service | Port | Description |
|---------|------|-------------|
| **API Gateway** | 8080 | Single entry point, routing, authentication |
| **Payment Service** | 8081 | Payment processing, account management |
| **Saga Service** | 8082 | Distributed transaction orchestration |
| **Eureka Server** | 8761 | Service discovery and registration |

### Infrastructure

- **PostgreSQL 16** - Primary database for each service
- **Redis 7** - Caching and distributed locking
- **Apache Kafka 3.x** - Event-driven messaging
- **Zookeeper** - Kafka coordination

### Architecture Diagram

```
Client → API Gateway (8080) → Payment Service (8081) + Saga Service (8082)
                                      ↓                    ↓
                                  PostgreSQL (5433)     PostgreSQL (5434)
                                      ↓                    ↓
                                  Kafka (9092) ←─────────┘
                                      ↓
                                  Redis (6379)
                                      ↓
                              Eureka Server (8761)
```

---

## 🚀 Getting Started

### Prerequisites

- Java 17 or higher
- Docker & Docker Compose
- Maven 3.8+

### Quick Start with Docker Compose

```bash
# Start all services and infrastructure
docker-compose up -d

# View logs
docker-compose logs -f

# Stop services
docker-compose down
```

### Manual Setup

Start infrastructure services first, then run each microservice:

```bash
# Terminal 1 - Eureka Server
cd meshpay-eureka-server
mvn spring-boot:run

# Terminal 2 - Payment Service
cd meshpay-payment-service
mvn spring-boot:run

# Terminal 3 - Saga Service
cd meshpay-saga-service
mvn spring-boot:run

# Terminal 4 - API Gateway
cd meshpay-api-gateway
mvn spring-boot:run
```

---

## 📚 Services Documentation

### API Gateway (Port 8080)

**Health Check:**
```bash
curl http://localhost:8080/actuator/health
```

**Routes:**
- `/api/payment/**` → Payment Service
- `/api/saga/**` → Saga Service

### Payment Service (Port 8081)

**Health Check:**
```bash
curl http://localhost:8081/actuator/health
```

**API Endpoints:**
- `POST /api/payment/validate` - Validate payment
- `POST /api/payment/settlement` - Request settlement
- `POST /api/payment/complete` - Complete settlement
- `GET /api/payment/accounts` - List accounts
- `GET /api/payment/transactions` - List transactions

**Swagger UI:**
```
http://localhost:8081/swagger-ui/index.html
```

### Saga Service (Port 8082)

**Health Check:**
```bash
curl http://localhost:8082/actuator/health
```

**API Endpoints:**
- `POST /api/saga/start` - Start new saga
- `POST /api/saga/validate` - Validate payment in saga
- `POST /api/saga/settlement` - Request settlement in saga
- `POST /api/saga/complete` - Complete settlement in saga
- `GET /api/saga/status/{sagaId}` - Get saga status

**Swagger UI:**
```
http://localhost:8082/swagger-ui/index.html
```

### Eureka Server (Port 8761)

**Dashboard:**
```
http://localhost:8761
```

---

## 🔑 Key Patterns

### Transactional Outbox Pattern

Events are written to the outbox table in the same transaction as business data, ensuring reliable event publication with at-least-once delivery semantics:

```java
@Transactional
public void processPayment(Payment payment) {
    // Business logic
    paymentRepository.save(payment);
    
    // Event in same transaction
    outboxService.saveEvent(paymentEvent, "payment-validated", payment.getPacketHash());
}
```

A scheduled processor publishes unprocessed events to Kafka every 5 seconds using pessimistic locking to prevent concurrent processing. Consumer-side idempotency ensures exactly-once processing.

### Saga Pattern

Distributed transactions are orchestrated using the Saga pattern with persistent state management:

1. **Saga Start** - Initialize saga state in database
2. **Payment Validation** - Validate payment details via Kafka command
3. **Settlement Request** - Request settlement via Kafka command
4. **Settlement Completion** - Finalize transaction via Kafka command
5. **Compensation** - Rollback actions on failure

Saga state is persisted to PostgreSQL to survive orchestrator crashes. Each step has defined compensation actions for failure recovery.

### Event-Driven Architecture

Services communicate via Kafka topics:

- `saga-validate-payment-command` - Commands from Saga to Payment
- `saga-request-settlement-command` - Settlement request commands
- `saga-complete-settlement-command` - Completion commands
- `payment-validated` - Validation completion events
- `payment-settlement-requested` - Settlement request events
- `payment-settled` - Final settlement events
- `payment-failed` - Failure events
- `saga-completed` - Saga completion events
- `saga-failed` - Saga failure events

### Idempotency & Duplicate Prevention

Defense-in-depth approach prevents duplicate payment processing:

1. **Redis SETNX** - Distributed idempotency using SHA-256 hash of ciphertext as tamper-proof key
2. **Database Unique Constraints** - Unique constraint on packetHash in transactions table as fallback
3. **Consumer-Side Idempotency** - EventProcessed table tracks processed Kafka events

This ensures exactly-once payment settlement even under concurrent bridge node uploads.

---

## 🔐 Security

### Profile-Based Configuration

**Dev Profile (default):**
- Authentication disabled for testing
- All endpoints permitted

**Prod Profile:**
- OAuth2 JWT validation enabled
- Actuator, Swagger, and health endpoints public
- All other endpoints require authentication

### Configuration

Set the profile in `application.yml` or environment variable:

```yaml
spring:
  profiles:
    active: prod
```

Or:
```bash
export SPRING_PROFILES_ACTIVE=prod
```

---

## 📊 Observability

### Health Checks

Each service exposes health endpoints:

```bash
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
```

### Metrics (Prometheus)

```bash
curl http://localhost:8081/actuator/prometheus
curl http://localhost:8082/actuator/prometheus
```

### Distributed Tracing (OpenTelemetry)

Services are configured for distributed tracing with OpenTelemetry:

- **Sampling Probability**: 1.0 (100% for development)
- **OTLP Exporter**: Configured for Jaeger/Tempo compatibility
- **Service Names**: Unique per service for trace correlation

Tracing provides visibility into request flows across service boundaries.

### Custom Metrics

- Payment validation count
- Settlement success/failure rates
- Saga completion rates
- Active saga count
- Outbox processing metrics

### Resilience Patterns

**Circuit Breakers:**
- Payment Service: Circuit breaker for saga service calls
- Saga Service: Circuit breaker for payment service calls
- API Gateway: Circuit breakers for both services
- Configured with 50% failure rate threshold, 10s wait duration

**Retry Configuration:**
- Automatic retry with 3 max attempts
- 1s wait duration between retries
- Exponential backoff for transient failures

**Service Discovery:**
- Eureka Server for dynamic service registration
- Gateway discovers services via service names
- Health monitoring and load balancing support

---

## 🛠️ Development

### Project Structure

```
Meshpay/
├── meshpay-api-gateway/
│   ├── src/main/java/com/demo/meshpay/gateway/
│   └── pom.xml
├── meshpay-payment-service/
│   ├── src/main/java/com/demo/meshpay/payment/
│   │   ├── config/
│   │   ├── controller/
│   │   ├── model/
│   │   ├── repository/
│   │   ├── service/
│   │   ├── processor/
│   │   └── PaymentServiceApplication.java
│   └── pom.xml
├── meshpay-saga-service/
│   ├── src/main/java/com/demo/meshpay/saga/
│   │   ├── config/
│   │   ├── controller/
│   │   ├── consumer/
│   │   ├── events/
│   │   ├── model/
│   │   ├── repository/
│   │   ├── service/
│   │   ├── processor/
│   │   └── SagaServiceApplication.java
│   └── pom.xml
├── meshpay-eureka-server/
│   ├── src/main/java/com/demo/meshpay/eureka/
│   │   └── EurekaServerApplication.java
│   └── pom.xml
├── docker-compose.yml
├── test-microservices.sh
└── README.md
```

### Building

```bash
# Build all services
cd meshpay-payment-service && mvn clean package
cd ../meshpay-saga-service && mvn clean package
cd ../meshpay-api-gateway && mvn clean package
cd ../meshpay-eureka-server && mvn clean package
```

### Testing

```bash
# Run integration tests
cd meshpay-payment-service && mvn test
cd ../meshpay-saga-service && mvn test
```

### End-to-End Testing

```bash
# Run the test script
./test-microservices.sh
```

This script:
- Starts all services via Docker Compose
- Verifies service health
- Tests Eureka registration
- Tests API Gateway routing
- Verifies Kafka topics
- Tests basic API endpoints

---

## 🔧 Configuration

### Environment Variables

Key environment variables can be set in `docker-compose.yml`:

```yaml
environment:
  - SPRING_PROFILES_ACTIVE=prod
  - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/meshpay_payment
  - SPRING_REDIS_HOST=redis
  - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

### Database Configuration

Each service has its own database:

- **Payment Service**: `meshpay_payment` (port 5433)
- **Saga Service**: `meshpay_saga` (port 5434)

---

## 📝 Technical Stack

- **Java**: 17
- **Spring Boot**: 3.3.5
- **Spring Cloud**: 2023.0.3
- **Spring Cloud Gateway**: 4.1.x
- **Spring Cloud Netflix Eureka**: 4.1.x
- **PostgreSQL**: 16
- **Redis**: 7
- **Kafka**: 3.x
- **Resilience4j**: 2.1.0
- **Lombok**: 1.18.30
- **Maven**: 3.8+

---

## 🐛 Troubleshooting

**Service won't start:**
```bash
docker-compose ps                    # Check infrastructure status
docker-compose logs payment-service # Check service logs
```

**Kafka issues:**
```bash
docker-compose exec kafka kafka-topics --bootstrap-server localhost:9092 --list
```

**Service discovery:**
```bash
curl http://localhost:8761/eureka/apps  # Check Eureka registration
```

---

<div align="center">
  <h3>MeshPay Microservices Architecture</h3>
  <p>Event-driven distributed payment infrastructure with Saga orchestration</p>
</div>
