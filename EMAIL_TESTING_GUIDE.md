# Email Testing Guide

This guide explains how to test email sending functionality in the Order Management System.

## Option 1: Using MailHog (Recommended for Local Testing)

MailHog is a local email testing tool that captures all emails sent by your application and provides a web UI to view them.

### Setup

1. **MailHog is already added to `docker-compose.yml`** - it will start automatically when you run:
   ```bash
   docker-compose up -d
   ```

2. **Access MailHog Web UI**:
   - Open your browser and go to: `http://localhost:8025`
   - This is where you'll see all emails sent by the application

3. **The application is already configured** to use MailHog:
   - SMTP Host: `mailhog` (container name)
   - SMTP Port: `1025`
   - No authentication required

### Testing Steps

1. **Start all services**:
   ```bash
   docker-compose up -d
   ```

2. **Create an order** using the API (this will trigger an email):
   ```bash
   curl -X POST http://localhost:8080/api/orders \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_TOKEN_HERE" \
     -d '{
       "customerData": {
         "name": "Test Customer",
         "email": "test@example.com"
       },
       "items": [
         {
           "productName": "Test Product",
           "quantity": 1,
           "unitPrice": 99.99
         }
       ]
     }'
   ```

3. **Check MailHog Web UI**:
   - Go to `http://localhost:8025`
   - You should see an email with:
     - **To**: `test@example.com`
     - **Subject**: "Your order {orderId} has been created"
     - **Body**: Order details

4. **Check application logs**:
   ```bash
   docker-compose logs -f app
   ```
   Look for messages like:
   - `"Attempting to send ORDER_CREATED email to test@example.com for order {orderId}"`
   - `"Email sent successfully to test@example.com"`

5. **Test status change email**:
   ```bash
   curl -X PATCH http://localhost:8080/api/orders/{orderId}/status \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer YOUR_TOKEN_HERE" \
     -d '{
       "status": "PROCESSING"
     }'
   ```
   Check MailHog again for the status change email.

---

## Option 2: Using Gmail SMTP (Real Email Testing)

If you want to send real emails to actual email addresses:

### Setup

1. **Enable 2-Factor Authentication** on your Gmail account

2. **Generate an App Password**:
   - Go to: https://myaccount.google.com/apppasswords
   - Select "Mail" and "Other (Custom name)"
   - Enter "Order Management" as the name
   - Copy the generated 16-character password

3. **Update `application.properties`** or set environment variables:
   ```properties
   spring.mail.host=smtp.gmail.com
   spring.mail.port=587
   spring.mail.username=your-email@gmail.com
   spring.mail.password=your-16-char-app-password
   spring.mail.properties.mail.smtp.auth=true
   spring.mail.properties.mail.smtp.starttls.enable=true
   ```

4. **Or set environment variables in `docker-compose.yml`**:
   ```yaml
   environment:
     MAIL_HOST: smtp.gmail.com
     MAIL_PORT: 587
     MAIL_USERNAME: your-email@gmail.com
     MAIL_PASSWORD: your-16-char-app-password
     MAIL_SMTP_AUTH: true
     MAIL_SMTP_STARTTLS: true
   ```

5. **Restart the application**:
   ```bash
   docker-compose restart app
   ```

### Testing

Follow the same testing steps as Option 1, but check your actual email inbox instead of MailHog.

---

## Option 3: Using Other SMTP Services

### SendGrid

1. Sign up at https://sendgrid.com
2. Create an API key
3. Configure:
   ```properties
   spring.mail.host=smtp.sendgrid.net
   spring.mail.port=587
   spring.mail.username=apikey
   spring.mail.password=your-sendgrid-api-key
   spring.mail.properties.mail.smtp.auth=true
   spring.mail.properties.mail.smtp.starttls.enable=true
   ```

### Mailgun

1. Sign up at https://www.mailgun.com
2. Get SMTP credentials from dashboard
3. Configure:
   ```properties
   spring.mail.host=smtp.mailgun.org
   spring.mail.port=587
   spring.mail.username=your-mailgun-username
   spring.mail.password=your-mailgun-password
   spring.mail.properties.mail.smtp.auth=true
   spring.mail.properties.mail.smtp.starttls.enable=true
   ```

---

## Verifying Email Sending

### 1. Check Application Logs

```bash
# View real-time logs
docker-compose logs -f app

# Look for these log messages:
# - "Attempting to send ORDER_CREATED email to..."
# - "Email sent successfully to..."
# - "Failed to send email (mail server may not be configured): ..."
```

### 2. Check Notification Logs Table

Query the database to see notification records:

```sql
SELECT * FROM notification_logs ORDER BY created_at DESC LIMIT 10;
```

- **Status should be `SUCCESS`** if email was processed (even if email sending failed, the event was processed)
- **Payload contains** the full event JSON
- **Event type** should be `ORDER_CREATED` or `ORDER_STATUS_CHANGED`

### 3. Check MailHog (if using)

- Open `http://localhost:8025`
- You should see emails listed with sender, recipient, subject, and body

### 4. Check Your Email Inbox (if using real SMTP)

- Check the recipient's inbox
- Check spam folder if not in inbox

---

## Troubleshooting

### Email not appearing in MailHog

1. **Check if MailHog is running**:
   ```bash
   docker-compose ps
   ```

2. **Check MailHog logs**:
   ```bash
   docker-compose logs mailhog
   ```

3. **Verify application can reach MailHog**:
   ```bash
   docker-compose exec app ping mailhog
   ```

4. **Check application logs for errors**:
   ```bash
   docker-compose logs app | grep -i mail
   ```

### Email sending fails silently

- Check application logs for warnings: `"Failed to send email (mail server may not be configured)"`
- Verify SMTP settings in `application.properties`
- Test SMTP connection manually using a tool like `telnet` or `swaks`

### Testing SMTP Connection Manually

```bash
# Test MailHog connection
docker-compose exec app sh -c "echo 'QUIT' | nc mailhog 1025"

# Test Gmail connection (from your local machine)
telnet smtp.gmail.com 587
```

---

## Quick Test Commands

### Test Order Creation (triggers ORDER_CREATED email)

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "customerData": {
      "name": "Email Test Customer",
      "email": "customer@example.com"
    },
    "items": [
      {
        "productName": "Test Product",
        "quantity": 2,
        "unitPrice": 49.99
      }
    ]
  }'
```

### Test Status Change (triggers ORDER_STATUS_CHANGED email)

```bash
curl -X PATCH http://localhost:8080/api/orders/1/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{
    "status": "PROCESSING"
  }'
```

---

## Notes

- **Email sending is optional**: If the mail server is unavailable, the application will log a warning but continue processing orders normally
- **Notification logs are always saved**: Even if email fails, the notification event is logged to the database
- **MailHog is for development only**: Don't use MailHog in production - use a real SMTP service

