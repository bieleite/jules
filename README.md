# jules

# Project: E‑commerce Webapp (Amazon-like)

**Stack:** Angular 17+, Java 21, Spring Boot 3.3, PostgreSQL 16, Docker, NGINX, Keycloak (optional) / Spring Security OAuth2 Client, PayPal + Stripe adapters, Redis, OpenSearch (optional), Matomo (self‑hosted) for analytics, ag‑Grid in Admin.

**Targets:** Classic Linux VM, AWS, GCP, Docker (compose/k8s). CI/CD ready.

**Non‑functional goals:** Security (OWASP ASVS L2+), Accessibility (WCAG 2.2 AA + high contrast), Performance (Core Web Vitals), Observability (logs/metrics/traces), Test automation end‑to‑end.

---

## 1) High‑Level Features (refined)

1. **Catalog & Discovery**

   * Browse products by category; filter/sort by price, newest, rating.
   * Search with typeahead (Postgres full‑text → optional OpenSearch if scale).
   * Product page: price, photos (gallery/zoom), description, materials (“made of”), category, stock status, customer reviews, Q\&A.
   * "You might also like"—4 recommendations from same category (fallback rules + future ML).

2. **Cart & Checkout**

   * Add/Remove/Update quantities; persistent cart (anon cookie + merge on login).
   * Delivery price calculation: simple table by weight/destination + courier API adapters.
   * Checkout steps: address → shipping method → payment → confirmation.
   * Payments via **adapters** (feature‑flagged): PayPal, Stripe (credit card). Admin can enable/disable.

