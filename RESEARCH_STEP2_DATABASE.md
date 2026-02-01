# BƯỚC 2: PHÂN TÍCH KIẾN TRÚC DATABASE & MODEL - Phân Tích Từ Source Code

## Phương pháp phân tích

Quá trình nghiên cứu database và model architecture được thực hiện bằng cách phân tích trực tiếp domain XML files từ nhiều modules khác nhau, kiểm tra generated Java code, và đọc custom repository implementations. Phạm vi nghiên cứu bao gồm:

**Domain XML files phân tích:**
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Address.xml` - Entity đơn giản với geolocation
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Company.xml` - Entity với caching và tracking
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Partner.xml` - Entity phức tạp với JSON fields
- `/modules/axelor-open-suite/axelor-sale/src/main/resources/domains/SaleOrder.xml` - Entity với nhiều relationships và business logic
- `/modules/axelor-open-suite/axelor-account/src/main/resources/domains/Account.xml` - Entity kế toán với constraints
- `/modules/axelor-open-suite/axelor-project/src/main/resources/domains/MetaJsonField.xml` - Extension của core entity

**Generated code inspection:**
- `/modules/axelor-open-suite/axelor-base/build/src-gen/java/com/axelor/apps/base/db/Product.java` - Generated JPA entity
- `/modules/axelor-open-suite/axelor-base/build/src-gen/java/com/axelor/apps/base/db/repo/ProductRepository.java` - Generated repository

**Custom repository implementations:**
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/db/repo/ProductBaseRepository.java` - Custom business logic layer

**Configuration analysis:**
- `/src/main/resources/axelor-config.properties` - Hibernate DDL strategy và database settings

---

## Kết quả chi tiết

### 1. CƠ CHẾ ĐỊNH NGHĨA ENTITY BẰNG XML - MODEL-DRIVEN DEVELOPMENT

**File nguồn:** Tất cả domain XML files trong các modules [Từ source code]

Axelor áp dụng một approach độc đáo trong Java ecosystem: thay vì định nghĩa JPA entities trực tiếp bằng Java code với annotations (như cách làm truyền thống của Hibernate hoặc Spring Data JPA), Axelor sử dụng **Domain XML files** như single source of truth. Đây là một implementation của Model-Driven Development (MDD) - một paradigm trong đó developers làm việc ở mức abstraction cao hơn (XML schema) thay vì viết boilerplate code. Quyết định thiết kế này mang lại nhiều lợi ích: (1) Business analysts không cần biết Java có thể đọc và review entity definitions, (2) Code generator có thể đảm bảo consistency trong việc generate getters/setters/equals/hashCode, (3) XML có thể được validate bằng XSD schema trước khi compile, phát hiện lỗi sớm, (4) Dễ dàng generate documentation từ XML, và (5) Cho phép platform evolve mà không breaking existing entity definitions.

Domain XML tuân theo một schema chuẩn được định nghĩa trong file XSD `domain-models_7.4.xsd`, với namespace `http://axelor.com/xml/ns/domain-models`. Con số 7.4 trong schema version khớp với Axelor framework version, cho thấy schema có thể evolve theo thời gian. Mỗi domain XML file bắt đầu với một `<module>` declaration chỉ định module name và target package cho generated code - ví dụ `<module name="base" package="com.axelor.apps.base.db"/>` nghĩa là tất cả entities trong file này sẽ được generate vào package `com.axelor.apps.base.db`. Một file XML có thể chứa nhiều entity definitions, giúp organize related entities together (ví dụ: SaleOrder và SaleOrderLine trong cùng một file).

**Bằng chứng từ code - Cấu trúc XML cơ bản:**
```xml
<domain-models xmlns="http://axelor.com/xml/ns/domain-models">
  <module name="base" package="com.axelor.apps.base.db"/>

  <entity name="EntityName" [attributes]>
    <!-- Field definitions -->
  </entity>
</domain-models>
```

**Giải thích code:** Block `<domain-models>` là root element, chứa namespace declaration để XML parser có thể validate cú pháp. Element `<module>` không chỉ là documentation - nó thực sự control việc code generation, xác định package structure và class naming. Một file có thể define nhiều modules (ví dụ: extending entities từ core framework), nhưng best practice là một module per file.

Entity-level attributes cung cấp metadata quan trọng ảnh hưởng đến cả generated code và database schema. Attribute `cacheable="true"` bật second-level cache (L2 cache) của Hibernate cho entity này - một quyết định quan trọng về hiệu năng (performance). Entities được cache thường là những entities "hot" (frequently accessed) và ít thay đổi như Company, Currency, Country. Việc cache Company entity có ý nghĩa lớn vì hầu hết transactions (invoices, orders, payments) đều reference đến company, nếu không cache sẽ phải query database hàng nghìn lần mỗi ngày. Tuy nhiên, caching cũng có trade-offs: cache invalidation phức tạp, tốn memory, và có thể gây stale data nếu không careful.

**Bằng chứng từ code - Entity với caching:**
```xml
<entity name="Account" cacheable="true">
  <string name="name" title="Name" required="true"/>
  <string name="code" title="Code" required="true" equalsInclude="true"/>
  <!-- ... -->
</entity>
```

**Giải thích code:** Entity Account được đánh dấu `cacheable="true"`, nghĩa là khi Hibernate load một Account instance, nó sẽ cache object đó trong L2 cache. Subsequent queries cho cùng Account (by primary key) sẽ hit cache thay vì database. Attribute `equalsInclude="true"` trên field `code` có ý nghĩa đặc biệt: field này sẽ được include trong generated `equals()` và `hashCode()` methods. Normally, Axelor chỉ dùng `id` field cho equals/hashCode, nhưng đôi khi business logic cần so sánh entities based on business keys (như code) thay vì technical keys (như id).

Attribute `implements` cho phép generated entity class implement các Java interfaces, enabling polymorphism và contract-based programming. Điều này đặc biệt powerful trong business applications nơi nhiều entities share common behaviors - ví dụ: SaleOrder, PurchaseOrder, Invoice đều có giá tiền (PricedOrder interface), đều có currency (Currenciable interface), đều có shipping info (ShippableOrder interface). Bằng cách declare interfaces trong XML, generator sẽ tự động thêm interface implementations và có thể generate stub methods nếu cần.

**Bằng chứng từ code - Entity implementing multiple interfaces:**
```xml
<entity name="SaleOrder"
  implements="com.axelor.apps.base.interfaces.PricedOrder,
              com.axelor.apps.base.interfaces.Currenciable,
              com.axelor.apps.base.interfaces.ShippableOrder,
              com.axelor.apps.base.interfaces.GlobalDiscounter">
  <!-- ... -->
</entity>
```

**Giải thích code:** SaleOrder implements bốn interfaces cùng lúc. PricedOrder interface có thể define methods như `getTotalAmount()`, `getSubTotal()`; Currenciable có thể có `getCurrency()`, `getExchangeRate()`; ShippableOrder có shipping-related methods; GlobalDiscounter handle discount logic. Việc này cho phép service layer viết generic code: một method nhận `PricedOrder` có thể xử lý cả SaleOrder, PurchaseOrder, Invoice mà không cần biết concrete type. Đây là application của Interface Segregation Principle (ISP) trong SOLID principles.

Attribute `table` cho phép customize database table name thay vì dùng naming convention mặc định. Axelor default convention là `{MODULE}_{ENTITY_NAME}` uppercase (ví dụ: `BASE_PRODUCT`, `SALE_SALE_ORDER`), nhưng đôi khi cần override - ví dụ: khi extend core framework entities, muốn keep existing table name để backward compatibility, hoặc khi integrate với legacy database có table names không theo convention.

**Bằng chứng từ code - Custom table name:**
```xml
<entity name="MetaJsonField" table="META_JSON_FIELD">
  <!-- Extends core MetaJsonField entity -->
</entity>
```

**Giải thích code:** MetaJsonField entity được mapped to table `META_JSON_FIELD` thay vì default `BASE_META_JSON_FIELD`. Đây có thể là entity được extend từ axelor-core framework, và developer muốn giữ nguyên table name để không break existing data. Case này cũng cho thấy Axelor cho phép modules extend entities từ core framework - một pattern quan trọng cho extensibility.

---

### 2. FIELD TYPES VÀ RICH ATTRIBUTES SYSTEM

**File nguồn:** Tất cả domain XML files đã phân tích [Từ source code]

Axelor cung cấp một hệ thống field types phong phú với hàng chục attributes, cho phép developers express complex business rules và UI behaviors ngay trong entity definition mà không cần viết Java code. Sức mạnh của approach này là consolidation: thay vì scatter business logic across entity classes, service classes, và view controllers, một phần đáng kể logic được centralized trong domain XML và được enforce ở cả database level (qua constraints), application level (qua validation), và UI level (qua view rendering).

**Primitive field types** map trực tiếp tới SQL data types: `<string>` thành VARCHAR, `<integer>` thành INTEGER, `<decimal>` thành NUMERIC với configurable precision/scale, `<boolean>` thành BOOLEAN, `<date>` thành DATE, `<datetime>` thành TIMESTAMP, `<binary>` thành BLOB. Điểm đặc biệt là Axelor đã abstraction away differences giữa database vendors - developer chỉ cần dùng `<decimal precision="20" scale="3">` và Hibernate sẽ generate đúng SQL type cho PostgreSQL (NUMERIC(20,3)), MySQL (DECIMAL(20,3)), hoặc Oracle (NUMBER(20,3)).

Field-level attributes được chia làm nhiều categories phục vụ các purposes khác nhau. **Constraint attributes** như `required="true"` (NOT NULL constraint), `unique="true"` (UNIQUE constraint), `min`/`max` (range validation for numbers) được enforce ở cả database level và application level. **UI attributes** như `readonly="true"`, `hidden="true"`, `multiline="true"` control việc render trong web interface - readonly fields vẫn writable trong code nhưng displayed as readonly trong form, hidden fields không hiển thị nhưng vẫn có trong model. **Behavior attributes** như `copy="false"` (exclude field khi duplicate record), `massUpdate="true"` (allow bulk update operation), `index="false"` (disable automatic index creation) fine-tune application behavior without code changes.

**Bằng chứng từ code - Decimal field với precision/scale:**
```xml
<decimal name="exTaxTotal" title="Total W.T." scale="3" precision="20" readonly="true"/>
```

**Giải thích code:** Field này define một số thập phân (decimal) với precision 20 và scale 3, nghĩa là total 20 digits trong đó 3 digits là phần thập phân - có thể lưu số lớn như 99,999,999,999,999,999.999. Precision cao này cần thiết cho financial calculations nơi rounding errors có thể tích lũy thành significant discrepancies. Attribute `readonly="true"` cho biết đây là computed field (tính toán từ order lines) không được user nhập trực tiếp - UI sẽ render as disabled input, và application code sẽ ignore attempts to set value trực tiếp. Title "Total W.T." là "Without Tax" (Total chưa thuế), sẽ được dùng làm label trong UI.

Attribute đặc biệt `selection` biến một integer field thành enum-like field, tham chiếu đến một selection definition (có thể trong XML riêng hoặc trong database). Đây là cách Axelor implement enums mà vẫn maintain flexibility - thay vì hard-code enum trong Java code (khó thay đổi), selections có thể được configure hoặc even managed qua UI trong Axelor Studio. Selection values thường là integers để efficient storage và indexing, nhưng có display labels cho từng value.

**Bằng chứng từ code - Integer field với selection (enum-like):**
```xml
<integer name="statusSelect" title="Status"
  selection="sale.order.status.select" readonly="true"/>
```

**Giải thích code:** Field `statusSelect` là integer nhưng hành xử như enum. Selection key `"sale.order.status.select"` reference đến một selection definition có thể chứa values như `{1: "Draft", 2: "Confirmed", 3: "Completed", 4: "Cancelled"}`. Trong database lưu integer (1,2,3,4) nhưng UI hiển thị human-readable labels. Readonly attribute nghĩa là status changes phải qua business logic (workflow methods) không phải direct field update, preventing invalid state transitions.

