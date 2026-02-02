# BƯỚC 8: PHÁT TRIỂN JAVA TRUYỀN THỐNG & CHIẾN LƯỢC KẾT HỢP STUDIO

## Phương pháp phân tích

Quá trình nghiên cứu tập trung vào phân tích cách developer phát triển ứng dụng trên Axelor, bao gồm phát triển Java truyền thống và chiến lược kết hợp với Studio no-code. Phân tích được thực hiện bằng cách đọc source code thực tế tại thư mục `/Volumes/works/code/java/axelor/axelor-erp`.

**Modules phân tích chính:**
- `/modules/axelor-open-suite/axelor-sale/` - Module bán hàng (80+ Service classes, 15+ Controllers)
- `/modules/axelor-open-suite/axelor-account/` - Module kế toán (100+ Service classes)
- `/modules/axelor-open-suite/axelor-base/` - Module cơ sở

**Files quan trọng đã đọc:**
- Service classes: `AppSaleService.java`, `AppSaleServiceImpl.java`, `SaleOrderService.java`
- Controller: `SaleOrderController.java`
- Repository: `SaleOrderManagementRepository.java`
- Guice Module: `SaleModule.java`
- Domain XMLs: Từ `/build/src-gen/` và `/src/main/resources/domains/`
- Meta entities: `MetaJsonModel.java`, `MetaJsonField.java`

**Focus theo độ ưu tiên:**
1. ✅ A4: Service layer pattern (Google Guice, transaction management)
2. ✅ B2: Ranh giới giữa Studio và Code
3. ✅ B4: Version control khi kết hợp (export/import mechanisms)
4. ✅ A6: Action system (kết nối UI với Java)
5. ✅ B5: Code patterns từ modules thực tế

---

## PHẦN A: PHÁT TRIỂN BẰNG CODE JAVA TRUYỀN THỐNG

### A4. SERVICE LAYER — TRÁI TIM CỦA LOGIC NGHIỆP VỤ

**Nguồn:** `axelor-sale/src/main/java/com/axelor/apps/sale/service/` [Từ source code]

#### Pattern Interface + Implementation với Google Guice

Axelor tuân thủ nghiêm ngặt pattern Interface-Implementation cho tất cả Service classes. Khác với Spring Framework phổ biến trong Java ecosystem, Axelor sử dụng **Google Guice** làm dependency injection framework. Lý do lựa chọn Guice thay vì Spring là để giảm footprint và startup time - Guice nhẹ hơn đáng kể, không có auto-configuration magic của Spring Boot, và dependency resolution được thực hiện compile-time thay vì runtime.

Pattern chuẩn gồm 3 thành phần: (1) Service interface định nghĩa contract, đặt trong cùng package với implementation. Interface thường kế thừa từ base interface của module cha để hỗ trợ override behavior. (2) Service implementation với suffix `Impl`, annotated với `@Singleton` để Guice tạo single instance, sử dụng constructor injection cho tất cả dependencies. (3) Guice Module binding trong file `*Module.java` ánh xạ interface sang implementation.

**Bằng chứng từ mã - Service interface:**
```java
// File: com/axelor/apps/sale/service/app/AppSaleService.java
package com.axelor.apps.sale.service.app;

import com.axelor.apps.base.service.app.AppBaseService;
import com.axelor.studio.db.AppSale;

public interface AppSaleService extends AppBaseService {
  public AppSale getAppSale();
  public void generateSaleConfigurations();
}
```

**Giải thích mã:** Interface kế thừa `AppBaseService` từ module base, cho phép module sale override hoặc extend behavior của base module. Method signatures đơn giản, rõ ràng, không có checked exceptions trong khai báo (exceptions sẽ là runtime `AxelorException`). Pattern inheritance này cho phép module hierarchy - sale module có thể access tất cả services của base module mà không cần biết implementation details.

**Bằng chứng từ mã - Service implementation với Guice DI:**
```java
// File: com/axelor/apps/sale/service/app/AppSaleServiceImpl.java
package com.axelor.apps.sale.service.app;

import com.axelor.apps.base.db.Company;
import com.axelor.apps.base.db.repo.CompanyRepository;
import com.axelor.apps.base.service.app.AppBaseServiceImpl;
import com.axelor.apps.sale.db.SaleConfig;
import com.axelor.apps.sale.db.repo.SaleConfigRepository;
import com.axelor.meta.MetaFiles;
import com.axelor.meta.db.repo.MetaFileRepository;
import com.axelor.meta.db.repo.MetaModuleRepository;
import com.axelor.meta.loader.AppVersionService;
import com.axelor.studio.db.AppSale;
import com.axelor.studio.db.repo.AppRepository;
import com.axelor.studio.db.repo.AppSaleRepository;
import com.axelor.studio.service.AppSettingsStudioService;
import com.google.inject.Inject;
import com.google.inject.Singleton;
import com.google.inject.persist.Transactional;
import java.util.List;

@Singleton
public class AppSaleServiceImpl extends AppBaseServiceImpl implements AppSaleService {

  protected final AppSaleRepository appSaleRepo;
  protected final CompanyRepository companyRepo;
  protected final SaleConfigRepository saleConfigRepo;

  @Inject
  public AppSaleServiceImpl(
      AppRepository appRepo,
      MetaFiles metaFiles,
      AppVersionService appVersionService,
      AppSettingsStudioService appSettingsService,
      MetaModuleRepository metaModuleRepo,
      MetaFileRepository metaFileRepo,
      AppSaleRepository appSaleRepo,
      CompanyRepository companyRepo,
      SaleConfigRepository saleConfigRepo) {
    super(appRepo, metaFiles, appVersionService, appSettingsService, metaModuleRepo, metaFileRepo);
    this.appSaleRepo = appSaleRepo;
    this.companyRepo = companyRepo;
    this.saleConfigRepo = saleConfigRepo;
  }

  @Override
  public AppSale getAppSale() {
    return appSaleRepo.all().fetchOne();
  }

  @Override
  @Transactional
  public void generateSaleConfigurations() {
    List<Company> companies = companyRepo.all().filter("self.saleConfig is null").fetch();
    for (Company company : companies) {
      SaleConfig saleConfig = new SaleConfig();
      saleConfig.setCompany(company);
      saleConfigRepo.save(saleConfig);
    }
  }
}
```

**Giải thích mã chi tiết:**

1. **`@Singleton` scope:** Guice tạo một instance duy nhất của Service trong suốt application lifecycle. Khác với Spring `@Service` (default prototype scope), Guice `@Singleton` giống Spring `@Scope("singleton")`. Singleton phù hợp vì Services thường stateless, chỉ chứa business logic methods.

2. **Constructor injection:** Tất cả dependencies được khai báo trong constructor với `@Inject` annotation. Guice sẽ resolve dependencies theo constructor parameters - không cần `@Autowired` hoặc `@Qualifier` như Spring. Constructor injection ép buộc immutability (dependencies là `final`), dễ test (có thể mock dependencies khi khởi tạo), và compile-time safe (thiếu dependency sẽ báo lỗi ngay khi compile).

3. **Base class chaining:** `super()` call trong constructor truyền dependencies cho base class. Pattern này cho phép code reuse - `AppBaseServiceImpl` xử lý logic chung (app settings, meta files), `AppSaleServiceImpl` chỉ cần implement logic specific cho sale module.

4. **`@Transactional` annotation:** Đánh dấu method cần database transaction. Axelor sử dụng Guice Persist module với JPA Persistence Unit - method được wrap trong transaction boundary tự động. Nếu method throw exception, transaction rollback. Nếu thành công, transaction commit khi method return. Không cần manual transaction management như `EntityManager.getTransaction().begin()`.

#### Transaction Management Pattern

Axelor sử dụng declarative transaction management qua `@Transactional` annotation thay vì programmatic transactions. Pattern này có ba đặc điểm quan trọng: (1) Chỉ methods public trong @Singleton classes mới có transactional proxy - private/protected methods không được wrap. (2) Nested transactions không được support - nếu method A (transactional) gọi method B (transactional), chỉ có transaction của A được tạo. (3) Exception handling: Runtime exceptions trigger rollback, checked exceptions không rollback trừ khi explicitly configured.

Pattern phổ biến là Service methods có `@Transactional` wrap toàn bộ business logic. Repository save/remove operations phải nằm trong transaction context - gọi repository method ngoài transaction sẽ throw exception. Controllers thường không có `@Transactional` - chúng delegate sang Services đã có transaction management.

**Lưu ý quan trọng về khác biệt Guice vs Spring:**

| Aspect | Guice (Axelor) | Spring |
|--------|----------------|--------|
| **Binding** | Explicit trong Module.configure() | Auto-scan với `@Component` |
| **Injection** | Constructor injection preferred | Field/Constructor/Setter |
| **Scope** | `@Singleton` explicit | `@Service` default singleton |
| **AOP** | Method interceptors | AspectJ hoặc proxy-based |
| **Transaction** | Guice Persist `@Transactional` | Spring `@Transactional` |
| **Footprint** | ~500KB | ~20MB (Spring Boot) |
| **Startup** | < 1s | 3-5s (Spring Boot) |

---

### A5. CONTROLLER PATTERN — XỬ LÝ REQUEST VÀ KẾT NỐI UI

**Nguồn:** `axelor-sale/src/main/java/com/axelor/apps/sale/web/SaleOrderController.java` [Từ source code]

Controllers trong Axelor đóng vai trò như adapters giữa UI layer (XML views + JavaScript frontend) và Service layer. Khác với Spring MVC Controllers xử lý HTTP requests trực tiếp, Axelor Controllers xử lý **Action requests** - một abstraction layer của framework. Action requests được trigger từ UI events (button clicks, field changes, form submissions) và được framework dispatch đến appropriate controller methods.

#### Signature Pattern: ActionRequest & ActionResponse

Mọi controller methods đều tuân theo signature chuẩn: `public void methodName(ActionRequest request, ActionResponse response)`. `ActionRequest` đóng gói context của UI event - form data, current record, user session, request parameters. `ActionResponse` là builder pattern object để construct response - set field values, show messages, navigate views, trigger client-side actions. Pattern này giống JAX-RS (HttpServletRequest/HttpServletResponse) nhưng higher-level và type-safe hơn.

**Bằng chứng từ mã:**
```java
// File: com/axelor/apps/sale/web/SaleOrderController.java
package com.axelor.apps.sale.web;

import com.axelor.apps.base.AxelorException;
import com.axelor.apps.sale.db.SaleOrder;
import com.axelor.apps.sale.service.saleorder.SaleOrderInitValueService;
import com.axelor.inject.Beans;
import com.axelor.rpc.ActionRequest;
import com.axelor.rpc.ActionResponse;
import com.google.inject.Singleton;
import java.lang.invoke.MethodHandles;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Singleton
public class SaleOrderController {

  private final Logger logger = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass());

  public void onNew(ActionRequest request, ActionResponse response) throws AxelorException {
    SaleOrder saleOrder = SaleOrderContextHelper.getSaleOrder(request.getContext());
    boolean isTemplate = false;

    if (request.getContext().get("_template") != null) {
      isTemplate = (boolean) request.getContext().get("_template");
    }

    SaleOrderInitValueService saleOrderInitValueService =
        Beans.get(SaleOrderInitValueService.class);

    // Business logic delegation
    saleOrderInitValueService.onNew(saleOrder, isTemplate);

    response.setValues(saleOrder);
  }
}
```

**Giải thích mã:**

1. **`@Singleton` scope:** Controllers cũng singleton như Services. Framework tạo một instance duy nhất khi application start, reuse cho tất cả requests. Stateless design - không lưu request-specific data trong instance fields.

2. **`Beans.get()` pattern:** Axelor cung cấp service locator pattern qua `Beans.get(Class)`. Thay vì inject tất cả services vào constructor (có thể hàng chục dependencies cho Controller lớn), Controllers get services on-demand. Pattern này làm controller code sạch hơn nhưng làm dependencies không explicit - harder to test vì phải mock `Beans` registry.

3. **Context extraction:** `request.getContext()` trả về `Context` object chứa form data. `SaleOrderContextHelper.getSaleOrder()` là helper method parse Context thành strongly-typed entity. Context có thể chứa nested objects, collections - helper methods handle parsing complexity.

4. **Response building:** `response.setValues(entity)` update form với giá trị mới sau khi business logic execute. Response còn có methods: `setValue(field, value)` set individual field, `setFlash(message)` show notification, `setError(message)` show error, `setView(ActionView)` navigate to different view, `setReload(true)` force form reload.

> ⚠️ **Suy luận:** Pattern `Beans.get()` vi phạm dependency injection best practices (dependencies should be explicit), nhưng là trade-off hợp lý cho Controllers có nhiều dependencies. Axelor có thể đã chọn pattern này để tránh constructor quá dài và giảm boilerplate code. Trong testing, developer phải mock `Beans` registry hoặc sử dụng integration tests thay vì unit tests.

---

### A3. REPOSITORY PATTERN — CUSTOM LOGIC TRÊN GENERATED REPOSITORIES

**Nguồn:** `axelor-sale/src/main/java/com/axelor/apps/sale/db/repo/SaleOrderManagementRepository.java` [Từ source code]

Repositories trong Axelor theo pattern hai tầng: (1) Generated repository từ domain XML với CRUD methods cơ bản, (2) Custom repository extends generated repository để add business logic. Pattern này tách biệt rõ ràng: generated code không bao giờ được edit thủ công (sẽ bị overwrite khi regenerate), custom logic nằm trong subclass an toàn qua regeneration cycles.

