# Monorepo Layout

```
amazonlike/
├─ README.md
├─ docker/
│  ├─ nginx.conf
│  └─ keycloak/ (optional if you self-host IdP later)
├─ deploy/
│  ├─ ansible/
│  │  ├─ inventory.ini
│  │  ├─ site.yml
│  │  ├─ roles/
│  │  │  ├─ docker
│  │  │  │  ├─ tasks/main.yml
│  │  │  ├─ app
│  │  │  │  ├─ tasks/main.yml
│  │  │  │  └─ templates/
│  │  │  │     ├─ docker-compose.yml.j2
│  │  │  │     └─ env.j2
│  └─ k8s/ (optional, for later)
├─ ci/
│  ├─ github-actions/
│  │  ├─ backend.yml
│  │  ├─ frontend.yml
│  │  └─ full-pipeline.yml
├─ infra/
│  ├─ docker-compose.yml
│  ├─ .env.example
│  ├─ initdb/
│  │  └─ 01-init.sql
│  └─ minio/
│     └─ policy.json
├─ backend/
│  ├─ build.gradle.kts
│  ├─ settings.gradle.kts
│  └─ src/
│     ├─ main/java/com/example/shop/
│     │  ├─ ShopApplication.java
│     │  ├─ config/
│     │  │  ├─ SecurityConfig.java
│     │  │  ├─ OAuth2ClientsConfig.java
│     │  │  ├─ CorsConfig.java
│     │  │  ├─ OpenApiConfig.java
│     │  │  ├─ WebhookSignatureVerifier.java
│     │  │  └─ FeatureFlags.java
│     │  ├─ domain/
│     │  │  ├─ Product.java
│     │  │  ├─ Category.java
│     │  │  ├─ UserAccount.java
│     │  │  ├─ Address.java
│     │  │  ├─ Order.java
│     │  │  ├─ OrderItem.java
│     │  │  └─ MediaAsset.java
│     │  ├─ dto/
│     │  │  ├─ ProductDto.java
│     │  │  ├─ CartDto.java
│     │  │  └─ CheckoutDto.java
│     │  ├─ repo/
│     │  │  ├─ ProductRepository.java
│     │  │  ├─ OrderRepository.java
│     │  │  └─ UserRepository.java
│     │  ├─ service/
│     │  │  ├─ CatalogService.java
│     │  │  ├─ CartService.java
│     │  │  ├─ CheckoutService.java
│     │  │  ├─ MediaService.java
│     │  │  ├─ AnalyticsService.java
│     │  │  └─ AiDescriptionService.java
│     │  ├─ web/
│     │  │  ├─ ProductController.java
│     │  │  ├─ CartController.java
│     │  │  ├─ CheckoutController.java
│     │  │  ├─ AccountController.java
│     │  │  └─ AdminController.java
│     │  └─ util/
│     │     └─ Pagination.java
│     ├─ main/resources/
│     │  ├─ application.yaml
│     │  └─ db/changelog/
│     │     ├─ master.xml
│     │     └─ changesets/
│     │        ├─ 001-schema.xml
│     │        └─ 002-indexes.xml
│     └─ test/java/com/example/shop/
│        ├─ CatalogServiceIT.java
│        ├─ CheckoutIT.java
│        └─ TestcontainersConfig.java
├─ frontend/
│  ├─ package.json
│  ├─ angular.json
│  ├─ tsconfig.json
│  └─ src/
│     ├─ main.server.ts
│     ├─ app/
│     │  ├─ app.config.ts
│     │  ├─ app.routes.ts
│     │  ├─ core/
│     │  │  ├─ interceptors/
│     │  │  ├─ services/
│     │  │  └─ store/
│     │  ├─ features/
│     │  │  ├─ catalog/
│     │  │  ├─ product/
│     │  │  ├─ cart/
│     │  │  ├─ account/
│     │  │  └─ admin/
│     │  ├─ shared/
│     │  │  ├─ ui/
│     │  │  └─ models/
│     │  └─ accessibility/
│     ├─ theme/
│     │  ├─ light.scss
│     │  ├─ dark.scss
│     │  └─ high-contrast.scss
│     └─ cypress/
│        ├─ e2e/
│        │  └─ catalog.cy.ts
│        └─ support/
└─ ai/
   ├─ ollama/README.md
   └─ server/
      ├─ pyproject.toml
      └─ main.py
```

