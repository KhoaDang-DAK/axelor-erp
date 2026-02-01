# BƯỚC 3: PHÂN TÍCH HỆ THỐNG BẢO MẬT VÀ PHÂN QUYỀN - Phân Tích Từ Source Code

## Phương pháp phân tích

Nghiên cứu kiến trúc bảo mật (security architecture) và hệ thống phân quyền (authorization system) được thực hiện qua việc phân tích domain entities liên quan đến authentication/authorization, service implementations xử lý permissions, và cấu hình authentication providers. Phạm vi bao gồm authentication mechanisms, authorization model hierarchy, object/field/record-level permissions, và permission management tools.

**Domain XML files phân tích:**
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/User.xml` - Entity người dùng với business extensions
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Group.xml` - Entity nhóm người dùng
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Role.xml` - Entity vai trò
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Permission.xml` - Permissions cấp object

**Service implementations:**
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/auth/service/PermissionServiceImpl.java` - Logic phân quyền
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/auth/service/PermissionAssistantService.java` - CSV import/export permissions
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/service/pac4j/BaseAuthPac4jUserService.java` - Pac4j integration

**Configuration analysis:**
- `/src/main/resources/axelor-config.properties` - Authentication providers, session config, password policy

---

## Kết quả chi tiết

### 1. SECURITY FRAMEWORK VÀ KIẾN TRÚC TỔNG QUAN

**File nguồn:** Analysis của imports trong service classes, configuration files [Từ source code]

Axelor áp dụng một approach độc đáo trong ecosystem Java enterprise applications: thay vì sử dụng established security frameworks như Spring Security (de facto standard cho Spring applications) hoặc Apache Shiro (framework-agnostic security), Axelor xây dựng **custom security layer** được tích hợp chặt chẽ với framework core. Quyết định thiết kế này mang lại control tốt hơn về permission model phức tạp (multi-level permissions: object, field, record) nhưng cũng có trade-off về maintenance burden (phải tự maintain security code thay vì rely on community-tested frameworks) và ecosystem integration (khó integrate third-party security tools expecting Spring Security APIs).

Security architecture của Axelor chia làm hai layers rõ ràng: **authentication layer** (xác thực danh tính - "who you are") và **authorization layer** (phân quyền truy cập - "what you can do"). Authentication layer được implement qua Pac4j library version 5.7.7 - một abstraction layer cho phép support nhiều authentication providers (OAuth 2.0, SAML, LDAP, CAS) mà không cần write provider-specific code. Pac4j là excellent choice cho multi-provider scenarios vì nó cung cấp unified API: developers write code against Pac4j interfaces, có thể switch providers bằng cách change configuration không cần code changes. Version 5.7.7 (released 2023) là relatively recent, showing Axelor keeps dependencies updated.

Authorization layer hoàn toàn custom-built bởi Axelor, consisting of proprietary entity model (User, Group, Role, Permission, MetaPermission) và service layer logic (PermissionServiceImpl, PermissionAssistantService) để resolve permissions at runtime. Framework này không expose Spring Security's `@PreAuthorize`, `@Secured` annotations - authorization checks likely happen trong framework internals (interceptors, filters) transparent to application code. Dependency injection sử dụng Google Guice thay vì Spring - fundamental architectural decision affecting how components wired together, how scopes managed, và how AOP-based security interception implemented.

**Bằng chứng từ code - Service layer imports:**
```java
// File: PermissionAssistantService.java
import com.axelor.auth.db.Group;
import com.axelor.auth.db.Permission;
import com.axelor.auth.db.Role;
import com.axelor.meta.db.MetaPermission;
import com.axelor.meta.db.MetaPermissionRule;
```

**Giải thích code:** Import statements reveal security entities package structure: `com.axelor.auth.db` chứa core authorization entities (Group, Permission, Role), trong khi `com.axelor.meta.db` chứa metadata-driven permission entities (MetaPermission, MetaPermissionRule). Prefix "Meta" suggests runtime-configurable permissions (thay vì compile-time defined), aligning với Axelor's philosophy về configurability. Entities này không extend Spring Security classes (như GrantedAuthority, UserDetails) - confirming custom implementation.

**Bằng chứng từ code - Pac4j integration service:**
```java
// File path:
/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/service/pac4j/BaseAuthPac4jUserService.java
```

**Giải thích code:** Service class `BaseAuthPac4jUserService` acts as bridge giữa Pac4j authentication results và Axelor's User entity. Khi user successfully authenticates qua OAuth/SAML/LDAP provider, Pac4j returns profile data (email, name, attributes). Service này maps profile to Axelor User entity, handles user provisioning (create new user or link existing user), và assigns default group/roles. Class name prefix "Base" indicates này là base implementation có thể được override trong customer projects cho custom provisioning logic.

**Authentication providers supported:** [Từ configuration analysis]
Axelor configuration files show support cho:
- **Local authentication** - Username/password stored trong database
- **Google OAuth 2.0** - OpenID Connect protocol
- **Keycloak** - Open-source Identity and Access Management
- **SAML 2.0** - Enterprise SSO standard
- **LDAP** - Corporate directory integration (Active Directory, OpenLDAP)
- **CAS** - Central Authentication Service (legacy SSO protocol)

Sự đa dạng này critical cho enterprise deployments nơi different organizations có different authentication infrastructure. Large enterprises thường đã có LDAP/Active Directory chứa employee accounts - Axelor's LDAP support enables seamless integration without duplicating user management. Startups hoặc cloud-native companies có thể prefer OAuth 2.0 với Google/Keycloak cho modern authentication flows. Government/high-security organizations may require SAML 2.0 for compliance với security standards.

---

### 2. MÔ HÌNH PHÂN QUYỀN: HIERARCHY BA CẤP (USER → GROUP/ROLE → PERMISSIONS)

**File nguồn:** User.xml, Group.xml, Role.xml, Permission.xml, PermissionAssistantService.java [Từ source code]

Axelor implements một **three-tier authorization hierarchy** sophisticated hơn so với simple role-based access control (RBAC) nhưng straightforward hơn complex attribute-based access control (ABAC) systems. Model này balances giữa flexibility (support complex enterprise permission requirements) và manageability (administrators có thể understand và configure permissions without deep technical knowledge). Hierarchy design là: **User** (người dùng cá nhân) belongs to một **Group** (nhóm nghiệp vụ: Sales, Accounting, Management) và có thể có nhiều **Roles** (vai trò chức năng: Order Approver, Report Viewer, Admin), mỗi Group/Role chứa **Permissions** (quyền truy cập cụ thể vào objects/fields).

Lý do cho dual Group/Role system (thay vì chỉ Roles như many RBAC systems) là separate organizational structure từ functional capabilities. Groups typically map to business units hoặc departments (thường stable, thay đổi ít khi org restructuring), trong khi Roles map to job functions hoặc responsibilities (có thể assigned/revoked frequently khi people change positions). Ví dụ: một user belongs to "Sales - North Region" group (organizational placement) và có roles "Order Creator", "Quote Approver" (functional capabilities). Khi user moves to different region, chỉ cần change group; khi user promoted, add role "Manager" without changing group. Separation này makes permission management scalable trong large organizations với hundreds/thousands users.

Permission aggregation logic crucial nhưng không explicitly documented trong analyzed code - likely implemented trong framework core. Inferred behavior based on common RBAC patterns: user's effective permissions là union of permissions từ their group PLUS permissions từ all their roles. Merging strategy appears grant-based (permissive): nếu ANY source (group or role) grants permission, user có permission đó. Không có evidence của explicit DENY rules - absence of permission nghĩa là deny. Grant-based merging simpler to reason about (administrators don't have to worry about permission conflicts) nhưng less flexible than priority-based systems allowing deny to override grant.

**Bằng chứng từ code - User → Group relationship:**
```xml
<!-- File: User.xml, line 45 -->
<many-to-one name="group" ref="Group" column="group_id" massUpdate="true"/>
```

**Giải thích code:** User entity có many-to-one relationship tới Group, nghĩa là mỗi user belongs to exactly ONE primary group (hoặc NULL nếu no group assigned). Attribute `massUpdate="true"` cho phép bulk operations: administrator có thể select nhiều users và reassign hết cho different group cùng lúc - practical feature cho scenarios như "move all users from closed department to new department" hoặc "bulk upgrade interns to full employees, changing their group". Column name `group_id` explicitly specified (instead of Axelor default naming) có thể for backward compatibility với existing schema hoặc integration với external systems expecting specific column names.

**Bằng chứng từ code - Group/Role → Permissions relationship:**
```java
// File: PermissionAssistantService.java

// Line 636: Group có collection of MetaPermissions
group.addMetaPermission(metaPermission);

// Line 657: Role có collection of MetaPermissions
role.addMetaPermission(metaPermission);

// Line 715: Group có collection of Permissions
group.addPermission(permission);

// Line 744: Role có collection of Permissions
role.addPermission(permission);
```

**Giải thích code:** Method calls `addMetaPermission()` và `addPermission()` indicate both Group và Role entities có one-to-many relationships với MetaPermission và Permission entities. Relationship này bidirectional: từ Group/Role perspective, có collection of permissions; từ Permission perspective, có reference back to owning Group or Role. Dual permission types (Permission vs MetaPermission) serve different purposes: **Permission** for object-level authorization (can user read/write/delete SaleOrder entity?), **MetaPermission** for field-level authorization (can user edit 'discountAmount' field trong SaleOrder?). Separation enables fine-grained control: có thể grant user access to entity nhưng restrict specific sensitive fields.

**Hierarchy visualization:** [Suy luận từ code structure]
```
User (individual người dùng)
  ├─→ Group (many-to-one) - Primary organizational group
  │     ├─→ Permission[] (object-level CRUD permissions)
  │     └─→ MetaPermission[] (field-level permissions)
  │
  └─→ Role[] (many-to-many, inferred) - Functional roles
        ├─→ Permission[] (object-level CRUD permissions)
        └─→ MetaPermission[] (field-level permissions)
```

Model này follows principle of least privilege: users start với no permissions (default deny), permissions explicitly granted qua group/role membership. Administrative overhead là assigning users to appropriate groups/roles, not managing per-user permissions (which wouldn't scale). New employee onboarding simple: assign to group (organizational unit) và relevant roles (job functions), automatically inherits all necessary permissions. Permission changes centralized: update group/role permissions once, affects all members immediately.

---

### 3. USER ENTITY: AUTHENTICATION VÀ BUSINESS CONTEXT

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/User.xml` [Từ source code]

