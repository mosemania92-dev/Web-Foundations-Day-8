# TicketHub: Design Document

A website that sells tickets for concerts and events.

**Contents:** Context and goals → Requirements → Estimates → API → Data model → Preventing double-booking → Architecture → Trade-offs → Risks and open questions

---

## 1. Context and goals

TicketHub has 2 million registered users. On a normal day it is a quiet, read-heavy website. When a very popular concert goes on sale, it suddenly faces a flash crowd where far more people want tickets than there are seats.

**Goals**
- Let people browse events, pick seats, pay and see their tickets.
- Never sell the same seat twice.
- Stay up and fair during a big sale.

**Non-goals (for this design)**
- Resale or ticket transfers between users.
- Refund workflows.
- Mobile apps (the API would serve them, but they are not designed here).

---

## 2. Requirements

### Functional
- Browse and search upcoming events.
- View an event's seat map with the status of each seat (available, held, sold).
- Hold one or more seats temporarily (for example 5 minutes) while the user pays.
- Pay for held seats and receive a confirmed order.
- View "my tickets" after purchase.
- Register and log in.

### Non-functional

| Quality | Requirement |
|---|---|
| **Correctness** | A seat is never sold to two people. A user is never charged without receiving the ticket, and never receives a ticket without paying. This is the most important requirement. |
| **Speed** | Browsing pages load in under 500 ms. Holding a seat responds in under 300 ms, even during a big sale. |
| **Fairness** | During a big sale, people are served roughly in the order they arrived. Each account is limited to a small number of seats (for example 4) and bots are rate-limited, so one person cannot grab everything. |
| **Availability** | The site stays up during a big sale. If something must be degraded, browsing is degraded before purchasing, and purchasing is never allowed to become incorrect. |
| **Scalability** | Handles about 600 times the normal request rate for short bursts. |
| **Security** | Passwords are hashed. Card details never touch our servers (a payment provider handles them). All traffic uses HTTPS. |

---

## 3. Estimates

Assumption: a day has about 100,000 seconds (rounded from 86,400, as in the cheat sheet).

### Normal day
- Page views: 50,000 visitors × 10 pages = **500,000 page views/day** ÷ 100,000 s ≈ **5 views/sec**. With a 5× peak factor, about **25 views/sec**.
- Sales: **5,000 tickets/day** ÷ 100,000 s ≈ **0.05 tickets/sec**, so a purchase happens roughly once every 20 seconds.
- Read-to-write ratio: about 100 page views for every ticket sold, so it is **read-heavy**.

### Big sale (one popular concert)
- **200,000 people in 10 minutes (600 s)** ≈ **330 new users/sec** arriving.
- Each person makes about 10 requests (open the event, load the seat map a few times, try a few seats, pay): 200,000 × 10 = 2,000,000 requests ÷ 600 s ≈ **3,300 requests/sec average**, roughly **10,000/sec at the sharpest peak**.
- Seat holds: if each user tries about 5 seats, that is 1,000,000 hold attempts ÷ 600 s ≈ **1,700 write attempts/sec**, most hitting the same popular seats.
- Demand vs supply: 200,000 people want 20,000 seats, a **10 : 1** ratio. About 90% of people will not get a ticket, so failure must be handled politely.
- Sales rate: 20,000 seats ÷ 600 s ≈ **33 tickets/sec** if they sell out in 10 minutes.
- Storage: tiny. 20,000 seat rows per event and about 5,000 orders/day (about 1.8 million/year at roughly 1 KB each, about 2 GB/year). Storage is not a bottleneck.

### Comparison

| | Normal day | Big sale (10 min) | Factor |
|---|---|---|---|
| Requests/sec | about 5 (25 at peak) | about 3,300 (10,000 at peak) | **about 130× to 650×** |
| Tickets sold/sec | 0.05 | 33 | **about 650×** |
| Tickets sold in 10 min | about 35 | 20,000 | The sale sells in 10 minutes about 4× a normal day's total |

**Conclusion:** the average day is easy; a single server and database could handle it. The big sale is the problem, and the real danger is not just volume but **contention**: thousands of people fighting over the same few thousand rows at once. That is where double-booking could happen.