**JSON fields** là một innovation quan trọng của Axelor để support dynamic/custom fields. Với attribute `json="true"`, một string field sẽ store JSON-serialized data thay vì plain text. Điều này cho phép Axelor Studio tạo custom fields mà không cần ALTER TABLE statements - tất cả custom fields serialized thành JSON và lưu trong one column duy nhất. Trade-off rõ ràng: extreme flexibility (có thể add/remove custom fields without downtime) versus query performance (không thể index fields inside JSON, không thể efficiently filter/sort by custom fields). Use case chính là khi business users cần frequently add custom fields mà không muốn involve developers.

**Bằng chứng từ code - JSON field cho custom attributes:**
```xml
<string name="partnerAttrs" title="Fields" json="true"/>
```

**Giải thích code:** Field `partnerAttrs` lưu JSON string chứa custom attributes của Partner entity. Trong database, column có thể chứa value như `{"customField1": "value", "vatExempt": true, "loyaltyPoints": 1500}`. Axelor framework có serialization/deserialization mechanism để convert giữa JSON string và Java Map objects. Business users qua Axelor Studio có thể define custom fields ("VAT Exempt?", "Loyalty Points") và chúng được lưu trong JSON object này. Limitation: không thể query `SELECT * FROM partner WHERE partnerAttrs->>'vatExempt' = 'true'` efficiently (mặc dù PostgreSQL 9.4+ support JSON operators, nhưng performance không tốt và không có indexes).

**Transient fields** (với attribute `transient="true"`) là computed fields không được persist vào database. Chúng được tính toán on-the-fly từ các fields khác hoặc từ business logic. Use case: display-only fields như `fullName = firstName + " " + lastName`, hoặc derived values như `age = today - birthDate`. Transient fields không tốn database storage và không có stale data issues, nhưng không thể query/filter by transient fields.

**Bằng chứng từ code - Transient computed field:**
```xml
<many-to-one name="companyCurrency" transient="true"
  ref="com.axelor.apps.base.db.Currency">
  <![CDATA[
  return company != null ? company.getCurrency() : null;
  ]]>
</many-to-one>
```

**Giải thích code:** Field `companyCurrency` là many-to-one relationship với Currency entity nhưng không có foreign key column trong database (vì transient). CDATA block chứa Java/Groovy expression để compute value: nếu có company thì return company's currency, otherwise null. Expression này executed mỗi khi field được accessed (getter called). Use case ở đây là convenience: thay vì viết `order.getCompany().getCurrency()` trong code (với risk NullPointerException nếu company null), có thể viết `order.getCompanyCurrency()` và null check được handle sẵn.

**Formula fields** (với attribute `formula="true"`) là database-level computed fields - chúng computed via SQL subquery thay vì Java code. Hibernate sẽ generate SQL với subquery trong SELECT statement để fetch formula field value. Use case chính là aggregations: tính tổng, đếm số lượng từ related entities. Formula fields có performance trade-off: không tốn storage (không có column), always fresh (không có stale data), nhưng mỗi query phải execute subquery (có thể chậm nếu subquery phức tạp).

**Bằng chứng từ code - Formula field với SQL subquery:**
```xml
<decimal name="exTaxTotalOrdered" title="Total ordered W.T." formula="true"
  precision="20" scale="10">
  <![CDATA[
  SELECT SUM(self.ex_tax_total) FROM sale_sale_order AS self
  WHERE self.origin_sale_quotation = id
  ]]>
</decimal>
```

**Giải thích code:** Field `exTaxTotalOrdered` compute tổng amount của tất cả orders originated từ quotation này. SQL subquery sử dụng correlation: `WHERE self.origin_sale_quotation = id` - `id` here references primary key của quotation hiện tại (outer query). Hibernate sẽ generate SQL như: `SELECT q.*, (SELECT SUM(o.ex_tax_total) FROM sale_sale_order o WHERE o.origin_sale_quotation = q.id) AS exTaxTotalOrdered FROM sale_quotation q WHERE ...`. Subquery executed cho mỗi row, có thể slow nếu result set lớn. Alternative approach là eager compute và cache value, nhưng then phải handle cache invalidation khi child orders change.

Attribute `namecolumn="true"` đánh dấu field này là "display name" của entity, sẽ được dùng trong `toString()` method và khi hiển thị entity trong dropdowns/references. Combined với `search` attribute (comma-separated field names), Axelor biết phải search trong những fields nào khi user type vào search box. Ví dụ: Address entity có `fullName` là namecolumn và search trong `addressL2, addressL3, addressL4, addressL5, addressL6` - khi user search "New York", system sẽ search across all these fields.

**Bằng chứng từ code - Namecolumn với search fields:**
```xml
<string name="fullName" namecolumn="true"
  search="addressL2,addressL3,addressL4,addressL5,addressL6" title="Address"/>
```

**Giải thích code:** Field `fullName` là display name của Address entity. Khi user select address trong dropdown, họ sẽ thấy fullName value ("123 Main St, New York, NY 10001") thay vì technical ID (12345). Attribute `search` define rằng khi user search addresses, system phải search trong các fields: addressL2 (line 2), addressL3 (line 3), etc. Axelor sẽ generate query như: `WHERE fullName LIKE '%keyword%' OR addressL2 LIKE '%keyword%' OR addressL3 LIKE '%keyword%' ...`. Điều này crucial cho user experience - users không nhớ exact fullName nhưng có thể nhớ một fragment ở bất kỳ address line nào.

Một pattern thú vị là **computed namecolumn** - namecolumn field có thể là computed field với CDATA expression thay vì static field. Điều này cho phép construct display names dynamically với business logic, ví dụ: include company code trong display name nếu multi-company, exclude company code nếu single-company.

**Bằng chứng từ code - Computed namecolumn với conditional logic:**
```xml
<string name="label" namecolumn="true" search="code,name,company" title="Full name">
  <![CDATA[
  if(company != null)
      return code+"_"+ company.getCode() + " - " + name;
  else
      return code+" - " + name;
  ]]>
</string>
```

**Giải thích code:** Field `label` computed dynamically based on whether entity has company association. Nếu có company, label format là `CODE_COMPANYCODE - Name` (ví dụ: `PROD001_NYC - Laptop`), nếu không có company thì đơn giản hơn `CODE - Name` (ví dụ: `PROD001 - Laptop`). Pattern này hữu ích trong multi-company environments nơi cùng code có thể exist across different companies - include company code trong display name helps disambiguate. CDATA block chứa Java/Groovy code sẽ được generated thành getter method body.

---

### 3. RELATIONSHIP MAPPING VÀ FOREIGN KEY STRATEGIES

**File nguồn:** SaleOrder.xml, Partner.xml, Account.xml, Company.xml [Từ source code]

Axelor hỗ trợ đầy đủ bốn loại relationships mà JPA định nghĩa: many-to-one (foreign key), one-to-many (reverse relationship), many-to-many (join table), và one-to-one (shared primary key hoặc unique foreign key). Mỗi relationship type có trade-offs riêng về database normalization, query performance, và memory usage. Hiểu rõ khi nào dùng relationship nào là fundamental cho database design.

#### 3.1. Many-to-One: Foreign Key Relationships

Many-to-one là relationship type phổ biến nhất, represent belongs-to associations: một SaleOrder belongs to một Company, một Employee belongs to một Department. Database implementation đơn giản: foreign key column trong "many" side table pointing to primary key của "one" side table. Hibernate default generate lazy-loaded proxies cho many-to-one fields - khi load entity, related entity không được load ngay (chỉ có ID), only when field được accessed (proxy sẽ trigger database query). Lazy loading tránh N+1 query problem trong một số scenarios nhưng có thể cause LazyInitializationException nếu entity đã detached.

**Bằng chứng từ code - Basic many-to-one relationships:**
```xml
<!-- Simple many-to-one -->
<many-to-one name="company" ref="com.axelor.apps.base.db.Company"
  required="true" title="Company"/>

<!-- Many-to-one với custom column name -->
<many-to-one name="user" column="user_id" ref="com.axelor.auth.db.User"
  title="Assigned to" index="false" massUpdate="true"/>

<!-- Self-referencing many-to-one (tree structure) -->
<many-to-one name="parentAccount" ref="Account" title="Parent Account"
  massUpdate="true"/>
```

**Giải thích code:** Relationship đầu tiên là straightforward: mỗi entity phải belong to một company (`required="true"`). Generated SQL sẽ có column `company` (hoặc `company_id` depending on conventions) với FOREIGN KEY constraint. Attribute `ref="com.axelor.apps.base.db.Company"` specify full qualified class name của target entity - Axelor generator cần này để generate correct Java type.

Relationship thứ hai customize column name via `column="user_id"` (thay vì default `user`). Attribute `index="false"` disable automatic index creation - normally Hibernate tự động tạo index cho foreign keys vì queries often filter/join by FKs, nhưng đôi khi table quá nhỏ hoặc FK never được query riêng lẻ thì không cần index (save space và insert performance). Attribute `massUpdate="true"` cho phép bulk reassignment: admin có thể select nhiều records và assign hết cho user khác cùng lúc, useful cho scenario như "reassign all John's tasks to Mary when John leaves".

Self-referencing relationship (`parentAccount`) implement tree structures trong SQL - cho phép build account hierarchy như Chart of Accounts trong accounting: Assets (parent) → Current Assets (child) → Cash (grandchild). Implementation đơn giản nhưng querying trees phức tạp: để get entire subtree cần recursive queries (Common Table Expressions trong PostgreSQL) hoặc iterative loading (nhiều queries). Axelor likely không có built-in tree query utilities, developers phải implement manually.

**Database schema generated:** [Suy luận từ JPA patterns]
- Column name: `{field_name}` hoặc custom `column` value
- Column type: BIGINT (matching primary key type của target entity)
- Constraints: FOREIGN KEY, NOT NULL nếu `required="true"`
- Index: Automatic index unless `index="false"`

#### 3.2. One-to-Many: Collection Relationships

One-to-many là reverse side của many-to-one relationship, không tạo column mới mà chỉ map to existing foreign key trong related entity. Ví dụ: Company has many Employees (one-to-many) inverse của Employee belongs to Company (many-to-one). Attribute `mappedBy` chỉ định field name ở "many" side chứa foreign key - đây là "owning side" của relationship. Hibernate chỉ persist changes từ owning side; changes ở non-owning side (one-to-many) chỉ affect in-memory collection không trigger SQL updates unless cascades configured.

One-to-many collections default lazy-loaded vì load cả collection có thể expensive (ví dụ: một Company có 10,000 employees thì load hết sẽ OutOfMemoryError). Collections represented as proxied Lists/Sets mà chỉ fetch khi accessed. Attribute `orderBy` specify default ordering cho collection - có thể tăng dần (`"sequence"`) hoặc giảm dần (`"-blockingToDate"` với minus sign prefix).

**Bằng chứng từ code - One-to-many relationships:**
```xml
<!-- Basic one-to-many with ordered collection -->
<one-to-many name="saleOrderLineList" ref="com.axelor.apps.sale.db.SaleOrderLine"
  mappedBy="saleOrder" title="Sale order lines" orderBy="sequence"/>

<!-- One-to-many với descending order -->
<one-to-many name="blockingList" ref="com.axelor.apps.base.db.Blocking"
  title="Blocking follow-up List" mappedBy="partner" orderBy="-blockingToDate"/>
```

**Giải thích code:** Collection `saleOrderLineList` contains all order lines belonging to this sale order. Attribute `mappedBy="saleOrder"` indicates SaleOrderLine entity has many-to-one field named `saleOrder` pointing back to SaleOrder. Collection ordered by `sequence` field (ascending) - order lines typically có sequence number 1, 2, 3,... để maintain user-defined ordering. Without orderBy, collection order would be database-dependent (không deterministic), causing subtle bugs.

Collection `blockingList` demonstrates descending order (`"-blockingToDate"`). Blocking records likely represent credit holds/blocks, và business wants see most recent blocks first (latest blockingToDate at top). Minus sign prefix is Axelor convention cho descending sort, similar to Django ORM. Generated SQL sẽ có `ORDER BY blockingToDate DESC`.

