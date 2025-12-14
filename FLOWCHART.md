# Insurance Microservices Process Flowcharts

## Overview
This document outlines the main business processes in the Insurance Microservices system. All flows use the API Gateway (http://localhost:8080) as the entry point and require JWT authentication (Bearer token in Authorization header), except for public endpoints (login, register).

---

## 1. User Authentication Flow

```
User (Credentials)
  |
  v
[API Gateway: POST /api/v1/auth/login]
  |
  v
[Auth Service (auth-service:8080)]
  |
  ├─ Validate email & password
  ├─ Query auth_svc.users table
  ├─ Verify password (BCrypt)
  ├─ Generate JWT token (HS256, HS256 algorithm)
  |
  v
[Return token with accessToken]
  |
  v
User (Stores JWT token)
```

---

## 2. User Registration Flow

```
User (Registration Data)
  |
  v
[API Gateway: POST /api/v1/auth/register]
  |
  v
[Auth Service (auth-service:8080)]
  |
  ├─ Validate input (username, email, password)
  ├─ Hash password using BCrypt
  ├─ Create user in auth_svc.users table
  ├─ Set default role: USER
  ├─ Set active status: true
  |
  v
[Return ApiResponse (success)]
  |
  v
User (Account Created)
```

---

## 3. Customer Management Flow

```
User (with JWT token)
  |
  v
[API Gateway: /api/v1/customer]
  |
  v
[Customer Service (customer-service:8080)]
  |
  ├─ GET /all (ADMIN only)
  |   └─ Query customer_svc.customer → List of customers
  |
  ├─ GET /{customerId} (USER/ADMIN)
  |   └─ Query customer_svc.customer → Customer
  |
  ├─ POST / (USER/ADMIN)
  |   ├─ Validate input
  |   ├─ Create customer record
  |   └─ Save to customer_svc.customer
  |
  └─ PUT /{customerId} (ADMIN only)
      ├─ Validate input
      ├─ Update customer record
      └─ Save to customer_svc.customer
  |
  v
User (Data returned or operation confirmed)
```

---

## 4. Policy Creation & Underwriting Flow

### 4a. Synchronous Response (Immediate)

```
User (with JWT)
  |
  v
[API Gateway: POST /api/v1/policy]
  |
  v
[Policy Service (policy-service:8080)]
  |
  ├─ Validate policy request
  ├─ Call Customer Service to verify customer exists
  ├─ Create policy record
  ├─ Generate policy_number
  ├─ Set status: PENDING_UNDERWRITING
  ├─ Set createdAt timestamp
  ├─ Save to policy_svc.policy
  |
  v
[Return ApiResponse (success)]
  |
  v
User (Policy Created - Status: PENDING_UNDERWRITING)
```

### 4b. Asynchronous Underwriting Process (Background)

```
[Policy Service - Kafka Producer]
  |
  ├─ Publish underwriting request message to Kafka
  |
  v
[Kafka Topic: underwriting-requests]
  |
  v
[Underwriting Service Consumer (underwriting-service:8080)]
  |
  ├─ Consume message
  ├─ Extract policy details
  ├─ Perform risk assessment:
  |   ├─ Analyze policy type
  |   ├─ Calculate risk score
  |   └─ Determine coverage eligibility
  |
  ├─ Generate underwriting decision: APPROVED or REJECTED
  |
  v
[Publish underwriting result to Kafka]
  |
  v
[Kafka Topic: underwriting-results]
  |
  v
[Policy Service Consumer]
  |
  ├─ Consume underwriting response
  ├─ Update policy status (APPROVED or REJECTED)
  ├─ Set approvedAt/rejectedAt timestamp
  ├─ Save changes to policy_svc.policy
  |
  v
[Policy status updated asynchronously]
```

---

## 5. Payment Processing Flow

### 5a. Synchronous Response (Immediate)

```
User (with JWT)
  |
  v
[API Gateway: POST /api/v1/payment]
  |
  v
[Payment Service (payment-service:8080)]
  |
  ├─ Validate payment request
  ├─ Call Policy Service to verify policy exists
  ├─ Create payment record
  ├─ Mock payment processing (success by default)
  ├─ Set status: PAID
  ├─ Set paymentDate: current timestamp
  ├─ Save to payment_svc.payment
  |
  v
[Return ApiResponse (success)]
  |
  v
User (Payment Processed - Status: PAID)
```

### 5b. Asynchronous Policy Activation (Background)

```
[Payment Service - Kafka Producer]
  |
  ├─ Publish payment confirmation to Kafka
  |
  v
[Kafka Topic: payment-responses]
  |
  v
[Policy Service Consumer]
  |
  ├─ Consume payment confirmation (paymentStatus: PAID)
  ├─ Retrieve associated policy
  ├─ Update policy status: ACTIVE
  ├─ Set activatedAt timestamp
  ├─ Save changes to policy_svc.policy
  |
  v
[Policy status updated: APPROVED → ACTIVE]
```

---

## 6. Claim Submission Flow

```
User (with JWT, Active Policy)
  |
  v
[API Gateway: POST /api/v1/claim]
  |
  v
[Claim Service (claim-service:8080)]
  |
  ├─ Validate claim request
  ├─ Call Policy Service (policy-service:8080)
  |   └─ Verify policy exists
  |   └─ Verify policy status is ACTIVE
  |
  ├─ If policy not ACTIVE → Return claim failed
  |
  ├─ Create claim record
  ├─ Generate claimNumber
  ├─ Set status: PENDING
  ├─ Set submittedAt timestamp
  ├─ Save to claim_svc.claim
  |
  v
[Return ApiResponse (success)]
  |
  v
User (Claim Submitted - Status: PENDING)
```

---

## 7. Claim Approval Flow

```
Admin User (with JWT)
  |
  v
[API Gateway: PUT /api/v1/claim/approve/{claimId}]
  |
  v
[Claim Service (claim-service:8080)]
  |
  ├─ Retrieve claim record by claimId
  ├─ Validate claim status is PENDING
  ├─ If status not PENDING → Return approval failed
  |
  ├─ Update status: APPROVED
  ├─ Set approvedAt timestamp
  ├─ Update approval notes if provided
  ├─ Save to claim_svc.claim
  |
  v
[Return ApiResponse (success)]
  |
  v
Admin User (Claim Approved)
```

---

## 8. Policy Query Flow

```
User (with JWT)
  |
  v
[API Gateway: /api/v1/policy]
  |
  v
[Policy Service (policy-service:8080)]
  |
  ├─ GET /customer/{customerId} (USER/ADMIN)
  |   └─ Query policy_svc.policy table
  |   └─ Return List of policies
  |
  └─ GET /{policyId} (USER/ADMIN)
      └─ Query policy_svc.policy table
      └─ Return Policy
  |
  v
User (Data retrieved)
```

---

## 9. Claim Query Flow

```
User (with JWT)
  |
  v
[API Gateway: /api/v1/claim]
  |
  v
[Claim Service (claim-service:8080)]
  |
  ├─ GET /policy/{policyId} (USER/ADMIN)
  |   └─ Query claim_svc.claim table
  |   └─ Return List of claims
  |
  └─ GET /{claimId} (USER/ADMIN)
      └─ Query claim_svc.claim table
      └─ Return Claim
  |
  v
User (Data retrieved)
```

---

## 10. Payment Query Flow

```
User (with JWT)
  |
  v
[API Gateway: /api/v1/payment]
  |
  v
[Payment Service (payment-service:8080)]
  |
  ├─ GET /policy/{policyId} (USER/ADMIN)
  |   └─ Query payment_svc.payment table
  |   └─ Return List of payments
  |
  └─ GET /{paymentId} (USER/ADMIN)
      └─ Query payment_svc.payment table
      └─ Return Payment
  |
  v
User (Data retrieved)
```

---

## 11. Complete Policy Lifecycle (End-to-End Sequence)

```
User/Admin
  |
  ├─ 1. REGISTER & AUTHENTICATE
  |   ├─ POST /auth/register → Account Created
  |   ├─ POST /auth/login → JWT Token
  |
  ├─ 2. CREATE CUSTOMER
  |   ├─ POST /customer → Customer Record
  |
  ├─ 3. CREATE POLICY
  |   ├─ POST /policy → Policy Created (PENDING_UNDERWRITING)
  |   ├─ Kafka: underwriting-requests
  |   ├─ Underwriting Service: Risk Assessment
  |   ├─ Kafka: underwriting-results
  |   └─ Policy Status: APPROVED or REJECTED
  |
  ├─ 4. PROCESS PAYMENT
  |   ├─ POST /payment → Payment Record (PAID)
  |   ├─ Kafka: payment-responses
  |   └─ Policy Status: ACTIVE
  |
  ├─ 5. SUBMIT CLAIM
  |   ├─ POST /claim → Claim Record (PENDING)
  |   └─ Requires: Policy status ACTIVE
  |
  ├─ 6. APPROVE CLAIM (Admin)
  |   ├─ PUT /claim/approve/{claimId} → Claim (APPROVED)
  |
  └─ 7. QUERY DATA
      ├─ GET /policy/{policyId} → Policy Status
      ├─ GET /claim/{claimId} → Claim Status
      └─ GET /payment/{paymentId} → Payment Status
```

---

## Service Communication Map

| Service | Communication | Target | Purpose |
|---------|--------------|--------|---------|
| Policy Service | REST | Customer Service | Validate customer exists |
| Payment Service | REST | Policy Service | Validate policy exists |
| Claim Service | REST | Policy Service | Validate policy status = ACTIVE before claim creation |
| Policy Service | Kafka Producer | underwriting-requests | Send policy for underwriting evaluation |
| Underwriting Service | Kafka Consumer | underwriting-requests | Receive policy for evaluation |
| Underwriting Service | Kafka Producer | underwriting-results | Send underwriting decision |
| Policy Service | Kafka Consumer | underwriting-results | Receive underwriting decision & update policy |
| Payment Service | Kafka Producer | payment-responses | Send payment confirmation |
| Policy Service | Kafka Consumer | payment-responses | Receive payment confirmation & activate policy |
