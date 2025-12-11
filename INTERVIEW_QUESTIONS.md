# Interview Questions - OrderFlow Project

This document contains 50 potential interview questions covering various aspects of the OrderFlow Order Management System project.

---

## Table of Contents
1. [Architecture & Design (10 questions)](#architecture--design)
2. [Spring Boot & Framework (8 questions)](#spring-boot--framework)
3. [Security & JWT (8 questions)](#security--jwt)
4. [Database & JPA (7 questions)](#database--jpa)
5. [RabbitMQ & Messaging (7 questions)](#rabbitmq--messaging)
6. [Docker & DevOps (5 questions)](#docker--devops)
7. [Testing (5 questions)](#testing)
8. [Code Quality & Best Practices (5 questions)](#code-quality--best-practices)
9. [Performance & Scalability (5 questions)](#performance--scalability)

---

## Architecture & Design

### 1. Why did you choose a layered architecture (Controller-Service-Repository)? What are the benefits?
**Expected Answer:** Separation of concerns, maintainability, testability, reusability. Controllers handle HTTP, services contain business logic, repositories handle data access.

### 2. Why did you use DTOs instead of exposing entities directly?
**Expected Answer:** Security (hide internal structure), decoupling, versioning, validation, prevents lazy loading issues, allows data transformation.

### 3. Explain the difference between `@Service`, `@Repository`, and `@Component` annotations. When would you use each?
**Expected Answer:** 
- `@Service`: Business logic layer
- `@Repository`: Data access layer (adds exception translation)
- `@Component`: Generic Spring-managed bean
- Use appropriate annotation for semantic clarity and Spring features

### 4. Why did you choose MySQL over MongoDB for this project?
**Expected Answer:** Relational data (orders, customers, users), ACID transactions, complex queries, existing Spring Data JPA expertise, better for structured data with relationships.

### 5. How would you handle the case where RabbitMQ is down when creating an order?
**Expected Answer:** Use try-catch around RabbitMQ publish, log error but don't fail the transaction. Could implement retry mechanism, dead letter queue, or circuit breaker pattern.

### 6. Why did you make `orderId` nullable in `NotificationLog`?
**Expected Answer:** To handle error cases where message parsing fails and we can't extract orderId. Allows logging failed notifications for debugging.

### 7. How would you scale this application horizontally?
**Expected Answer:** Stateless design (JWT), load balancer, shared database, RabbitMQ cluster, session management via Redis if needed, container orchestration (Kubernetes).

### 8. What design patterns did you use in this project?
**Expected Answer:** 
- Repository Pattern (data access abstraction)
- DTO Pattern (data transfer objects)
- Strategy Pattern (JWT validation)
- Observer Pattern (RabbitMQ events)
- Factory Pattern (Spring bean creation)

### 9. How would you implement idempotency for order creation?
**Expected Answer:** Use idempotency key in request, check if order with key exists, return existing order if found. Store idempotency key in database.

### 10. Why did you separate `OrderItem` into a separate entity instead of embedding it?
**Expected Answer:** Normalization, easier queries, better for reporting, allows future features (tracking individual items), follows relational database best practices.

---

## Spring Boot & Framework

### 11. Explain the Spring Boot auto-configuration. How does it work?
**Expected Answer:** Spring Boot automatically configures beans based on classpath dependencies. Uses `@ConditionalOnClass`, `@ConditionalOnMissingBean` annotations. Can be overridden with custom configuration.

### 12. What is the difference between `@Autowired` and constructor injection? Which do you prefer?
**Expected Answer:** 
- `@Autowired`: Field/setter injection, can be null, harder to test
- Constructor injection: Required dependencies, immutable, easier testing, recommended by Spring
- Prefer constructor injection for required dependencies

### 13. How does Spring Security filter chain work?
**Expected Answer:** Request goes through filter chain. Each filter checks authentication/authorization. `JwtAuthenticationFilter` extracts token, validates, sets SecurityContext. `SecurityFilterChain` defines order and rules.

### 14. Why did you use `@Transactional` in the service layer?
**Expected Answer:** Ensures ACID properties, automatic rollback on exceptions, manages database connections. Service layer is appropriate because it contains business logic that may span multiple repository calls.

### 15. Explain the difference between `@RestController` and `@Controller`.
**Expected Answer:** 
- `@RestController` = `@Controller` + `@ResponseBody`
- `@RestController` returns JSON/XML directly
- `@Controller` returns view names (for MVC with templates)

### 16. How does Spring Data JPA work? What is the magic behind repository methods?
**Expected Answer:** Spring creates proxy implementations at runtime. Parses method names (findBy, countBy) and generates queries. Uses `@Query` for custom queries. Provides CRUD operations automatically.

### 17. What is the purpose of `@EnableRabbit` annotation?
**Expected Answer:** Enables RabbitMQ listener endpoints. Without it, `@RabbitListener` methods won't be processed. Registers necessary infrastructure beans for message consumption.

### 18. How would you handle configuration properties in Spring Boot?
**Expected Answer:** Use `@ConfigurationProperties` with `application.properties` or `application.yml`. Externalize via environment variables. Use `@Value` for simple properties. Prefer type-safe `@ConfigurationProperties`.

---

## Security & JWT

### 19. Explain how JWT authentication works in your implementation.
**Expected Answer:** 
1. User logs in with credentials
2. Server validates and generates JWT token
3. Client stores token and sends in Authorization header
4. `JwtAuthenticationFilter` extracts token, validates signature/expiration
5. Sets authentication in SecurityContext
6. Request proceeds with authenticated user

### 20. Why is JWT better than session-based authentication for this API?
**Expected Answer:** Stateless (scalable), works across domains, no server-side storage, mobile-friendly, microservices-friendly. Trade-off: harder to revoke tokens.

### 21. How would you implement token refresh mechanism?
**Expected Answer:** Issue short-lived access token and long-lived refresh token. Store refresh token in database. Endpoint validates refresh token and issues new access token. Can revoke refresh tokens.

### 22. What security vulnerabilities should we be aware of with JWT?
**Expected Answer:** 
- Token theft (XSS, man-in-the-middle)
- Token expiration handling
- Secret key management
- Algorithm confusion attacks
- Token size (can be large)
- No built-in revocation

### 23. How does role-based access control (RBAC) work in your implementation?
**Expected Answer:** User has role (USER/ADMIN). `@PreAuthorize("hasRole('ADMIN')")` checks role. Spring Security converts role to `ROLE_ADMIN` authority. Filter chain checks authorities before allowing access.

### 24. Why did you use BCrypt for password hashing?
**Expected Answer:** One-way hashing, salt included, adaptive (cost factor), industry standard, Spring Security default. Prevents rainbow table attacks, makes brute force expensive.

### 25. How would you prevent brute force attacks on login?
**Expected Answer:** Rate limiting (per IP), account lockout after failed attempts, CAPTCHA after N failures, exponential backoff, monitoring and alerting.

### 26. What is the difference between authentication and authorization?
**Expected Answer:** 
- Authentication: Who are you? (login, verify identity)
- Authorization: What can you do? (permissions, roles, access control)
- In this project: JWT handles authentication, roles handle authorization

---

## Database & JPA

### 27. Explain the relationship between Order and OrderItem. Why did you use `@OneToMany`?
**Expected Answer:** One Order has many OrderItems. `@OneToMany` with `cascade = CascadeType.ALL` ensures order items are saved/deleted with order. `orphanRemoval = true` removes items when removed from collection.

### 28. What is the N+1 problem? How did you avoid it in this project?
**Expected Answer:** N+1 occurs when fetching parent entities triggers separate queries for each child. Used `FetchType.LAZY` and proper DTO mapping. Could use `@EntityGraph` or `JOIN FETCH` for eager loading when needed.

### 29. Why did you use `@GeneratedValue(strategy = GenerationType.IDENTITY)`?
**Expected Answer:** Uses database auto-increment. Efficient, database-managed, works well with MySQL. Alternative strategies: SEQUENCE (PostgreSQL), TABLE (portable but slower).

### 30. How would you handle database migrations in production?
**Expected Answer:** Use Flyway or Liquibase. Version-controlled SQL scripts. Test migrations on staging. Rollback strategy. `spring.jpa.hibernate.ddl-auto=validate` in production (not `update`).

### 31. What is the difference between `LAZY` and `EAGER` fetching?
**Expected Answer:** 
- LAZY: Load on access (default for @OneToMany, @ManyToMany)
- EAGER: Load immediately (default for @OneToOne, @ManyToOne)
- LAZY prevents unnecessary data loading, but can cause LazyInitializationException outside transaction

### 32. How would you optimize a slow query that fetches orders with customer and user information?
**Expected Answer:** 
- Add database indexes on foreign keys
- Use `@EntityGraph` or `JOIN FETCH` in repository
- Pagination to limit results
- Projection DTOs to select only needed fields
- Query optimization (EXPLAIN ANALYZE)

### 33. Explain transaction isolation levels. Which one does Spring use by default?
**Expected Answer:** 
- READ_UNCOMMITTED: Dirty reads possible
- READ_COMMITTED: Prevents dirty reads (default in most databases)
- REPEATABLE_READ: Prevents non-repeatable reads
- SERIALIZABLE: Highest isolation, prevents phantom reads
- Spring uses database default (usually READ_COMMITTED)

---

## RabbitMQ & Messaging

### 34. Why did you choose RabbitMQ over other message brokers (Kafka, ActiveMQ)?
**Expected Answer:** 
- RabbitMQ: Good for request-response, routing, simpler setup, good for this use case
- Kafka: Better for high-throughput, event streaming, log aggregation
- For order notifications, RabbitMQ's routing and reliability features are sufficient

### 35. Explain the difference between Direct, Topic, and Fanout exchanges.
**Expected Answer:** 
- Direct: Routes by exact routing key match (used in this project)
- Topic: Routes by pattern matching (wildcards)
- Fanout: Broadcasts to all bound queues
- Chose Direct for simple, predictable routing

### 36. How would you ensure message delivery guarantees?
**Expected Answer:** 
- Publisher confirms (acknowledgment from broker)
- Persistent messages (durable queues, persistent delivery)
- Consumer acknowledgments (manual ack)
- Dead letter queues for failed messages
- Idempotent consumers

### 37. What happens if a consumer crashes while processing a message?
**Expected Answer:** If auto-ack is disabled and message not acknowledged, RabbitMQ redelivers. Could cause duplicate processing. Solution: Idempotent processing, check if already processed, use database transaction.

### 38. How would you handle message ordering requirements?
**Expected Answer:** 
- Single consumer per queue (not scalable)
- Partitioning by order ID (same order goes to same queue)
- Sequence numbers in messages
- Kafka is better for strict ordering

### 39. Explain the difference between `@RabbitListener` and `RabbitTemplate`.
**Expected Answer:** 
- `@RabbitListener`: Consumes messages (receiver), declarative
- `RabbitTemplate`: Sends messages (publisher), programmatic
- Used both: Template to publish, Listener to consume

### 40. How would you implement retry logic for failed message processing?
**Expected Answer:** 
- Spring Retry with `@Retryable`
- Dead letter queue after max retries
- Exponential backoff
- Store failed messages in database for manual review
- Circuit breaker pattern

---

## Docker & DevOps

### 41. Why did you use multi-stage Docker build?
**Expected Answer:** Smaller final image (only runtime dependencies), faster builds (cached layers), security (no build tools in production image), separation of concerns.

### 42. How does Docker Compose handle service dependencies?
**Expected Answer:** `depends_on` ensures order, but doesn't wait for readiness. Used `condition: service_healthy` with healthchecks to wait for MySQL and RabbitMQ to be ready before starting app.

### 43. How would you deploy this application to production?
**Expected Answer:** 
- CI/CD pipeline (GitHub Actions, Jenkins)
- Build Docker image, push to registry
- Deploy to Kubernetes/ECS/Docker Swarm
- Use secrets management for credentials
- Blue-green or rolling deployment
- Health checks and monitoring

### 44. What environment variables would you externalize in production?
**Expected Answer:** 
- Database credentials
- RabbitMQ credentials
- JWT secret (from secrets manager)
- Log levels
- Feature flags
- External service URLs

### 45. How would you monitor this application in production?
**Expected Answer:** 
- Spring Boot Actuator endpoints
- Prometheus metrics
- Distributed tracing (Zipkin/Jaeger)
- Log aggregation (ELK stack)
- APM tools (New Relic, Datadog)
- Health checks and alerts

---

## Testing

### 46. What types of tests did you write? Why?
**Expected Answer:** 
- Unit tests: Test service logic in isolation (OrderServiceTest)
- Integration tests: Test API endpoints with Spring context (AuthIntegrationTest)
- Used mocks for external dependencies (RabbitMQ, repositories)
- Integration tests verify end-to-end flow

### 47. How would you test the RabbitMQ consumer?
**Expected Answer:** 
- Use `@RabbitListenerTest` or embedded RabbitMQ
- Mock `NotificationLogRepository`
- Send test message to queue
- Verify consumer processes and saves log
- Test error scenarios

### 48. How would you test security endpoints?
**Expected Answer:** 
- Use `@WithMockUser` for authenticated requests
- Use `MockMvc` with `SecurityMockMvcRequestPostProcessors`
- Test unauthorized access returns 401/403
- Test role-based access control
- Verify JWT token generation and validation

### 49. What is the difference between unit tests and integration tests?
**Expected Answer:** 
- Unit tests: Test single component in isolation, fast, use mocks
- Integration tests: Test multiple components together, slower, use real dependencies (database, Spring context)
- Both are important: unit for logic, integration for flow

### 50. How would you test error scenarios (e.g., database down, RabbitMQ unavailable)?
**Expected Answer:** 
- Use test containers for database failures
- Mock RabbitTemplate to throw exceptions
- Verify error handling and logging
- Test graceful degradation
- Verify user gets appropriate error response
- Test retry mechanisms

---

## Code Quality & Best Practices

### 51. How did you ensure code quality in this project?
**Expected Answer:** 
- Separation of concerns (layered architecture)
- DTOs for data transfer
- Validation annotations
- Exception handling (@ControllerAdvice)
- Logging for debugging
- Meaningful variable names
- Comments where necessary

### 52. Why did you use `@Valid` annotation?
**Expected Answer:** Triggers Bean Validation (JSR-303). Validates request DTOs before reaching controller. Returns 400 with field errors if validation fails. Prevents invalid data from entering system.

### 53. How would you handle pagination in a large dataset?
**Expected Answer:** 
- Use Spring Data's `Pageable` interface
- Limit page size (e.g., max 100)
- Use database indexes
- Consider cursor-based pagination for very large datasets
- Return total count only if needed (expensive)

### 54. What is the purpose of `@ControllerAdvice`?
**Expected Answer:** Global exception handler. Catches exceptions from all controllers. Centralizes error handling. Returns consistent error responses. Better than handling in each controller.

### 55. How would you improve this codebase?
**Expected Answer:** 
- Add more comprehensive tests
- Implement caching (Redis) for frequently accessed data
- Add API versioning
- Implement rate limiting
- Add request/response logging middleware
- Implement distributed tracing
- Add API documentation (Swagger/OpenAPI)
- Implement audit logging

---

## Performance & Scalability

### 56. How would you optimize database queries?
**Expected Answer:** 
- Add indexes on foreign keys and frequently queried columns
- Use `@Query` with JOIN FETCH to avoid N+1
- Use pagination
- Use database connection pooling
- Query optimization (EXPLAIN)
- Consider read replicas for read-heavy workloads

### 57. How would you handle high traffic on order creation?
**Expected Answer:** 
- Horizontal scaling (multiple app instances)
- Database connection pooling
- Async processing (don't wait for RabbitMQ)
- Caching frequently accessed data
- Load balancer
- Database read replicas
- Consider event sourcing for audit trail

### 58. What caching strategies would you implement?
**Expected Answer:** 
- Cache user details (Redis)
- Cache customer information
- Cache frequently accessed orders
- Use `@Cacheable` annotations
- TTL based on data volatility
- Cache invalidation strategy
- Consider Caffeine for in-memory cache

### 59. How would you handle database connection pool exhaustion?
**Expected Answer:** 
- Increase pool size (but monitor)
- Use connection pool monitoring
- Fix connection leaks (ensure transactions complete)
- Use read replicas
- Implement circuit breaker
- Optimize slow queries
- Consider async processing

### 60. How would you implement rate limiting?
**Expected Answer:** 
- Use Spring Cloud Gateway or API Gateway
- Redis-based rate limiting (token bucket, sliding window)
- Per-user or per-IP limits
- Return 429 Too Many Requests
- Different limits for different endpoints
- Consider rate limiting libraries (Bucket4j)

---

## Bonus Questions

### 61. How would you implement audit logging for orders?
**Expected Answer:** 
- Use JPA Auditing (`@CreatedDate`, `@LastModifiedDate`)
- Create audit table
- Use Hibernate Envers
- Log all changes (who, what, when)
- Store before/after values

### 62. How would you implement search functionality for orders?
**Expected Answer:** 
- Use JPA Specifications for dynamic queries
- Full-text search with MySQL FULLTEXT indexes
- Consider Elasticsearch for advanced search
- Search by customer name, order ID, date range, status
- Paginated results

### 63. How would you handle concurrent order updates?
**Expected Answer:** 
- Optimistic locking (`@Version` annotation)
- Pessimistic locking (`LockModeType.PESSIMISTIC_WRITE`)
- Database constraints
- Handle `OptimisticLockException`
- Retry mechanism

### 64. How would you implement order cancellation with refunds?
**Expected Answer:** 
- Check order status (can't cancel COMPLETED)
- Update order status to CANCELLED
- Publish cancellation event
- Integrate with payment gateway for refund
- Update inventory if applicable
- Send notification to customer

### 65. How would you implement order status webhooks for external systems?
**Expected Answer:** 
- Store webhook URLs per customer/system
- Publish to RabbitMQ queue
- Separate consumer for webhooks
- Retry mechanism with exponential backoff
- Signature verification for security
- Store webhook delivery status

---

## Tips for Answering

1. **Be specific**: Reference actual code from the project
2. **Explain trade-offs**: Why you chose one approach over another
3. **Think about production**: Consider real-world scenarios
4. **Admit limitations**: It's okay to say "I would research X" or "I would consult with team"
5. **Show learning mindset**: Mention how you would improve or learn more

---

## Key Concepts to Review

- Spring Boot fundamentals
- Spring Security and JWT
- REST API design
- Database design and JPA
- Message queues (RabbitMQ)
- Docker and containerization
- Testing strategies
- Design patterns
- Security best practices
- Performance optimization

---

Good luck with your interview! 🚀