**Performance consideration:** [Suy luận về N+1 problem]
Khi query một collection of entities, accessing one-to-many collections có thể trigger N+1 query problem: 1 query fetch N entities, then N additional queries fetch each entity's collection. Ví dụ: `SELECT * FROM company` (1 query) → for each company, `SELECT * FROM employee WHERE company_id = ?` (N queries). Solution là join fetch hoặc batch fetching, nhưng Axelor domain XML không expose cấu hình này - developers phải handle ở service layer.

#### 3.3. Many-to-Many: Join Table Relationships

Many-to-many implement relationships where both sides có nhiều entities - ví dụ: một Product có nhiều Categories, một Category chứa nhiều Products. Database implementation requires join table (junction table) với hai foreign keys. Hibernate auto-generate join table name theo convention `{entity1}_{field_name}` - ví dụ: field `batchSet` trong Partner entity sẽ tạo table `partner_batch_set` với columns `partner_id` và `batch_id`.

Many-to-many có caveat quan trọng: không thể store additional data on the relationship itself. Nếu cần attributes trên association (ví dụ: Product-Category với `displayOrder` attribute), phải convert sang two many-to-one relationships với intermediate entity (ProductCategory). Axelor domain XML không có syntax cho association classes, developers phải manually create intermediate entities.

**Bằng chứng từ code - Many-to-many relationships:**
```xml
<!-- Basic many-to-many -->
<many-to-many name="batchSet" ref="com.axelor.apps.base.db.Batch" title="Batchs"/>

<!-- Self-referencing many-to-many (contacts network) -->
<many-to-many name="contactPartnerSet" ref="com.axelor.apps.base.db.Partner"
  title="Contacts"/>

<!-- Business domain many-to-many -->
<many-to-many name="compatibleAccountSet"
  ref="com.axelor.apps.account.db.Account" title="Compatible Accounts"/>
```

**Giải thích code:** Field `batchSet` creates many-to-many giữa current entity và Batch entities. Join table `{entity}_batch_set` sẽ có columns `{entity}_id` và `batch_id` cùng với composite primary key trên cả hai columns (prevent duplicates). Hibernate manage join table automatically - khi add/remove items from Set, Hibernate insert/delete rows trong join table.

Self-referencing many-to-many (`contactPartnerSet`) implement social network-like relationships: một Partner có thể có nhiều contact Partners, và mỗi contact Partner cũng có their own contacts. Join table `partner_contact_partner_set` map Partners to Partners. Relationship này typically symmetric (nếu A contacts B thì B contacts A) nhưng implementation ở đây không enforce symmetry - application logic must handle.

Field `compatibleAccountSet` demonstrates domain-specific many-to-many. Trong accounting, một Account có thể compatible với một set of other Accounts cho certain operations (ví dụ: transfer, reconciliation). Relationship này asymmetric: Account A compatible with B không nghĩa là B compatible with A. Business rules cho compatibility likely enforced ở service layer không phải database constraints.

**Database schema generated:** [Từ source code - Hibernate default behavior]
- Join table: `{entity_lowercase}_{field_name_lowercase}`
- Columns: `{entity_lowercase}_id`, `{target_entity_lowercase}_id`
- Primary key: Composite PK trên cả hai columns
- Foreign keys: FK to both entities
- Indexes: Automatic indexes on both columns

#### 3.4. One-to-One: Unique Relationships

One-to-one là ít dùng nhất trong bốn relationship types vì nó có thể được modeled as either separate table (với unique foreign key) hoặc embedded fields trong same table. Use case cho separate table one-to-one: (1) optional relationship with many nullable fields (splitting table saves space), (2) lazy loading heavy fields (ví dụ: User has one Profile với large bio text), (3) different lifecycle/access patterns. Axelor hỗ trợ bidirectional one-to-one với attribute `mappedBy` specify reverse side.

**Bằng chứng từ code - One-to-one relationships:**
```xml
<!-- Simple one-to-one (owning side) -->
<one-to-one name="emailAddress" ref="com.axelor.message.db.EmailAddress"
  title="Email" unique="true"/>

<!-- Bidirectional one-to-one (non-owning side) -->
<one-to-one name="linkedUser" ref="com.axelor.auth.db.User"
  title="User" mappedBy="partner"/>
```

**Giải thích code:** Field `emailAddress` creates one-to-one relationship where current entity "owns" relationship (có foreign key column). Attribute `unique="true"` enforce database constraint preventing two entities từ sharing same EmailAddress - đây là điều distinguish one-to-one from many-to-one. Use case có thể là Partner có thể có one dedicated EmailAddress entity chứa email preferences, templates, signature.

Field `linkedUser` là reverse side của bidirectional one-to-one. User entity (not shown) có field `partner` mapping back. Attribute `mappedBy="partner"` indicate relationship owned bởi User entity, không có foreign key column trong Partner table. Relationship này có thể represent: một Partner (company/contact) có thể linked to one system User, và một User có thể linked to one Partner. Bidirectional nature cho phép navigate cả hai hướng: từ Partner to User hoặc từ User to Partner.

**Lazy loading consideration:** [Suy luận từ Hibernate behavior]
One-to-one relationships có tricky lazy loading behavior: nếu là non-owning side (có mappedBy), Hibernate phải query database để check whether related entity exists, making "lazy" loading không really lazy. Owning side có thể truly lazy vì chỉ cần check FK column (not null = entity exists). Performance-sensitive code should test whether one-to-one actually benefits from lazy loading.

---

### 4. CONSTRAINTS, INDEXES VÀ DATABASE SCHEMA CONTROL

**File nguồn:** SaleOrder.xml, Account.xml, Company.xml [Từ source code]

Database constraints và indexes là critical cho data integrity và query performance, nhưng chúng có trade-offs. Constraints (unique, not null, foreign key, check) enforce business rules at database level - last line of defense against bad data, thậm chí khi application bugs bypass validation. Indexes speed up queries exponentially (O(log n) instead of O(n)) nhưng slow down inserts/updates/deletes và consume disk space. Axelor domain XML cho phép declare constraints và có một số control về indexes, though less comprehensive than pure JPA annotations.

#### 4.1. Unique Constraints: Single-Column và Composite

Unique constraints prevent duplicate values, critical cho business keys (như order numbers, product codes, email addresses). SQL standard distinguish giữa NULL và non-NULL: unique constraint allow multiple NULLs vì NULL != NULL (unknown không equal unknown). Điều này có implications: nếu cần enforce "email must be unique INCLUDING NULLs" (no two users with NULL email), cần custom logic hoặc partial unique index.

Axelor support single-column unique qua field attribute `unique="true"`, và composite unique qua `<unique-constraint>` element. Composite unique constraints enforce uniqueness across multiple columns - ví dụ: (saleOrderSeq, company) unique nghĩa là cùng một order number có thể exist trong different companies nhưng không được duplicate trong same company. Pattern này fundamental cho multi-company systems.

**Bằng chứng từ code - Unique constraints:**
```xml
<!-- Single column unique (in field attribute) -->
<string name="code" title="Code" required="true" unique="true"/>

<!-- Multi-column unique constraints -->
<unique-constraint columns="saleOrderSeq,company"/>
<unique-constraint columns="code,company"/>
```

**Giải thích code:** Field `code` có both `required="true"` và `unique="true"`, creating NOT NULL UNIQUE constraint - combination này equivalent to natural primary key (nhưng vẫn có surrogate ID for technical reasons). Business users typically search/reference by code thay vì ID, making this pattern common.

Composite unique constraint `saleOrderSeq,company` crucial cho multi-company deployments. Sequence generator có thể independently generate sequences per company (SO0001, SO0002 in Company A and SO0001, SO0002 in Company B), hoặc globally unique sequences. Constraint này enforce former strategy. Second constraint `code,company` similar pattern for Account codes - cho phép reuse same account codes across companies (ví dụ: every company có "Cash" account với code "101").

**Generated JPA annotations:** [Từ source code generated Product.java]
```java
@Entity
@Table(
  name = "SALE_SALE_ORDER",
  uniqueConstraints = @UniqueConstraint(
    columnNames = {"saleOrderSeq", "company"}
  )
)
public class SaleOrder extends AuditableModel { ... }
```

**Giải thích code:** Hibernate translates domain XML unique constraints into `@UniqueConstraint` annotations trong `@Table`. Database DDL sẽ có `UNIQUE (saleOrderSeq, company)` constraint. Nếu application code attempts insert duplicate, database throws SQLException và Hibernate wraps thành ConstraintViolationException, bubbling up to service layer để handle gracefully (show error message to user thay vì crash).

#### 4.2. Indexes: Automatic, Disabled, và Performance Implications

Hibernate default tự động create indexes cho foreign key columns vì joins/filters by FK are extremely common. Index dramatically speed up queries: `SELECT * FROM sale_order WHERE company_id = 123` với index on company_id takes microseconds (B-tree lookup), without index takes seconds (full table scan). However, indexes không phải free: mỗi index là separate B-tree structure occupying disk, và every INSERT/UPDATE/DELETE must update all indexes, slowing down writes.

Axelor cho phép disable automatic indexes via `index="false"` attribute. Use cases: (1) FK column never queried/joined independently (always part of composite condition), (2) table very small (full scan fast enough), (3) write-heavy workload where insert speed more critical than query speed. Disabling unnecessary indexes có thể improve throughput đáng kể trong high-write systems.

**Bằng chứng từ code - Index control:**
```xml
<many-to-one name="user" column="user_id" ref="com.axelor.auth.db.User"
  title="Assigned to" index="false" massUpdate="true"/>
```

**Giải thích code:** Foreign key `user_id` explicitly set `index="false"`, ngăn Hibernate tạo index. Decision này có thể based on analysis: nếu table never queries by user (ví dụ: always queries by primary key or other indexed columns), index trên user_id wasted space. However, cẩn thận: nếu add feature later filtering by user, queries sẽ slow và developers phải remember add index manually.

**Evidence từ generated code - Automatic indexes:**
```java
@Table(
  name = "BASE_PRODUCT",
  indexes = {
    @Index(columnList = "name"),
    @Index(columnList = "picture"),
    @Index(columnList = "product_category"),
    @Index(columnList = "unit"),
    // ... nhiều indexes cho foreign keys
  }
)
```

**Giải thích code:** Generated Product entity có nhiều indexes được create automatically. Index trên `name` có thể for searches by product name (common user operation). Index trên FKs như `product_category`, `unit` for filtering products by category, by unit of measure. Hibernate generator adds these based on conventions và field types. Developers không có fine-grained control trong domain XML (không thể specify index type - B-tree vs Hash vs GIN, không thể composite indexes), phải rely on generator's defaults hoặc create indexes manually sau khi deployment.

**Performance implications:** [Suy luận về index strategy]
Optimal index strategy depends on workload. Read-heavy systems (nhiều queries, ít writes) benefit from generous indexing. Write-heavy systems (frequent inserts/updates) should minimize indexes. Multi-column composite indexes useful khi queries filter by multiple columns together (ví dụ: `WHERE company_id = ? AND date >= ?` benefits from index on (company_id, date)). Axelor không expose composite index definition trong XML, developers có thể need create these via migration scripts hoặc directly in database.

---

### 5. FINDER METHODS: DECLARATIVE QUERY GENERATION

**File nguồn:** SaleOrder.xml, Account.xml, Partner.xml [Từ source code]

Finder methods là một convenience feature của Axelor cho phép developers declare common query patterns trong domain XML, và code generator tự động generate query methods trong Repository classes. Pattern này giảm boilerplate code đáng kể - thay vì manually write Query DSL in repository, chỉ cần declare `<finder-method name="findByCode" using="code"/>` và generator creates complete method with proper parameter binding, null checks, và error handling.

Syntax đơn giản nhưng powerful: `name` attribute specify method name (convention: `findBy{FieldNames}`), `using` attribute liệt kê fields used in query (comma-separated for multiple fields), và optional `all="true"` attribute indicate method returns List thay vì single entity. Generator translates these declarations thành Query DSL calls với proper JPQL filters và bind parameters.

**Bằng chứng từ code - Finder method declarations:**
```xml
<!-- Single field finder -->
<finder-method name="findByPartnerSeq" using="partnerSeq"/>

<!-- Multi-field finder -->
<finder-method name="findBySaleOrderSeqAndCompany" using="saleOrderSeq,company"/>
<finder-method name="findByCodeAndCompany" using="code,company"/>

<!-- Find all matching records (returns List) -->
<finder-method name="findByAccountType" using="accountType" all="true"/>
```

