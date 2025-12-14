# Insurance Microservices System Architecture

## Overview

The Insurance Microservices system is a distributed architecture designed to handle insurance policy management, claims processing, payments, customer management, and authentication. The system uses a microservices pattern with an API Gateway for routing, Kafka for asynchronous messaging, and PostgreSQL for data persistence.

## System Architecture Diagram

```
        Client
          │ HTTP
          ▼
    API Gateway (8080)
      │
      ├──→ Auth Service
      ├──→ Customer Service
      ├──→ Policy Service ←──HTTP── Claim Service
      ├──→ Payment Service
      └──→ Underwriting Service

    Kafka Message Flow:
    
    Policy Service ─→ underwriting-requests ─→ Underwriting Service
                                                       │
                                                       ↓
    Policy Service ←─ underwriting-results ←────────────
    
    Policy Service ←─ payment-responses ←─ Payment Service
    
    PostgreSQL (All Services)
```

**Communication Channels:**
- **HTTP**: Client ↔ Gateway, Gateway ↔ Services, Claim ↔ Policy (REST call)
- **Kafka Topics**:
  - `underwriting-requests`: Policy → Underwriting (policy evaluation requests)
  - `underwriting-results`: Underwriting → Policy (evaluation results)
  - `payment-responses`: Payment → Policy (payment status updates)
- **PostgreSQL**: All services read/write (isolated schemas)

## Architecture Components

### 1. API Gateway
**Container:** `gateway-service` | **Port:** 8080 (exposed)

The entry point for all client requests. Routes requests to appropriate microservices based on URL paths.

**Routes:**
- `/api/v1/auth/**` → `auth-service:8080`
- `/api/v1/customer/**` → `customer-service:8080`
- `/api/v1/policy/**` → `policy-service:8080`
- `/api/v1/payment/**` → `payment-service:8080`
- `/api/v1/claim/**` → `claim-service:8080`
- `/api/v1/underwriting/**` → `underwriting-service:8080`

### 2. Auth Service
**Container:** `auth-service` | **Port:** 8080 (internal)

Handles JWT token generation and user authentication.

**Database:** `user_svc` schema in PostgreSQL
**Key Table:** users

### 3. Customer Service
**Container:** `customer-service` | **Port:** 8080 (internal)

Manages customer information and profiles.

**Database:** `customer_svc` schema in PostgreSQL
**Key Table:** customer

### 4. Policy Service
**Container:** `policy-service` | **Port:** 8080 (internal)

Central orchestrator for policy management. Publishes/consumes Kafka messages.

**Database:** `policy_svc` schema in PostgreSQL
**Key Table:** policy

**Kafka Topics:**
- Publishes: `underwriting-requests` → Underwriting Service
- Consumes: `underwriting-results` ← Underwriting Service
- Consumes: `payment-responses` ← Payment Service

### 5. Claim Service
**Container:** `claim-service` | **Port:** 8080 (internal)

Handles insurance claims processing.

**Database:** `claim_svc` schema in PostgreSQL
**Key Table:** claim

**Calls:** `policy-service:8080` for policy validation (REST)

### 6. Payment Service
**Container:** `payment-service` | **Port:** 8080 (internal)

Manages payment processing for policies.

**Database:** `payment_svc` schema in PostgreSQL
**Key Table:** payment

**Kafka Topics:**
- Publishes: `payment-responses` → Policy Service

### 7. Underwriting Service
**Container:** `underwriting-service` | **Port:** 8080 (internal)

Handles policy underwriting evaluation.

**Kafka Topics:**
- Consumes: `underwriting-requests` ← Policy Service
- Publishes: `underwriting-results` → Policy Service

## Data Persistence

### PostgreSQL Database
**Container:** `postgres` | **Port:** 5432

Central database serving all microservices with separate schemas for data isolation.

**Login Credentials:**
- **Host (Internal):** `postgres:5432`
- **Host (External):** `localhost:5432`
- **Username:** `db_user`
- **Password:** `secret4321!!`
- **Database:** `insurance_db`

**Example Connection String:**
```
psql -h localhost -U db_user -d insurance_db -p 5432
# Password: secret4321!!
```

**Schemas:**
- `user_svc`: Authentication and user management
- `customer_svc`: Customer information
- `policy_svc`: Insurance policies
- `claim_svc`: Insurance claims
- `payment_svc`: Payment records
- `underwriting_svc`: Underwriting data

## Message Broker

### Kafka
- Asynchronous communication between services
- Event streaming for policy, underwriting, and payment events
- Bootstrap servers: kafka:9092

**Topics:**
- underwriting-topic: Policy events requiring underwriting
- underwriting-response: Underwriting decisions
- payment-topic: Payment-related events
- policy-topic: Policy status updates

## Infrastructure Services

### Zookeeper
- Kafka cluster coordination
- Port: 2181

### Kafdrop
- Kafka UI for monitoring topics and messages
- Port: 9000
- Access: http://localhost:9000

## Network Architecture

All services are connected through a Docker bridge network named `insurance-network` for secure inter-service communication.

**Service Communication Paths:**
1. Client -> API Gateway (port 8080)
2. API Gateway -> Microservices (internal network)
3. Microservices <-> PostgreSQL (database network)
4. Microservices <-> Kafka (messaging network)

## Security Architecture

### Authentication & Authorization
- **JWT Token-based Authentication**: HS256 algorithm with secret key
- **OAuth2 Resource Server**: API Gateway validates tokens using secret key
- **Role-based Access Control (RBAC)**: Two roles - USER and ADMIN

### Access Control Rules

**Public Endpoints (No Authentication):**
- `POST /api/v1/auth/register` - Register new user
- `POST /api/v1/auth/login` - User login

**Customer Service:**
- `GET /api/v1/customer/all` - ADMIN only
- `GET /api/v1/customer/{customerId}` - USER, ADMIN
- `POST /api/v1/customer` - USER, ADMIN
- `PUT /api/v1/customer/{customerId}` - ADMIN only

**Policy Service:**
- `GET /api/v1/policy/customer/**` - USER, ADMIN
- `GET /api/v1/policy/**` - USER, ADMIN
- `POST /api/v1/policy` - USER, ADMIN
- `PUT /api/v1/policy/**` - ADMIN only

**Payment Service:**
- `GET /api/v1/payment/policy/**` - USER, ADMIN
- `GET /api/v1/payment/**` - USER, ADMIN
- `POST /api/v1/payment` - USER, ADMIN
- `PUT /api/v1/payment/**` - ADMIN only

**Claim Service:**
- `GET /api/v1/claim/policy/**` - USER, ADMIN
- `POST /api/v1/claim` - USER, ADMIN
- `GET /api/v1/claim/**` - USER, ADMIN
- `PUT /api/v1/claim/approve/**` - ADMIN only

**Underwriting Service:**
- All endpoints - ADMIN only

### Security Implementation
- Service-to-service communication over internal Docker network
- Database schema isolation per service
- JWT roles extracted from token claims and converted to Spring authorities

## Scalability Considerations

- Stateless microservices for horizontal scaling
- Kafka for decoupled asynchronous processing
- Separate database schemas for independent scaling
- API Gateway for load distribution

## Technology Stack

- Language: Java 17
- Framework: Spring Boot 4.0.0
- Cloud Framework: Spring Cloud 2025.1.0
- Message Broker: Kafka 7.7.0
- Database: PostgreSQL 16.9
- Container Platform: Docker & Docker Compose
- API Pattern: REST
- Authentication: JWT (JSON Web Tokens)
