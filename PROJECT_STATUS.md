# AXELOR OPEN SUITE - PROJECT STATUS & CONTEXT

**Last Updated:** 2026-02-03 (cập nhật lần 5 — Phase 4 HOÀN THÀNH ✅)
**Project:** Nghiên cứu và phân tích Axelor Open Suite ERP (Java-based)
**Version:** 8.5.10
**Repository:** axelor-erp (local analysis)

---

## 🔥 CÔNG VIỆC HIỆN TẠI — PHASE 4: SUPPLEMENTAL RESEARCH (HOÀN THÀNH ✅)

### Mô tả
Bổ sung các chủ đề chuyên sâu chưa được cover đầy đủ trong STEP 1-6:
- **STEP 7:** DMN Engine (Decision Model and Notation) - Phân tích engine quyết định nghiệp vụ
- **STEP 8:** Traditional Java Development - Patterns phát triển code truyền thống, kết hợp Studio, extension strategies

### Tiến độ Phase 4

| # | File | Dòng | Trạng thái | Ghi chú |
|---|------|------|-----------|---------|
| 7 | RESEARCH_STEP7_DMN.md | ~1,400 | ✅ Hoàn thành | DMN architecture, Camunda 7.23.0, hit policies, FEEL expressions |
| 8 | RESEARCH_STEP8_DEVELOPMENT.md | 2,730 | ✅ Hoàn thành | Service layers, Studio integration, Extension patterns, Execution order |

### ✅ Phase 4 HOÀN THÀNH — 2 files research chuyên sâu

**Kết quả:**
- ✅ STEP 7: DMN Engine analysis hoàn chỉnh (8 sections, 1,400+ dòng)
- ✅ STEP 8: Traditional development + Extensions (Sections A1-A8, B1-B7, 2,730 dòng)
- ✅ Evidence-based findings từ source code thực tế
- ✅ Best practices, workflows, troubleshooting guides
- ✅ Tiếng Việt thuần túy theo chuẩn Phase 3

---

## ✅ COMPLETED TASKS

### Phase 1: Initial Research (Completed 2026-01)
Đã hoàn thành nghiên cứu sơ bộ và tạo 6 files research ban đầu:
- ✅ RESEARCH_STEP1_STRUCTURE.md (766 lines - version cũ)
- ✅ RESEARCH_STEP2_DATABASE.md (626 lines - version cũ)
- ✅ RESEARCH_STEP3_SECURITY.md (943 lines - version cũ)
- ✅ RESEARCH_STEP4_BPM.md (789 lines - version cũ)
- ✅ RESEARCH_STEP5_NOCODE.md (original version)
- ✅ RESEARCH_STEP6_PERFORMANCE.md (1,609 lines - version cũ)

### Phase 2: Detailed Analysis Rewrite (✅ COMPLETED 2026-02-02)

Đã viết lại TOÀN BỘ 6 files với phong cách mới chi tiết hơn.

### Phase 3: Vietnamese Pure Rewrite (✅ HOÀN THÀNH — 6/6)

Đã viết lại TOÀN BỘ từ Vinglish sang tiếng Việt thuần. Tất cả 6 files đã hoàn thành:

#### ✅ STEP 1: STRUCTURE - 859 dòng
**File:** RESEARCH_STEP1_STRUCTURE.md
**Nội dung:**
- Multi-module architecture (13+ modules analyzed)
- Framework layers (presentation, business, data access)
- Module dependencies và coupling analysis
- Configuration management (axelor-config.properties)
- Build system (Gradle multi-project)

**Key Findings:**
- Modular monolith architecture
- Clean separation of concerns
- Strong convention-over-configuration

#### ✅ STEP 2: DATABASE - 1,414 dòng (+788 từ gốc)
**File:** RESEARCH_STEP2_DATABASE.md
**Nội dung:**
- JPA/Hibernate entity mapping
- Domain-driven design patterns
- 16 sections covering: entity types, relationships, constraints, audit tracking
- Repository pattern implementation
- Query DSL analysis

**Key Findings:**
- Code generation từ XML domain models
- AuditableModel for automatic tracking
- JSON fields for flexible data
- Hibernate DDL auto-update strategy