---

## Backend — Spring Boot 3.3 (Java 21)

### `build.gradle.kts`
```kotlin
plugins {
  id("java")
  id("org.springframework.boot") version "3.3.2"
  id("io.spring.dependency-management") version "1.1.5"
}
java { toolchain { languageVersion.set(JavaLanguageVersion.of(21)) } }
repositories { mavenCentral() }
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-web")
  implementation("org.springframework.boot:spring-boot-starter-security")
  implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
  implementation("org.springframework.boot:spring-boot-starter-data-jpa")
  implementation("org.springframework.boot:spring-boot-starter-validation")
  implementation("org.springframework.boot:spring-boot-starter-actuator")
  implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")
  implementation("org.springframework.data:spring-data-redis")
  implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
  runtimeOnly("org.postgresql:postgresql")
  implementation("org.liquibase:liquibase-core")
  implementation("software.amazon.awssdk:s3:2.25.60")
  testImplementation("org.springframework.boot:spring-boot-starter-test")
  testImplementation("org.testcontainers:junit-jupiter")
  testImplementation("org.testcontainers:postgresql")
}
```

### `application.yaml`
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/shop
    username: shop
    password: shop
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate.jdbc.time_zone: UTC
  liquibase:
    change-log: classpath:db/changelog/master.xml
  redis:
    host: localhost
    port: 6379
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
server:
  port: 8080
shop:
  media:
    s3:
      endpoint: http://localhost:9000
      bucket: media
      region: us-east-1
      accessKey: minioadmin
      secretKey: minioadmin
  features:
    payments: false
    aiDescriptions: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

### Entities (excerpt)
```java
@Entity
@Table(name = "product")
public class Product {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String sku;
  private String name;
  @Column(columnDefinition = "text")
  private String description;
  private BigDecimal price;
  private String currency;
  private Integer stock;
  @ManyToOne(fetch = FetchType.LAZY)
  private Category category;
  @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<MediaAsset> media = new ArrayList<>();
  // getters/setters
}
```

### SecurityConfig (email/password + Google/Apple)
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
  @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
      .csrf(csrf -> csrf.ignoringRequestMatchers("/webhooks/**"))
      .cors(Customizer.withDefaults())
      .authorizeHttpRequests(auth -> auth
        .requestMatchers("/actuator/**","/v3/api-docs/**","/swagger-ui/**","/api/catalog/**").permitAll()
        .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
        .requestMatchers("/api/admin/**").hasRole("ADMIN")
        .anyRequest().authenticated())
      .oauth2Login(Customizer.withDefaults())
      .oauth2ResourceServer(oauth2 -> oauth2.jwt());
    return http.build();
  }
}
```

### OAuth2 Clients
```java
@Configuration
public class OAuth2ClientsConfig {
  @Bean
  public ClientRegistrationRepository clients() {
    var google = CommonOAuth2Provider.GOOGLE.getBuilder("google")
      .clientId(System.getenv("GOOGLE_CLIENT_ID"))
      .clientSecret(System.getenv("GOOGLE_CLIENT_SECRET")).build();
    var apple = ClientRegistration.withRegistrationId("apple")
      .clientId(System.getenv("APPLE_CLIENT_ID"))
      .clientSecret(System.getenv("APPLE_CLIENT_SECRET"))
      .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
      .redirectUri("{baseUrl}/login/oauth2/code/apple")
      .authorizationUri("https://appleid.apple.com/auth/authorize")
      .tokenUri("https://appleid.apple.com/auth/token")
      .jwkSetUri("https://appleid.apple.com/auth/keys")
      .scope("name","email").build();
    return new InMemoryClientRegistrationRepository(google, apple);
  }
}
```

### Controllers (excerpt)
```java
@RestController
@RequestMapping("/api/catalog")
public class ProductController {
  private final CatalogService service;
  public ProductController(CatalogService service) { this.service = service; }