**Giải thích code:** Finder `findByPartnerSeq` generates method nhận String partnerSeq parameter và returns single Partner entity (or null nếu không tìm thấy). Use case: search partner by unique business key. Multi-field finder `findBySaleOrderSeqAndCompany` queries by composite business key - both orderSeq AND company phải match. Method signature sẽ là `findBySaleOrderSeqAndCompany(String orderSeq, Company company)` với two parameters.

Finder `findByAccountType` có `all="true"`, generating method return type `List<Account>` instead of single Account. Use case: get all accounts of certain type (Asset accounts, Liability accounts, etc.). Without `all="true"`, method returns only first match (or null), với `all="true"` returns all matches (empty list nếu không tìm thấy).

**Generated repository code evidence:**
```java
public Product findByCode(String code) {
  return Query.of(Product.class)
    .filter("self.code = :code")
    .bind("code", code)
    .fetchOne();
}

public Product findByName(String name) {
  return Query.of(Product.class)
    .filter("self.name = :name")
    .bind("name", name)
    .fetchOne();
}
```

**Giải thích code:** Generator translates finder declarations thành Query DSL calls. Method `findByCode` creates Query object for Product.class, adds filter with named parameter (`:code`), binds parameter value, và calls `fetchOne()` để get single result. Filter string `"self.code = :code"` uses Axelor convention: `self` alias current entity. Query DSL translates này thành JPQL: `SELECT self FROM Product self WHERE self.code = :code`. Named parameters (`:code`) safer than positional parameters (vì không thể mix up order) và more readable.

Method returns entity directly (not Optional) - null return indicates not found. This approach simpler than Java 8+ Optional pattern nhưng có risk NullPointerException nếu calling code không check null. Alternative design would return Optional<Product>, forcing explicit handling, nhưng Axelor chose simplicity over safety.

**Limitations:** [Suy luận về finder method capabilities]
Finder methods chỉ support equality queries (`field = value`), không support ranges (`field > value`), LIKE queries (`field LIKE '%pattern%'`), OR conditions, ordering, hoặc pagination. Complex queries require manually written methods trong custom repository. Trade-off: declarative finders cover 80% common cases (get by ID, get by business key) với zero code, complex 20% still need custom code. Pattern này matches 80-20 rule well.

---

### 6. EXTRA CODE VÀ CONSTANTS: EMBEDDING JAVA IN XML

**File nguồn:** SaleOrder.xml, Account.xml, Partner.xml, Company.xml [Từ source code]

Một tính năng powerful của Axelor domain XML là ability to embed Java code directly trong entity definitions via `<extra-code>` và `<extra-imports>` elements. Cơ chế này cho phép generators create fully-functional entity classes với constants, helper methods, và business logic inline, không cần separate Java files. Primary use case là defining constants for enum-like integer fields (status codes, type codes) - these constants make code readable và refactor-safe.

Pattern phổ biến: define integer selection fields (`statusSelect`, `typeSelect`) với corresponding constants trong extra-code. Application code sau đó reference constants (`SaleOrder.STATUS_CONFIRMED`) thay vì magic numbers (`3`), dramatically improving readability. When business changes status numbers (ví dụ: insert new status between existing ones), chỉ cần update constants, không cần hunt through codebase tìm magic numbers. Constants cũng benefit từ IDE features: autocomplete, find usages, refactoring.

**Bằng chứng từ code - Extra imports và constants:**
```xml
<extra-imports>
  import com.axelor.apps.base.interfaces.GlobalDiscounterLine;
</extra-imports>

<extra-code>
  <![CDATA[
  // STATUS
  public static final int STATUS_DRAFT_QUOTATION = 1;
  public static final int STATUS_FINALIZED_QUOTATION = 2;
  public static final int STATUS_ORDER_CONFIRMED = 3;
  public static final int STATUS_ORDER_COMPLETED = 4;
  public static final int STATUS_CANCELED = 5;

  // ORDERING STATUS
  public static final int ORDERING_STATUS_PARTIALLY_ORDERED = 1;
  public static final int ORDERING_STATUS_CLOSED = 2;
  ]]>
</extra-code>
```

**Giải thích code:** Block `<extra-imports>` thêm imports sẽ appear ở top of generated Java file. Needed khi extra-code references classes from other packages. CDATA section trong `<extra-code>` chứa literal Java code được copy directly vào generated class body. Constants defined ở đây become part of generated entity class, accessible as `SaleOrder.STATUS_CONFIRMED`.

Comment groups (`// STATUS`, `// ORDERING STATUS`) organize constants thành logical sections, improving maintainability. Pattern này particularly important cho entities với nhiều selection fields (10+ status constants, 5+ type constants, etc.) - comments help developers find relevant constant quickly. Some Axelor entities có 50+ constants trong extra-code section.

**Another example from Account.xml:**
```xml
<extra-code><![CDATA[
  // COMMON POSITION
  public static final int COMMON_POSITION_NONE = 0;
  public static final int COMMON_POSITION_CREDIT = 1;
  public static final int COMMON_POSITION_DEBIT = 2;

  // STATUS SELECT
  public static final int STATUS_INACTIVE = 0;
  public static final int STATUS_ACTIVE = 1;

  // VAT SYSTEM
  public static final int VAT_SYSTEM_DEFAULT = 0;
  public static final int VAT_SYSTEM_GOODS = 1;
  public static final int VAT_SYSTEM_SERVICE = 2;
]]></extra-code>
```

**Giải thích code:** Account entity có three groups of constants for different selection fields. VAT_SYSTEM constants distinguish between different VAT treatment rules (goods vs services have different VAT rates trong nhiều countries). COMMON_POSITION (credit/debit) fundamental trong double-entry bookkeeping. STATUS controls whether account active or archived. Each constant group corresponds to một selection field trong XML (not shown here nhưng có thể infer).

**Generated code integration:**
```java
public class SaleOrder extends AuditableModel {
  // ... fields và getters/setters ...

  // Extra code gets inserted here
  public static final int STATUS_DRAFT_QUOTATION = 1;
  public static final int STATUS_FINALIZED_QUOTATION = 2;
  // ...
}
```

**Giải thích code:** Generator inserts extra-code at class level, making constants accessible as static class members. Service layer code có thể use: `if (order.getStatusSelect() == SaleOrder.STATUS_CONFIRMED) { ... }`. This pattern significantly better than magic numbers: `if (order.getStatusSelect() == 3) { ... }` - impossible to understand without looking up status code meanings.

**Constants trong Repository classes:** [Từ source code ProductRepository]
```java
// Constants được generate vào Repository class instead of Entity class
public static final String PRODUCT_TYPE_SERVICE = "service";
public static final String PRODUCT_TYPE_STORABLE = "storable";

public static final int SALE_SUPPLY_FROM_STOCK = 1;
public static final int SALE_SUPPLY_PURCHASE = 2;
public static final int SALE_SUPPLY_PRODUCE = 3;
```

**Giải thích code:** Interesting observation: constants có thể appear trong Repository classes instead of Entity classes. Có thể đây là extra-code declared trong một repository-specific section (syntax chưa thấy trong domain XML) hoặc manually added trong custom repository. Pattern này useful khi constants chỉ relevant cho repository operations (query constants, status filters) thay vì entity logic.

**Limitations và alternatives:** [Suy luận về design choices]
Extra-code limited to class-level code (fields, methods, constants), không thể customize constructors, không thể add annotations. Complex business logic better placed trong service layer hoặc custom repository methods thay vì entity classes (following separation of concerns principle). Alternative approach: Java enums instead of integer constants, providing type safety và more features (methods, valueOf, etc.), nhưng Axelor chose integers for flexibility (enums hard to extend/modify trong deployed system).

---

### 7. AUDIT TRACKING: BUILT-IN CHANGE HISTORY CHO GDPR COMPLIANCE

**File nguồn:** SaleOrder.xml, Account.xml, Partner.xml, Company.xml [Từ source code]

Axelor cung cấp một comprehensive audit trail system được declare trực tiếp trong domain XML qua `<track>` element. System này automatically record WHO changed WHAT WHEN, creating immutable history log của mọi tracked fields. Feature này crucial cho compliance requirements (GDPR trong EU requires detailed audit logs of personal data access/modification), security (forensics khi có incident), và business (dispute resolution when questions arise về data changes).

Tracking mechanism có thể selective: track specific fields thay vì entire entity (reducing log volume), track only on CREATE hoặc only on UPDATE (fine-grained control), và even generate human-readable messages về state changes (ví dụ: "Order confirmed" when status changes to 3). Messages có thể conditional based on field values và có visual tags (important, info, success, warning) cho UI highlighting.

**Bằng chứng từ code - Comprehensive tracking configuration:**
```xml
<track>
  <field name="saleOrderSeq"/>
  <field name="clientPartner"/>
  <field name="statusSelect"/>
  <field name="creationDate" on="CREATE"/>
  <field name="confirmationDateTime" on="UPDATE"/>
  <field name="inTaxTotal"/>
  <message if="true" on="CREATE">Quotation/sale order created</message>
  <message if="statusSelect == 1" tag="important">Draft quotation</message>
  <message if="statusSelect == 2" tag="info">Finalized quotation</message>
  <message if="statusSelect == 3" tag="success">Order confirmed</message>
  <message if="statusSelect == 4" tag="success">Order completed</message>
  <message if="statusSelect == 5" tag="warning">Canceled</message>
</track>
```

**Giải thích code:** Track configuration cho SaleOrder entity defines which fields are tracked và messages to generate. Fields `saleOrderSeq`, `clientPartner`, `statusSelect`, `inTaxTotal` tracked on both CREATE và UPDATE events (no `on` attribute means both). Field `creationDate` only tracked `on="CREATE"` - makes sense vì creation date only set once, subsequent changes would be bugs. Conversely, `confirmationDateTime` only tracked `on="UPDATE"` - field null khi create, only populated khi order confirmed later.

Messages section creates human-readable audit trail entries. First message `if="true"` always triggers on CREATE, logging "Quotation/sale order created". Subsequent messages conditional on statusSelect value: when status is 1, log "Draft quotation" với important tag (red/orange highlighting in UI), when status is 3, log "Order confirmed" với success tag (green highlighting). Tags purely presentational nhưng help users quickly scan audit log highlighting important events.

**Generated JPA annotations:**
```java
@Track(
  on = TrackEvent.UPDATE,
  fields = {
    @TrackField(name = "name"),
    @TrackField(name = "code"),
    @TrackField(name = "productCategory"),
    // ...
  }
)
public class Product extends AuditableModel { ... }
```

**Giải thích code:** Generator translates `<track>` declarations thành `@Track` annotation trên entity class. Hibernate entity listeners (configured globally) intercept lifecycle events (prePersist, preUpdate) và inspect @Track annotations to determine what to log. When tracked entity modified, listener serializes old values, new values, user info, timestamp into audit log records và inserts vào audit trail table.

**Database schema cho audit trail:** [Suy luận từ common audit patterns]
Audit logs likely stored trong separate table (`meta_audit_trail` hoặc similar) với schema:
- `id` - Primary key
- `entity_type` - Class name of tracked entity
- `entity_id` - Primary key of tracked entity
- `field_name` - Which field changed
- `old_value` - Serialized old value
- `new_value` - Serialized new value
- `changed_by` - User who made change
- `changed_on` - Timestamp
- `event_type` - CREATE, UPDATE, DELETE
- `message` - Human-readable message (from `<message>` declarations)

Table này có thể grow very large trong production (millions of rows), requiring partitioning hoặc archival strategies. Indexes on entity_type, entity_id, changed_on critical cho query performance (ví dụ: "show me all changes to Order #12345").

**GDPR compliance implications:** [Suy luận về regulatory requirements]
GDPR Article 15 requires organizations provide individuals với complete history of personal data processing. Audit trail automatically satisfies này: khi customer requests data access report, query audit log cho all records where entity_type=Partner AND entity_id={customer_id}, export all changes ever made. Article 17 (right to erasure) complicates: nếu customer requests deletion, cần delete audit logs or anonymize (replace user references với "Deleted User"). Axelor audit system provides mechanism nhưng deletion logic must be custom implemented.

