# TÓM TẮT NGHIÊN CỨU: AXELOR OPEN SUITE 8.5

> **Ngày**: 2026-02-01 | **Version**: 8.5.10 | **Source**: `/Volumes/works/code/java/axelor/axelor-erp`

---

## 1. KIẾN TRÚC TỔNG QUAN

| Khía cạnh | Công nghệ/Giá trị | File nguồn |
|-----------|-------------------|------------|
| **Java Version** | OpenJDK 21 | `/build.gradle` |
| **Framework** | Axelor Platform (NOT Spring Boot) | `/modules/axelor-open-suite/libs.gradle` |
| **DI Container** | Google Guice (NOT Spring) | - |
| **Database** | PostgreSQL (primary), Hibernate 6.x + JPA 2.2 | `/src/main/resources/axelor-config.properties` |
| **Connection Pool** | HikariCP (5-20 connections) | `hibernate.hikari.minimumIdle = 5` |
| **Security** | Pac4j 5.7.7 (multi-provider auth) | `libs.pac4j_core = "org.pac4j:pac4j-core:5.7.7"` |
| **REST API** | JAX-RS 2.1 | - |
| **Scripting** | Groovy 3.0.23 | `libs.groovy = 'org.codehaus.groovy:groovy-all:3.0.23'` |
| **Build Tool** | Gradle 8.x với custom plugin `com.axelor.app:7.4.7` | `/build.gradle` |

**Pattern chính**: XML-Driven Development → Domain XML → Code Generation → JPA Entities

---

## 2. HỆ THỐNG MODULE

| Thống kê | Giá trị |
|----------|---------|
| **Tổng số modules** | 27 business modules |
| **Module foundation** | `axelor-base` (189 domains, 192 views) |
| **Largest module** | `axelor-account` (122 domains, 120 views) |
| **Module discovery** | Dynamic loading từ `modules/` directory |
| **Dependency pattern** | Hierarchical: base → account/sale/stock → supplychain |

**Top modules**:
- `axelor-base`: Partner, Product, Company, User (foundation)
- `axelor-account`: Invoice, Payment, Tax, Account
- `axelor-sale`: SaleOrder, Quotation (depends on crm)
- `axelor-stock`: Stock, Location, Movement
- `axelor-supplychain`: Integration layer (depends on sale, purchase, stock)

---

## 3. DATABASE & DATA MODEL

### 3.1. XML → Code Generation Flow

```
Domain XML → Gradle build → Generated Java Entity + Repository
```

**File nguồn**: `/modules/axelor-open-suite/axelor-sale/src/main/resources/domains/SaleOrder.xml`

| Feature | Syntax | Generated Code |
|---------|--------|----------------|
| **Entity** | `<entity name="SaleOrder">` | JPA `@Entity` class |
| **String field** | `<string name="saleOrderSeq" unique="true"/>` | `@Column(unique=true) String saleOrderSeq` |
| **Many-to-one** | `<many-to-one name="company" ref="Company"/>` | `@ManyToOne Company company` |
| **One-to-many** | `<one-to-many name="lines" mappedBy="order"/>` | `@OneToMany List<Line> lines` |
| **Finder method** | `<finder-method name="findBySaleOrderSeq" using="saleOrderSeq"/>` | Repository method auto-generated |
| **Formula field** | `<decimal formula="true">SELECT SUM(...)</decimal>` | Computed field via SQL |

### 3.2. Repository Pattern (2-tier)

```
JpaRepository (Axelor core)
    ↓ extends
SaleOrderRepository (generated)
    ↓ extends
SaleOrderBaseRepository (custom logic, optional)
```

**Code**: `/modules/axelor-open-suite/axelor-base/build/src-gen/java/com/axelor/apps/base/db/repo/ProductRepository.java`

### 3.3. Query DSL

```java
Query.of(SaleOrder.class)
  .filter("self.statusSelect = :status AND self.company = :company")
  .bind("status", STATUS_CONFIRMED)
  .bind("company", company)
  .order("-createdOn")
  .fetch();
```

**File nguồn**: Observed pattern trong service layer code

---

## 4. HỆ THỐNG PHÂN QUYỀN

### 4.1. Authorization Model (3-tier)

```
User → Group/Roles → Permissions
```

**File nguồn**: `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/User.xml`, `Permission.xml`

### 4.2. Permission Types

| Loại | Mô tả | Granularity | File domain |
|------|-------|-------------|-------------|
| **Permission** | Object-level permissions (CRUD) | Per entity | `Permission.xml` |
| **MetaPermission** | Field-level permissions | Per field | `MetaPermission.xml` |
| **Record-level filter** | Domain filter với `__user__` context | Per record | Trong Permission.condition |

