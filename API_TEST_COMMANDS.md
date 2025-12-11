# API Test Commands

This document contains curl commands to test the OrderFlow API. Make sure the application is running on `http://localhost:8080`.

## Prerequisites

- Application running on `http://localhost:8080`
- `curl` installed
- `jq` (optional, for pretty JSON output)

---

## 1. Register a New User

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

## 2. Register an Admin User

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

## 3. Login

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

## 4. Get Current User Info

```bash
curl -X GET http://localhost:8080/api/users/me \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

Replace `YOUR_TOKEN_HERE` with the token from step 3.

---

## 5. Create an Order (New Customer)

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

## 6. Create an Order (Existing Customer)

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

## 7. Get Order by ID

```bash
curl -X GET http://localhost:8080/api/orders/1 \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

---

## 8. Get My Orders

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

## 9. Update Order Status

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

## 10. Admin: Get All Users

```bash
curl -X GET "http://localhost:8080/api/admin/users?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

## 11. Admin: Get All Orders

```bash
curl -X GET "http://localhost:8080/api/admin/orders?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

## 12. Admin: Get All Notifications

```bash
curl -X GET "http://localhost:8080/api/admin/notifications?page=0&size=20" \
  -H "Authorization: Bearer ADMIN_TOKEN_HERE"
```

**Requires ADMIN role**

---

## Complete Test Flow Example

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

## Windows PowerShell Commands

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

---

## Error Responses

### 401 Unauthorized
```json
{
  "status": 401,
  "error": "Authentication failed",
  "message": "Invalid email or password",
  "timestamp": "2025-01-01T10:00:00"
}
```

### 403 Forbidden
```json
{
  "status": 403,
  "error": "Access denied",
  "message": "You don't have permission to access this resource",
  "timestamp": "2025-01-01T10:00:00"
}
```

### 404 Not Found
```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Order not found",
  "timestamp": "2025-01-01T10:00:00"
}
```

### 400 Bad Request (Validation Error)
```json
{
  "status": 400,
  "error": "Validation failed",
  "message": "{email=Email should be valid, password=Password must be at least 6 characters}",
  "timestamp": "2025-01-01T10:00:00"
}
```

---

## Tips

1. **Save tokens**: Store the JWT token from login for subsequent requests
2. **Check RabbitMQ**: After creating/updating orders, check RabbitMQ management UI at `http://localhost:15672` (guest/guest) to see messages
3. **Check logs**: Application logs will show notification processing
4. **Use jq**: Install `jq` for pretty JSON output: `curl ... | jq '.'`
5. **Health check**: Test if app is running: `curl http://localhost:8080/actuator/health`

