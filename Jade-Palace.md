🏯 Jade Palace

## 💼 Business Context: "FinSync: Scalable User Onboarding System"

FinSync is a fast-growing digital financial service platform that offers investment, wallet, and lending solutions across Southeast Asia.
The company is launching a major marketing campaign and expects a surge in new user sign-ups. To support this, they need a system that can:
Handle thousands of new registrations each day without slowing down.
Quickly and accurately collect user information.
Allow internal staff to review, update, and manage user data easily.
Keep the system secure, reliable, and ready to grow.
Help the company track what’s happening in real time and fix issues quickly.

# 5 Weeks of Practices to build real system

## Rules

1. use `main` branch for development.

## Week 1: Provide the API for registration

**Objective:**
Provide an API for registration that can be used by the frontend or other systems to register new users.

<details>
<summary>Business Requirements</summary>

- Create a registration API that allows new users to sign up.
- The API should accept user information such as name, email, phone number and ID.
- Validate the input data to ensure it meets the required format and criteria.
  - name: 2 - 200 characters
  - email: valid email format
  - phone number: valid phone number format
  - ID: valid ID format
- Users should receive a confirmation email after successful registration.
- High availability is a must
- The system should be able to handle a large number of concurrent requests without crashing or slowing down.

</details>

<details open>
<summary>Technical Requirements</summary>

## Overview

- Use a RESTful API design for the registration endpoint.
- Implement input validation using Go with the Gin framework.
- Unit test the API to ensure it works as expected.
- Integration test the API to ensure it works with the database and email service.
- Use a relational database PostgreSQL to store user information.
- Use Kafka for message queuing to handle high traffic.

## API Design specification

### Onboarder Service


<details>
<summary>Hints — Try it yourself first before looking at the hints</summary>

- Endpoint: POST /api/v1/register
  - Request Body:

```json
{
    "name": "string",
    "email": "string",
    "phoneNumber": "string",
    "thaiID": "string"
}
```

- Response:

```json
{
  "status": "success",
  "message": "Registration successful",
  "data": {
        "userID": "string"  // Generated unique user ID
    }
}
```

- Error Response:

```json
{
    "status": "error",
    "message": "Invalid input data"
}
```

- Validation:
  - Name: 2 - 200 characters
  - Email: valid email format
  - Phone Number: valid phone number format
  - Thai ID: valid Thai ID format

- Confirmation Email:
  - Send a confirmation email to the user after successful registration.
  - Use a third-party email service provider (e.g., SendGrid, Mailgun) to send emails.
  - Email template should include:
    - Subject: "Welcome to FinSync!"
    - Body: "Thank you for registering with FinSync. Your user ID is {userID}."
  - Use a message queue (e.g., Kafka) to handle email sending asynchronously.

### User Service

- Endpoint: GET /users
  
</details>

## System Architecture

| Layer     | Service Name      | Role                                                                                           |
|-----------|-------------------|------------------------------------------------------------------------------------------------|
| API       | onboarder-service | **Onboarder Service**: Receives registration requests, validates input, and publishes to Kafka |
| DB Writer | user-service      | **User Service**:Saves user information to DB, triggers follow-ups                             |
| Email     | notifier-service  | **Notifier Service**: Sends confirmation emails to users after successful registration         |
| Cache     | cache-updater     | **Cache Updater**: Maintains user Redis cache                                                  |

"onboarder-service" will be the entry point for user registration and will validate the input data. If the data is valid, it will publish a message to Kafka for further processing and store the data in Redis cache for quick access check duplicates with status "processing".

The "user-service" will listen to the Kafka topic and persist the data to PostgreSQL.

The "notifier-service" will also listen to the Kafka topic and send confirmation emails to users.

The "cache-updater" will update the Redis cache with the latest user data upon successful creation.

The "user-service" will listen to the Kafka topic and persist the data to PostgreSQL.

The "notifier-service" will also listen to the Kafka topic and send confirmation emails to users.

```mermaid
flowchart TD
  user[User: Registers as a new user]

  subgraph "FinSync Platform"
    onboarder[Onboarder Service\nValidates, checks Redis, sends Kafka event]
    user[User Service\nConsumes Kafka, stores in Postgres, emits events]
    notifier[Notifier Service\nSends confirmation email]
    cacheupdater[Cache Updater\nUpdates Redis cache]

    redis[Redis Cache]
    postgres[User Database - Postgres]
    kafka[Kafka - Event Streaming]
    email[SMTP Email Service]
  end

  user -->|Submits registration| onboarder
  onboarder -->|Check if exists| redis
  onboarder -->|Publish event: user.registration| kafka
  kafka -->|Event: user.registration| user
  user -->|Write to DB| postgres
  user -->|Emit event: user.created| kafka
  user -->|Emit event: user.confirmation| kafka
  kafka -->|Event: user.confirmation| notifier
  notifier -->|Send confirmation| email
  kafka -->|Event: user.created| cacheupdater
  cacheupdater -->|Update cache| redis
```