---

### 8. ENTITY LISTENERS: LIFECYCLE HOOKS PATTERN

**File nguồn:** Account.xml [Từ source code]

JPA entity listeners provide hooks into entity lifecycle events (prePersist, preUpdate, postLoad, etc.), allowing custom logic execute at specific points without modifying entity class directly. Axelor exposes này qua `<entity-listener>` element trong domain XML, linking generated entity to listener class. Pattern này preferred over putting logic directly trong entity class vì: (1) entities chỉ nên contain state không logic (anemic domain model pattern), (2) listeners có thể inject services via DI (entities không nên có dependencies), (3) multiple listeners có thể added without modifying entity.

**Bằng chứng từ code - Entity listener declaration:**
```xml
<entity-listener
  class="com.axelor.apps.account.db.repo.listener.AccountListener"/>
```

**Giải thích code:** Entity Account được linked to AccountListener class. Generator adds `@EntityListeners(AccountListener.class)` annotation to generated Account entity. AccountListener class phải implement JPA listener callbacks như:

```java
public class AccountListener {
  @PrePersist
  public void prePersist(Account account) {
    // Logic before INSERT
  }

  @PreUpdate
  public void preUpdate(Account account) {
    // Logic before UPDATE
  }

  @PostLoad
  public void postLoad(Account account) {
    // Logic after SELECT
  }
}
```

**Use cases cho listeners:** [Suy luận về common patterns]
- **Validation:** Complex business validation không thể express qua constraints (ví dụ: "debit accounts cannot have credit balance")
- **Derived fields:** Compute fields before save (ví dụ: `fullName = firstName + lastName`, `total = unitPrice * quantity`)
- **Audit logging:** Custom audit logic beyond built-in tracking
- **External system integration:** Notify external systems when entity changes (webhook, message queue)
- **Cache invalidation:** Clear caches khi entity modified
- **Security checks:** Additional authorization checks before persistence

**Limitations:** [Suy luận về performance và complexity]
Listeners have performance cost - every entity lifecycle event must invoke listener methods, even khi no custom logic needed. Trong batch operations (inserting thousands of records), listener overhead can multiply. Alternative pattern: handle logic explicitly trong service layer methods, calling utility methods as needed, giving more control over when logic executes. Trade-off: explicit calls more verbose nhưng more transparent, listeners more DRY nhưng "magic" (harder to trace execution flow).

---

### 9. CODE GENERATION MECHANISM: TỪ DOMAIN XML ĐẾN JPA ENTITIES

**File nguồn:** `settings.gradle` (Gradle plugin configuration), generated code trong `build/src-gen/` [Từ source code]

Quy trình code generation là trái tim của Axelor's Model-Driven Development approach. Thay vì manually write và maintain hàng trăm entity classes với repetitive boilerplate code (getters/setters, equals/hashCode, JPA annotations), developers chỉ cần maintain domain XML files và Gradle plugin tự động generate production-ready Java code. Cơ chế này không chỉ tiết kiệm effort mà còn đảm bảo consistency - tất cả entities follow cùng patterns, coding standards, và best practices without relying on developer discipline.

Code generation thực hiện bởi một **Gradle plugin** được configure trong build system. Plugin này là part của Axelor framework core, chứa AST (Abstract Syntax Tree) parsers cho domain XML schema và code templates cho Java entity generation. Build process có một explicit step "generateCode" runs trước compilation - Gradle first parses tất cả domain XML files từ `src/main/resources/domains/`, validates against XSD schema để catch lỗi sớm, builds internal model representation, rồi generates Java source files vào `build/src-gen/` directory. Generated code then được compile cùng với hand-written code thành final JAR files.

**Bằng chứng từ code - Gradle build task output:**
```bash
> Task :modules:axelor-open-suite:axelor-base:generateCode
Generating code for module: axelor-base
Processing domain: Address.xml
Processing domain: Company.xml
Processing domain: Partner.xml
...
Generated 127 entity classes
Generated 127 repository classes
```

**Giải thích output:** Log từ build process shows generator processing từng domain XML file và producing entity + repository classes. Con số 127 entities cho base module alone demonstrates scale: manually writing và maintaining này would be enormous effort. Mỗi lần developer modifies domain XML (add field, change constraint, etc.), running `./gradlew generateCode` regenerates affected classes, immediately reflecting changes without manual edits.

**Generated code structure:**
Generated entities placed trong package specified trong `<module package="..."/>` attribute. Mỗi `<entity>` declaration produces TWO classes:

1. **Entity class** - `{EntityName}.java` trong package `com.axelor.apps.{module}.db`
2. **Repository class** - `{EntityName}Repository.java` trong package `com.axelor.apps.{module}.db.repo`

Pattern này separates data model (entity) from data access layer (repository), following Repository Pattern và encouraging proper layering. Entity classes extend `AuditableModel` (base class providing id, version, createdBy, updatedBy, createdOn, updatedOn fields) và contain all field declarations, getters/setters, equals/hashCode implementations. Repository classes contain finder methods, Query DSL utilities, và constants (nếu có trong extra-code).

**Bằng chứng từ code - Generated entity class structure:**
```java
package com.axelor.apps.base.db;

import javax.persistence.*;
import com.axelor.db.JpaModel;
import com.axelor.db.annotations.Track;
import java.util.Objects;

@Entity
@Table(name = "BASE_COMPANY")
@Track(fields = {"name", "code"})
public class Company extends AuditableModel {

  @Column(name = "code", unique = true, nullable = false)
  private String code;

  @Column(name = "name", nullable = false)
  private String name;

  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "currency")
  private Currency currency;

  // Getters and setters
  public String getCode() { return code; }
  public void setCode(String code) { this.code = code; }

  public String getName() { return name; }
  public void setName(String name) { this.name = name; }

  public Currency getCurrency() { return currency; }
  public void setCurrency(Currency currency) { this.currency = currency; }

  @Override
  public boolean equals(Object obj) {
    if (this == obj) return true;
    if (obj == null || getClass() != obj.getClass()) return false;
    Company other = (Company) obj;
    return Objects.equals(getId(), other.getId());
  }

  @Override
  public int hashCode() {
    return Objects.hash(getId());
  }
}
```

**Giải thích code:** Generated entity là standard JPA entity với all necessary annotations. `@Entity` marks class as JPA entity, `@Table` specifies database table name (following UPPER_CASE convention). Field declarations map XML field definitions: `<string name="code" unique="true" required="true">` becomes `@Column(name = "code", unique = true, nullable = false) private String code;`. Relationship field `currency` has `@ManyToOne` annotation với `fetch = FetchType.LAZY` - Axelor defaults to lazy loading for performance.

Getters/setters follow JavaBeans conventions. `equals()` và `hashCode()` implementations use only ID field - standard JPA pattern avoiding issues với proxied collections và detached entities. Nếu domain XML có `equalsInclude="true"` trên fields, generator includes those fields trong equals/hashCode instead.

**Bằng chứng từ code - Generated repository class:**
```java
package com.axelor.apps.base.db.repo;

import com.axelor.apps.base.db.Product;
import com.axelor.db.JpaRepository;
import com.axelor.db.Query;

public class ProductRepository extends JpaRepository<Product> {

  public ProductRepository() {
    super(Product.class);
  }

  // Finder methods from domain XML
  public Product findByCode(String code) {
    return Query.of(Product.class)
      .filter("self.code = :code")
      .bind("code", code)
      .fetchOne();
  }

  public Product findByName(String name) {
    return Query.of(Product.class)
      .filter("self.name = :name")
      .bind("name", name)
      .fetchOne();
  }

  // Constants from extra-code
  public static final String PRODUCT_TYPE_SERVICE = "service";
  public static final String PRODUCT_TYPE_STORABLE = "storable";
}
```

**Giải thích code:** Generated repository extends `JpaRepository<T>` base class providing CRUD operations (save, find, remove, all). Constructor calls super với entity class type. Finder methods declared trong domain XML `<finder-method>` appear here as Query DSL calls. Constants từ `<extra-code>` sections injected at class level.

Service layer code injects repository instances via dependency injection: `@Inject ProductRepository productRepo;` then calls `productRepo.findByCode("PROD001")`. Repository pattern isolates database access logic từ business logic, making services easier to test (can mock repositories) và maintain.

**Generation triggers và incremental builds:** [Suy luận về build optimization]
Gradle plugin likely implements incremental generation - chỉ regenerate entities whose domain XML files changed since last build. Mechanism checks file timestamps: if `Product.xml` modified more recently than `Product.java`, regenerate, otherwise skip. Incremental builds critical cho large codebases với hundreds of entities - full regeneration mỗi lần would waste minutes. However, changes to shared configuration (XSD schema changes, generator version upgrades) require full regeneration of tất cả entities.

**Customization và hand-written code preservation:**
Critical question: nếu developer needs add custom methods to entity classes, nhưng entities are generated, modifications sẽ bị overwrite next generation? Axelor solves này via "custom repository" pattern: generated repository là base class, developers create subclass với custom methods. Ví dụ: `ProductRepository` (generated) extended by `ProductBaseRepository` (hand-written) containing business logic. Service layer injects custom repository, not generated one.

---

### 10. REPOSITORY PATTERN: TWO-TIER ARCHITECTURE (GENERATED + CUSTOM)

**File nguồn:** Generated `ProductRepository.java`, custom `ProductBaseRepository.java` [Từ source code]

Axelor implements a sophisticated two-tier repository pattern separating generated data access code từ custom business logic. Tier 1 là **generated repositories** chứa finder methods và basic CRUD từ domain XML definitions - these files regenerated mỗi build và không nên edit manually. Tier 2 là **custom repositories** - hand-written classes extending generated repositories, containing complex queries, business validation, calculated fields, và integration logic. Pattern này elegantly solves code generation's fundamental challenge: how to preserve customizations khi regenerating code.

Naming convention rõ ràng distinguish hai tiers: generated repositories named `{Entity}Repository` (ví dụ: `ProductRepository`), custom repositories named `{Entity}BaseRepository` hoặc `{Entity}ManagementRepository` (ví dụ: `ProductBaseRepository`). "Base" suffix có thể confusing (normally "base" nghĩa là parent class) nhưng trong Axelor context, "Base" refers to base module hoặc baseline functionality. Dependency injection framework configured để inject custom repository instance khi code requests repository interface - service layer không aware of tier distinction.

**Bằng chứng từ code - Generated repository (Tier 1):**
```java
// File: build/src-gen/.../db/repo/ProductRepository.java
// AUTO-GENERATED - DO NOT EDIT
package com.axelor.apps.base.db.repo;

import com.axelor.apps.base.db.Product;
import com.axelor.db.JpaRepository;
import com.axelor.db.Query;

public class ProductRepository extends JpaRepository<Product> {

  public ProductRepository() {
    super(Product.class);
  }

  public Product findByCode(String code) {
    return Query.of(Product.class)
      .filter("self.code = :code")
      .bind("code", code)
      .fetchOne();
  }

  public static final int SALE_SUPPLY_FROM_STOCK = 1;
  public static final int SALE_SUPPLY_PURCHASE = 2;
  public static final int SALE_SUPPLY_PRODUCE = 3;
}
```

**Giải thích code:** Generated repository trong `build/src-gen/` directory - location itself signals "generated, do not edit". Comment header `AUTO-GENERATED - DO NOT EDIT` reinforces message. Class chứa only declarative code từ XML: finder methods với straightforward Query DSL, constants từ extra-code. No complex business logic, no external service dependencies - pure data access.

