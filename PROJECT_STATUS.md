# AXELOR OPEN SUITE - PROJECT STATUS & CONTEXT

**Last Updated:** 2026-02-02 (cập nhật lần 3 — Phase 3 còn 1 file cuối)
**Project:** Nghiên cứu và phân tích Axelor Open Suite ERP (Java-based)
**Version:** 8.5.10
**Repository:** axelor-erp (local analysis)

---

## 🔥 CÔNG VIỆC HIỆN TẠI — PHASE 3: VIẾT LẠI TIẾNG VIỆT THUẦN

### Mô tả
Viết lại toàn bộ 6 file RESEARCH_STEP*.md từ kiểu "Vinglish" (trộn lẫn tiếng Anh-Việt trong câu) sang **tiếng Việt thuần túy, mạch lạc**. Thuật ngữ kỹ thuật: tiếng Việt trước, tiếng Anh trong ngoặc lần đầu xuất hiện, sau đó dùng tiếng Việt.

### Quy tắc viết lại
1. **Thuật ngữ:** Tiếng Việt + (tiếng Anh) lần đầu → sau đó chỉ tiếng Việt
   - "Bộ khung (framework)" → lần sau: "bộ khung"
   - "Bộ đệm (cache)" → lần sau: "bộ đệm"
   - "Phiên làm việc (session)" → lần sau: "phiên"
2. **Giải thích:** Viết đoạn văn tiếng Việt tự nhiên, 3-5 câu tối thiểu
3. **Code snippets:** Giữ nguyên — chỉ viết lại phần giải thích
4. **Đánh dấu:** Giữ `[Từ source code]` và `[Suy luận]`

### Bảng thuật ngữ chính
| Tiếng Anh | Tiếng Việt |
|-----------|------------|
| Framework | Bộ khung (framework) |
| Cache | Bộ đệm (cache) |
| Session | Phiên làm việc |
| Entity | Thực thể (entity) |
| Trade-off | Đánh đổi (trade-off) |
| Record-level security | Bảo mật cấp bản ghi |
| Connection pool | Nhóm kết nối |
| Lazy loading | Tải lười |
| Batch processing | Xử lý hàng loạt |
| Multi-tenancy | Đa thuê bao |

### Tiến độ Phase 3

| # | File | Trạng thái | Ghi chú |
|---|------|-----------|---------|
| 1 | RESEARCH_STEP1_STRUCTURE.md | ❌ **CHƯA LÀM** | Cần viết lại tiếp |
| 2 | RESEARCH_STEP2_DATABASE.md | ✅ Hoàn thành | Đã viết lại tiếng Việt thuần |
| 3 | RESEARCH_STEP3_SECURITY.md | ✅ Hoàn thành | Đã viết lại tiếng Việt thuần |
| 4 | RESEARCH_STEP4_BPM.md | ✅ Hoàn thành | Đã viết lại tiếng Việt thuần |
| 5 | RESEARCH_STEP5_NOCODE.md | ✅ Hoàn thành | Đã viết lại tiếng Việt thuần |
| 6 | RESEARCH_STEP6_PERFORMANCE.md | ✅ Hoàn thành | Đã viết lại tiếng Việt thuần (~1538 dòng) |

### Trạng thái STEP1
- File đã được đọc hoàn toàn (859 dòng), chưa bắt đầu viết lại
- File gốc đã viết tiếng Việt khá tốt nhưng vẫn còn nhiều đoạn trộn lẫn tiếng Anh
- Gồm 10 mục chính + sơ đồ kiến trúc + câu hỏi mở

### Hướng dẫn tiếp tục
1. Mở Claude Code trong thư mục `axelor-erp`
2. Nói: **"Đọc PROJECT_STATUS.md và tiếp tục viết lại STEP1 sang tiếng Việt thuần"**
3. Claude sẽ:
   - Đọc file RESEARCH_STEP1_STRUCTURE.md hiện tại (859 dòng, đọc 1 lần là đủ)
   - Viết lại hoàn toàn bằng tiếng Việt theo quy tắc ở trên
   - Giữ nguyên code snippets, chỉ viết lại phần giải thích
   - Sau khi xong, dừng lại đưa tóm tắt để review
   - **Đây là file cuối cùng — xong là hoàn thành Phase 3**

