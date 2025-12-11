# Order Management System – Quick Guide

This is a Spring Boot REST API for managing customer orders with JWT authentication, MySQL, RabbitMQ, and email notifications (via MailHog by default).

## Quick Start

- **Run the app**: from project root run  
  ```bash
  docker-compose up -d --build
  ```   
  - API: `http://localhost:8080`  
  - RabbitMQ UI: `http://localhost:15672` (guest/guest)  
  - MailHog UI (emails): `http://localhost:8025`  
  - Follow logs:  
    ```bash
    docker-compose logs -f app
    ```
- **Run without Docker (alternative)**:  
  ```bash
  mvn clean install
  mvn spring-boot:run
  ```
- **Basic test flow** (curl or Postman, base URL `http://localhost:8080`):
  1. `POST /api/auth/register` with email/password/fullName to create a user.  
  2. `POST /api/auth/login` with same credentials, copy `accessToken`.  
  3. Call secured endpoints with header `Authorization: Bearer <accessToken>` – e.g. `GET /api/users/me`.  
  4. Create an order via `POST /api/orders` (customerData + items), then `GET /api/orders/my` to see it.  
  5. If running with MailHog, open `http://localhost:8025` and watch `docker-compose logs -f app` while testing.

## How to Run the System

### Prerequisites

- Java 21 and Maven (for local runs)
- Docker and Docker Compose (recommended)

### Option 1: Docker Compose (recommended)

From the project root:

```bash
docker-compose up -d --build
```

This starts:
- MySQL (`db`) on `localhost:3306`
- RabbitMQ (`rabbitmq`) on `localhost:5672` (management UI at `http://localhost:15672`, guest/guest)
- MailHog (`mailhog`) on `http://localhost:8025` (SMTP on port `1025`)
- Spring Boot app (`app`) on `http://localhost:8080`

To view logs:

```bash
docker-compose logs -f app
```

### Option 2: Local run (without Docker)

1. Start MySQL and RabbitMQ locally (matching the settings in `src/main/resources/application.properties`).
2. Build and run:

```bash
mvn clean install
mvn spring-boot:run
```

## Docker Instructions

- **Start all services**:

  ```bash
  docker-compose up -d --build
  ```

- **Stop services**:

  ```bash
  docker-compose down
  ```

- **Rebuild app image after code changes**:

  ```bash
  docker-compose up -d --build app
  ```

Key URLs when running via Docker:
- API: `http://localhost:8080`
- RabbitMQ UI: `http://localhost:15672` (guest/guest)
- MailHog UI: `http://localhost:8025`

## Sample API Usage with Postman and MailHog (Email Testing)

1. **Start the stack**:
   - Run `docker-compose up -d --build` from the project root.
2. **Exercise the API**:
   - Use the curl commands in the **API Test Commands (curl)** section below, or the **Using Postman** section, to register a user, log in, and create/update orders.
3. **Verify emails in MailHog**:
   - Open `http://localhost:8025` in your browser to see captured ORDER_CREATED and status-change emails.

## API Test Commands (curl)

This section contains curl commands to test the OrderFlow API. Make sure the application is running on `http://localhost:8080`.

### Prerequisites

- Application running on `http://localhost:8080`
- `curl` installed
- `jq` (optional, for pretty JSON output)

---

### 1. Register a New User

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "password": "password123",
    "fullName": "John Doe"
  }'
```

**Response:** User details (without password)

---

### 2. Register an Admin User

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "admin123",
    "fullName": "Admin User",
    "role": "ADMIN"
  }'
```

---

### 3. Login

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "password": "password123"
  }'
```

**Response:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "tokenType": "Bearer",
  "expiresIn": 86400
}
```

**Save the `accessToken` for subsequent requests!**

---

### 4. Get Current User Info

```bash
curl -X GET http://localhost:8080/api/users/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

Replace `YOUR_TOKEN_HERE` with the token from step 3.

---

### 5. Create an Order (New Customer)

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "customerData": {
      "name": "Jane Customer",
      "email": "jane.customer@example.com",
      "phone": "1234567890"
    },
    "items": [
      {
        "productName": "Laptop",
        "quantity": 1,
        "unitPrice": 999.99
      },
      {
        "productName": "Mouse",
        "quantity": 2,
        "unitPrice": 29.99
      }
    ]
  }'
```

**Response:** Order details with calculated `totalAmount`

---

### 6. Create an Order (Existing Customer)

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "customerId": 1,
    "items": [
      {
        "productName": "Keyboard",
        "quantity": 1,
        "unitPrice": 79.99
      }
    ]
  }'
```

---

### 7. Get Order by ID

```bash
curl -X GET http://localhost:8080/api/orders/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

### 8. Get My Orders

```bash
curl -X GET "http://localhost:8080/api/orders/my?page=0&size=20" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**With filters:**

```bash
# Filter by status
curl -X GET "http://localhost:8080/api/orders/my?status=NEW&page=0&size=20" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"