#### ✅ STEP 3: SECURITY - 1,305 dòng (+362 từ gốc)
**File:** RESEARCH_STEP3_SECURITY.md
**Nội dung:**
- Pac4j multi-provider authentication
- User → Group → Role → Permissions hierarchy
- 18 sections covering object/field/record-level security
- CSV-based permission management
- LDAP integration

**Key Findings:**
- Flexible authentication (LDAP, OAuth2, SAML, CAS)
- Fine-grained authorization (3 levels)
- Permission caching for performance
- Groovy-based dynamic rules

#### ✅ STEP 4: BPM - 1,113 dòng (+324 từ gốc)
**File:** RESEARCH_STEP4_BPM.md
**Nội dung:**
- External BPM addon architecture
- Camunda engine identification (~75% confidence)
- 11 sections analyzing workflow architecture
- Dual workflow strategy (BPM + action chains)
- Groovy scripting trong process definitions

**Key Findings:**
- BPM là external addon (axelor-studio dependency)
- Separate database connection pool (50 connections)
- BPMN 2.0 compliant
- No-code workflow builders

#### ✅ STEP 5: NO-CODE - 1,294 dòng (Hoàn toàn mới)
**File:** RESEARCH_STEP5_NOCODE.md
**Nội dung:**
- XML-driven development paradigm (MDD)
- 5 view types: Grid, Form, Calendar, Cards, Chart
- 7 action types: method, record, view, attrs, group, condition, validate, script
- View inheritance mechanism (XPath-based)
- Code generation system
- Expression language (Groovy)
- Widget system (20+ widgets)
- I18N support
- Template system (email, reports)

**Key Findings:**
- 80-90% declarative (XML) vs 10-20% imperative (Java)
- Comparable to Odoo's no-code capabilities
- Lower-code platform for developers, not citizen developers
- 60-70% application logic achievable without Java

#### ✅ STEP 6: PERFORMANCE - 2,031 dòng (+421 từ gốc)
**File:** RESEARCH_STEP6_PERFORMANCE.md
**Nội dung:**
- Multi-layer caching (L1, L2, Groovy script, permission)
- HikariCP connection pooling (best-in-class)
- REST API architecture (JAX-RS)
- Async processing (Quartz + Batch framework)
- Scalability patterns (horizontal/vertical)
- Monitoring & observability
- Performance optimization roadmap
- Anti-patterns to avoid
- Production deployment checklist

**Key Findings:**
- L2 cache disabled by default (needs enabling)
- Max pagination too high (100K → should be 5K)
- JDBC batching disabled (should enable)
- Good defaults for development, needs tuning for production
- Scalable from 50 users (single server) to 1000+ users (3-node cluster)

### Phase 4: Supplemental Research (✅ HOÀN THÀNH — 2/2)

Bổ sung nghiên cứu chuyên sâu các chủ đề bổ trợ:

#### ✅ STEP 7: DMN ENGINE - 1,400+ dòng (MỚI)
**File:** RESEARCH_STEP7_DMN.md
**Nội dung:**
- DMN architecture và engine identification
- Camunda DMN Engine 7.23.0 (DMN 1.3 compliant)
- Data structures: WkfDmnModel, DmnTable, DmnField
- All hit policies: UNIQUE, FIRST, PRIORITY, ANY, RULE ORDER, OUTPUT ORDER, COLLECT (SUM/MIN/MAX/COUNT)
- FEEL expression language (JUEL + Scala implementations)
- DMN-BPMN integration patterns
- Configuration: connection pool, history TTL
- Limitations và gaps analysis

**Key Findings:**
- DMN là part of axelor-studio addon (commercial, version 3.5.1)
- Uses Camunda DMN 7.23.0 engine
- Separate database connection pool (50 max)
- History TTL = P180D (6 months)
- FEEL expressions với both JUEL (lightweight) và Scala (full spec) implementations
- No built-in testing framework for DMN tables
- No version control integration for DMN models

**Evidence Sources:**
- WkfDmnModel.xml, DmnTable.xml, DmnField.xml entity definitions
- Gradle cache dependencies (camunda-dmn-*)
- axelor-config.properties (studio.bpm.* settings)
- No direct source code (binary addon)

#### ✅ STEP 8: TRADITIONAL JAVA DEVELOPMENT - 2,730 dòng (MỚI)
**File:** RESEARCH_STEP8_DEVELOPMENT.md
**Nội dung:**

