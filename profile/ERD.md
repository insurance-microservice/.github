# Entity Relationship Diagram (ERD)

## Database Schema Overview

The system uses PostgreSQL with separate schemas for each microservice to ensure data isolation while maintaining relational integrity across services.

## User Service Schema (user_svc)

```
users
+---+----------+----------+----------+----------+----------+----------+
| * | user_id  | username | email    | password | role     | is_active|
|   | (BIGINT) | (VARCHAR)| (VARCHAR)| (VARCHAR)| (VARCHAR)| (BOOLEAN)|
+---+----------+----------+----------+----------+----------+----------+
|   |          |          |          | _hash    |          | DEFAULT  |
|   |          | UNIQUE   | UNIQUE   |          |          | TRUE     |
+---+----------+----------+----------+----------+----------+----------+

Primary Key: user_id (BIGSERIAL)
Indexes: username (UNIQUE), email (UNIQUE)
Schema: user_svc
```

**Fields:**
- user_id: Unique identifier (Primary Key)
- username: Unique username for login
- email: Unique email address
- password_hash: Hashed password (BCrypt)
- role: User role (USER, ADMIN)
- is_active: Account activation status (DEFAULT: true)
- created_at: Account creation timestamp

---

## Customer Service Schema (customer_svc)

```
customer
+---+----------+----------+----------+----------+-----+----------+
| * |customer  |full_name | email    | phone    |age  |created   |
|   | _id      |          |          |          |     | _at      |
| 1 |(BIGINT)  |(VARCHAR) |(VARCHAR) |(VARCHAR) |(INT)|(TIMESTAMP)|
+---+----------+----------+----------+----------+-----+----------+
|   |          |NOT NULL  |UNIQUE    |          |NOT  |DEFAULT   |
|   |          |          |NOT NULL  |          |NULL | NOW()    |
+---+----------+----------+----------+----------+-----+----------+

Primary Key: customer_id (BIGSERIAL)
Indexes: email (UNIQUE)
Foreign Relations: Referenced by policy_svc.policy (customer_id)
```

**Fields:**
- customer_id: Unique identifier (Primary Key)
- full_name: Customer full name
- email: Unique email address
- phone: Contact phone number
- age: Customer age
- created_at: Account creation timestamp

---

## Policy Service Schema (policy_svc)

```
policy
+---+--------+----------+----------+----------+----------+----------+
| * |policy  |customer  |policy    |type      |premium   |coverage  |
|   | _id    | _id      |_number   |          |_amount   |_amount   |
| 1 |(BIGINT)|(BIGINT)  |(VARCHAR) |(VARCHAR) |(DECIMAL) |(DECIMAL) |
+---+--------+----------+----------+----------+----------+----------+
|   |        |NOT NULL  |          |NOT NULL  |          |          |
+---+--------+----------+----------+----------+----------+----------+

+----------+----------+----------+----------+
| status   | start    | end      |created   |
|          | _date    | _date    | _at      |
|(VARCHAR) |(TIMESTAMP)|(TIMESTAMP)|(TIMESTAMP)
+----------+----------+----------+----------+
|NOT NULL  |          |          |DEFAULT   |
|          |          |          | NOW()    |
+----------+----------+----------+----------+

Primary Key: policy_id (BIGSERIAL)
Foreign Key: customer_id -> customer_svc.customer(customer_id)
Indexes: customer_id, policy_number
Referenced by: claim_svc.claim (policy_id), payment_svc.payment (policy_id)
```

**Fields:**
- policy_id: Unique identifier (Primary Key)
- customer_id: Reference to customer (Foreign Key)
- policy_number: Unique policy identification number
- type: Type of insurance policy
- premium_amount: Premium amount in decimal
- coverage_amount: Coverage amount in decimal
- status: Policy status (PENDING, PENDING_UNDERWRITING, APPROVED, REJECTED, ACTIVE)
- created_at: Policy creation timestamp
- start_date: Policy start date
- end_date: Policy end date
- created_at: Policy creation timestamp

---

## Claim Service Schema (claim_svc)

```
claim
+---+--------+--------+----------+----------+----------+----------+
| * |claim   |policy  |claim     |description|status   |created   |
|   | _id    | _id    |_number   |          |         | _at      |
| 1 |(BIGINT)|(BIGINT)|(VARCHAR) |(VARCHAR) |(VARCHAR)|(TIMESTAMP)|
+---+--------+--------+----------+----------+----------+----------+
|   |        |NOT NULL|          |          |NOT NULL |DEFAULT   |
|   |        |        |          |          |         | NOW()    |
+---+--------+--------+----------+----------+----------+----------+

+----------+
| approved |
| _at      |
|(TIMESTAMP)|
+----------+
|          |
+----------+

Primary Key: claim_id (BIGSERIAL)
Foreign Key: policy_id -> policy_svc.policy(policy_id)
Indexes: policy_id, claim_number
```

**Fields:**
- claim_id: Unique identifier (Primary Key)
- policy_id: Reference to policy (Foreign Key)
- claim_number: Unique claim identification number
- description: Claim description
- status: Claim status (PENDING, APPROVED)
- created_at: Claim submission timestamp
- approved_at: Claim approval timestamp

---

## Payment Service Schema (payment_svc)

```
payment
+---+--------+--------+--------+--------+----------+----------+
| * |payment |policy  |amount  |status  |billing   |payment   |
|   | _id    | _id    |        |        |_date     | _date    |
| 1 |(BIGINT)|(BIGINT)|(DECIMAL)|(VARCHAR)|(TIMESTAMP)|(TIMESTAMP)|
+---+--------+--------+--------+--------+----------+----------+
|   |        |NOT NULL|        |NOT NULL|          |          |
+---+--------+--------+--------+--------+----------+----------+

Primary Key: payment_id (BIGSERIAL)
Foreign Key: policy_id -> policy_svc.policy(policy_id)
Indexes: policy_id
```

**Fields:**
- payment_id: Unique identifier (Primary Key)
- policy_id: Reference to policy (Foreign Key)
- amount: Payment amount in decimal
- status: Payment status (UNPAID, PAID, FAILED)
- billing_date: Billing date
- payment_date: Actual payment date

---

## Relationships Summary

```
user_svc.users (1) -----> (1) customer_svc.customer  [Regular USER: 1:1]
user_svc.users (1) -----> (N) customer_svc.customer  [ADMIN: 1:many]
                         
customer_svc.customer (1) -----> (N) policy_svc.policy
                         
policy_svc.policy (1) -----> (N) claim_svc.claim
policy_svc.policy (1) -----> (N) payment_svc.payment
```

**User-to-Customer Relationship:**
- Regular USER: 1 user → 1 customer (one-to-one)
- ADMIN: 1 user → many customers (one-to-many)

### Cross-Service Communication

1. **Policy Service -> Underwriting Service (Kafka)**
   - Policy creation events flow through underwriting-topic
   - Underwriting decisions return via underwriting-response topic

2. **Payment Service -> Policy Service (Kafka)**
   - Payment events flow through policy-topic
   - Policy status updates based on payment status

3. **Claim Service -> Policy Service (REST)**
   - Claim service validates policies via REST client
   - Ensures only valid policies can have claims

---

## Schema Isolation Pattern

Each microservice has its own database schema:
- user_svc: Authentication and user management
- customer_svc: Customer data
- policy_svc: Insurance policies
- payment_svc: Payment transactions
- claim_svc: Claims processing
- underwriting_svc: Underwriting evaluations

This ensures:
- Data isolation and autonomy
- Independent scaling
- Clear service boundaries
- Reduced coupling between services