# Filter by date range
curl -X GET "http://localhost:8080/api/orders/my?startDate=2025-01-01T00:00:00&endDate=2025-12-31T23:59:59&page=0&size=20" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

### 9. Update Order Status

```bash
curl -X PATCH http://localhost:8080/api/orders/1/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "status": "PROCESSING"
  }'
```

**Valid statuses:** `NEW`, `PROCESSING`, `COMPLETED`, `CANCELLED`

---

### 10. Admin: Get All Users

```bash
curl -X GET "http://localhost:8080/api/admin/users?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

### 11. Admin: Get All Orders

```bash
curl -X GET "http://localhost:8080/api/admin/orders?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

### 12. Admin: Get All Notifications

```bash
curl -X GET "http://localhost:8080/api/admin/notifications?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

### Complete Test Flow Example (Linux/macOS)

```bash
# Set variables
BASE_URL="http://localhost:8080"
EMAIL="test@example.com"
PASSWORD="password123"

# 1. Register
curl -X POST "$BASE_URL/api/auth/register" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"$PASSWORD\",\"fullName\":\"Test User\"}"

# 2. Login and extract token
TOKEN=$(curl -s -X POST "$BASE_URL/api/auth/login" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$EMAIL\",\"password\":\"$PASSWORD\"}" | \
  grep -o '"accessToken":"[^"]*' | cut -d'"' -f4)

echo "Token: $TOKEN"

# 3. Create order
curl -X POST "$BASE_URL/api/orders" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "customerData": {
      "name": "Customer Name",
      "email": "customer@example.com",
      "phone": "1234567890"
    },
    "items": [
      {
        "productName": "Product 1",
        "quantity": 2,
        "unitPrice": 50.00
      }
    ]
  }'

# 4. Get my orders
curl -X GET "$BASE_URL/api/orders/my" \
  -H "Authorization: Bearer $TOKEN"
```

---

### Windows PowerShell Commands

For Windows PowerShell users:

```powershell
# Register
$body = @{
    email = "john.doe@example.com"
    password = "password123"
    fullName = "John Doe"
} | ConvertTo-Json

Invoke-RestMethod -Uri "http://localhost:8080/api/auth/register" `
    -Method Post `
    -ContentType "application/json" `
    -Body $body

# Login
$loginBody = @{
    email = "john.doe@example.com"
    password = "password123"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "http://localhost:8080/api/auth/login" `
    -Method Post `
    -ContentType "application/json" `
    -Body $loginBody

$token = $response.accessToken
Write-Host "Token: $token"

# Create order
$orderBody = @{
    customerData = @{
        name = "Jane Customer"
        email = "jane@example.com"
        phone = "1234567890"
    }
    items = @(
        @{
            productName = "Laptop"
            quantity = 1
            unitPrice = 999.99
        }
    )
} | ConvertTo-Json -Depth 10

$headers = @{
    Authorization = "Bearer $token"
}

Invoke-RestMethod -Uri "http://localhost:8080/api/orders" `
    -Method Post `
    -ContentType "application/json" `
    -Headers $headers `
    -Body $orderBody
```

## Email Testing – Short Summary

- **Default setup (MailHog via Docker)**  
  - The app sends all emails to MailHog using SMTP host `mailhog` and port `1025` (configured via `docker-compose.yml`).  
  - Start everything with `docker-compose up -d --build` and open `http://localhost:8025` to see captured emails (no real emails are sent).

- **Switching to real email (e.g., Gmail/SMTP provider)**  
  - Update mail-related properties in `src/main/resources/application.properties` or use environment variables (`MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, etc.).  
  - Restart the app; from then on, order creation and status changes will send real emails instead of going to MailHog.

## Using Postman

- **Base URL**: `http://localhost:8080`
- **1. Register**
  - Method: `POST`
  - URL: `http://localhost:8080/api/auth/register`
  - Body (tab: “raw”, type: JSON):
    ```json
    {
      "email": "user@example.com",
      "password": "password123",
      "fullName": "Test User"
    }
    ```
- **2. Login**
  - Method: `POST`
  - URL: `http://localhost:8080/api/auth/login`
  - Body (JSON) with the same email/password as above.
  - Copy the `accessToken` from the response.
- **3. Call secured endpoints (e.g., create order)**
  - In Postman, go to the **Authorization** tab, choose **Bearer Token**, and paste the token.
  - Method: `POST`
  - URL: `http://localhost:8080/api/orders`
  - Body (JSON), e.g.:
    ```json
    {
      "customerData": {
        "name": "Email Test Customer",
        "email": "test@example.com",
        "phone": "1234567890"
      },
      "items": [
        {
          "productName": "Test Product",
          "quantity": 1,
          "unitPrice": 99.99
        }
      ]
    }
    ```