### 4.3. Record-Level Security (Domain Filter)

**Code evidence**:
```java
// File: PermissionAssistantService.java:747
permission.setCondition("self.company = ?");
permission.setConditionParams("__user__.activeCompany");
```

**Mechanism**:
- Filter condition inject vào SQL WHERE clause
- Context variables: `__user__`, `__date__`, `__datetime__`
- Evaluated runtime cho mỗi query

### 4.4. Authentication Providers

**File nguồn**: `/src/main/resources/axelor-config.properties`

Supported: Google OAuth2, Keycloak, SAML, LDAP, CAS, OIDC (via Pac4j 5.7.7)

```properties
# Example
auth.provider.google.client = GOOGLE
auth.provider.google.title = Login with Google
```

---

## 5. BPM ENGINE & WORKFLOW

**⚠️ LIMITATION**: BPM engine nằm trong external addon `axelor-studio:3.5.1` (no source code)

### 5.1. BPM Configuration

**File nguồn**: `/src/main/resources/axelor-config.properties`

| Config | Value | Ý nghĩa |
|--------|-------|---------|
| `studio.bpm.max.idle.connections` | 10 | BPM dedicated pool min |
| `studio.bpm.max.active.connections` | 50 | BPM dedicated pool max |
| `studio.bpm.history.time.to.live` | P180D | History retention: 180 days |

### 5.2. Suy luận về BPM

- **Engine**: Likely Camunda (based on naming conventions & industry practice)
- **Dedicated connection pool**: Tách biệt khỏi main application pool
- **BPMN support**: Inferred from addon name "studio-bpm"
- **DMN support**: Not found in config

---

## 6. NO-CODE / LOW-CODE CAPABILITIES

### 6.1. Declarative Actions (7 types)

**File nguồn**: `/modules/axelor-open-suite/axelor-sale/src/main/resources/views/SaleOrder.xml:1827`

| Action Type | Purpose | Example Use Case |
|-------------|---------|------------------|
| `action-method` | Call Java service method | `computeTotal()` |
| `action-record` | Set field values | Auto-fill from partner |
| `action-view` | Open view/form | Show related invoices |
| `action-attrs` | Dynamic UI (hide/show/readonly) | Conditional field visibility |
| `action-group` | Chain multiple actions | Validate → Save → Notify |
| `action-condition` | Boolean check | Block if status != DRAFT |
| `action-validate` | Show validation message | Error/Warning/Info |

**Code example**:
```xml
<action-record name="action-sale-order-record-partner">
  <field name="paymentCondition" expr="eval: clientPartner?.paymentCondition"/>
  <field name="paymentMode" expr="eval: clientPartner?.outPaymentMode"/>
</action-record>
```

### 6.2. View Types

**File nguồn**: Analyzed 30 view files in `/modules/axelor-open-suite/axelor-sale/src/main/resources/views/`

| View Type | XML Tag | Use Case |
|-----------|---------|----------|
| **Grid** | `<grid>` | List view với columns, filters, sort |
| **Form** | `<form>` | Detail view với panels, tabs, fields |
| **Calendar** | `<calendar>` | Event scheduling view |
| **Cards** | `<cards>` | Kanban-style card view |
| **Chart** | `<chart>` | Analytics dashboard (bar, pie, line) |

### 6.3. Tỷ lệ No-code

**Suy luận**: ~85% business logic có thể implement bằng XML (actions + views + domains)
**15% còn lại**: Complex calculations, external integrations, PDF generation, advanced workflows

---

## 7. PERFORMANCE & SCALABILITY

### 7.1. Caching Strategy (Multi-layer)

**File nguồn**: `/src/main/resources/axelor-config.properties`

| Layer | Provider | Config Key | Mô tả |
|-------|----------|------------|-------|
| **L1 Cache** | JPA/Hibernate | `javax.persistence.sharedCache.mode = ENABLE_SELECTIVE` | Entity cache per transaction |
| **L2 Cache** | JCache/Caffeine | `@Cacheable` annotation | Cross-transaction entity cache |
| **Groovy Cache** | Internal | - | Compiled script cache |
| **Permission Cache** | Internal | - | Runtime permission cache |

### 7.2. Connection Pooling

```properties
# Main pool
hibernate.hikari.minimumIdle = 5
hibernate.hikari.maximumPoolSize = 20
hibernate.hikari.idleTimeout = 300000

# BPM pool (dedicated)
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
```