---

## 4. API

All endpoints are JSON over HTTPS. Endpoints that change data require a logged-in user (token in the `Authorization` header).

| # | Method and path | Purpose | Success | Errors |
|---|---|---|---|---|
| 1 | `GET /events?query=&city=&page=` | Browse and search events | `200 OK` with a list | `400` bad parameters |
| 2 | `GET /events/{eventId}` | Event details | `200 OK` | `404 Not Found` |
| 3 | `GET /events/{eventId}/seats` | Seat map with each seat's status and price | `200 OK` | `404` |
| 4 | `POST /events/{eventId}/holds` with `{ "seatIds": [101, 102] }` | Hold seats for 5 minutes | `201 Created` with `{ holdId, seatIds, expiresAt }` | `401` not logged in, `409 Conflict` if any seat is already taken, `422` too many seats, `429` too many requests |
| 5 | `DELETE /holds/{holdId}` | Release held seats early | `204 No Content` | `404`, `403` not your hold |
| 6 | `POST /orders` with `{ "holdId": "...", "paymentToken": "..." }` and an `Idempotency-Key` header | Pay and confirm the order | `201 Created` with `{ orderId, tickets }` | `401`, `402 Payment Required` payment failed, `409` hold no longer valid, `410 Gone` hold expired |
| 7 | `GET /me/tickets` | View my purchased tickets | `200 OK` | `401` |
| 8 | `POST /auth/register`, `POST /auth/login` | Create account and log in | `201` / `200` | `400`, `401`, `409` email already used |

Notes:
- `GET` endpoints 1 to 3 are safe and cacheable. Endpoints 4 to 6 change state and are never cached.
- The `Idempotency-Key` header on `POST /orders` means that if a user's browser retries (slow network, double click), we return the same order instead of charging twice.
- `409 Conflict` on a hold is a normal, expected answer in a big sale, not a bug. The front end shows "Sorry, that seat was just taken. Choose another."

---

## 5. Data model

Relational database (PostgreSQL), because we need transactions and constraints.

**users**
- `id` (PK), `email` (UNIQUE, NOT NULL), `password_hash` (NOT NULL), `name`, `created_at`

**events**
- `id` (PK), `name`, `venue`, `city`, `starts_at`, `on_sale_at`, `status`, `created_at`

**seats** (one row per physical seat *per event*)
- `id` (PK)
- `event_id` (FK → events.id)
- `section`, `row_label`, `seat_number`
- `price_cents`
- `status` (`available` / `held` / `sold`)
- `held_by` (FK → users.id, nullable)
- `hold_id` (nullable)
- `hold_expires_at` (nullable)
- **UNIQUE (`event_id`, `section`, `row_label`, `seat_number`)**, so the same physical seat cannot exist twice for an event.
- Index on (`event_id`, `status`) for fast seat-map queries.

**orders**
- `id` (PK), `user_id` (FK → users.id), `event_id` (FK → events.id), `total_cents`, `status` (`paid` / `failed`), `payment_ref`, `idempotency_key` (UNIQUE), `created_at`

**tickets** (one row per purchased seat)
- `id` (PK), `order_id` (FK → orders.id), `seat_id` (FK → seats.id), `event_id`, `user_id`, `qr_code`, `created_at`
- **UNIQUE (`seat_id`)**: the final safety net. A seat can only ever appear on one ticket.

### Relationships
- A user has many orders (1 : N).
- An event has many seats (1 : N) and many orders (1 : N).
- An order has many tickets (1 : N).
- A seat has at most one ticket (1 : 0..1), enforced by `UNIQUE (seat_id)`.

---

## 6. How double-booking is prevented

Everything in the system (CDN, caches, waiting room, many app servers) can show stale information or let two people reach checkout for the same seat. **Only the database can be the judge**, because it is the single source of truth and it processes competing writes one at a time. Three layers protect each seat:

### Layer 1: Atomic conditional update to hold a seat
The check and the write happen in **one SQL statement**, not "read, then decide, then write" (which would allow a race):