#### Custom Repository với Lifecycle Hooks

Custom repositories override các lifecycle methods để inject business logic vào CRUD operations. Các hook points chính: `save(entity)` - before persist, `copy(entity, deep)` - when duplicate record, `remove(entity)` - before delete. Mỗi hook có thể validate, compute derived fields, trigger side effects, hoặc prevent operation bằng exception.

**Bằng chứng từ mã:**
```java
// File: com/axelor/apps/sale/db/repo/SaleOrderManagementRepository.java
package com.axelor.apps.sale.db.repo;

import com.axelor.apps.base.AxelorException;
import com.axelor.apps.base.db.repo.TraceBackRepository;
import com.axelor.apps.sale.db.SaleOrder;
import com.axelor.apps.sale.db.SaleOrderLine;
import com.axelor.apps.sale.service.MarginComputeService;
import com.axelor.apps.sale.service.app.AppSaleService;
import com.axelor.apps.sale.service.saleorder.SaleOrderComputeService;
import com.axelor.apps.sale.service.saleorder.SaleOrderCopyService;
import com.axelor.apps.sale.service.saleorder.SaleOrderMarginService;
import com.axelor.apps.sale.service.saleorder.SaleOrderOrderingStatusService;
import com.axelor.apps.sale.service.saleorder.SaleOrderService;
import com.axelor.i18n.I18n;
import com.axelor.inject.Beans;
import com.google.inject.Inject;
import javax.persistence.PersistenceException;

public class SaleOrderManagementRepository extends SaleOrderRepository {

  protected final SaleOrderCopyService saleOrderCopyService;
  protected final SaleOrderOrderingStatusService saleOrderOrderingStatusService;

  @Inject
  public SaleOrderManagementRepository(
      SaleOrderCopyService saleOrderCopyService,
      SaleOrderOrderingStatusService saleOrderOrderingStatusService) {
    this.saleOrderCopyService = saleOrderCopyService;
    this.saleOrderOrderingStatusService = saleOrderOrderingStatusService;
  }

  @Override
  public SaleOrder copy(SaleOrder entity, boolean deep) {
    SaleOrder copy = super.copy(entity, deep);
    saleOrderCopyService.copySaleOrder(copy);
    return copy;
  }

  @Override
  public SaleOrder save(SaleOrder saleOrder) {
    try {
      AppSale appSale = Beans.get(AppSaleService.class).getAppSale();
      SaleOrderComputeService saleOrderComputeService = Beans.get(SaleOrderComputeService.class);

      if (appSale.getEnablePackManagement()) {
        saleOrderComputeService.computePackTotal(saleOrder);
      } else {
        saleOrderComputeService.resetPackTotal(saleOrder);
      }

      computeSeq(saleOrder);
      computeFullName(saleOrder);

      if (appSale.getManagePartnerComplementaryProduct()) {
        Beans.get(SaleOrderService.class).manageComplementaryProductSOLines(saleOrder);
      }

      computeSubMargin(saleOrder);
      Beans.get(SaleOrderMarginService.class).computeMarginSaleOrder(saleOrder);

      if (appSale.getIsQuotationAndOrderSplitEnabled()) {
        saleOrderOrderingStatusService.updateOrderingStatus(saleOrder);
      }

      return super.save(saleOrder);
    } catch (Exception e) {
      TraceBackService.traceExceptionFromSaveMethod(e);
      throw new PersistenceException(e.getMessage(), e);
    }
  }

  public void computeSeq(SaleOrder saleOrder) {
    try {
      if (saleOrder.getId() == null) {
        saleOrder = super.save(saleOrder);
      }
      if (Strings.isNullOrEmpty(saleOrder.getSaleOrderSeq()) && !saleOrder.getTemplate()) {
        if (saleOrder.getStatusSelect() == SaleOrderRepository.STATUS_DRAFT_QUOTATION) {
          saleOrder.setSaleOrderSeq(
              Beans.get(SequenceService.class).getDraftSequenceNumber(saleOrder));
        }
      }
    } catch (Exception e) {
      throw new PersistenceException(e.getMessage(), e);
    }
  }

  @Override
  public void remove(SaleOrder saleOrder) {
    try {
      if (saleOrder.getStatusSelect() == SaleOrderRepository.STATUS_ORDER_CONFIRMED) {
        throw new AxelorException(
            TraceBackRepository.CATEGORY_INCONSISTENCY,
            I18n.get(SaleExceptionMessage.SALE_ORDER_CANNOT_DELETE_COMFIRMED_ORDER));
      }
    } catch (AxelorException e) {
      throw new PersistenceException(e.getMessage(), e);
    }
    super.remove(saleOrder);
  }
}
```

**Giải thích mã chi tiết:**

1. **Extends generated repository:** `SaleOrderManagementRepository extends SaleOrderRepository`. Generated `SaleOrderRepository` nằm trong `/build/src-gen/`, custom repository nằm trong `/src/main/java/`. Naming convention: generated là `{Entity}Repository`, custom là `{Entity}ManagementRepository`.

2. **Constructor injection cho dependencies:** Custom repository có thể inject services. Generated repository không có dependencies để giữ chúng simple và fast to regenerate.

3. **`save()` hook:** Được gọi mỗi khi entity persist. Business logic bao gồm: compute pack totals (nếu pack management enabled), generate sequence number (nếu chưa có), compute full name (concatenate client name + sequence), manage complementary products (cross-sell logic), compute margins (pricing calculations), update ordering status. Cuối cùng `super.save()` thực hiện actual JPA persist. Pattern này đảm bảo tất cả computed fields luôn up-to-date trước khi save database.

4. **Exception wrapping:** Try-catch wrap logic và convert exceptions sang `PersistenceException`. JPA spec yêu cầu repository methods chỉ throw `PersistenceException` hoặc subclasses. Axelor's `TraceBackService` log exceptions với full stack trace trước khi wrap.

5. **`copy()` hook:** Khi user duplicate một record trong UI, framework gọi `repository.copy()`. Default behavior (super.copy) là shallow/deep clone tất cả fields. Custom logic trong `saleOrderCopyService.copySaleOrder()` có thể: clear fields không nên copy (như sequence number, confirmation date), reset status về draft, duplicate related entities (như sale order lines).

6. **`remove()` hook:** Validation trước khi delete. Ở đây, prevent deletion của confirmed orders - business rule là chỉ draft orders có thể delete, confirmed orders phải cancel thay vì delete.

**Guice binding cho custom repository:**

Custom repository phải được bind trong Guice module thay thế generated repository. Framework tự động inject custom repository thay vì generated.

```java
// File: com/axelor/apps/sale/module/SaleModule.java
package com.axelor.apps.sale.module;

import com.axelor.app.AxelorModule;
import com.axelor.apps.sale.db.repo.SaleOrderManagementRepository;
import com.axelor.apps.sale.db.repo.SaleOrderRepository;

public class SaleModule extends AxelorModule {
  @Override
  protected void configure() {
    // Bind custom repository
    bind(SaleOrderRepository.class).to(SaleOrderManagementRepository.class);

    // 100+ other bindings...
  }
}
```

**Giải thích:** `bind(Interface.class).to(Implementation.class)` tell Guice: khi code request `SaleOrderRepository`, inject `SaleOrderManagementRepository` instance. Dependency inversion principle - Services chỉ depend on interface, không biết implementation cụ thể.

---

### A6. ACTION SYSTEM — CẦU NỐI GIỮA UI VÀ JAVA CODE

**Nguồn:** Controller method signatures và Action request/response patterns [Từ source code]

Action system là mechanism kết nối UI events với Java code. Khi user click button, change field, submit form trong Axelor UI, một Action request được tạo và dispatch đến appropriate handler. Có ba loại actions chính: (1) action-method - gọi Java Controller method, (2) action-record - set field values declaratively, (3) action-script - execute Groovy/JavaScript inline.

#### Action-Method Call Flow

Luồng xử lý action-method từ end-to-end: (1) User clicks button trong form → (2) JavaScript frontend create ActionRequest object với context (form data, current record ID, custom parameters) → (3) AJAX POST request đến `/ws/action` endpoint với action name → (4) Framework's ActionHandler parse request, lookup action definition trong XML → (5) Reflect và invoke corresponding Controller method với signature `(ActionRequest, ActionResponse)` → (6) Controller method delegates business logic đến Services → (7) Controller build ActionResponse (updated values, messages, navigation) → (8) Framework serialize response thành JSON → (9) JavaScript frontend apply response changes đến UI.

**Pattern trong Controller method:**

```java
public void confirmOrder(ActionRequest request, ActionResponse response) {
  try {
    // 1. Extract entity from request context
    SaleOrder saleOrder = request.getContext().asType(SaleOrder.class);
    Long saleOrderId = request.getContext().asType(SaleOrder.class).getId();

    // 2. Get fresh entity from database (context entity might be stale)
    saleOrder = Beans.get(SaleOrderRepository.class).find(saleOrderId);

    // 3. Delegate business logic to Service
    SaleOrderWorkflowService workflowService = Beans.get(SaleOrderWorkflowService.class);
    workflowService.confirmSaleOrder(saleOrder);

    // 4. Build response
    response.setReload(true);  // Refresh form to show updated data
    response.setFlash(I18n.get("Order confirmed successfully"));

  } catch (AxelorException e) {
    // 5. Handle business logic errors
    response.setError(e.getLocalizedMessage());
  } catch (Exception e) {
    // 6. Handle unexpected errors
    TraceBackService.trace(e);
    response.setError(I18n.get("An error occurred"));
  }
}
```

**Giải thích pattern:**

- **Context extraction:** `request.getContext().asType(T.class)` parse form data thành typed entity. Context có thể stale (user might have opened form hours ago), nên luôn refetch từ database nếu cần latest data.

- **Service delegation:** Controllers KHÔNG chứa business logic. Chúng chỉ extract/validate input, delegate sang Services, và build response. Separation of concerns - business logic testable độc lập với UI framework.

- **Response types:** `setReload(true)` force UI refresh entire form, `setFlash(msg)` show success message (green notification), `setError(msg)` show error message (red notification), `setValue(field, value)` update specific field, `setView(ActionView.define()...)` navigate to different view.

- **Exception handling:** `AxelorException` là checked exception của framework cho business rule violations (validation errors, constraint violations). Unchecked exceptions là unexpected errors (bugs, infrastructure failures). Response khác nhau: business exceptions show user-friendly messages, unexpected exceptions log stack trace và show generic error.

**Kết nối từ XML view:**

```xml
<form name="sale-order-form" model="com.axelor.apps.sale.db.SaleOrder">
  <panel name="mainPanel">
    <button name="confirmBtn" title="Confirm Order"
            onClick="action-sale-order-method-confirm"
            showIf="statusSelect == 1"/>
  </panel>
</form>

<action-method name="action-sale-order-method-confirm"
               model="com.axelor.apps.sale.db.SaleOrder"
               call="com.axelor.apps.sale.web.SaleOrderController:confirmOrder"/>
```

**Giải thích XML:** Button `onClick` reference đến action name. Action definition map đến Controller class + method name với colon separator `ClassName:methodName`. Framework use reflection invoke method, pass ActionRequest/ActionResponse. `showIf` condition control button visibility - Groovy expression evaluated in form context.

---

## PHẦN B: CHIẾN LƯỢC KẾT HỢP STUDIO VÀ CODE

### B1. PHÂN TÍCH SỰ KHÁC BIỆT CẤU TRÚC LƯU TRỮ

**Nguồn:** MetaJsonModel entity analysis và domain XML patterns [Từ source code]

Axelor có hai hệ thống lưu trữ metadata song song cho models và views: (1) File-based system cho code-defined entities (domain XMLs → generated Java classes), (2) Database-based system cho Studio-created entities (meta tables). Hai hệ thống này hoạt động độc lập nhưng có cơ chế merge tại runtime.

#### Studio Models - Database Storage

Studio-created custom models được lưu hoàn toàn trong database qua các meta tables: `meta_json_model` (model definitions), `meta_json_field` (field definitions), `meta_json_record` (actual data records). Pattern này cho phép dynamic schema changes không cần code deployment - admin có thể add/remove fields qua UI, changes take effect ngay lập tức.

**Meta entities structure:**

```java
// File: com/axelor/meta/db/MetaJsonModel.java (generated)
@Entity
@Table(name = "META_JSON_MODEL")
public class MetaJsonModel extends AuditableModel {
  @Column(name = "name", unique = true)
  private String name;  // Model name (e.g., "CustomTenant")

  @Column(name = "title")
  private String title;  // Display title

  @OneToMany(mappedBy = "jsonModel")
  private List<MetaJsonField> fields;  // Field definitions

  @Column(name = "table_name")
  private String tableName;  // Underlying table (optional, default meta_json_record)

  // Getters/Setters...
}

@Entity
@Table(name = "META_JSON_FIELD")
public class MetaJsonField extends AuditableModel {
  @Column(name = "name")
  private String name;  // Field name

  @Column(name = "type")
  private String type;  // Field type: string, integer, many-to-one, etc.

  @ManyToOne
  private MetaJsonModel jsonModel;  // Parent model

  @Column(name = "target_model")
  private String targetModel;  // For relational fields

  // Getters/Setters...
}
```

