Olá, Gabriel Leite 👋

I’ve dropped a full **production-oriented scaffold** for your Amazon-like app into the canvas: repo layout, Spring Boot 3.3 (Java 21) backend with Liquibase, OAuth2 (Google/Apple), Redis, S3-compatible media, Angular 18 + SSR + NgRx + Material + ag-Grid frontend, **Docker Compose** for local dev, **NGINX** reverse proxy, **Cypress/Jest** tests, **Testcontainers**, **OWASP scans**, and an **Ansible** playbook for a one-host VM deploy. It maps exactly to your Delivery Plan (1→6) and includes WCAG/a11y hooks and feature flags (payments/AI) out of the box.

### How to run locally (quick start)

```bash
# 1) Infra services
cp infra/.env.example infra/.env
cd infra && docker compose up -d db redis minio createbucket

# 2) Backend
cd ../backend
./gradlew bootRun   # Spring Boot on :8080

# 3) Frontend (SSR)
cd ../frontend
npm ci
npm run dev:ssr     # Angular SSR on :4200 via NGINX :80
```

### What you’ll find ready

* **Catalog API & Product details** (incl. `/related` 4 items)
* **Auth** (email/password stubs + Google/Apple wiring)
* **Feature toggles** (`payments`, `aiDescriptions`) in `application.yaml`
* **Admin panel skeleton** with ag-Grid CRUD + S3 uploads
* **AI service hook** (Ollama `llava`) for description from image
* **Security posture**: CSRF exception only for `/webhooks/**`, BCrypt, CORS, OWASP dep-check in CI, GDPR endpoints stubs
* **A11y** themes (light/dark/high-contrast) + `axe` checks in Cypress

### Suggested next commits

1. Implement shipping rules (flat-rate + weight/zone matrix).
2. Toggle **Stripe/PayPal**: enable in `shop.features.payments=true` and add webhook secret envs.
3. Fill out DTO validations + error handling (Problem Details).
4. Hook analytics events (views, sales) to a small event table + Redis counters.

If you want, I can also split this into **real files** (ZIP) or tailor it to your exact module naming (matches your Olga pattern) and wire your **PostgreSQL schema** specifics.
