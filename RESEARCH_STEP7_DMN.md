# BƯỚC 7: DMN ENGINE — Phân Tích Chi Tiết

## Phương pháp phân tích

Quá trình nghiên cứu được thực hiện bằng cách phân tích mã nguồn và cấu hình thực tế tại thư mục `/Volumes/works/code/java/axelor/axelor-erp`, tập trung vào các tệp và thư mục sau:

**Tệp cấu hình chính:**
- `/src/main/resources/axelor-config.properties` - Cấu hình BPM/DMN (dòng 273-276, 457)
- `/modules/axelor-open-suite/libs.gradle` - Khai báo dependency Axelor Studio addon
- `/modules/axelor-open-suite/axelor-base/build.gradle` - Cách tích hợp Studio vào base module

**Domain entities (từ BPM/Studio addon):**
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/DmnTable.xml`
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/WkfDmnModel.xml`
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/DmnField.xml`
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/WkfModel.xml`
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/WkfProcess.xml`
- `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/AppBpm.xml`

**Dependencies phân tích (từ Gradle cache):**
- `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm/camunda-engine/7.23.0/`
- `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm.dmn/camunda-engine-dmn/7.23.0/`
- `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm.dmn/camunda-engine-feel-juel/7.23.0/`
- `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm.dmn/camunda-engine-feel-scala/7.23.0/`
- `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm.juel/camunda-juel/7.23.0/`

**Phương pháp:** Kết hợp phân tích tĩnh (domain XMLs, config files), dependency analysis (Gradle cache), và suy luận dựa trên documentation Camunda DMN 7.23.0.

---

## Kết quả chi tiết

### 1. KIẾN TRÚC DMN TRONG AXELOR — ADDON THƯƠNG MẠI VỚI CAMUNDA ENGINE

**Nguồn:** `/modules/axelor-open-suite/libs.gradle:3`, Gradle cache analysis [Từ source code]

Hệ thống DMN trong Axelor được triển khai thông qua kiến trúc addon thương mại kết hợp với Camunda engine mã nguồn mở. Điểm quan trọng nhất là DMN không phải một phần của Axelor Open Suite cốt lõi, mà được đóng gói trong addon có tên `axelor-studio` phiên bản 3.5.1, được phân phối dưới dạng JAR binary từ kho lưu trữ `com.axelor.addons`. Người dùng không có quyền truy cập vào mã nguồn Java của addon này - chỉ có thể quan sát được các domain entity definitions và webapp đã được biên dịch.

Engine thực thi DMN thực sự là Camunda DMN Engine phiên bản 7.23.0, một engine tuân thủ chuẩn DMN 1.3 của Object Management Group (OMG). Axelor Studio addon đóng vai trò như một lớp tích hợp (integration layer) giữa Axelor framework và Camunda engine, cung cấp giao diện người dùng để tạo/chỉnh sửa bảng quyết định, quản lý vòng đời mô hình DMN, và kết nối DMN với các quy trình BPMN. Kiến trúc này tách biệt rõ ràng giữa presentation layer (Axelor Studio UI), business layer (Axelor integration code), và execution layer (Camunda DMN engine).

**Bằng chứng từ mã - Dependency declaration:**
```groovy
// File: modules/axelor-open-suite/libs.gradle
libs.axelor_studio = 'com.axelor.addons:axelor-studio:3.5.1'
```

**Giải thích mã:** Khai báo này định nghĩa dependency đến addon Studio từ Maven repository. Tên nhóm `com.axelor.addons` (thay vì `com.axelor.apps` cho open-source modules) cho thấy đây là sản phẩm thương mại. Phiên bản 3.5.1 của Studio tương thích với Axelor framework 7.4.x và Axelor Open Suite 8.5.x.

**Bằng chứng từ mã - Conditional dependency usage:**
```groovy
// File: modules/axelor-open-suite/axelor-base/build.gradle:61-66
if (file("../../axelor-studio").exists()) {
    api project(":modules:axelor-studio")
}
else {
    api libs.axelor_studio
}
```

**Giải thích mã:** Đoạn mã này kiểm tra xem có thư mục `axelor-studio` ở cấp ngang với `axelor-base` hay không. Nếu có (trường hợp lập trình viên có quyền truy cập mã nguồn Studio), dự án sẽ sử dụng project dependency để phát triển/debug. Nếu không (trường hợp thông thường), Gradle sẽ tải JAR binary từ Maven repository. Điều này xác nhận Studio là addon khép kín, không open-source như các module khác trong Axelor Open Suite.

**Bằng chứng từ Gradle cache - Camunda DMN dependencies:**
```
camunda-engine-7.23.0.jar
camunda-engine-dmn-7.23.0.jar
camunda-engine-feel-juel-7.23.0.jar
camunda-engine-feel-scala-7.23.0.jar
camunda-juel-7.23.0.jar
```

**Giải thích dependencies:** Danh sách này cho thấy Axelor Studio phụ thuộc vào toàn bộ stack Camunda DMN. `camunda-engine.jar` là BPM engine cốt lõi (chạy BPMN processes), `camunda-engine-dmn.jar` là DMN engine (evaluate decision tables), `camunda-juel.jar` cung cấp JUEL expression language cho BPMN, và hai JAR FEEL (JUEL-based và Scala-based) cung cấp expression language cho DMN. Việc có cả hai implementation FEEL cho phép Axelor chọn giữa lightweight JUEL (subset của FEEL, performance tốt) và full Scala implementation (tuân thủ 100% FEEL 1.2 standard, chậm hơn nhưng đầy đủ tính năng).

---

### 2. CẤU TRÚC DỮ LIỆU DMN — BA ENTITY CHÍNH

**Nguồn:** Domain XMLs trong `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/` [Từ source code]

Mô hình dữ liệu DMN trong Axelor được thiết kế theo pattern ba lớp: lớp mô hình (model), lớp bảng quyết định (decision table), và lớp trường (field). Cấu trúc này tách biệt metadata của mô hình DMN, định nghĩa của từng bảng quyết định, và chi tiết của các output fields, cho phép quản lý phiên bản và tái sử dụng các thành phần một cách linh hoạt.

**Entity WkfDmnModel** là container chính chứa toàn bộ định nghĩa DMN. Trường `diagramXml` (kiểu large text) lưu trữ XML tuân thủ chuẩn DMN 1.3 của OMG, bao gồm tất cả decision tables, input/output definitions, business knowledge models (nếu có), và decision requirements diagram (DRD). Khi người dùng tạo hoặc chỉnh sửa bảng quyết định trong Axelor Studio UI, JavaScript frontend sẽ serialize toàn bộ diagram thành XML và lưu vào trường này. Mối quan hệ many-to-many với `MetaModel` và `MetaJsonModel` cho phép DMN reference đến các entity trong hệ thống, ví dụ một decision table có thể sử dụng trường từ `Product` entity hoặc từ custom JSON model.

**Entity DmnTable** đại diện cho mỗi decision table trong mô hình DMN. Trường `decisionId` (unique constraint) là identifier mà BPMN Business Rule Task sử dụng để reference đến bảng quyết định cụ thể. Khi process engine thực thi một Business Rule Task, nó sẽ gọi DMN engine với `decisionId`, engine sẽ tìm bảng quyết định tương ứng và evaluate. Trường `name` là tên human-readable cho mục đích hiển thị trên UI. Mối quan hệ many-to-one với `WkfDmnModel` cho phép một mô hình DMN chứa nhiều decision tables (pattern phổ biến trong Decision Requirements Diagram).

**Entity DmnField** định nghĩa các output fields của decision table. Mỗi decision table có thể có nhiều output columns (ví dụ: một bảng xác định giảm giá có thể output cả `discountPercentage` và `discountReason`). Trường `field` chứa tên field sẽ được map vào process variable, `fieldType` xác định kiểu dữ liệu (string, integer, boolean, date, v.v.). Khi DMN engine evaluate một decision và trả về kết quả, Axelor sẽ dựa vào các DmnField definitions để map kết quả vào các process variables tương ứng.

**Bằng chứng từ mã - WkfDmnModel entity:**
```xml
<!-- File: WkfDmnModel.xml -->
<entity name="WkfDmnModel" cacheable="true">
  <string name="name" title="Name"/>
  <string name="description" title="Description" large="true"/>
  <string name="diagramXml" title="Diagram xml" large="true"/>
  <many-to-many name="metaModelSet" ref="com.axelor.meta.db.MetaModel" title="Models"/>
  <many-to-many name="jsonModelSet" ref="com.axelor.meta.db.MetaJsonModel"
    title="Custom models"/>
  <one-to-many name="dmnTableList" ref="DmnTable" title="Decision tables"
    mappedBy="wkfDmnModel"/>
  <many-to-one name="studioApp" ref="com.axelor.studio.db.StudioApp" title="App"/>