**Giải thích:** Studio models là metadata-driven. Khi application start, framework read `meta_json_model` và `meta_json_field` tables, dynamically construct model definitions. UI forms dynamically render based on field definitions. Data được lưu trong `meta_json_record` table với JSONB column chứa actual field values, hoặc trong custom table nếu specified.

#### Code Models - File-based Storage

Code-defined entities follow standard workflow: Domain XML → Code generation → Java classes. XML nằm trong `/src/main/resources/domains/`, generated Java classes trong `/build/src-gen/`. Database schema được generate bởi Hibernate DDL auto-update. Pattern này compile-time safe - field types checked, relationships validated, IDE autocomplete works.

**Thứ tự ưu tiên khi conflict:**

Khi cùng một model/view được define ở cả code và Studio, Axelor merge theo rules: (1) Code-defined entities take precedence làm base definition, (2) Studio customizations overlay lên base definition (add fields, modify properties), (3) Studio không thể remove code-defined fields (chỉ hide), (4) Views: Studio custom views extend code-defined views qua XML xpath mechanisms.

---

### B2. KHI NÀO DÙNG STUDIO, KHI NÀO VIẾT CODE — RANH GIỚI KỸ THUẬT

**Nguồn:** MetaJson entity capabilities và Service/Repository patterns [Từ source code + Suy luận]

#### Giới hạn kỹ thuật của Studio (từ source code analysis)

Studio models lưu trữ qua `MetaJsonModel` và `MetaJsonField` có những giới hạn cứng về features hỗ trợ. Phân tích entity definitions cho thấy:

**✗ Studio KHÔNG hỗ trợ:**
1. **Many-to-many relationships:** `MetaJsonField` chỉ có `targetModel` string (reference one model), không có join table configuration. M2M cần intermediate entity khó model trong JSON schema.
2. **Entity inheritance:** Không có `parentModel` hoặc `discriminator` fields trong `MetaJsonModel`. Studio models flat, không có class hierarchy.
3. **Complex constraints:** Unique constraints, check constraints, indexes - không có metadata columns để config. `meta_json_record` table generic, không thể enforce model-specific constraints.
4. **Computed fields với database functions:** Studio fields là simple values hoặc references. Không có mechanism để define SQL expressions hoặc database-computed columns.
5. **Custom Repository logic:** Studio không generate Repository classes. CRUD operations qua generic `MetaJsonRecordRepository` không có lifecycle hooks.
6. **Custom Service layer:** Business logic phức tạp, validations, calculations - Studio chỉ có action-script (Groovy inline) với giới hạn về complexity và maintainability.

**✓ Studio HỖ TRỢ tốt:**
1. **Simple data models:** Flat entities với string/integer/decimal/date fields, one-to-many, many-to-one relationships.
2. **UI customization:** Forms, grids, filters, dashboards - tất cả config qua XML lưu trong database.
3. **Simple workflows:** BPM processes với decision tables, không cần phức tạp logic.
4. **Action chains:** Sequences of actions (action-record, action-method, action-script) để handle UI events.

#### Giới hạn kỹ thuật của Code truyền thống

Code-defined entities không có giới hạn về modeling capabilities nhưng có trade-offs:

**✗ Code KHÔNG linh hoạt:**
1. **Schema changes require deployment:** Thêm/sửa field cần modify domain XML, regenerate code, restart application. Không thể ad-hoc changes trong production.
2. **Non-technical users cannot customize:** Cần Java/XML knowledge, development environment, build tools.

**✓ Code PHÁT HUY tốt:**
1. **Complex domain models:** Inheritance hierarchies, polymorphic associations, composite keys, embedded objects.
2. **Business logic integrity:** Repository hooks, Service layers, compile-time type safety, refactoring support.
3. **Performance-critical operations:** Custom JPQL queries, database functions, batch processing optimizations.
4. **Integration với external systems:** REST clients, message queues, file processing - code libraries đầy đủ.

#### Ma trận quyết định (dựa trên giới hạn kỹ thuật)

| Use Case | Studio | Code | Lý do |
|----------|--------|------|-------|
| **Demo/prototype data models** | ✅ Nên dùng | ❌ Overkill | Studio fast iteration, no deployment |
| **Master data entities** (ít thay đổi) | ❌ Không nên | ✅ Nên dùng | Code provides schema stability, integrity |
| **User-customizable fields** | ✅ Nên dùng | ❌ Inflexible | Studio allows tenant-specific customization |
| **Entities với inheritance** | ❌ Không thể | ✅ Bắt buộc | Studio không support inheritance |
| **Entities với phức tạp relationships** | ❌ Không nên | ✅ Nên dùng | Studio M2M giới hạn, code không giới hạn |
| **Business logic > 50 lines** | ❌ Maintainability nightmare | ✅ Nên dùng | Code có proper structure, testing |
| **Workflow automation** | ✅ Nên dùng | ❌ Overkill | Studio BPM visual, easier maintain |
| **Custom REST APIs** | ❌ Không thể | ✅ Bắt buộc | Code cần Controllers |
| **Reports với complex queries** | ❌ Không nên | ✅ Nên dùng | Code có query optimization |
| **Views customization** | ✅ Nên dùng | ⚖️ Cả hai OK | Studio faster, Code version-controlled |

---

### B3. MÔ HÌNH KẾT HỢP — CODE EXTEND STUDIO VÀ NGƯỢC LẠI

**Nguồn:** Service patterns và MetaJson entity references [Từ source code + Suy luận]

#### Pattern 1: Studio Model + Code Service

Studio tạo simple model, Code viết Service xử lý complex logic. Example: Tenant management system.

**Step 1 - Studio tạo model:**
- Model name: `CustomTenant`
- Fields: `name` (string), `startDate` (date), `endDate` (date), `company` (many-to-one to Company)
- View: form grid với filters

**Step 2 - Code viết Service:**
```java
@Singleton
public class TenantRenewalService {

  @Inject
  private MetaJsonRecordRepository metaJsonRecordRepo;

  @Transactional
  public void renewExpiredTenants() {
    // Query Studio model data
    List<MetaJsonRecord> records = metaJsonRecordRepo
        .all()
        .filter("self.jsonModel.name = 'CustomTenant' AND self.attrs.endDate < :today")
        .bind("today", LocalDate.now())
        .fetch();

    for (MetaJsonRecord record : records) {
      Map<String, Object> attrs = record.getAttrs();
      LocalDate oldEndDate = (LocalDate) attrs.get("endDate");
      LocalDate newEndDate = oldEndDate.plusYears(1);

      attrs.put("endDate", newEndDate);
      attrs.put("renewed", true);
      record.setAttrs(attrs);

      metaJsonRecordRepo.save(record);
    }
  }
}
```

**Giải thích:** Code Service query `meta_json_record` table filtering by model name. Data accessed qua `getAttrs()` map (key-value pairs). Logic xử lý renewals, updates attrs, saves back. Pattern này cho phép business logic phức tạp trên Studio models.

**Step 3 - BPM trigger Service:**
```groovy
// Trong BPM Script Task
def tenantService = __beans__.get("com.axelor.apps.custom.service.TenantRenewalService")
tenantService.renewExpiredTenants()
```

#### Pattern 2: Code Model + Studio BPM

Code định nghĩa entity phức tạp, Studio tạo workflow automation.

**Step 1 - Code domain XML:**
```xml
<entity name="SaleOrder">
  <many-to-one name="clientPartner" ref="Partner"/>
  <decimal name="totalAmount"/>
  <integer name="statusSelect" selection="sale.order.status.select"/>
  <!-- 50+ other fields... -->
</entity>
```

**Step 2 - Studio BPM process:**
- Process name: "Sale Order Approval"
- Start event: When status = DRAFT and totalAmount > 10000
- Decision table: Approve/Reject based on amount + customer credit rating
- Service tasks: Send email notifications, update ERP backend
- End event: Set status = CONFIRMED or REJECTED

Studio BPM có thể thao tác code-defined entities hoàn toàn - read fields, update values, trigger business logic. Integration seamless vì Camunda engine không phân biệt entity source.

---

### B4. QUẢN LÝ PHIÊN BẢN (VERSION CONTROL) KHI KẾT HỢP

**Nguồn:** Data init mechanisms và Studio app structures [Không tìm thấy export/import trong source code]

#### Vấn đề cốt lõi

Code nằm trong Git, Studio config nằm trong database. Khi team develop trên local machines hoặc deploy lên servers khác nhau, làm sao đồng bộ Studio configurations?

#### Cơ chế seed data (tìm thấy trong source code)

Axelor có CSV import mechanism cho master data. Pattern: đặt CSV files trong `/src/main/resources/data-init/` hoặc `/src/main/resources/data-demo/`, framework auto-import khi application first startup.

```
axelor-sale/
└── src/main/resources/
    ├── data-init/
    │   ├── studio_appSale.csv
    │   ├── meta_view.csv
    │   └── meta_action.csv
    └── domains/
```

CSV files có thể chứa Studio configurations - `meta_json_model` rows, `meta_json_field` rows, `meta_view` custom views. Developer export Studio configs sang CSV, commit vào Git, colleagues import khi pull code.

#### Những gì KHÔNG tìm thấy trong source code

❌ **Built-in export/import Studio configs:** Không tìm thấy Gradle tasks, Controller endpoints, hoặc Service methods để export entire Studio app (models + views + actions + BPM processes) sang portable format (XML/JSON/ZIP).

❌ **Studio app versioning:** Không có `version` field trong `MetaJsonModel` hoặc version control tables để track configuration changes over time.

❌ **Merge conflict resolution:** Khi hai developers modify cùng Studio model trên local machines, không có mechanism detect conflicts khi merge.

❌ **Database migration tools:** Không có Liquibase hoặc Flyway integration để manage Studio schema changes incrementally.

> ⚠️ **Suy luận:** Đây là pain point lớn trong Axelor development. Best practice có thể là: (1) Designate một "Studio admin" manage tất cả Studio changes, (2) Export Studio tables sang CSV sau mỗi change, commit CSV vào Git, (3) Use database dumps cho complex Studio apps thay vì CSV, (4) Prefer code-defined entities cho core models, Studio chỉ cho auxiliary/tenant-specific models.

---

### B5. CODE PATTERNS TỪ MODULES THỰC TẾ — AXELOR-SALE ANALYSIS

**Nguồn:** axelor-sale module structure analysis [Từ source code]

#### Package Structure Pattern

Axelor-sale module (880+ Java files) tổ chức theo functional decomposition:

```
com.axelor.apps.sale/
├── db/
│   └── repo/               # Custom Repositories (8 classes)
├── service/
│   ├── app/                # Application-level services (2 classes)
│   ├── cart/               # Shopping cart services (7 classes)
│   ├── cartline/           # Cart line services (8 classes)
│   ├── config/             # Configuration services (2 classes)
│   ├── configurator/       # Product configurator (10 classes)
│   ├── saleorder/          # Sale order core (40+ classes)
│   │   ├── status/         # Workflow status management
│   │   ├── print/          # Printing services
│   │   ├── views/          # View-related services
│   │   └── ...
│   ├── saleorderline/      # Sale order line services (20+ classes)
│   └── ...
├── web/                    # Controllers (15 classes)
├── exception/              # Custom exceptions (1 class)
└── module/                 # Guice module (1 class - SaleModule)
```

**Naming conventions quan sát được:**
- Services: `{Entity}{Operation}Service` (ví dụ: `SaleOrderComputeService`, `SaleOrderPrintService`)
- Controllers: `{Entity}Controller` (ví dụ: `SaleOrderController`)
- Repositories: `{Entity}ManagementRepository` hoặc `{Entity}SaleRepository`
- Packages: Verb-based (`creation/`, `print/`, `status/`) hoặc Domain-based (`cart/`, `configurator/`)

#### Service Granularity Pattern

Axelor prefer nhiều small focused services thay vì few large services. Example: SaleOrder entity có 40+ services, mỗi service handle một aspect:
- `SaleOrderComputeService` - Calculations (totals, taxes, discounts)
- `SaleOrderCreateService` - Factory methods để create new orders
- `SaleOrderCopyService` - Duplication logic
- `SaleOrderCheckService` - Validations
- `SaleOrderWorkflowService` - Status transitions
- `SaleOrderPrintService` - Report generation

**Lợi ích pattern này:**
- Single Responsibility Principle - mỗi service một concern rõ ràng
- Easier testing - test từng service độc lập
- Easier maintenance - thay đổi một aspect không affect others
- Parallel development - team members work on different services không conflict

**Trade-off:**
- Nhiều service classes → harder navigate codebase
- Dependency graphs phức tạp → services call services call services
- Risk của circular dependencies → phải careful design interfaces

---

### B6. QUẢN LÝ THỨ TỰ THỰC THI LOGIC — BEST PRACTICES KHI KẾT HỢP CODE VÀ STUDIO

**Nguồn:** Event system, Action chains, Repository hooks [Từ source code]

Khi hệ thống vừa có code truyền thống vừa có Studio customizations, việc quản lý thứ tự thực thi logic trở thành thách thức lớn. Logic có thể nằm ở nhiều tầng khác nhau: Repository hooks, Service methods, Controller actions, Action chains (XML), BPM processes, Event observers. Hiểu rõ execution order và transaction boundaries là critical để tránh bugs, data inconsistencies, và performance issues.

---

#### 1. NĂM TẦNG THỰC THI LOGIC (EXECUTION LAYERS)