  @GetMapping("/products")
  public Page<ProductDto> list(@RequestParam Optional<Long> categoryId,
                               @RequestParam(defaultValue = "0") int page,
                               @RequestParam(defaultValue = "12") int size,
                               @RequestParam(defaultValue = "newest") String sort,
                               @RequestParam Optional<String> q) {
    return service.list(categoryId, page, size, sort, q);
  }

  @GetMapping("/products/{id}")
  public ProductDto get(@PathVariable Long id) { return service.get(id); }

  @GetMapping("/products/{id}/related")
  public List<ProductDto> related(@PathVariable Long id) { return service.related(id, 4); }
}
```

### AI Description Service (Ollama via HTTP)
```java
@Service
public class AiDescriptionService {
  private final WebClient client = WebClient.builder().baseUrl("http://localhost:11434").build();
  @Value("${shop.features.aiDescriptions:true}") boolean enabled;
  public String describe(byte[] imageBytes) {
    if (!enabled) return "";
    return client.post()
      .uri("/api/generate")
      .contentType(MediaType.APPLICATION_JSON)
      .bodyValue(Map.of(
        "model", "llava:latest",
        "prompt", "Write a concise, SEO-friendly product description.",
        "images", List.of(Base64.getEncoder().encodeToString(imageBytes))))
      .retrieve().bodyToMono(Map.class)
      .map(m -> (String)m.getOrDefault("response",""))
      .block(Duration.ofSeconds(30));
  }
}
```

### Liquibase — `master.xml`
```xml
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.1.xsd">
  <include file="db/changelog/changesets/001-schema.xml"/>
  <include file="db/changelog/changesets/002-indexes.xml"/>
</databaseChangeLog>
```

### Liquibase — schema (excerpt)
```xml
<changeSet id="001" author="gabriel">
  <createTable tableName="category">
    <column name="id" type="BIGSERIAL" autoIncrement="true" primaryKey="true"/>
    <column name="name" type="VARCHAR(128)"/>
  </createTable>
  <createTable tableName="product">
    <column name="id" type="BIGSERIAL" autoIncrement="true" primaryKey="true"/>
    <column name="sku" type="VARCHAR(64)"/>
    <column name="name" type="VARCHAR(255)"/>
    <column name="description" type="TEXT"/>
    <column name="price" type="NUMERIC(12,2)"/>
    <column name="currency" type="VARCHAR(3)"/>
    <column name="stock" type="INT"/>
    <column name="category_id" type="BIGINT"/>
  </createTable>
  <addForeignKeyConstraint baseTableName="product" baseColumnNames="category_id"
    referencedTableName="category" referencedColumnNames="id"/>
  <createTable tableName="user_account">
    <column name="id" type="BIGSERIAL" primaryKey="true"/>
    <column name="email" type="VARCHAR(255)"/>
    <column name="password_hash" type="VARCHAR(255)"/>
    <column name="provider" type="VARCHAR(32)"/>
  </createTable>
  <createTable tableName="order_header">
    <column name="id" type="BIGSERIAL" primaryKey="true"/>
    <column name="user_id" type="BIGINT"/>
    <column name="total" type="NUMERIC(12,2)"/>
    <column name="status" type="VARCHAR(32)"/>
    <column name="created_at" type="TIMESTAMP"/>
  </createTable>
  <createTable tableName="order_item">
    <column name="id" type="BIGSERIAL" primaryKey="true"/>
    <column name="order_id" type="BIGINT"/>
    <column name="product_id" type="BIGINT"/>
    <column name="quantity" type="INT"/>
    <column name="unit_price" type="NUMERIC(12,2)"/>
  </createTable>