**Part A: Traditional Java Development (A1-A8):**
- A1: Custom module structure và organization
- A2: Service layer patterns (Google Guice DI, Interface+Implementation)
- A3: Repository pattern (Generated + Custom Management Repositories)
- A4: Controller layer (ActionRequest/ActionResponse patterns)
- A5: Domain models (XML → Java code generation)
- A6: View system (Grid/Form XML definitions)
- A7: Action system (7 action types analysis)
- A8: Testing strategies (service tests, integration tests)

**Part B: Studio Integration & Extensions (B1-B7):**
- B1: Studio model architecture (MetaJsonModel, meta_json_record table)
- B2: Studio vs Code boundaries decision matrix
- B3: Version control strategies khi combine Code + Studio
- B4: Action system khi mix Code và Studio
- B5: Real patterns from axelor-sale module (80+ Services analyzed)
- **B6: Execution order management** — Best practices quản lý thứ tự thực thi logic
  - 5 execution layers: Repository → Service → Controller → Action Chain → BPM → Event System
  - Transaction boundaries và propagation rules
  - Real-world scenario với timeline visualization
  - Anti-patterns và debugging strategies
- **B7: Extension & Custom Module Development** — Quy trình kế thừa và extend modules
  - Custom module structure chuẩn
  - Entity extension patterns (domain XML)
  - Service override patterns (Guice bindings)
  - View extension với XPath selectors
  - Menu extension và organization
  - 10-step workflow từ identify → implement → test → deploy
  - Troubleshooting common issues
  - Upgrade strategies

**Key Findings:**
- Uses Google Guice (NOT Spring!) for dependency injection
- Repository pattern: Generated base + Custom Management repositories
- Controllers use Beans.get() service locator pattern (anti-pattern nhưng common)
- Studio models stored in database (dynamic), Code models stored as XML (static)
- Studio limitations: no M2M relationships, no inheritance, no complex constraints
- Version control pain point: no built-in export/import cho Studio configs
- Extension > Override > Replacement strategy cho maintainability
- Execution order: Repository hooks run BEFORE Service logic trong same transaction
- Action chains create MULTIPLE transactions (each action-method = separate transaction)
- Event observers synchronous, NO guaranteed order

**Evidence Sources:**
- axelor-sale module: 80+ Service classes, 15+ Controllers analyzed
- SaleOrderManagementRepository.java (Repository hooks pattern)
- SaleModule.java (Guice Module bindings: 150+ bind() statements)
- SaleOrderController.java (ActionRequest/ActionResponse patterns)
- BankPaymentModule.java (Service override bindings)
- MoveLine.xml (Entity extension example)
- BankDetails.xml (View extension với XPath)
- MetaJsonModel, MetaJsonField, MetaJsonRecord entities (Studio architecture)

---

## 📊 STATISTICS

### Code Analysis Coverage
- **Total lines documented:** ~11,800+ dòng (8 files)
- **Source files examined:** 100+ files
- **Modules analyzed in-depth:**
  - axelor-sale (80+ Services, Repository patterns)
  - axelor-account (Security, Invoice processing)
  - axelor-bank-payment (Extension patterns)
  - axelor-base (Core entities, User management)
  - axelor-studio (DMN, BPM, MetaJson models)

### File Growth

| File | Phase | Lines | Notes |
|------|-------|-------|-------|
| STEP1 | 3 | 859 | Architecture overview |
| STEP2 | 3 | 1,414 | ORM & database |
| STEP3 | 3 | 1,305 | Security & auth |
| STEP4 | 3 | 1,113 | BPM engine |
| STEP5 | 3 | 1,294 | No-code capabilities |
| STEP6 | 3 | 2,031 | Performance & scaling |
| **STEP7** | **4** | **~1,400** | **DMN engine (NEW)** |
| **STEP8** | **4** | **2,730** | **Java development (NEW)** |
| **TOTAL** | — | **~12,146** | **8 comprehensive files** |

### Writing Style Evolution
**Phase 1 → 2:** Bullet points → Detailed explanations
**Phase 2 → 3:** Vinglish → Pure Vietnamese prose
**Phase 3 → 4:** Maintained Vietnamese prose + Added 2 new specialized topics

