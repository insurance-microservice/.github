# Insurance Microservices Setup Guide

This guide provides step-by-step instructions to set up and run the Insurance Microservices architecture using Docker Compose.

## Prerequisites

Before you begin, ensure you have the following installed on your system:
- Git
- Docker
- Docker Compose
- A terminal or command prompt

## Setup Instructions

### Step 1: Create Project Directory

Create a new directory for the project and navigate to it:

```bash
mkdir -p ~/projects
cd ~/projects
```

### Step 2: Clone All Repositories

Clone all microservice repositories as siblings in the same parent directory. All services must be at the same level for Docker Compose to properly reference them:

```bash
git clone https://github.com/insurance-microservice/infra.git
git clone https://github.com/insurance-microservice/api-gateway.git
git clone https://github.com/insurance-microservice/auth-service.git
git clone https://github.com/insurance-microservice/customer-service.git
git clone https://github.com/insurance-microservice/policy-service.git
git clone https://github.com/insurance-microservice/claim-service.git
git clone https://github.com/insurance-microservice/payment-service.git
git clone https://github.com/insurance-microservice/underwriting-service.git
```

This creates the required directory structure:

```
projects/
├── infra/                    ← Docker Compose runs from here
├── api-gateway/              ← Entry point
├── auth-service/             ← Authentication & Authorization
├── customer-service/         ← Customer Management
├── policy-service/           ← Policy Management
├── claim-service/            ← Claim Management
├── payment-service/          ← Payment Processing
└── underwriting-service/     ← Underwriting Logic
```

**Important**: The `infra` directory contains the `docker-compose.yml` file that references all other services using relative paths. Ensure all repositories are cloned as siblings.

### Step 3: Open Infrastructure Directory

Open the `infra` directory where the Docker Compose configuration is located. From the parent directory containing all repositories:

```bash
cd infra
```

Docker Compose will use relative paths (`../auth-service`, `../policy-service`, etc.) to locate and build all microservices from this directory.

### Step 4: Review Configuration Files

Before running the services, review the configuration files:

- `.env` - Environment variables for database and services
- `docker-compose.yml` - Docker Compose configuration
- `database/init.sql` - Database initialization script

Ensure that all paths and environment variables are correctly configured.

### Step 5: Build and Start Services

Run Docker Compose with the build flag to build all images and start the services. Choose one of the two options below:

**Option 1: Run in Foreground (View Logs)**

```bash
docker-compose up --build
```

This command will display logs from all services in real-time. Press CTRL+C to stop (services will be stopped).

**Option 2: Run in Background (Detached Mode)**

```bash
docker-compose up -d --build
```

This command runs containers in the background without blocking the terminal. Use `docker logs [service-name]` to view logs when needed.

Both commands will:
- Build Docker images for all microservices
- Start all services including infrastructure components
- Create necessary networks and volumes
- Initialize the database

### Step 6: Verify Services Are Running

Once all services have started, verify that they are running correctly by checking running containers:

```bash
docker ps
```

You should see all 11 containers running with status `Up`. The microservices platform consists of the following components:

**Core Services:**
- `gateway-service` - API Gateway (port 8080) - Entry point for all API requests
- `auth-service` - Authentication Service - Handles authentication and authorization
- `customer-service` - Customer Service - Manages customer information
- `policy-service` - Policy Service - Manages insurance policies
- `claim-service` - Claim Service - Handles insurance claims
- `payment-service` - Payment Service - Processes payments
- `underwriting-service` - Underwriting Service - Handles underwriting processes

**Infrastructure:**
- `postgres` - PostgreSQL Database (port 5432, healthy) - Primary database
- `kafka` - Kafka Message Broker - Asynchronous communication
- `zookeeper` - Zookeeper Coordination - Kafka coordination and synchronization
- `kafdrop` - Kafka UI (port 9000) - Kafka monitoring and topic management

**All containers should show status `Up` and `(healthy)` for postgres. If any container shows `Exited`, check logs with `docker logs [service-name]`.**

### Step 7: Access Services

After the services are running, you can access them at:

- API Gateway: http://localhost:8080
- Kafka UI (Kafdrop): http://localhost:9000
- PostgreSQL: localhost:5432

## Environment Variables

The `.env` file (located in the `infra` directory) contains all configuration variables for the microservices platform:

### Database Configuration
- `DB_HOST` - Database host (default: `postgres`)
- `DB_PORT` - Database port (default: `5432`)
- `DB_NAME` - Database name (default: `insurance_db`)
- `DB_USER` - Database user (default: `db_user`)
- `DB_PASSWORD` - Database password (default: `secret4321!!`)

### JWT Configuration
- `JWT_SECRET` - Secret key for JWT token signing
- `JWT_EXPIRATION` - JWT token expiration time in milliseconds (default: `3600000`)

### Admin Credentials
- `ADMIN_USERNAME` - Admin username (default: `admin`)
- `ADMIN_EMAIL` - Admin email (default: `admin@local.prod`)
- `ADMIN_PASSWORD` - Admin password (default: `admin321`)

> **Note**: Use these credentials for admin access to the API Gateway

### Kafka Configuration
- `KAFKA_BOOTSTRAP_SERVERS` - Kafka bootstrap servers (default: `kafka:9092`)

### Service URLs
- `AUTH_SERVICE_URL` - Auth Service URL
- `CUSTOMER_SERVICE_URL` - Customer Service URL
- `POLICY_SERVICE_URL` - Policy Service URL
- `PAYMENT_SERVICE_URL` - Payment Service URL
- `CLAIM_SERVICE_URL` - Claim Service URL
- `UNDERWRITING_SERVICE_URL` - Underwriting Service URL

## Stopping Services

To stop all running services, press CTRL+C in the terminal where Docker Compose is running, or run:

```bash
docker-compose down
```

To stop services and remove volumes:

```bash
docker-compose down -v
```

## Troubleshooting

### Services Not Starting

- Check that all ports are available: 5432, 2181, 29092, 9000, 8080
- Review logs with `docker logs [service-name]`
- Ensure Docker daemon is running

### Database Connection Issues

- Verify PostgreSQL is running: `docker logs postgres`
- Check database credentials in `.env` file
- Ensure `database/init.sql` is present in the infra directory

### Kafka Issues

- Verify Zookeeper is running: `docker logs zookeeper`
- Check Kafka logs: `docker logs kafka`
- Access Kafdrop at http://localhost:9000 to monitor Kafka topics

### Port Conflicts

If ports are already in use:
- Modify port mappings in `docker-compose.yml`
- Or stop conflicting services on your system

## Development Workflow

To rebuild a specific service after code changes:

```bash
docker-compose up --build [service-name]
```

For example, to rebuild only the auth-service:

```bash
docker-compose up --build auth-service
```

## Additional Commands

Execute a command in a running container:
```bash
docker exec [service-name] bash
```

## Documentation

For comprehensive documentation about the system architecture, API endpoints, and database design, please refer to the documentation files in this directory:

- [System Architecture](SYSTEM_ARCHITECTURE.md) - Overview of microservices architecture, components, and interactions
- [Process Flowcharts](FLOWCHART.md) - Visual flowcharts of main business processes
- [Entity Relationship Diagram](ERD.md) - Database schema and relationships
- [Postman Collection](POSTMAN_COLLECTION.json) - Complete API collection for testing all endpoints

### Quick Documentation Access

Import the Postman collection into Postman to test all API endpoints. The collection includes:
- **Authentication**: Register and login endpoints
- **Customer Management**: CRUD operations (Create, Read, Update, Delete)
- **Policy Management**: Create policies and query policy details
- **Claim Management**: Submit and approve insurance claims
- **Payment Management**: Create and update payments
- **Automation Test Flow**: Complete end-to-end workflow (9 sequential steps)

**Testing the Automation Flow**:
1. Import `POSTMAN_COLLECTION.json` into Postman
2. Go to "Automation Test Flow" folder
3. Run all requests in sequence - they will automatically capture and use the JWT token
4. Uses admin credentials: `admin` / `admin321`

All endpoints use the API Gateway base URL: http://localhost:8080
