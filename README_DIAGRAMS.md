# Hotel Booking Platform – Architecture & Flow Diagrams

These diagrams document the backend implementation in the `main` branch.

## 1. Backend architecture

```mermaid
flowchart LR
    Client[Client / Partner] --> Controller[REST Controllers]
    Controller --> Service[Application Services]
    Service --> Repo[Repositories]
    Repo --> DB[(PostgreSQL)]
    Service --> SePay[SePay / VietQR]
```

## 2. Booking and inventory reservation flow

```mermaid
flowchart TD
    A["Customer submits booking request"] --> B["Check idempotency key<br/>(userId + key)"]
    B --> C{"Existing booking<br/>for this key?"}

    C -->|Yes| D["Return existing booking<br/>(no second reservation)"]
    C -->|No| E["Validate user, dates, rooms, guests<br/>and primary contact"]
    E --> F["Start a fresh transaction<br/>(createBookingOnce via Spring proxy)"]
    F --> G["Load DailyInventory rows<br/>for every date in the stay"]
    G --> H{"Stay dates open and<br/>inventory range complete?"}

    H -->|No| I["Reject booking<br/>Return conflict / validation error"]
    H -->|Yes| J{"Enough rooms available?<br/>availableRooms - blockedRooms >= quantity"}

    J -->|Yes| K["Increase blockedRooms<br/>for each stay date<br/>DailyInventory uses @Version"]
    K --> L["Persist Booking + Contact + Items<br/>Calculate total price<br/>Set PENDING_PAYMENT / expiry"]
    L --> M["Flush / commit transaction<br/>Database checks @Version"]
    M --> N{"Optimistic-lock<br/>conflict?"}

    N -->|No| O["Commit succeeds"]
    O --> P["Return booking response<br/>Continue to payment"]

    N -->|Yes| Q["Rollback failed attempt<br/>Wait briefly; retry in a fresh transaction<br/>Maximum 3 attempts"]
    Q --> R{"Attempts exhausted?"}
    R -->|No| F
    R -->|Yes| S["Return CONFLICT<br/>Ask customer to try again"]

    J -->|No| I
```

## 3. VietQR / SePay payment flow

```mermaid
flowchart TD
    A[Customer opens payment for booking] --> B[Load owned booking<br/>Check / apply booking expiration]
    B --> C{"Booking still<br/>PENDING_PAYMENT?"}

    C -->|No| D[Reject payment session<br/>Booking is not payable]
    D --> Z[Return latest payment / booking status]

    C -->|Yes| E[Find valid pending payment<br/>for this booking]
    E --> F{"Reusable payment<br/>transaction exists?"}

    F -->|No| G[Create PENDING transaction<br/>Generate unique paymentCode<br/>Amount + expiry + bank details]
    G --> H[Build VietQR URL / payment session<br/>Return QR + paymentCode to client]

    F -->|Yes| H
    H --> I[Customer transfers money<br/>using banking app / QR]
    I --> J[SePay receives bank transaction<br/>and sends webhook]
    J --> K[Webhook endpoint<br/>Public to JWT; service verifies<br/>Authorization: Apikey ...]
    K --> L{"Incoming transfer<br/>and unseen gateway ID?"}

    L -->|No| M[Ignore duplicate / unrelated<br/>webhook safely]
    M --> Z

    L -->|Yes| N[Resolve paymentCode<br/>Find matching PaymentTransaction]
    N --> O{"Payment code found<br/>and amount matches?"}

    O -->|No| P[Do not auto-confirm<br/>Record mismatch / expired state<br/>Requires review or correction]
    P --> Z

    O -->|Yes| Q{"Booking still<br/>PENDING_PAYMENT?"}

    Q -->|Yes| R[Set transaction SUCCESS<br/>Set booking CONFIRMED<br/>Persist gateway transaction ID]
    R --> Z

    Q -->|No| P
```

## Implementation notes

- This is shown as a layered Spring Boot backend, not a microservices architecture.
- Booking creation checks idempotency before inventory reservation.
- Inventory reservation uses `DailyInventory.@Version`; optimistic-lock failures cause the current attempt to roll back and the wrapper retries in a fresh transaction, up to three attempts.
- SePay webhook requests do not authenticate as an end-user JWT request; the payment service separately validates the configured SePay API key.
- Payment confirmation checks incoming transfer type, gateway transaction identity, payment code, amount, and whether the booking is still payable.
- The customer-triggered reconciliation path queries SePay's Transaction API and reuses the payment confirmation checks.
- Scheduled expiration releases inventory for overdue unpaid bookings. Each booking expiration is isolated in a `REQUIRES_NEW` transaction.

## Source files used for the diagrams

- `backend/src/main/java/com/hotel/hotel_backend/security/SecurityConfig.java`
- `backend/src/main/java/com/hotel/hotel_backend/security/JwtAuthFilter.java`
- `backend/src/main/java/com/hotel/hotel_backend/security/JwtService.java`
- `backend/src/main/java/com/hotel/hotel_backend/service/impl/BookingServiceImpl.java`
- `backend/src/main/java/com/hotel/hotel_backend/service/impl/InventoryServiceImpl.java`
- `backend/src/main/java/com/hotel/hotel_backend/entity/DailyInventory.java`
- `backend/src/main/java/com/hotel/hotel_backend/service/BookingPaymentGatewayService.java`
- `backend/src/main/java/com/hotel/hotel_backend/controller/PaymentWebhookController.java`
- `backend/src/main/java/com/hotel/hotel_backend/service/BookingExpirationService.java`