### Research Depth
- **Entity definitions analyzed:** 50+ domain XMLs
- **View definitions examined:** 30+ view XMLs
- **Service classes studied:** 100+ Java files
- **Configuration keys documented:** 200+ properties
- **Patterns identified:** 40+ architectural patterns
- **Best practices documented:** 60+ recommendations

---

## 🎯 KEY INSIGHTS & FINDINGS

### Architecture Strengths
1. **Modular Design:** Clean separation, loosely coupled modules
2. **Framework Maturity:** Industry-standard patterns (JPA, JAX-RS, Pac4j, Quartz, Camunda)
3. **Extensibility:** Module system + view inheritance + service override enables customization
4. **No-Code Capabilities:** 60-70% application logic declarative
5. **Decision Automation:** DMN engine cho business rules (declarative decision tables)
6. **Dependency Injection:** Google Guice lightweight và explicit bindings

### Architecture Weaknesses
1. **Complexity:** High learning curve (XML schemas, action types, domain DSL, Guice)
2. **BPM/DMN Addons:** External dependencies, binary only, no source access
3. **Performance Defaults:** L2 cache disabled, JDBC batching off, max pagination too high
4. **Documentation:** Limited official docs, requires source code analysis
5. **Studio Limitations:** No M2M, no inheritance, no version control integration
6. **Testing Infrastructure:** Very few test examples, unclear testing strategies
7. **Service Locator Anti-Pattern:** `Beans.get()` usage trong Controllers hides dependencies

### Comparison với Odoo
| Aspect | Axelor | Odoo |
|--------|--------|------|
| **Language** | Java + Groovy | Python |
| **ORM** | JPA/Hibernate | Odoo ORM (custom) |
| **View System** | XML (similar to Odoo) | XML |
| **BPM** | External Camunda | Built-in workflows |
| **DMN** | External Camunda DMN | No native DMN |
| **DI Framework** | Google Guice | Python decorators |
| **Code Gen** | Domain XML → Java | Python direct |
| **Frontend** | React | Owl (custom) |
| **Community** | Smaller | Larger |
| **Target** | Enterprise (Java shops) | SMBs |
| **Extension Model** | Module + Guice bindings | Inheritance + monkey patching |

**Verdict:** Axelor = "Java equivalent of Odoo" với stronger type safety, DMN support, nhưng smaller ecosystem và steeper learning curve.

---

## 📁 PROJECT FILE STRUCTURE

```
axelor-erp/
├── RESEARCH_STEP1_STRUCTURE.md      (859 lines) - Architecture overview
├── RESEARCH_STEP2_DATABASE.md       (1,414 lines) - ORM & data layer
├── RESEARCH_STEP3_SECURITY.md       (1,305 lines) - Auth & authorization
├── RESEARCH_STEP4_BPM.md            (1,113 lines) - Workflow engine
├── RESEARCH_STEP5_NOCODE.md         (1,294 lines) - No-code capabilities
├── RESEARCH_STEP6_PERFORMANCE.md    (2,031 lines) - Performance & scaling
├── RESEARCH_STEP7_DMN.md            (~1,400 lines) - DMN decision engine
├── RESEARCH_STEP8_DEVELOPMENT.md    (2,730 lines) - Java development & extensions
├── PROJECT_STATUS.md                (this file) - Project tracking
│
├── src/main/resources/
│   ├── axelor-config.properties     (618 lines) - Main configuration
│   └── domains/                     - Entity definitions
│
├── modules/axelor-open-suite/
│   ├── axelor-base/                 - Core module
│   ├── axelor-sale/                 - Sales module (80+ Services analyzed)
│   ├── axelor-account/              - Accounting module
│   ├── axelor-bank-payment/         - Extension example
│   └── [10+ other modules]
│
└── build.gradle                     - Multi-project build
```

---

## 🧠 RESEARCH METHODOLOGY — HỆ THỐNG PHƯƠNG PHÁP NGHIÊN CỨU

### Nguyên tắc chung

Khi nghiên cứu Axelor Open Suite, tuân thủ các nguyên tắc sau:

#### 1. Evidence-Based Analysis
- **LUÔN trích dẫn source code** với đường dẫn file và line numbers
- **Đánh dấu rõ ràng:** `[Từ source code]` vs `⚠️ **Suy luận:**`
- **Không đoán mò** — Nếu không tìm thấy evidence, ghi rõ "Không tìm thấy trong source code"
- **Prefer code > docs** — Source code là nguồn chân lý, documentation có thể outdated

