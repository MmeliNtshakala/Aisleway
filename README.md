# Aisleway

> Your aisles, wherever you are.

Aisleway is an online shop that delivers anywhere in South Africa through a courier. It launches with **soap** as its first product, then grows into more aisles (perfumes, supplements, and more) and, later, a marketplace where other sellers can run their own stalls.

This README is the working blueprint for the website: what we are building, how it is structured, how we test it, how we ship it, and how we keep customer data safe.

**Status:** Draft v0.1. Items in `[SQUARE BRACKETS]` are placeholders to fill in. Section 17 lists the decisions still open.

---

## Table of contents

1. [Goals and scope](#1-goals-and-scope)
2. [Business rules](#2-business-rules)
3. [Tech stack](#3-tech-stack)
4. [Architecture](#4-architecture)
5. [Project structure](#5-project-structure)
6. [Domain model and order states](#6-domain-model-and-order-states)
7. [Test-driven development workflow](#7-test-driven-development-workflow)
8. [Payments](#8-payments)
9. [Delivery and courier](#9-delivery-and-courier)
10. [Security and privacy (POPIA)](#10-security-and-privacy-popia)
11. [Configuration](#11-configuration)
12. [Running locally with Docker](#12-running-locally-with-docker)
13. [CI/CD](#13-cicd)
14. [Deployment and operations](#14-deployment-and-operations)
15. [Frontend](#15-frontend)
16. [Roadmap](#16-roadmap)
17. [Open decisions](#17-open-decisions)
18. [Contributing and conventions](#18-contributing-and-conventions)

---

## 1. Goals and scope

### Goals

- Let customers order products online and pay with the bank they already use.
- Deliver anywhere via a courier, with the delivery cost shown clearly and paid in the same checkout.
- Keep the owner's workflow simple: a paid order appears in an order list, the owner packs it, drops it at the courier, and the customer is notified.
- Be secure by default. Card and bank details never touch our servers.
- Be built test-first, so that rules about money and orders are proven by tests before they ship.
- Stay cheap to run at launch, and structured so marketplace features can be added later without a rewrite.

### In scope for launch (MVP)

- Product catalogue (soap first), product pages, cart, checkout.
- Minimum order value and delivery fee rules (section 2).
- Online payment through a South African payment provider, including instant EFT (bank selection).
- Order confirmation, order tracking page, and transactional emails.
- Admin area: product management, order list, mark packed/shipped, enter tracking number, refund/cancel.
- Docker-based deployment and a CI/CD pipeline.

### Out of scope for launch

- Multi-seller marketplace ("stalls"). The design keeps room for it (section 16), but nothing is built yet.
- Customer accounts with saved cards. Guest checkout first. We never store card data.
- International shipping.
- Native mobile apps. The site must work well on phones.

---

## 2. Business rules

These rules live in the **domain core** (section 4) and are covered by unit tests. They must never be enforced only in the browser.

| Rule | Detail |
|---|---|
| Currency | South African rand (ZAR). All money is stored and calculated as **integer cents**. Never floats. |
| Minimum order | Product subtotal must be **at least R100.00** (10 000 cents). Delivery is not counted toward the minimum. |
| Delivery fee | Shown as its own line at checkout. Calculated on the server from the order's total weight and parcel size. |
| Single payment | The customer pays one total: product subtotal + delivery fee. |
| Prices | Always calculated on the server. The browser never sends a trusted price. |
| Shippable flag | Each product has a `can_ship_by_courier` flag. Restricted items (for example some liquids or perfumes) cannot be added until the courier confirms they are allowed. |
| Cancellation | Free until the parcel is handed to the courier. After that, the lost/damaged parcel flow applies. `[CONFIRM POLICY]` |
| Free delivery | Not at launch. Configurable threshold later (for example from `[AMOUNT]`). |

### Example

An order of 4 bars of soap at `[PRICE]` each, with a subtotal above R100, passes the minimum. If delivery for the parcel size is R79, the checkout shows:

```
Subtotal   [AMOUNT]
Delivery   R79.00 (+ fuel surcharge buffer)
Total      [TOTAL]
```

---

## 3. Tech stack

Chosen to be affordable, mainstream, and easy to test.

| Layer | Choice | Why |
|---|---|---|
| Language | **Python 3.12** | Team language; strong testing ecosystem. |
| Web framework | **FastAPI** | Typed, async, automatic request validation, easy dependency injection for fakes in tests. |
| Validation | **Pydantic v2** | Request/response models and settings. |
| Database | **PostgreSQL 16** (SQL) | Reliable, transactional, free. |
| ORM and migrations | **SQLAlchemy 2.x** + **Alembic** | Typed models, versioned schema changes. |
| Templates | **Jinja2** | Server-rendered pages: fast, SEO-friendly, small attack surface. |
| Frontend interactivity | **HTMX** + a little vanilla JS | Cart and checkout updates without a JS framework. |
| Styling | **Tailwind CSS** (built in CI) or plain CSS | No runtime cost; consistent with the Aisleway brand. |
| Testing | **pytest**, **pytest-cov**, **httpx**, **respx**, **factory-boy**, **Hypothesis** (optional) | See section 7. |
| Lint, format, types | **Ruff**, **mypy** | Fast, enforced in CI. |
| Password hashing | **argon2-cffi** | Admin login. |
| Containers | **Docker** + **Docker Compose** | Same environment in dev, CI and production. |
| Reverse proxy and HTTPS | **Caddy** | Automatic HTTPS certificates. |
| CI/CD | **GitHub Actions** | Free tier is enough to start. `[CONFIRM HOST]` |
| Container registry | GitHub Container Registry (GHCR) | Free for private use within limits. `[CONFIRM]` |
| Hosting | A small VPS or container host, ideally in South Africa or a nearby region | Low cost; lower latency and easier POPIA reasoning. `[DECIDE]` |
| Email | A transactional email provider via SMTP/API | Behind an adapter, so it can be swapped. `[DECIDE]` |

The brand colours for the frontend are in section 15.

---

## 4. Architecture

### System overview

```mermaid
flowchart LR
    C[Customer browser] --> S[Aisleway server]
    A[Admin dashboard] --> S
    S -->|create payment| P[Payment provider]
    P -->|signed webhook: paid| S
    S -->|rates, booking| K[Courier]
    S --> D[(PostgreSQL)]
    S --> E[Email service]
```

The browser and the admin dashboard only ever talk to **our server**. The server is the single place that sets prices, enforces rules and talks to outside services.

### Inside the server (ports and adapters)

```mermaid
flowchart TB
    R[API routes - thin] --> CORE[Domain core - pure logic]
    CORE --> PORTS[Ports - interfaces]
    PORTS --> AP[Payments adapter]
    PORTS --> AK[Courier adapter]
    PORTS --> AD[Database adapter]
    PORTS --> AE[Email adapter]
```

- **Domain core**: pure Python, no network, no database, no clock calls. Pricing, minimum order, delivery fee, order state transitions. Unit-tested in milliseconds.
- **Ports**: small interfaces (`typing.Protocol`) owned by the core, for example `PaymentGateway`, `CourierService`, `OrderRepository`, `Notifier`.
- **Adapters**: real implementations (PayFast/Ozow, the courier API, SQLAlchemy, SMTP) plus **fakes** used in tests.
- **Routes**: parse the request, call the core, return the response. No business logic.

**Rule of thumb:** if a function needs a database, the network or the current time, it lives in an adapter. Time is passed in through a `Clock` port so tests can control it.

---

## 5. Project structure

```
aisleway/
├── app/
│   ├── domain/                 // pure logic, no I/O
│   │   ├── money.py            // Money value object (cents)
│   │   ├── products.py         // Product, weight, size, shippable flag
│   │   ├── cart.py             // Cart and line items
│   │   ├── pricing.py          // subtotal, minimum order, totals
│   │   ├── delivery.py         // parcel size selection, delivery fee
│   │   ├── orders.py           // Order entity and state machine
│   │   └── errors.py           // domain exceptions
│   ├── ports/                  // interfaces the core depends on
│   │   ├── payments.py
│   │   ├── courier.py
│   │   ├── repositories.py
│   │   ├── notifier.py
│   │   └── clock.py
│   ├── adapters/
│   │   ├── payments/           # payfast.py, ozow.py, fake.py
│   │   ├── courier/            # courier_api.py, manual.py, fake.py
│   │   ├── db/                 # models.py, repositories.py, session.py
│   │   └── email/              # smtp.py, fake.py
│   ├── api/                    # FastAPI routers, dependencies
│   │   ├── shop.py             # catalogue, cart, checkout
│   │   ├── webhooks.py         # payment provider callbacks
│   │   └── admin.py            # admin routes (auth required)
│   ├── web/
│   │   ├── templates/          # Jinja2 templates
│   │   └── static/             # compiled CSS, images
│   ├── config.py               # typed settings from environment
│   └── main.py                 # app factory
├── migrations/                 # Alembic migrations
├── tests/
│   ├── unit/                   # domain core, no I/O
│   ├── integration/            # adapters against Postgres and sandboxes
│   ├── contract/               # webhook signature and provider contracts
│   └── e2e/                    # a few full-journey tests
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml   # local dev
│   └── Caddyfile
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
├── pyproject.toml
├── Makefile
├── .env.example
└── README.md
```

---

## 6. Domain model and order states

### Core entities

- **Product**: `id`, `name`, `price_cents`, `weight_g`, `length_mm/width_mm/height_mm`, `can_ship_by_courier`, `active`. Add a nullable `seller_id` from day one (unused at launch) so the marketplace phase does not need a data redesign.
- **Cart / CartLine**: product, quantity.
- **Order**: `id`, public `reference`, customer details, address, lines (price copied at the time of order), `subtotal_cents`, `delivery_cents`, `total_cents`, `status`, `courier_tracking_no`, timestamps.
- **Payment**: provider, provider reference, amount, status, raw webhook event id (for idempotency).
- **ProcessedWebhookEvent**: stores provider event ids so a replayed message is ignored.
- **AdminUser**: email, password hash, two-step login secret.

### Order state machine

```mermaid
stateDiagram-v2
    [*] --> pending_payment
    pending_payment --> paid: verified paid webhook
    pending_payment --> payment_failed: declined or error
    pending_payment --> expired: not paid in time
    payment_failed --> pending_payment: customer retries
    paid --> packed: admin marks packed
    paid --> cancelled: cancelled before shipping
    packed --> shipped: courier handover and tracking number
    packed --> cancelled: cancelled before shipping
    shipped --> delivered: courier confirms
    shipped --> claim_open: lost or damaged
    claim_open --> replaced: replacement sent
    claim_open --> refund_pending: refund chosen
    cancelled --> refund_pending
    refund_pending --> refunded: refund confirmed
    delivered --> [*]
    replaced --> [*]
    refunded --> [*]
    expired --> [*]
```

**Invariants (each has tests):**

1. An order can only become `paid` through a **verified** payment event for the **exact** expected amount.
2. Only `paid` or later states can be packed or shipped. An unpaid order can never be shipped.
3. A shipped order cannot be cancelled. It goes through the claim flow.
4. Totals always equal `subtotal + delivery`, and `subtotal` is at least the minimum order.
5. Illegal transitions raise `InvalidTransition`. They never silently succeed.

---

## 7. Test-driven development workflow

We write the failing test first, make it pass with the simplest code, then refactor.

### Cycle

1. **Red**: write a test that describes one behaviour. Run it and watch it fail for the right reason.
2. **Green**: write the smallest code that passes.
3. **Refactor**: clean up with the tests green.
4. Commit small. Each commit leaves the test suite green.

### Test pyramid

| Level | What it covers | Speed | Uses |
|---|---|---|---|
| **Unit** (most tests) | Domain core: money, pricing, minimum order, delivery fee, state machine | Milliseconds | No I/O at all |
| **Integration** | Database adapter (real Postgres in Docker), payment/courier adapters against sandboxes or recorded responses | Seconds | Postgres service, `respx` to mock HTTP |
| **Contract** | Webhook signature checking, replay protection, provider payload formats | Fast | Fixtures of real provider payloads |
| **End-to-end** (few) | Browse → cart → pay in test mode → order appears in admin | Slowest | Full stack in Docker with fakes or sandboxes |

### Rules

- No domain code without a test that failed first.
- Domain tests must not touch the network, filesystem, database or system clock.
- Every bug fix starts with a test that reproduces the bug.
- Use factories for test data, not copied fixtures.
- CI enforces a coverage gate: **`[90]%` overall and 100% on `app/domain/`**. `[CONFIRM]`
- Test names describe behaviour: `test_order_below_minimum_is_rejected`.

### First tests to write (in this order)

**Money**
- Adding and multiplying cents is exact; formatting renders `R79.00`.
- Negative amounts are rejected where not allowed.

**Minimum order**
- Subtotal R99.99 is rejected; R100.00 is accepted; R100.01 is accepted.
- Delivery fee does not count toward the minimum.

**Delivery fee**
- 4 items totalling 600 g selects the smallest parcel size.
- An order that exceeds a size's weight limit moves up to the next size.
- An order that exceeds the largest size is rejected with a clear error.
- The fuel-surcharge buffer is added from configuration.

**Shippable flag**
- Adding a product with `can_ship_by_courier = False` raises an error.

**Order state machine**
- `pending_payment → paid` succeeds only with a verified payment of the exact amount.
- Paying the wrong amount leaves the order unpaid and flags it for review.
- A paid event delivered twice marks the order paid once.
- A `pending_payment` order cannot be marked packed or shipped.
- A shipped order cannot be cancelled.

**Webhook security (contract tests)**
- A valid signature is accepted.
- A missing, malformed or forged signature is rejected.
- A replayed event id is ignored.

### Example test (pytest)

```python
import pytest
from app.domain.money import Money
from app.domain.pricing import validate_minimum_order, MIN_ORDER
from app.domain.errors import BelowMinimumOrder


@pytest.mark.parametrize("cents", [0, 9_999])
def test_order_below_minimum_is_rejected(cents):
    with pytest.raises(BelowMinimumOrder):
        validate_minimum_order(Money(cents))


@pytest.mark.parametrize("cents", [10_000, 10_001])
def test_order_at_or_above_minimum_is_accepted(cents):
    validate_minimum_order(Money(cents))  # does not raise


def test_minimum_order_is_one_hundred_rand():
    assert MIN_ORDER == Money(10_000)
```

### Faking outside services

```python
class FakePaymentGateway:
    """Implements the PaymentGateway port for tests."""
    def __init__(self):
        self.created = []
        self.fail_next = False

    def create_payment(self, order_ref, amount):
        if self.fail_next:
            self.fail_next = False
            raise PaymentDeclined()
        self.created.append((order_ref, amount))
        return PaymentLink(url=f"https://fake.test/pay/{order_ref}")
```

### Handy commands

```bash
make test           # unit + contract (fast)
make test-all       # includes integration (needs Docker for Postgres)
make test-watch     # re-run on file change
make cov            # coverage report
make lint           # ruff + mypy
```

---

## 8. Payments

Customers pay on the **payment provider's hosted page**. Our servers never see card numbers or bank logins. The provider settles the money into our bank account after its fee.

### Requirements

- Cards and **instant EFT** (customer picks their bank). Plan for two payment methods, for example a card gateway plus an instant-EFT provider, behind one `PaymentGateway` port. `[DECIDE PROVIDERS]`
- Start in the provider's **sandbox/test mode** and keep it there until the security review passes.
- Refunds from the admin area (through the provider where supported).

### Flow

1. Checkout creates an order in `pending_payment` with server-calculated totals.
2. The server asks the payment adapter for a payment link and redirects the customer.
3. The customer pays on the provider page.
4. The provider calls our **webhook**. The adapter verifies it (signature and/or a server-to-server confirmation, per the provider's documentation), checks the **amount and order reference**, and checks the event id has not been processed before.
5. Only then does the core mark the order `paid`, and the notifier emails the customer and alerts the owner.

### Webhook checklist (all covered by contract tests)

- Verify authenticity exactly as the provider documents.
- Compare the paid amount to the order total in cents.
- Be idempotent: processing the same event twice has no extra effect.
- Return the status code the provider expects, quickly. Do slow work after responding.
- Log the event id and result. Never log secrets or full payment payloads containing personal data.

### Fees and settlement

Provider fees are a percentage plus a small fixed amount, and settlement takes about one to two business days. Fees and settlement terms change, so check the live terms before launch. Keep a small cash float to cover courier costs while waiting for settlement. `[CONFIRM FEES]`

---

## 9. Delivery and courier

### Operating model

The owner packs orders, drops the parcel at the courier's locker or kiosk, and the courier delivers to the customer's door. The customer pays the delivery fee at checkout, and the owner pays the courier.

### Parcel sizes and fee table

Courier prices are **configuration, not code**, loaded from a table so they can be updated without a deploy. The values below come from the courier's price screen effective **1 September 2026** (VAT-inclusive, door services exclude a monthly fuel surcharge). **Verify them against the courier's app or a business quote before launch.**

| Size | Max weight | Dimensions (cm) | Locker to door | Kiosk to door | Locker to locker |
|---|---|---|---|---|---|
| Small | 2 kg | 60 × 17 × 8 | R79 | R93 | R59 |
| Small plus | 5 kg | 60 × 41 × 8 | R89 | R105 | R69 |
| Medium | 10 kg | 60 × 41 × 19 | R119 | R135 | R79 |
| Large | 15 kg | 60 × 41 × 41 | R176 | R210 | R109 |
| Extra large | 20 kg | 60 × 41 × 69 | R239 | R280 | R149 |

`[CONFIRM SIZE LABELS AND PRICES]`

### Design notes

- `delivery.py` picks the smallest size that fits the order's total weight **and** packed dimensions, then looks up the fee from config. Include packaging weight in the calculation.
- The fee shown to the customer adds a configurable **fuel-surcharge buffer** so a monthly change does not push delivery below cost.
- The `CourierService` port has two implementations: a **manual** adapter at launch (the owner enters the tracking number in the admin area) and an **API** adapter later, if the courier offers an API for rates, booking and tracking. `[CONFIRM API ACCESS]`
- Confirm in writing which products the courier will carry (soap, perfume, supplements) before each product goes live. Reflect the answer in `can_ship_by_courier`.
- Photograph each packed parcel before sealing, for damage and loss claims.

---

## 10. Security and privacy (POPIA)

Security is a feature of every pull request, not a final step.

### Application security

- **No card or bank data stored**, ever. Hosted payment page only.
- **HTTPS everywhere**: Caddy terminates TLS; HTTP redirects to HTTPS; HSTS enabled.
- **Server-side pricing**: totals are recomputed on the server from product ids and quantities.
- **Input validation** with Pydantic on every endpoint. Reject unknown fields.
- **SQL injection**: use SQLAlchemy parameters only. No string-built SQL.
- **XSS**: Jinja2 autoescaping on; no `|safe` on user content; a strict Content Security Policy.
- **CSRF** protection on all state-changing form routes, including HTMX requests.
- **Security headers**: CSP, `X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options` or `frame-ancestors`.
- **Rate limiting** on checkout, login and webhook-adjacent routes.
- **Admin area**: Argon2 password hashes, two-step login, session timeout, secure and `HttpOnly` cookies, an audit log of refunds and status changes.
- **Webhooks**: signature verification, amount checks and replay protection (section 8).
- **Secrets** come from environment variables or a secrets manager, never from the repository. Rotate keys if exposed.
- **Dependencies**: pinned versions, automated update PRs, and `pip-audit` plus an image scan in CI.
- **Errors**: generic messages to users, details in logs. No stack traces in responses.
- **Backups**: automated database backups, encrypted, with a **tested restore** before launch.

### Customer data and POPIA

South Africa's Protection of Personal Information Act (POPIA) applies to the personal data we hold. Before launch, confirm the details with a qualified adviser. `[LEGAL REVIEW NEEDED]`

- **Collect only what is needed**: name, phone, email, delivery address, order history.
- Publish a **privacy policy** explaining what we collect, why, how long we keep it, and who we share it with (payment provider, courier, email service).
- Have a documented process to **access, correct and delete** a customer's data on request, within what the law and tax records allow.
- Check the **information officer** and registration requirements with the Information Regulator.
- Limit who can see customer data. Log admin access to orders.
- Record a **breach response plan**: who to notify, how, and how fast.
- Retention: define how long orders and personal data are kept. `[DECIDE]`

### Before go-live

- [ ] Independent security review or penetration test
- [ ] All secrets rotated from test values
- [ ] Backup restore tested
- [ ] Payment provider moved from sandbox to live with a real low-value test order and refund
- [ ] Privacy policy, terms and returns policy published

---

## 11. Configuration

All configuration is read from environment variables through a typed `Settings` class (`app/config.py`). Copy `.env.example` to `.env` for local work. **Never commit `.env`.**

```dotenv
# App
APP_ENV=local                  # local | test | staging | production
APP_SECRET_KEY=change-me       # long random value
APP_BASE_URL=http://localhost:8000
ALLOWED_HOSTS=localhost

# Database
DATABASE_URL=postgresql+psycopg://aisleway:aisleway@db:5432/aisleway

# Business rules
MIN_ORDER_CENTS=10000
FUEL_SURCHARGE_BUFFER_PERCENT=[10]
COURIER_RATES_FILE=config/courier_rates.json

# Payments (use sandbox values until launch)
PAYMENT_PROVIDER_CARD=[provider]
PAYMENT_PROVIDER_EFT=[provider]
PAYMENT_MERCHANT_ID=
PAYMENT_MERCHANT_KEY=
PAYMENT_WEBHOOK_SECRET=
PAYMENT_SANDBOX=true

# Courier
COURIER_MODE=manual            # manual | api
COURIER_API_KEY=

# Email
EMAIL_PROVIDER=[provider]
EMAIL_API_KEY=
OWNER_ALERT_EMAIL=[YOUR EMAIL]

# Admin
ADMIN_SESSION_MINUTES=30
```

---

## 12. Running locally with Docker

### Prerequisites

- Docker and Docker Compose
- Make (optional but convenient)
- Python 3.12 if you want to run tests outside Docker

### Quick start

```bash
git clone [REPO URL] aisleway && cd aisleway
cp .env.example .env
make up            # docker compose up --build
make migrate       # run Alembic migrations
make seed          # load sample products
```

The site runs at `http://localhost:8000`. Admin is at `/admin`.

### Dockerfile (multi-stage, non-root)

```dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS builder
COPY pyproject.toml ./
RUN pip install --no-cache-dir --prefix=/install .

FROM base AS runtime
RUN adduser --disabled-password --gecos "" appuser
COPY --from=builder /install /usr/local
COPY app ./app
COPY migrations ./migrations
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz')"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml (outline)

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: aisleway
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: aisleway
    volumes: [db_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U aisleway"]
      interval: 5s
      retries: 10

  app:
    build: { context: .., dockerfile: docker/Dockerfile }
    env_file: ../.env
    depends_on:
      db: { condition: service_healthy }

  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
    depends_on: [app]

volumes:
  db_data:
  caddy_data:
```

The local override file exposes port 8000 directly, mounts the source for reload, and skips Caddy.

### Make targets

```
make up | down | logs | shell
make migrate | migration name="add_orders"
make seed
make test | test-all | cov | lint | fmt
```

---

## 13. CI/CD

GitHub Actions runs on every pull request and on pushes to `main`.

### Pipeline stages

```mermaid
flowchart LR
    PR[Pull request] --> L[Lint and types]
    L --> T[Unit and contract tests]
    T --> I[Integration tests with Postgres]
    I --> B[Build Docker image]
    B --> SC[Dependency and image scan]
    SC --> M{Merge to main}
    M --> ST[Deploy to staging]
    ST --> SM[Smoke tests]
    SM --> AP{Manual approval}
    AP --> PR2[Deploy to production]
    PR2 --> HC[Health check and rollback on failure]
```

### `ci.yml` (outline)

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: aisleway
          POSTGRES_PASSWORD: test
          POSTGRES_DB: aisleway_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U aisleway"
          --health-interval 5s --health-retries 10
    env:
      APP_ENV: test
      DATABASE_URL: postgresql+psycopg://aisleway:test@localhost:5432/aisleway_test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -e ".[dev]"
      - run: ruff check . && ruff format --check .
      - run: mypy app
      - run: pytest tests/unit tests/contract --cov=app --cov-fail-under=90
      - run: alembic upgrade head
      - run: pytest tests/integration

  build-and-scan:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install pip-audit && pip-audit
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: docker/Dockerfile
          push: false
          tags: aisleway:${{ github.sha }}
      # add an image vulnerability scan step here
```

### `deploy.yml` (outline)

- Triggered after CI succeeds on `main`.
- Builds and pushes the image to the registry, tagged with the commit SHA.
- **Staging**: deploys automatically, runs migrations, runs smoke tests (`/healthz`, a test checkout against the sandbox).
- **Production**: uses a GitHub **environment with required reviewers**, so a human approves each release. Runs migrations, swaps containers, checks `/healthz`, and rolls back to the previous image tag if the check fails.
- Deployment secrets (server SSH key, registry token, production environment variables) live in GitHub **Secrets**, scoped per environment.

### Branching and releases

- Trunk-based: short-lived branches, pull request, squash merge to `main`.
- Every merge to `main` is deployable. Tags like `v0.1.0` mark releases.
- Database migrations must be backwards compatible with the previous app version, so rollback is safe. Destructive changes happen in two steps across two releases.

### Branch protection

- Require passing CI and at least one review before merge.
- No direct pushes to `main`.
- Require signed commits `[OPTIONAL]`.

---

## 14. Deployment and operations

- **Environments**: `local`, `staging` (same setup as production, sandbox payments), `production`.
- **Health**: `/healthz` (process alive) and `/readyz` (database reachable).
- **Logging**: structured JSON to stdout, collected by the host. Include a request id. Never log personal data beyond what is needed, and never log secrets.
- **Monitoring and alerts**: uptime check on `/healthz`, an alert on error-rate spikes, and an alert when a paid order has been waiting to be packed longer than `[24h]`.
- **Backups**: daily automated Postgres backups, retained for `[30]` days, restore rehearsed before launch and then regularly.
- **Migrations**: run by the deploy job before the new app version starts.
- **Runbook**: keep short guides for "payment webhook failing", "site down", "restore from backup" and "rotate a leaked key" in `docs/runbooks/`.
- **Cost control**: a single small server and a single database to start. Scale only when numbers demand it.

---

## 15. Frontend

Server-rendered pages with Jinja2, enhanced with HTMX. This keeps the stack small, fast on mobile data, and easy to secure.

### Pages (match the designs on the Aisleway brand board)

- Home with "Shop by aisle"
- Product page (soap first)
- Cart and checkout, with minimum order message and separate delivery line
- Payment redirect and return pages
- Order confirmation
- Order tracking
- Admin: order list, order detail, products, refunds

### Brand tokens

| Token | Value |
|---|---|
| Fresh teal (primary) | `#0B6E6B` |
| Warm coral (accent) | `#FF6B57` |
| Leaf green | `#5FAE4B` |
| Charcoal (text) | `#1F2A2E` |
| Off-white (background) | `#F7F5F0` |
| Font | Nunito |

Use charcoal text on coral and leaf green, and white text on teal and charcoal, to keep contrast readable.

### Guidelines

- Mobile first. Large tap targets (at least 44 px).
- Show the **minimum order message** before checkout, not after a failure.
- Always show delivery as its own line.
- Accessible by default: semantic HTML, labels on every field, visible focus states, adequate contrast.
- Keep pages light: compiled Tailwind, no heavy JS bundles, optimised product images.

---

## 16. Roadmap

| Phase | Focus | Outcome |
|---|---|---|
| **0. Foundations** | Repo, Docker, CI, test setup, config | Green pipeline on an empty app |
| **1. Domain core (TDD)** | Money, pricing, minimum order, delivery fee, order state machine | 100% tested rules |
| **2. Catalogue and cart** | Products, cart, checkout without payment | Customers can build an order |
| **3. Payments** | Payment adapter, webhooks, sandbox testing, refunds | Orders can be paid and refunded in test mode |
| **4. Fulfilment** | Admin order list, mark packed/shipped, tracking page, emails | Owner can run daily operations |
| **5. Hardening** | Security review, backups tested, POPIA documents, load check | Ready for real customers |
| **6. Launch** | Live payments, soap on sale | First real orders |
| **7. Growth** | More aisles (perfume, supplements), courier API, free delivery threshold | Wider catalogue |
| **8. Marketplace** | Seller accounts, seller stalls, split payments, seller payouts, seller verification | Other sellers can sell on Aisleway |

### Keeping the marketplace door open

- `seller_id` on products and order lines from the start (nullable).
- Money rules centralised in the domain core, so commission and splits can be added there.
- Payment port designed to allow split payouts later. Confirm the chosen provider supports it before the marketplace phase.

---

## 17. Open decisions

- [ ] Payment providers (card and instant EFT) and their live fees
- [ ] Courier API availability and the business account rates
- [ ] Hosting provider and region
- [ ] Email provider
- [ ] Delivery fee policy: always charged, or free above a threshold
- [ ] Cancellation and returns policy
- [ ] Data retention periods
- [ ] Whether to launch with a guest checkout only (current plan) or add customer accounts
- [ ] Domain name and trademark confirmation
- [ ] VAT registration status and how VAT appears on invoices

---

## 18. Contributing and conventions

- **TDD is the default.** A pull request that adds behaviour without a test will not be merged.
- **Commits**: small and focused. Use clear messages, for example `feat(pricing): reject orders below R100`.
- **Code style**: Ruff for lint and format, mypy for types. Full type hints in `app/domain/`.
- **Money**: integers in cents, always. Never floats.
- **No business logic in routes.** Routes call the domain core.
- **No outside calls in the domain core.** Use ports and adapters.
- **Secrets** never enter the repository. If one does, rotate it immediately.
- **Documentation**: update this README and add short architecture decision records in `docs/adr/` when a decision in section 17 is made.

---

## Licence

`[CHOOSE LICENCE OR MARK AS PROPRIETARY]`

© `[YEAR]` Aisleway. All rights reserved.