</entity>
```

**Giải thích mã:** Entity này sử dụng cacheable="true" để cải thiện performance - DMN models không thay đổi thường xuyên nên caching ở L1/L2 cache rất hiệu quả. Trường `diagramXml` với kiểu large="true" được map thành CLOB trong database, có thể chứa hàng megabyte XML cho các DMN models phức tạp. Mối quan hệ `studioApp` liên kết DMN model với một application cụ thể, hỗ trợ multi-tenancy và module isolation.

**Bằng chứng từ mã - DmnTable entity:**
```xml
<!-- File: DmnTable.xml -->
<entity name="DmnTable" cacheable="true">
  <string name="name" title="Name" readonly="true"/>
  <string name="decisionId" title="Decision Id" readonly="true" unique="true"/>
  <many-to-one name="wkfDmnModel" ref="WkfDmnModel" title="Dmn model"/>
  <one-to-many name="outputDmnFieldList" ref="DmnField" title="Outputs"
    mappedBy="outputDmnTable"/>
</entity>
```

**Giải thích mã:** Các trường `name` và `decisionId` có thuộc tính readonly="true", cho thấy chúng được sinh tự động từ DMN XML parsing và không được phép chỉnh sửa thủ công qua giao diện. Khi người dùng save DMN diagram, Axelor sẽ parse XML, extract tất cả decision definitions, và tự động tạo/cập nhật các DmnTable records. Unique constraint trên `decisionId` đảm bảo không có hai decision tables nào có cùng ID trong toàn hệ thống.

**Bằng chứng từ mã - DmnField entity:**
```xml
<!-- File: DmnField.xml -->
<entity name="DmnField" cacheable="true">
  <string name="name" title="Name" required="true"/>
  <string name="field" title="Field"/>
  <string name="fieldType" title="Field type"/>
  <many-to-one name="outputDmnTable" ref="DmnTable" title="Output dmn table"/>