#### 2. Structured Documentation
- **Mỗi section có header rõ ràng** với nguồn evidence
- **Code snippets phải có context** — Giải thích trước và sau code block
- **Tổ chức theo layers** — Từ high-level architecture xuống implementation details
- **Cross-reference** — Link giữa các sections liên quan

#### 3. Vietnamese Writing Standards
- **Thuật ngữ:** Tiếng Việt trước, tiếng Anh trong ngoặc lần đầu xuất hiện
  - Ví dụ: "Bộ khung (framework)" → lần sau chỉ viết "bộ khung"
- **Câu văn tự nhiên:** 3-5 câu tối thiểu cho mỗi concept, không viết bullet points khô khan
- **Giải thích "tại sao"** — Không chỉ liệt kê "cái gì", mà explain "tại sao thiết kế như vậy"
- **Contextual examples** — Đưa ví dụ thực tế từ source code

#### 4. Analysis Workflow

**Bước 1: Reconnaissance (Khám phá)**
```
- Đọc configuration files (axelor-config.properties)
- List modules trong /modules directory
- Identify key entities trong /domains
- Map dependencies trong build.gradle files
```

**Bước 2: Deep Dive (Đào sâu)**
```
- Trace code execution flows (Controller → Service → Repository → Entity)
- Analyze patterns (Interface + Implementation, Repository hooks, etc.)
- Read view XMLs để hiểu UI structure
- Study Guice modules để hiểu dependency graph
```

**Bước 3: Pattern Recognition (Nhận diện patterns)**
```
- So sánh multiple implementations (e.g., 80+ Services trong axelor-sale)
- Identify common patterns (e.g., @Transactional, Beans.get(), etc.)
- Document anti-patterns (e.g., service locator usage)
- Extract best practices từ well-designed modules
```

**Bước 4: Documentation (Ghi chép)**
```
- Write detailed explanations với Vietnamese prose
- Include code snippets với full context
- Add tables/diagrams cho visualization
- Provide troubleshooting tips
- Document limitations và workarounds
```

**Bước 5: Validation (Kiểm chứng)**
```
- Cross-check findings với multiple source files
- Test hypotheses bằng code tracing
- Verify assumptions với configuration values
- Mark uncertain findings với "⚠️ Suy luận:"
```

### Tools và Commands

#### Code Search Commands
```bash
# Find entity definitions
find modules/axelor-open-suite -name "*.xml" -path "*/domains/*"

# Find services
find modules/axelor-open-suite -name "*Service*.java" -path "*/service/*"

# Find Guice modules
find modules/axelor-open-suite -name "*Module.java"

# Search for patterns
grep -r "bind(" modules/*/src/main/java/*/module/*.java
grep -r "@Transactional" modules/*/src/main/java/*/service/

# Count lines
wc -l RESEARCH_STEP*.md
```

#### Analysis Priorities
1. **Configuration first** — Hiểu capabilities qua config keys
2. **Entities second** — Data model là foundation
3. **Services third** — Business logic implementation
4. **Views fourth** — UI/UX presentation layer
5. **Integration last** — Cross-cutting concerns (BPM, DMN, etc.)

### Quality Checklist

Mỗi RESEARCH_STEP file phải có:
- ✅ Header với nguồn evidence rõ ràng
- ✅ Mục lục (table of contents) nếu file > 500 lines
- ✅ Ít nhất 5 code snippets với full explanation
- ✅ Ít nhất 3 tables để visualize concepts
- ✅ Section "Những điều KHÔNG tìm thấy" — Acknowledge gaps
- ✅ Section "Best practices" hoặc "Recommendations"
- ✅ Cross-references tới related RESEARCH_STEP files
- ✅ Tiếng Việt thuần túy (không Vinglish)

### Example Analysis Flow

**Scenario:** Phân tích Service Layer patterns