```mermaid
flowchart TD
  user[Public User: Registers]
  admin[Internal Admin: Manage Users]

  user -->|Submit form| OnboarderService

  subgraph OnboarderService
    checkRedis[Check Redis: Duplicate User?]
    validateInput[Validate Input Data]
    publishKafka[Publish Event: user.registration]
  end

  OnboarderService --> checkRedis
  checkRedis -->|Exists?| rejectRequest[Reject Duplicate Registration]
  checkRedis -->|Not Exists| validateInput
  validateInput --> publishKafka
  publishKafka --> KafkaTopic[Kafka Topic: user.registration]

  KafkaTopic --> UserService
  UserService -->|Save to| PostgresDB[Postgres DB: Save User]
  UserService -->|Emit Event| KafkaTopic2[Kafka Topic: user.created, user.confirmation]

  KafkaTopic2 --> NotifierService
  NotifierService -->|Send confirmation email| SMTPServer

  KafkaTopic2 --> CacheUpdater
  CacheUpdater -->|Update| RedisCache[Redis Cache: Save User]

  admin -->|Login| AdminFrontend
  AdminFrontend -->|API calls| UserService
  UserService -->|Return list| AdminFrontend

```

```mermaid
flowchart TD
  User -->|POST /register| OnboarderService
  OnboarderService -->|Check Redis| Redis
  Redis -->|Exists?| RejectDuplicate
  Redis -->|Not Exists| ValidateData
  ValidateData -->|Publish Event| KafkaTopic
  KafkaTopic --> UserService
  UserService -->|Save to| PostgresDB
  UserService -->|Emit Event| KafkaTopic2
  KafkaTopic2 --> NotifierService
  KafkaTopic2 --> CacheUpdater
  NotifierService -->|Send confirmation email| SMTP
  CacheUpdater -->|Update Redis| Redis

```

## Database Design

- Use PostgreSQL as the database to store user information.
- Create a table named "users" with the following schema:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    thai_id VARCHAR(20) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## User Service

- Consume the Kafka topic "user.registration" to process registration events.
- Validate the data and store it in the PostgreSQL database.
- Emit events to Kafka for confirmation and cache updates.

</details>

## Week 2: Provide UI for Admin to manage users

**Objective:**
Provide a UI for internal staff to manage users, including viewing, updating, and deleting user information.
<details>
<summary>Business Requirements</summary>
- Create a web-based UI for internal staff to manage users.
- The UI should allow staff to view a list of registered users.
- Staff should be able to view user details, including name, email, phone number, ID and registration date.
</details>

<details open>
<summary>Technical Requirements</summary>

## Overview

- Use a Next.js framework for the UI.
- Use Tailwind CSS for styling.
- UI should be responsive and user-friendly.
- Implement authentication and authorization for staff access.

## UI Design

- Create a dashboard page that displays a list of registered users.
- Each user entry should include:
  - Name
  - Email
  - Phone Number
  - Thai ID
  - Registration Date
- Provide a search bar to filter users by name or email.

## NOTE

- Techinique : Vertical slice / incremental delivery for each sprint
- Rate Limiting: 100 requests per minute per IP address

🎯 The Goal for the Team:
Your team is hired to design and build this user onboarding system.
It must be:

Feature | Description
Dead-letter Queue | Retry or log failed Kafka messages (e.g., broken DB or email)
Schema Registry | Validate event structure between services
Trace ID Middleware | Add correlation IDs for tracing requests across microservices
API Gateway (e.g., Kong) | Route and secure external access

```yaml
version: v1
services:
  onboarder-service:
    description: |
      Entry point for user registration. Validates input, checks Redis cache,
      and publishes to Kafka if user does not exist.
    endpoints:
      - POST /register
    dependencies:
      - redis
      - kafka (topic: user.registration)

  user-store-service:
    description: |
      Listens to registration events, validates and persists data to Postgres.
      Publishes confirmation and cache update events.
    kafka:
      consume:
        - topic: user.registration
      produce:
        - topic: user.created
        - topic: user.confirmation
    dependencies:
      - postgres
      - kafka

  notifier-service:
    description: |
      Sends confirmation emails to users after registration is successful.
    kafka:
      consume:
        - topic: user.confirmation
    dependencies:
      - smtp (or 3rd-party email provider)

  cache-updater:
    description: |
      Updates Redis cache with the latest user data upon successful creation.
    kafka:
      consume:
        - topic: user.created
    dependencies:
      - redis

infrastructure:
  kafka:
    topics:
      - user.registration
      - user.created
      - user.confirmation
  redis:
    ttl_minutes: 15
  postgres:
    schema: users
  smtp:
    provider: mailgun
```

Fast — can handle high traffic smoothly

Reliable — no data loss, no crashes

Easy to use — both for new users and internal staff

Secure — protects personal information and login access

Scalable — ready to support millions of users as the business grows

👩‍🏫 Why This Matters:
This registration system is the front door of the business.
If it fails, FinSync loses users, revenue, and trust.

Your job is to make sure that doesn’t happen.

Quick facts:
Located on top of Jade Mountain (overlooking the Valley of Peace)

Home of:

Master Oogway 🐢

Master Shifu 🐼

The Furious Five 🐯🐵🐍🐶🐞

Holds the Dragon Scroll 🐉

Sacred dojo of Kung Fu learning and wisdom