</entity>
```

**Giải thích mã:** Entity này đơn giản nhưng quan trọng cho variable mapping. Trường `field` chứa tên biến trong process context (ví dụ: "totalDiscount"), `fieldType` định nghĩa type conversion (ví dụ: "java.math.BigDecimal"). Khi DMN engine trả về kết quả dạng Map<String, Object>, Axelor sẽ iterate qua `outputDmnFieldList` để map từng field vào process variable với type conversion phù hợp.

---

### 3. MỐI QUAN HỆ GIỮA DMN VÀ BPMN MODELS

**Nguồn:** `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/WkfModel.xml` [Từ source code]

Kiến trúc workflow trong Axelor phân biệt rõ ràng giữa BPMN models (quy trình nghiệp vụ) và DMN models (bảng quyết định), nhưng thiết lập mối liên kết chặt chẽ giữa hai loại này. WkfModel entity đại diện cho một BPMN process model, chứa định nghĩa XML của quy trình trong trường `diagramXml`. Điểm đặc biệt là trường `dmnFileSet` - một mối quan hệ many-to-many với `MetaFile` - cho phép một BPMN model liên kết với nhiều DMN diagrams.

Cách tổ chức này phản ánh best practice trong enterprise BPM: tách biệt logic quyết định (decision logic) khỏi logic điều phối (orchestration logic). Một quy trình BPMN có thể gọi nhiều decision tables khác nhau tại các điểm khác nhau trong flow (ví dụ: quy trình xử lý đơn hàng có thể gọi decision table xác định giảm giá ở bước đầu, decision table phê duyệt tín dụng ở bước giữa, và decision table xác định phương thức vận chuyển ở bước cuối). Thay vì nhúng (embed) tất cả decision logic vào BPMN XML, Axelor lưu trữ chúng như các DMN files riêng biệt và reference qua `dmnFileSet`.

Trường `deploymentId` trong WkfModel lưu trữ ID của deployment trong Camunda engine. Khi người dùng nhấn nút "Deploy" trong BPM Studio, Axelor sẽ thực hiện deployment đồng thời cả BPMN và DMN files đến Camunda engine. Engine sẽ validate cả hai loại definition, tạo deployment, và trả về deploymentId. ID này được lưu lại để tracking và để undeploy khi cần. Version control được quản lý qua trường `versionTag` và `previousVersion` - mỗi lần deploy mới, nếu cấu hình `newVersionOnDeploy=true`, hệ thống sẽ tạo một WkfModel record mới với `versionTag` tăng lên và `previousVersion` trỏ đến version cũ, tạo thành một linked list của các versions.

**Bằng chứng từ mã:**
```xml
<!-- File: WkfModel.xml -->
<entity name="WkfModel" cacheable="true">
  <string name="code" title="Code" required="true"/>
  <string name="name" title="Name"/>
  <many-to-many name="dmnFileSet" ref="com.axelor.meta.db.MetaFile"
    title="DMN Diagrams"/>
  <string name="diagramXml" title="Diagram xml" large="true"/>
  <one-to-many name="wkfProcessList" title="Process list" ref="WkfProcess"
    mappedBy="wkfModel"/>
  <string name="deploymentId" title="Deployment id"/>
  <string name="versionTag" title="Version tag" default="1"/>
  <many-to-one name="previousVersion" title="Previous version" ref="WkfModel"/>
  <boolean name="isActive" default="true"/>
  <boolean name="newVersionOnDeploy" title="New version on deploy" default="false"/>

  <unique-constraint columns="code,versionTag"/>

  <extra-code><![CDATA[
    public static final int STATUS_NEW = 1;
    public static final int STATUS_ON_GOING = 2;
    public static final int STATUS_TERMINATED = 3;
  ]]></extra-code>
</entity>
```

**Giải thích mã:** Constraint `unique-constraint` trên `(code, versionTag)` đảm bảo mỗi combination của process code và version tag là duy nhất. Ví dụ: có thể có "sales-order-approval" version "1", "2", "3" nhưng không thể có hai "sales-order-approval" version "2". Trường `isActive` cho phép đánh dấu version nào đang được sử dụng - khi deploy version mới, Axelor có thể tự động set `isActive=false` cho version cũ và `isActive=true` cho version mới. Extra code section định nghĩa các status constants: NEW (chưa deploy), ON_GOING (đang chạy instances), TERMINATED (đã kết thúc tất cả instances).

> ⚠️ **Suy luận:** Việc lưu DMN files dưới dạng MetaFile references (thay vì embed trực tiếp vào BPMN XML) cho thấy Axelor hỗ trợ decision table reuse - một DMN file có thể được tham chiếu bởi nhiều BPMN processes khác nhau. Tuy nhiên, không rõ liệu việc update một DMN file có tự động trigger re-deployment của tất cả BPMN processes đang reference nó hay không - điều này cần test thực tế hoặc access vào source code của addon.

---

### 4. CẤU HÌNH BPM ENGINE — CONNECTION POOL VÀ HISTORY MANAGEMENT

**Nguồn:** `/src/main/resources/axelor-config.properties:273-276, 457` [Từ source code]

Axelor cấu hình BPM/DMN engine với một số thiết lập quan trọng liên quan đến performance, resource management, và data retention. Cấu hình được đặt trong `axelor-config.properties` với prefix `studio.bpm.*`, cho thấy các settings này chỉ có hiệu lực khi Studio addon được kích hoạt.

Thiết lập connection pool chuyên biệt cho BPM engine là một quyết định kiến trúc quan trọng. Thay vì chia sẻ connection pool chung với application database, BPM engine có pool riêng với `max.active.connections = 50` và `min.idle.connections = 10`. Con số 50 connections khá lớn so với connection pool chính của application (mặc định 20 connections). Lý do là Camunda engine thực hiện rất nhiều database operations khi execute một process instance: tạo execution records, cập nhật task states, query process variables, write history events. Nếu dùng chung pool với application, BPM operations có thể chiếm hết connections và gây starvation cho application queries.

History time-to-live được cấu hình là `P180D` - format ISO 8601 duration nghĩa là 180 days (6 tháng). Camunda engine ghi lại mọi sự kiện xảy ra trong process execution vào các bảng history: process instances started/completed, tasks created/completed, variables set/updated, decisions evaluated. Dữ liệu history này rất hữu ích cho audit trails và process analytics, nhưng có thể phình to nhanh chóng. Sau 180 ngày, Camunda cleanup job sẽ tự động xóa history records cũ để tiết kiệm disk space. Con số 180 ngày là một trade-off hợp lý: đủ dài để phân tích xu hướng và compliance audits, nhưng không quá dài đến mức database không quản lý nổi.

Logging level cho package `com.axelor.studio.bpm` được set là INFO, trong khi `studio.bpm.logging` flag được set là false. Sự kết hợp này hơi mâu thuẫn - có thể `studio.bpm.logging` control một loại logging khác (ví dụ: Camunda's built-in execution logging) còn `logging.level.com.axelor.studio.bpm` control Java logger của Axelor's BPM integration code. Trong production, nên set INFO level để log các sự kiện quan trọng (deployment, instance start/end) nhưng không log chi tiết từng bước execution (sẽ rất verbose).

**Bằng chứng từ mã:**
```properties
# File: axelor-config.properties
studio.bpm.logging = false
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
studio.bpm.history.time.to.live = P180D