```markdown
## A2. SERVICE LAYER PATTERNS

**Nguồn:** axelor-sale module, SaleModule.java, AppSaleServiceImpl.java [Từ source code]

Axelor sử dụng Google Guice làm bộ khung dependency injection (DI), không phải Spring Framework
như nhiều developer Java thường nghĩ. Điều này quan trọng vì Guice có semantics khác Spring,
đặc biệt về lifecycle management và scope handling. Service layer được tổ chức theo pattern
Interface + Implementation, với binding explicit trong Guice Module.

### Pattern: Interface + Implementation

**File nguồn:** `/modules/axelor-open-suite/axelor-sale/src/main/java/com/axelor/apps/sale/service/app/AppSaleService.java:15-18`

```java
public interface AppSaleService extends AppBaseService {
  public AppSale getAppSale();
  public void generateSaleConfigurations();
}
```

Interface khai báo contract, implementation cung cấp logic...

[Continue với detailed explanation, code snippets, và tables...]
```

---

## 🚀 NEXT STEPS / RECOMMENDATIONS

### Immediate (if continuing research)

1. **STEP 9: Integration Patterns** (SUGGESTED)
   - REST API usage patterns
   - External system connectors
   - Data import/export strategies
   - Web services (SOAP/REST)
   - Message queuing integration
   - File processing patterns

2. **STEP 10: Module Deep Dives** (SUGGESTED)
   - axelor-sale business logic flows (Order → Invoice → Shipment)
   - axelor-purchase procurement workflows
   - axelor-stock inventory management
   - axelor-account GL posting patterns
   - Real-world scenarios với complete traces

3. **STEP 11: Testing Strategies** (CRITICAL GAP)
   - Unit testing patterns (currently very few examples)
   - Integration testing strategies
   - Mock frameworks setup
   - Test data management
   - CI/CD pipeline recommendations

### Medium-term

4. **Comparative Analysis**
   - Create COMPARISON_AXELOR_VS_ODOO.md
   - Side-by-side feature comparison
   - Migration considerations (Odoo → Axelor or vice versa)
   - Use case recommendations (when to choose which)

5. **Implementation Guide**
   - Create IMPLEMENTATION_GUIDE.md
   - Step-by-step setup instructions (dev environment)
   - Best practices from all 8 RESEARCH_STEPs
   - Common pitfalls và troubleshooting
   - Starter project templates

6. **Deployment Guide**
   - Docker containerization recipes
   - Kubernetes deployment manifests
   - CI/CD pipeline setup (GitLab/GitHub Actions)
   - Production monitoring setup
   - Backup/restore procedures

### Long-term

7. **Performance Benchmarking**
   - Create benchmark test suite
   - Measure throughput (requests/sec)
   - Load testing scenarios (JMeter/Gatling)
   - Tuning recommendations based on metrics

8. **Security Hardening**
   - Security audit checklist
   - Penetration testing scenarios
   - OWASP compliance review
   - Secure deployment configurations

---

## 🔧 CONFIGURATION NOTES

### Important Settings (from STEP6)

#### Must Change for Production
```properties
# Enable L2 cache
hibernate.cache.region.factory_class = jcache
hibernate.javax.cache.provider = com.github.benmanes.caffeine.jcache.spi.CaffeineJCachingProvider

# Lower max pagination
api.pagination.max-per-page = 5000

# Enable JDBC batching
hibernate.jdbc.batch_size = 50

# Secure session cookies
session.cookie.secure = true
session.cookie.httpOnly = true
```

#### Connection Pool Tuning
```properties
# For 8-core server
hibernate.hikari.maximumPoolSize = 35
hibernate.hikari.minimumIdle = 10
hibernate.hikari.leakDetectionThreshold = 60000

# BPM/DMN separate pool (from STEP7)
studio.bpm.max.active.connections = 50
```

#### Recommended JVM Flags
```bash
-Xms4g -Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:MetaspaceSize=512m
-XX:MaxMetaspaceSize=1g
```

---

## 📝 WORKING NOTES

### Research Methodology (Detailed Above)
- Evidence-based analysis
- Structured documentation
- Vietnamese writing standards
- 5-step workflow: Reconnaissance → Deep Dive → Pattern Recognition → Documentation → Validation

### Tools Used
- Claude Code (AI-assisted analysis)
- VS Code (code navigation)
- Git (version control)
- Grep/Glob/Find (code search)
- Bash scripting (automation)

### Challenges Encountered

1. **BPM Engine Identification (STEP 4):** Binary addon, no source access
   - Solution: Inferred Camunda from package names, BPMN references, connection pool config