Axelor có 5 tầng logic thực thi với thứ tự rõ ràng từ thấp đến cao:

**Layer 1: Repository Hooks** (Database tier)
- Vị trí: Custom Repository classes (`*ManagementRepository`)
- Timing: Được gọi TỰ ĐỘNG khi save/copy/remove entities
- Transaction: Nằm TRONG transaction đã bắt đầu từ caller
- Use case: Computed fields, sequence generation, validations cấp entity

**Layer 2: Service Methods** (Business logic tier)
- Vị trí: Service classes (`*ServiceImpl`)
- Timing: Được gọi EXPLICITLY từ Controllers hoặc Services khác
- Transaction: Tạo transaction boundary nếu có `@Transactional`
- Use case: Complex business rules, cross-entity logic, external integrations

**Layer 3: Controller Actions** (Presentation tier)
- Vị trí: Controller classes (`*Controller`)
- Timing: Được gọi từ UI events qua ActionRequest
- Transaction: Delegates sang Services (không tự tạo transaction)
- Use case: Input extraction, response building, UI state management

**Layer 4: Action Chains** (XML-defined orchestration)
- Vị trí: XML view files (`<action-group>`, `<action-method>`, `<action-record>`)
- Timing: Sequential execution theo order trong XML
- Transaction: Mỗi action-method tạo transaction riêng (nếu Service có @Transactional)
- Use case: UI workflows, field updates, multi-step processes

**Layer 5: BPM Processes** (Workflow orchestration)
- Vị trí: Camunda BPMN/DMN models
- Timing: Async execution trong Camunda engine threads
- Transaction: Mỗi Service Task là một transaction
- Use case: Long-running workflows, human tasks, decision automation

**Layer 0: Event System** (Cross-cutting)
- Vị trí: Event observers (`*Observer` classes)
- Timing: Fired từ bất kỳ layer nào, observers execute synchronously
- Transaction: Share transaction với event firer
- Use case: Loose coupling, plugin architecture, cross-module communication

---

#### 2. REPOSITORY HOOKS — THỨ TỰ THỰC THI CHI TIẾT

**Nguồn:** `SaleOrderManagementRepository.save()` analysis [Từ source code]

Repository hooks execute theo thứ tự cố định khi entity lifecycle events occur:

**Save operation execution order:**

```java
// 1. Application code calls
saleOrderRepo.save(saleOrder);

// 2. Custom Repository.save() được gọi
@Override
public SaleOrder save(SaleOrder saleOrder) {
  // 3. Pre-save business logic
  computeSeq(saleOrder);              // Generate sequence if needed
  computeFullName(saleOrder);         // Concatenate display name
  computePackTotal(saleOrder);        // Calculate pack totals
  manageComplementaryProducts();      // Add complementary products
  computeMargins(saleOrder);          // Calculate profit margins
  updateOrderingStatus(saleOrder);    // Update workflow status

  // 4. Actual JPA persist
  return super.save(saleOrder);       // Hibernate INSERT/UPDATE SQL
}

// 5. Post-persist (trong same transaction)
// - Cascade saves to related entities (OneToMany collections)
// - Audit fields updated (createdOn, updatedOn, updatedBy)
// - Database triggers (nếu có)

// 6. Transaction commit (nếu đây là outermost @Transactional)
// - Changes flushed to database
// - Constraints validated
```

**Giải thích thứ tự:**

- **Bước 3 (Pre-save logic)** thực thi TRƯỚC database save. Đây là nơi compute derived fields, generate IDs, validate business rules. Nếu throw exception ở đây, transaction rollback, database không bị modify.

- **Bước 4 (super.save)** là actual JPA persist operation. Hibernate chuyển entity state sang MANAGED, nhưng chưa execute SQL (deferred đến flush time). Entity có ID sau bước này nếu là new entity.

- **Bước 5 (Cascade)** xảy ra sau main entity save. OneToMany collections (ví dụ: SaleOrder.saleOrderLineList) được iterate, mỗi child entity gọi repository.save() của nó - tạo recursive save chain.

- **Bước 6 (Commit)** là khi SQL thực sự execute. Database constraints (unique, foreign key, not null) được validate ở đây. Nếu constraint violation, exception throw và rollback toàn bộ transaction.

**Best practice pattern:**

```java
@Override
public SaleOrder save(SaleOrder saleOrder) {
  try {
    // Validation phase - fail fast
    if (saleOrder.getClientPartner() == null) {
      throw new AxelorException("Client partner required");
    }

    // Computation phase - derive values
    computeFields(saleOrder);

    // Persistence phase - actual save
    SaleOrder saved = super.save(saleOrder);

    // Post-save phase - side effects
    fireEvent(new SaleOrderSavedEvent(saved));

    return saved;

  } catch (AxelorException e) {
    // Business errors - user-facing messages
    throw new PersistenceException(e.getMessage(), e);
  } catch (Exception e) {
    // Infrastructure errors - log and wrap
    TraceBackService.trace(e);
    throw new PersistenceException("Failed to save sale order", e);
  }
}
```

---

#### 3. ACTION CHAINS — THỨ TỰ ORCHESTRATION TRONG XML

**Nguồn:** `SaleOrder.xml` action definitions [Từ source code]

Action chains trong XML views cho phép orchestrate multiple actions sequentially. Framework execute actions theo thứ tự khai báo, stop nếu bất kỳ action nào fail.

**Example from SaleOrder form:**

```xml
<form name="sale-order-form" model="com.axelor.apps.sale.db.SaleOrder"
      onNew="action-sale-order-method-onnew,action-sale-order-method-create-order,action-sale-order-record-hide-discount"
      onLoad="action-group-sale-saleorder-onload">

  <button name="confirmBtn" title="Confirm Order"
          onClick="save,action-sale-order-validate,action-sale-order-method-confirm,action-sale-order-record-set-confirmed"/>
</form>

<action-group name="action-group-sale-saleorder-onload">
  <action name="action-sale-order-attrs-readonly-fields"/>
  <action name="action-sale-order-attrs-scale-and-precision"/>
  <action name="action-sale-order-method-compute-end-of-validity-date"/>
  <action name="action-sale-order-record-delivery-state"/>
</action-group>
```

**Execution order breakdown:**

**onNew event** (khi user click "New" button):
1. `action-sale-order-method-onnew` - Java method set default values
2. `action-sale-order-method-create-order` - Java method initialize order structure
3. `action-sale-order-record-hide-discount` - XML action set field visibility

**onClick="save,..." event** (khi user click Confirm button):
1. `save` - Built-in action save form → triggers Repository.save()
2. `action-sale-order-validate` - Validation action check business rules
3. `action-sale-order-method-confirm` - Java method change status, send notifications
4. `action-sale-order-record-set-confirmed` - XML action update UI state

**onLoad event** (khi form được open):
1. `action-group-sale-saleorder-onload` - Action group chứa sub-actions:
   - `action-sale-order-attrs-readonly-fields` - Compute field readonly states
   - `action-sale-order-attrs-scale-and-precision` - Set numeric field precision
   - `action-sale-order-method-compute-end-of-validity-date` - Calculate date field
   - `action-sale-order-record-delivery-state` - Set delivery status badge

**Transaction boundaries trong action chains:**

⚠️ **QUAN TRỌNG:** Mỗi `action-method` tạo SEPARATE transaction nếu Java method có `@Transactional`. Nghĩa là:

```xml
<button onClick="action-method-1,action-method-2,action-method-3"/>
```

Tạo **3 transactions độc lập**. Nếu action-method-2 fail, action-method-1 đã COMMIT (không rollback). Đây là source of data inconsistencies phổ biến.

**Best practice:** Nếu cần atomic behavior (all-or-nothing), wrap logic trong một Service method duy nhất:

```xml
<!-- ❌ BAD: Multiple transactions -->
<button onClick="action-update-stock,action-create-invoice,action-send-email"/>

<!-- ✅ GOOD: Single transaction -->
<button onClick="action-process-order"/>
```

```java
// Service method wraps all logic
@Transactional
public void processOrder(SaleOrder order) {
  updateStock(order);      // Step 1
  createInvoice(order);    // Step 2 - rollback if fail
  sendEmail(order);        // Step 3 - rollback if fail
}
```

---

#### 4. EVENT SYSTEM — OBSERVER PATTERN TRONG AXELOR

**Nguồn:** `SaleOrderLineFireServiceImpl.java` [Từ source code]

Axelor implements event-driven architecture với Guice Event system. Pattern này cho phép loose coupling giữa modules - publishers không biết subscribers, subscribers có thể add/remove dynamically.

**Event lifecycle:**

```java
// File: SaleOrderLineFireServiceImpl.java
public class SaleOrderLineFireServiceImpl {
  protected Event<SaleOrderLineViewOnNew> saleOrderLineViewOnNewEvent;

  @Inject
  public SaleOrderLineFireServiceImpl(
      Event<SaleOrderLineViewOnNew> saleOrderLineViewOnNewEvent) {
    this.saleOrderLineViewOnNewEvent = saleOrderLineViewOnNewEvent;
  }

  public Map<String, Object> getOnNewAttrs(SaleOrderLine line, SaleOrder order) {
    // 1. Create event object
    SaleOrderLineViewOnNew event = new SaleOrderLineViewOnNew(line, order);

    // 2. Fire event - ALL observers execute SYNCHRONOUSLY
    saleOrderLineViewOnNewEvent.fire(event);

    // 3. Collect results từ observers
    return event.getSaleOrderLineMap();
  }
}
```

**Observer implementation:**

```java
// File: SaleOrderLineObserver.java
@Singleton
public class SaleOrderLineObserver {

  public void onSaleOrderLineViewOnNew(
      @Observes SaleOrderLineViewOnNew event) {

    // Observer logic execute trong SAME transaction với event firer
    SaleOrderLine line = event.getSaleOrderLine();
    SaleOrder order = event.getSaleOrder();

    // Modify event object - results propagate back
    if (order.getInAti()) {
      event.addAttr("price", "hidden", true);
      event.addAttr("priceWithTax", "hidden", false);
    }
  }
}
```

**Execution order của observers:**

⚠️ **KHÔNG CÓ GUARANTEED ORDER** giữa multiple observers. Nếu có 5 classes observe cùng event, execution order là UNDEFINED (depends on Guice injection order, class loading order).

**Best practices:**

1. **Observers phải INDEPENDENT:** Không assume observer A chạy trước observer B.
2. **Không modify shared state:** Observers chỉ nên modify event object, không modify entities directly (risk of conflicts).
3. **Lightweight logic only:** Complex logic nên ở Services, observers chỉ là glue code.
4. **Document dependencies:** Nếu observer B depend on observer A results, phải document rõ hoặc refactor thành single observer.

---

#### 5. TRANSACTION BOUNDARIES — KHI NÀO COMMIT, KHI NÀO ROLLBACK

**Nguồn:** `@Transactional` usage patterns [Từ source code]

Transaction management trong Axelor có 3 patterns chính:

**Pattern 1: Service method với @Transactional (recommended)**

```java
@Transactional
public void confirmSaleOrder(SaleOrder saleOrder) {
  // Transaction START

  saleOrder.setStatusSelect(STATUS_CONFIRMED);
  saleOrder.setConfirmDate(LocalDate.now());

  saleOrderRepo.save(saleOrder);          // Update entity

  stockMoveService.generateStockMoves(saleOrder);  // Create related records

  emailService.sendConfirmationEmail(saleOrder);   // External call

  // Transaction COMMIT - nếu không có exception
  // Transaction ROLLBACK - nếu có exception
}
```

**Pattern 2: Nested @Transactional (transaction propagation)**

```java
@Transactional
public void processOrders(List<SaleOrder> orders) {
  // Outer transaction START

  for (SaleOrder order : orders) {
    confirmSaleOrder(order);  // Inner @Transactional - KHÔNG tạo nested transaction
  }

  // Outer transaction COMMIT
}

@Transactional
public void confirmSaleOrder(SaleOrder order) {
  // Logic here REUSES outer transaction
  // Không tạo new transaction vì already trong transaction context
}
```

⚠️ **QUAN TRỌNG:** Guice Persist **KHÔNG HỖ TRỢ NESTED TRANSACTIONS**. Nếu method A (@Transactional) gọi method B (@Transactional), chỉ transaction của A được tạo. Method B reuse transaction của A.

**Implication:** Nếu method B throw exception, transaction của A cũng rollback. Không thể "commit một phần" của A và rollback B.

**Pattern 3: Custom rollback conditions**

```java
// Default: Rollback trên Runtime exceptions, KHÔNG rollback trên checked exceptions
@Transactional
public void method1() { ... }

// Explicit: Rollback trên ALL exceptions
@Transactional(rollbackOn = Exception.class)
public void method2() { ... }

// Partial rollback (KHÔNG SUPPORT trong Guice Persist)
// Phải manually manage với EntityManager
```

**Best practices:**

1. **Granular @Transactional:** Đặt ở Service methods, KHÔNG đặt ở Controllers.
2. **Minimize transaction scope:** Transaction càng ngắn càng tốt - giảm database locks.
3. **Fail fast:** Validation ở ĐẦU method, trước khi modify data.
4. **Document propagation:** Comment rõ methods nào expect outer transaction, methods nào tạo new transaction.

---

#### 6. BEST PRACTICES MATRIX — PHÂN BỐ LOGIC THEO LAYER

Khi implement một feature, phải quyết định logic nằm ở layer nào. Matrix này guide decisions:

| Logic Type | Repository Hook | Service | Controller | Action Chain | BPM Process | Event Observer |
|------------|----------------|---------|------------|--------------|-------------|----------------|
| **Generate sequence number** | ✅ Preferred | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Compute derived fields** | ✅ Preferred | ⚖️ OK | ❌ | ❌ | ❌ | ❌ |
| **Validate business rules** | ⚖️ OK | ✅ Preferred | ❌ | ❌ | ❌ | ❌ |
| **Cross-entity orchestration** | ❌ | ✅ Preferred | ❌ | ❌ | ⚖️ OK | ❌ |
| **External API calls** | ❌ | ✅ Preferred | ❌ | ❌ | ⚖️ OK | ❌ |
| **UI field visibility** | ❌ | ❌ | ⚖️ OK | ✅ Preferred | ❌ | ✅ Preferred |
| **Multi-step workflows** | ❌ | ⚖️ OK | ❌ | ⚖️ OK | ✅ Preferred | ❌ |
| **Async long-running tasks** | ❌ | ❌ | ❌ | ❌ | ✅ Preferred | ❌ |
| **Cross-module communication** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ Preferred |
| **Audit logging** | ⚖️ OK | ❌ | ❌ | ❌ | ❌ | ✅ Preferred |

**Legend:**
- ✅ Preferred: Best practice, follow Axelor patterns
- ⚖️ OK: Acceptable nhưng có trade-offs
- ❌ Anti-pattern: Avoid, sẽ gây problems

---

#### 7. DEBUGGING STRATEGIES — TRACKING EXECUTION ORDER

Khi logic không execute theo expected order, debugging techniques:

**Strategy 1: Logging với execution markers**

```java
@Override
public SaleOrder save(SaleOrder saleOrder) {
  logger.debug(">>> REPO HOOK START: save() - Order ID: {}", saleOrder.getId());

  computeSeq(saleOrder);
  logger.debug("    REPO HOOK: After computeSeq - Seq: {}", saleOrder.getSaleOrderSeq());

  computeMargins(saleOrder);
  logger.debug("    REPO HOOK: After computeMargins - Margin: {}", saleOrder.getTotalMargin());

  SaleOrder saved = super.save(saleOrder);
  logger.debug("<<< REPO HOOK END: save() - Order ID: {}", saved.getId());

  return saved;
}
```

**Strategy 2: Breakpoint chaining trong IDE**

Set breakpoints với conditions:
1. Repository.save() entry point
2. Each computation method
3. super.save() call
4. Transaction commit point

**Strategy 3: Transaction logging**

Enable Hibernate transaction logging trong `axelor-config.properties`:

```properties
# Show SQL statements
logging.level.org.hibernate.SQL = DEBUG

# Show transaction begin/commit/rollback
logging.level.org.hibernate.engine.transaction = TRACE

# Show bind parameters
logging.level.org.hibernate.type.descriptor.sql.BasicBinder = TRACE
```

**Strategy 4: Action chain tracing**

Trong XML actions, add logging:

```xml
<action-method name="action-debug-log-step1"
               model="com.axelor.apps.sale.db.SaleOrder"
               call="com.axelor.apps.base.web.LogController:log('STEP 1 executed')"/>

<button onClick="action-debug-log-step1,action-real-logic,action-debug-log-step2"/>
```

---

#### 8. REAL-WORLD SCENARIO — ORDER CONFIRMATION WORKFLOW

**Scenario:** Khi user click "Confirm" button trong Sale Order form, có logic ở:
- Repository hook (compute sequence, margins)
- Service method (validate stock, create invoice)
- Action chain (update UI, show notification)
- BPM process (approval workflow nếu amount > $10K)
- Event observer (send email notification)

**Execution flow breakdown:**

```
1. USER CLICKS "Confirm" button
   └─> onClick="save,action-validate,action-confirm"

2. ACTION: save (Built-in)
   ├─> Controller: implicit save action
   ├─> Repository: SaleOrderManagementRepository.save()
   │   ├─> computeSeq() - Generate SO number
   │   ├─> computeMargins() - Calculate profits
   │   └─> super.save() - JPA persist
   │       └─> Transaction COMMIT #1
   └─> UI: Form reloaded với fresh data

3. ACTION: action-validate (action-record)
   ├─> Framework: Set field values
   ├─> No Java code execute
   └─> No transaction

4. ACTION: action-confirm (action-method)
   ├─> Controller: SaleOrderController.confirmOrder()
   ├─> Service: SaleOrderWorkflowService.confirmSaleOrder()
   │   ├─> @Transactional START
   │   ├─> Validate stock availability
   │   ├─> Set status = CONFIRMED
   │   ├─> Repository.save() - Update entity
   │   ├─> Create invoice (InvoiceService)
   │   ├─> Generate stock moves (StockMoveService)
   │   ├─> Fire event: SaleOrderConfirmedEvent
   │   │   └─> Observer: EmailObserver.sendConfirmationEmail()
   │   └─> @Transactional COMMIT #2
   ├─> Response: setFlash("Order confirmed")
   └─> UI: Show success notification

5. BPM TRIGGER (if configured)
   ├─> Event listener detects status = CONFIRMED
   ├─> Start BPMN process "Order Approval"
   ├─> Async execution trong Camunda thread
   │   ├─> Service Task 1: Check credit limit
   │   ├─> Decision Table: Approve/Reject
   │   ├─> Service Task 2: Notify manager
   │   └─> Each task = separate transaction
   └─> Process continues independently của UI

TOTAL TRANSACTIONS: 2 (save hook) + 1 (confirm logic) + N (BPM tasks) = 3+ transactions
```

**Timeline visualization:**

```
Time →

User      [Click]
          ↓
UI        [save action]─────[validate action]─[confirm action]──────[show notification]
          ↓                                    ↓                      ↓
Repo      [save hook]────────                 │                      │
          ↓    ↓                              │                      │
          Seq  Margin                         │                      │
          ↓                                    ↓                      │
DB        [TX1: COMMIT]                       [TX2: COMMIT]          │
                                              ↓         ↓            │
                                              Validate  Invoice      │
                                                        StockMove    │
                                              ↓                      │
Event                                         [Fire event]           │
                                              ↓                      │
Observer                                      [Send email]           │
                                                                     │
BPM       [Process start]──────────────────────────────────────────┘
          ↓                ↓                ↓
          [Check credit]  [Approve]  [Notify]
          [TX3]           [TX4]      [TX5]
```

**Key takeaways:**

1. **2 transactions riêng biệt** cho save hook và confirm logic - không atomic cross actions.
2. **Repository hook chạy TRƯỚC** Service logic - sequence generated before confirm.
3. **Event observer trong cùng transaction** với Service - email send failure rollback entire confirm.
4. **BPM process async** - không block UI, failures không affect main transaction.

---

#### 9. PITFALLS VÀ ANTI-PATTERNS

**Anti-pattern 1: Complex logic trong Repository hooks**

```java
// ❌ BAD: Repository hook gọi Services
@Override
public SaleOrder save(SaleOrder saleOrder) {
  super.save(saleOrder);

  // Calling Services from Repository = BAD IDEA
  invoiceService.createInvoice(saleOrder);  // What if this fails?
  stockMoveService.generateMoves(saleOrder); // Circular dependency risk
  emailService.sendNotification(saleOrder);  // Slow I/O in save hook

  return saleOrder;
}
```

**Tại sao BAD:**
- Repository hooks nên lightweight (< 50ms). External calls có thể timeout.
- Circular dependencies: Repository → Service → Repository calls.
- Transaction scope unclear: Services có thể có own @Transactional, nested behavior undefined.

**✅ GOOD pattern:**

```java
// Repository hook chỉ compute fields
@Override
public SaleOrder save(SaleOrder saleOrder) {
  computeSeq(saleOrder);
  computeMargins(saleOrder);
  return super.save(saleOrder);
}

// Service orchestrates complex logic
@Transactional
public void processOrder(SaleOrder saleOrder) {
  saleOrderRepo.save(saleOrder);              // Trigger repo hook
  invoiceService.createInvoice(saleOrder);    // Then create invoice
  stockMoveService.generateMoves(saleOrder);  // Then generate moves
  emailService.sendNotification(saleOrder);   // Finally notify
}
```

**Anti-pattern 2: Assuming action chain atomicity**

```xml
<!-- ❌ BAD: Expecting all-or-nothing behavior -->
<button onClick="action-deduct-stock,action-create-invoice,action-send-email"/>
```

Nếu action-create-invoice fail, action-deduct-stock đã COMMITTED. Kết quả: stock deducted nhưng no invoice - data inconsistency.

**✅ GOOD pattern:**

```xml
<!-- Wrap trong single Service call -->
<button onClick="action-process-order"/>
```

**Anti-pattern 3: Order-dependent observers**

```java
// ❌ BAD: Observer B assumes Observer A ran first
@Observes
public void observerA(OrderEvent event) {
  event.set("computedPrice", calculatePrice());
}

@Observes
public void observerB(OrderEvent event) {
  // Assumes observerA already set computedPrice
  BigDecimal price = event.get("computedPrice");  // NPE risk!
  applyDiscount(price);
}
```

**✅ GOOD pattern:**

```java
// Single observer handles dependent logic
@Observes
public void observerPricing(OrderEvent event) {
  BigDecimal price = calculatePrice();
  BigDecimal discounted = applyDiscount(price);
  event.set("finalPrice", discounted);
}
```

---

#### 10. DOCUMENTATION PRACTICES — MAKING EXECUTION ORDER VISIBLE

**Practice 1: Method naming conventions**

```java
// Prefix với execution phase
public class SaleOrderService {

  // Pre-save computations
  public void preComputeMargins(SaleOrder order) { }

  // Main business logic
  public void confirmOrder(SaleOrder order) { }

  // Post-save side effects
  public void postNotifyStakeholders(SaleOrder order) { }
}
```

**Practice 2: Inline documentation**

```java
/**
 * Confirms sale order và creates related documents.
 *
 * EXECUTION ORDER:
 * 1. Validate stock availability (fail-fast)
 * 2. Set status = CONFIRMED
 * 3. Save entity (triggers Repository.save() hook)
 *    3a. Repository computes sequence
 *    3b. Repository computes margins
 * 4. Create invoice (InvoiceService.createInvoice())
 * 5. Generate stock moves (StockMoveService.generateMoves())
 * 6. Fire SaleOrderConfirmedEvent
 *    6a. EmailObserver sends notification
 *    6b. AnalyticsObserver logs event
 *
 * TRANSACTION: Single @Transactional - all steps atomic
 * ROLLBACK: Any step failure rolls back entire operation
 */
@Transactional
public void confirmSaleOrder(SaleOrder saleOrder) throws AxelorException {
  // Implementation...
}
```

**Practice 3: Sequence diagrams trong README**

```
README.md trong module:

## Order Confirmation Flow

```
User → UI → Controller → Service → Repository → Database
                ↓           ↓          ↓
              Validate   Confirm   Compute
                           ↓
                      Fire Event
                           ↓
                    ┌──────┴───────┐
                    ↓              ↓
               EmailObserver  AnalyticsObserver
```

---

### Tổng kết best practices

**10 nguyên tắc vàng quản lý execution order:**

1. **Repository hooks cho computed fields only** - Không gọi Services, không external calls.
2. **Services là orchestration layer** - Coordinate multiple repositories, handle transactions.
3. **Controllers là thin adapters** - Extract input, delegate Services, build response.
4. **Action chains không atomic** - Mỗi action-method = separate transaction.
5. **Observers không assume order** - Phải independent, không depend lẫn nhau.
6. **@Transactional granular** - Service methods, không Controllers, không Repositories.
7. **Fail fast trong transactions** - Validation đầu method, minimize rollback cost.
8. **Document execution flow** - Inline comments, method naming, sequence diagrams.
9. **Minimize transaction scope** - Càng ngắn càng tốt, reduce lock contention.
10. **Test execution order explicitly** - Integration tests verify transaction boundaries.

**Khi kết hợp Code và Studio:**

- **Studio actions execute AFTER code logic** - Action chains sau khi Repository save.
- **BPM processes async** - Không block main transaction, handle separately.
- **Event system bridges Code-Studio** - Observers có thể trigger Studio actions, BPM processes.
- **Transaction boundaries rõ ràng** - Document which layer owns transaction.

Hiểu rõ execution order và transaction boundaries là chìa khóa để build reliable, maintainable Axelor applications. Khi debug issues, luôn bắt đầu bằng câu hỏi: "Logic này nằm ở layer nào? Transaction boundary ở đâu? Thứ tự execute ra sao?"

---
## B7. QUY TRÌNH EXTENSION VÀ CUSTOM MODULE DEVELOPMENT

**Nguồn:** axelor-bank-payment module structure, BankPaymentModule.java, MoveLine.xml, BankDetails.xml [Từ source code]

Một trong những strengths lớn nhất của Axelor là khả năng extend existing modules mà không cần modify source code trực tiếp. Hiểu rõ extension patterns là critical để build maintainable customizations và upgrade framework versions dễ dàng.

---

### 1. CUSTOM MODULE STRUCTURE — CHUẨN BỊ MODULE MỚI

#### a) Module Layout

Custom module phải follow cấu trúc chuẩn:

```
modules/
└── my-custom-module/                      # Module root
    ├── build.gradle                       # Gradle build config
    ├── src/
    │   └── main/
    │       ├── java/                      # Java source code
    │       │   └── com/mycompany/apps/
    │       │       └── mymodule/
    │       │           ├── module/        # Guice module
    │       │           │   └── MyCustomModule.java
    │       │           ├── db/            # Entities (generated)
    │       │           │   └── repo/      # Custom repositories
    │       │           ├── service/       # Services
    │       │           │   ├── MyService.java
    │       │           │   └── MyServiceImpl.java
    │       │           └── web/           # Controllers
    │       │               └── MyController.java
    │       └── resources/
    │           ├── domains/               # Domain XML definitions
    │           │   └── MyEntity.xml
    │           └── views/                 # View XML definitions
    │               ├── MyEntity.xml
    │               └── Menu.xml
    └── README.md