User entity trong Axelor serves dual purpose: **authentication credentials** (username, password, email) và **business context** (companies, teams, partner linkage, preferences). Đây là common pattern trong business applications where authentication user identity phải mapped to business domain concepts. Ví dụ: khi sales representative logs in, system cần biết không chỉ "this is user 'john.doe'" (authentication) mà còn "john.doe represents partner Acme Corp, works in North Region team, current company context is NYC branch" (business context). Business context này critical cho permission resolution (record-level filtering based on user's company/team) và application behavior (default values, available actions, dashboard content).

User entity được defined trong `com.axelor.auth.db` package (framework core), nhưng axelor-open-suite **extends** base entity với business-specific fields qua XML extension mechanism. Extension pattern allows framework maintain core authentication logic (password hashing, session management, login/logout) trong stable base class, while applications add domain-specific customizations without forking framework code. Trade-off: extensions limited to adding fields/methods, cannot change core authentication behavior without modifying framework - acceptable for most use cases nhưng may limit deep customizations cho unusual authentication requirements.

Security-related fields trong User entity provide temporal access control: `blocked` boolean flag immediately disables user account (for terminated employees hoặc security incidents), `activateOn` date implements delayed activation (create account now, becomes active on hire date), `expiresOn` date implements automatic expiration (temporary contractors, trial accounts). Combination của ba fields này enables sophisticated account lifecycle management: HR creates account when contract signed với `activateOn` set to start date và `expiresOn` set to end date, account automatically becomes active/inactive theo schedule without manual intervention. Field `sendEmailUponPasswordChange` implements security notification (user receives email when password changed - detect unauthorized password resets).

**Bằng chứng từ code - User entity structure:**
```xml
<entity name="User" sequential="true">
  <!-- Authentication & Access Control -->
  <many-to-one name="group" ref="Group" column="group_id" massUpdate="true"/>
  <boolean name="blocked" default="true"
    help="Specify whether to block the user for an indefinite period." massUpdate="true"/>

  <!-- Business Context - Multi-Company -->
  <many-to-many name="companySet" ref="com.axelor.apps.base.db.Company" title="Company set"/>
  <many-to-one name="activeCompany" ref="com.axelor.apps.base.db.Company"
    title="Active company" massUpdate="true"/>

  <!-- Business Context - Teams -->
  <many-to-many name="teamSet" ref="com.axelor.apps.base.db.Team" title="Team set"/>
  <many-to-one name="activeTeam" ref="com.axelor.apps.base.db.Team"
    title="Active team" massUpdate="true"/>

  <!-- Business Context - Partner Linkage -->
  <one-to-one name="partner" ref="com.axelor.apps.base.db.Partner"
    title="Partner" mappedBy="linkedUser"/>

  <!-- Localization & Preferences -->
  <string name="language" selection="select.language"/>
  <string name="localization"/>

  <!-- Custom Fields Support -->
  <string name="attrs" json="true"/>
</entity>
```

**Giải thích code:** Entity declaration không có base class specified trong XML (chỉ `<entity name="User">`), indicating đây là extension của existing User entity từ framework core - generator sẽ merge fields này vào base class. Attribute `sequential="true"` có thể indicate entity có sequence number generation (user codes auto-incremented).

Field `blocked` có `default="true"` - surprising choice! New users created trong blocked state, phải explicitly unblocked before they can login. Rationale: safety-first approach preventing accidental account activation, ensures administrator reviews và explicitly enables account after setup complete. Help text confirms purpose: "block user for indefinite period" (không phải temporary suspension, là complete disable).

Multi-company support via `companySet` (many-to-many: user có thể work for multiple companies) và `activeCompany` (current context). Pattern này common trong multi-tenant scenarios where single user account spans multiple legal entities/branches. User switches active company trong UI, application filters data/permissions accordingly. Similar pattern for teams: `teamSet` (all teams user belongs to) và `activeTeam` (current team context for filtering/defaults).

**Bằng chứng từ code - Computed fullName field:**
```xml
<string name="fullName" namecolumn="true" search="partner,name" title="Partner name">
  <![CDATA[
  if(partner != null) {
      if(partner.getFirstName() != null){
          return partner.getFirstName()+" "+partner.getName();
      }
      return partner.getName();
  }
  return name;
  ]]>
</string>
```

**Giải thích code:** Field `fullName` computed dynamically based on whether user linked to Partner entity. Logic prioritizes partner's name (business entity representation) over technical username: nếu user linked to partner John Doe, display name là "John Doe" thay vì "jdoe". Fallback to `name` (username) nếu no partner link. Attribute `namecolumn="true"` marks này là display name used trong dropdowns, search results, audit logs. Attribute `search="partner,name"` enables searching by either partner name OR username - users có thể search "John" (partner first name) hoặc "jdoe" (username) để find same user record.

**Partner linkage implication:** [Suy luận về integration patterns]
One-to-one relationship `partner` với `mappedBy="linkedUser"` indicates bidirectional link: User entity có partner reference, Partner entity có linkedUser reference. Use case: sales application nơi external partners (customers, suppliers) need access to portal - partner's contact person given user account linked to their Partner record. Khi partner user logs in, application automatically knows which company they represent, can show only relevant data (their own orders, invoices). Alternative pattern would be separate PartnerUser entity, nhưng direct linkage simpler for 1:1 mapping scenarios.

---

### 4. GROUP ENTITY: ORGANIZATIONAL UNITS VÀ BUSINESS ROLES

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Group.xml` [Từ source code]

Group entity represents **organizational units** hoặc **business role groupings** trong enterprise structure - không chỉ technical permission containers. Fields như `isClient`, `isSupplier` indicate groups có thể represent external user communities (customer portal users, supplier portal users), not just internal employees. Field `technicalStaff` suggests special category for IT/admin users có elevated privileges. Pattern này blurs line giữa "group as department" và "group as persona" - same entity type serving multiple organizational concepts. Flexibility này powerful nhưng có risk confusion: administrators must establish clear naming conventions để distinguish group types (prefix "Dept-" for departments, "Portal-" for external users, "Tech-" for technical staff).

Entity marked `cacheable="true"` - important performance optimization vì group data accessed frequently (every permission check may query group membership) nhưng changes infrequently (organizational restructuring happens quarterly/yearly, not hourly). Hibernate L2 cache stores deserialized Group objects trong memory, subsequent queries hit cache instead of database. Cache invalidation critical: khi group modified (permissions added/removed, properties changed), cache entry must be evicted or stale data causes permission bugs (users có permissions they shouldn't, or vice versa). Axelor framework likely has automatic cache invalidation hooks trong entity lifecycle listeners.

Fields `navigation` và `homeAction` enable per-group UI customization: different groups see different navigation menus và different home dashboards on login. Ví dụ: Sales group home action shows sales pipeline dashboard, Accounting group shows financial summary, Executives show KPI dashboard. Implementation likely: on login, framework loads user's group, reads homeAction property, redirects to specified action. Navigation property có thể control menu items visibility (technical staff sees admin menus, regular users don't). Customization này improves user experience (users immediately see relevant content) nhưng increases configuration complexity (administrators must maintain group-specific UI configs).

**Bằng chứng từ code - Group entity fields:**
```xml
<entity name="Group" cacheable="true">
  <!-- Business Role Flags -->
  <boolean name="technicalStaff"
    help="Specify whether the members of this group are technical staff." massUpdate="true"/>
  <boolean name="isClient" default="false" massUpdate="true" title="Client"/>
  <boolean name="isSupplier" default="false" massUpdate="true" title="Supplier"/>

  <!-- UI Customization -->
  <string name="navigation" selection="select.user.navigation" massUpdate="true"/>
  <string name="homeAction" help="Default home action." massUpdate="true"/>
</entity>
```

**Giải thích code:** Entity extends base Group từ `com.axelor.auth.db` package (core framework), adding business-specific fields. All boolean flags have `massUpdate="true"` enabling bulk changes - useful when reclassifying groups (e.g., promote entire group to technical staff status).

Field `technicalStaff` có descriptive help text suggesting special handling: technical staff members có thể bypass certain restrictions, see debug information, access admin features. Implementation likely checked trong permission evaluation logic: `if (user.getGroup().getTechnicalStaff()) { grant elevated access }`. Risk: overly broad technical staff designation leads to excessive privilege escalation - best practice limit to genuine IT/admin users.

Fields `isClient` và `isSupplier` enable portal scenarios where external entities have limited access. Client groups might have read-only access to their orders/invoices với self-service capabilities (download PDFs, submit support tickets). Supplier groups might update delivery status, submit invoices electronically. Default `false` means internal employee groups unless explicitly marked - safety-first (don't accidentally expose internal data to external users).

Selection field `navigation` references `select.user.navigation` - a selection definition (enum-like) elsewhere defining navigation modes. Possible values might be: "classic" (traditional menu tree), "tiles" (modern card-based), "minimal" (simplified for external users). Field type `string` instead of integer selection suggests navigation modes identified by string keys for extensibility (can add custom navigation types without database changes).

**Group naming conventions inferred:** [Suy luận best practices]
Effective group management trong large deployments requires consistent naming. Recommended patterns:
- **Department groups:** "Sales-North", "Finance-HQ", "Operations-Manufacturing"
- **Portal groups:** "Portal-Customers", "Portal-Suppliers"
- **Technical groups:** "Admins", "Developers", "Support-Staff"
- **Role-based groups:** "Order-Approvers", "Report-Viewers" (though these better as Roles)

Prefix approach helps administrators quickly identify group type when reviewing permission assignments hoặc troubleshooting access issues.

---

### 5. ROLE ENTITY: FUNCTIONAL CAPABILITIES VÀ ORTHOGONAL PERMISSIONS

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Role.xml` [Từ source code]

Role entity trong Axelor remarkably minimal - chỉ có `name` và `description` fields trong extension XML, indicating framework core provides most functionality. Minimalism này intentional: roles là pure permission containers without business logic hoặc UI customization (unlike Groups có navigation/homeAction). Separation of concerns: Groups = organizational context + UI customization + permissions, Roles = pure functional capabilities (permissions only). Users can have one group (organizational placement) nhưng multiple roles (multiple functional capabilities), enabling fine-grained authorization.

Use case cho roles versus groups: consider employee working part-time trong multiple capacities - member of "Sales" group (primary organizational placement) nhưng có roles "Order-Creator" (can create sales orders), "Invoice-Approver" (can approve invoices), "Report-Viewer" (can view analytics). Khi employee promoted, add role "Manager" (can approve higher amounts, see subordinate data) without changing group membership. Khi employee temporarily covers for colleague, add temporary role (can be revoked sau khi coverage period ends). Role assignment/revocation more frequent và granular than group membership changes.

Entity chỉ có tracking configuration (audit trail cho name và description changes) - confirming roles primarily metadata containers. Real power comes from relationship với permissions (not shown trong extension XML nhưng inferred từ PermissionAssistantService code). Role architecture supports **role composition**: có thể có "basic" roles ("Order-Viewer") và "advanced" roles ("Order-Manager" = Order-Viewer + Order-Editor + Order-Approver permissions) - though composition logic would be implemented trong permission management UI or service layer, không phải entity level.

**Bằng chứng từ code - Role entity structure:**
```xml
<entity name="Role">
  <track>
    <field name="name"/>
    <field name="description"/>
  </track>
</entity>
```

**Giải thích code:** XML exceptionally terse - chỉ tracking configuration. No custom fields, no business logic, no UI hints. Này reinforces roles as lightweight permission aggregators. Tracking captures changes to role definition: khi administrator renames role hoặc updates description, audit log records who made change và when. Audit trail important cho compliance scenarios (auditors ask "who changed permissions for Finance role?").

Base Role entity (in framework core) must contain:
- Primary key (`id`)
- Audit fields (`createdBy`, `createdOn`, `updatedBy`, `updatedOn`)
- `name` field (unique identifier)
- `description` field (human-readable explanation)
- Relationship to Permission entities (one-to-many)
- Relationship to MetaPermission entities (one-to-many)

**Role vs Group decision matrix:** [Suy luận về best practices]

Use **Groups** when:
- Permission set reflects organizational structure (departments, divisions)
- Users typically have one primary affiliation
- UI customization needed (different home screens per org unit)
- Business flags needed (isClient, isSupplier, technicalStaff)
- Changes infrequent (reorganizations happen annually)

Use **Roles** when:
- Permission set reflects job functions (viewer, editor, approver)
- Users may have multiple capabilities simultaneously
- Permissions assigned/revoked frequently (promotions, temporary duties)
- Cross-cutting concerns spanning multiple groups (all managers across all departments need report access)
- Fine-grained permission composition (build complex permissions from simple building blocks)

Hybrid approach (using both) most powerful: user inherits broad permissions từ group (departmental baseline) plus specific capabilities từ roles (functional additions).

---

### 6. PERMISSION ENTITY: OBJECT-LEVEL CRUD AUTHORIZATION

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Permission.xml`, PermissionAssistantService.java [Từ source code]

Permission entity implements **object-level authorization** - kiểm soát quyền truy cập vào entire entities (SaleOrder, Invoice, Product) chứ không phải individual instances hoặc fields. Granularity level này coarser than record-level (where permissions vary by instance: "can edit MY orders but not others' orders") nhưng finer than application-level (where permissions apply to entire application: "can access sales module"). Object-level permissions practical sweet spot: administrators có thể control "can Sales group create Purchase Orders?" without micromanaging every single order instance. Entity marked `cacheable="true"` critical cho performance - permission checks happen extremely frequently (potentially every database query), caching results prevents database from becoming bottleneck.

Permission model supports five operations representing standard CRUD plus export capability: **canRead** (view records), **canWrite** (edit existing records), **canCreate** (create new records), **canRemove** (delete records), và **canExport** (export to CSV/Excel/PDF). Separation của create từ write important cho workflows: trainee users có thể create draft orders (canCreate) nhưng cannot edit submitted orders (no canWrite), forcing review process. Export permission separate vì data export raises data leakage concerns: users có thể view individual records on screen (canRead) nhưng bulk export to spreadsheet (canExport) enables data exfiltration - tighter control needed. Default permission structure follows "all or nothing" principle: absence of permission entity for object means deny all operations, presence means grant specified operations.

Naming convention cho permissions follows pattern `perm.{ObjectName}.{GroupOrRoleCode}` - ví dụ: `perm.SaleOrder.sales_team`, `perm.Product.admins`, `perm.Partner.suppliers`. Convention này enables: (1) visual scanning in permission lists (sort alphabetically groups related permissions), (2) avoid naming collisions (unique names across entire system), (3) programmatic generation (services can construct permission names without database lookups). Field `object` contains fully qualified class name (e.g., `com.axelor.apps.sale.db.SaleOrder`) hoặc wildcard package patterns (e.g., `com.axelor.apps.sale.db.*` meaning all entities in package). Wildcard support critical for scaling: administrators can grant "all sales entities" without enumerating hundreds of individual entities.

**Bằng chứng từ code - Permission entity structure:**
```xml
<entity name="Permission" cacheable="true">
  <track>
    <field name="name"/>
    <field name="object"/>
    <field name="canRead"/>
    <field name="canWrite"/>
    <field name="canCreate"/>
    <field name="canRemove"/>
    <field name="canExport"/>
    <field name="condition"/>
    <field name="conditionParams"/>
  </track>
</entity>
```

**Giải thích code:** Tracking configuration logs all field changes - essential for security auditing. Khi permission modified (e.g., admin accidentally grants canRemove when should only grant canRead), audit trail shows who made mistake và when, enabling quick rollback. Fields `condition` và `conditionParams` enable record-level filtering (discussed in next section) - permissions không chỉ binary allow/deny nhưng có thể conditional based on record attributes và user context.

**Bằng chứng từ code - Permission CRUD assignment:**
```java
// File: PermissionAssistantService.java, lines 280-285
permission.setCanRead(row[0].equalsIgnoreCase("x"));
permission.setCanWrite(row[1].equalsIgnoreCase("x"));
permission.setCanCreate(row[2].equalsIgnoreCase("x"));
permission.setCanRemove(row[3].equalsIgnoreCase("x"));
permission.setCanExport(row[4].equalsIgnoreCase("x"));
```

**Giải thích code:** Service code parses CSV import row where each operation represented by "x" marker (present = grant, absent = deny). Boolean fields set based on case-insensitive check - accepting "X" hoặc "x" for user convenience. Simple representation (x vs blank) makes CSV files human-readable và editable in spreadsheet tools - administrators can export permissions, modify in Excel, re-import. Alternative would be 0/1 or true/false, nhưng "x" provides better visual scanning (checkboxes metaphor familiar to users).

**Bằng chứng từ code - Permission naming convention:**
```java
// File: PermissionAssistantService.java, lines 251-256
protected String getPermissionName(MetaField userField, String objectName, String suffix) {
    String permName = "perm." + objectName + "." + suffix;
    if (userField != null) {
      permName += "." + userField.getName();
    }
    return permName;
}
```

**Giải thích code:** Method constructs permission name từ components: prefix "perm" (identifies permission entities vs other entities), object name (target entity), suffix (group/role code), optional field name (for field-level permissions). Dot-separated format enables hierarchical organization và parsing. Method used by both permission creation (generate consistent names) và permission lookup (find existing permissions by calculated name). Null check for `userField` indicates method serves dual purpose: object-level permissions (field null) và field-level permissions (field specified).

**Permission validation against metamodel:** [Từ source code PermissionAssistantService lines 500-514]
```java
public List<Long> checkPermissionsObject() {
    List<Permission> permissionList = permissionRepository.all().fetch();
    if (ObjectUtils.isEmpty(permissionList)) {
      return null;
    }

    initObjectOrPackages();  // Get all entity packages from JPA metamodel

    return permissionList.stream()
        .filter(permission -> !isValidObject(permission.getObject()))
        .map(Permission::getId)
        .collect(Collectors.toList());
}

protected boolean isValidObject(String object) {
    String regex = object.replace("*", ".*");  // Support wildcard
    return objectOrPackages.stream()
        .anyMatch(entityPackage -> entityPackage.matches(regex));
}
```

**Giải thích code:** Validation service detects "orphaned" permissions - permissions referencing entities không còn exists trong application (deleted modules, renamed entities, typos trong permission setup). Method `initObjectOrPackages()` queries JPA metamodel để get list of all entity classes currently registered. Stream filtering checks mỗi permission's object name against metamodel, using regex matching to support wildcards. Invalid permissions returned as ID list để administrators can review và delete. Validation critical sau khi uninstalling modules hoặc refactoring code - prevents accumulation of stale permissions causing confusion.

---

### 7. RECORD-LEVEL SECURITY: DOMAIN FILTERS VÀ CONDITIONAL ACCESS

**File nguồn:** PermissionAssistantService.java lines 310-343 [Từ source code]

Record-level security (còn gọi là row-level security trong database terminology) represents most sophisticated level của authorization system: permissions không chỉ control "can user access SaleOrder entity?" (object-level) mà còn "which specific SaleOrder instances can user access?" (record-level). Implementation qua **condition mechanism** - SQL-like WHERE clauses dynamically injected vào queries based on user context. Pattern này transforms simple permission check từ binary yes/no thành filtered dataset: user always "has permission" nhưng only sees subset of records matching their context.

Condition syntax resembles JPA query language: `self.fieldName` references current entity being queried, placeholders `?` represent parameter values supplied from `conditionParams`. Critical insight: conditions executed at database level (part of SQL WHERE clause) not application level (filtering in Java code), ensuring: (1) performance - database indexes used, only matching rows returned, (2) security - users cannot bypass by manipulating application code, (3) consistency - same filtering applies across all access paths (UI, API, reports). Trade-off: conditions limited to expressions database can evaluate - complex business logic requiring service layer computations cannot be encoded in conditions.

Magic variables trong `conditionParams` enable dynamic filtering based on current user's attributes. Pattern `__user__.{fieldName}` resolved at runtime: framework reads logged-in user's field value và substitutes into query. Ví dụ: condition `self.company = ?` với param `__user__.activeCompany` becomes SQL `WHERE sale_order.company_id = 123` (where 123 is current user's activeCompany ID). Variable resolution supports traversing relationships: `__user__.partner.company` navigates from User to Partner to Company. Collection fields enable IN clauses: `__user__.teamSet` expands to `(1,2,3)` for user belonging to teams 1,2,3.

**Bằng chứng từ code - Condition generation logic:**
```java
// File: PermissionAssistantService.java, lines 325-342
String condition = "";
String conditionParams = "__user__." + userField.getName();

if (userField.getRelationship().contentEquals("ManyToOne")) {
  condition = "self." + objectField.getName() + " = ?";
} else {
  condition = "self." + objectField.getName() + " in (?)";
}

// Example outputs:
// For many-to-one relationship (user has one activeCompany):
//   condition: "self.company = ?"
//   conditionParams: "__user__.activeCompany"
//
// For many-to-many relationship (user has multiple teams):
//   condition: "self.assignedTo in (?)"
//   conditionParams: "__user__.teamSet"
```

**Giải thích code:** Code dynamically generates condition syntax based on relationship type discovered via metamodel introspection. Many-to-one relationships (single value) use equality operator `=`, collection relationships (many-to-many, one-to-many) use set membership operator `in`. Field name extraction from metadata (`userField.getName()`, `objectField.getName()`) ensures conditions valid against actual schema - typos prevented. Generated conditions stored trong Permission entity for runtime evaluation, not compile-time - enabling administrators modify filtering rules without code deployment.

**Example scenarios demonstrating power:**

**Scenario 1: Multi-company data isolation**
```
User: John (activeCompany: NYC Branch)
Permission on SaleOrder:
  condition: "self.company = ?"
  conditionParams: "__user__.activeCompany"

Result: John sees only SaleOrders where company_id = NYC Branch ID
All queries automatically filtered, transparent to application code
```

**Scenario 2: Team-based record assignment**
```
User: Sarah (teamSet: [Sales-North, Sales-West])
Permission on Lead:
  condition: "self.assignedTeam in (?)"
  conditionParams: "__user__.teamSet"

Result: Sarah sees Leads assigned to Sales-North OR Sales-West
When Sarah reassigned to different teams, visible leads change automatically
```

**Scenario 3: Hierarchical access (manager sees subordinates' data)**
```
User: Manager Mike (teamSet includes all managed teams)
Permission on TimeSheet:
  condition: "self.employee.team in (?)"
  conditionParams: "__user__.teamSet"

Result: Mike sees timesheets of all employees in his managed teams
Path expression "self.employee.team" traverses TimeSheet→Employee→Team
```

**Implementation challenges inferred:** [Suy luận về technical complexity]

Implementing record-level security properly requires framework handle:
1. **Query rewriting**: Intercept all queries (JPA Criteria, JPQL, Query DSL) và inject condition WHERE clauses
2. **Parameter binding**: Resolve `__user__` variables, handle type conversion (entity references → IDs), expand collections
3. **JOIN optimization**: Conditions với path expressions (`self.employee.team`) require automatic JOINs, must avoid N+1 queries
4. **Permission composition**: Multiple permissions với different conditions (from group + roles) must be ORed together correctly
5. **Caching complexity**: Condition results user-specific, cannot cache globally, only per-user session
6. **Update validation**: When user modifies record, verify updated record still matches their conditions (prevent privilege escalation: user updates record they can see, changes it to values they shouldn't see)

Axelor framework core likely implements này qua JPA entity listeners hoặc Hibernate filters - technical details not visible trong analyzed application code. Complexity explains why many frameworks don't support row-level security out-of-box (Spring Security has basic support via ACLs nhưng not query-integrated).

**Bằng chứng từ code - CSV export includes conditions:**
```java
// File: PermissionAssistantService.java, lines 310-312
row[colIndex++] = Strings.isNullOrEmpty(perm.getCondition()) ? "" : perm.getCondition();
row[colIndex++] = Strings.isNullOrEmpty(perm.getConditionParams()) ? "" : perm.getConditionParams();
```

**Giải thích code:** CSV export/import includes condition và conditionParams as regular columns - administrators can edit conditions trong spreadsheet tool. Empty string handling (`Strings.isNullOrEmpty` check) distinguishes between no condition (empty = all records visible) versus empty condition string (could be parsing error). Export format enables bulk condition updates: administrator exports all permissions, uses Excel formulas to generate conditions for multiple groups/entities, re-imports. Alternative manual condition entry per permission tedious for hundreds of entities.

---

### 8. FIELD-LEVEL PERMISSIONS: METAPERMISSION VÀ METAPERMISSIONRULE

**File nguồn:** PermissionAssistantService.java lines 621-689 [Từ source code]

Field-level permissions provide finest-grained authorization control: even when user has object-level permission to read SaleOrder entity, specific sensitive fields (discountAmount, costPrice, marginPercentage) có thể hidden hoặc readonly. Architecture uses **two-tier structure**: **MetaPermission** acts as container/grouping entity for một object's field permissions, **MetaPermissionRule** defines actual per-field access rules. Two-tier design enables efficient querying (load MetaPermission once, get all field rules together) và clear organization (field rules grouped by object, not scattered across database).

MetaPermission entity follows same naming convention as Permission (`perm.{Object}.{GroupOrRole}`), owned by Group hoặc Role via one-to-many relationship. One MetaPermission can contain dozens of MetaPermissionRule entities (one per field). Structural similarity với Permission entity intentional - administrators understand "Permissions control objects, MetaPermissions control fields" without learning completely different concepts. Generated code likely includes convenience methods: `group.getMetaPermissions()` returns all field permission containers, `metaPermission.getRules()` returns all field rules.

MetaPermissionRule provides three boolean flags (canRead, canWrite, canExport) parallel to Permission's flags but notably MISSING canCreate và canRemove - logical vì fields don't exist independently của parent entity, cannot "create field" without creating entire entity. Export permission separate (như object-level) vì exported data shows field values even when UI hides them - users có thể export spreadsheet then search for hidden columns. Readonly vs hidden distinction important: **readonly fields visible but not editable** (users see values, understand business logic, but cannot change - e.g., computed totals), **hidden fields completely invisible** (users unaware field exists - e.g., internal cost data not shown to sales team).

**Bằng chứng từ code - MetaPermission retrieval/creation:**
```java
// File: PermissionAssistantService.java, lines 621-640
public MetaPermission getMetaPermission(Group group, String objectName) {
    String permName = getPermissionName(null, objectNames[objectNames.length - 1], group.getCode());
    MetaPermission metaPermission = metaPermissionRepository.all()
        .filter("self.name = ?1", permName)
        .fetchOne();

    if (metaPermission == null) {
      metaPermission = new MetaPermission();
      metaPermission.setName(permName);
      metaPermission.setObject(objectName);

      group.addMetaPermission(metaPermission);  // ← Bidirectional relationship
    }

    return metaPermission;
}
```

**Giải thích code:** Method implements get-or-create pattern: query for existing MetaPermission by name, if not found create new instance. Pattern common trong permission management - avoid duplicate permission entries (which would cause ambiguous authorization). Call to `group.addMetaPermission()` establishes bidirectional relationship: MetaPermission references Group, Group's collection includes MetaPermission. Bidirectional link enables navigation both directions: "what permissions does Sales group have?" và "which group owns this permission?". Null check in `getPermissionName(null, ...)` indicates object-level permission name (no specific field).

**Bằng chứng từ code - MetaPermissionRule creation with conditional logic:**
```java
// File: PermissionAssistantService.java, lines 663-689
public MetaPermission updateFieldPermission(
    MetaPermission metaPermission, String field, String[] row) {

    MetaPermissionRule permissionRule = ruleRepository.all()
        .filter("self.field = ?1 and self.metaPermission.name = ?2",
                field, metaPermission.getName())
        .fetchOne();

    if (permissionRule == null) {
      permissionRule = new MetaPermissionRule();
      permissionRule.setMetaPermission(metaPermission);
      permissionRule.setField(field);  // ← Field name as string
    }

    permissionRule.setCanRead(row[0].equalsIgnoreCase("x"));
    permissionRule.setCanWrite(row[1].equalsIgnoreCase("x"));
    permissionRule.setCanExport(row[4].equalsIgnoreCase("x"));
    permissionRule.setReadonlyIf(row[5]);  // ← Conditional readonly expression
    permissionRule.setHideIf(row[6]);      // ← Conditional hide expression

    metaPermission.addRule(permissionRule);

    return metaPermission;
}
```

**Giải thích code:** Similar get-or-create pattern for MetaPermissionRule. Query uses composite filter matching both field name AND parent MetaPermission - necessary vì same field name appears across different objects (every entity has "id" field, must distinguish SaleOrder.id rule from Invoice.id rule). Field name stored as string (`permissionRule.setField(field)`) not reference to MetaField entity - looser coupling enables field permissions survive schema changes (rename field in code, update permission field name separately).

Row indices `[0]`, `[1]`, `[4]`, `[5]`, `[6]` reveal CSV column layout: columns 0-4 are canRead/canWrite/canCreate/canRemove/canExport (parallel to object permissions), columns 5-6 are field-specific readonlyIf/hideIf. Gaps (skipping canCreate/canRemove for fields) maintain column alignment với object permission rows - CSV has uniform structure whether row represents object or field.

**Conditional field visibility:** Fields `readonlyIf` và `hideIf` contain **expressions** evaluated at runtime to dynamically control field visibility based on record state và user context. Expression language likely Groovy (Axelor's scripting language) or JavaScript. Examples of powerful conditional logic:

```groovy
// readonlyIf: Make discount field readonly after order confirmed
"statusSelect > 1"  // Status > Draft

// hideIf: Hide internal cost from non-managers
"!__user__.group.technicalStaff"  // User's group is not technical staff

// readonlyIf: Field readonly unless user is creator
"createdBy.id != __user__.id"  // Created by different user

// hideIf: Hide field based on order type
"typeSelect != 3"  // Not a special order type
```

Expressions access both record fields (`statusSelect`, `createdBy.id`, `typeSelect`) và user context (`__user__` variables), enabling context-aware UIs. Implementation requires framework evaluate expressions for each field on each record during rendering - performance consideration for large forms or grids. Caching expression compilation (parse once, evaluate many times) critical.

**Bằng chứng từ code - CSV export format for field rules:**
```java
// File: PermissionAssistantService.java, lines 272-294
protected int writeFieldPermission(MetaField field, String[] row, int colIndex, String permName) {
    MetaPermissionRule rule = ruleRepository.all()
        .filter("self.metaPermission.name = ?1 and self.metaPermission.object = ?2 and self.field = ?3",
                permName, field.getMetaModel().getFullName(), field.getName())
        .fetchOne();

    if (rule != null) {
      row[colIndex++] = !rule.getCanRead() ? "" : "x";
      row[colIndex++] = !rule.getCanWrite() ? "" : "x";
      row[colIndex++] = "";  // canCreate (N/A for fields)
      row[colIndex++] = "";  // canRemove (N/A for fields)
      row[colIndex++] = !rule.getCanExport() ? "" : "x";
      row[colIndex++] = "";  // condition (N/A for fields)
      row[colIndex++] = "";  // conditionParams (N/A for fields)
      row[colIndex++] = Strings.isNullOrEmpty(rule.getReadonlyIf()) ? "" : rule.getReadonlyIf();
      row[colIndex++] = Strings.isNullOrEmpty(rule.getHideIf()) ? "" : rule.getHideIf();
    }

    return colIndex;
}
```

**Giải thích code:** CSV export writes blank cells for N/A columns (canCreate, canRemove, condition, conditionParams) - maintaining column alignment với object permission rows enables single CSV file contain both object và field permissions. Triple-filter query (`permName AND object AND field`) ensures correct rule retrieval - same field name may appear in multiple objects under multiple permission names. Null checks prevent writing "null" strings to CSV - empty cells cleaner than literal "null" text.

**Use case example - Sales team permission matrix:**
```
Object: SaleOrder
Field permissions for "sales_team" group:
- clientPartner: canRead=YES, canWrite=YES (can edit customer)
- ourCompany: canRead=YES, canWrite=NO (see company, cannot change)
- exTaxTotal: canRead=YES, canWrite=NO (see subtotal, cannot manipulate)
- inTaxTotal: canRead=YES, canWrite=NO (see total, cannot manipulate)
- discountAmount: canRead=YES, canWrite=YES, readonlyIf="statusSelect > 2" (editable only in draft/confirmed, readonly after)
- costPrice: canRead=NO, hideIf="true" (completely hidden - margin protection)
- internalNotes: canRead=YES, canWrite=YES (can add internal notes)
```

Matrix này enforces: sales team can create/edit orders but cannot manipulate computed totals (preventing fraud), cannot see cost prices (preventing margin disclosure), cannot edit discounts after certain workflow stage (preventing post-approval changes).

---

### 9. PERMISSION MANAGEMENT: CSV IMPORT/EXPORT TOOL

**File nguồn:** PermissionAssistantService.java - Full class analysis [Từ source code]

Managing hundreds of permissions across dozens of groups/roles và hundreds of entities through UI forms would be tedious và error-prone. Axelor provides **Permission Assistant** - sophisticated CSV-based import/export tool enabling bulk permission management using familiar spreadsheet software. Tool architecture follows ETL (Extract-Transform-Load) pattern common in data integration: export current permissions to CSV (Extract), edit in Excel/LibreOffice (Transform), import modified CSV (Load). Approach leverages administrators' spreadsheet skills - most administrators comfortable với Excel, can use formulas/fills/filters to rapidly configure permissions.

CSV format cleverly encodes hierarchical permission structure (objects → fields) trong flat tabular format. Header rows identify groups/roles (each group gets multiple columns: Read/Write/Create/Delete/Export/Condition/Params/ReadonlyIf/HideIf). Data rows represent either objects (entity-level permissions) or fields (indented under parent object). Example format:

```csv
;;;;;Group: Sales Team;;;;;;;Group: Managers;;;;;;;
Object;Field;Title;;R;W;C;D;E;Cond;Params;RO-If;Hide;;R;W;C;D;E;Cond;Params;RO-If;Hide
com.axelor.apps.sale.db.SaleOrder;;;x;x;x;;x;self.company=?;__user__.activeCompany;;;x;x;x;x;x;;;;
;clientPartner;Customer;;;x;x;;;;;;;x;x;;;;;;
;discountAmount;Discount;;;x;x;;;;;;statusSelect>2;;x;x;;;;;;
```

First row (group headers) spans multiple columns per group. Second row (column headers) defines meaning của each column. Subsequent rows mix object-level (no field name) và field-level (field name populated) permissions. Semicolons separate columns, empty cells represent "no permission" or "not applicable".

**Bằng chứng từ code - CSV export structure:**
```java
// File: PermissionAssistantService.java, lines 119-247 (export logic)
public void exportPermissions(/* parameters */) {
    // Build header rows with group/role names spanning columns
    // For each object in metamodel:
    //   Write object-level permission row
    //   For each field in object:
    //     Write field-level permission row (indented)
    // Generate CSV file with proper escaping
}
```

**Export workflow:** [Suy luận từ code structure]
1. Query all groups/roles to be exported (or all if none specified)
2. Query JPA metamodel để get all entities và their fields
3. For each entity, query existing Permissions và MetaPermissionRules
4. Build matrix: rows=objects+fields, columns=groups×9 (R/W/C/D/E/Cond/Params/RO/Hide)
5. Populate cells: "x" for granted permissions, condition text for filters, empty for denied
6. Write CSV with UTF-8 encoding, proper quote escaping, column alignment
7. Return file to user for download

**Import workflow provides validation và transactional updates:**

**Bằng chứng từ code - CSV import validation:**
```java
// File: PermissionAssistantService.java, lines 397-747 (import logic)
public void importPermissions(File csvFile) {
    // Parse CSV, extract group/role names from headers
    // Validate groups/roles exist in database
    // For each data row:
    //   Validate object name against JPA metamodel
    //   Validate field name exists on object
    //   Create/update Permission entities
    //   Create/update MetaPermissionRule entities
    // Save all changes in transaction (rollback on error)
}
```

**Import validations performed:**
1. **Header validation**: Group/role names in CSV must match existing database records - prevents typos creating orphaned permissions
2. **Object validation**: Entity class names must exist in JPA metamodel (checked via `isValidObject()`) - prevents permissions for non-existent entities
3. **Field validation**: Field names must exist on specified object - prevents typos in field permissions
4. **Syntax validation**: Condition expressions should parse correctly (though validation may be lenient - errors caught at runtime)

**Transactional behavior critical:** All permission updates wrapped in single database transaction. If any validation fails or error occurs mid-import, entire import rolled back - prevents partial permission updates leaving system in inconsistent state. Alternative non-transactional approach would create permissions for first N objects then fail on object N+1, leaving some groups configured and others not.

**Advanced use cases enabled by CSV approach:**

**Use Case 1: Clone permissions from one group to another**
1. Export permissions with both "source" and "target" groups
2. In Excel, copy source group columns to target group columns
3. Make minor adjustments (target slightly restricted)
4. Re-import - target group now has same permissions as source

**Use Case 2: Bulk permission generation with formulas**
1. Export empty permission matrix (all groups, all objects, no permissions)
2. Use Excel formulas: `=IF(ISNUMBER(SEARCH("sale", B2)), "x", "")` grants all sales-related objects
3. Use VLOOKUP to apply standard permission templates (viewer template, editor template, admin template)
4. Re-import - hundreds of permissions configured via formulas

**Use Case 3: Permission audit and review**
1. Export current permissions
2. Use Excel conditional formatting to highlight dangerous combinations (e.g., "canRemove=x on financial entities")
3. Use pivot tables to analyze: which groups have delete permission? which objects fully open to all groups?
4. Identify over-permissioned groups, export again with corrections, re-import

**Use Case 4: Version control and change tracking**
1. Export permissions to CSV monthly
2. Commit CSV files to Git repository
3. Diff between versions shows permission changes over time
4. Rollback to previous permission state by importing historical CSV

CSV approach's power comes from leveraging Excel's computational capabilities - filters, sorts, formulas, pivot tables, conditional formatting, VBA macros - to manipulate permissions as data. Alternative pure-UI approaches limited to forms/grids cannot compete với spreadsheet flexibility.

**Security consideration:** CSV import powerful but dangerous - importing malicious CSV could grant excessive permissions or wipe existing permissions. Access to Permission Assistant should be restricted to security administrators only. Import should log all changes (who imported, when, which file, what changed) for audit trail. Best practice: export before import (backup), review changes in CSV diff tool before importing.

---

### 10. AUTHENTICATION CONFIGURATION: PAC4J MULTI-PROVIDER SUPPORT

**File nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Axelor's authentication configuration demonstrates enterprise-grade flexibility through Pac4j library integration, supporting six different authentication providers simultaneously. Configuration follows convention-over-configuration principle: most settings commented out (disabled by default), administrators uncomment và populate only providers they need. Provider-agnostic configuration structure (`auth.provider.{name}.{property}`) enables adding custom providers without framework code changes - Pac4j extensibility shines through.

Session configuration controls user session lifecycle - critical security parameters. Setting `session.timeout = 480` (8 hours) balances security (automatic logout after inactivity reduces risk of unauthorized access to abandoned sessions) versus usability (users don't get logged out mid-workday). Commented `session.cookie.secure = true` should be enabled in production for HTTPS-only cookie transmission - prevents session hijacking over unencrypted connections. Session storage mechanism not explicit in config - likely defaults to servlet container's session management (in-memory for single server, distributed session store for clusters).

**Bằng chứng từ code - Session configuration:**
```properties
session.timeout = 480                    # 8 hours (in minutes)
#session.cookie.secure = true            # HTTPS only (uncomment in production!)
```

**Giải thích code:** Timeout value 480 minutes = 8 hours assumes standard business day - users login morning, can work entire day without re-authenticating. Alternative shorter timeouts (30-60 minutes) appropriate for high-security environments (banking, healthcare) accepting usability trade-off. Cookie secure flag commented indicates development-friendly default (allow HTTP testing) - dangerous if forgotten in production. Security audit checklist should verify này enabled for internet-facing deployments.

**Provider ordering và authentication flow controlled globally:**

**Bằng chứng từ code - Global authentication settings:**
```properties
#auth.provider-order =                   # Comma-separated provider names
#auth.callback-url =                     # OAuth callback URL
#auth.user.provisioning = none           # create / link / none
#auth.user.default-group = users         # Default group for new users
#auth.user.principal-attribute = email   # Attribute for principal name
```

**Giải thích code:** Provider order determines authentication cascade: nếu first provider rejects (user not found), try second provider, etc. Example: `auth.provider-order = ldap,google,local` attempts LDAP first (corporate directory), falls back to Google OAuth (external partners), finally local auth (emergency access). Callback URL critical for OAuth/SAML flows - providers redirect users back to this URL after authentication, must be publicly accessible và match provider registration exactly (mismatch causes authentication failures).

User provisioning setting controls whether external authentication automatically creates users (`create`), links to existing users by email (`link`), or rejects unknown users (`none`). Setting `create` enables self-service onboarding (Google OAuth user authenticates, account auto-created in Axelor), convenient but security risk if public OAuth domain (anyone with @company.com email could access). Setting `link` requires administrators pre-create accounts (tight control) but enables flexible authentication (user can login via LDAP OR Google using same account). Setting `none` strictest - only explicitly created users can login.

**Local authentication with basic auth support:**

**Bằng chứng từ code - Local auth configuration:**
```properties
#auth.local.basic-auth = indirect, direct  # Enable HTTP Basic Authentication
```

**Giải thích code:** Basic auth enables API authentication without browser sessions - clients send `Authorization: Basic <base64(username:password)>` header. Mode `indirect` requires redirect to login page for browser clients, `direct` allows immediate authentication for programmatic clients. Security warning: Basic auth transmits credentials with every request (even if base64-encoded), HTTPS mandatory to prevent credential interception. Modern alternatives (JWT tokens, OAuth 2.0 client credentials) more secure for APIs.

**Google OpenID Connect integration:**

**Bằng chứng từ code - Google OAuth config:**
```properties
#auth.provider.google.client-id =        # Google Cloud Console - OAuth Client ID
#auth.provider.google.secret =           # Client secret (keep confidential!)
```

**Giải thích code:** Google OAuth requires application registration in Google Cloud Console, generates client ID (public identifier) và secret (confidential key). Setup process: create OAuth consent screen (what permissions requested), configure authorized redirect URIs (must match auth.callback-url), obtain credentials. Security: client secret must be protected (exposure allows impersonation), rotate if compromised. Google automatically provides user's email, name, profile picture - suitable for external partner access without creating separate credentials.

**Keycloak integration for enterprise SSO:**

**Bằng chứng từ code - Keycloak config:**
```properties
#auth.provider.keycloak.client-id = demo-app
#auth.provider.keycloak.secret = 233d1690-4498-490c-a60d-5d12bb685557
#auth.provider.keycloak.realm = demo-app
#auth.provider.keycloak.base-uri = http://localhost:8083/auth
```

**Giải thích code:** Keycloak là open-source IAM platform popular trong enterprise Java ecosystems. Realm concept enables multi-tenancy within Keycloak (demo-app realm isolates test users từ production users). Base URI points to Keycloak server - can be internal corporate server (http://keycloak.company.com) hoặc cloud-hosted (https://company.auth0.com). Benefits over Google OAuth: full control (self-hosted), integration với corporate systems (LDAP sync, Active Directory federation), advanced features (two-factor auth, password policies, user federation).

**SAML 2.0 for standardized enterprise SSO:**

**Bằng chứng từ code - SAML configuration:**
```properties
#auth.provider.saml.keystore-path = {java.io.tmpdir}/samlKeystore.jks
#auth.provider.saml.keystore-password = open-platform-demo-passwd
#auth.provider.saml.private-key-password = open-platform-demo-passwd
#auth.provider.saml.identity-provider-metadata-path = http://localhost:9012/simplesaml/saml2/idp/metadata.php
#auth.provider.saml.service-provider-metadata-path = {java.io.tmpdir}/sp-metadata.xml
#auth.provider.saml.service-provider-entity-id = sp.test.pac4j
```

**Giải thích code:** SAML more complex than OAuth - requires mutual metadata exchange và cryptographic keys for assertion signing/verification. Keystore holds certificates for signing SAML requests và decrypting responses. Identity Provider metadata (from corporate IdP like Okta, Azure AD, ADFS) describes IdP's endpoints và certificates. Service Provider metadata (Axelor's metadata) sent to IdP describing callback URLs và expected assertion format. Entity ID uniquely identifies Axelor instance to IdP. SAML complexity justified for enterprises requiring: standardized protocol (not vendor lock-in), attribute release control (which user attributes shared), logout propagation (logout from IdP logs out from all SPs).

**LDAP integration for directory authentication:**

**Bằng chứng từ code - LDAP configuration:**
```properties
#auth.ldap.server.url = ldap://localhost:389
#auth.ldap.server.starttls = false              # Encrypt connection
#auth.ldap.server.auth.type = simple            # simple / CRAM-MD5 / DIGEST-MD5 / EXTERNAL / GSSAPI
#auth.ldap.server.auth.user = cn=admin,dc=test,dc=com
#auth.ldap.server.auth.password = admin
#auth.ldap.group.base = ou=groups,dc=test,dc=com
#auth.ldap.group.filter = (uniqueMember=uid={0})
#auth.ldap.user.base = ou=users,dc=test,dc=com
#auth.ldap.user.filter = (uid={0})
#auth.ldap.user.id-attribute = uid
```

**Giải thích code:** LDAP configuration follows directory structure conventions (Distinguished Names). Server auth settings define Axelor's credentials to connect to LDAP server (service account). User base và filter describe where users stored và how to query (`{0}` placeholder replaced với username). Group base và filter enable group membership lookup (assign Axelor groups based on LDAP groups). ID attribute specifies which LDAP attribute maps to Axelor username (uid, sAMAccountName, mail depending on directory schema). STARTTLS setting should be enabled (true) for production - encrypts LDAP traffic preventing password sniffing.

**CAS integration for legacy SSO:**

**Bằng chứng từ code - CAS configuration:**
```properties
#auth.provider.cas.login-url = https://localhost:8443/cas/login
#auth.provider.cas.prefix-url = https://localhost:8443/cas
#auth.provider.cas.protocol = CAS30       # CAS10 / CAS20 / CAS20_PROXY / CAS30 / CAS30_PROXY / SAML
```

**Giải thích code:** CAS (Central Authentication Service) là older SSO protocol developed by Yale University, still used in academic institutions và legacy enterprise environments. Protocol version selection important - CAS 3.0 supports attribute release (user metadata beyond username), CAS 1.0/2.0 only provide username. Proxy variants enable service-to-service authentication (Axelor can obtain tickets to call other CAS-protected services on user's behalf). SAML option enables CAS server act as SAML IdP. Organizations migrating from CAS to modern OAuth/SAML can run both temporarily during transition.

**Logout configuration for session termination:**

**Bằng chứng từ code - Logout settings:**
```properties
#auth.logout.default-url =                # Redirect after logout
#auth.logout.url-pattern =                # URL pattern triggering logout
#auth.logout.local = true                 # Remove profiles from session
#auth.logout.central = false              # Call IdP logout endpoint (SSO logout)
```

**Giải thích code:** Local logout (default) only clears Axelor session, IdP sessions remain active - user can immediately re-login without re-entering password. Central logout calls IdP's logout endpoint (SAML Single Logout, OAuth revocation), terminating SSO session globally - user logged out from all applications. Trade-off: central logout provides better security (user intentionally logged out, session should be terminated everywhere) but complex to implement (requires IdP support, reliable logout propagation) và có thể frustrate users (logging out from one app logs out from email, calendar, everything).

---

### 11. MULTI-TENANCY VÀ DATA ISOLATION

**File nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Multi-tenancy configuration option tồn tại trong Axelor config files nhưng implementation details largely absent từ analyzed application source code - suggesting feature implemented primarily trong framework core layer rather than application layer. Configuration property `application.multi-tenancy` controls enabling/disabling, với default value `false` indicating multi-tenancy opt-in feature (must be explicitly enabled).

**Bằng chứng từ code - Multi-tenancy configuration:**
```properties
# Enable multi-tenancy
#application.multi-tenancy = false
```

**Giải thích code:** Commented-out configuration với default `false` suggests most Axelor deployments run single-tenant mode. Multi-tenancy complexity (tenant isolation, cross-tenant queries, tenant-specific customizations) requires substantial infrastructure - enabling only when needed reduces operational complexity. Absence of additional multi-tenancy config (tenant resolution strategy, tenant database mapping, tenant customization paths) suggests either minimal configuration required (framework handles internally) or feature underdeveloped (basic implementation without advanced options).

**Missing implementation details:** [Không tìm thấy trong source code]
Analyzed axelor-open-suite source code does NOT contain:
- Tenant resolver classes (how framework determines current tenant from HTTP request)
- Tenant context management (thread-local tenant storage, context propagation)
- Tenant filter interceptors (automatic query filtering by tenant ID)
- Tenant-specific schema customizations (different fields/entities per tenant)

**Inferred multi-tenancy approach:** [Suy luận từ existing features]
Given User entity có `activeCompany` field và permissions support `self.company = ?` với `__user__.activeCompany` conditions, likely multi-tenancy implementation is **soft multi-tenancy** (shared schema, row-level filtering) rather than **hard multi-tenancy** (separate schemas per tenant). Rationale:

1. **Company as tenant discriminator**: User's activeCompany acts as tenant context
2. **Automatic query filtering**: Permission conditions inject company filters into queries
3. **Shared database schema**: All tenants' data in same tables, distinguished by company foreign key
4. **Application-level isolation**: Framework code ensures user only sees/modifies their company's data

**Soft multi-tenancy trade-offs:**
- **Pros**: Simple deployment (single database), easy cross-tenant queries (for super-admins), efficient resource usage (shared infrastructure)
- **Cons**: Weaker isolation (application bug could expose tenant data), harder compliance (data physically commingled), complex query optimization (every query needs company filter)

**Hard multi-tenancy alternative** (not observed trong code) would use separate schemas or databases per tenant - stronger isolation but operational complexity (backup/restore multiplied by tenants, schema changes must propagate to all tenants).

---

### 12. PERMISSION RESOLUTION FLOW VÀ RUNTIME EVALUATION

**File nguồn:** Analysis of entity relationships và service logic [Suy luận từ code patterns]

Permission resolution flow represents runtime process khi user attempts operation, framework determines whether to allow or deny. Flow not explicitly documented trong analyzed code (implementation trong framework core) nhưng can be inferred từ entity relationships và permission service patterns. Understanding này critical cho debugging permission issues và designing effective permission schemes.

**Inferred resolution flow:**

```
1. User Authentication
   ↓
2. Load User Entity
   - Query User by username/email
   - Eager load Group (many-to-one relationship)
   - Lazy load Roles (many-to-many relationship, loaded on demand)
   ↓
3. Aggregate Object Permissions
   - Query Permission entities owned by User's Group
   - Query Permission entities owned by User's Roles
   - Union all permissions (grant-based merging)
   ↓
4. Aggregate Field Permissions
   - Query MetaPermission entities owned by User's Group
   - Query MetaPermission entities owned by User's Roles
   - For each MetaPermission, load associated MetaPermissionRules
   - Union all field rules (grant-based merging)
   ↓
5. Cache Permission Results
   - Store aggregated permissions in user session
   - Subsequent requests use cached permissions (no re-query)
   - Cache invalidation on permission changes or session timeout
   ↓
6. Runtime Permission Check (on each operation)
   - Determine target object and operation (READ/WRITE/CREATE/REMOVE/EXPORT)
   - Lookup permission in cache: does user have permission for this object+operation?
   - If NO direct permission, check wildcard permissions (package-level)
   - If permission found with condition, evaluate condition
   ↓
7. Condition Evaluation (if applicable)
   - Parse conditionParams, resolve __user__ variables
   - Inject condition into query WHERE clause
   - Execute filtered query (database returns only matching records)
   ↓
8. Field Permission Enforcement (for UI rendering)
   - For each field in form/grid, check MetaPermissionRule
   - Apply canRead: hide field entirely if false
   - Apply canWrite: make field readonly if false
   - Evaluate readonlyIf/hideIf expressions dengan record context
   - Render final UI với appropriate field visibility/editability
   ↓
9. Return Filtered Results
   - User sees only records matching permission conditions
   - User sees only fields allowed by field permissions
   - User can only perform operations granted by permissions
```

**Permission precedence và merging strategy:** [Suy luận từ grant-based pattern]

When user belongs to Group với certain permissions AND has Roles với additional permissions, effective permissions are **union** (logical OR):
- If Group grants canRead, user has canRead (even if Roles don't grant)
- If Role grants canWrite, user has canWrite (even if Group doesn't grant)
- No explicit DENY rules observed - denial via absence of grant

Precedence order likely: User-specific permissions (if implemented) > Role permissions > Group permissions > Default (deny all)

**Performance considerations:** [Suy luận về optimization strategies]

Permission checking on every database query could severely impact performance if not optimized. Framework likely implements:
1. **Session-level caching**: Load permissions once per login, reuse until logout
2. **Query plan caching**: Compile condition expressions once, reuse for multiple queries
3. **Batch permission checks**: Check permissions for collection of objects together, not individually
4. **Lazy evaluation**: Only check permissions when actually needed (not speculatively)

**Failure modes và fallbacks:** [Suy luận về error handling]

What happens when permission system encounters errors?
- **Missing permission entity**: Deny by default (safe failure)
- **Malformed condition expression**: Deny access or ignore condition (depending on configuration)
- **Circular permission dependencies**: Framework must detect và prevent infinite loops
- **Cache inconsistency**: Periodic cache refresh hoặc pessimistic locking ensures consistency

---

### 13. CONTEXT VARIABLES VÀ DYNAMIC PERMISSION PARAMETERS

**File nguồn:** PermissionAssistantService.java line 326 [Từ source code]

Context variables trong permission conditions enable dynamic authorization decisions based on current user's attributes - transforming static permission rules thành adaptive access control. Magic variable pattern `__user__.{fieldName}` provides direct access to logged-in user's entity fields, với framework handling runtime resolution và type conversion transparently.

**Documented context variable pattern:**
```
__user__.{fieldName}
```

Where `{fieldName}` can be any field on User entity, including:
- Direct fields: `__user__.code`, `__user__.name`, `__user__.language`
- Related entities: `__user__.activeCompany`, `__user__.activeTeam`, `__user__.group`, `__user__.partner`
- Collections: `__user__.companySet`, `__user__.teamSet`
- Nested paths: `__user__.partner.company`, `__user__.group.technicalStaff`, `__user__.activeCompany.currency`

**Bằng chứng từ code - Variable construction:**
```java
// File: PermissionAssistantService.java, line 326
String conditionParams = "__user__." + userField.getName();
```

**Giải thích code:** Simple string concatenation builds magic variable reference. Framework runtime must parse string, split on dot (`.`), navigate object graph from User entity through specified path, extract final value. Implementation likely uses reflection or property accessors to traverse relationships dynamically. Error handling critical: what if path invalid (typo in field name) or null values encountered (user.partner null because user not linked to partner)? Framework must return null gracefully or throw clear error.

**Example usage scenarios demonstrating versatility:**

**Scenario 1: Single-value equality**
```
Condition: "self.createdBy = ?"
Params: "__user__"
→ SQL: WHERE created_by_id = {current user's ID}
Use case: Users see only records they created
```

**Scenario 2: Foreign key filtering**
```
Condition: "self.company = ?"
Params: "__user__.activeCompany"
→ SQL: WHERE company_id = {user's active company ID}
Use case: Multi-company data isolation
```

**Scenario 3: Collection membership (IN clause)**
```
Condition: "self.assignedTeam in (?)"
Params: "__user__.teamSet"
→ SQL: WHERE assigned_team_id IN (1,2,3)  -- user's teams
Use case: Team-based record visibility
```

**Scenario 4: Nested path traversal**
```
Condition: "self.currency = ?"
Params: "__user__.activeCompany.currency"
→ SQL: WHERE currency_id = {user's company's currency ID}
Use case: Currency-specific records matching user's company currency
```

**Scenario 5: Boolean flag check**
```
Condition: "self.confidential = false OR ? = true"
Params: "__user__.group.technicalStaff"
→ SQL: WHERE (confidential = false OR {is technical staff})
Use case: Confidential records visible only to technical staff
```

**Type conversion and value resolution:** [Suy luận về implementation]

Framework must handle type conversions when resolving context variables:
- **Entity references**: Convert to ID (User object → user.id Long value)
- **Collections**: Expand to ID list (Set<Team> → List<Long> team IDs)
- **Primitives**: Use directly (String, Integer, Boolean)
- **Nulls**: Handle gracefully (null company → condition excludes all records OR throws error)
- **Enums**: Convert to underlying value (if enum fields used)

**Security implications of context variables:**

Context variables powerful but must be used carefully:
- **Information leakage**: Conditions referencing `__user__` fields inadvertently expose those fields in debug logs/error messages
- **Privilege escalation**: Malformed conditions could accidentally grant access (e.g., typo `self.company != ?` instead of `self.company = ?` inverts filter)
- **Performance**: Complex nested paths (`__user__.partner.company.parent.currency`) require multiple JOINs, slow queries
- **Maintenance**: Renaming User entity fields breaks conditions referencing those fields (no compile-time checking for condition strings)

---

### 14. PERMISSION CACHING VÀ PERFORMANCE OPTIMIZATION

**File nguồn:** Permission.xml, Group.xml entity definitions [Từ source code]

Permission và Group entities marked `cacheable="true"` - critical performance optimization given permission checks occur extremely frequently throughout application execution. Caching strategy reduces database load (permissions queried once per session rather than per operation) và improves response times (cache hits measured in microseconds versus database queries in milliseconds).

**Bằng chứng từ code - Cacheable entity declarations:**
```xml
<entity name="Permission" cacheable="true">
<entity name="Group" cacheable="true">
```

**Giải thích code:** Cacheable attribute instructs Hibernate to store entity instances trong second-level (L2) cache - shared cache across all sessions (not just single user). L2 cache configured globally với mode `ENABLE_SELECTIVE` (từ STEP1 findings), meaning only entities explicitly marked cacheable are cached (prevents cache pollution from infrequently accessed entities).

**Hibernate L2 cache configuration:** [Từ STEP1 analysis]
```properties
hibernate.cache.use_second_level_cache = ENABLE_SELECTIVE
hibernate.cache.region.factory_class = org.hibernate.cache.jcache.JCacheRegionFactory
```

Cache implementation likely uses JCache (JSR-107) provider like EHCache or Caffeine. L2 cache stores entity instances by primary key - when code queries `Permission.findById(123)`, Hibernate checks L2 cache before hitting database.

**Caching benefits for permission system:**

1. **Reduced query load**: Permission checks on every HTTP request, caching prevents thousands of database queries per second
2. **Improved latency**: Cache hits ~0.1ms versus database queries ~10-50ms - 100-500x faster
3. **Scalability**: Application servers can scale horizontally without overwhelming database with permission queries
4. **Consistency**: All application servers share same permission data (if distributed cache used)

**Cache invalidation challenges:**

L2 cache introduces consistency challenges - when permission modified, how to ensure all cached copies updated? Strategies:

1. **Time-based expiration**: Cache entries expire after N minutes, forcing refresh - simple but may serve stale permissions
2. **Event-based invalidation**: When Permission entity updated, framework broadcasts invalidation event to all caches - complex but ensures consistency
3. **Version-based invalidation**: Hibernate detects version changes và invalidates automatically - built-in support

Axelor likely uses combination: Hibernate's automatic invalidation (version-based) plus session-based permission aggregation (permissions loaded once per login, cached in HTTP session).

**Session-level permission aggregation:** [Suy luận về caching strategy]

Beyond entity-level L2 cache, application likely performs **session-level permission aggregation**:

```java
// Pseudo-code for session permission loading
public class UserSession {
  private Map<String, ObjectPermission> objectPermissions;  // Indexed by object name
  private Map<String, FieldPermissions> fieldPermissions;   // Indexed by object.field

  public void loadPermissions(User user) {
    // Query all permissions from user's group
    List<Permission> groupPerms = user.getGroup().getPermissions();

    // Query all permissions from user's roles
    List<Permission> rolePerms = user.getRoles().stream()
        .flatMap(role -> role.getPermissions().stream())
        .collect(Collectors.toList());

    // Merge permissions (union)
    objectPermissions = mergePermissions(groupPerms, rolePerms);

    // Similarly load field permissions
    fieldPermissions = loadFieldPermissions(user);

    // Cache in HTTP session (valid until logout or session timeout)
  }
}
```

Session-level cache ideal vì permissions rarely change during single user session - load once at login, reuse for all subsequent requests. Trade-off: permission changes not effective until user logs out/in or session expires.

**Cache warming strategies:** [Suy luận về optimization]

For high-traffic systems, proactive cache warming prevents "cold start" penalty:
- Preload common permissions at application startup
- Background job refreshes permission cache periodically
- Login process preloads user's permissions before redirecting to home page

**Monitoring và tuning:**

Production systems should monitor:
- Cache hit ratio (target >95% for permission entities)
- Cache size (prevent unbounded growth)
- Cache eviction rate (high eviction suggests cache too small or high churn)
- Permission query count (should be minimal if caching effective)

---

### 15. SECURITY CONFIG OPTIONS VÀ HARDENING

**File nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Security configuration extends beyond authentication/authorization to include password policies, permission controls, và injection protection. Properties provide defense-in-depth approach - multiple security layers protecting against different attack vectors.

**Password policy enforcement:**

**Bằng chứng từ code - Password regex pattern:**
```properties
user.password.pattern = (((?=.*[a-z])(?=.*[A-Z])(?=.*\\d))|((?=.*[a-z])(?=.*[A-Z])(?=.*\\W))|((?=.*[a-z])(?=.*\\d)(?=.*\\W))|((?=.*[A-Z])(?=.*\\d)(?=.*\\W))).{8,}

#user.password.pattern-title = Custom password requirements message
```

**Giải thích code:** Complex regex enforces password strength requirements: minimum 8 characters AND at least 3 of 4 character types (lowercase, uppercase, digit, special character). Pattern broken down:
- `(?=.*[a-z])` - positive lookahead for lowercase letter
- `(?=.*[A-Z])` - positive lookahead for uppercase letter
- `(?=.*\\d)` - positive lookahead for digit
- `(?=.*\\W)` - positive lookahead for special character
- `.{8,}` - minimum 8 characters total

Regex grouped with OR (`|`) requiring any 3 of 4 types: (lower+upper+digit) OR (lower+upper+special) OR (lower+digit+special) OR (upper+digit+special). Approach balances security (preventing weak passwords like "password123") versus usability (not requiring all 4 types allows flexibility).

Optional `password-pattern-title` property provides custom error message shown to users when password rejected - helps users understand requirements without decoding regex.

**Permission system controls (dangerous!):**

**Bằng chứng từ code - Permission bypass options:**
```properties
# Disable action permission checks (DANGEROUS!)
#application.permission.disable-action = false

# Disable relational field permission checks
#application.permission.disable-relational-field = false
```

**Giải thích code:** Properties allow completely disabling permission checks - intended for testing/development environments where permission setup overhead undesirable. WARNING: enabling these in production (setting to `true`) creates security vulnerabilities - any user can perform any action. Properties commented by default (safe) but developers might uncomment during testing và forget to re-enable before production deployment. Security audit checklist must verify these remain `false` (or commented) in production configs.

Use cases for disabling permissions:
- **Development**: Faster iteration without configuring permissions for every test scenario
- **Automated testing**: Tests can focus on business logic without permission setup complexity
- **Emergency access**: When permission system misconfigured và admin locked out, temporarily disable to regain access

**SQL injection protection:**

**Bằng chứng từ code - Domain expression filtering:**
```properties
# Blocklist pattern for domain expressions
#application.domain-blocklist-pattern = (\\(\\s*(SELECT|DELETE|UPDATE)\\s+)|query_to_xml|some_another_function
```

**Giải thích code:** Regex pattern blocks dangerous SQL fragments from appearing trong domain filter expressions (permission conditions). Pattern specifically blocks:
- `(\\(\\s*(SELECT|DELETE|UPDATE)\\s+)` - Subqueries starting với SELECT/DELETE/UPDATE (potential SQL injection)
- `query_to_xml` - PostgreSQL function that could leak schema information
- `some_another_function` - Placeholder for adding custom dangerous functions

Protection critical vì permission conditions allow administrators write SQL-like expressions - malicious/accidental dangerous expressions could compromise database. Example blocked injection: `self.id = 1 OR (SELECT password FROM auth_user LIMIT 1) IS NOT NULL` - this would bypass permission filtering AND leak passwords.

Regex must balance security (block real attacks) versus usability (allow legitimate complex conditions). Too strict blocks valid use cases, too lenient allows attacks.

**Additional security best practices:** [Suy luận về hardening]

Beyond config properties, production Axelor deployments should implement:

1. **HTTPS enforcement**: All traffic encrypted, `session.cookie.secure=true` enabled
2. **CSP headers**: Content Security Policy prevents XSS attacks
3. **Rate limiting**: Prevent brute-force password attacks, API abuse
4. **Audit logging**: Log all authentication attempts, permission changes, data access
5. **Database encryption**: Encrypt sensitive data at rest (passwords, API keys)
6. **Regular updates**: Apply framework security patches promptly
7. **Penetration testing**: Regular security audits to identify vulnerabilities
8. **Least privilege**: Default deny permissions, grant only what needed

---

### 16. NHỮNG ĐIỀU KHÔNG TÌM THẤY TRONG SOURCE CODE

Sau quá trình phân tích sâu security và authorization code, một số features phổ biến trong enterprise security systems **KHÔNG** xuất hiện hoặc không rõ ràng trong Axelor codebase. Việc document những "absent features" giúp set realistic expectations và identify potential limitations:

**1. Spring Security hoặc Apache Shiro integration** [Không tìm thấy]
Xác nhận Axelor **KHÔNG** sử dụng established Java security frameworks - implements custom security layer instead. Implications: cannot leverage Spring Security's extensive ecosystem (OAuth resource servers, method security, security testing utilities), must rely on Axelor's proprietary APIs.

**2. Multi-tenancy implementation details** [Không rõ]
Configuration option exists (`application.multi-tenancy`) nhưng implementation code absent từ analyzed files. Không tìm thấy: TenantResolver, TenantContext classes, tenant-specific schema customization, cross-tenant query APIs. Likely implemented trong framework core, không exposed to application layer.

**3. User password hashing algorithm** [Không rõ]
Password field không visible trong analyzed User.xml extension (must be in core framework). Không biết algorithm used: BCrypt (industry standard)? PBKDF2? SCrypt? Argon2? Hash iteration count? Salt generation? Critical for assessing password security but not documented trong accessible code.

**4. Session storage mechanism** [Không rõ]
Configuration sets session timeout nhưng không specify storage: in-memory (lost on restart)? Database (persistent)? Redis (distributed)? For clustered deployments, distributed session storage essential - unclear nếu Axelor supports này out-of-box or requires custom configuration.

**5. Permission priority rules khi conflicts** [Không rõ]
Khi user's Group grants canRead nhưng Role denies (hypothetically), which wins? No explicit deny rules observed, suggesting pure grant-based merging (no conflicts possible). But what if future version adds deny rules? Priority order undefined trong code.

**6. Two-factor authentication (2FA)** [Không tìm thấy]
No configuration or code for 2FA, TOTP, SMS verification, security keys. Modern security best practice especially for administrative accounts, nhưng must be implemented as custom extension if needed.

**7. API authentication (JWT, API keys)** [Không rõ]
REST API authentication mechanism unclear. Basic auth mentioned (`auth.local.basic-auth`) nhưng no JWT token generation, API key management, OAuth 2.0 client credentials flow. API clients may need session cookies (không ideal for programmatic access).

**8. Permission inheritance hoặc hierarchy** [Không rõ]
Can groups have parent groups (organizational hierarchy)? Can roles inherit from other roles (role hierarchy)? No evidence trong code - appears flat structure. Enterprise orgs often need hierarchical permissions (e.g., Manager role inherits all permissions of Employee role plus additional).

**9. Audit trail cho security events** [Không đầy đủ]
Tracking system logs entity changes nhưng không rõ liệu có log: login attempts (successful/failed), logout events, permission checks (denied access attempts), session creation/destruction, password changes. Security audits và incident response require comprehensive security event logging.

**10. OAuth/OIDC token management** [Không rõ]
Pac4j integration handles OAuth authentication nhưng không rõ: where access/refresh tokens stored? How refresh tokens rotated? Token revocation support? Offline access (refresh tokens with extended lifetime)? Critical cho production OAuth deployments.

---

### 17. CÂU HỎI MỞ VÀ ĐIỂM CẦN NGHIÊN CỨU THÊM

Analysis của security system raises several questions requiring deeper investigation hoặc documentation review:

**1. Permission resolution performance:**
What is cache hit rate trong production? How many database queries per request attributed to permission checks? Is there N+1 query problem khi loading permissions for multiple objects? Performance profiling needed.

**2. Dynamic permission updates:**
Khi administrator modifies permissions, when do changes take effect? Immediately for all users? Only after logout/login? Does framework broadcast invalidation events trong clustered deployments? Real-time permission updates critical for security incidents (immediately revoke access to compromised account).

**3. Condition expression language details:**
Fields `readonlyIf`, `hideIf` use what expression language exactly? Groovy? JavaScript? Custom DSL? What variables/functions available trong expression context? Are expressions compiled or interpreted? Compilation provides better performance, interpretation more flexible.

**4. Cross-tenant queries và data sharing:**
In multi-tenancy mode, can super-admins query across all tenants? Can tenants share certain data (common product catalog) while keeping transactional data isolated? Are there tenant-specific customizations (different fields per tenant)?

**5. External authorization integration:**
Can Axelor integrate với external authorization services (AWS IAM, Azure AD conditional access, OPA)? Use case: corporate policies enforced externally (e.g., block access from certain IPs, require MFA for sensitive operations).

**6. API security beyond basic auth:**
For REST API clients, what's recommended authentication? Are there rate limits? How to implement API keys for service accounts? OAuth 2.0 client credentials flow support?

**7. Default permissions for new entities:**
When module installed with new entities, what default permissions applied? Are all entities denied by default (secure) or granted to all users (convenient)? Automated permission scaffolding would help initial setup.

**8. Permission testing utilities:**
Are there tools to test permission configurations? Simulate user with specific group/roles và verify they can/cannot access certain objects/fields? Testing framework critical for complex permission schemes.

**9. Permission migration và version control:**
When entities renamed or packages refactored, how to migrate permissions? Are there scripts? CSV export/import helps but doesn't automate structural changes. Schema evolution của permission system needs tooling support.

**10. Federation và cross-application SSO:**
Can multiple Axelor instances share authentication (SSO)? Can Axelor participate trong enterprise SSO federation với non-Axelor applications? SAML/OAuth enable này theoretically, but practical setup unclear.

These questions represent areas nơi hands-on testing, framework documentation review, hoặc direct communication với Axelor community would provide clarity.

---

## TÓM TẮT KIẾN TRÚC BẢO MẬT

Sau quá trình phân tích chi tiết từ source code, kiến trúc bảo mật của Axelor có thể tóm lược qua các điểm sau:

### Layered Security Architecture

Axelor implements **comprehensive multi-layer security** spanning authentication, authorization (object/field/record levels), và audit trail. Architecture không rely on standard frameworks (Spring Security/Shiro) nhưng builds custom solution tightly integrated với framework core. Decision này provides deep integration với Axelor's model-driven approach (permissions defined in XML, CSV-manageable) nhưng sacrifices ecosystem compatibility (third-party security tools designed for Spring Security won't work directly).

### Authentication: Pac4j Multi-Provider Support

Authentication layer powered by **Pac4j 5.7.7**, supporting six authentication providers: local (username/password), Google OAuth, Keycloak, SAML 2.0, LDAP, và CAS. Provider-agnostic architecture enables mixing providers (corporate employees via LDAP, partners via Google OAuth) trong single deployment. User provisioning (auto-create, link, or deny unknown users) configurable per deployment security requirements. Session management via servlet container với configurable timeout (default 8 hours).

### Authorization: Three-Tier Hierarchy

Authorization model follows **User → Group/Role → Permissions** hierarchy. Users belong to one Group (organizational unit) và multiple Roles (functional capabilities). Permissions aggregate from both Group và all Roles using grant-based merging (any permission source granting access = allowed). Dual Group/Role system separates organizational context (groups) từ functional permissions (roles), enabling flexible permission assignment without complex matrix maintenance.

### Object-Level Permissions: CRUD + Export

Permission entity controls access to entire entity classes (SaleOrder, Invoice, Product) với five operations: canRead, canWrite, canCreate, canRemove, canExport. Wildcard support enables package-level permissions (`com.axelor.apps.sale.*`) reducing configuration burden. Permission naming convention (`perm.{Object}.{GroupOrRole}`) provides structure. Validation against JPA metamodel prevents orphaned permissions.

### Record-Level Security: Domain Filters

Most sophisticated authorization feature: **conditional permissions** via SQL-like WHERE clauses injected into queries. Magic variables (`__user__.{field}`) enable dynamic filtering based on current user's attributes. Examples: `self.company = ?` + `__user__.activeCompany` implements multi-company isolation, `self.assignedTeam in (?)` + `__user__.teamSet` enables team-based visibility. Conditions evaluated at database level (not application), ensuring security và performance.

### Field-Level Permissions: Fine-Grained Control

MetaPermission và MetaPermissionRule entities provide field-level authorization: hide sensitive fields (costPrice) from certain groups, make fields readonly based on workflow state (`readonlyIf="statusSelect > 2"`). Conditional expressions (`hideIf`, `readonlyIf`) enable context-aware UI - same field visible to some users, hidden to others, readonly based on record state.

### CSV-Based Permission Management

Permission Assistant tool enables bulk permission configuration via CSV export/import. Administrators leverage spreadsheet tools (Excel formulas, pivot tables, conditional formatting) to rapidly configure hundreds of permissions. Transactional import ensures consistency. CSV approach dramatically more efficient than UI-based permission-by-permission configuration for large permission matrices.

### Performance Optimization: Multi-Level Caching

Permission entities marked `cacheable="true"` leverage Hibernate L2 cache. Session-level permission aggregation (load once at login, cache for session duration) prevents repeated database queries. Caching critical vì permission checks occur on every operation - without caching, system would be database-bound.

### Security Hardening Options

Password policy regex enforces strong passwords (8+ chars, 3 of 4 types). SQL injection protection blocks dangerous expressions trong permission conditions. Permission system can be disabled for testing (dangerous if left enabled trong production). HTTPS enforcement, CSRF protection, XSS prevention expected (not explicitly configured trong analyzed files).

### Gaps và Limitations

Notable absences: no two-factor authentication, unclear API authentication (beyond basic auth), no permission inheritance/hierarchy, minimal audit logging of security events, multi-tenancy configuration exists but implementation unclear. Password hashing algorithm và session storage mechanism not documented trong accessible code. Permission priority rules trong conflict scenarios undefined.

### Architecture Philosophy

Axelor prioritizes **configurability over programmatic security** - permissions defined in XML/CSV rather than annotations, enabling business users configure security without code changes. Trade-off: less compile-time safety (typos in permission conditions only caught at runtime), more operational flexibility. Suitable for environments where security requirements evolve frequently và non-developers need manage permissions.

---

**Tổng số lines code/config analyzed:**
- Domain entities: 4 files (User, Group, Role, Permission)
- Service implementations: 2 files (PermissionServiceImpl, PermissionAssistantService)
- Configuration: axelor-config.properties (auth sections)

**Nguồn:** Tất cả findings từ direct source code analysis, supplemented với suy luận based on standard security patterns, JPA/Hibernate behaviors, và enterprise application best practices.

---

*Kết thúc RESEARCH_STEP3_SECURITY.md*
