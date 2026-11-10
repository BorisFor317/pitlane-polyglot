# Advanced Driving School App

## Users and roles

| Role        | Description                                                                              |
|-------------|------------------------------------------------------------------------------------------|
| Customer    | Register, browse lessons and free slots, book, cancel own booking, see own history       |
| Instructor  | See own schedule, mark lesson completed or no-show, add notes on the student (phase 2)   |
| Admin       | Manage disciplines, cars, lesson types, instructors, slots; see all bookings and reports |

## Functional requirements

IDs let you reference requirements from commits and tests.

### Accounts

- FR-A1: Register with email + password; passwords stored as a hash (bcrypt/argon2), never plain text
- FR-A2: Log in and receive a token (JWT or session cookie)
- FR-A3: Customer profile holds name, phone, driving license number
- FR-A4: Customer has a separate experience level (beginner / intermediate / advanced) per discipline; a customer with no level in a discipline counts as beginner

### Catalogue

- FR-C1: Lesson types belong to one discipline and have name, description, duration in minutes, price, minimum experience level, max students per slot
- FR-C2: Cars have make, model, year, drivetrain, horsepower, transmission, status (available / maintenance / retired)
- FR-C3: Instructors are users with a bio and the lesson types they are certified to teach
- FR-C4: Admin manages disciplines (drift, skid control, track day, rally, winter driving, ...) with name and description
- FR-C5: Cars are tagged with the disciplines they suit; one car can suit several
- FR-C6: A lesson type can require other lesson types as prerequisites, in chains of any depth

### Scheduling and booking

- FR-B1: Admin creates time slots: start time, lesson type, instructor
- FR-B2: Customers list free slots, filtered by date range, discipline, lesson type and car
- FR-B3: Customer books a slot; booking starts as pending, becomes confirmed once payment is recorded
- FR-B4: Customer can rent a car for track time without an instructor (car booking only)
- FR-B5: Customer cancels a booking; refund depends on how far ahead (see business rules)
- FR-B6: Instructor marks a booking completed or no_show after the slot ends
- FR-B7: Customer sees which prerequisites are still missing for a lesson type, and booking is rejected until all are completed
- FR-B8: After a completed lesson, the instructor can raise the customer's level in that discipline

Booking status flow: `pending` → `confirmed` → `completed` or `no_show`. From `pending` or `confirmed` a booking can go to `cancelled`. No other transitions are allowed.

### Payments (simulated)

- FR-P1: Record a payment against a booking (amount, method, timestamp, status); no real provider
- FR-P2: Record refunds as separate rows, never by editing the original payment

### Phase 2: Feedback

- FR-F1: Instructor can leave a note on a student after at least one completed lesson with that student
- FR-F2: Customer can leave one review per completed lesson, rating the car, the instructor and the lesson separately, plus notes on what to change. Car-only rentals have no instructor rating

### Phase 3: Venues

- FR-V1: Venues have name, location, type (circuit, skid pad, gravel stage, frozen lake) and opening hours
- FR-V2: Every slot happens at a venue; a venue has a max number of cars on it at once
- FR-V3: Cars can be based at a venue; booking a car at another venue is rejected
- FR-V4: A venue lists the disciplines it supports; a slot's discipline must be one of them
- FR-V5: Venues can have seasonal availability windows (e.g. a frozen lake only in winter); slots outside a window are rejected

### Phase 4: Async jobs (outbox, reminders, emails)

TBD

### Phase 5: Media

TBD

## Business rules and constraints

| ID       | Rule                                                                                        |
|----------|---------------------------------------------------------------------------------------------|
| BR-1     | A car cannot be in two slots that overlap in time                                           |
| BR-2     | An instructor cannot teach two overlapping slots                                            |
| BR-3     | A slot cannot have more active bookings than max_students                                   |
| BR-4     | A customer cannot book the same slot twice                                                  |
| BR-5     | Customer's experience level in the lesson's discipline must meet the lesson's minimum       |
| BR-6     | Cars in maintenance or retired cannot get new slots                                         |
| BR-7     | Instructor must be certified for the slot's lesson type                                     |
| BR-8     | Cancel ≥ 48 h before: 100% refund; 24–48 h: 50%; < 24 h: none. Canceled by school 100% refund.                             |
| BR-9     | Bookings are never deleted, only moved to cancelled                                         |
| BR-10    | Prices are copied onto the booking at booking time                                          |
| BR-11    | (Phase 2) Reviews only for completed bookings, one per booking                              |
| BR-12    | All prerequisites of a lesson type, transitively, must be completed before booking it       |
| BR-13    | A slot's car must suit the slot's discipline                                                |
| BR-14    | Prerequisite chains cannot form a cycle, and a lesson type cannot require itself            |
| BR-15    | (Phase 2) An instructor can note a student only after at least one completed lesson together |
| BR-16    | (Phase 3) A slot's venue must support its discipline and be open in that season             |

## Non-functional requirements

- NFR-1: Schema changes only through numbered migration files checked into git; no manual edits to the DB
- NFR-2: Write raw SQL for v1 (no ORM) so you see every query; an ORM or query builder can come later as a comparison
- NFR-3: Every booking write runs in a transaction; two concurrent bookings for the last seat must result in exactly one success
- NFR-4: Integration tests run against a real Postgres (Docker), seeded fresh per test run
- NFR-5: Indexes justified by EXPLAIN ANALYZE on a seeded dataset of ~100k bookings
- NFR-6: Config via environment variables; secrets never in the repo
- NFR-7: Structured request logging with a request ID
- NFR-8: docker compose up brings up DB + API from a clean checkout

## Stack

- PostgreSQL 16+
- Docker Compose
- plain .sql migrations (dbmate) in one folder
- black-box API tests (Hurl) that every implementation must pass

I will implement the same API in several languages, in this order.

| Order | Language             | Stack                                 | What it teaches                                                                         |
|-------|----------------------|---------------------------------------|-----------------------------------------------------------------------------------------|
| 1     | Go                   | chi or net/http + pgx (sqlc optional) | Explicit errors, context, connection pools; least friction, so the SQL stays the focus |
| 2     | Rust                 | axum + tokio + sqlx                   | Ownership in async code; sqlx checks queries against the real schema at compile time    |
| 3     | Node.js (TypeScript) | Fastify + node-postgres (pg) + zod    | Async I/O, the event loop, runtime validation                                           |
| 4     | C++                  | Drogon + libpqxx                      | How to suffer)                                                                          |