```sql
UPDATE seats
SET status = 'held',
    held_by = :user_id,
    hold_id = :hold_id,
    hold_expires_at = now() + interval '5 minutes'
WHERE id = :seat_id
  AND (status = 'available'
       OR (status = 'held' AND hold_expires_at < now()));
```

If two people click the same seat at the same moment, the database locks the row for the first statement. The first sees an available seat and succeeds (1 row affected). The second then sees the updated row, matches 0 rows, and gets a `409 Conflict`. There is no window where both succeed.

For several seats, everything is done in **one transaction**. If any seat returns 0 rows, the whole transaction is rolled back so the user gets all their seats or none (no half-held groups).

### Layer 2: Transaction when paying
`POST /orders` runs in one database transaction:

1. Confirm the hold still belongs to this user and has not expired:
   ```sql
   UPDATE seats
   SET status = 'sold'
   WHERE hold_id = :hold_id AND held_by = :user_id
     AND status = 'held' AND hold_expires_at > now();
   ```
   If the rows affected do not equal the number of seats in the hold, **roll back** and return `409` / `410`.
2. Insert the `orders` row and one `tickets` row per seat.
3. Commit.

The payment is authorized before the commit using a payment token. If the payment fails, we roll back and release the hold. If the commit fails after payment, we refund or void the authorization automatically. The idempotency key stops duplicate charges on retries.

### Layer 3: Unique constraint as a backstop
`UNIQUE (seat_id)` on `tickets` means that even if a bug in our code skipped the checks above, the database would **reject** a second ticket for the same seat. The error would roll back the transaction. A constraint cannot be forgotten by a developer.

### Expired holds
If someone abandons checkout, their hold simply expires. The `OR hold_expires_at < now()` condition lets the next person take the seat immediately, without waiting for a cleanup job. A background job also resets expired seats to `available` to keep the seat map tidy.

### What does *not* prevent double-booking
- The **seat-map cache** can be stale (it may show a sold seat as available). That only costs a polite `409`, never a double sale.
- The **CDN** and **waiting room** reduce load and keep order, but they do not coordinate writes.
- **Front-end checks** can be bypassed and are for user experience only.

---

## 7. Architecture

### Start simple
One app server and one database is enough for a normal day, since 25 requests/sec at peak is easy. Then add one component at a time, each for a named problem.

### Diagram

```
                         Users (browsers)
                                |
                                v
                     +---------------------+
                     |        CDN          |  static files, images, event pages
                     +---------------------+
                                |
                                v
                     +---------------------+
                     |    Waiting room     |  during big sales: admit users
                     |   (queue in Redis)  |  at a safe rate, first-come-first-served
                     +---------------------+
                                |
                                v
                     +---------------------+
                     |    Load balancer    |
                     +---------------------+
                       |        |        |
                       v        v        v
                    +-----+  +-----+  +-----+
                    | App |  | App |  | App |   stateless app servers
                    +-----+  +-----+  +-----+   (add more for big sales)
                       |        |        |
          +------------+--------+--------+-------------+
          |                                             |
          v                                             v
   +--------------+                             +---------------+
   |    Redis     |                             |  Message queue |
   | seat-map     |                             |  (tickets,     |
   | cache,       |                             |  emails, QR)   |
   | rate limits  |                             +---------------+
   +--------------+                                     |
          |                                             v
          |                                     +---------------+
          |                                     |   Workers     |
          |                                     +---------------+
          v
   +-----------------------+   replication   +-----------------+
   |  PostgreSQL PRIMARY   | --------------> |  Read replicas  |
   |  (all seat writes,    |                 |  (browsing,     |
   |   holds, orders)      |                 |   my tickets)   |
   +-----------------------+                 +-----------------+

                  App servers <--> Payment provider (external)
```

### Each component and the problem it solves