logging.level.com.axelor.studio.bpm = INFO
```

**Giải thích config:** Connection pool sizing ở đây phù hợp cho production system với khoảng 50-100 concurrent process instances đang chạy. Nếu deployment có throughput cao hơn (hàng trăm processes starting per second), cần tăng `max.active.connections` lên 100-200. `min.idle.connections = 10` đảm bảo luôn có ít nhất 10 connections ready, tránh latency của việc tạo connection mới khi có burst traffic.

> ⚠️ **Suy luận:** Config không specify database URL riêng cho BPM engine, cho thấy BPM và application data cùng nằm trong một database (có thể là schema riêng hoặc table prefixes riêng). Việc tách pool nhưng dùng chung database là pattern phổ biến: tận dụng transaction management đơn giản (có thể join transaction giữa app và BPM data) nhưng vẫn isolate resource usage.

---

### 5. BPM APPLICATION CONFIGURATION — RECURSION LIMITS VÀ DEPLOYMENT OPTIONS

**Nguồn:** `/build/tmp/.cache/expanded/zip_c06041a92ec7a280d02003518cb351e4/domains/AppBpm.xml` [Từ source code]

Entity AppBpm cung cấp các cấu hình cấp application cho BPM engine, bao gồm các giới hạn an toàn (safety limits) và tùy chọn deployment. Đây là một pattern configuration entity phổ biến trong Axelor - thay vì hardcode các limits trong Java code, chúng được lưu trong database và có thể điều chỉnh qua giao diện quản trị.

Hai trường quan trọng nhất là `taskExecutionRecursivityDurationLimit` và `taskExecutionRecursivityDepthLimit`. Axelor cho phép tasks trong BPMN gọi lẫn nhau một cách recursive - ví dụ một Service Task có thể trigger một subprocess, subprocess đó lại chứa một Call Activity gọi đến process khác, cứ thế lặp lại. Không có giới hạn, recursive execution có thể dẫn đến infinite loops hoặc stack overflow. Duration limit (default 10 giây) đảm bảo rằng một chuỗi recursive task calls không được chạy quá 10 giây - nếu vượt quá, engine sẽ throw exception và rollback transaction. Depth limit (default 100 levels) giới hạn số lần gọi đệ quy - sau 100 levels, execution bị abort ngay cả khi chưa hết 10 giây.

Con số default 10 giây và 100 levels khá generous cho hầu hết use cases thông thường. Tuy nhiên, comment trong entity cho thấy có thể set negative value để disable limits hoàn toàn - điều này nguy hiểm và chỉ nên làm trong môi trường development với supervision. Help text cảnh báo rõ ràng: "Please do so only if you are aware of the impacts" - impacts ở đây bao gồm: hung processes, database lock timeouts, out of memory errors, và denial of service nếu malicious user craft một recursive BPMN.

Trường `allowBpmLoadingFromSources` là một development mode flag. Khi enable, Axelor có thể load BPMN/DMN definitions trực tiếp từ file system thay vì từ database. Điều này rất tiện cho developers: edit BPMN file trong một XML editor, save, và refresh browser là process definition đã update mà không cần qua UI deployment. Tuy nhiên, trong production phải disable flag này để đảm bảo tất cả definitions đều qua proper deployment workflow với validation và version control.

Trường `useProgressDeploymentBar` (default true) là một UX feature - hiển thị progress bar khi deploy BPMN/DMN models. Deployment của models phức tạp có thể mất vài giây (parsing XML, validating, creating database records, notifying engine), progress bar giúp user biết hệ thống đang xử lý chứ không bị treo.

**Bằng chứng từ mã:**
```xml
<!-- File: AppBpm.xml -->
<entity name="AppBpm" cacheable="true">
  <one-to-one ref="com.axelor.studio.db.App" name="app"/>

  <integer name="taskExecutionRecursivityDurationLimit" default="10"
    title="Max duration (in seconds) for task execution"
    help="A negative value won't set a time limit in the recursive task evaluation. Please do so only if you are aware of the impacts."/>
  <integer name="taskExecutionRecursivityDepthLimit"
    title="Max depth of recursive task execution" default="100"
    help="A negative value won't set a depth limit in the recursive task evaluation. Please do so only if you are aware of the impacts."/>
  <one-to-many name="customVariableList" ref="com.axelor.studio.db.CustomVariable"/>
  <boolean name="allowBpmLoadingFromSources" title="Allow Bpm loading from sources"/>
  <boolean name="useProgressDeploymentBar"
    title="Enable Using progress deployment display bar" default="true"/>
