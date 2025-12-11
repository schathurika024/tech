# OrderFlow - Complete Architecture Documentation

This document provides a comprehensive view of the OrderFlow Order Management System architecture, including multiple diagrams and detailed explanations.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Application Architecture (Layered)](#2-application-architecture-layered)
3. [Component Diagram](#3-component-diagram)
4. [Database Schema](#4-database-schema)
5. [Security Architecture](#5-security-architecture)
6. [Message Flow (RabbitMQ)](#6-message-flow-rabbitmq)
7. [Deployment Architecture](#7-deployment-architecture)
8. [Request Flow Diagrams](#8-request-flow-diagrams)
9. [Technology Stack](#9-technology-stack)

---

## 1. System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Web Client  │  │ Mobile App   │  │  Postman/    │         │
│  │  (Browser)    │  │               │  │  curl        │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          │         HTTPS/REST API              │
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
          ┌──────────────────▼──────────────────┐
          │     SPRING BOOT APPLICATION          │
          │     (Port 8080)                      │
          │  ┌──────────────────────────────┐   │
          │  │  REST Controllers            │   │
          │  │  - AuthController            │   │
          │  │  - OrderController           │   │
          │  │  - UserController             │   │
          │  │  - AdminController           │   │
          │  └──────────────┬───────────────┘   │
          │                 │                    │
          │  ┌──────────────▼───────────────┐   │
          │  │  Service Layer               │   │
          │  │  - AuthService                │   │
          │  │  - OrderService               │   │
          │  │  - UserService                │   │
          │  │  - AdminService               │   │
          │  └──────────────┬───────────────┘   │
          │                 │                    │
          │  ┌──────────────▼───────────────┐   │
          │  │  Repository Layer             │   │
          │  │  - UserRepository             │   │
          │  │  - OrderRepository            │   │
          │  │  - CustomerRepository         │   │
          │  └──────────────┬───────────────┘   │
          └──────────────────┼───────────────────┘
                             │
          ┌──────────────────┼───────────────────┐
          │                  │                   │
    ┌─────▼─────┐    ┌───────▼────────┐         │
    │   MySQL    │    │    RabbitMQ     │         │
    │  Database  │    │  Message Broker │         │
    │  (Port     │    │  (Port 5672)    │         │
    │   3306)    │    │  Management UI  │         │
    │            │    │  (Port 15672)   │         │
    └────────────┘    └─────────────────┘         │
                                                     │
          ┌──────────────────────────────────────────┘
          │
    ┌─────▼─────────────┐
    │  Notification     │
    │  Consumer         │
    │  (Async Handler)  │
    └───────────────────┘
```

**Key Components:**
- **Client Layer**: External consumers (web, mobile, API clients)
- **Application Layer**: Spring Boot REST API
- **Data Layer**: MySQL for persistent storage
- **Message Layer**: RabbitMQ for asynchronous event processing
- **Consumer**: Background processor for notifications

---

## 2. Application Architecture (Layered)

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Controllers                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │   │
│  │  │AuthController│  │OrderController│ │UserController│ │   │
│  │  └──────┬───────┘  └──────┬───────┘ └──────┬──────┘ │   │
│  │         │                 │                 │        │   │
│  │  ┌──────▼─────────────────▼─────────────────▼──────┐ │   │
│  │  │         GlobalExceptionHandler                    │ │   │
│  │  │         (@ControllerAdvice)                       │ │   │
│  │  └──────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                              │
                              │ DTOs (Data Transfer Objects)
                              │
┌────────────────────────────▼────────────────────────────────┐
│                    BUSINESS LOGIC LAYER                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Services                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │   │
│  │  │ AuthService │  │OrderService  │  │UserService  │ │   │
│  │  │             │  │              │  │             │ │   │
│  │  │ - register()│  │ - createOrder│  │ - getCurrent│ │   │
│  │  │ - login()   │  │ - updateStatus│  │   User()    │ │   │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘ │   │
│  │         │                │                 │        │   │
│  │  ┌──────▼────────────────▼─────────────────▼──────┐ │   │
│  │  │         Security Layer                          │ │   │
│  │  │  - JwtService                                   │ │   │
│  │  │  - JwtAuthenticationFilter                      │ │   │
│  │  │  - SecurityConfig                               │ │   │
│  │  └──────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                              │
                              │ Entities
                              │
┌────────────────────────────▼────────────────────────────────┐
│                    DATA ACCESS LAYER                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Repositories (Spring Data JPA)                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐ │   │
│  │  │UserRepo     │  │OrderRepo     │  │CustomerRepo │ │   │
│  │  └──────┬──────┘  └──────┬───────┘  └──────┬──────┘ │   │
│  └─────────┼────────────────┼──────────────────┼────────┘   │
└────────────┼────────────────┼──────────────────┼─────────────┘
              │                │                  │
              │                │                  │
┌─────────────▼────────────────▼──────────────────▼───────────┐
│                    DATABASE LAYER                             │
│                    MySQL 8.0                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  users   │  │  orders  │  │ customers│  │order_items│   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Layer Responsibilities:**

1. **Presentation Layer**: 
   - Handles HTTP requests/responses
   - Request validation
   - Error handling
   - DTO conversion

2. **Business Logic Layer**:
   - Core business rules
   - Transaction management
   - Security (JWT, authentication)
   - Event publishing

3. **Data Access Layer**:
   - Database operations
   - Query optimization
   - Entity management

4. **Database Layer**:
   - Persistent storage
   - ACID transactions
   - Data integrity

---

## 3. Component Diagram

```
┌──────────────────────────────────────────────────────────────┐
│              OrderManagementApplication                      │
│              (@SpringBootApplication)                        │
└───────────────────────┬──────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
│   Security   │ │  Controllers│ │   Config    │
│   Package    │ │  Package    │ │   Package   │
│              │ │             │ │             │
│ ┌──────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │
│ │JwtService│ │ │ │AuthCtrl│ │ │ │RabbitMQ│ │
│ │JwtFilter │ │ │ │OrderCtrl│ │ │ │ Config │ │
│ │Security  │ │ │ │UserCtrl │ │ │ └────────┘ │
│ │Config    │ │ │ │AdminCtrl│ │ │             │
│ └──────────┘ │ │ └────┬───┘ │ │             │
└──────────────┘ └──────┼──────┘ └─────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
┌───────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
│   Service    │ │  Repository │ │  Consumer  │
│   Package    │ │  Package    │ │  Package   │
│              │ │             │ │             │
│ ┌──────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │
│ │AuthService│ │ │ │UserRepo│ │ │ │Order   │ │
│ │OrderService│ │ │ │OrderRepo│ │ │ │Notification│ │
│ │UserService│ │ │ │Customer│ │ │ │Consumer│ │
│ │AdminService│ │ │ │Repo    │ │ │ └────────┘ │
│ └─────┬─────┘ │ │ └────┬───┘ │ │             │
└───────┼───────┘ └───────┼──────┘ └─────────────┘
        │                 │
        │                 │
┌───────▼─────────────────▼───────┐
│         Model Package             │
│  ┌────────┐  ┌────────┐  ┌──────┐│
│  │  User  │  │ Order  │  │Customer││
│  │Customer│  │OrderItem│ │Notify││
│  └────────┘  └────────┘  └──────┘│
└──────────────────────────────────┘
```

**Key Components:**

- **Security**: JWT authentication and authorization
- **Controllers**: REST endpoints
- **Services**: Business logic
- **Repositories**: Data access
- **Models**: Domain entities
- **Config**: Infrastructure configuration
- **Consumer**: Async message processing

---

## 4. Database Schema

```
┌─────────────────────────────────────────────────────────────┐
│                      DATABASE: ordermanagement               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────┐
│       users         │
├─────────────────────┤
│ id (PK)             │◄─────┐
│ email (UNIQUE)      │      │
│ password            │      │
│ full_name           │      │
│ role (ENUM)         │      │
│ created_at          │      │
└─────────────────────┘      │
                              │
┌─────────────────────┐       │
│     customers       │       │
├─────────────────────┤       │
│ id (PK)             │       │
│ name                │       │
│ email               │       │
│ phone               │       │
│ created_at          │       │
└──────────┬──────────┘       │
           │                   │
           │                   │
┌──────────▼──────────┐       │
│       orders         │       │
├──────────────────────┤       │
│ id (PK)              │       │
│ customer_id (FK)─────┼───────┼──┐
│ created_by_user_id   │       │  │
│        (FK)──────────┼───────┘  │
│ status (ENUM)        │           │
│ total_amount         │           │
│ created_at           │           │
│ updated_at           │           │
└──────────┬───────────┘           │
           │                       │
           │ 1:N                   │
┌──────────▼───────────┐           │
│    order_items        │           │
├───────────────────────┤           │
│ id (PK)               │           │
│ order_id (FK)─────────┼───────────┘
│ product_name          │
│ quantity              │
│ unit_price            │
└───────────────────────┘

┌─────────────────────┐
│ notification_logs    │
├─────────────────────┤
│ id (PK)              │
│ event_type           │
│ order_id (FK)───────┼──┐
│ payload (TEXT)       │  │
│ status (ENUM)        │  │
│ created_at           │  │
└─────────────────────┘  │
                          │
                          │ (Optional reference)
                          │
                          └───┐
                              │
                              │
                    ┌─────────▼─────────┐
                    │  orders (id)       │
                    └───────────────────┘

Relationships:
- User 1:N Orders (created_by_user_id)
- Customer 1:N Orders (customer_id)
- Order 1:N OrderItems (order_id)
- Order 1:N NotificationLogs (order_id, nullable)
```

**Entity Relationships:**

1. **User → Order**: One user can create many orders
2. **Customer → Order**: One customer can have many orders
3. **Order → OrderItem**: One order contains many items
4. **Order → NotificationLog**: One order can have many notification logs

---

## 5. Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY FLOW                            │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐
│   Client     │
│  (Browser)   │
└──────┬───────┘
       │
       │ 1. POST /api/auth/register
       │    {email, password, fullName}
       │
       ▼
┌─────────────────────┐
│  AuthController     │
│  /api/auth/register │
└──────┬──────────────┘
       │
       │ 2. AuthService.register()
       │
       ▼
┌─────────────────────┐
│  PasswordEncoder     │
│  (BCrypt)            │
│  - hash password     │
└──────┬──────────────┘
       │
       │ 3. Save to DB
       │
       ▼
┌─────────────────────┐
│   MySQL (users)     │
└─────────────────────┘

┌──────────────┐
│   Client     │
└──────┬───────┘
       │
       │ 1. POST /api/auth/login
       │    {email, password}
       │
       ▼
┌─────────────────────┐
│  AuthController     │
│  /api/auth/login    │
└──────┬──────────────┘
       │
       │ 2. AuthenticationManager
       │    .authenticate()
       │
       ▼
┌─────────────────────┐
│  CustomUserDetails  │
│  Service            │
│  - loadUserByUsername│
└──────┬──────────────┘
       │
       │ 3. Load from DB
       │
       ▼
┌─────────────────────┐
│   MySQL (users)     │
└──────┬──────────────┘
       │
       │ 4. Verify password
       │
       ▼
┌─────────────────────┐
│  JwtService          │
│  - generateToken()   │
└──────┬──────────────┘
       │
       │ 5. Return JWT
       │
       ▼
┌──────────────┐
│   Client     │
│  (Stores JWT)│
└──────────────┘

┌─────────────────────────────────────────────────────────────┐
│              PROTECTED REQUEST FLOW                          │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐
│   Client     │
│  (With JWT)  │
└──────┬───────┘
       │
       │ 1. GET /api/orders
       │    Header: Authorization: Bearer <token>
       │
       ▼
┌─────────────────────┐
│  SecurityFilterChain│
│  - CSRF disabled    │
│  - Stateless        │
└──────┬──────────────┘
       │
       │ 2. JwtAuthenticationFilter
       │    - Extract token
       │    - Validate token
       │
       ▼
┌─────────────────────┐
│  JwtService          │
│  - extractUsername() │
│  - validateToken()   │
└──────┬──────────────┘
       │
       │ 3. Load user
       │
       ▼
┌─────────────────────┐
│  CustomUserDetails   │
│  Service             │
└──────┬──────────────┘
       │
       │ 4. Set Authentication
       │    in SecurityContext
       │
       ▼
┌─────────────────────┐
│  OrderController    │
│  - getMyOrders()    │
└──────┬──────────────┘
       │
       │ 5. Check @PreAuthorize
       │    (if admin endpoint)
       │
       ▼
┌─────────────────────┐
│  OrderService        │
│  - getMyOrders()     │
└─────────────────────┘
```

**Security Components:**

1. **Authentication**: 
   - Registration with password hashing (BCrypt)
   - Login with JWT token generation

2. **Authorization**:
   - Role-based access (USER, ADMIN)
   - Method-level security (@PreAuthorize)

3. **JWT Flow**:
   - Token generation on login
   - Token validation on each request
   - Stateless authentication

---

## 6. Message Flow (RabbitMQ)

```
┌─────────────────────────────────────────────────────────────┐
│              RABBITMQ MESSAGE FLOW                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────┐
│  OrderService       │
│  - createOrder()    │
└──────┬──────────────┘
       │
       │ 1. Save order to DB
       │
       ▼
┌─────────────────────┐
│   MySQL (orders)    │
└──────┬──────────────┘
       │
       │ 2. Publish event
       │
       ▼
┌─────────────────────┐
│  RabbitTemplate     │
│  .convertAndSend()  │
└──────┬──────────────┘
       │
       │ 3. Send to Exchange
       │
       ▼
┌─────────────────────────────────────────┐
│      RabbitMQ Exchange                  │
│      "order.events" (Direct)            │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │ Routing Key: "order.created"     │  │
│  │ Message: OrderCreatedEvent       │  │
│  │ {                                 │  │
│  │   eventType: "ORDER_CREATED",    │  │
│  │   orderId: 123,                  │  │
│  │   customerEmail: "...",          │  │
│  │   totalAmount: 999.99,           │  │
│  │   createdAt: "2025-01-01..."    │  │
│  │ }                                 │  │
│  └──────────────────────────────────┘  │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │ Routing Key: "order.status.changed"│ │
│  │ Message: OrderStatusChangedEvent │  │
│  │ {                                 │  │
│  │   eventType: "ORDER_STATUS_CHANGED",│ │
│  │   orderId: 123,                   │  │
│  │   oldStatus: "NEW",               │  │
│  │   newStatus: "PROCESSING",        │  │
│  │   updatedAt: "2025-01-01..."     │  │
│  │ }                                 │  │
│  └──────────────────────────────────┘  │
└─────────────┬───────────────────────────┘
              │
              │ 4. Route to Queue
              │
              ▼
┌─────────────────────────────────────────┐
│      RabbitMQ Queue                     │
│      "order.notifications"              │
│      (Durable)                          │
└─────────────┬───────────────────────────┘
              │
              │ 5. Consumer receives
              │
              ▼
┌─────────────────────────────────────────┐
│  OrderNotificationConsumer              │
│  @RabbitListener                        │
│  - handleOrderNotification()            │
└──────┬──────────────────────────────────┘
       │
       │ 6. Process message
       │    - Log notification
       │    - Save to DB
       │
       ▼
┌─────────────────────────────────────────┐
│   MySQL (notification_logs)             │
└─────────────────────────────────────────┘

Exchange Configuration:
- Type: Direct Exchange
- Name: "order.events"
- Bindings:
  * Queue: "order.notifications"
  * Routing Key: "order.created"
  * Routing Key: "order.status.changed"
```

**Message Flow Steps:**

1. **Order Creation/Update**: Service publishes event
2. **Exchange Routing**: Direct exchange routes by routing key
3. **Queue Storage**: Message stored in durable queue
4. **Consumer Processing**: Async consumer processes message
5. **Notification Log**: Event logged in database

---

## 7. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DOCKER COMPOSE                            │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                    Docker Network                             │
│                    "order-network"                            │
│                    (Bridge)                                  │
└──────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐   ┌────────▼────────┐   ┌───────▼────────┐
│   MySQL 8.0    │   │   RabbitMQ      │   │  Spring Boot  │
│                │   │   3-management  │   │   Application  │
│ Container:     │   │                │   │                │
│ ordermanagement│   │ Container:      │   │ Container:     │
│ -db            │   │ ordermanagement │   │ ordermanagement│
│                │   │ -rabbitmq       │   │ -app           │
│ Port: 3306     │   │                 │   │                │
│                │   │ Ports:          │   │ Port: 8080     │
│ Volume:        │   │ - 5672 (AMQP)   │   │                │
│ mysql_data     │   │ - 15672 (UI)    │   │ Health Check:   │
│                │   │                 │   │ /actuator/health│
│ Health Check:  │   │ Volume:         │   │                │
│ mysqladmin ping│   │ rabbitmq_data   │   │ Depends on:    │
│                │   │                 │   │ - db (healthy) │
│                │   │ Health Check:   │   │ - rabbitmq     │
│                │   │ rabbitmq-diagnostics│ │   (healthy)   │
└────────────────┘   └─────────────────┘   └────────────────┘

Environment Variables:
┌──────────────────────────────────────────────────────────────┐
│  Application Container:                                      │
│  - DB_HOST=db                                                │
│  - DB_PORT=3306                                              │
│  - DB_NAME=ordermanagement                                   │
│  - DB_USER=root                                              │
│  - DB_PASSWORD=rootpassword                                 │
│  - RABBITMQ_HOST=rabbitmq                                   │
│  - RABBITMQ_PORT=5672                                       │
│  - RABBITMQ_USER=guest                                      │
│  - RABBITMQ_PASSWORD=guest                                 │
│  - JWT_SECRET=...                                            │
└──────────────────────────────────────────────────────────────┘

Dockerfile (Multi-stage):
┌──────────────────────────────────────────────────────────────┐
│  Stage 1: Build                                              │
│  - Base: maven:3.9-eclipse-temurin-21                       │
│  - Copy pom.xml, download dependencies                      │
│  - Copy src, build JAR                                       │
│                                                              │
│  Stage 2: Run                                                │
│  - Base: eclipse-temurin:21-jre-alpine                      │
│  - Copy JAR from Stage 1                                     │
│  - Expose port 8080                                          │
│  - Health check endpoint                                     │
│  - Run: java -jar app.jar                                    │
└──────────────────────────────────────────────────────────────┘
```

**Deployment Features:**

- **Containerization**: All services in Docker containers
- **Networking**: Isolated Docker network
- **Volumes**: Persistent data storage
- **Health Checks**: Ensures services are ready
- **Dependencies**: App waits for DB and RabbitMQ
- **Multi-stage Build**: Optimized Docker image size

---

## 8. Request Flow Diagrams

### 8.1 Order Creation Flow

```
Client                    Controller              Service              Repository          RabbitMQ          Consumer          Database
  │                          │                      │                      │                    │                  │                  │
  │ POST /api/orders         │                      │                      │                    │                  │                  │
  │ + JWT Token              │                      │                      │                    │                  │                  │
  │ + Order Data             │                      │                      │                    │                  │                  │
  │─────────────────────────>│                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │ Validate JWT         │                      │                    │                  │                  │
  │                          │ (SecurityFilter)     │                      │                    │                  │                  │
  │                          │─────────────────────>│                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Validate Request     │                    │                  │                  │
  │                          │                      │ (@Valid)              │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Get/Create Customer  │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │ SELECT/INSERT     │                  │                  │
  │                          │                      │                      │───────────────────>│                  │                  │
  │                          │                      │                      │<───────────────────│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Create Order         │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │ INSERT orders      │                  │                  │
  │                          │                      │                      │───────────────────>│                  │                  │
  │                          │                      │                      │<───────────────────│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Create OrderItems    │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │ INSERT order_items │                  │                  │
  │                          │                      │                      │───────────────────>│                  │                  │
  │                          │                      │                      │<───────────────────│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Publish Event        │                    │                  │                  │
  │                          │                      │──────────────────────────────────────────>│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │ Route to Queue    │                  │
  │                          │                      │                      │                    │──────────────────>│                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │ Consume Message │
  │                          │                      │                      │                    │                  │─────────────────>│
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │ Save Log        │
  │                          │                      │                      │                    │                  │─────────────────>│
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Return OrderResponse │                    │                  │                  │
  │                          │                      │<──────────────────────│                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │ Return Response      │                      │                    │                  │                  │
  │                          │<─────────────────────│                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │ 201 Created              │                      │                      │                    │                  │                  │
  │ + Order Data             │                      │                      │                    │                  │                  │
  │<─────────────────────────│                      │                      │                    │                  │                  │
```

### 8.2 Authentication Flow

```
Client                    AuthController          AuthService          UserRepository      JwtService          Database
  │                          │                      │                      │                    │                  │
  │ POST /api/auth/login     │                      │                      │                    │                  │
  │ {email, password}        │                      │                      │                    │                  │
  │─────────────────────────>│                      │                      │                    │                  │
  │                          │                      │                      │                    │                  │
  │                          │ login()              │                      │                    │                  │
  │                          │─────────────────────>│                      │                    │                  │
  │                          │                      │                      │                    │                  │
  │                          │                      │ Authenticate          │                    │                  │
  │                          │                      │ (AuthenticationManager)│                    │                  │
  │                          │                      │──────────────────────>│                    │                  │
  │                          │                      │                      │ Load User          │                  │
  │                          │                      │                      │───────────────────>│                  │
  │                          │                      │                      │<───────────────────│                  │
  │                          │                      │                      │                    │                  │
  │                          │                      │ Verify Password       │                    │                  │
  │                          │                      │ (BCrypt)              │                    │                  │
  │                          │                      │                      │                    │                  │
  │                          │                      │ Generate JWT          │                    │                  │
  │                          │                      │──────────────────────────────────────────>│                  │
  │                          │                      │                      │                    │                  │
  │                          │                      │                      │                    │ Return Token     │
  │                          │                      │                      │                    │<─────────────────│
  │                          │                      │                      │                    │                  │
  │                          │                      │ Return LoginResponse  │                    │                  │
  │                          │                      │<──────────────────────│                    │                  │
  │                          │                      │                      │                    │                  │
  │                          │ Return Response      │                      │                    │                  │
  │                          │<─────────────────────│                      │                    │                  │
  │                          │                      │                      │                    │                  │
  │ 200 OK                   │                      │                      │                    │                  │
  │ {accessToken, expiresIn} │                      │                      │                    │                  │
  │<─────────────────────────│                      │                      │                    │                  │
```

### 8.3 Order Status Update Flow

```
Client                    OrderController         OrderService         OrderRepository    RabbitMQ          Consumer          Database
  │                          │                      │                      │                    │                  │                  │
  │ PATCH /api/orders/1/status│                      │                      │                    │                  │                  │
  │ + JWT Token              │                      │                      │                    │                  │                  │
  │ + {status: "PROCESSING"} │                      │                      │                    │                  │                  │
  │─────────────────────────>│                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │ Validate JWT         │                      │                    │                  │                  │
  │                          │ Check Authorization  │                      │                    │                  │                  │
  │                          │─────────────────────>│                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Load Order           │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │ SELECT * FROM orders│                  │                  │
  │                          │                      │                      │───────────────────>│                  │                  │
  │                          │                      │                      │<───────────────────│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Check Permission     │                    │                  │                  │
  │                          │                      │ (Creator or Admin)    │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Update Status         │                    │                  │                  │
  │                          │                      │──────────────────────>│                    │                  │                  │
  │                          │                      │                      │ UPDATE orders       │                  │                  │
  │                          │                      │                      │───────────────────>│                  │                  │
  │                          │                      │                      │<───────────────────│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Publish Event        │                    │                  │                  │
  │                          │                      │──────────────────────────────────────────>│                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │ Route to Queue    │                  │
  │                          │                      │                      │                    │──────────────────>│                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │ Consume Message │
  │                          │                      │                      │                    │                  │─────────────────>│
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │ Save Log        │
  │                          │                      │                      │                    │                  │─────────────────>│
  │                          │                      │                      │                    │                  │                  │
  │                          │                      │ Return OrderResponse │                    │                  │                  │
  │                          │                      │<──────────────────────│                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │                          │ Return Response      │                      │                    │                  │                  │
  │                          │<─────────────────────│                      │                    │                  │                  │
  │                          │                      │                      │                    │                  │                  │
  │ 200 OK                   │                      │                      │                    │                  │                  │
  │ + Updated Order          │                      │                      │                    │                  │                  │
  │<─────────────────────────│                      │                      │                    │                  │                  │
```

---

## 9. Technology Stack

```
┌─────────────────────────────────────────────────────────────┐
│                    TECHNOLOGY STACK                        │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Language & Framework                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Java 21                                               │  │
│  │ Spring Boot 3.5.8                                     │  │
│  │ Spring Framework 6.x                                  │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Spring Modules                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Spring Web (REST API)                                │  │
│  │ Spring Security (JWT Authentication)                 │  │
│  │ Spring Data JPA (Database Access)                    │  │
│  │ Spring AMQP (RabbitMQ Integration)                    │  │
│  │ Spring Boot Actuator (Health Checks)                  │  │
│  │ Spring Validation (Bean Validation)                 │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Database                                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ MySQL 8.0                                            │  │
│  │ Hibernate (JPA Implementation)                       │  │
│  │ MySQL Connector/J                                    │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Message Broker                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ RabbitMQ 3-management                                │  │
│  │ Spring AMQP                                           │  │
│  │ Jackson2JsonMessageConverter                         │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Security                                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ JWT (JSON Web Tokens)                                │  │
│  │ JJWT Library (0.12.3)                                │  │
│  │ BCrypt Password Encoding                              │  │
│  │ Spring Security                                       │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Build & Deployment                                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Maven 3.9+                                           │  │
│  │ Docker                                               │  │
│  │ Docker Compose                                       │  │
│  │ Multi-stage Dockerfile                               │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Testing                                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ JUnit 5                                               │  │
│  │ Mockito                                               │  │
│  │ Spring Boot Test                                      │  │
│  │ Spring Security Test                                  │  │
│  │ H2 Database (Test)                                    │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 10. Key Design Decisions

### 10.1 Why Layered Architecture?
- **Separation of Concerns**: Each layer has a single responsibility
- **Testability**: Easy to mock dependencies
- **Maintainability**: Changes isolated to specific layers
- **Scalability**: Can scale layers independently

### 10.2 Why DTOs?
- **Security**: Don't expose internal entity structure
- **Versioning**: Can change DTOs without changing entities
- **Performance**: Select only needed fields
- **Validation**: Separate validation rules for API

### 10.3 Why JWT?
- **Stateless**: No server-side session storage
- **Scalable**: Works across multiple servers
- **Mobile-friendly**: Works well with mobile apps
- **Microservices**: Easy to use across services

### 10.4 Why RabbitMQ?
- **Asynchronous**: Don't block order creation
- **Reliability**: Message persistence and delivery guarantees
- **Decoupling**: Order service doesn't need to know about notifications
- **Scalability**: Can add more consumers easily

### 10.5 Why MySQL?
- **ACID Transactions**: Critical for order data
- **Relationships**: Complex queries with joins
- **Maturity**: Well-established, reliable
- **Spring Integration**: Excellent JPA support

---

## 11. API Endpoints Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    API ENDPOINTS                            │
└─────────────────────────────────────────────────────────────┘

Authentication (Public):
  POST   /api/auth/register
  POST   /api/auth/login

User Management:
  GET    /api/users/me                    [Authenticated]

Order Management:
  POST   /api/orders                      [Authenticated]
  GET    /api/orders/{id}                 [Authenticated]
  GET    /api/orders/my                   [Authenticated]
  PATCH  /api/orders/{id}/status          [Authenticated]

Admin Endpoints:
  GET    /api/admin/users                 [ADMIN]
  GET    /api/admin/orders                [ADMIN]
  GET    /api/admin/notifications         [ADMIN]

Health Check:
  GET    /actuator/health                 [Public]
```

---

## 12. Data Flow Summary

```
Input → Controller → Service → Repository → Database
                                    ↓
                              RabbitMQ → Consumer → Database
```

**Synchronous Flow**: Request → Response (immediate)
**Asynchronous Flow**: Event → Queue → Consumer (background)

---

## Conclusion

This architecture provides:
- ✅ **Scalability**: Stateless design, horizontal scaling
- ✅ **Security**: JWT authentication, role-based access
- ✅ **Reliability**: Database transactions, message persistence
- ✅ **Maintainability**: Clean layers, separation of concerns
- ✅ **Testability**: Mockable dependencies, isolated components
- ✅ **Deployability**: Docker containerization, easy setup

---

**Document Version**: 1.0  
**Last Updated**: 2025-01-01  
**Project**: OrderFlow Order Management System