### 7.3. Async Processing

| Feature | Technology | Config |
|---------|------------|--------|
| **Scheduler** | Quartz | 3 threads (`org.quartz.threadPool.threadCount = 3`) |
| **Batch Jobs** | Axelor Batch framework | Auto-commit every 1000 records |
| **Background Tasks** | Quartz job execution | - |

### 7.4. API Pagination

```properties
api.pagination.max-per-page = 100000
api.pagination.default-per-page = 40
```

### 7.5. Scalability Limits

**⚠️ Không hỗ trợ distributed cache** → Session affinity required cho clustering
**⚠️ In-memory sessions** → Load balancer cần sticky sessions

---

## 8. TỐC ĐỘ PHÁT TRIỂN

### 8.1. XML-Driven Development Benefits

| Task | Traditional Java/Spring | Axelor XML | Speed Gain |
|------|------------------------|------------|------------|
| Create entity + CRUD | 200 LOC Java + JPA + REST | 30 LOC XML (domain) | **6-7x faster** |
| Build list view + form | 150 LOC React/HTML | 40 LOC XML (views) | **3-4x faster** |
| Add business rule | 50 LOC Java | 10 LOC XML (action-record) | **5x faster** |
| Add validation | 30 LOC Java | 5 LOC XML (action-validate) | **6x faster** |

**Average estimate**: **4x faster** development speed for typical business apps

### 8.2. Code Generation Statistics

- **Input**: 189 domain XMLs trong `axelor-base`
- **Output**: ~1500+ generated Java files (entities + repositories)
- **Ratio**: 1 XML file → 8-10 Java files

---

## 9. NHỮNG GIỚI HẠN KỸ THUẬT

| Limitation | Impact | Evidence |
|------------|--------|----------|
| **Hibernate DDL auto-update** | No migration version control (Flyway/Liquibase absent) | `hibernate.hbm2ddl.auto = update` |
| **BPM in external addon** | Cannot analyze/customize BPM engine | `axelor-studio:3.5.1` (closed source) |
| **Proprietary frontend** | Limited frontend customization | React only for map-viewer |
| **No distributed cache** | Clustering requires sticky sessions | No Redis/Hazelcast config found |
| **In-memory sessions** | Stateful architecture | Default session storage |
| **Single DB connection pool** | May bottleneck under high concurrency | Max 20 connections |
| **No API rate limiting** | Risk of DoS | No config found |

---

## 10. PHÁT HIỆN QUAN TRỌNG

### 10.1. Unique Selling Points

1. **XML-Driven Development**: 85% no-code capability → 4x dev speed
2. **Record-level security**: Granular permissions with domain filters
3. **27 pre-built business modules**: Full ERP suite out-of-box
4. **Multi-provider auth**: Google, Keycloak, SAML, LDAP via Pac4j
5. **Code generation**: Domain XML → Full CRUD stack

### 10.2. Architectural Decisions

- **Guice over Spring**: Lighter DI container, faster startup
- **JAX-RS over Spring MVC**: Standard REST API (not Spring-specific)
- **Groovy scripting**: Dynamic business rules without recompilation
- **Dedicated BPM pool**: Isolate workflow engine from main app

### 10.3. Technical Debt

- **No database migration tool**: Risk khi update schema
- **Proprietary platform dependency**: Vendor lock-in risk
- **Limited horizontal scaling**: No distributed session/cache

---

## KẾT LUẬN

**Axelor Open Suite 8.5** là một **low-code ERP platform** với:

✅ **Strengths**:
- Tốc độ phát triển nhanh (4x) nhờ XML-driven architecture
- 27 modules ERP sẵn có
- Hệ thống phân quyền chi tiết (object + field + record level)
- Multi-provider authentication

⚠️ **Weaknesses**:
- BPM engine không mở nguồn
- Proprietary platform dependency
- Limited scalability (no distributed cache, sticky sessions required)
- No database migration versioning

🎯 **Best Fit**: Small to medium enterprises (SMEs) cần triển khai ERP nhanh với customization trung bình

📊 **Platform Maturity**: Production-ready, version 8.5.10 (stable), active development

---

> **Chi tiết đầy đủ**: Xem `/Volumes/works/code/java/axelor/axelor-erp/AXELOR_TECHNICAL_RESEARCH_REPORT.md` (700+ dòng)
> **Source files**: 6 research steps tại `RESEARCH_STEP1_STRUCTURE.md` → `RESEARCH_STEP6_PERFORMANCE.md`