</changeSet>
```

---

## Frontend — Angular 18 + SSR + NgRx + Material + ag‑Grid

### Commands
```bash
# inside /frontend
npm create @angular@latest . -- --ssr --routing --style=scss
npm i @ngrx/store @ngrx/effects @ngrx/entity @ngrx/store-devtools ag-grid-community ag-grid-angular @angular/material @angular/cdk
npm i -D cypress @types/node jest @types/jest jest-preset-angular axe-core cypress-axe
```

### Global store slices (simplified)
```ts
// store/catalog.actions.ts
export const loadProducts = createAction('[Catalog] Load', props<{categoryId?: number, page?: number, sort?: string, q?: string}>());
export const loadProductsSuccess = createAction('[Catalog] Load Success', props<{page: Page<Product>}>());
```

### Accessibility themes
```scss
/* theme/high-contrast.scss */
$theme: mat.define-theme((color: (primary: mat.$cyan-palette, theme-type: high-contrast)));
@include mat.all-component-themes($theme);
```

### Product page (related items)
```ts
// features/product/product.component.ts
ngOnInit(){
  this.route.paramMap.pipe(
    switchMap(params => this.api.getProduct(+params.get('id')!))
  ).subscribe(p => { this.product = p; this.api.getRelated(p.id).subscribe(r => this.related = r); });
}
```

### Admin grid (ag‑Grid)
```ts
// features/admin/products-grid.component.ts
columnDefs = [
 {field:'id', sortable:true, filter:true},
 {field:'name', editable:true},
 {field:'price', editable:true},
 {field:'stock', editable:true},
 {field:'category', valueGetter: p=>p.data.category?.name}
];
```

### A11y tests in Cypress
```ts
// cypress/e2e/a11y.cy.ts
it('home is WCAG-friendly', () => {
  cy.visit('/');
  cy.injectAxe();
  cy.checkA11y();
});
```

---

## Infra — Docker Compose (local dev)

### `infra/.env.example`
```
POSTGRES_PASSWORD=shop
POSTGRES_USER=shop
POSTGRES_DB=shop
REDIS_PASSWORD=
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
S3_BUCKET=media
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
APPLE_CLIENT_ID=
APPLE_CLIENT_SECRET=
STRIPE_SECRET=
PAYPAL_SECRET=
FEATURE_PAYMENTS=false
FEATURE_AI=true
```

### `infra/docker-compose.yml`
```yaml
version: '3.9'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./initdb:/docker-entrypoint-initdb.d
  redis:
    image: redis:7
    ports: ["6379:6379"]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
    ports: ["9000:9000","9001:9001"]
    volumes:
      - minio:/data
  createbucket:
    image: minio/mc
    depends_on: [minio]
    entrypoint: >
      /bin/sh -c "mc alias set local http://minio:9000 $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD;
      mc mb -p local/$S3_BUCKET || true; mc anonymous set download local/$S3_BUCKET;"
  backend:
    build: ../backend
    environment:
      SPRING_PROFILES_ACTIVE: dev
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      APPLE_CLIENT_ID: ${APPLE_CLIENT_ID}
      APPLE_CLIENT_SECRET: ${APPLE_CLIENT_SECRET}
    ports: ["8080:8080"]
    depends_on: [db, redis, minio]
  frontend:
    build: ../frontend
    ports: ["4200:4200"]
    environment:
      API_URL: http://localhost:8080
    depends_on: [backend]
  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    volumes:
      - ../docker/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on: [frontend]
volumes:
  pgdata: {}
  minio: {}
