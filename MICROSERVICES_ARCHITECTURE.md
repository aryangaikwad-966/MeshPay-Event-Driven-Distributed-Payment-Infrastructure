# MeshPay Microservices Architecture

## Overview

This document describes the microservices architecture for the MeshPay payment system, which has been refactored from a monolithic application into independent, scalable microservices.

## Architecture

### Services

1. **Payment Service** (port 8081)
   - Handles payment processing and settlement
   - Consumes Kafka commands from Saga Service
   - Implements Transactional Outbox pattern for reliable event publishing
   - Database: `meshpay_payment`

2. **Saga Service** (port 8082)
   - Orchestrates distributed payment transactions using Saga pattern
   - Sends Kafka commands to Payment Service
   - Maintains persistent saga state
   - Database: `meshpay_saga`

3. **Eureka Server** (port 8761)
   - Service discovery registry
   - Enables dynamic service registration and discovery

4. **API Gateway** (port 8080)
   - Single entry point for all microservices
   - Routes requests to appropriate services
   - Implements circuit breaking with Resilience4j

## Communication Patterns

### Kafka Event-Driven Communication

The Saga and Payment services communicate via Kafka commands:

- **saga-validate-payment-command**: Saga → Payment (validate payment)
- **saga-request-settlement-command**: Saga → Payment (request settlement)
- **saga-complete-settlement-command**: Saga → Payment (complete settlement)

### Service Discovery

All services register with Eureka Server and discover each other dynamically via service names:
- `payment-service`
- `saga-service`
- `api-gateway`

## Key Patterns

### Saga Pattern

The Saga Service implements the orchestration pattern for distributed transactions:
- State machine with persistent state in database
- Compensation actions for each failure scenario
- Kafka-based command sending instead of direct service calls

### Transactional Outbox Pattern

The Payment Service ensures reliable event publishing:
- Events written to outbox table within same transaction as business data
- Background processor publishes events to Kafka
- Pessimistic locking prevents concurrent processing

### Idempotency

Defense-in-depth approach:
- Redis for distributed caching
- Database-level unique constraint on `packetHash` in transactions table
- Idempotency checks before processing payments

## Infrastructure

### Docker Compose

The `docker-compose.yml` orchestrates all services:
- PostgreSQL databases (separate for Payment and Saga)
- Redis for distributed caching
- Kafka for event streaming
- Eureka Server for service discovery
- API Gateway for routing
- Payment Service
- Saga Service

### Configuration

Each service has its own `application.yml` with:
- Database connection settings
- Kafka configuration
- Redis configuration
- Eureka client settings
- Resilience4j circuit breaker and retry settings
- Observability (Actuator, Prometheus, OpenTelemetry)

## Observability

### Metrics
- Custom metrics: `PaymentMetrics`, `SagaMetrics`
- Prometheus endpoint: `/actuator/prometheus`
- Micrometer integration

### Tracing
- OpenTelemetry tracing bridge
- OTLP exporter
- Distributed tracing across services

### Health Checks
- Actuator health endpoint: `/actuator/health`
- Health checks configured in Docker Compose

## Security

- Spring Security with OAuth2 Resource Server
- JWT validation support
- OAuth2 JOSE for JWT handling

## Resilience

### Circuit Breakers
- Payment Service: Circuit breaker for sagaService
- Saga Service: Circuit breaker for paymentService
- API Gateway: Circuit breakers for both services

### Retry
- Automatic retry with configurable max attempts
- Exponential backoff

## Running the System

### Prerequisites
- Docker and Docker Compose
- Java 17
- Maven

### Start Services

```bash
docker-compose up -d
```

### Stop Services

```bash
docker-compose down
```

### Build Individual Services

```bash
# Payment Service
cd meshpay-payment-service
mvn clean package

# Saga Service
cd meshpay-saga-service
mvn clean package

# Eureka Server
cd meshpay-eureka-server
mvn clean package

# API Gateway
cd meshpay-api-gateway
mvn clean package
```

## API Endpoints

### API Gateway
- Health: `http://localhost:8080/actuator/health`
- Payment routes: `http://localhost:8080/api/payment/**`
- Saga routes: `http://localhost:8080/api/saga/**`

### Eureka Server
- Dashboard: `http://localhost:8761`
- Health: `http://localhost:8761/actuator/health`

### Payment Service
- Health: `http://localhost:8081/actuator/health`
- Metrics: `http://localhost:8081/actuator/prometheus`
- Swagger: `http://localhost:8081/swagger-ui.html`

### Saga Service
- Health: `http://localhost:8082/actuator/health`
- Metrics: `http://localhost:8082/actuator/prometheus`
- Swagger: `http://localhost:8082/swagger-ui.html`

## Testing

### Integration Tests

```bash
# Payment Service integration tests
cd meshpay-payment-service
mvn test

# Saga Service integration tests
cd meshpay-saga-service
mvn test
```

## Migration Path

The monolithic application has been refactored into microservices while preserving:
- Kafka event-driven architecture
- Transactional Outbox pattern
- Idempotency mechanisms
- Saga orchestration pattern
- Resilience4j configuration
- Observability stack
- Security configuration

## Next Steps

- [ ] Add API controllers for REST endpoints
- [ ] Implement outbox background processor
- [ ] Add Kafka event consumers for payment events
- [ ] Configure OAuth2 provider
- [ ] Add comprehensive error handling
- [ ] Implement rate limiting
- [ ] Add distributed tracing configuration
- [ ] Set up monitoring dashboards (Grafana/Prometheus)