- **CDN:** serves images, scripts and event pages from locations near the user. *Problem:* worldwide users and heavy static content would otherwise hit our servers. It does not hold seat state.
- **Waiting room (queue backed by Redis):** during a big sale, 200,000 people arrive within minutes. The waiting room lets in only as many as the system can handle (for example 2,000 at a time) in arrival order, and shows everyone else their position. *Problem:* a stampede that would crush the app servers and database, and unfairness if the fastest connection always wins.
- **Load balancer + several stateless app servers:** spreads requests across servers, which keep no local state (sessions live in Redis or tokens). *Problem:* one server is too slow and a single point of failure. We add servers before a big sale, and if one dies the others continue.
- **Redis seat-map cache:** the seat map for an event is cached for 1 to 2 seconds. Thousands of people viewing the same event read the cache instead of running thousands of identical queries. *Problem:* reads overwhelm the database. The cache is allowed to be slightly stale because the database decides the real outcome (see section 6). Redis also holds rate-limit counters (per user and per IP) against bots.
- **PostgreSQL primary:** the single place where holds and sales are decided, using transactions and constraints. *Problem:* correctness. A single writer means no two servers can disagree about who owns a seat.
- **Read replicas:** handle browsing, search and "my tickets". They lag slightly, which is acceptable for those reads. *Problem:* the primary should spend its capacity on seat writes, and we want a replica ready if the primary fails.
- **Message queue and workers:** after the order commits, ticket PDFs/QR codes, confirmation emails and analytics are processed in the background. *Problem:* slow work must not delay the purchase response or fail the purchase if the email service is down.
- **Payment provider:** handles card details, so we never store them. Calls have timeouts, and the idempotency key protects against duplicates.

### How it survives the big sale

1. **Before the sale:** warm the cache, add app servers, and enable the waiting room for that event.
2. **At the start:** 200,000 people hit the waiting room, which is cheap because it only manages a queue. Only a controlled flow reaches the app servers (about 2,000 users at a time), so the load is the 3,300 requests/sec average at worst and the database never sees the full stampede.
3. **Seat map reads** come from Redis, not the database.
4. **Seat holds** go to the primary database as small, fast, single-row conditional updates on different rows. Most contention is on popular seats, and the loser gets a quick `409` rather than waiting.
5. **Payment and emails:** payment is limited to people holding seats (at most 20,000 holds at a time), and emails are queued.
6. **If something is overloaded**, the system degrades gracefully: the waiting room admits people more slowly, and browsing other events can be slowed, but seat sales stay correct.

---

## 8. Trade-offs

1. **Stale seat map vs speed.** The seat map is cached for 1 to 2 seconds, which keeps the database free for writes. The cost: users sometimes click a seat that was just taken and get a `409`. We accept this because the database still guarantees correctness, and the front end shows a friendly message and refreshes the map.

2. **Hold duration (5 minutes) vs fairness and revenue.** A longer hold gives people time to find their card; a shorter hold frees seats faster for people still waiting. Too long and abandoned carts lock seats during a sell-out; too short and honest buyers lose seats mid-payment. We chose 5 minutes, and the time can be tuned per event.

3. **Waiting room vs user experience.** The queue protects the system and makes access fair (first come, first served), but it makes users wait, and some will give up. The alternative (no queue) would risk a crash that ruins the sale for everyone. A random lottery would be fairer to people with slow connections, but less simple to explain.

4. **Single primary database vs complexity.** All seat writes go to one primary, which is simple and correct, but it is a limit on write capacity and a single point of failure (mitigated by a replica that can take over). At 1,700 write attempts/sec on small rows this is fine. If TicketHub grows much larger, we could shard by event (each event's seats on its own database) at the cost of more complexity.

5. **More servers cost money.** Extra app servers and replicas are only needed for big sales, so we scale up for them and back down afterwards. The cost is justified by the revenue of a sold-out event.

---

## 9. Risks and open questions

- **Payment succeeds but commit fails (or the reverse).** Mitigation: idempotency keys, a reconciliation job, and automatic voids/refunds.
- **Bots buying seats for resale.** Rate limits, per-account seat limits, and possibly CAPTCHA at the waiting room. Needs further work.
- **Primary database failure during a sale.** Failover to a replica needs to be tested before big events.
- **Open question:** should popular events use a random lottery instead of a first-come queue?