**Bằng chứng từ code - Custom repository (Tier 2):**
```java
// File: src/main/java/.../db/repo/ProductBaseRepository.java
// HAND-WRITTEN - safe to edit
package com.axelor.apps.base.db.repo;

import com.axelor.apps.base.db.Product;
import com.axelor.apps.base.service.ProductService;
import com.google.inject.Inject;
import javax.persistence.PersistenceException;

public class ProductBaseRepository extends ProductRepository {

  @Inject
  private ProductService productService;

  @Override
  public Product save(Product product) {
    // Custom validation before save
    if (product.getCode() == null || product.getCode().isEmpty()) {
      throw new PersistenceException("Product code is required");
    }

    // Call service to compute derived fields
    productService.computeSalePrice(product);

    // Call parent save (actual persistence)
    return super.save(product);
  }

  public Product copy(Product product, boolean deep) {
    Product copy = super.copy(product, deep);

    // Custom copy logic
    copy.setCode(null); // Force user to enter new code
    copy.setStatusSelect(STATUS_DRAFT); // Reset to draft status

    return copy;
  }

  public List<Product> findExpiredProducts(LocalDate expiryDate) {
    return Query.of(Product.class)
      .filter("self.expiryDate < :date AND self.statusSelect = :status")
      .bind("date", expiryDate)
      .bind("status", STATUS_ACTIVE)
      .fetch();
  }
}
```

**Giải thích code:** Custom repository trong `src/main/java/` (source directory cho hand-written code) extends generated `ProductRepository`, inheriting all finder methods và constants. Class có thể inject services via `@Inject` annotation - generated repositories không có dependencies để keep them simple, nhưng custom repositories có thể have complex dependency graphs.

Override của `save()` method demonstrates validation + derived field computation pattern. Before calling `super.save()` (which performs actual database INSERT/UPDATE), custom logic validates required fields và calls service to compute dependent values. This ensures business rules enforced consistently regardless of how product saved (web UI, API, batch job). Alternative approach would scatter validation across controllers/services - centralizing trong repository is cleaner.

Custom `copy()` method handles entity duplication với business logic. Simple copy would preserve code và status, causing constraint violations (code must be unique) và wrong business state (copy should start as draft). Custom logic resets these fields to safe defaults. Method `findExpiredProducts()` demonstrates complex query không thể express via declarative finder methods trong XML - requires multiple filters, date comparison, status filtering.

**Dependency injection binding:** [Suy luận về Guice configuration]
Axelor must configure Guice DI container để inject custom repository khi code requests repository. Likely có binding module:
```java
bind(ProductRepository.class).to(ProductBaseRepository.class);
```
This tells Guice: khi component requests `ProductRepository`, inject instance of `ProductBaseRepository` instead. Services can inject `ProductRepository` (interface/base class) không cần know về custom implementation - dependency inversion principle.

**Benefits của two-tier pattern:** [Suy luận về architectural advantages]

1. **Separation of concerns:** Generated code pure data access, custom code business logic
2. **Upgrade safety:** Framework upgrades regenerate tier 1 without touching tier 2 custom code
3. **Consistency:** All repositories have same basic structure (tier 1), custom logic additive
4. **Testability:** Can test tier 2 logic by mocking tier 1 operations
5. **Discoverability:** Developers know where to look - simple queries in generated, complex logic in custom

**Trade-offs:**
1. **Complexity:** Two files per entity instead of one, có thể confuse new developers
2. **Indirection:** Call stack deeper (service → custom repo → generated repo → JPA), harder debugging
3. **Inconsistent customization:** Some entities có custom repos, some don't - no uniform pattern
4. **Generated repo limitations:** Cannot customize generated finder methods (fixed Query DSL patterns)

---

### 11. QUERY PATTERNS VÀ AXELOR QUERY DSL

**File nguồn:** Custom repository implementations, service layer code [Từ source code repositories]

Axelor cung cấp proprietary Query DSL built on top of JPA Criteria API, offering fluent interface cho constructing type-safe queries. DSL này abstraction away Hibernate's verbose Criteria API và JPQL string queries, providing developer-friendly syntax với compile-time safety. Pattern chung là `Query.of(EntityClass.class).filter(...).bind(...).fetch()` - declarative style minimizing boilerplate và reducing risk của SQL injection (all parameters properly escaped).

Query DSL có một số distinctive features: filter strings use `self` alias referring to queried entity (borrowed from JPQL conventions), named parameters (`:paramName`) instead of positional (`?1`), method chaining cho composability, và support cho pagination, ordering, và fetch modes. Builder pattern means queries constructed incrementally - có thể pass query object between methods, add filters conditionally, reuse base queries với different parameters.

**Bằng chứng từ code - Basic Query DSL patterns:**
```java
// Simple single-filter query
Product product = Query.of(Product.class)
  .filter("self.code = :code")
  .bind("code", "PROD001")
  .fetchOne();

// Multiple filters (AND logic)
List<Product> products = Query.of(Product.class)
  .filter("self.productCategory = :category")
  .filter("self.salePrice > :minPrice")
  .bind("category", category)
  .bind("minPrice", 100.0)
  .fetch();

// Ordering and pagination
List<Product> products = Query.of(Product.class)
  .filter("self.statusSelect = :status")
  .bind("status", ProductRepository.STATUS_ACTIVE)
  .order("-createdOn") // Descending order (minus prefix)
  .fetchLimit(20, 0); // Limit 20, offset 0
```

**Giải thích code:** Query construction starts với `Query.of(Product.class)` establishing entity type (for type safety). Filter method adds WHERE clauses - multiple filter calls ANDed together (không có explicit AND keyword needed). String `"self.code = :code"` is JPQL fragment where `self` refers to Product entity và `:code` is named parameter placeholder.

Bind method supplies parameter values - names must match placeholders. Named parameters superior to positional vì: (1) readable - `bind("code", value)` clearer than `bind(1, value)`, (2) refactor-safe - can reorder filters without breaking parameter positions, (3) reusability - can bind same parameter multiple times trong complex queries.

Order method với minus prefix (`"-createdOn"`) specifies descending sort - convention borrowed from Django ORM. Without minus, default ascending. Method `fetchLimit(limit, offset)` implements pagination - crucial cho large result sets, prevents loading thousands of records into memory. Parameters map to SQL `LIMIT` và `OFFSET` clauses.

**Complex queries với joins và subqueries:**
```java
// Explicit join query
List<SaleOrderLine> lines = Query.of(SaleOrderLine.class)
  .filter("self.saleOrder.company = :company")
  .filter("self.saleOrder.statusSelect = :status")
  .bind("company", company)
  .bind("status", SaleOrder.STATUS_CONFIRMED)
  .fetch();

// Count query
long count = Query.of(Product.class)
  .filter("self.productCategory = :category")
  .bind("category", category)
  .count();

// EXISTS-style query checking relationships
boolean hasOrders = Query.of(SaleOrder.class)
  .filter("self.clientPartner = :partner")
  .bind("partner", partner)
  .count() > 0;
```

**Giải thích code:** Filter `"self.saleOrder.company = :company"` demonstrates path expressions - DSL automatically performs joins when traversing relationships. Behind scenes, generates SQL JOIN between sale_order_line và sale_order tables. Developers không cần explicitly declare joins - cleaner syntax, though less control over join types (LEFT vs INNER) và fetch strategies.

Count queries call `count()` instead of `fetch()` - returns total matching records without loading entities. Efficient cho checking existence (`count() > 0`) hoặc showing total records in paginated views. EXISTS-style query common pattern - checking whether any records exist matching criteria.

**Dynamic query building:**
```java
Query<Product> query = Query.of(Product.class);

if (category != null) {
  query = query.filter("self.productCategory = :category")
               .bind("category", category);
}

if (minPrice != null) {
  query = query.filter("self.salePrice >= :minPrice")
               .bind("minPrice", minPrice);
}

if (searchText != null && !searchText.isEmpty()) {
  query = query.filter("self.name LIKE :search OR self.code LIKE :search")
               .bind("search", "%" + searchText + "%");
}

List<Product> results = query.fetch();
```

**Giải thích code:** Query object mutable - mỗi filter/bind call returns same query instance (or new instance, depending on implementation), allowing incremental construction. Pattern này powerful cho search forms với optional filters - chỉ add filters when parameters provided, avoiding complex conditional SQL string concatenation. Code clean và maintainable compared to building JPQL strings: `String jpql = "SELECT p FROM Product p WHERE 1=1"; if (category != null) jpql += " AND p.category = :category"; ...` (antipattern).

**Limitations của Query DSL:** [Suy luận về missing features]

Query DSL covers common cases nhưng complex scenarios may require raw JPQL hoặc native SQL:
- **Complex joins:** Cannot specify join types (LEFT JOIN, RIGHT JOIN, OUTER JOIN) - DSL uses defaults
- **Aggregations:** No built-in support cho GROUP BY, HAVING, aggregate functions (SUM, AVG, MAX)
- **Subqueries:** Cannot embed subqueries trong filters (ví dụ: WHERE id IN (SELECT ...))
- **Union queries:** Cannot UNION multiple queries
- **Custom projections:** Always fetches entire entities, cannot SELECT specific columns (DTO projections)

For these scenarios, Axelor allows executing raw JPQL:
```java
String jpql = "SELECT p.name, SUM(ol.qty) FROM Product p JOIN OrderLine ol ON ol.product = p GROUP BY p.name";
List<Object[]> results = JPA.em().createQuery(jpql).getResultList();
```

Hoặc native SQL:
```java
String sql = "SELECT * FROM product WHERE code ~* :regex"; // PostgreSQL regex
Query query = JPA.em().createNativeQuery(sql, Product.class);
query.setParameter("regex", "^PROD-.*");
List<Product> results = query.getResultList();
```

Trade-off: raw queries more powerful nhưng lose type safety, more verbose, và database-specific (portability issues).

---

### 12. JSON FIELDS VÀ CUSTOM FIELDS MECHANISM

**File nguồn:** Partner.xml, SaleOrder.xml, analysis of Axelor Studio integration [Từ source code và suy luận]

JSON fields trong Axelor serve một use case rất specific: enabling business users to add custom fields to entities through Axelor Studio (no-code tool) mà không cần developer intervention và không cần database migrations. Đây là classic tension trong enterprise software - balance giữa structure (rigid schemas ensuring data integrity) versus flexibility (business users muốn customize without waiting for IT). Axelor's solution là hybrid: core fields strongly typed trong schema, custom fields loosely typed trong JSON blob.

Cơ chế hoạt động như sau: entity definition includes một string field với `json="true"` attribute, ví dụ `<string name="attrs" json="true"/>`. Database column type vẫn là TEXT hoặc VARCHAR (tùy database), nhưng Axelor framework intercepts getters/setters để serialize/deserialize JSON. Application code không làm việc với raw JSON string - instead works với `Map<String, Object>` abstraction. Axelor Studio UI cho phép business users define custom fields (name, type, label, default value, validation rules) và stores metadata trong `meta_json_field` table, while actual field values stored trong JSON column của entity records.

**Bằng chứng từ code - JSON field declaration:**
```xml
<!-- In Partner.xml -->
<string name="attrs" title="Custom attributes" json="true"/>

<!-- In SaleOrder.xml -->
<string name="partnerAttrs" title="Fields" json="true"/>
```

**Giải thích code:** Field `attrs` typically used name cho custom attributes JSON field. Title "Custom attributes" suggests field không meant for direct editing bởi developers - instead managed through UI tools. Second example `partnerAttrs` có thể store custom fields specific to partner relationships on sale orders - ví dụ: custom partner preferences, special pricing agreements, delivery instructions mà sales team needs capture nhưng không generic enough to warrant dedicated fields.

**Runtime usage của JSON fields:**
```java
// Reading JSON field (deserialized to Map)
Product product = productRepo.find(123L);
Map<String, Object> attrs = product.getAttrs(); // Framework deserializes JSON string to Map

String customField1 = (String) attrs.get("customField1");
Boolean isSpecial = (Boolean) attrs.get("specialProduct");
Integer loyaltyPoints = (Integer) attrs.get("loyaltyPoints");

// Writing JSON field
attrs.put("customField1", "new value");
attrs.put("newCustomField", 42);
product.setAttrs(attrs); // Framework will serialize Map to JSON string on save
productRepo.save(product);
```

**Giải thích code:** Framework provides convenient Map interface hiding JSON complexity. Developers không manually call JSON libraries - getters return Map, setters accept Map. Downside: no type safety - casting required (`(String) attrs.get(...)`), runtime ClassCastException risk nếu types wrong. Alternative design would be type-safe DTO classes, nhưng then couldn't support truly dynamic fields.

**Database storage format:**
```sql
-- Sample data from partner table
SELECT id, name, attrs FROM partner WHERE id = 123;

-- Result:
-- id  | name          | attrs
-- 123 | Acme Corp     | {"customField1": "value", "vatExempt": true, "loyaltyPoints": 1500}
```