```

**Nguồn:** axelor-bank-payment directory structure [Lines follow standard Axelor layout]

---

#### b) Module Registration

**`settings.gradle` (root project):**

Axelor auto-discovers modules trong `/modules` directory:

```groovy
// settings.gradle (root project)
def modules = []
file("modules").traverse(type: groovy.io.FileType.DIRECTORIES, maxDepth: 1) { it ->
  if (new File(it, "build.gradle").exists()) {
    modules.add(it)
  }
}

modules.each { dir ->
  include "modules:$dir.name"
  project(":modules:$dir.name").projectDir = dir
}
```

**Nguồn:** `/Volumes/works/code/java/axelor/axelor-erp/settings.gradle:52-64`

✅ **Best practice:** Chỉ cần tạo module directory trong `/modules/` với `build.gradle` file — Gradle sẽ tự động discover và include.

---

#### c) Module Dependencies

**`build.gradle` (custom module):**

Declare dependencies vào modules khác:

```gradle
// modules/my-custom-module/build.gradle
plugins {
  id 'com.axelor.app'
}

axelor {
  title "My Custom Module"
  description "Custom module extending Sale"
}

dependencies {
  // Depend on existing modules
  api project(":modules:axelor-sale")
  api project(":modules:axelor-account")

  // External libraries if needed
  implementation libs.commons_lang3
}
```

**Nguồn:** `/modules/axelor-open-suite/axelor-bank-payment/build.gradle:22` — `api project(":modules:axelor-account")`

**Dependency types:**
- `api` — Transitive dependency, exposed to dependent modules (prefer này)
- `implementation` — Internal dependency, không expose

---

### 2. EXTENDING ENTITIES — THÊM FIELDS VÀO MODEL CÓ SẴN

#### a) Domain XML Extension Pattern

**Scenario:** Muốn thêm field `customNote` vào `SaleOrder` entity từ axelor-sale module.

**`modules/my-custom-module/src/main/resources/domains/SaleOrder.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<domain-models xmlns="http://axelor.com/xml/ns/domain-models"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://axelor.com/xml/ns/domain-models
                      http://axelor.com/xml/ns/domain-models/domain-models_7.4.xsd">

  <!-- Specify the module and package of the ORIGINAL entity -->
  <module name="sale" package="com.axelor.apps.sale.db"/>

  <entity name="SaleOrder">
    <!-- Add new fields to existing entity -->
    <string name="customNote" title="Custom Note" large="true"/>
    <decimal name="customDiscount" title="Custom Discount %" precision="5" scale="2"/>
    <many-to-one name="customApprover" ref="com.axelor.auth.db.User" title="Custom Approver"/>
  </entity>

</domain-models>
```

**Nguồn:** `/build/tmp/.cache/expanded/.../domains/MoveLine.xml:8-13` — Bank payment module extends MoveLine entity từ account module

**Key rules:**
- ✅ **PHẢI khai báo `<module name="sale" package="...">` matching original entity** — Đây là cách Axelor biết bạn đang extend entity nào
- ✅ **Entity name PHẢI match exactly** — Tên entity phải giống hệt original
- ✅ **Không được duplicate existing fields** — Chỉ khai báo fields mới
- ❌ **Không thể remove hoặc modify existing fields** — Chỉ add được

**Sau khi thêm field:**

```bash
./gradlew generateCode
```

Generated Java class sẽ có getter/setter cho `customNote`, `customDiscount`, `customApprover`.

---

#### b) Entity Inheritance (Tạo Entity Con)

**Scenario:** Muốn tạo entity mới kế thừa từ base entity.

**IMPORTANT:** Axelor **KHÔNG hỗ trợ entity inheritance** trong domain XML. Không có `<entity extends="BaseEntity">` pattern.

**Workaround:** Use **composition** thay vì inheritance:

```xml
<entity name="MyCustomOrder">
  <!-- Embed base entity via many-to-one -->
  <many-to-one name="saleOrder" ref="com.axelor.apps.sale.db.SaleOrder" required="true"/>

  <!-- Add custom fields -->
  <string name="customField1"/>
  <decimal name="customField2"/>
</entity>
```

❌ **Anti-pattern:** Copy-paste entire entity definition để modify — Breaks upgrades.

---

### 3. EXTENDING SERVICES — THÊM LOGIC VÀO METHODS CÓ SẴN

#### a) Service Override Pattern

**Scenario:** Override `InvoicePaymentValidateService` để add custom validation logic.

**Step 1: Create extended implementation**

**`MyInvoicePaymentValidateServiceImpl.java`:**

```java
package com.mycompany.apps.mymodule.service;

import com.axelor.apps.account.service.payment.invoice.payment.InvoicePaymentValidateServiceImpl;
import com.axelor.apps.account.db.InvoicePayment;
import com.axelor.apps.base.AxelorException;
import com.google.inject.Inject;
import com.google.inject.servlet.RequestScoped;

@RequestScoped
public class MyInvoicePaymentValidateServiceImpl extends InvoicePaymentValidateServiceImpl {

  protected MyCustomService myCustomService;  // New dependency

  @Inject
  public MyInvoicePaymentValidateServiceImpl(
      InvoicePaymentRepository invoicePaymentRepository,
      InvoicePaymentToolService invoicePaymentToolService,
      InvoicePaymentMoveCreateService invoicePaymentMoveCreateService,
      MyCustomService myCustomService) {  // Inject new service
    // Call parent constructor
    super(invoicePaymentRepository, invoicePaymentToolService, invoicePaymentMoveCreateService);
    this.myCustomService = myCustomService;
  }

  @Override
  protected void setInvoicePaymentStatus(InvoicePayment invoicePayment) throws AxelorException {
    // Add custom logic BEFORE parent logic
    myCustomService.validateCustomRules(invoicePayment);

    // Call parent method to preserve original behavior
    super.setInvoicePaymentStatus(invoicePayment);

    // OR completely replace logic:
    // invoicePayment.setStatusSelect(MyCustomStatus.CUSTOM_STATUS);
  }

  // Add entirely NEW methods
  public void myNewMethod(InvoicePayment invoicePayment) {
    // Custom logic
  }
}
```

**Nguồn:** `/modules/axelor-open-suite/axelor-bank-payment/.../InvoicePaymentValidateServiceBankPayImpl.java:39-78` — Extends parent service, calls super(), overrides method

---

**Step 2: Register override trong Guice Module**

**`MyCustomModule.java`:**

```java
package com.mycompany.apps.mymodule.module;

import com.axelor.app.AxelorModule;
import com.axelor.apps.account.service.payment.invoice.payment.InvoicePaymentValidateServiceImpl;
import com.mycompany.apps.mymodule.service.MyInvoicePaymentValidateServiceImpl;

public class MyCustomModule extends AxelorModule {

  @Override
  protected void configure() {
    // Override parent service implementation
    bind(InvoicePaymentValidateServiceImpl.class)
        .to(MyInvoicePaymentValidateServiceImpl.class);

    // Register new services
    bind(MyCustomService.class).to(MyCustomServiceImpl.class);
  }
}
```

**Nguồn:** `/modules/axelor-open-suite/axelor-bank-payment/.../BankPaymentModule.java:218-221` — Service override binding

**Key pattern:**
- ✅ **Bind parent implementation class (not interface)** — `bind(ParentServiceImpl.class).to(MyServiceImpl.class)`
- ✅ **Custom implementation extends parent** — Inherit existing behavior
- ✅ **Call super() trong constructor** — Preserve parent dependencies
- ✅ **Use @Override annotation** — Clarity và compile-time checking

---

#### b) Wrapper Pattern (Khi Không Có Interface)

**Scenario:** Service gốc không có interface, khó extend.

**Pattern:** Tạo wrapper service delegate calls:

```java
@Singleton
public class SaleOrderServiceWrapper {

  @Inject
  protected SaleOrderService originalService;

  @Inject
  protected MyCustomService myCustomService;

  @Transactional
  public void computeSaleOrder(SaleOrder saleOrder) throws AxelorException {
    // Add logic BEFORE
    myCustomService.preComputeHook(saleOrder);

    // Call original
    originalService.computeSaleOrder(saleOrder);

    // Add logic AFTER
    myCustomService.postComputeHook(saleOrder);
  }
}
```

**Bind wrapper trong Module:**

```java
bind(SaleOrderService.class).to(SaleOrderServiceWrapper.class);
```

❌ **Limitation:** Wrapper KHÔNG inherit parent methods — Phải manually delegate mọi method muốn expose.

---

#### c) Repository Override Pattern

**Scenario:** Override `save()` hook trong Repository.

**`MyCustomSaleOrderRepository.java`:**

```java
package com.mycompany.apps.mymodule.db.repo;

import com.axelor.apps.sale.db.SaleOrder;
import com.axelor.apps.sale.db.repo.SaleOrderRepository;
import com.google.inject.Inject;

public class MyCustomSaleOrderRepository extends SaleOrderRepository {

  @Inject
  protected MyCustomService myCustomService;

  @Override
  public SaleOrder save(SaleOrder saleOrder) {
    // Add logic BEFORE save
    if (saleOrder.getCustomNote() != null) {
      myCustomService.processCustomNote(saleOrder);
    }

    // Call parent save (includes ManagementRepository logic if any)
    saleOrder = super.save(saleOrder);

    // Add logic AFTER save
    myCustomService.afterSaveHook(saleOrder);

    return saleOrder;
  }
}
```

**Register trong Module:**

```java
bind(SaleOrderRepository.class).to(MyCustomSaleOrderRepository.class);
```

**Nguồn:** SaleOrderManagementRepository pattern [Analyzed in Section A3]

⚠️ **Warning:** Nếu axelor-sale đã có `SaleOrderManagementRepository`, custom repository phải extend **ManagementRepository** (not base generated Repository):

```java
public class MyCustomSaleOrderRepository extends SaleOrderManagementRepository {
  // ...
}
```

---

### 4. EXTENDING VIEWS — THÊM FIELDS VÀ BUTTONS VÀO UI

#### a) Form Extension với Xpath

**Scenario:** Thêm field `customNote` vào form `sale-order-form`.

**`modules/my-custom-module/src/main/resources/views/SaleOrder.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<object-views xmlns="http://axelor.com/xml/ns/object-views"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://axelor.com/xml/ns/object-views
                      http://axelor.com/xml/ns/object-views/object-views_7.4.xsd">

  <!-- Extend existing form -->
  <form name="sale-order-form" title="Sale Order"
        model="com.axelor.apps.sale.db.SaleOrder"
        extension="true">

    <!-- Target field 'customer' và insert sau nó -->
    <extend target="//field[@name='customer']">
      <insert position="after">
        <field name="customApprover" domain="self.group.code = 'admins'"/>
      </insert>
    </extend>

    <!-- Target panel 'mainPanel' và insert field vào cuối -->
    <extend target="//panel[@name='mainPanel']">
      <insert position="inside">
        <field name="customNote" colSpan="12" widget="html"/>
        <field name="customDiscount" readonly="true"/>
      </insert>
    </extend>

    <!-- Add button to toolbar -->
    <extend target="//toolbar">
      <insert position="inside">
        <button name="customApproveBtn" title="Custom Approve"
                onClick="action-sale-order-method-custom-approve"
                showIf="statusSelect == 1"/>
      </insert>
    </extend>

  </form>

  <!-- Define button action -->
  <action-method name="action-sale-order-method-custom-approve">
    <call class="com.mycompany.apps.mymodule.web.SaleOrderController"
          method="customApprove"/>
  </action-method>

</object-views>
```

**Nguồn:** `/modules/axelor-open-suite/axelor-bank-payment/.../BankDetails.xml:39-56` — Form extension với xpath và insert positions

---

#### b) Xpath Selectors Reference

**Common xpath patterns:**

| Xpath | Target |
|-------|--------|
| `//field[@name='customer']` | Field có attribute name="customer" |
| `//panel[@name='mainPanel']` | Panel có name="mainPanel" |
| `//toolbar` | Toolbar element |
| `//button[@name='confirmBtn']` | Button có name="confirmBtn" |
| `//panel[@name='detailsPanel']/field[@name='amount']` | Field 'amount' bên trong panel 'detailsPanel' |

**Insert positions:**

| Position | Effect |
|----------|--------|
| `after` | Insert SAU target element |
| `before` | Insert TRƯỚC target element |
| `inside` | Insert VÀO TRONG target container (panel, toolbar, etc.) |
| `replace` | THAY THẾ target element hoàn toàn |

---

#### c) Grid Extension

**Scenario:** Thêm columns vào grid view.

