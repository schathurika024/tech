# OrderFlow - Secure Order Management & Notification Service

A Spring Boot-based REST API for managing customer orders with JWT authentication, MySQL database, and RabbitMQ for asynchronous notification processing.

## Features

- **JWT-based Authentication**: Secure API access with role-based authorization (ADMIN, USER)
- **Order Management**: Create, update, and query orders with customer information
- **RabbitMQ Integration**: Asynchronous event processing for order notifications
- **MySQL Database**: Persistent storage for users, customers, orders, and notifications
- **Docker Support**: Complete containerization with Docker Compose

## Technology Stack

- **Java**: 21
- **Spring Boot**: 3.5.8
- **Database**: MySQL 8.0
- **Message Broker**: RabbitMQ 3-management
- **Security**: Spring Security with JWT
- **Build Tool**: Maven

## Prerequisites

- Java 21 or higher
- Maven 3.9+
- Docker and Docker Compose (for containerized deployment)
- MySQL 8.0 (if running locally without Docker)

## Project Structure

```
OrderManagement/
├── src/
│   ├── main/
│   │   ├── java/com/example/OrderManagement/
│   │   │   ├── config/          # Configuration classes (RabbitMQ, etc.)
│   │   │   ├── consumer/        # RabbitMQ consumers
│   │   │   ├── controller/      # REST controllers
│   │   │   ├── dto/             # Data Transfer Objects
│   │   │   ├── exception/       # Exception handlers
│   │   │   ├── model/           # Entity models
│   │   │   ├── repository/      # JPA repositories
│   │   │   ├── security/        # Security configuration and JWT
│   │   │   └── service/         # Business logic services
│   │   └── resources/
│   │       └── application.properties
│   └── test/                     # Unit and integration tests
├── Dockerfile
├── docker-compose.yml
└── pom.xml
```

## Getting Started

### Option 1: Using Docker Compose (Recommended)

1. **Clone the repository** (if applicable) or navigate to the project directory

2. **Build and start all services**:
   ```bash
   docker-compose up --build
   ```

   This will start:
   - MySQL database on port 3306
   - RabbitMQ on ports 5672 (AMQP) and 15672 (Management UI)
   - Spring Boot application on port 8080

3. **Access the services**:
   - API: http://localhost:8080
   - RabbitMQ Management UI: http://localhost:15672 (guest/guest)
   - MySQL: localhost:3306

### Option 2: Local Development

1. **Start MySQL and RabbitMQ**:
   - Ensure MySQL is running on `localhost:3306`
   - Ensure RabbitMQ is running on `localhost:5672`

2. **Update application.properties** if needed:
   ```properties
   spring.datasource.url=jdbc:mysql://localhost:3306/ordermanagement
   spring.datasource.username=root
   spring.datasource.password=yourpassword
   ```

3. **Build the project**:
   ```bash
   mvn clean install
   ```

4. **Run the application**:
   ```bash
   mvn spring-boot:run
   ```

## API Endpoints

### Authentication

#### Register User
```bash
POST /api/auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123",
  "fullName": "John Doe",
  "role": "USER"  // Optional, defaults to USER
}
```

#### Login
```bash
POST /api/auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "password123"
}

Response:
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 86400
}
```

### User Management

#### Get Current User
```bash
GET /api/users/me
Authorization: Bearer <token>
```

### Order Management

#### Create Order
```bash
POST /api/orders
Authorization: Bearer <token>
Content-Type: application/json

{
  "customerId": 1,  // Optional: use existing customer
  "customerData": {  // Required if customerId not provided
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "1234567890"
  },
  "items": [
    {
      "productName": "Product 1",
      "quantity": 2,
      "unitPrice": 29.99
    },
    {
      "productName": "Product 2",
      "quantity": 1,
      "unitPrice": 49.99
    }
  ]
}
```

#### Update Order Status
```bash
PATCH /api/orders/{id}/status
Authorization: Bearer <token>
Content-Type: application/json

{
  "status": "PROCESSING"  // NEW, PROCESSING, COMPLETED, CANCELLED
}
```

#### Get Order by ID
```bash
GET /api/orders/{id}
Authorization: Bearer <token>
```

#### Get My Orders
```bash
GET /api/orders/my?page=0&size=20&status=NEW
Authorization: Bearer <token>
```

### Admin Endpoints