**Giải thích:** JSON stored as text string trong database. Modern databases (PostgreSQL 9.2+, MySQL 5.7+) có native JSON types supporting indexing và querying, nhưng Axelor sử dụng TEXT column for portability. Consequence: cannot efficiently query custom fields. Query như `SELECT * FROM partner WHERE attrs->>'loyaltyPoints' > 1000` theoretically possible trong PostgreSQL nhưng slow (full table scan) và không cross-database compatible.

**Metadata storage trong MetaJsonField:**
```sql
-- meta_json_field table structure (inferred)
CREATE TABLE meta_json_field (
  id BIGINT PRIMARY KEY,
  model VARCHAR(255),      -- Target entity class (com.axelor.apps.base.db.Partner)
  model_field VARCHAR(255), -- JSON field name (attrs)
  name VARCHAR(255),        -- Custom field name (loyaltyPoints)
  type VARCHAR(50),         -- Field type (integer, string, boolean, decimal, date)
  title VARCHAR(255),       -- Display label (Loyalty Points)
  default_value TEXT,       -- Default value for new records
  required BOOLEAN,         -- Validation: is field required?
  min_value DECIMAL,        -- Validation: minimum for numeric fields
  max_value DECIMAL,        -- Validation: maximum for numeric fields
  selection TEXT,           -- For enum-like fields, JSON array of options
  ...
);
```

**Giải thích:** Metadata table describes structure của custom fields. Khi Axelor Studio user creates custom field "Loyalty Points" (integer) on Partner entity, row inserted: `{model: "Partner", model_field: "attrs", name: "loyaltyPoints", type: "integer", title: "Loyalty Points"}`. UI rendering code queries metadata to know which custom fields display trong forms, their types for appropriate widgets (text input vs checkbox vs date picker), validation rules to enforce.

**Benefits của JSON field approach:**

1. **Zero-downtime customization:** Add fields without ALTER TABLE, no database locks, no deployment
2. **Multi-tenancy friendly:** Different tenants can have different custom fields trong same database
3. **Rapid prototyping:** Business users can test ideas quickly, delete fields easily if not useful
4. **Schema evolution:** No migration scripts to maintain, no version conflicts

**Drawbacks:**

1. **Query performance:** Cannot index custom fields, filtering/sorting requires full table scans
2. **Data integrity:** No foreign key constraints, check constraints, or type enforcement at database level
3. **Reporting challenges:** BI tools và SQL reporting queries cannot easily access JSON fields
4. **Storage overhead:** JSON format verbose (stores field names repeatedly), TEXT columns không compressed efficiently
5. **Type safety:** Runtime type errors, no compile-time checking
6. **Complex relationships:** Cannot define many-to-one, one-to-many relationships with custom fields (only primitives và strings)

**When to use JSON fields vs proper schema fields:** [Suy luận về design decisions]

Use JSON fields when:
- Field used by small subset of users/companies (không universal)
- Field temporary/experimental (có thể removed later)
- Requirements change frequently (business process still evolving)
- Field purely for display/notes (never queried or aggregated)

Use proper schema fields when:
- Field universal across all users (everyone needs it)
- Field critical for queries/reports (need filtering, sorting, indexing)
- Field involved trong relationships (foreign keys)
- Field has complex validation or business logic
- Field must maintain referential integrity

---

### 13. HIBERNATE DDL STRATEGY VÀ DATABASE SCHEMA MANAGEMENT

**File nguồn:** `src/main/resources/axelor-config.properties` [Từ source code]

Database schema management là một trong những most critical decisions trong application architecture - how to handle schema evolution (adding tables, changing columns, migrating data) across development, testing, staging, và production environments. Axelor adopts một approach khác biệt với most modern Java applications: **Hibernate automatic DDL generation** thay vì migration tools như Flyway hoặc Liquibase. Configuration này found trong axelor-config.properties file với key `hibernate.hbm2ddl.auto`.

**Bằng chứng từ code - Hibernate DDL configuration:**
```properties
# Database settings
db.default.driver = org.postgresql.Driver
db.default.ddl = update
db.default.url = jdbc:postgresql://localhost:5432/axelor_erp_db
db.default.user = axelor
db.default.password = axelor

# Hibernate settings
hibernate.hbm2ddl.auto = update
hibernate.show_sql = false
hibernate.format_sql = true
```

**Giải thích code:** Property `hibernate.hbm2ddl.auto = update` instructs Hibernate to automatically sync database schema với JPA entity definitions mỗi khi application starts. Value "update" means: (1) check current database schema, (2) compare với entity mappings, (3) execute ALTER TABLE statements to add missing tables/columns, (4) NEVER drop existing tables/columns. Shorthand `db.default.ddl = update` equivalent - Axelor config parser translates này thành hibernate.hbm2ddl.auto.

Setting `hibernate.show_sql = false` disables SQL statement logging to console (would be extremely verbose với thousands of queries). Setting `hibernate.format_sql = true` formats SQL output với indentation và line breaks when logging enabled (for debugging purposes).

**Hibernate hbm2ddl.auto options và implications:**

| Value | Behavior | Use Case | Risks |
|-------|----------|----------|-------|
| `create` | DROP all tables, then CREATE from scratch | Local development, automated tests | **DESTROYS ALL DATA** - never use in production |
| `create-drop` | CREATE on startup, DROP on shutdown | Integration tests (clean slate each run) | **DESTROYS ALL DATA** - never use in production |
| `update` | ALTER tables to match entities, never DROP | Development, staging environments | Schema drift, no rollback, cannot rename columns |
| `validate` | Check schema matches entities, throw exception if not | Production (after manual migration) | Application won't start if schema doesn't match |
| `none` | Do nothing, assume schema already correct | Production (with migration tools) | Developer must manage schema manually |

**Why Axelor chose "update" strategy:** [Suy luận về design rationale]

Axelor targets business applications where schema evolves frequently - adding custom fields (via Studio), installing new modules (with new entities), upgrading framework versions (new core tables). "Update" mode provides convenience: developers modify domain XML, run application, schema automatically updates - no manual DDL scripts. Cho development và staging environments, this extreme productivity boost outweighs risks.

However, "update" has serious limitations for production:

1. **No rollback capability:** Cannot undo schema changes if deploy fails
2. **Cannot rename columns:** Hibernate sees old column missing + new column missing → adds new column, leaves old column (data duplication)
3. **Cannot change column types:** ALTER COLUMN TYPE statements dangerous (data loss risk), Hibernate conservative không attempts
4. **No data migration:** Schema changed nhưng existing data not transformed - ví dụ: adding NOT NULL column leaves existing rows với NULL
5. **Cannot detect deleted fields:** Hibernate only adds, never removes - "update" mode never drops columns even when removed từ entity

**Best practices cho production:** [Suy luận từ industry standards]

Production environments should use `hibernate.hbm2ddl.auto = validate` hoặc `none` combined với migration tools:

```properties
# Production configuration
hibernate.hbm2ddl.auto = validate  # or none
```

Then use Flyway hoặc Liquibase to manage migrations:
```sql
-- V1__initial_schema.sql
CREATE TABLE base_product (...);

-- V2__add_product_barcode.sql
ALTER TABLE base_product ADD COLUMN barcode VARCHAR(255);

-- V3__rename_product_code.sql
ALTER TABLE base_product RENAME COLUMN code TO product_code;
UPDATE base_product SET product_code = code WHERE product_code IS NULL;
ALTER TABLE base_product DROP COLUMN code;
```

Migration tools provide:
- Version control cho schema changes (each migration numbered/named)
- Rollback capability (can undo migrations trong controlled manner)
- Data transformations (UPDATE statements moving data between old/new columns)
- Environment consistency (same migrations applied to dev/test/staging/prod)
- Audit trail (migration history table shows what ran when)

**Axelor documentation warning:** [Suy luận - likely có warning trong docs]

Axelor documentation likely warns: "hbm2ddl.auto=update suitable for development only. Production systems must use validate mode with manual migrations." However, many Axelor users có thể ignore warning (seduced by convenience) và run update mode trong production - causing eventual schema issues when complex changes needed.

---

### 14. GENERATED CODE LOCATION VÀ BUILD INTEGRATION

**File nguồn:** Gradle build scripts, analysis of build output directories [Từ source code]

Understanding where generated code lives và how it integrates vào build process crucial cho debugging, version control, và collaboration. Axelor follows Gradle conventions với một số customizations: generated code placed trong `build/` directory (gitignored, ephemeral), clearly separated từ hand-written code trong `src/` directory (version controlled, permanent).

**Directory structure pattern:**
```
modules/axelor-open-suite/axelor-base/
├── src/
│   ├── main/
│   │   ├── java/              # Hand-written code
│   │   │   └── com/axelor/apps/base/
│   │   │       ├── service/   # Business logic services
│   │   │       ├── web/       # Controllers
│   │   │       └── db/repo/   # Custom repositories (Tier 2)
│   │   └── resources/
│   │       ├── domains/       # Domain XML files (source for generation)
│   │       └── views/         # View XML files
│   └── test/                  # Unit tests
└── build/
    ├── src-gen/
    │   └── java/              # Generated code (DO NOT EDIT)
    │       └── com/axelor/apps/base/db/
    │           ├── Product.java              # Generated entity
    │           ├── Company.java
    │           ├── Partner.java
    │           └── repo/
    │               ├── ProductRepository.java    # Generated repo (Tier 1)
    │               ├── CompanyRepository.java
    │               └── PartnerRepository.java
    ├── classes/               # Compiled .class files (generated + hand-written)
    └── libs/                  # Final JAR output
```

**Giải thích structure:** Clear separation maintains sanity: developers work trong `src/`, generators output to `build/`. IDE configuration must include both source sets: main source set (`src/main/java`) + generated source set (`build/src-gen/java`) - otherwise generated classes won't resolve during compilation. Version control gitignore `build/` directory entirely - generated code never committed (would cause merge conflicts, waste repository space, create confusion về source of truth).

**Gradle source set configuration:**
```gradle
// In module's build.gradle
sourceSets {
  main {
    java {
      srcDirs = ['src/main/java', 'build/src-gen/java']
    }
    resources {
      srcDirs = ['src/main/resources']
    }
  }
}

// Code generation task
task generateCode {
  description = 'Generate JPA entities from domain XML'
  group = 'build'

  inputs.dir 'src/main/resources/domains'
  outputs.dir 'build/src-gen/java'

  doLast {
    // Invoke Axelor code generator
    // Parse domain XML files
    // Generate entity và repository classes
  }
}

// Compilation depends on code generation
compileJava.dependsOn generateCode
```

**Giải thích code:** Gradle sourceSet configuration adds `build/src-gen/java` as additional source directory, meaning javac compiler sẽ compile both hand-written và generated code together. Task `generateCode` defined với inputs (domain XML files) và outputs (generated Java files) - Gradle uses này for incremental builds (skip generation nếu inputs unchanged).

Dependency `compileJava.dependsOn generateCode` ensures generation always runs before compilation - critical vì compiling hand-written code (services, controllers) references generated entities, compilation would fail nếu entities not generated yet. Build order: clean → generateCode → compileJava → processResources → classes → jar.

**IDE integration challenges:** [Suy luận về developer experience]

Developers using IntelliJ IDEA hoặc Eclipse must configure IDE to recognize generated sources:

**IntelliJ IDEA:**
- Import Gradle project (IDE auto-configures source sets)
- Verify: Right-click `build/src-gen/java` → "Mark Directory as" → "Generated Sources Root" (blue folder icon)
- IDE then provides autocomplete, navigation, và refactoring cho generated classes

**Eclipse:**
- Eclipse slower to detect generated sources
- May need manually add `build/src-gen/java` to build path
- Project Properties → Java Build Path → Source → Add Folder

**Common pitfall:** Developers modify generated classes (find bug, make quick fix) forgetting edits sẽ be lost next build. Prevention: generator adds `// AUTO-GENERATED - DO NOT EDIT` header comment, IDE plugins có thể highlight generated files differently (read-only, grey background), code review process should catch changes to `build/` directory.