### Quy trình cho mỗi STEP
1. Đọc toàn bộ file gốc (có thể cần chia chunk nếu file lớn)
2. Viết lại hoàn toàn bằng Write tool (một lần)
3. Dừng lại, đưa bảng tóm tắt + gợi ý review
4. Chờ user xác nhận trước khi làm step tiếp

---

## ✅ COMPLETED TASKS

### Phase 1: Initial Research (Completed)
Đã hoàn thành nghiên cứu sơ bộ và tạo 6 files research ban đầu:
- ✅ RESEARCH_STEP1_STRUCTURE.md (766 lines - version cũ)
- ✅ RESEARCH_STEP2_DATABASE.md (626 lines - version cũ)
- ✅ RESEARCH_STEP3_SECURITY.md (943 lines - version cũ)
- ✅ RESEARCH_STEP4_BPM.md (789 lines - version cũ)
- ✅ RESEARCH_STEP5_NOCODE.md (original version)
- ✅ RESEARCH_STEP6_PERFORMANCE.md (1,609 lines - version cũ)

### Phase 2: Detailed Analysis Rewrite (✅ COMPLETED 2026-02-02)

Đã viết lại TOÀN BỘ 6 files với phong cách mới chi tiết hơn:

### Phase 3: Vietnamese Pure Rewrite (🔄 ĐANG LÀM — 5/6 hoàn thành)

Viết lại từ Vinglish sang tiếng Việt thuần. Đã xong STEP 2-6, còn STEP 1:

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

---

## 📊 STATISTICS

### Code Analysis Coverage
- **Total lines analyzed:** ~7,016 dòng documentation
- **Source files examined:** 50+ files
- **Key files:**
  - axelor-config.properties (618 lines)
  - SaleOrder.xml (1,827 lines - view definitions)
  - Partner.xml, Product.xml (view extensions)
  - UserRestController.java, SaleOrderController.java
  - BatchDirectDebit.java (batch processing)
  - Domain XMLs (Currency.xml, Country.xml, etc.)

### Writing Style Transformation
**Old style:** Bullet points, technical English, terse
**New style:**
- ✅ Vietnamese prose paragraphs (3-5 sentences)
- ✅ Vietnamize terms (English in parentheses first time)
- ✅ Explain "WHY" not just "WHAT"
- ✅ Every code snippet has detailed explanation
- ✅ Clear distinction: [Từ source code] vs [Suy luận]

### File Growth
| File | Original | New | Growth |
|------|----------|-----|--------|
| STEP1 | 766 | 859 | +12% |
| STEP2 | 626 | 1,414 | +126% |
| STEP3 | 943 | 1,305 | +38% |
| STEP4 | 789 | 1,113 | +41% |
| STEP5 | ~400 | 1,294 | +224% |
| STEP6 | 1,609 | 2,031 | +26% |
| **TOTAL** | **5,133** | **7,016** | **+37%** |

---

## 🎯 KEY INSIGHTS & FINDINGS

### Architecture Strengths
1. **Modular Design:** Clean separation, loosely coupled modules
2. **Framework Maturity:** Industry-standard patterns (JPA, JAX-RS, Pac4j, Quartz)
3. **Extensibility:** Module system + view inheritance enables customization
4. **No-Code Capabilities:** 60-70% application logic declarative

### Architecture Weaknesses
1. **Complexity:** High learning curve (XML schemas, action types, domain DSL)
2. **BPM Addon:** External dependency, binary only, no source access
3. **Performance Defaults:** L2 cache disabled, JDBC batching off, max pagination too high
4. **Documentation:** Limited official docs, requires source code analysis

### Comparison với Odoo
| Aspect | Axelor | Odoo |
|--------|--------|------|
| **Language** | Java + Groovy | Python |
| **ORM** | JPA/Hibernate | Odoo ORM (custom) |
| **View System** | XML (similar to Odoo) | XML |
| **BPM** | External Camunda | Built-in workflows |
| **Code Gen** | Domain XML → Java | Python direct |
| **Frontend** | React | Owl (custom) |
| **Community** | Smaller | Larger |
| **Target** | Enterprise (Java shops) | SMBs |