```xml
<grid name="sale-order-grid" title="Sale Orders"
      model="com.axelor.apps.sale.db.SaleOrder"
      extension="true">

  <!-- Insert column after 'customer' column -->
  <extend target="//field[@name='customer']">
    <insert position="after">
      <field name="customApprover"/>
      <field name="customDiscount"/>
    </insert>
  </extend>

</grid>
```

---

#### d) View Override (Complete Replacement)

**Scenario:** Hoàn toàn replace view thay vì extend.

**Method 1: Re-define same view name**

```xml
<!-- This will OVERRIDE the original sale-order-form -->
<form name="sale-order-form" title="Sale Order (Custom)"
      model="com.axelor.apps.sale.db.SaleOrder">
  <!-- Completely new form layout -->
  <panel name="myCustomPanel">
    <field name="customer"/>
    <field name="customNote"/>
  </panel>
</form>
```

⚠️ **Risk:** Khi upgrade Axelor, original form có thể thêm critical fields mà custom form không có.

**Method 2: Create new view với priority**

Axelor load views theo module order — modules loaded sau override views loaded trước. Control order trong `settings.gradle` hoặc dependencies.

❌ **Anti-pattern:** Override entire view — Prefer extension với xpath.

---

### 5. EXTENDING MENUS — THÊM MENU ITEMS

#### a) Add Submenu to Existing Menu

**`Menu.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<object-views xmlns="http://axelor.com/xml/ns/object-views"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://axelor.com/xml/ns/object-views
                      http://axelor.com/xml/ns/object-views/object-views_7.4.xsd">

  <!-- Add submenu under existing 'sales-root' menu -->
  <menuitem name="sales-root-custom-reports"
            parent="sales-root"
            title="Custom Reports"
            action="action-custom-reports"
            order="900"/>

  <action-view name="action-custom-reports" title="Custom Reports"
               model="com.mycompany.apps.mymodule.db.CustomReport">
    <view type="grid" name="custom-report-grid"/>
    <view type="form" name="custom-report-form"/>
  </action-view>

</object-views>
```

**Nguồn:** `/modules/axelor-open-suite/axelor-account/.../Menu.xml:6-10` — Menuitem hierarchy với parent attribute

**Menu attributes:**
- `name` — Unique identifier (kebab-case convention)
- `parent` — Parent menu name (để tạo hierarchy)
- `title` — Display text (có thể translate)
- `action` — Action name to execute khi click
- `order` — Sort order (số càng nhỏ càng lên trên, negative numbers OK)
- `icon` — FontAwesome icon class (e.g., "fa-bar-chart")
- `if` — Conditional expression (e.g., `__config__.app.isApp('mymodule')`)

---

#### b) Create Top-Level Menu

```xml
<!-- Top-level menu -->
<menuitem name="custom-root"
          title="Custom Module"
          order="-500"
          icon="puzzle-piece"
          icon-background="#FF5733"
          if="__config__.app.isApp('custom-module')"/>

<!-- Submenu level 1 -->
<menuitem name="custom-root-orders"
          parent="custom-root"
          title="Custom Orders"
          action="action-custom-orders"
          order="100"/>

<!-- Submenu level 2 -->
<menuitem name="custom-root-orders-pending"
          parent="custom-root-orders"
          title="Pending Orders"
          action="action-pending-orders"
          order="100"/>
```

**Best practices:**
- ✅ **Use `if` condition để show menu khi module enabled** — `if="__config__.app.isApp('mymodule')"`
- ✅ **Order negative cho top menus** — E.g., -1000 (Invoicing), -900 (Sales), -500 (Custom)
- ✅ **Prefix menu names với module prefix** — E.g., `mymodule-root`, `mymodule-settings`

---

### 6. ORGANIZING CUSTOM MODULES — NAMING CONVENTIONS

#### a) Module Naming Strategy

**Pattern:** `<company-prefix>-<functional-area>`

Examples:
- `acme-sales-extension` — ACME company extending sales
- `acme-custom-reports` — ACME company custom reports
- `acme-integration-sap` — ACME company SAP integration

**Nguồn:** Axelor naming: `axelor-sale`, `axelor-account`, `axelor-bank-payment`

---

#### b) Package Naming

**Pattern:** `com.<company>.apps.<module>.{service,web,db.repo}`

Examples:
- `com.acme.apps.salesext.service` — Services
- `com.acme.apps.salesext.web` — Controllers
- `com.acme.apps.salesext.db.repo` — Repositories

✅ **Follow Axelor convention:** `com.axelor.apps.<module>.*`

---

#### c) Module Dependency Order

**Strategy: Layer modules theo complexity**

```
Level 0: Base framework
  └── axelor-core, axelor-common

Level 1: Core business modules
  └── axelor-base, axelor-auth

Level 2: Functional modules
  └── axelor-sale, axelor-account, axelor-purchase

Level 3: Integration modules
  └── axelor-bank-payment (depends on axelor-account)

Level 4: Custom modules
  └── acme-sales-extension (depends on axelor-sale)
  └── acme-custom-integration (depends on acme-sales-extension)
```

**Rule:** Custom modules nên depend vào functional/integration modules, KHÔNG nên có circular dependencies.

---

#### d) File Organization trong Custom Module

```
my-custom-module/
├── src/main/java/
│   └── com/mycompany/apps/mymodule/
│       ├── module/
│       │   └── MyCustomModule.java          # Guice bindings
│       ├── service/
│       │   ├── app/                          # App-level services
│       │   │   ├── AppMyModuleService.java
│       │   │   └── AppMyModuleServiceImpl.java
│       │   ├── order/                        # Domain services
│       │   │   ├── OrderProcessService.java
│       │   │   └── OrderProcessServiceImpl.java
│       │   └── integration/                  # Integration services
│       │       ├── SAPIntegrationService.java
│       │       └── SAPIntegrationServiceImpl.java
│       ├── db/repo/                          # Custom repositories
│       │   └── CustomOrderRepository.java
│       └── web/                              # Controllers
│           └── CustomOrderController.java
├── src/main/resources/
│   ├── domains/
│   │   ├── CustomOrder.xml                   # New entities
│   │   └── SaleOrder.xml                     # Extensions to existing
│   └── views/
│       ├── CustomOrder.xml                   # Views for new entities
│       ├── SaleOrder.xml                     # Extensions to existing views
│       └── Menu.xml                          # Menu definitions
└── build.gradle
```

**Organizing principles:**
- **Group services by domain** — `order/`, `invoice/`, `integration/`
- **Separate new entities from extensions** — Different domain XML files
- **One view file per entity** — `CustomOrder.xml` cho views của `CustomOrder`
- **Shared Menu.xml** — All menus trong một file

---

### 7. WORKFLOW: QUY TRÌNH EXTEND MỘT MODULE

#### Step-by-step workflow khi extend existing module:

---

**STEP 1: Identify extension points**

Analyze module bạn muốn extend:
- Entities nào cần thêm fields?
- Services nào cần override logic?
- Views nào cần thêm fields/buttons?
- Menus nào cần thêm submenus?

Tools:
```bash
# Find entity definitions
find modules/axelor-sale -name "*.xml" -path "*/domains/*"

# Find services
find modules/axelor-sale -name "*Service*.java" -path "*/service/*"

# Find views
find modules/axelor-sale -name "*.xml" -path "*/views/*"
```

---

**STEP 2: Create custom module structure**

```bash
mkdir -p modules/my-custom-module/src/main/java/com/mycompany/apps/mymodule/module
mkdir -p modules/my-custom-module/src/main/java/com/mycompany/apps/mymodule/service
mkdir -p modules/my-custom-module/src/main/java/com/mycompany/apps/mymodule/web
mkdir -p modules/my-custom-module/src/main/java/com/mycompany/apps/mymodule/db/repo
mkdir -p modules/my-custom-module/src/main/resources/domains
mkdir -p modules/my-custom-module/src/main/resources/views
```

---

**STEP 3: Create `build.gradle`**

```gradle
plugins {
  id 'com.axelor.app'
}

axelor {
  title "My Custom Module"
  description "Extensions to Sale module"
}

dependencies {
  api project(":modules:axelor-sale")
}
```

Gradle auto-discovers module (no settings.gradle edit needed).

---

**STEP 4: Extend entities**

**`src/main/resources/domains/SaleOrder.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<domain-models xmlns="http://axelor.com/xml/ns/domain-models"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://axelor.com/xml/ns/domain-models
                      http://axelor.com/xml/ns/domain-models/domain-models_7.4.xsd">

  <module name="sale" package="com.axelor.apps.sale.db"/>

  <entity name="SaleOrder">
    <string name="customApprovalNotes" large="true"/>
    <many-to-one name="customApprover" ref="com.axelor.auth.db.User"/>
  </entity>

</domain-models>
```

---

**STEP 5: Generate code**

```bash
./gradlew generateCode
```

Verify generated getters:
```bash
grep "customApprovalNotes" modules/axelor-sale/build/src-gen/java/com/axelor/apps/sale/db/SaleOrder.java
```

---

**STEP 6: Extend views**

**`src/main/resources/views/SaleOrder.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<object-views xmlns="http://axelor.com/xml/ns/object-views"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://axelor.com/xml/ns/object-views
                      http://axelor.com/xml/ns/object-views/object-views_7.4.xsd">

  <form name="sale-order-form" title="Sale Order"
        model="com.axelor.apps.sale.db.SaleOrder"
        extension="true">

    <extend target="//panel[@name='mainPanel']">
      <insert position="inside">
        <panel name="customApprovalPanel" title="Custom Approval" colSpan="12">
          <field name="customApprover"/>
          <field name="customApprovalNotes" colSpan="12"/>
        </panel>
      </insert>
    </extend>

  </form>

</object-views>
```

---

**STEP 7: Override services (if needed)**

**`MyCustomSaleOrderServiceImpl.java`:**

```java
package com.mycompany.apps.mymodule.service;

import com.axelor.apps.sale.service.saleorder.SaleOrderComputeServiceImpl;
import com.axelor.apps.sale.db.SaleOrder;
import com.axelor.apps.base.AxelorException;
import com.google.inject.Inject;
import com.google.inject.Singleton;

@Singleton
public class MyCustomSaleOrderServiceImpl extends SaleOrderComputeServiceImpl {

  @Inject
  public MyCustomSaleOrderServiceImpl(/* parent dependencies */) {
    super(/* pass to parent */);
  }

  @Override
  public void computeSaleOrder(SaleOrder saleOrder) throws AxelorException {
    // Custom logic before
    validateCustomApproval(saleOrder);

    // Call parent
    super.computeSaleOrder(saleOrder);

    // Custom logic after
    notifyCustomApprover(saleOrder);
  }

  protected void validateCustomApproval(SaleOrder saleOrder) throws AxelorException {
    if (saleOrder.getCustomApprover() == null) {
      throw new AxelorException(TraceBackRepository.CATEGORY_MISSING_FIELD,
          "Custom approver is required");
    }
  }

  protected void notifyCustomApprover(SaleOrder saleOrder) {
    // Send notification logic
  }
}
```

---

**STEP 8: Register bindings**

**`MyCustomModule.java`:**

```java
package com.mycompany.apps.mymodule.module;

import com.axelor.app.AxelorModule;
import com.axelor.apps.sale.service.saleorder.SaleOrderComputeServiceImpl;
import com.mycompany.apps.mymodule.service.MyCustomSaleOrderServiceImpl;

public class MyCustomModule extends AxelorModule {

  @Override
  protected void configure() {
    bind(SaleOrderComputeServiceImpl.class).to(MyCustomSaleOrderServiceImpl.class);
  }
}
```

---

**STEP 9: Test extension**

```bash
# Build
./gradlew build

# Run application
./gradlew run

# Navigate to Sale Order form, verify:
# - Custom fields visible
# - Custom logic executes
```

---

**STEP 10: Version control**

```bash
git add modules/my-custom-module
git commit -m "feat: add custom approval workflow to sale orders"
```

**IMPORTANT:** Custom modules tracked trong Git, base Axelor modules NOT tracked (declared as dependencies).

---

### 8. BEST PRACTICES & PATTERNS

#### ✅ DO's

1. **Always prefer extension over modification**
   - Extend entities via domain XML
   - Extend views via xpath
   - Override services via Guice bindings
   - ❌ **NEVER modify source code of base modules** (axelor-sale, axelor-account, etc.)

2. **Use semantic versioning cho custom modules**
   ```gradle
   version = "1.2.0"  // Major.Minor.Patch
   ```

3. **Document extension points**
   ```java
   /**
    * Custom extension of SaleOrderComputeService.
    *
    * Overrides:
    * - computeSaleOrder(): Adds custom approval validation
    *
    * New methods:
    * - validateCustomApproval(): Checks custom approver field
    * - notifyCustomApprover(): Sends notification to approver
    */
   public class MyCustomSaleOrderServiceImpl extends SaleOrderComputeServiceImpl {
   ```

4. **Test extensions in isolation**
   - Create integration tests cho custom logic
   - Mock base services để verify override behavior

5. **Keep custom modules small and focused**
   - One module per functional area
   - E.g., `acme-sales-approval` (approval workflow), `acme-sales-reporting` (custom reports)

---

#### ❌ DON'Ts

1. **Don't copy-paste entity definitions**
   ```xml
   <!-- ❌ BAD: Copy entire SaleOrder entity definition để modify -->
   <entity name="SaleOrder">
     <!-- 50 fields copied from original... -->
     <string name="myCustomField"/>  <!-- Only this is new! -->
   </entity>

   <!-- ✅ GOOD: Only declare new field -->
   <module name="sale" package="com.axelor.apps.sale.db"/>
   <entity name="SaleOrder">
     <string name="myCustomField"/>
   </entity>
   ```