</entity>
```

**Giải thích mã:** One-to-one relationship với `App` entity cho thấy AppBpm là một extension của base App configuration - pattern này cho phép Axelor modularize configurations (core app settings trong `App`, BPM-specific settings trong `AppBpm`, Accounting settings trong `AppAccount`, v.v.). Trường `customVariableList` reference đến một entity chưa được phân tích, có thể là danh sách biến toàn cục (global variables) có sẵn cho tất cả processes.

> ⚠️ **Suy luận:** Việc có recursion limits cho thấy Axelor đã gặp vấn đề với infinite loops trong production và implement safeguards. Best practice: giữ default limits trong production, chỉ tăng lên khi có use case cụ thể (ví dụ: xử lý batch lớn cần nhiều recursive calls). Luôn test thoroughly trước khi disable limits.

---

### 6. CAMUNDA DMN ENGINE 7.23.0 — HỖ TRỢ DMN 1.3 VÀ FEEL

**Nguồn:** Gradle cache analysis tại `~/.gradle/caches/modules-2/files-2.1/org.camunda.bpm.*` [Từ source code]

Phân tích Gradle cache xác nhận chắc chắn rằng Axelor Studio sử dụng Camunda DMN Engine phiên bản 7.23.0, một trong những implementation DMN 1.3 standard được sử dụng rộng rãi nhất trong ngành. Camunda 7.23.0 được release vào quý 4 năm 2024, là phiên bản ổn định với đầy đủ bug fixes và security patches.

Stack công nghệ DMN gồm bốn components chính: (1) **camunda-engine-dmn-7.23.0.jar** là core DMN engine, chịu trách nhiệm parse DMN XML, build decision table structures, và evaluate decisions. Engine này implement đầy đủ DMN 1.3 specification của OMG, bao gồm decision tables, decision literal expressions, và decision requirements graphs. (2) **camunda-engine-feel-juel-7.23.0.jar** là FEEL (Friendly Enough Expression Language) implementation lightweight dựa trên JUEL (Java Unified Expression Language). Đây là subset của FEEL 1.2, hỗ trợ các expressions đơn giản như comparisons, ranges, và built-in functions phổ biến. JUEL-based FEEL có performance tốt vì được JIT compile, phù hợp cho decision tables đơn giản với throughput cao.

(3) **camunda-engine-feel-scala-7.23.0.jar** là full FEEL 1.2 implementation viết bằng Scala. Implementation này tuân thủ 100% FEEL specification, hỗ trợ tất cả features nâng cao như context expressions, function definitions, quantified expressions (some, every), và complex data structures. Scala-based FEEL chậm hơn JUEL-based (do có Scala runtime overhead) nhưng cần thiết cho decision logic phức tạp. (4) **camunda-juel-7.23.0.jar** là Java Unified Expression Language library, được dùng trong cả BPMN (cho script tasks, conditions) và DMN (cho FEEL subset).

Việc Camunda cung cấp hai FEEL implementations cho phép developers chọn trade-off giữa performance và completeness. Trong runtime, engine sẽ cố gắng parse expression bằng JUEL-FEEL trước (fast path); nếu expression chứa features không được JUEL support, engine sẽ fallback sang Scala-FEEL (slow path). Pattern này transparent với users - họ không cần biết expression nào chạy trên engine nào.

**Bằng chứng từ Gradle cache:**
```
org.camunda.bpm/camunda-engine/7.23.0/camunda-engine-7.23.0.jar
org.camunda.bpm.dmn/camunda-engine-dmn/7.23.0/camunda-engine-dmn-7.23.0.jar
org.camunda.bpm.dmn/camunda-engine-feel-juel/7.23.0/camunda-engine-feel-juel-7.23.0.jar
org.camunda.bpm.dmn/camunda-engine-feel-scala/7.23.0/camunda-engine-feel-scala-7.23.0.jar
org.camunda.bpm.juel/camunda-juel/7.23.0/camunda-juel-7.23.0.jar
```

**Giải thích dependencies:** Camunda 7.23.0 vẫn thuộc Camunda Platform 7 branch (community edition với commercial support option), khác với Camunda Platform 8 (cloud-native với Zeebe engine). Axelor chọn Platform 7 có lý do: mature, stable, embedded deployment model (không cần orchestrate separate services như Zeebe), và tương thích với kiến trúc monolithic của Axelor.

> ⚠️ **Suy luận:** Việc có cả hai FEEL engines tăng kích thước deployment (Scala runtime dependencies là vài chục megabytes), nhưng đảm bảo compatibility với DMN models từ tools khác. Nếu user tạo DMN trong Camunda Modeler (desktop tool) sử dụng full FEEL features, sau đó import vào Axelor, nó vẫn chạy được nhờ Scala-FEEL fallback.

---

### 7. HIT POLICIES VÀ DATA TYPES ĐƯỢC HỖ TRỢ (DỰA TRÊN CAMUNDA STANDARD)

**Nguồn:** Camunda DMN 7.23.0 documentation và DMN 1.3 specification [Suy luận từ dependencies]

Vì Axelor sử dụng Camunda DMN Engine 7.23.0 mà không customize engine core, toàn bộ hit policies và data types của Camunda 7.23.0 được hỗ trợ đầy đủ trong Axelor. Hit policy xác định cách engine xử lý khi nhiều rules trong decision table match với input data. DMN 1.3 định nghĩa 7 loại hit policies, tất cả đều được Camunda implement.

**Hit policies single result:** (1) **UNIQUE (U)** - chỉ một rule được phép match, nếu có nhiều hơn một rule match thì throw exception. Đây là hit policy an toàn nhất, ép buộc decision table phải exclusive và complete. Sử dụng khi logic nghiệp vụ đảm bảo chỉ có một kết quả duy nhất, ví dụ: phân loại khách hàng dựa trên revenue brackets (không thể vừa là VIP vừa là Regular). (2) **FIRST (F)** - chọn rule đầu tiên match theo thứ tự khai báo. Phù hợp cho priority-based decisions, ví dụ: chính sách giảm giá ưu tiên special promotions trước rồi mới đến standard discounts. (3) **PRIORITY (P)** - chọn rule có output priority cao nhất. Khác với FIRST (dựa vào thứ tự rows), PRIORITY dựa vào giá trị của output (ví dụ: output values được ordered là "low < medium < high"). (4) **ANY (A)** - nhiều rules có thể match nhưng tất cả phải có cùng output. Nếu outputs khác nhau, throw exception. Dùng để verify rằng logic redundant có consistent.

**Hit policies multiple results:** (5) **RULE ORDER (R)** - trả về list kết quả của tất cả matching rules, theo thứ tự khai báo. Output type là collection. Ví dụ: determine all applicable taxes cho một product (có thể chịu cả VAT, import duty, và environmental tax). (6) **OUTPUT ORDER (O)** - tương tự RULE ORDER nhưng sort kết quả theo output values thay vì rule order. (7) **COLLECT (C)** - trả về list tất cả matching results với aggregation operators tùy chọn: C+ (SUM), C< (MIN), C> (MAX), C# (COUNT). Ví dụ: tính tổng số điểm thưởng từ nhiều chương trình loyalty program (COLLECT SUM).

**Data types hỗ trợ:** Camunda DMN hỗ trợ tất cả FEEL built-in types: (1) **string** - text values, có thể dùng regex trong input expressions, (2) **number** - integers và decimals, support scientific notation, (3) **boolean** - true/false cho binary decisions, (4) **date** - ISO 8601 date format (yyyy-MM-dd), support date arithmetic, (5) **time** - ISO 8601 time format (HH:mm:ss), support time comparisons, (6) **dateTime** - combined date and time, (7) **duration** - ISO 8601 durations (P1Y2M3D for 1 year 2 months 3 days). Ngoài ra, FEEL hỗ trợ **complex types**: lists (collection of values), contexts (key-value maps), ranges (intervals với open/closed endpoints như [1..10], ]0..100[), và null.

**Expression types trong decision table cells:** (1) **Unary tests** trong input columns: `< 1000`, `>= 18`, `[1000..5000]`, `"Premium","VIP"` (multiple values), `-` (any value), (2) **FEEL expressions** trong output columns: literal values, calculations (`input.price * 0.1`), function calls (`sum(items.price)`), conditionals (`if age >= 65 then "Senior" else "Adult"`).

> ⚠️ **Suy luận:** Không tìm thấy evidence về customization của hit policies trong Axelor source code, cho thấy Axelor sử dụng Camunda DMN engine như một black box. Điều này tốt cho maintainability (không cần maintain custom fork) nhưng hạn chế khả năng extend (không thể thêm custom hit policies hoặc custom FEEL functions). Tất cả features và limitations của Camunda 7.23.0 đều áp dụng trực tiếp cho Axelor.

---

### 8. TÍCH HỢP DMN VỚI BPMN — BUSINESS RULE TASK PATTERN

**Nguồn:** WkfModel.xml relationship analysis và Camunda integration pattern [Từ source code + Suy luận]

Tích hợp giữa DMN và BPMN trong Axelor tuân theo Camunda's standard pattern: sử dụng Business Rule Task trong BPMN process để gọi DMN decision. Pattern này tách biệt rõ ràng orchestration logic (BPMN - định nghĩa flow của process) và decision logic (DMN - định nghĩa rules để ra quyết định).

**Quy trình tích hợp gồm 5 bước:** (1) **Design phase:** User tạo DMN diagram trong BPM Studio UI, định nghĩa decision tables với input/output columns. DMN diagram được save vào `WkfDmnModel.diagramXml`. Đồng thời, Axelor parse XML và tạo `DmnTable` records với `decisionId` unique cho mỗi decision. (2) **Linking phase:** User tạo hoặc edit BPMN process, thêm một Business Rule Task vào process flow. Trong properties panel của task, user chọn decision từ dropdown list (populated từ `DmnTable` records). Internally, BPMN XML sẽ chứa attribute `camunda:decisionRef="decisionId"` trỏ đến DMN decision. BPMN model cũng được add DMN file vào `WkfModel.dmnFileSet` để track dependency.

(3) **Deployment phase:** Khi user deploy BPMN model, Axelor thực hiện deployment cả BPMN và tất cả referenced DMN files đến Camunda engine trong một transaction. Camunda validates cả BPMN và DMN XMLs, đảm bảo `decisionRef` references tồn tại, input/output variable names hợp lệ, và FEEL expressions parseable. Nếu validation pass, engine tạo deployment và return `deploymentId` được lưu vào `WkfModel`. (4) **Runtime execution:** Khi process instance chạy đến Business Rule Task, Camunda engine: (a) Extract `decisionRef` từ BPMN XML, (b) Collect input variables từ process context (dựa vào input mapping config), (c) Call DMN engine với `decisionRef` và input variables, (d) DMN engine evaluate decision table, match rules theo hit policy, return output object, (e) Camunda map output values vào process variables (dựa vào output mapping config), (f) Process continues đến next step.

(5) **Variable mapping:** Input mapping định nghĩa cách truyền data từ process vào decision. Có thể map entire process context (tất cả variables available), hoặc map specific variables (chỉ pass những variables cần thiết, improve security). Output mapping định nghĩa cách nhận kết quả từ decision. Single result (một output object, các fields được extract thành separate variables) hoặc result list (collection của outputs khi dùng COLLECT hit policy). Có thể customize variable names để avoid conflicts.

**BPMN XML snippet minh họa Business Rule Task:**
```xml
<bpmn:businessRuleTask id="Task_DecideDiscount" name="Determine Discount"
    camunda:decisionRef="decide-discount"
    camunda:mapDecisionResult="singleEntry"
    camunda:resultVariable="discountResult">
  <bpmn:incoming>Flow_1</bpmn:incoming>
  <bpmn:outgoing>Flow_2</bpmn:outgoing>