**Verdict:** Axelor = "Java equivalent of Odoo" với stronger type safety nhưng smaller ecosystem.

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
├── PROJECT_STATUS.md                (this file) - Project tracking
│
├── src/main/resources/
│   ├── axelor-config.properties     (618 lines) - Main configuration
│   └── domains/                     - Entity definitions
│
├── modules/axelor-open-suite/
│   ├── axelor-base/                 - Core module
│   ├── axelor-sale/                 - Sales module
│   ├── axelor-account/              - Accounting module
│   └── [10+ other modules]
│
└── build.gradle                     - Multi-project build
```

---

## 🚀 NEXT STEPS / RECOMMENDATIONS

### Immediate (if continuing research)

1. **Comparative Analysis**
   - Create COMPARISON_AXELOR_VS_ODOO.md
   - Side-by-side feature comparison
   - Migration considerations
   - Use case recommendations

2. **Implementation Guide**
   - Create IMPLEMENTATION_GUIDE.md
   - Step-by-step setup instructions
   - Best practices
   - Common pitfalls

3. **Performance Tuning Guide**
   - Extract recommendations from STEP6
   - Create actionable checklist
   - Benchmark testing procedures

### Medium-term

4. **Module Deep Dives**
   - Analyze specific business modules (Sale, Purchase, Inventory)
   - Document business logic flows
   - API usage examples

5. **Integration Patterns**
   - REST API integration guide
   - External system connectors
   - Data import/export strategies

6. **Deployment Guide**
   - Docker containerization
   - Kubernetes deployment
   - CI/CD pipeline setup

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

### Research Methodology
- Start with configuration files (understand capabilities)
- Trace code execution flows (understand implementation)
- Analyze patterns (infer design decisions)
- Compare with industry standards (assess maturity)
- Document thoroughly (enable knowledge transfer)

### Tools Used
- Claude Code (AI-assisted analysis)
- VS Code (code navigation)
- Git (version control)
- Grep/Glob (code search)

### Challenges Encountered
1. **BPM Engine Identification:** Binary addon, no source access
   - Solution: Inferred Camunda from package names, BPMN references, connection pool config

2. **Code Generation Details:** Not obvious from source
   - Solution: Analyzed domain XML schema, examined generated entity structure

3. **Performance Defaults:** Why cache disabled?
   - Solution: Inferred development-friendly defaults, single-server assumption

### Open Questions
- [ ] BPM engine licensing (Camunda Community vs Enterprise?)
- [ ] Studio addon pricing model
- [ ] Official performance benchmarks
- [ ] Production deployment case studies

---

## 🔄 CONTINUATION INSTRUCTIONS

### If Resuming Work on Different Machine

1. **Pull Latest Code**
   ```bash
   git pull
   ```

2. **Read This File First**
   - Understand what's been completed
   - Check Next Steps section
   - Review Key Insights

3. **Review Relevant RESEARCH_STEP Files**
   - Based on what you need to work on
   - All files have detailed context

4. **Start New Claude Code Session**
   - Provide context: "Continuing Axelor research, read PROJECT_STATUS.md"
   - Claude will read files and understand context

### If Onboarding Someone New

1. **Start Here:** Read this PROJECT_STATUS.md
2. **Then Read:** RESEARCH_STEP1_STRUCTURE.md (architecture overview)
3. **Then Based on Interest:**
   - Database/ORM → STEP2
   - Security → STEP3
   - Workflows → STEP4
   - No-code → STEP5
   - Performance → STEP6

---

## 📞 CONTACT & RESOURCES

### Official Resources
- Website: https://www.axelor.com
- GitHub: https://github.com/axelor/axelor-open-suite
- Docs: https://docs.axelor.com
- Community: https://community.axelor.com

### This Analysis
- Author: Claude Code assisted research
- Date: January-February 2026
- Version: Comprehensive rewrite with Vietnamese prose style
- Status: Phase 2 COMPLETE ✅

---

**Note:** This is a living document. Update as research progresses or new insights discovered.

**Last Activity:** Phase 3 — đã xong 5/6 file (STEP2-6), còn lại STEP1 chưa viết lại (2026-02-02)