3. **Accounts & Auth**

   * Email/password (optional) + **Google**/**Apple** login.
   * User profile: name, emails, phone, addresses, order history, GDPR export/delete.
   * Roles: CUSTOMER, ADMIN.

4. **Admin**

   * Dashboard: traffic, conversion, sales, product views.
   * Products CRUD with **ag‑Grid** editable table (inline edit, bulk import CSV/Excel).
   * Inventory & pricing management; media upload with S3-compatible storage.
   * Analytics: views by product, top categories, funnel metrics.
   * Feature flags (payments on/off, AI description on/off, maintenance banner).

5. **AI Product Description (Local)**

   * When an image is uploaded, a **local VLM** service generates a first draft (e.g., Ollama + LLaVA/Florence2). Prompt tuned for commercial tone; editor for manual tweaks. Fallback to external API if enabled.

6. **Theming & Accessibility**

   * Site‑wide toggle: **Light**, **Dark**, **High‑contrast**.
   * Keyboard navigation, ARIA labels, focus states, color contrast 7:1 in HC mode.

---

## 2) Architecture Overview

* **Frontend (Angular)**

  * Angular 17 standalone components, SSR via Angular Universal.
  * State: NgRx (signals/store) for cart, auth, catalog.
  * UI: Angular Material + ag‑Grid (Enterprise not required; Community ok).
  * i18n: built‑in Angular i18n (pt‑BR, fr‑FR, en‑US initially). Currency pipes per locale.

* **Backend (Spring Boot 3.3, Java 21)**

  * Modules: catalog, pricing, inventory, cart, checkout, orders, payments, auth, analytics, media, ai‑description.
  * REST API + OpenAPI 3; (optional) GraphQL gateway for product detail/aggregation.
  * Security: Spring Security, OAuth2 client (Google/Apple), Resource Server (JWT), rate limits, input validation.
  * Persistence: JPA/Hibernate → PostgreSQL; Redis for sessions/cache; Flyway/Liquibase migrations.
  * Media storage: S3‑compatible (MinIO locally) + img resize variants via queue/worker.
  * Analytics events (view/add‑to‑cart/purchase): async to Kafka/RabbitMQ (or Postgres queue) → aggregated for admin dashboards.

* **AI Service**

  * Separate microservice (container) exposing `/describe` that accepts image(s) and returns text. Provider interface: `local` (Ollama), `openai` (optional). Feature flagged.

* **Payments**

  * Payment provider SPI: `PaypalAdapter`, `StripeAdapter`. Toggle per environment. Webhook handling + signature verification.

* **Delivery Calculation**

  * Pluggable strategy: table rates (weight x zone), carrier API adapters later.

* **Observability**

  * Logs: JSON to STDOUT → Loki/ELK; Metrics: Micrometer → Prometheus; Tracing: OpenTelemetry.

---

## 3) Data Model (relational)

**Core Tables**

* `users(id, email, hash, name, phone, created_at, role)`
* `oauth_accounts(id, user_id, provider, provider_user_id, created_at)`
* `addresses(id, user_id, line1, line2, city, state, zip, country, is_default)`
* `categories(id, parent_id, name, slug)`
* `products(id, category_id, sku, name, description, materials, created_at, updated_at, is_active)`
* `product_media(id, product_id, url, alt, sort_order)`
* `inventory(id, product_id, stock, reserved)`
* `prices(id, product_id, currency, amount, valid_from, valid_to)`
* `carts(id, user_id, anon_token, created_at, updated_at)`
* `cart_items(id, cart_id, product_id, qty, price_snapshot)`
* `orders(id, user_id, status, total_amount, currency, shipping_fee, address_id, created_at)`
* `order_items(id, order_id, product_id, qty, unit_price)`
* `payments(id, order_id, provider, status, provider_ref, created_at)`
* `shipments(id, order_id, carrier, tracking, cost, status)`
* `reviews(id, product_id, user_id, rating, title, body, created_at, is_approved)`
* `analytics_events(id, user_id, session_id, type, product_id, meta, ts)`

**Indexes**

* GIN on `products(name, description)` (tsvector), btree on `prices(product_id, valid_from DESC)`, `analytics_events(ts)`, `orders(user_id, created_at DESC)`.

---

## 4) API Surface (REST, sample)

```
GET   /api/v1/categories
GET   /api/v1/products?category=slug&sort=price|newest&order=asc|desc&page=&size=
GET   /api/v1/products/{id}
GET   /api/v1/products/{id}/related?limit=4
POST  /api/v1/cart/items { productId, qty }
PATCH /api/v1/cart/items/{itemId} { qty }
DELETE /api/v1/cart/items/{itemId}
POST  /api/v1/checkout/shipping-quote { addressId|address, cartId }
POST  /api/v1/checkout/create-order
POST  /api/v1/payments/{provider}/create-intent { orderId }
POST  /api/v1/payments/{provider}/webhook  (no auth, sig verify)
GET   /api/v1/orders/my
GET   /api/v1/auth/me

// Admin
GET   /api/v1/admin/dashboard
GET   /api/v1/admin/products?query=&page=&size=
POST  /api/v1/admin/products
PUT   /api/v1/admin/products/{id}
PATCH /api/v1/admin/products/{id}
POST  /api/v1/admin/products/{id}/media
POST  /api/v1/admin/bulk-import
GET   /api/v1/admin/analytics/product-views?from=&to=&productId=
POST  /api/v1/admin/ai/describe  (image upload)
PATCH /api/v1/admin/feature-flags/{flag} { enabled }
```

* OpenAPI 3 YAML generated; use springdoc-openapi.

---

## 5) Security & Compliance

* **AuthN/AuthZ:** Spring Security, JWT (RS256). OAuth2 login (Google, Apple). CSRF protection (cookies sameSite=strict + token), CORS whitelists.
* **Input Validation:** Bean Validation, size limits, file type whitelisting, image AV scanning (ClamAV container) on upload.
* **Secrets:** Externalized via env/Secrets Manager; JCE strong crypto; password hashing Argon2id.
* **Payments:** PCI‑aware design (tokenize; no PAN stored). Webhook signature checks, idempotency keys.
* **OWASP:** Use ESAPI encoders where needed, security headers (CSP, HSTS, X‑Frame‑Options deny), rate limiting & IP blocking.
* **GDPR:** Data export/delete, consent for analytics, anonymize IPs, DPA for 3rd parties.

---

## 6) Accessibility & UX

* Theme toggle (Light/Dark/High‑contrast) persisted per user/device.
* Focus order, skip‑to‑content, visible focus ring, ARIA roles.
* Labels for forms, error summaries, keyboard‑operable menus, proper table semantics in ag‑Grid.
* Automated a11y tests with `axe-core` (Cypress plugin) + manual audits.

---

## 7) Testing Strategy

* **Backend**: JUnit 5, Testcontainers (Postgres/Redis/MinIO), Spring MockMvc, WireMock for provider stubs, Pact for consumer/provider.
* **Frontend**: Jest/Karma + Testing Library, Cypress E2E (SSR paths too).
* **Security**: OWASP ZAP baseline in CI, dependency check (OWASP/DC), SAST (CodeQL).
* **Performance**: k6 load tests (cart/checkout/product page), Lighthouse CI for CWV.

**Definition of Done (per story)**

* Code + unit tests (≥80% critical paths), E2E covering happy path, docs updated, feature flags wired, observability (logs/metrics), review passed, a11y checks.

---

## 8) CI/CD & Environments

* **CI** (GitHub Actions or GitLab CI):

  * Build Java (Maven), run tests, integration w/ Testcontainers.
  * Build Angular, run unit + Cypress (component tests), Lighthouse CI.
  * SAST (CodeQL), Dependency scanning, OWASP ZAP baseline.
  * Build Docker images (backend, frontend, ai‑service) → push to registry.
  * Versioning via SemVer + Git tags; SBOM (CycloneDX).

* **CD**:

  * **Docker Compose** for local; **Ansible** playbooks for classic Linux VM (NGINX reverse proxy, systemd services, Postgres, Redis, MinIO, Matomo optional).
  * **AWS**: ECS or EKS; RDS Postgres; S3; CloudFront. **GCP**: GKE/Cloud Run; Cloud SQL; Cloud Storage; Cloud CDN.
  * IaC optional: Terraform modules per env.

* **Configs**: Helm charts or K8s manifests; feature flags from configmap or Unleash.

---

## 9) Backlog → Epics → Stories (with acceptance)

### EPIC A — Project Setup & Foundations

* A1: Repo mono (nx optional) or poly; commit hooks; code style (Prettier/ESLint, Spotless). **AC:** build passes; pre‑commit hooks enforce lint/test.
* A2: Spring Boot skeleton + health endpoints + Postgres + Liquibase. **AC:** `/actuator/health` green; migration runs.
* A3: Angular app with SSR, routing, Material theme (light/dark/high‑contrast). **AC:** toggle persists; SSR renders home.
* A4: Auth skeleton (JWT, OAuth2 Google/Apple). **AC:** login works; `/me` returns profile; roles enforced.

### EPIC B — Catalog & Product Detail

* B1: Categories tree API + UI. **AC:** browsing by category works.
* B2: Product listing with sort/filter/pagination. **AC:** sort by price/newest; query params reflected in URL.
* B3: Product detail page (gallery, description, materials, reviews). **AC:** SEO meta tags; structured data.
* B4: Related products (same category) logic + cache. **AC:** 4 items shown; fallback when <4.

### EPIC C — Cart & Checkout

* C1: Cart API + UI (anon + authenticated merge). **AC:** cart persists; concurrency safe.
* C2: Shipping quote service (table rates) + UI. **AC:** fee computed accurately across zones/weights.
* C3: Order creation + stock reservation. **AC:** idempotent; inventory decremented on payment capture.
* C4: Payments adapters (PayPal, Stripe). **AC:** sandbox payments succeed; webhooks verified; retries safe.

### EPIC D — Accounts & Orders

* D1: Profile management + addresses CRUD. **AC:** validations; default address; GDPR export.
* D2: Order history page. **AC:** pageable list; detail shows items, totals, shipment.

### EPIC E — Admin

* E1: Admin auth (ADMIN role) + shell. **AC:** protected routes.
* E2: Products grid (ag‑Grid) with inline edit + bulk import/export. **AC:** validations; optimistic UI; audit log.
* E3: Dashboard metrics (traffic, product views, sales). **AC:** charts show last 7/30/90 days; filters.
* E4: Feature flags toggles (payments/AI). **AC:** flags persist/env‑override.

### EPIC F — AI Description Service

* F1: Deploy local AI service (Ollama + chosen VLM). **AC:** `/describe` returns text for test images.
* F2: Admin UI integration w/ prompt templates. **AC:** drafts generated; editor saves to product.
* F3: Safety: block PII/toxicity; length limits. **AC:** validator trims/redacts.

### EPIC G — Observability, Security, A11y, Perf

* G1: OpenTelemetry + Prometheus + dashboards. **AC:** key SLIs visible.
* G2: ZAP baseline + CSP/HSTS headers. **AC:** no high findings.
* G3: Lighthouse ≥ 90 on key pages; CWV pass. **AC:** lab scores recorded in CI.
* G4: Axe a11y tests pass; manual audit. **AC:** WCAG 2.2 AA items documented.

---

## 10) Delivery Plan (increments)

* **Milestone 0 — Bootstrap (1–2 weeks)**: A1–A4 basics.
* **Milestone 1 — Browse & Product (2–3 weeks)**: B1–B3.
* **Milestone 2 — Cart/Checkout MVP (3–4 weeks)**: C1–C3 (payments behind flag).
* **Milestone 3 — Admin Core (2–3 weeks)**: E1–E2.
* **Milestone 4 — Payments & Analytics (2–3 weeks)**: C4 + E3.
* **Milestone 5 — AI Drafts & Polish (2–3 weeks)**: F1–F2 + G‑series.

(Adjust based on team size; above assumes 2–4 devs.)

---

## 11) Deployment Blueprints

* **Docker Compose (local):** backend, frontend (SSR), Postgres, Redis, MinIO, Matomo (optional), AI service, NGINX.
* **Ansible (classic VM):** roles for Java service (systemd), Angular SSR service, NGINX reverse proxy, Let’s Encrypt, Postgres, Redis; logrotate; backups (pg\_dump + S3/SSH).
* **AWS/GCP:** container images + IaC; RDS/Cloud SQL; S3/Cloud Storage; CDN front; secrets via SSM/Secret Manager.

---

## 12) Example Prompts (AI Description)

> “Analyze the uploaded product photo(s) and produce a concise, commercial product description (120–180 words) in Brazilian Portuguese. Emphasize benefits, materials, size, and use cases. Add 5 SEO keywords and a 160‑char meta description. Avoid claims you can’t infer from the image. Tone: helpful, trustworthy.”

Admin can edit prompt templates per locale.

---

## 13) Risks & Mitigations

* **Payments complexity:** Start sandbox only; strict webhooks + idempotency.
* **Image abuse:** Virus scan + MIME sniffing; size limits; EXIF strip.
* **AI hallucinations:** Human‑in‑the‑loop approval; provenance note; prompt & length guards.
* **Scaling search:** Start with Postgres FTS; move to OpenSearch if queries > p95.

---

## 14) Definition of Ready (story intake)

* User story, acceptance criteria, mock or wireframe, data model impact, feature flag plan, a11y notes, test plan, telemetry events defined.

---

## 15) Quick Start (dev)

* `docker compose up -d` (db, redis, minio, matomo, ai)
* Backend: `mvn -Pdev spring-boot:run`
* Frontend: `pnpm install && pnpm dev:ssr`
* Admin: login via seeded admin user; toggle features.

---

## 16) Postgres Schema Snippet (illustrative)

```sql
create table categories (
  id bigserial primary key,
  parent_id bigint references categories(id),
  name text not null,
  slug text unique not null
);

create table products (
  id bigserial primary key,
  category_id bigint not null references categories(id),
  sku text unique not null,
  name text not null,
  description text,
  materials text,
  is_active boolean default true,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table prices (
  id bigserial primary key,
  product_id bigint not null references products(id),
  currency char(3) not null,
  amount numeric(12,2) not null,
  valid_from timestamptz not null default now(),
  valid_to timestamptz
);
```

---

## 17) Open Questions (to refine later)

* Carriers & shipping zones specifics for Brazil/EU? (table setup)
* Exact payment providers by region (Stripe/MercadoPago/Adyen)?
* Admin roles granularity (catalog manager vs. analyst)?
* Review moderation workflow & anti‑spam (Akismet‑like)?
* SEO/i18n content strategy & sitemap cadence.