2. **DMN Engine Identification (STEP 7):** Binary addon, no source access
   - Solution: Analyzed Gradle cache dependencies, found camunda-dmn-7.23.0 JARs

3. **Code Generation Details (STEP 2):** Not obvious from source
   - Solution: Analyzed domain XML schema, examined generated entity structure

4. **Performance Defaults (STEP 6):** Why cache disabled?
   - Solution: Inferred development-friendly defaults, single-server assumption

5. **Studio Version Control (STEP 8):** No built-in export/import
   - Solution: Documented pain point, suggested CSV export workaround

6. **Testing Infrastructure (STEP 8):** Almost no test examples
   - Solution: Documented gap, recommended JUnit + Mockito setup

### Open Questions
- [ ] BPM engine licensing (Camunda Community vs Enterprise?)
- [ ] DMN engine licensing model
- [ ] Studio addon pricing (commercial addon)
- [ ] Official performance benchmarks
- [ ] Production deployment case studies
- [ ] Upgrade migration tools (8.x → 9.x)
- [ ] Multi-tenancy implementation patterns

---

## 🔄 CONTINUATION INSTRUCTIONS

### If Resuming Work on Different Machine

1. **Pull Latest Code**
   ```bash
   cd /Volumes/works/code/java/axelor/axelor-erp
   git pull
   ```

2. **Read This File First**
   - Understand what's been completed (Phase 1-4 ✅)
   - Check Next Steps section
   - Review Key Insights

3. **Review Relevant RESEARCH_STEP Files**
   - All 8 files có detailed context
   - Start với file relevant to your task

4. **Start New Claude Code Session**
   - Context: "Continuing Axelor research, read PROJECT_STATUS.md and RESEARCH_METHODOLOGY section"
   - Claude sẽ hiểu full context và methodology

### If Onboarding Someone New

**Reading Order:**

1. **Start Here:** Read this PROJECT_STATUS.md (especially RESEARCH_METHODOLOGY section)
2. **Architecture:** RESEARCH_STEP1_STRUCTURE.md
3. **Then Based on Interest:**
   - Backend developer → STEP2 (Database) → STEP8 (Development)
   - Security engineer → STEP3 (Security)
   - Business analyst → STEP4 (BPM) → STEP7 (DMN) → STEP5 (No-code)
   - DevOps → STEP6 (Performance)
   - Full-stack → Read all 8 in order

**Time Estimate:**
- Quick overview (all files): ~4 hours
- Deep study (all files): ~16 hours
- Mastery (với code tracing): ~40 hours

---

## 📞 CONTACT & RESOURCES

### Official Resources
- Website: https://www.axelor.com
- GitHub: https://github.com/axelor/axelor-open-suite
- Docs: https://docs.axelor.com
- Community: https://community.axelor.com
- Demo: https://demo.axelor.com

### This Analysis
- Author: Claude Code assisted research
- Date: January-February 2026
- Version: 4 Phases Complete
  - Phase 1: Initial research (6 files)
  - Phase 2: Detailed rewrite (6 files)
  - Phase 3: Vietnamese pure prose (6 files)
  - Phase 4: Supplemental research (2 files: DMN + Development)
- Status: **8 FILES COMPLETE ✅** (~12,146 lines total)

### Research Completeness

| Topic | Coverage | Confidence | Notes |
|-------|----------|-----------|-------|
| Architecture | 95% | High | Fully mapped |
| Database/ORM | 90% | High | Complete patterns |
| Security | 85% | High | Some LDAP gaps |
| BPM | 75% | Medium | Binary addon limitations |
| No-code | 90% | High | Comprehensive |
| Performance | 85% | High | Production tuning documented |
| DMN | 70% | Medium | Binary addon, inferred from cache |
| Development | 95% | High | Extensive Service/Repository analysis |
| **Testing** | **20%** | **Low** | **MAJOR GAP** — Needs STEP 9+ |
| **Integration** | **30%** | **Low** | **NEEDS RESEARCH** — Suggested STEP 10+ |

---

**Note:** This is a living document. Update as research progresses or new insights discovered.

**Last Activity:** Phase 4 HOÀN THÀNH ✅ — STEP 7 (DMN) và STEP 8 (Development) đã hoàn thành (2026-02-03)

**Research Quality:** Evidence-based, source code verified, Vietnamese prose style maintained throughout.