2. **Don't create unnecessary custom repositories**
   - Only override Repository nếu cần custom hooks
   - ❌ **Don't override just để inject services** — Use services trong Controllers/Services

3. **Don't hardcode module order**
   - Let Gradle manage dependencies
   - Avoid manual ordering trong settings.gradle

4. **Don't break encapsulation**
   ```java
   // ❌ BAD: Access private fields via reflection
   Field privateField = saleOrder.getClass().getDeclaredField("privateData");
   privateField.setAccessible(true);

   // ✅ GOOD: Use public API hoặc extend entity để add field
   saleOrder.getCustomData();  // via extended field
   ```

5. **Don't override views completely**
   - Prefer xpath extension
   - Complete override = merge conflicts khi upgrade

---

### 9. TROUBLESHOOTING EXTENSIONS

#### Issue: "Entity not found after adding field"

**Symptom:** Field không xuất hiện trong generated Java class sau `generateCode`.

**Solution:**
- ✅ Verify `<module name="..." package="...">` matches ORIGINAL entity package
- ✅ Entity name exactly matches (case-sensitive)
- ✅ Run `./gradlew clean generateCode` — Clean build cache

---

#### Issue: "View extension not working"

**Symptom:** Xpath extension không apply, field không hiện trong form.

**Solution:**
- ✅ Set `extension="true"` trong form/grid definition
- ✅ Verify xpath selector — Use browser DevTools để inspect actual view structure
- ✅ Check module load order — Custom module phải load AFTER base module
- ✅ Test xpath:
  ```bash
  # Search views trong base module
  grep -r 'name="mainPanel"' modules/axelor-sale/src/main/resources/views/
  ```

---

#### Issue: "Service override not applied"

**Symptom:** Custom service không được gọi, original service vẫn execute.

**Solution:**
- ✅ Verify Guice binding — `bind(ParentServiceImpl.class).to(CustomServiceImpl.class)`
- ✅ Bind implementation class, NOT interface (trừ khi muốn override interface)
- ✅ Check custom module loaded — Module phải có trong `settings.gradle` auto-discovery
- ✅ Verify constructor injection — Parent dependencies phải pass qua `super()`

---

#### Issue: "Circular dependency detected"

**Symptom:** Application fails to start, Guice reports circular dependency.

**Solution:**
- ✅ Review service dependencies — A depends on B, B depends on A
- ✅ Use Provider<T> để break cycle:
  ```java
  @Inject
  protected Provider<ServiceB> serviceBProvider;

  public void method() {
    ServiceB serviceB = serviceBProvider.get();
  }
  ```
- ✅ Refactor logic — Extract common logic vào third service

---

### 10. UPGRADING AXELOR WITH EXTENSIONS

#### Strategy khi upgrade Axelor framework versions:

**BEFORE upgrade:**

1. **Document current extensions**
   ```bash
   # List all extended entities
   find modules/my-custom-module -name "*.xml" -path "*/domains/*" -exec basename {} \;

   # List all overridden services
   grep "bind(" modules/my-custom-module/src/main/java/*/module/*.java
   ```

2. **Backup database**
   ```bash
   pg_dump axelor_db > backup_before_upgrade.sql
   ```

---

**DURING upgrade:**

1. **Update Axelor version**
   ```gradle
   // build.gradle (root)
   axelor {
     version = "8.6.0"  // New version
   }
   ```

2. **Update plugin version**
   ```gradle
   // settings.gradle
   plugins {
     id 'com.axelor.app' version '7.5.0'  // New plugin version
   }
   ```

3. **Regenerate code**
   ```bash
   ./gradlew clean
   ./gradlew generateCode
   ```

4. **Fix compilation errors**
   - Parent service signatures changed? Update custom service constructors
   - Entity fields removed? Remove references trong custom code
   - New required fields? Add trong extension XMLs

---

**AFTER upgrade:**

1. **Test all extension points**
   - Verify forms render correctly
   - Test overridden service methods execute
   - Verify menu items appear

2. **Check logs cho warnings**
   ```bash
   grep "WARN" logs/axelor.log | grep -i "custom"
   ```

3. **Run integration tests**
   ```bash
   ./gradlew test
   ```

---

### Tổng kết Extension Strategy

| Extension Type | Complexity | Upgrade Risk | Use Case |
|----------------|------------|--------------|----------|
| **Entity fields** | Low | Low | Thêm data fields vào existing entities |
| **View xpath** | Low | Medium | Thêm UI elements vào forms/grids |
| **Service override** | Medium | Medium | Modify business logic behavior |
| **Repository hooks** | Medium | Low | Add entity lifecycle logic |
| **Menu items** | Low | Low | Add navigation entries |
| **Complete view override** | Low | High | Replace entire UI (avoid!) |

**Golden rule:** Càng ít override càng tốt. Extension > Override > Replacement.

Hiểu rõ extension patterns giúp build custom modules maintainable, upgrade-friendly, và collaborate với base Axelor modules một cách sạch sẽ.

## Những điều KHÔNG tìm thấy trong source code

### 1. Quy trình tạo custom module đầy đủ

❌ **Không tìm thấy:** Documentation hoặc example project cho việc tạo module mới từ đầu. Không có scaffold tools hoặc Gradle tasks để generate module skeleton.

**Ảnh hưởng:** Developers phải reverse-engineer existing modules để hiểu structure. Likely copy-paste một module đơn giản rồi modify. Error-prone và inconsistent naming.

### 2. Domain XML code generation tooling

❌ **Không tìm thấy:** Source code của code generator (tool đọc domain XML sinh Java classes). Chỉ thấy generated output trong `/build/src-gen/`.

**Ảnh hưởng:** Không hiểu exact generation rules. Khi domain XML có lỗi cú pháp, error messages cryptic. Không customize được generator behavior.

### 3. Testing infrastructure và examples

❌ **Rất ít test code:** Chỉ tìm thấy 1 test file trong axelor-sale (`TestSaleOrderDiscountService.java`). Không có test base classes, test utilities, hoặc mock frameworks setup.

**Ảnh hưởng:** Unclear làm sao write tests cho Services, Controllers, Repositories. No guidance về integration testing strategies. Production code coverage likely very low.

### 4. Studio export/import mechanisms

❌ **Không tìm thấy:** Built-in tools để export Studio apps (models + views + actions + BPM) sang portable format. Không có version control cho Studio configs.

**Ảnh hưởng:** Pain point lớn cho team collaboration. Studio changes khó track, hard merge conflicts, no audit trail.

### 5. Error handling best practices

❌ **Inconsistent patterns:** Một số Services throw checked `AxelorException`, một số throw unchecked exceptions, một số wrap trong `PersistenceException`.

**Ảnh hưởng:** Unclear khi nào dùng exception type nào. Controllers phải catch multiple exception types. User-facing error messages inconsistent.

### 6. View extension documentation

❌ **Không tìm thấy:** Detailed docs về XML xpath syntax để extend views. View inheritance rules unclear (khi nào override, khi nào merge).

**Ảnh hưởng:** Trial-and-error khi customize views. Breaking changes khi base views thay đổi structure.

### 7. Performance optimization guidelines

❌ **Không tìm thấy:** Guidance về N+1 query problems, lazy loading strategies, batch processing patterns, caching strategies cho custom code.

**Ảnh hưởng:** Developers write inefficient code. Performance issues discovered in production.

### 8. Migration tools khi upgrade Axelor versions

❌ **Không tìm thấy:** Tools hoặc scripts để migrate custom code/Studio configs khi upgrade Axelor framework (ví dụ: 8.x → 9.x).

**Ảnh hưởng:** Risky upgrades. Might break custom code hoặc Studio apps. Manual testing required.

---

## Câu hỏi mở cần kiểm chứng thực tế

### 1. Transaction management edge cases

**Câu hỏi:** Khi Service A (transactional) gọi Service B (transactional), và B throw exception, transaction của A có rollback không? Nested transaction behavior exactly như thế nào?

**Cách kiểm chứng:** Write integration test with two services, deliberately throw exception in inner service, verify database state.

### 2. Beans.get() vs Constructor Injection performance

**Câu hỏi:** `Beans.get(Service.class)` có performance penalty so với injected services không? Nếu có, bao nhiêu?

**Cách kiểm chứng:** Benchmark 1000 calls `Beans.get()` vs 1000 calls injected service method. Measure overhead.

### 3. Studio model với large datasets

**Câu hỏi:** `meta_json_record` table với JSONB column handle scale ra sao? Performance với 1M+ records?

**Cách kiểm chứng:** Create Studio model, insert 1M records, benchmark query performance vs code-defined entity with indexed columns.

### 4. Studio BPM calling complex Code Services

**Câu hỏi:** BPM Groovy script gọi Service có nhiều dependencies - dependencies được inject đúng không? Circular dependencies có cause problems không?

**Cách kiểm chứng:** Create BPM process calling Service with complex dependency graph. Test execution, check for injection errors.

### 5. Custom Repository hooks trong transaction context

**Câu hỏi:** Nếu Repository `save()` hook throw exception, transaction có rollback không? Hook được execute trong cùng transaction với actual save không?

**Cách kiểm chứng:** Override `save()`, throw exception after `super.save()`, verify database unchanged.

### 6. View inheritance merge conflicts

**Câu hỏi:** Khi module A và module B cùng extend view của module Core, merge order ra sao? Ai "thắng" nếu conflict?

**Cách kiểm chứng:** Create scenario: Core has form, Module A extends adding field X at position 1, Module B extends adding field Y at position 1. Check resulting form.

### 7. Guice module load order và override behaviors

**Câu hỏi:** Khi multiple modules bind cùng interface, module nào loaded last thắng? Có thể explicitly control bind priority không?

**Cách kiểm chứng:** Create two modules binding same interface to different implementations. Check which implementation injected.

### 8. ActionRequest context data types

**Câu hỏi:** `request.getContext().get("field")` return type exactly là gì khi field là date, decimal, many-to-one? Có type coercion không?

**Cách kiểm chứng:** Create form với various field types, submit action, log exact types returned by `getContext().get()`.

### 9. Studio export/import workarounds

**Câu hỏi:** Có unofficial tools hoặc community scripts để export/import Studio configs không? Best practices từ real projects?

**Cách kiểm chứng:** Search Axelor forums, GitHub, Stack Overflow. Ask community.

### 10. Code-defined entity trong Studio views

**Câu hỏi:** Studio có thể tạo custom views cho code-defined entities không? Customizations persist qua code deployments không?

**Cách kiểm chứng:** Create code entity, use Studio create custom form for it, redeploy code, check if Studio form still exists.

---

## Tổng kết và khuyến nghị

### Điểm mạnh của Axelor development model

1. **Clear separation of concerns:** Service-Controller-Repository layers rõ ràng, dễ navigate codebase lớn.
2. **Dependency injection với Guice:** Lightweight, fast startup, explicit bindings dễ debug.
3. **Generated code pattern:** Domain XML → Java giảm boilerplate, đảm bảo consistency.
4. **Studio cho rapid prototyping:** Business users có thể tạo simple models và workflows không cần developers.
5. **Extensibility:** Custom repositories, service overrides, view extensions cho phép customize mà không fork framework.

### Điểm yếu và pain points

1. **Thiếu testing infrastructure:** Rất ít test examples, unclear how test Services/Controllers, low confidence khi refactor.
2. **Studio version control:** Không có built-in tools cho team collaboration, config tracking, conflict resolution.
3. **Documentation gaps:** Thiếu docs về advanced patterns, best practices phải reverse-engineer từ source code.
4. **Steep learning curve:** Nhiều concepts phải học (Guice, domain XML generation, action system, Groovy scripting), không có guided onboarding.
5. **`Beans.get()` anti-pattern:** Service locator thay vì pure DI làm dependencies không explicit, harder to test.

### Chiến lược phát triển khuyến nghị

**Cho new projects:**
1. Bắt đầu với Studio để prototype data models và workflows - nhanh, không cần deployment.
2. Khi models stabilize, migrate sang code-defined entities - tốt hơn cho integrity và performance.
3. Giữ Studio cho tenant-specific customizations và ad-hoc models.

**Cho team collaboration:**
1. Designate "Studio admin" quản lý Studio changes, tránh conflicts.
2. Export Studio configs sang CSV sau major changes, commit vào Git.
3. Document Studio models trong README - code comments không cover được Studio.

**Cho code quality:**
1. Viết tests cho Services ngay từ đầu - dù không có examples, setup JUnit + Mockito vẫn work.
2. Keep Services small và focused - theo pattern của axelor-sale.
3. Prefer constructor injection trong Services, accept `Beans.get()` trong Controllers.
4. Document exception handling strategy - khi throw `AxelorException` vs runtime exceptions.

**Cho performance:**
1. Use code-defined entities cho high-volume data - Studio `meta_json_record` không optimize cho scale.
2. Implement custom Repository hooks cho computed fields thay vì compute runtime.
3. Leverage Guice singleton scope - Services nặng (với nhiều dependencies) nên singleton.

Axelor là một low-code platform mạnh mẽ cho Java developers, với balance tốt giữa flexibility (Studio no-code) và control (code traditional). Success phụ thuộc vào hiểu rõ ranh giới giữa hai approaches và áp dụng đúng tool cho đúng job.