```

### `docker/nginx.conf`
```nginx
worker_processes auto;
events { worker_connections 1024; }
http {
  include       /etc/nginx/mime.types;
  server {
    listen 80;
    location /api/ { proxy_pass http://backend:8080/api/; }
    location / { proxy_pass http://frontend:4200/; }
  }
}
```

---

## CI/CD — GitHub Actions (build, test, scan, a11y)

### `ci/github-actions/full-pipeline.yml`
```yaml
name: CI-CD
on: [push, pull_request]
jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_DB: shop, POSTGRES_USER: shop, POSTGRES_PASSWORD: shop }
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: 'temurin', java-version: '21' }
      - name: Build & Test
        run: |
          cd backend
          ./gradlew test
      - name: Security Scan (OWASP Dependency-Check)
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'backend'
          path: 'backend'
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: |
          cd frontend
          npm ci
          npm run test:ci
          npm run build:ssr
      - name: A11y (Cypress + axe)
        run: |
          cd frontend
          npx cypress run
  docker:
    needs: [backend, frontend]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build images
        run: |
          docker build -t ghcr.io/you/shop-backend:$(git rev-parse --short HEAD) backend
          docker build -t ghcr.io/you/shop-frontend:$(git rev-parse --short HEAD) frontend
```

---

## Ansible — one‑host Linux VM deploy

### `deploy/ansible/inventory.ini`
```
[web]
shop-vm ansible_host=YOUR_IP ansible_user=ubuntu
```

### `deploy/ansible/site.yml`
```yaml
- hosts: web
  become: yes
  vars:
    docker_compose_path: /opt/shop/docker-compose.yml
  roles:
    - docker
    - app
```

### `deploy/ansible/roles/docker/tasks/main.yml`
```yaml
- name: Install Docker
  apt:
    name: [ docker.io, docker-compose-plugin ]
    state: present
    update_cache: yes
- name: Ensure docker service
  service: { name: docker, state: started, enabled: yes }
```

### `deploy/ansible/roles/app/tasks/main.yml`
```yaml
- name: Create app directory
  file: { path: /opt/shop, state: directory }
- name: Render env file
  template: { src: env.j2, dest: /opt/shop/.env, mode: '0600' }
- name: Render compose
  template: { src: docker-compose.yml.j2, dest: "{{ docker_compose_path }}" }
- name: Pull & up
  community.docker.docker_compose_v2:
    project_src: /opt/shop
    state: present
```

---

## Security Notes
- CSRF disabled only for `/webhooks/**`.
- Use `@Validated` DTOs, sanitize HTML inputs, set strong `Content-Security-Policy` in NGINX.
- Passwords hashed with BCrypt. Enforce HTTPS in prod.
- Verify Stripe/PayPal webhooks with HMAC signature and idempotency keys.
- GDPR: data export endpoint (`/api/account/export`), data deletion, consent banner.

---

## Testing Snippets

### Testcontainers bootstrap
```java
@Testcontainers
@SpringBootTest
public class CatalogServiceIT {
  @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");
  @DynamicPropertySource static void dbProps(DynamicPropertyRegistry r){
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
  }
  @Test void listsProducts(){ /* ... */ }
}
```

### Cypress catalog test
```ts
describe('Catalog', () => {
  it('sorts by newest', () => {
    cy.visit('/catalog');
    cy.get('[data-cy=sort-select]').select('newest');
    cy.get('[data-cy=product-card]').should('have.length.at.least', 1);
  });
});
```

---

## How to Run (dev)
```bash
cp infra/.env.example infra/.env
cd infra && docker compose up -d db redis minio createbucket
# backend
cd ../backend && ./gradlew bootRun
# frontend (SSR dev server)
cd ../frontend && npm ci && npm run dev:ssr
# browse http://localhost
```

---

## Next Steps (Delivery Plan mapping)
1) **Setup & Auth**: Wire OAuth2, email/password, Redis sessions, CORS, SSR.
2) **Catalog & Product Pages**: REST endpoints + Angular views, search/sort, related items.
3) **Cart & Checkout**: Persistent cart (Redis), shipping calc (rule-based stub), order creation.
4) **Admin Panel**: ag‑Grid CRUD, S3 uploads, analytics dashboards.
5) **Payments & Analytics**: Stripe/PayPal toggle via `shop.features.payments`.
6) **AI Descriptions**: Ollama llava endpoint + image upload → description autofill.

> This scaffold is intentionally minimal but production-oriented. Expand DTOs, validations, and error handling as you iterate.