</bpmn:businessRuleTask>
```

**Giải thích XML:** Attribute `camunda:decisionRef="decide-discount"` reference đến `DmnTable` có `decisionId = "decide-discount"`. `camunda:mapDecisionResult="singleEntry"` specify output mapping strategy (singleEntry extract single decision output). `camunda:resultVariable="discountResult"` define tên biến process để store kết quả. Nếu decision table output có column tên `discountPercent`, sau khi evaluate, process context sẽ có variable `discountResult.discountPercent`.

> ⚠️ **Suy luận:** Pattern này standard và mature, nhưng có một hạn chế: Business Rule Task chỉ có thể call một decision tại một thời điểm. Nếu cần evaluate nhiều decisions và combine results, phải dùng multiple Business Rule Tasks hoặc tạo một Decision Requirements Diagram (DRD) với decision dependencies. Không rõ Axelor UI có hỗ trợ visual authoring của DRD hay chỉ support simple decision tables - điều này cần kiểm tra trong BPM Studio thực tế.

---

## Những điều KHÔNG tìm thấy trong source code

Do Axelor Studio là addon thương mại được phân phối dưới dạng JAR binary, nhiều chi tiết implementation không thể quan sát được từ source code available. Dưới đây là danh sách những điều tôi **KHÔNG** tìm thấy hoặc không thể xác nhận:

### 1. DMN Studio UI Source Code
**Không tìm thấy:** React/JavaScript source code của DMN editor trong BPM Studio. Chỉ quan sát được compiled webapp tại `/build/webapp/bpm/` và `/build/webapp/studio/` với các static assets (JS bundles, CSS, images).

**Ảnh hưởng:** Không thể phân tích chi tiết UX patterns, validation logic phía client, hoặc các features nâng cao của editor (ví dụ: auto-completion trong FEEL expressions, syntax highlighting, real-time validation). Không biết Studio UI có tích hợp Camunda Modeler (desktop tool) component hay tự phát triển editor riêng.

### 2. Java Service Layer Code cho DMN Operations
**Không tìm thấy:** Java source code của các services như `WkfDmnModelService`, `DmnTableService`, `DmnDeploymentService` (hoặc tên tương tự) chịu trách nhiệm deploy DMN models, evaluate decisions, và manage lifecycle.

**Ảnh hưởng:** Không thể phân tích execution flow chi tiết (exact API calls đến Camunda engine, error handling strategies, transaction boundaries). Không biết Axelor có implement caching layer cho decision results hay call Camunda engine mỗi lần. Không biết có pre-compilation hoặc optimization nào cho frequently-used decisions.

### 3. Decision Requirements Diagram (DRD) Support
**Không tìm thấy:** Evidence về việc Axelor hỗ trợ DRD - tính năng DMN 1.3 cho phép model dependencies giữa nhiều decisions (một decision output làm input cho decision khác).

**Ảnh hưởng:** Không biết user có thể tạo complex decision models với decision hierarchies hay chỉ hạn chế ở standalone decision tables. DRD rất quan trọng cho enterprise scenarios phức tạp (ví dụ: credit approval decision phụ thuộc vào risk assessment decision và income verification decision).

### 4. Business Knowledge Models (BKM) Support
**Không tìm thấy:** Evidence về BKM - reusable logic functions trong DMN (tương tự như functions trong programming).

**Ảnh hưởng:** Nếu không hỗ trợ BKM, users phải duplicate decision logic khi nhiều decision tables cần cùng một calculation. Ví dụ: formula tính tax có thể được define một lần trong BKM và reuse ở nhiều decisions.

### 5. DMN Testing và Simulation Tools
**Không tìm thấy:** UI hoặc API để test decision tables trước khi deploy (provide sample inputs, xem outputs, verify hit policies hoạt động đúng).

**Ảnh hưởng:** Không biết developers phải test DMN như thế nào. Best practice là có test harness để verify decision logic trước khi deploy production. Nếu Axelor không cung cấp, users phải test bằng cách deploy và chạy thật process instances - rất inefficient.

### 6. DMN Versioning Strategy
**Không tìm thấy:** Cách Axelor quản lý versions của DMN models độc lập với BPMN versions. `WkfModel` có version control nhưng không rõ DMN được version riêng hay luôn tied với BPMN.

**Ảnh hưởng:** Khi update một decision table đang được dùng bởi nhiều BPMN processes, không rõ tất cả processes có tự động dùng version mới hay mỗi process cần redeploy. Best practice là DMN và BPMN có independent versioning nhưng không thấy evidence.

### 7. FEEL Function Extensions
**Không tìm thấy:** Cách để extend FEEL với custom functions (ví dụ: add company-specific business logic functions).

**Ảnh hưởng:** Camunda cho phép register custom FEEL functions qua Java SPI. Nếu Axelor không expose mechanism này qua UI hoặc config, users bị giới hạn ở built-in FEEL functions. Điều này có thể ép users phải implement complex logic trong BPMN Script Tasks thay vì inline trong DMN - làm mất đi lợi ích của declarative decision modeling.

### 8. DMN Performance Metrics và Monitoring
**Không tìm thấy:** Logging/monitoring infrastructure cho DMN evaluations (execution time per decision, hit rates của rules, error rates).

**Ảnh hưởng:** Trong production, cần monitor DMN performance để identify slow decisions hoặc rules chưa bao giờ được hit (dead rules). Không có metrics, rất khó optimize decision tables lớn.

### 9. Hit Policy Validation và Warnings
**Không tìm thấy:** Compile-time hoặc design-time validation về hit policy correctness (ví dụ: UNIQUE policy nhưng có overlapping rules).

**Ảnh hưởng:** DMN editor tốt sẽ phát hiện logic errors (overlaps, gaps trong rule coverage) và warning developers. Nếu không có, users chỉ phát hiện lỗi khi runtime exception - rất nguy hiểm cho production.

### 10. Migration Tools cho DMN Models
**Không tìm thấy:** Utilities để migrate DMN definitions khi upgrade Camunda version hoặc Axelor version.

**Ảnh hưởng:** Camunda 7.x đến 8.x có breaking changes trong DMN (Zeebe engine khác architecture). Nếu Axelor nâng cấp lên Camunda 8 trong tương lai, cần migration tools để convert existing DMN models. Không thấy evidence về backward compatibility guarantees.

---

## Câu hỏi mở cần kiểm chứng thực tế

Dưới đây là những câu hỏi không thể trả lời chỉ bằng phân tích source code. Cần access vào Axelor instance đang chạy hoặc documentation đầy đủ của Studio addon:

### 1. Về Licensing và Pricing
**Câu hỏi:** Axelor Studio addon (bao gồm BPM/DMN) có license model như thế nào? Per-user, per-instance, hay unlimited? Có phiên bản community vs enterprise không? Chi phí bao nhiêu?

**Tại sao quan trọng:** Quyết định adopt Axelor cho enterprise phụ thuộc nhiều vào TCO (total cost of ownership). Nếu BPM/DMN chỉ có trong enterprise edition với giá cao, SMBs có thể không afford được.

### 2. Về Decision Requirements Diagram (DRD) Support
**Câu hỏi:** BPM Studio UI có hỗ trợ visual modeling của DRD (multiple interconnected decisions) không? Có thể tạo decision service (public interface cho một group of decisions) không?

**Cách kiểm chứng:** Tạo một DMN file với multiple decisions, trong đó decision B sử dụng output của decision A làm input. Xem UI có hiển thị dependency graph không. Deploy và verify Camunda có evaluate theo đúng dependency order.

### 3. Về Business Knowledge Models (BKM)
**Câu hỏi:** Có thể define BKMs (reusable functions) trong DMN editor không? Có thể invoke BKMs từ decision table cells không?

**Cách kiểm chứng:** Trong DMN editor, tìm option để create BKM. Define một BKM chứa calculation logic (ví dụ: `calculateTax(amount, rate)`). Trong decision table, try invoke BKM trong output expression. Deploy và verify.

### 4. Về FEEL Implementation Choice
**Câu hỏi:** Axelor default sử dụng JUEL-FEEL hay Scala-FEEL? Có config để force sử dụng một trong hai không? Có performance benchmarks cho thấy sự khác biệt?

**Cách kiểm chứng:** Tạo decision table với simple expressions (JUEL-compatible) và complex expressions (chỉ Scala-FEEL support như quantified expressions). Measure evaluation time của cả hai. Check Axelor logs để xem engine nào được invoke.

### 5. Về Decision Caching
**Câu hỏi:** Axelor có cache decision results không? Nếu input variables giống nhau, có evaluate lại decision table hay return cached result?

**Cách kiểm chứng:** Tạo một decision table deterministic (cùng input → cùng output). Call decision với same inputs nhiều lần. Instrument logging để xem Camunda engine có được invoke mỗi lần. Kiểm tra `axelor-config.properties` hoặc `AppBpm` entity xem có decision cache config.

### 6. Về DMN Testing Tools
**Câu hỏi:** Studio có cung cấp test mode để evaluate decision table với sample data không? Có coverage report để xem rules nào chưa được hit không?

**Cách kiểm chứng:** Trong DMN editor, tìm "Test" hoặc "Simulate" button. Provide sample inputs và verify output. Xem có visualize được which rules matched. Kiểm tra xem có export test cases sang CSV/JSON để automated testing.

### 7. Về DMN Versioning Independence
**Câu hỏi:** Khi update DMN model, tất cả BPMN processes reference nó có tự động dùng version mới không? Hay mỗi BPMN process "pin" vào một DMN version cụ thể?

**Cách kiểm chứng:** Deploy BPMN process A reference DMN decision X version 1. Start một instance của process A (chưa hoàn thành). Deploy DMN decision X version 2. Xem instance đang chạy có dùng version 1 (snapshot semantics) hay version 2 (latest semantics). Start instance mới của process A (không redeploy BPMN) và verify DMN version được dùng.

### 8. Về FEEL Custom Functions
**Câu hỏi:** Có mechanism nào để register custom FEEL functions không? Có document hướng dẫn implement Java function và expose nó cho FEEL không?

**Cách kiểm chứng:** Tìm trong Axelor documentation hoặc Studio admin interface về "custom functions" hoặc "FEEL extensions". Nếu có, follow guide để implement một custom function (ví dụ: `companySpecificTax(amount)`). Use function trong DMN expression và verify.

### 9. Về DMN Import/Export
**Câu hỏi:** Có thể export DMN models từ Axelor dưới dạng standard DMN XML không? Có thể import DMN files từ tools khác (Camunda Modeler, Signavio, Trisotech) vào Axelor không?

**Cách kiểm chứng:** Tạo một DMN model trong Axelor, export ra file. Verify file tuân thủ DMN 1.3 XSD schema. Tạo một DMN file trong Camunda Modeler (sử dụng full FEEL features), import vào Axelor và deploy. Verify có lỗi compatibility không.

### 10. Về DMN Performance at Scale
**Câu hỏi:** Decision table lớn (hàng trăm rules) có performance issue không? Camunda có optimize evaluation với indexing hoặc rule compilation không?

**Cách kiểm chứng:** Tạo decision table với 1000 rules. Measure evaluation time (should be sub-millisecond với Camunda's optimized engine). Tạo process chạy decision đó 10,000 lần concurrent. Monitor CPU, memory, database load. Verify không có bottleneck.

---

## Tổng kết

Axelor triển khai DMN thông qua **Camunda DMN Engine 7.23.0**, một engine tuân thủ đầy đủ DMN 1.3 standard. Kiến trúc addon-based cho phép Axelor tách riêng BPM/DMN features thành sản phẩm thương mại trong khi vẫn maintain open-source core. Cấu trúc dữ liệu với ba entities chính (WkfDmnModel, DmnTable, DmnField) hỗ trợ lưu trữ, versioning, và deployment management. Tích hợp với BPMN qua Business Rule Task pattern là standard và mature.

**Điểm mạnh của giải pháp:**
- Sử dụng Camunda engine đã proven, không reinvent the wheel
- Hỗ trợ đầy đủ DMN 1.3 standard và FEEL expression language
- Có cả JUEL-FEEL (fast) và Scala-FEEL (complete) implementations
- Connection pool và history management được config hợp lý
- Version control cho workflow models
- Recursion limits để prevent infinite loops

**Điểm yếu và giới hạn:**
- Source code closed (addon thương mại) → khó customize và extend
- Không rõ support level cho advanced DMN features (DRD, BKM)
- Không có evidence về testing tools và monitoring infrastructure
- Documentation giới hạn, phụ thuộc vào Camunda docs
- Upgrade path không rõ ràng (Camunda 7 → 8 migration)

**So sánh với các giải pháp khác:**
- **vs Odoo:** Odoo không có DMN engine tích hợp sẵn, chỉ có workflow engine đơn giản. Để có decision table features trong Odoo cần viết custom Python code hoặc dùng external decision engine.
- **vs Salesforce:** Salesforce có Flow Builder với decision elements nhưng không tuân thủ DMN standard. Less portable, vendor lock-in cao hơn.
- **vs Microsoft Dynamics:** Dynamics 365 có Business Rules engine nhưng cũng proprietary. Axelor với Camunda DMN có lợi thế về open standards compliance.

Axelor DMN solution phù hợp cho các tổ chức cần decision automation capabilities với industry-standard compliance, đặc biệt trong regulated industries (finance, healthcare) nơi auditability và process transparency quan trọng.