#### List All Users
```bash
GET /api/admin/users?page=0&size=20
Authorization: Bearer <token>  // Requires ADMIN role
```

#### List All Orders
```bash
GET /api/admin/orders?page=0&size=20
Authorization: Bearer <token>  // Requires ADMIN role
```

#### List All Notifications
```bash
GET /api/admin/notifications?page=0&size=20
Authorization: Bearer <token>  // Requires ADMIN role
```

## Example API Usage

### Complete Flow Example

1. **Register a new user**:
   ```bash
   curl -X POST http://localhost:8080/api/auth/register \
     -H "Content-Type: application/json" \
     -d '{
       "email": "john@example.com",
       "password": "password123",
       "fullName": "John Doe"
     }'
   ```

2. **Login**:
   ```bash
   curl -X POST http://localhost:8080/api/auth/login \
     -H "Content-Type: application/json" \
     -d '{
       "email": "john@example.com",
       "password": "password123"
     }'
   ```
   
   Save the `accessToken` from the response.

3. **Create an order**:
   ```bash
   curl -X POST http://localhost:8080/api/orders \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer <your-token>" \
     -d '{
       "customerData": {
         "name": "Jane Customer",
         "email": "jane@example.com",
         "phone": "1234567890"
       },
       "items": [
         {
           "productName": "Laptop",
           "quantity": 1,
           "unitPrice": 999.99
         }
       ]
     }'
   ```

4. **Check notifications** (as ADMIN):
   ```bash
   curl -X GET http://localhost:8080/api/admin/notifications \
     -H "Authorization: Bearer <admin-token>"
   ```

## User Roles

- **USER**: Can create orders and view only their own orders
- **ADMIN**: Can manage users, view all orders, and access notification logs

## RabbitMQ Configuration

The application uses RabbitMQ with:
- **Exchange**: `order.events` (Direct)
- **Queue**: `order.notifications`
- **Routing Keys**:
  - `order.created` - When an order is created
  - `order.status.changed` - When order status is updated

### Event Messages

**ORDER_CREATED Event**:
```json
{
  "eventType": "ORDER_CREATED",
  "orderId": 123,
  "customerEmail": "customer@example.com",
  "totalAmount": 150.0,
  "createdAt": "2025-01-01T10:00:00Z"
}
```

**ORDER_STATUS_CHANGED Event**:
```json
{
  "eventType": "ORDER_STATUS_CHANGED",
  "orderId": 123,
  "oldStatus": "NEW",
  "newStatus": "PROCESSING",
  "updatedAt": "2025-01-01T11:00:00Z"
}
```

## Environment Variables

The application supports the following environment variables:

- `DB_HOST`: Database host (default: localhost)
- `DB_PORT`: Database port (default: 3306)
- `DB_NAME`: Database name (default: ordermanagement)
- `DB_USER`: Database username (default: root)
- `DB_PASSWORD`: Database password
- `RABBITMQ_HOST`: RabbitMQ host (default: localhost)
- `RABBITMQ_PORT`: RabbitMQ port (default: 5672)
- `RABBITMQ_USER`: RabbitMQ username (default: guest)
- `RABBITMQ_PASSWORD`: RabbitMQ password (default: guest)
- `JWT_SECRET`: JWT secret key (must be at least 256 bits)

## Testing

Run tests with:
```bash
mvn test
```

The project includes:
- Unit tests for service layer (OrderService)
- Integration tests for authentication endpoints

## Logging

The application logs:
- Authentication failures
- Order lifecycle events
- RabbitMQ publish/consume operations

Logs are output to console with timestamp format: `yyyy-MM-dd HH:mm:ss - message`

## Health Check

The application exposes a health endpoint:
```bash
GET /actuator/health
```

## Troubleshooting

### Application won't start
- Ensure MySQL and RabbitMQ are running
- Check database connection settings in `application.properties`
- Verify ports 8080, 3306, and 5672 are available

### Docker Compose issues
- Ensure Docker and Docker Compose are installed
- Check if ports are already in use: `docker ps`
- View logs: `docker-compose logs app`

### Authentication issues
- Verify JWT token is included in Authorization header
- Check token expiration (default: 24 hours)
- Ensure user has correct role for admin endpoints

## License

This project is created for interview assignment purposes.

## Author

Created as part of OrderFlow assignment submission.

