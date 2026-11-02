# Hotel Hub

Hotel Hub is a hotel management and booking platform designed for hotel operators, guests, and administrators. The system manages room inventory, reservations, payments, user identity, and operational reporting in a single backend service that can support a web or mobile frontend.

Main URL: https://www.hotelhub.online

---

## Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Authentication Flow](#authentication-flow)
- [Booking Concurrency Handling](#booking-concurrency-handling)
- [Payment Flow](#payment-flow)
- [Database Design](#database-design)
- [Installation & Configuration](#installation--configuration)
- [API Documentation & Testing](#api-documentation--testing)

---

## Project Overview

Hotel Hub provides a backend platform for hotel booking, room management, user authentication, payment processing, and operational analytics. The application is built to support both online booking flows and back-office administration.

The system is designed around the following goals:

- Support real-time room availability and reservation handling
- Enable secure user authentication and access control
- Streamline booking lifecycle from search to payment to confirmation
- Improve operational visibility for hotel staff and administrators
- Support future expansion to mobile apps and third-party integrations

---

## Tech Stack

The project is designed as a modern backend application using the following core technologies:

- Node.js
- Express.js / NestJS (depending on project implementation)
- TypeScript
- PostgreSQL
- Redis
- JWT for authentication
- Swagger / OpenAPI for API documentation
- Docker / Docker Compose for local environment setup
- PM2 or similar process manager for deployment
- CI/CD support via GitHub Actions or external automation tools

This stack allows the project to balance performance, reliability, extensibility, and maintainability.

---

## Key Features

- Hotel and room catalog management
- Room search and filtering by date, room type, price, and facilities
- Reservation creation and booking confirmation
- Booking status tracking: pending, confirmed, checked in, completed, canceled
- User authentication and authorization for guests and administrators
- Payment processing for bookings and invoice settlement
- Notification flow for confirmation, cancellation, and reminders
- Reporting and operational analytics for occupancy and revenue
- Role-based access control for admin/staff/customer personas
- Audit-friendly transaction history for critical booking events

---

## System Architecture

Hotel Hub follows a modular backend architecture that separates concerns across business domains and infrastructure services.

Typical architecture:

```text
Client App / Web / Mobile
           |
           v
   API Layer (Node.js / Express)
           |
   +-------+-----------------------+
   |                               |
   v                               v
Authentication Service         Booking & Reservation Service
   |                               |
   v                               v
   User DB / Session Store        PostgreSQL / Redis
                               |
                               v
                        Payment Service
                               |
                               v
                        External Payment Gateway
```

Key design principles:

- Separate authentication, booking, and payment concerns
- Centralize business validation in backend services
- Use a transactional database for core booking records
- Use Redis for caching, rate limiting, and temporary state
- Keep APIs REST-friendly and versionable for frontend integration

---

## Authentication Flow

The authentication flow ensures secure access for customers and administrators.

1. User registers with email and password or social login provider.
2. Backend validates input and creates a user profile.
3. Passwords are stored in a hashed format.
4. User logs in and receives a JWT access token.
5. Protected endpoints validate the token before processing requests.
6. Refresh tokens or session policies may be used for extended sessions.
7. Authorization checks ensure role-based access for admin-only operations.

Typical flow:

```text
User -> POST /auth/login
      -> Backend validates credentials
      -> Generate JWT
      -> Return access token + refresh token (if enabled)
      -> Client includes token in Authorization header
      -> Backend verifies token on each protected request
```

Security practices:

- Use strong JWT secret management
- Store secrets in environment variables
- Apply middleware for authenticated routes
- Validate permission scopes by role and resource ownership

---

## Booking Concurrency Handling

Booking concurrency is one of the most important requirements for a hotel reservation platform. To prevent double-booking the same room on overlapping dates, the system must handle concurrent requests safely.

Recommended approach:

- Use database transactions for booking creation
- Validate room availability within a transaction
- Lock the room record or relevant inventory row during reservation check
- Re-check capacity and date range before final confirmation
- Treat booking creation as a critical section for inventory validation

Example pattern:

```text
1. Customer requests booking for room X on date range D1-D2
2. Backend begins transaction
3. Check room availability and occupancy records
4. If available, reserve the inventory and create booking record
5. Commit transaction
6. If transaction fails, return conflict or retry response
```

This approach reduces race conditions and helps maintain consistency under simultaneous bookings.

---

## Payment Flow

The payment flow supports booking confirmation and secure settlement.

1. Customer selects room and submits booking details.
2. Backend validates room availability and pricing.
3. Payment request is created for the reservation amount.
4. Payment gateway receives the transaction request.
5. Gateway returns success or failure result.
6. Backend updates booking status to confirmed only after successful payment.
7. Payment event or webhook is processed to update ledger and notification state.

Typical flow:

```text
Customer -> Create booking
         -> Backend calculates total price
         -> Redirect or call payment gateway
         -> Gateway confirms successful payment
         -> Backend updates booking status
         -> Send confirmation email / notification
```

Important considerations:

- Idempotency for payment retries
- Secure handling of payment metadata and webhook validation
- Clear handling of failed, pending, and refunded transactions
- Reconciliation between booking records and payment logs

---

## Database Design

The system uses a relational database as the source of truth for core transactional data.

Core entities may include:

- Users
- Roles
- Hotels
- Rooms
- Room Types
- Room Availability
- Bookings
- Booking Details
- Payments
- Transactions
- Reviews
- Notifications
- Audit Logs

Recommended relationships:

- One user can have many bookings
- One hotel can contain many rooms
- One room belongs to one room type and one hotel
- One booking can contain multiple room allocations or line items
- One booking can have one or many payment records
- One user can have many reviews and notifications

Example logical schema:

```text
users (id, email, password_hash, role, created_at)
roles (id, name)

hotels (id, name, address, description, status)
rooms (id, hotel_id, room_type_id, room_number, price_per_night, status)
room_types (id, name, capacity, amenities)
booking_status (id, name)
bookings (id, user_id, hotel_id, room_id, check_in, check_out, total_price, status)
payments (id, booking_id, amount, method, status, transaction_reference, created_at)
notifications (id, user_id, type, message, is_read, created_at)
reviews (id, user_id, hotel_id, rating, comment, created_at)
```

The database should enforce integrity constraints and support indexing for date-based availability queries.

---

## Installation & Configuration

### Prerequisites

- Node.js 16+ or the version required by the project
- npm or yarn
- PostgreSQL
- Redis
- Git
- Optional: Docker and Docker Compose

### Clone Repository

```bash
git clone https://github.com/TuanAnh2483/vlu_hotel_hub.git
cd vlu_hotel_hub
```

### Install Dependencies

```bash
npm install
# or
yarn install
```

### Environment Variables

Create a `.env` file at the project root and configure values similar to the following:

```env
PORT=3000
NODE_ENV=development
DATABASE_URL=postgresql://user:password@localhost:5432/hotel_hub
REDIS_URL=redis://localhost:6379
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=7d
PAYMENT_GATEWAY_KEY=your_gateway_key
PAYMENT_GATEWAY_SECRET=your_gateway_secret
```

### Run the Application

Development mode:

```bash
npm run dev
# or
yarn dev
```

Production build:

```bash
npm run build
npm start
```

### Database Migration

If the project uses Prisma, TypeORM, or Sequelize, run migration commands as needed:

```bash
npm run migrate
# or
db migration commands depending on the project setup
```

---

## API Documentation & Testing

### API Documentation

The backend should expose API documentation using Swagger/OpenAPI or a dedicated docs portal.

Typical access points:

- `/api-docs`
- Swagger JSON output for code generation
- Postman or Insomnia collections for manual testing

### Testing

The project should include automated verification for all critical business flows.

Recommended tests:

- Unit tests for validation and utility logic
- Integration tests for booking and payment endpoints
- Authentication tests for JWT validation and role authorization
- Concurrency tests to verify no double-booking occurs under load

Example commands:

```bash
npm test
npm run test:coverage
```

### Test Coverage Goals

- Booking workflow validation
- Authentication edge cases
- Payment success/failure handling
- Room availability under concurrent requests
- API contract validation for request/response payloads

---

## Contact

If you need support or would like to report issues, please contact:

- Email: tuananh240803@gmail.com
- GitHub: https://github.com/TuanAnh2483

---