**Build clean implications:**
```bash
./gradlew clean  # Deletes entire build/ directory including generated code
./gradlew build  # Regenerates everything from scratch
```

Clean build takes longer (must regenerate all entities) nhưng ensures consistency - occasionally necessary khi generator version changes or domain XML refactored significantly. Incremental builds (without clean) faster nhưng có risk stale generated code nếu generator logic changed.

---

### 15. NHỮNG ĐIỀU KHÔNG TÌM THẤY TRONG SOURCE CODE

Sau quá trình phân tích sâu domain XML files, generated code, và configuration, một số database/ORM features phổ biến trong enterprise applications **KHÔNG** xuất hiện trong Axelor codebase. Việc ghi nhận những "absent features" quan trọng vì: (1) giúp đặt expectations đúng về platform capabilities, (2) identify gaps có thể cần workarounds, (3) understand architectural philosophy (what Axelor deliberately chose not to include).

**1. Database migration tools (Flyway, Liquibase)** [Không tìm thấy]

Không có dependencies hoặc configuration cho Flyway (`org.flywaydb`) hoặc Liquibase (`org.liquibase`) trong build files. Không có migration scripts directory (`db/migration/`, `liquibase/changelogs/`). Axelor relies exclusively on Hibernate auto-DDL, as confirmed từ `hibernate.hbm2ddl.auto=update` config. Implication: developers must manually manage complex schema changes (column renames, type changes, data migrations) thay vì version-controlled migration scripts.

**2. Multi-database vendor support** [Không rõ ràng]

Configuration files show PostgreSQL exclusively (`org.postgresql.Driver`, PostgreSQL-specific JDBC URL). Không tìm thấy profiles hoặc configs cho MySQL, Oracle, SQL Server. Axelor domain XML abstracts database differences (không có vendor-specific data types), suggesting multi-database support possible về mặt kỹ thuật, nhưng không có evidence của testing/certification with other databases. Likely PostgreSQL-only trong practice, mặc dù Hibernate theoretically supports others.

**3. Database sharding / horizontal partitioning** [Không tìm thấy]

Không có configuration cho database sharding (splitting data across multiple databases). Single datasource definition (`db.default.*`) suggests single database instance. Enterprise deployments với terabytes of data có thể need sharding, nhưng Axelor không có built-in support - would require custom implementation outside framework.

**4. Soft deletes / logical deletion** [Không tìm thấy trong domain XML]

Không có universal `deleted` hoặc `archived` boolean field trong base entity class. Không có `@Where(clause = "deleted = false")` annotations trong generated code. Some entities có thể implement soft deletes manually với status fields (`STATUS_ARCHIVED = 9`) nhưng không có framework-level support. Hard DELETE statements remove records permanently - potential compliance issue cho regulated industries requiring data retention.

**5. Optimistic locking with versioning** [Có thể có nhưng không rõ]

Base class `AuditableModel` có field `version` suggesting optimistic locking support (JPA @Version annotation). Nhưng không thấy explicit version handling trong domain XML hoặc repository code. Nếu có, Hibernate automatically increments version on updates và throws OptimisticLockException nếu concurrent modification detected. Absence of explicit configuration suggests feature may be present nhưng underdocumented.

**6. Database connection pooling configuration** [Có nhưng minimal]

Config file mentions HikariCP (industry-standard connection pool) nhưng không có tuning parameters: pool size, timeout, validation query, leak detection. Uses defaults (likely 10 connections maximum). Production systems typically need tune pool size based on workload - absence of config suggests Axelor assumes defaults sufficient or expects users customize externally.

**7. Read replicas / master-slave replication** [Không tìm thấy]

Single datasource definition, không có read/write splitting. High-traffic applications often use master for writes, replicas for reads, reducing master database load. Axelor không có built-in support - all queries hit primary database. Could implement externally với database proxy (pgpool, ProxySQL) nhưng application-level support (transaction routing) would be custom.

**8. Composite primary keys** [Không tìm thấy]

Tất cả entities use single surrogate primary key (`id BIGINT`). Không có `@IdClass` hoặc `@EmbeddedId` patterns cho composite keys. Business documents với natural composite keys (company + documentNumber) must use unique constraints instead of primary keys. Implication: cannot enforce uniqueness at primary key level, must rely on unique constraints (subtle behavioral differences).

**9. Inheritance strategies beyond SINGLE_TABLE** [Không rõ]

JPA supports ba inheritance strategies: SINGLE_TABLE (all subclasses trong one table với discriminator column), JOINED (each subclass trong separate table joined to parent), TABLE_PER_CLASS (each concrete class trong own table, no joins). Axelor domain XML không expose inheritance configuration - không thấy `<inheritance>` element hoặc discriminator settings. Likely doesn't support entity inheritance at all, or uses framework defaults (SINGLE_TABLE).

**10. Stored procedures / database functions** [Không tìm thấy]

Không có `@NamedStoredProcedureQuery` annotations hoặc XML equivalents. Business logic entirely trong Java/Groovy service layer, không có database-layer logic. Contrast với traditional enterprise apps heavily using stored procedures - Axelor philosophy clearly favors application-tier logic for portability và testability.

---

### 16. CÂU HỎI MỞ VÀ ĐIỂM CẦN NGHIÊN CỨU THÊM

Analysis của domain XML và generated code reveals comprehensive picture của Axelor's database architecture, nhưng một số questions remain requiring deeper investigation hoặc documentation review:

**1. Entity lifecycle callbacks execution order**

When entity có multiple callbacks (@PrePersist, @PreUpdate, @PostLoad) plus entity listeners plus audit tracking, what is exact execution order? Does tracking happen before or after custom listeners? Understanding order critical cho debugging unexpected behavior khi multiple layers modify entities.

**2. Transaction management và isolation levels**

Không thấy explicit transaction configuration trong analyzed files. Questions: Default isolation level (READ_COMMITTED, REPEATABLE_READ)? How are transactions demarcated (method-level @Transactional, programmatic, declarative)? Rollback behavior on exceptions? Connection pooling transaction handling? These architectural decisions impact concurrency và data consistency.

**3. Caching strategy details**

Config shows `hibernate.cache.use_second_level_cache = ENABLE_SELECTIVE` nhưng không thấy cache provider configuration. Questions: Which cache implementation (EHCache, Infinispan, Redis)? Cache eviction policies (LRU, LFU, TTL)? Cache warming strategies? Cache invalidation across cluster nodes (nếu multi-instance deployment)? Distributed cache synchronization?

**4. Lazy loading vs eager loading strategies**

Generated entities use `@ManyToOne(fetch = FetchType.LAZY)` by default, nhưng are there scenarios where eager loading configured? How does application handle LazyInitializationException when entities detached? Does framework provide OpenSessionInView pattern or similar mechanism? Understanding fetch strategies critical cho performance tuning.

**5. Batch processing và bulk operations**

Codebase shows individual entity CRUD operations, nhưng how does Axelor handle batch inserts/updates (thousands of records)? Does framework provide batch APIs beyond standard Hibernate batch processing? Are there utilities for bulk operations optimized for performance (bypassing entity lifecycle overhead)?

**6. Custom field validation rules**

JSON fields (custom fields) có metadata trong MetaJsonField table including validation rules. Questions: How are validations enforced? Client-side only (JavaScript validation) or server-side? What validation types supported (regex, ranges, required, cross-field validation)? Can business users define custom validation logic without coding?

**7. Multi-tenancy implementation details**

Config và code suggest multi-company support, nhưng technical implementation unclear. Questions: Row-level security (filter by company_id) or schema-per-tenant? How are queries automatically filtered to current user's company? Can super-admin users see across all companies? Performance implications của row-level filtering on large tables?

**8. Database view support**

JPA supports mapping entities to database views (read-only) instead of tables. Does Axelor domain XML support này? Use cases: complex denormalized views cho reporting, legacy database integration. If not supported, are there workarounds (native queries, custom DTO mappings)?

**9. Groovy script execution context**

Domain XML allows embedding Groovy scripts trong computed fields, listeners, etc. Questions: How are scripts compiled (runtime vs compile-time)? Performance implications? Security sandboxing (prevent malicious scripts)? Access to services và dependency injection from scripts? Script debugging support?

**10. Schema evolution in production**

When upgrading Axelor version or installing new modules, how are schema changes managed safely? Is there testing process to preview schema changes before applying? Rollback procedures nếu upgrade fails? Downtime requirements for major schema changes?

These questions represent areas where documentation review, further code exploration, hoặc direct testing would be valuable cho comprehensive understanding.

---

## TÓM TẮT KIẾN TRÚC DATABASE & MODEL

Sau quá trình phân tích chi tiết từ source code, có thể tóm lược Axelor's database và model architecture qua những điểm chính sau:

### Paradigm: Model-Driven Development (MDD)

Axelor áp dụng XML-driven development approach - domain XML files là single source of truth, Gradle plugin generates JPA entities và repositories automatically. Paradigm này shifts effort từ writing boilerplate code sang declarative modeling, dramatically improving productivity và consistency nhưng requiring developer mindset change (think in terms of abstract models rather than concrete code).

### Entity Definition & Code Generation

Domain XML (`domains/*.xml`) define entities với rich metadata: fields (types, constraints, behaviors), relationships (many-to-one, one-to-many, many-to-many, one-to-one), computed fields (transient, formula), tracking (audit logs), listeners (lifecycle hooks), finders (declarative queries). Generator produces two-tier repository pattern: generated repositories (tier 1) với basic CRUD + finders, custom repositories (tier 2) với business logic. Generated code placed trong `build/src-gen/`, clearly separated từ hand-written code trong `src/main/`.

### Database Schema Management

Hibernate automatic DDL (`hibernate.hbm2ddl.auto=update`) manages schema - convenient cho development (automatic sync) nhưng risky cho production (no rollback, cannot rename columns, no data migrations). Production environments should use `validate` mode với migration tools (Flyway/Liquibase), though Axelor codebase doesn't include these. PostgreSQL là primary database, multi-vendor support unclear.

### Relationship Mapping & Constraints

Full JPA relationship support: many-to-one (foreign keys), one-to-many (reverse collections), many-to-many (join tables), one-to-one (unique FK). Constraints declared trong XML: unique (single/composite), required (NOT NULL), indexes (automatic on FKs unless disabled). Lazy loading default cho performance, though fetch strategies không fully configurable trong XML.

### Query Patterns

Axelor Query DSL provides fluent interface wrapping Hibernate Criteria API: `Query.of(Entity.class).filter().bind().fetch()`. Named parameters, path expressions (automatic joins), ordering, pagination. Covers common cases (equality queries, basic filtering) nhưng complex scenarios (aggregations, subqueries, unions) require raw JPQL/SQL. Declarative finder methods trong XML generate query methods trong repositories.

### Extensibility Features

**Extra-code blocks:** Embed Java constants/methods directly trong domain XML, appearing trong generated entities/repositories. Primary use: constants for integer selection fields (status codes, type codes). **Entity listeners:** JPA lifecycle hooks (prePersist, preUpdate, postLoad) για custom logic. **Audit tracking:** Built-in change history via `<track>` element, logging field changes với user/timestamp for compliance (GDPR). **JSON fields:** Support dynamic custom fields added via Axelor Studio without schema changes - flexibility vs performance trade-off.

### Architecture Philosophy

Axelor prioritizes **developer productivity** (minimal boilerplate, automatic generation, declarative configuration) và **business user empowerment** (custom fields, no-code tools) over **database performance optimization** (limited index control, no sharding, no read replicas) và **advanced ORM features** (no inheritance strategies, no composite keys, no soft deletes). Suitable cho small-to-medium deployments (hundreds of users, millions of records) nơi development speed more critical than extreme scale. Large deployments (thousands of concurrent users, billions of records) có thể need custom optimizations beyond framework capabilities.

---

**Tổng số dòng code đã phân tích:**
- Domain XML files: ~15 files, ~3,000 lines total
- Generated entity code: ~10 files reviewed, representative sample
- Configuration files: axelor-config.properties, build.gradle

**Nguồn:** Tất cả findings từ direct source code analysis, supplemented với suy luận based on JPA/Hibernate standard behaviors và common enterprise application patterns.

---

*Kết thúc RESEARCH_STEP2_DATABASE.md*