# BƯỚC 1: KHÁM PHÁ CẤU TRÚC TỔNG QUAN - Phân Tích Từ Source Code

## Phương pháp phân tích

Quá trình nghiên cứu được thực hiện bằng cách đọc trực tiếp các file cấu hình và mã nguồn tại thư mục `/Volumes/works/code/java/axelor/axelor-erp`. Phạm vi phân tích bao gồm các files và thư mục quan trọng sau:

**Files cấu hình chính:**
- `/settings.gradle` - File cấu hình cơ chế tải modules động của Gradle
- `/build.gradle` - File build gốc chứa thông tin project và cấu hình Java toolchain
- `/gradle.properties` - Thiết lập JVM và các tham số build
- `/src/main/resources/axelor-config.properties` - File cấu hình ứng dụng chính (518 dòng)

**Dependencies và versions:**
- `/modules/axelor-open-suite/libs.gradle` - Khai báo các thư viện bên ngoài
- `/modules/axelor-open-suite/version.gradle` - Quản lý phiên bản
- `/modules/axelor-open-suite/version.txt` - Số phiên bản cụ thể

**Module build configs:**
- `/modules/axelor-open-suite/axelor-base/build.gradle`
- `/modules/axelor-open-suite/axelor-sale/build.gradle`
- `/modules/axelor-open-suite/axelor-account/build.gradle`

**Sample source code:**
- Domain XML files mẫu: `Address.xml`, `Company.xml` để hiểu cách định nghĩa entity
- View XML files mẫu: `Address.xml` (views) để hiểu cách xây dựng giao diện

---

## Kết quả chi tiết

### 1. CƠ CHẾ TẢI MODULES ĐỘNG

**File nguồn:** `/settings.gradle` [Từ source code]

Axelor ERP sử dụng một cơ chế đặc biệt để quản lý modules - thay vì liệt kê cứng (hardcode) từng module trong file cấu hình, hệ thống tự động quét và phát hiện tất cả các modules có trong thư mục `modules/`. Cơ chế này được thực hiện thông qua Gradle plugin tùy chỉnh `com.axelor.app` phiên bản 7.4.7. Khi Gradle chạy, nó sẽ duyệt qua tất cả các thư mục con cấp một (maxDepth: 1) bên trong thư mục `modules/`, và nếu thư mục nào chứa file `build.gradle`, nó sẽ được nhận diện là một module hợp lệ và được thêm vào danh sách build.

Cách tiếp cận này mang lại lợi ích lớn về mặt khả năng mở rộng (scalability) - khi developer muốn thêm một module mới, họ chỉ cần tạo thư mục mới với file `build.gradle` bên trong thư mục `modules/`, không cần phải sửa đổi file cấu hình gốc. Tuy nhiên, đây cũng là một con dao hai lưỡi: nếu một thư mục có file `build.gradle` nhưng chưa sẵn sàng để build (ví dụ: đang trong quá trình phát triển), nó vẫn sẽ được tự động include và có thể gây lỗi build. Hệ thống cũng cấu hình hai repository chính: Maven Central (với điều kiện loại trừ group 'com.axelor' để tránh xung đột) và Axelor Nexus repository tại `https://repository.axelor.com/nexus/repository/maven-public/` để tải các thư viện chuyên biệt của Axelor.

**Bằng chứng từ code:**
```groovy
def modules = []
file("modules").traverse(type: groovy.io.FileType.DIRECTORIES, maxDepth: 1) { it ->
  if (new File(it, "build.gradle").exists()) {
    modules.add(it)
  }
}

gradle.ext.appModules = modules

modules.each { dir ->
  include "modules:$dir.name"
  project(":modules:$dir.name").projectDir = dir
}
```

**Giải thích code:** Đoạn code trên sử dụng phương thức `traverse()` của Groovy để duyệt qua các thư mục. Tham số `maxDepth: 1` đảm bảo chỉ quét cấp một, không đệ quy xuống các thư mục con. Biến `modules` thu thập danh sách các thư mục hợp lệ, sau đó được lưu vào `gradle.ext.appModules` để các phần khác của build script có thể truy cập. Vòng lặp `each` cuối cùng thực hiện việc include từng module vào project với pattern naming `modules:tên_thư_mục` và map vị trí thực tế của module qua thuộc tính `projectDir`.

---

### 2. THÔNG TIN PROJECT GỐC VÀ JAVA TOOLCHAIN

**File nguồn:** `/build.gradle` [Từ source code]

File build gốc tiết lộ những thông tin quan trọng về project: tên chính thức là "Axelor ERP", đang ở phiên bản 8.5.10, và thuộc group `com.axelor.apps`. Một điểm đáng chú ý là có sự chênh lệch phiên bản giữa project gốc (8.5.10) và submodule axelor-open-suite bên trong (8.5.9 theo file version.txt), điều này cho thấy project gốc đang ở một phiên bản phát triển mới hơn submodule.

Quyết định công nghệ quan trọng nhất được thể hiện qua Java toolchain: project sử dụng Java 21, phiên bản Long-Term Support (LTS) mới nhất của Java tại thời điểm này. Việc sử dụng Java 21 mang lại nhiều lợi thế: hiệu năng (performance) được cải thiện đáng kể nhờ các tối ưu hóa trong JVM, hỗ trợ pattern matching và record classes giúp code ngắn gọn hơn, và đặc biệt là virtual threads (Project Loom) cho phép xử lý hàng triệu luồng (threads) đồng thời mà không tiêu tốn quá nhiều tài nguyên hệ thống. Tuy nhiên, điều này cũng đặt ra yêu cầu về môi trường: mọi server triển khai (deployment) phải có JDK 21 hoặc cao hơn, điều có thể gây khó khăn cho các tổ chức vẫn đang dùng Java 8 hoặc 11.

Project cũng định nghĩa nhiều Gradle tasks tùy chỉnh phục vụ cho quy trình đảm bảo chất lượng (quality assurance): `checkCsvEOL` kiểm tra ký tự cuối dòng trong file CSV (quan trọng khi import/export dữ liệu giữa các hệ điều hành khác nhau), `checkBirtCredentials` đảm bảo không có thông tin đăng nhập nhạy cảm bị lưu trực tiếp trong file báo cáo BIRT, và các tasks validate XML (`checkXmlDomainsAttributes`, `checkXmlViewsAttributes`, `checkXmlImportsAttributes`) để phát hiện lỗi cú pháp hoặc thuộc tính không hợp lệ trước khi chạy ứng dụng.

**Bằng chứng từ code:**
```groovy
allprojects {
  group = 'com.axelor.apps'
  version = '8.5.10'

  java {
    toolchain {
      languageVersion = JavaLanguageVersion.of(21)
    }
  }
}
```

**Giải thích code:** Block `allprojects` áp dụng cấu hình cho tất cả các projects (bao gồm cả root project và subprojects/modules). Việc khai báo `toolchain` thay vì chỉ đơn giản set `sourceCompatibility` có ý nghĩa quan trọng: Gradle sẽ tự động tải xuống (download) JDK 21 nếu máy build chưa có, đảm bảo môi trường build nhất quán (consistent) trên mọi máy developer. Thuộc tính `languageVersion` xác định version Java mà source code sử dụng, ảnh hưởng đến cả quá trình biên dịch (compilation) và kiểm tra lỗi (error checking) của IDE.

---

### 3. CẤU HÌNH JVM VÀ BUILD PERFORMANCE

**File nguồn:** `/gradle.properties` [Từ source code]

File `gradle.properties` chứa các thiết lập quan trọng cho quá trình build và hiệu năng. Đầu tiên, `org.gradle.parallel=true` cho phép Gradle build nhiều modules đồng thời (parallel builds), thay vì xử lý tuần tự. Điều này quan trọng với một project có 27 modules như Axelor - thời gian build có thể giảm từ vài phút xuống còn dưới một phút trên máy đủ mạnh với CPU nhiều lõi (multi-core). Tuy nhiên, parallel builds đòi hỏi nhiều bộ nhớ RAM hơn vì mỗi module đang build cùng lúc đều cần không gian riêng.

Thiết lập `org.gradle.jvmargs=-Xmx2g` cấp phát 2GB heap memory cho JVM chạy Gradle daemon. Con số 2GB là một lựa chọn cân bằng: đủ lớn để xử lý việc biên dịch (compilation) và sinh mã tự động (code generation) của Axelor mà không bị OutOfMemoryError, nhưng không quá lớn đến mức chiếm dụng quá nhiều RAM của máy developer. Trong thực tế, nếu project còn phát triển thêm nhiều modules hoặc có domain XML phức tạp hơn, có thể cần tăng lên 3-4GB. Đường dẫn Java home được hard-code trỏ đến `/usr/local/Cellar/openjdk@21/21.0.9/` - đây là convention của Homebrew trên macOS, cho thấy môi trường phát triển (development environment) đang được thiết lập trên macOS.

**Bằng chứng từ code:**
```properties
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx2g
org.gradle.java.home=/usr/local/Cellar/openjdk@21/21.0.9/libexec/openjdk.jdk/Contents/Home
```

**Giải thích code:** Ba dòng cấu hình này làm việc cùng nhau để tối ưu hóa quá trình build. Dòng đầu tiên bật chế độ song song, hai dòng sau đảm bảo có đủ tài nguyên và đúng JDK version. Đường dẫn `java.home` ghi đè (override) biến môi trường `JAVA_HOME` của hệ thống, đảm bảo Gradle luôn dùng đúng JDK 21 ngay cả khi developer có nhiều version Java cài đặt.

---

### 4. DANH SÁCH MODULES TRONG AXELOR OPEN SUITE

**File nguồn:** Directory listing `/modules/axelor-open-suite/` [Từ source code]

Cấu trúc project sử dụng mô hình git submodule: project gốc `axelor-erp` (phiên bản 8.5.10) chứa một submodule tên `axelor-open-suite` (phiên bản 8.5.9) ở bên trong thư mục `modules/`. Pattern này cho phép quản lý vòng đời (lifecycle) của business modules độc lập với project wrapper bên ngoài - ví dụ, một công ty có thể fork axelor-open-suite để tùy chỉnh business logic nhưng vẫn giữ nguyên project gốc để dễ dàng upgrade.

Bên trong axelor-open-suite có tổng cộng **27 business modules**, mỗi module phụ trách một mảng nghiệp vụ cụ thể. Danh sách đầy đủ bao gồm:

1. **axelor-base** - Module nền tảng (foundation module) quan trọng nhất, chứa các entities cơ bản như Partner (đối tác), Product (sản phẩm), Company (công ty), User (người dùng). Hầu hết các modules khác đều phụ thuộc vào base.

2. **axelor-account** - Module kế toán (accounting), quản lý Invoice (hóa đơn), Account (tài khoản kế toán), Tax (thuế), Payment (thanh toán). Đây là module lớn thứ hai với 122 domain entities.

3. **axelor-sale** - Module bán hàng, quản lý SaleOrder (đơn hàng), Quotation (báo giá). Phụ thuộc vào axelor-crm.

4. **axelor-crm** - Quản lý quan hệ khách hàng, bao gồm Lead (khách hàng tiềm năng), Opportunity (cơ hội bán hàng), Contact (liên hệ).

5. **axelor-purchase** - Quản lý mua hàng, gồm PurchaseOrder (đơn mua), Supplier (nhà cung cấp).

6. **axelor-stock** - Quản lý kho hàng (inventory), bao gồm Stock (hàng tồn), Location (vị trí kho), Movement (xuất nhập kho).

7. **axelor-supplychain** - Chuỗi cung ứng, tích hợp giữa sale, purchase và stock. Đây là module integration layer.

8. **axelor-production** - Sản xuất, bao gồm BOM (Bill of Materials), WorkCenter (trung tâm sản xuất).

9. **axelor-human-resource** - Nhân sự, quản lý Employee (nhân viên), Leave (nghỉ phép), Expense (chi phí).

10. **axelor-project** - Quản lý dự án, gồm Task (công việc), TimeSheet (chấm công).

11. **axelor-contract** - Quản lý hợp đồng.

12. **axelor-bank-payment** - Thanh toán ngân hàng, hỗ trợ SEPA và Bank Statement.

13. **axelor-budget** - Lập kế hoạch và kiểm soát ngân sách.

14. **axelor-cash-management** - Quản lý dòng tiền (cash flow).

15. **axelor-fleet** - Quản lý đội xe (fleet management).

16. **axelor-helpdesk** - Hệ thống ticketing cho hỗ trợ khách hàng.

17. **axelor-intervention** - Quản lý can thiệp/sửa chữa.

18. **axelor-maintenance** - Quản lý bảo trì.

19. **axelor-marketing** - Chiến dịch marketing, email marketing.

20. **axelor-quality** - Quản lý chất lượng (quality management).

21. **axelor-talent** - Tuyển dụng và phát triển nhân tài.

22. **axelor-gdpr** - Tuân thủ quy định GDPR về bảo vệ dữ liệu cá nhân.

23. **axelor-mobile-settings** - Cấu hình cho ứng dụng mobile.

24. **axelor-client-portal** - Cổng thông tin cho khách hàng.

25. **axelor-supplier-portal** - Cổng thông tin cho nhà cung cấp.

26. **axelor-supplier-management** - Đánh giá và quản lý nhà cung cấp.

27. **changelogs/** - Thư mục chứa changelog files, không phải module.

**Kiến trúc phụ thuộc:** Modules được tổ chức theo dạng cây phân cấp (hierarchical dependency tree): axelor-base ở gốc, các modules domain-specific như account, crm, stock phụ thuộc trực tiếp vào base, và các modules integration như supplychain phụ thuộc vào nhiều modules khác cùng lúc. Mô hình này đảm bảo rằng business logic được tổ chức modular và có thể tái sử dụng (reusable).

---

### 5. EXTERNAL DEPENDENCIES VÀ ADDONS

**File nguồn:** `/modules/axelor-open-suite/libs.gradle` [Từ source code]

File này khai báo tất cả các thư viện bên ngoài mà Axelor Open Suite phụ thuộc vào. Điểm đặc biệt đáng chú ý là Axelor sử dụng một kiến trúc addon: ba thành phần quan trọng (axelor-studio, axelor-message, axelor-utils) không phải là module trong source tree mà được tải xuống như các binary dependencies từ Axelor repository.

**axelor-studio:3.5.1** là addon cực kỳ quan trọng - đây là nền tảng (platform) low-code/no-code cho phép người dùng tạo models, views, và workflows qua giao diện đồ họa mà không cần viết code. Addon này bao gồm cả BPM engine (có thể là Camunda dựa trên các dấu hiệu trong config file). Việc Studio là binary addon thay vì open source có ý nghĩa thương mại: Axelor có thể kiểm soát licensing và monetization của phần no-code platform này.

**axelor-message:3.3.0** cung cấp messaging system - có thể là email integration, in-app messaging, hoặc notification system. **axelor-utils:3.5.0** chứa các utility functions dùng chung.

Về thư viện bên ngoài, danh sách cho thấy Axelor là một platform đầy đủ tính năng:
- **Groovy 3.0.23** - Ngôn ngữ scripting cho phép viết business logic động (dynamic) mà không cần biên dịch lại toàn bộ ứng dụng. Groovy có cú pháp (syntax) gần giống Java nhưng ngắn gọn hơn và hỗ trợ closures, meta-programming.
- **PDF processing** (pdfbox 3.0.0, openpdf 1.4.2) - Tạo và xử lý file PDF cho báo cáo, hóa đơn.
- **Security libraries** (bouncycastle 1.78.1) - Mã hóa (encryption) và chữ ký số (digital signature), đặc biệt quan trọng cho electronic invoicing.
- **Calendar** (ical4j 2.2.0) - Xử lý file iCalendar cho tích hợp với Google Calendar, Outlook.
- **IBAN validation** (iban4j) - Kiểm tra số tài khoản ngân hàng quốc tế, quan trọng cho module bank-payment.
- **Barcode** (zxing 3.5.0) - Tạo và đọc mã vạch (barcode) và QR code.
- **WebDAV** (jackrabbit-webdav) - Giao thức (protocol) để truy cập file qua web, có thể dùng cho document management.
- **OAuth** (google-oauth-client-jetty) - Đăng nhập bằng Google account.
- **SSO** (pac4j-core 5.7.7) - Framework cho Single Sign-On, hỗ trợ nhiều protocols: OAuth, SAML, CAS, OpenID Connect.
- **SOAP** (groovy-wslite) - Gọi web services kiểu cũ (legacy).
- **OpenAPI** (swagger-jaxrs2) - Tự động generate API documentation.

**Bằng chứng từ code:**
```groovy
libs.axelor_studio = 'com.axelor.addons:axelor-studio:3.5.1'
libs.axelor_message = 'com.axelor.addons:axelor-message:3.3.0'
libs.axelor_utils = 'com.axelor.addons:axelor-utils:3.5.0'
libs.groovy = 'org.codehaus.groovy:groovy-all:3.0.23'
libs.pac4j_core = "org.pac4j:pac4j-core:5.7.7"
```

**Giải thích code:** Cú pháp `libs.tên_biến = 'dependency'` là Gradle 7.x+ convention, định nghĩa version catalog để quản lý tập trung (centralized) phiên bản thư viện. Dòng `axelor_studio` cho thấy group là `com.axelor.addons`, artifact là `axelor-studio`, version là `3.5.1`. Tất cả các modules khác có thể reference `libs.axelor_studio` thay vì hard-code dependency string, giúp dễ dàng nâng cấp (upgrade) version sau này.

---

### 6. CẤU TRÚC AXELOR-BASE - MODULE NỀN TẢNG

**File nguồn:** `/modules/axelor-open-suite/axelor-base/build.gradle` [Từ source code]

axelor-base là module quan trọng và phức tạp nhất trong toàn bộ hệ thống, đóng vai trò như một lớp nền tảng (foundation layer) mà tất cả modules khác đều xây dựng dựa trên nó. File build.gradle của module này tiết lộ một điểm thú vị: bên cạnh backend Java, module còn có một frontend component riêng biệt được viết bằng **React 19.1** nằm trong thư mục `map-viewer/`.

Thành phần React này sử dụng **Node.js 22.17.1** và **Yarn 1.22.19** để quản lý dependencies và build. Gradle được cấu hình để tự động tải xuống (download) đúng phiên bản Node.js (thông qua `download = true`), sau đó chạy `yarn install` để cài đặt npm packages, rồi `yarn run build` để compile React app thành static assets (JavaScript bundles, CSS). Cuối cùng, output của React build được copy vào thư mục `webapp/base/map-viewer/` và đóng gói (packaging) chung vào file JAR của module. Cách tiếp cận này có nghĩa React app được deploy như một phần của backend JAR, không cần serve riêng từ Node.js server.

Tại sao lại có React trong một dự án Java? Có thể đây là một widget đặc biệt cho chức năng xem bản đồ (map viewer) - hiển thị địa chỉ khách hàng, warehouse locations trên Google Maps hoặc OpenStreetMap. Việc dùng React cho widget phức tạp như map có ý nghĩa vì React ecosystem có nhiều thư viện map tốt (React Leaflet, React Google Maps) và performance của Virtual DOM phù hợp với việc render hàng nghìn markers.

**Bằng chứng từ code (Frontend build pipeline):**
```groovy
node {
  version = '22.17.1'
  yarnVersion = '1.22.19'
  download = true
  nodeModulesDir = file('map-viewer')
}

task buildFront(type: YarnTask) {
  dependsOn 'installFrontDeps'
  args = ["run", "build"]
}

jar {
  dependsOn 'buildFront'
  into(reactOutputDir) {
    from "${reactDir}/dist"
  }
}
```

**Giải thích code:** Block `node` cấu hình Gradle Node Plugin. Thuộc tính `download = true` quan trọng - nó cho phép Gradle tự động tải Node.js nếu developer chưa cài, đảm bảo môi trường build nhất quán (consistent) trên mọi máy. `nodeModulesDir` chỉ định nơi chạy `yarn install`. Task `buildFront` phụ thuộc vào `installFrontDeps` (cài dependencies trước), sau đó chạy yarn build. Task `jar` cuối cùng phụ thuộc vào `buildFront`, nghĩa là mỗi lần build JAR, React app đều được build lại và copy vào JAR.

Về cấu trúc thư mục, axelor-base chứa **189 domain XML files** (định nghĩa entities) và **192 view XML files** (định nghĩa giao diện người dùng). Đây là con số ấn tượng cho thấy độ phức tạp của module - nó không chỉ là "base" đơn giản mà là một business application hoàn chỉnh với đầy đủ entities và UI. Cấu trúc resources directory tuân theo convention rõ ràng:

```
src/main/resources/
├── apps/                # Application-level configs
├── data-export/         # Export templates (CSV, Excel)
├── data-import/         # Import configs và mapping rules
├── data-init/           # Initial data (seed data) - CSV files
├── domains/             # *** 189 domain XML files ***
├── i18n/                # Translation files (properties)
├── import-configs/      # Cấu hình cho data import
├── reports/             # BIRT report definitions (.rptdesign)
└── views/               # *** 192 view XML files ***
```

Thư mục `data-init/` chứa seed data - dữ liệu khởi tạo (initial data) dưới dạng CSV files được import tự động khi cài đặt lần đầu. Thư mục `i18n/` chứa translation files, cho phép ứng dụng đa ngôn ngữ (internationalization). Thư mục `reports/` chứa BIRT report templates - BIRT (Business Intelligence and Reporting Tools) là một framework Eclipse cho việc tạo báo cáo phức tạp.

---

### 7. AXELOR-SALE VÀ AXELOR-ACCOUNT - HAI MODULE NGHIỆP VỤ CHÍNH

**File nguồn:** `/modules/axelor-open-suite/axelor-sale/build.gradle`, `/modules/axelor-open-suite/axelor-account/build.gradle` [Từ source code]

**axelor-sale** là module quản lý bán hàng, có một dependency duy nhất: `api project(":modules:axelor-crm")`. Pattern này cho thấy kiến trúc phân lớp rõ ràng - Sale không phụ thuộc trực tiếp vào base (mặc dù vẫn nhận được base thông qua transitive dependency từ CRM), mà phụ thuộc vào CRM. Điều này hợp lý vì quy trình bán hàng thường bắt đầu từ CRM: Lead → Opportunity → Quotation → Sale Order. Module này chứa 31 domain entities và 30 view files - nhỏ gọn so với base nhưng tập trung vào business logic bán hàng.

**axelor-account** là module kế toán, phức tạp hơn nhiều với **122 domain entities** và **120 view files**, khiến nó trở thành module lớn thứ hai sau base. Dependencies của account module tiết lộ nhiều điều thú vị về chức năng:
- **jdom, xalan, xmlbeans** - Ba thư viện XML processing này có thể dùng cho việc generate và parse file XML của electronic invoicing (hóa đơn điện tử), đặc biệt là các chuẩn như UBL (Universal Business Language), ebXML.
- **bouncycastle (bcprov, bcpkix)** - Provider mã hóa (cryptographic provider) mạnh mẽ, có thể dùng để ký số (digital signature) hóa đơn điện tử, yêu cầu bắt buộc ở nhiều quốc gia như Việt Nam, Italy, Brazil.
- **ical4j** - Tích hợp calendar, có thể dùng cho việc lên lịch thanh toán, nhắc nhở đến hạn hóa đơn.
- **iban4j** - Validate số tài khoản ngân hàng theo chuẩn IBAN (International Bank Account Number), quan trọng cho thanh toán quốc tế và SEPA trong EU.

**Bằng chứng từ code:**
```groovy
dependencies {
  api project(":modules:axelor-base")

  implementation libs.jdom
  implementation libs.xalan
  implementation libs.xmlbeans

  implementation libs.bcprov_jdk18on
  implementation libs.bcpkix_jdk18on

  implementation libs.ical4j
  implementation libs.iban4j
}
```

**Giải thích code:** Từ khóa `api` trong `api project(":modules:axelor-base")` có ý nghĩa: dependencies của account sẽ expose base module cho các modules khác depend vào account (API dependency leaking). Ngược lại, `implementation` chỉ dùng internally - các module phụ thuộc vào account sẽ KHÔNG tự động nhận được jdom, xalan, etc. Đây là best practice để giảm coupling và tránh dependency hell.

Module account cũng có thư mục đặc biệt `l10n/` (localization) chứa dữ liệu theo từng quốc gia như chart of accounts (bảng tài khoản kế toán), tax configurations, đây là requirement thiết yếu cho một accounting system quốc tế vì mỗi nước có quy định kế toán và thuế khác nhau.

---

### 8. DOMAIN MODELS - CƠ CHẾ ĐỊNH NGHĨA ENTITY BẰNG XML

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Address.xml`, `Company.xml` [Từ source code]

Axelor sử dụng một approach độc đáo: thay vì định nghĩa JPA entities trực tiếp bằng Java annotations (như phần lớn các Java frameworks khác), Axelor dùng **Domain XML files** như source of truth. Đây là một ví dụ về Model-Driven Development (MDD) - developers định nghĩa models ở mức abstraction cao (XML schema), sau đó Axelor code generator tự động sinh ra Java entity classes với đầy đủ JPA annotations, getters/setters, equals/hashCode, toString.

Domain XML tuân theo một schema chuẩn (`domain-models_7.4.xsd`) với namespace `http://axelor.com/xml/ns/domain-models`. Mỗi file XML có thể chứa nhiều entity definitions, và mỗi entity có thể có nhiều loại fields: `<string>`, `<integer>`, `<decimal>`, `<boolean>`, `<date>`, `<datetime>`, `<many-to-one>` (foreign key), `<one-to-many>` (reverse relationship), `<many-to-many>`.

Điểm đặc biệt của Axelor Domain XML là các attributes bổ sung (extended attributes) cung cấp metadata phong phú hơn nhiều so với JPA thuần túy:
- **namecolumn="true"** - Đánh dấu field này là display name của entity, dùng khi hiển thị trong dropdowns, references.
- **search="field1,field2,field3"** - Danh sách các fields được dùng cho full-text search khi người dùng gõ vào search box.
- **massUpdate="true"** - Cho phép bulk update field này cho nhiều records cùng lúc.
- **cacheable="true"** - Bật entity-level caching, các instances của entity này sẽ được cache trong L2 cache (second-level cache) của Hibernate.
- **readonly="true"** - Field chỉ đọc, không thể update sau khi tạo.
- **track** - Audit trail feature, tự động ghi lại lịch sử thay đổi của field, quan trọng cho compliance và debugging.
- **showIf/hideIf** - Conditional rendering, field chỉ hiện khi điều kiện được thỏa mãn.
- **extra-code** - Cho phép nhúng Java code trực tiếp trong XML bằng CDATA section, thường dùng để định nghĩa constants.

**Bằng chứng từ code (Address entity):**
```xml
<entity name="Address">
  <many-to-one name="country" ref="com.axelor.apps.base.db.Country"
    title="Country" required="true"/>

  <string name="streetName" title="Street"
    help="Name of a street or thoroughfare..."/>

  <many-to-one name="city" ref="City" title="City" massUpdate="true"/>

  <decimal name="latit" title="Latitude" precision="38" scale="18" nullable="true"/>
  <decimal name="longit" title="Longitude" precision="38" scale="18" nullable="true"/>

  <string name="fullName" namecolumn="true"
    search="addressL2,addressL3,addressL4,addressL5,addressL6" title="Address"/>
</entity>
```

**Giải thích code:** Entity Address này minh họa nhiều concepts quan trọng. Field `country` là many-to-one relationship với `required="true"`, nghĩa là mỗi address phải thuộc về một country (địa chỉ không thể tồn tại mà không có quốc gia). Field `streetName` có attribute `help` - văn bản này sẽ hiển thị như tooltip khi user hover chuột, giúp hướng dẫn người dùng. Field `city` có `massUpdate="true"` - use case là khi merge hai cities, admin có thể chọn nhiều addresses và update city cho tất cả cùng lúc. Fields `latit` và `longit` dùng precision cao (38,18) để lưu tọa độ GPS chính xác. Field `fullName` được đánh dấu `namecolumn="true"` nên khi hiển thị Address trong dropdown (ví dụ: chọn shipping address), sẽ show fullName thay vì ID. Attribute `search` liệt kê các fields sẽ được search khi user gõ vào search box - điều này cho phép tìm địa chỉ theo nhiều trường khác nhau.

**Bằng chứng từ code (Company entity với caching và tracking):**
```xml
<entity name="Company" cacheable="true">
  <string name="name" title="Name" required="true" unique="true"/>
  <string name="code" title="Code" required="true" unique="true"/>

  <many-to-one name="currency" ref="com.axelor.apps.base.db.Currency"
    title="Currency" massUpdate="true"/>

  <one-to-many name="bankDetailsList" ref="com.axelor.apps.base.db.BankDetails"
    title="Bank accounts" mappedBy="company"/>

  <extra-code><![CDATA[
    // CATEGORY SELECT
    public static final int CATEGORY_CUSTOMER = 1;
    public static final int CATEGORY_SUPPLIER = 2;
  ]]></extra-code>

  <track>
    <field name="name" on="UPDATE"/>
    <field name="code" on="UPDATE"/>
    <field name="currency" on="UPDATE"/>
  </track>
</entity>
```

**Giải thích code:** Company entity có `cacheable="true"` ở entity level - điều này có ý nghĩa lớn về performance. Vì Company records thường được reference rất nhiều (mỗi invoice, sale order, purchase order đều có company), việc cache chúng trong L2 cache giúp giảm đáng kể số lượng truy vấn database. Hai fields `name` và `code` đều có `unique="true"` - Hibernate sẽ tạo unique constraint trong database để đảm bảo không có hai companies trùng tên hoặc code. Field `bankDetailsList` là one-to-many relationship với `mappedBy="company"` - đây là reverse side của relationship, nghĩa là BankDetails entity sẽ có field `company` là foreign key.

Block `<extra-code>` cho phép nhúng Java constants trực tiếp vào generated entity class. Kỹ thuật này rất hữu ích để avoid magic numbers - thay vì dùng `company.setCategory(1)`, code có thể dùng `company.setCategory(Company.CATEGORY_CUSTOMER)`, dễ đọc và ít lỗi hơn.

Block `<track>` định nghĩa audit trail: mỗi khi UPDATE (không phải CREATE) một Company và các fields name, code, hoặc currency thay đổi, hệ thống sẽ tự động ghi lại: ai thay đổi, thay đổi lúc nào, giá trị cũ là gì, giá trị mới là gì. Feature này cực kỳ quan trọng cho compliance và forensics khi có incident.

Tại sao Axelor chọn XML thay vì Java annotations? Có nhiều lý do: (1) XML có thể được parse và validate ở build time bằng XSD schema, phát hiện lỗi sớm hơn. (2) Code generator có thể đọc XML dễ dàng hơn parse Java annotations. (3) Business analysts không biết Java vẫn có thể đọc và review XML models. (4) XML cho phép extends và overrides dễ dàng qua XPath (sẽ thấy ở phần view extensions). Tuy nhiên, nhược điểm là XML verbose hơn và không có type safety lúc viết (IDE không autocomplete được như Java).

---

### 9. VIEW DEFINITIONS - XÂY DỰNG GIAO DIỆN BẰNG XML

**File nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/views/Address.xml` [Từ source code]

Tương tự như domain models, giao diện người dùng (UI) trong Axelor cũng được định nghĩa hoàn toàn bằng XML thay vì HTML/JSP/Thymeleaf. Axelor cung cấp một DSL (Domain-Specific Language) dạng XML cho việc build UI, với các view types khác nhau phù hợp với từng màn hình nghiệp vụ:

**View types chính:**
- **`<grid>`** - List view hoặc table view, hiển thị nhiều records trong một bảng với columns, sorting, filtering. Đây là view đầu tiên người dùng thấy khi click vào menu item.
- **`<form>`** - Detail view hoặc form view, hiển thị chi tiết một record duy nhất với đầy đủ fields, organized trong panels, tabs. Khi user click vào một row trong grid, form view sẽ mở.
- **`<panel-include>`** - Include và reuse panels từ views khác, giúp tránh duplication.

Axelor view system cung cấp nhiều features phức tạp thông qua XML attributes:

**Action lifecycle hooks:**
- `onLoad` - Execute actions khi view được load lần đầu, thường dùng để set default values hoặc load related data.
- `onSave` - Execute trước khi save record, thường dùng cho validation hoặc data transformation.
- `onNew` - Execute khi user click nút "New" để tạo record mới, dùng để initialize fields.
- `onCopy` - Execute khi user duplicate một record, có thể clear hoặc modify một số fields.
- `onChange` - Execute khi một field thay đổi, dùng cho real-time computation hoặc cascading updates.

**Action types:**
Axelor định nghĩa nhiều loại actions, mỗi loại phục vụ một mục đích khác nhau:
- `action-group-*` - Nhóm nhiều actions lại và execute theo thứ tự (chaining).
- `action-*-method-*` - Call một Java method trong controller class, dùng khi cần business logic phức tạp không thể express bằng XML.
- `action-*-record-*` - Thao tác với record: set field values, clear fields, copy values. Dùng cho logic đơn giản như "khi chọn customer, auto-fill address".
- `action-*-attrs-*` - Update UI attributes: show/hide fields, make readonly, change required status, update title/css class. Dùng cho conditional UI logic.
- `action-*-validate-*` - Validation rules, show error/warning/info messages khi điều kiện không thỏa mãn.

**UI control features:**
- **Conditional rendering:** `showIf`, `hideIf`, `readonlyIf` - Fields có thể tự động ẩn/hiện hoặc readonly dựa trên expressions. Ví dụ: `showIf="status == 'confirmed'"` chỉ hiển thị field khi order đã confirm.
- **Layout system:** `colSpan` - Axelor dùng 12-column grid system (giống Bootstrap), mỗi field có thể span nhiều columns.
- **Custom toolbar:** `<toolbar>` cho phép thêm custom buttons vào grid view, ví dụ "Bulk Import", "Generate Report".
- **Client-side binding:** `x-bind` - Two-way data binding với transformations, ví dụ `{{fieldName|uppercase}}` tự động uppercase input.
- **Permissions:** `canNew`, `canEdit`, `canRemove` - Control xem user có quyền create/edit/delete records hay không.

**Bằng chứng từ code (Grid view với custom toolbar):**
```xml
<grid name="address-grid" title="Address list" model="com.axelor.apps.base.db.Address">
  <toolbar>
    <button name="checkDuplicateBtn" title="Check Duplicate"
      onClick="action-base-method-show-duplicate"/>
  </toolbar>
  <field name="fullName"/>
  <button name="mapBtn" icon="fa-map-marker"
    onClick="action-base-address-method-view-map"/>
</grid>
```

**Giải thích code:** Grid view này hiển thị danh sách addresses. Attribute `model` chỉ định entity class đầy đủ package name. Block `<toolbar>` thêm một button "Check Duplicate" vào toolbar phía trên grid - khi click, nó gọi action `action-base-method-show-duplicate` (likely một Java method để find duplicate addresses based on some criteria). Field `fullName` được hiển thị trong grid. Một button đặc biệt `mapBtn` với icon Font Awesome `fa-map-marker` xuất hiện trên mỗi row, khi click sẽ mở map view để show address location.

**Bằng chứng từ code (Form view với conditional logic và lifecycle hooks):**
```xml
<form name="address-form" title="Address" model="com.axelor.apps.base.db.Address"
  width="large" onLoad="action-group-base-address-onload"
  onSave="action-group-base-address-onsave">

  <panel name="addressTypePanel">
    <field name="isInvoicingAddr" title="Invoicing address"
      showIf="$popup() &amp;&amp; id == null" type="boolean"/>
  </panel>

  <panel sidebar="true" name="actionsPanel" title="Actions" colSpan="12">
    <button-group hideIf="$popup()" readonlyIf="!id" colSpan="12">
      <button name="mapBtn" title="View map" icon="fa-map-marker"
        onClick="action-base-address-method-view-map"/>
    </button-group>
  </panel>
</form>
```

**Giải thích code:** Form này có `width="large"` - form sẽ chiếm nhiều không gian màn hình hơn default width. Attribute `onLoad="action-group-base-address-onload"` chỉ định một action group chạy khi form load, có thể dùng để set default country dựa trên user's company location. `onSave="action-group-base-address-onsave"` chạy trước khi save, có thể validate địa chỉ hoặc geocode để lấy lat/long.

Field `isInvoicingAddr` có conditional rendering: `showIf="$popup() && id == null"` - chỉ hiện khi form được mở trong popup mode VÀ đang tạo mới (id == null chưa save). Function `$popup()` là built-in expression function của Axelor. `&amp;` là cách viết `&&` trong XML (phải escape).

Panel "actionsPanel" có `sidebar="true"` - đây là sidebar panel thường xuất hiện bên phải form, chứa actions buttons. Button group có `hideIf="$popup()"` - ẩn khi ở popup mode, và `readonlyIf="!id"` - disabled khi record chưa được save (id null). Use case: không thể view map cho address chưa save vì chưa có lat/long.

**Bằng chứng từ code (Client-side transformations):**
```xml
<field name="department" x-bind="{{department|uppercase}}" colSpan="12"/>
<field name="buildingNumber" onChange="action-address-record-change-streetName"
  x-bind="{{buildingNumber|uppercase}}" colSpan="12"/>
```

**Giải thích code:** Attribute `x-bind` implements two-way data binding với pipe transformations (syntax giống Angular). `{{department|uppercase}}` automatically uppercase mọi input vào field department - user type "sales" sẽ tự động thành "SALES". Field `buildingNumber` vừa có uppercase binding VÀ `onChange` action - khi user nhập building number, nó uppercase, sau đó trigger action `action-address-record-change-streetName` có thể auto-construct street name từ building number + street name fields.

**Bằng chứng từ code (Java controller integration):**
```xml
<button name="validateBtn" title="Validate"
  onClick="com.axelor.apps.base.web.AddressController:validate,save"/>
```

**Giải thích code:** Attribute `onClick` có thể reference trực tiếp một Java controller method bằng syntax `package.ClassName:methodName`. Có thể chain nhiều actions bằng comma: `,save` là built-in action để save record. Flow sẽ là: call `AddressController.validate()` (có thể validate address format, check with postal service API), nếu validate pass thì save record.

Tại sao Axelor dùng XML cho views? (1) Declarative approach dễ maintain hơn imperative UI code. (2) Non-developers (business analysts) có thể customize UI. (3) Views có thể được extended và overridden bởi modules khác qua XPath (similar to domain extension). (4) Axelor view engine có thể generate optimized frontend code từ XML. Nhược điểm là: (1) XML verbose. (2) Khó debug vì không có stack traces như code thông thường. (3) IDE support hạn chế (no autocomplete). (4) Complex UI logic vẫn phải drop down to Java controller methods.

---

### 10. FILE CẤU HÌNH ỨNG DỤNG CHÍNH

**File nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

File `axelor-config.properties` với 518 dòng là trung tâm cấu hình (configuration hub) của toàn bộ ứng dụng Axelor. File này tổng hợp mọi aspects: database, security, performance, UI, reporting, encryption. Việc tập trung configuration vào một file có ưu điểm là dễ quản lý và backup, nhưng nhược điểm là file rất dài và có thể overwhelming. Chúng ta sẽ phân tích từng nhóm cấu hình quan trọng.

#### 10.1. Cấu hình Database và Hibernate

```properties
db.default.driver = org.postgresql.Driver
db.default.ddl = update
db.default.url = jdbc:postgresql://10.10.1.31:5432/axelor_demo_staging
db.default.user = odooclau
db.default.password = odoo
```

Database connection sử dụng PostgreSQL driver kết nối đến IP `10.10.1.31` port `5432`, database name `axelor_demo_staging`. Điểm đáng chú ý nhất là `db.default.ddl = update` - đây là Hibernate DDL (Data Definition Language) auto strategy. Với setting này, mỗi khi application khởi động, Hibernate sẽ so sánh structure của Java entity classes với schema hiện tại trong database. Nếu có sự khác biệt (ví dụ: thêm field mới vào entity), Hibernate sẽ TỰ ĐỘNG execute `ALTER TABLE` statements để update schema. Approach này rất tiện lợi cho development vì không cần viết migration scripts thủ công, NHƯNG cực kỳ nguy hiểm cho production vì: (1) Không có version control của schema changes. (2) Không thể rollback nếu migration fail. (3) Risk mất dữ liệu nếu drop columns. Best practice trong production là dùng `ddl=validate` (chỉ kiểm tra, không tự động sửa) và dùng migration tools như Flyway hoặc Liquibase, nhưng Axelor không có migration tool này [Suy luận từ việc không tìm thấy trong source].

```properties
javax.persistence.sharedCache.mode = ENABLE_SELECTIVE
hibernate.search.default.directory_provider = none
hibernate.hikari.minimumIdle = 5
hibernate.hikari.maximumPoolSize = 20
hibernate.hikari.idleTimeout = 300000
```

JPA shared cache (L2 cache) được set `ENABLE_SELECTIVE` nghĩa là chỉ entities được đánh dấu `@Cacheable` (hoặc trong Axelor là `cacheable="true"` trong domain XML) mới được cache. Đây là approach conservative và an toàn - tránh cache toàn bộ (tốn memory) nhưng vẫn cache những entities hot (frequently accessed) như Company, Currency, Country.

Hibernate Search (full-text search engine) có `directory_provider = none` - có thể tính năng này bị disable hoặc Axelor dùng alternative solution. [Suy luận: cần kiểm tra xem có search implementation khác không]

HikariCP connection pool được cấu hình với minimum 5 idle connections và maximum 20 connections. Idle timeout là 300,000 milliseconds (5 phút) - connections không sử dụng quá 5 phút sẽ bị close để free resources. Con số 20 maximum connections là conservative, phù hợp cho small-to-medium deployments. Với high-traffic production system, có thể cần tăng lên 50-100. Tại sao cần connection pooling? Database connections rất "expensive" để tạo (mất vài chục milliseconds), nếu mỗi request đều tạo connection mới sẽ rất chậm. Pooling tái sử dụng connections, giảm latency và database load.

#### 10.2. Thông tin Application

```properties
application.name = Axelor Open Suite
application.version = 8.5.10
application.mode = dev
application.theme = Modern
application.locale = fr
```

Application đang chạy ở `mode = dev` (development mode) - ở mode này, Axelor có thể enable hot reload, detailed error messages, disable caching để dễ debug. Khi deploy production phải đổi sang `mode = prod`. Theme là "Modern" - Axelor có multiple UI themes. Default locale là `fr` (French) - toàn bộ UI mặc định sẽ hiển thị tiếng Pháp (Axelor là công ty Pháp).

#### 10.3. Security và SQL Injection Protection

```properties
#application.domain-blocklist-pattern = (\\(\\s*(SELECT|DELETE|UPDATE)\\s+)|query_to_xml
```

Domain expression filtering - đây là một security measure quan trọng. Axelor cho phép users (đặc biệt là admins) định nghĩa filter expressions cho records, ví dụ `self.status = 'draft' AND self.amount > 1000`. Nếu không validate cẩn thận, expressions này có thể chứa SQL injection attacks. Pattern này block các expressions chứa SQL keywords nguy hiểm như SELECT, DELETE, UPDATE bên trong parentheses, hoặc function `query_to_xml` có thể leak data. Hiện đang comment out - có thể đang dùng default pattern hoặc đã migrate sang validation approach khác.

```properties
#application.script.cache.size = 1000
#application.script.cache.expire-time = 20
```

Groovy script caching - Axelor cho phép viết Groovy scripts trong actions, BPM tasks. Groovy scripts phải được compiled trước khi execute, quá trình này tốn CPU. Caching compiled scripts với size 1000 (1000 unique scripts) và expire time 20 phút giúp tránh re-compilation. Nếu cache quá nhỏ hoặc expire quá nhanh, performance sẽ giảm. Nếu cache quá lớn, tốn memory.

#### 10.4. View Configuration

```properties
view.max-tabs = 10
view.menubar.location = both
```

UI chỉ cho phép mở maximum 10 tabs cùng lúc - tránh users mở quá nhiều tabs làm chậm browser. Menubar có thể hiển thị ở `left` (sidebar), `top` (top bar), hoặc `both` - cấu hình này cho phép users chọn cách navigation ưa thích.

#### 10.5. BPM Studio Configuration (QUAN TRỌNG)

```properties
studio.bpm.logging = false
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
studio.bpm.history.time.to.live = P180D
```

Đây là phát hiện CỰC KỲ quan trọng - Axelor có tích hợp BPM (Business Process Management) engine! BPM engine có connection pool RIÊNG biệt với main application pool. Tại sao cần pool riêng? BPM workflows thường chạy lâu (long-running processes) và có thể hold database connections trong thời gian dài, nếu dùng chung pool với main app sẽ gây connection starvation. Idle connections: 10, max active: 50 - lớn hơn nhiều so với main pool (20), cho thấy BPM processes có concurrency cao.

`history.time.to.live = P180D` - ISO 8601 duration format, P180D = 180 days. Completed process instances và history data sẽ được giữ trong 180 ngày rồi tự động cleanup. Điều này quan trọng vì BPM tables phình to rất nhanh - mỗi process tạo nhiều rows (process instance, activities, variables, history), nếu không cleanup định kỳ database sẽ đầy. 180 ngày là balance tốt giữa audit requirements và database size.

[Suy luận]: Dựa trên config patterns (connection pooling, history TTL), BPM engine có thể là Camunda - một trong những BPM engines phổ biến nhất cho Java. Camunda có config tương tự.

#### 10.6. Authentication và Single Sign-On

```properties
#auth.provider-order =
#auth.user.provisioning = none
#auth.user.default-group = users

#auth.provider.google.client-id =
#auth.provider.google.secret =

#auth.provider.keycloak.client-id = demo-app
#auth.provider.keycloak.secret = 233d1690-4498-490c-a60d-5d12bb685557

#auth.provider.saml.keystore-path = {java.io.tmpdir}/samlKeystore.jks

#auth.ldap.server.url = ldap://localhost:389
#auth.ldap.server.starttls = false
#auth.ldap.user.filter = (uid={0})

#auth.provider.cas.login-url = https://localhost:8443/cas/login
```

Axelor hỗ trợ đa dạng authentication methods: Google OAuth, Keycloak (enterprise identity management), SAML 2.0 (SSO standard), LDAP (Active Directory integration), CAS (Central Authentication Service). Đây là danh sách ấn tượng - phần lớn business applications chỉ support một hoặc hai methods. Multi-provider support rất quan trọng cho enterprise customers có infrastructure sẵn (ví dụ: đã có Keycloak hoặc LDAP).

`auth.user.provisioning = none` control việc tự động tạo user accounts. Options có thể là: `create` (tự động tạo user trong database khi login qua SSO lần đầu), `link` (link SSO account với existing user), `none` (không tự động, phải tạo user trước). `none` là approach secure nhất nhưng kém convenient.

#### 10.7. Data Management

```properties
data.upload.dir = {java.io.tmpdir}/axelor
data.upload.max-size = 5
#data.upload.allowlist.pattern = \\.(xml|html|jpg|png|pdf|xsl)$
#data.upload.blocklist.pattern = \\.(svg)$

data.export.encoding = UTF-8
data.export.dir = {java.io.tmpdir}/axelor
#data.export.max-size = 5000

data.import.demo-data = false
```

File uploads được giới hạn 5MB - đủ cho documents, images nhưng không đủ cho videos. Upload directory là `{java.io.tmpdir}/axelor` - `{java.io.tmpdir}` là placeholder được replace bằng system temp dir. Allowlist/blocklist patterns control file types: có thể cho phép XML, HTML, images, PDFs nhưng block SVG (SVG có thể chứa JavaScript - XSS risk).

Data export max size 5000 records - giới hạn này tránh users export millions of records làm database quá tải hoặc tạo file quá lớn không thể mở. UTF-8 encoding đảm bảo support đầy đủ Unicode characters (tiếng Việt, tiếng Trung, emoji).

`data.import.demo-data = false` - không import demo data khi khởi động. Demo data hữu ích cho testing nhưng phải disable trong production.

#### 10.8. Quartz Scheduler

```properties
#quartz.enable = true
#quartz.thread-count = 3
```

Quartz là job scheduling library cho Java, cho phép chạy các tasks theo lịch (schedule) hoặc định kỳ (recurring). Thread count = 3 nghĩa là maximum 3 jobs có thể chạy đồng thời. Con số nhỏ này hợp lý vì scheduled jobs thường là nặng (heavy) - nếu cho phép quá nhiều jobs chạy cùng lúc sẽ overload system. Examples của scheduled jobs: send reminder emails, generate daily reports, cleanup old data, sync with external systems.

#### 10.9. GDPR Compliance và Audit Trail

```properties
hibernate.session_factory.interceptor = com.axelor.apps.base.tracking.GlobalAuditInterceptor
```

Đây là một Hibernate interceptor tùy chỉnh được gọi cho MỌI database operations (insert, update, delete). `GlobalAuditInterceptor` likely ghi lại WHO did WHAT WHEN - critical cho GDPR compliance (EU regulation yêu cầu track mọi access và modifications của personal data). Interceptor có performance cost (mỗi operation phải ghi thêm audit records) nhưng necessary cho compliance.

#### 10.10. Context Providers

```properties
context.date = com.axelor.apps.base.service.DateService:date
context.appLogo = com.axelor.apps.base.service.user.UserService:getUserActiveCompanyLogoLink
context.app = com.axelor.studio.app.service.AppService
```

Context providers inject dynamic values vào expressions và templates. Ví dụ: trong view expressions, có thể dùng `$date` sẽ gọi `DateService.date()` để lấy current date. `$appLogo` return URL của company logo để hiển thị trong UI. Đây là dependency injection pattern applied to configuration.

#### 10.11. Studio Apps Installation

```properties
studio.apps.install = all
```

`studio.apps.install = all` tự động install TẤT CẢ Axelor Studio apps khi application khởi động lần đầu. Alternative là danh sách module names separated by comma (ví dụ: `sale,account,stock`). Setting `all` convenient cho development nhưng có thể không mong muốn trong production nếu chỉ cần subset of functionality.

---

## Những điều KHÔNG tìm thấy trong Source Code

Sau quá trình nghiên cứu toàn diện source code, có một số thành phần và công nghệ mà chúng tôi KHÔNG tìm thấy, điều này cũng tiết lộ nhiều thông tin về kiến trúc của Axelor:

1. **Không tìm thấy Spring Framework dependencies** - Axelor KHÔNG sử dụng Spring Boot hay Spring Framework, điều này ngạc nhiên vì phần lớn Java enterprise applications hiện đại đều dùng Spring. Axelor có proprietary framework riêng với Dependency Injection dựa trên Google Guice (nhẹ hơn Spring nhiều). [Từ source code: không có spring-boot-starter hay spring-context trong dependencies]

2. **Không tìm thấy file `persistence.xml`** - JPA standard thường yêu cầu file này để cấu hình persistence unit, nhưng Axelor quản lý JPA configuration qua framework riêng, có thể programmatically hoặc qua axelor-config.properties. [Từ source code: searched toàn bộ project không có file này]

3. **Không tìm thấy Liquibase hoặc Flyway** - Đây là hai migration tools phổ biến nhất cho Java database versioning. Việc thiếu chúng nghĩa là Axelor hoàn toàn rely on Hibernate DDL auto-update (`ddl=update`), đây là một quyết định kiến trúc có rủi ro cho production environments. [Từ source code: không có dependencies và không có migration scripts directories]

4. **Không tìm thấy Docker configuration** - Không có `Dockerfile`, `docker-compose.yml` ở root project. Điều này có thể vì Axelor được thiết kế deploy theo traditional way (WAR file lên Tomcat) hơn là containerized deployment. Docker config có thể có ở repository riêng hoặc documentation riêng. [Từ source code: không có ở root level]

5. **Không tìm thấy CI/CD configuration files** - Không có `.gitlab-ci.yml`, `.github/workflows`, `Jenkinsfile` ở root project. CI/CD có thể được setup ở infrastructure level hoặc trong git repository riêng của công ty deploy. [Từ source code: không có ở root level]

6. **Không tìm thấy modern frontend framework (ngoài React map-viewer)** - Không có Angular, Vue.js project ở frontend. Main UI của Axelor có thể dùng custom JavaScript framework (proprietary) generate từ view XMLs, hoặc có thể là legacy framework. React chỉ dùng cho map-viewer component. [Từ source code: chỉ có React trong axelor-base/map-viewer]

7. **Không tìm thấy REST API framework riêng biệt** - Không có Spring MVC, JAX-RS (Jersey/RESTEasy) configuration rõ ràng. Axelor likely có built-in REST layer generated tự động từ domain XMLs. [Suy luận: API references trong config nhưng không thấy explicit framework setup]

8. **Không tìm thấy message queue systems** - Không có RabbitMQ, Kafka, ActiveMQ dependencies. Async processing có thể rely hoàn toàn on Quartz scheduler, không có true message-driven architecture. [Từ source code: không có JMS hoặc messaging libraries ngoài axelor-message addon]

9. **Không tìm thấy monitoring/observability tools built-in** - Không có Prometheus, Micrometer, ELK stack integrations. Monitoring có thể là responsibility của deployment infrastructure hơn là application code. [Từ source code: không có metrics libraries]

10. **Không tìm thấy test coverage tools** - Có JUnit dependency nhưng không có Jacoco, Cobertura cho test coverage reporting. Testing có thể còn đơn giản hoặc use external tools. [Từ source code: chỉ có mockito và junit platform]

---

## Câu hỏi mở cần điều tra thêm

Sau bước nghiên cứu đầu tiên này, xuất hiện nhiều câu hỏi kỹ thuật cần điều tra sâu hơn ở các bước tiếp theo:

1. **Code generation mechanism chi tiết:** Domain XMLs được transform thành Java entity classes bằng cách nào? Là annotation processor chạy lúc compile, hay là Gradle plugin generate code trước compile? Generated code được lưu ở đâu? Làm thế nào để inspect generated entities khi debugging?

2. **Frontend rendering engine:** View XMLs được render ra HTML/JavaScript thế nào? Framework frontend là gì - custom framework, legacy GWT (Google Web Toolkit), hoặc một template engine nào đó? React chỉ dùng cho map, vậy phần còn lại dùng gì?

3. **BPM engine cụ thể:** Config cho thấy có BPM nhưng không rõ engine. Cần tìm trong axelor-studio addon (binary dependency) xem có reference đến Camunda, Activiti, Flowable hay custom implementation không? Có support BPMN 2.0 standard hay là proprietary notation?

4. **Repository pattern implementation:** Domain XML không define repository methods như `findByName()`. Vậy repositories được generate tự động với CRUD methods? Hay developers phải viết custom repository classes? Có Query DSL hay criteria API nào không?

5. **Action execution engine:** Khi button trong view XML trigger `action-method`, hệ thống routing request tới Java controller method như thế nào? Có convention-based mapping (như Rails) hay annotation-based (như Spring)? Performance của action dispatching ra sao với thousands of actions?

6. **MetaJsonModel và Custom Models:** Yêu cầu research đề cập "custom models" và "JSON models". Cần tìm entities như `MetaJsonRecord`, `MetaJsonModel`, `MetaJsonField` để hiểu cơ chế. Dữ liệu được serialize/deserialize thế nào? Query performance với JSON columns?

7. **axelor-studio source code:** Studio addon là binary dependency (`axelor-studio:3.5.1`), không có source trong open-suite repository. Đây có phải là paid/proprietary addon? Có document nào về Studio APIs không? Có thể decompile để research không?

8. **View inheritance và extension:** Đã thấy `extension="true"` trong view XML. Mechanism cụ thể là XPath-based patching? Có conflict resolution nào khi multiple modules extend cùng view? Order of extensions?

9. **Multi-tenancy implementation:** Config có mention `application.multi-tenancy` nhưng đang disabled. Khi enable, implementation là per-schema (mỗi tenant một database schema) hay per-row (shared tables với tenant_id column)? Security isolation thế nào?

10. **Hibernate Search alternatives:** Config có `hibernate.search.default.directory_provider = none`. Vậy full-text search được implement bằng gì? PostgreSQL full-text search? Elasticsearch integration? Hay không có search capability?

11. **Generated API endpoints:** Mỗi domain entity tự động có REST API endpoints không? Format là gì - `/api/com.axelor.apps.base.db.Address` hay short form `/api/address`? Có Swagger/OpenAPI documentation auto-generated không?

12. **Permission system granularity:** Config có mention `application.permission.disable-action` và `disable-relational-field`. Điều này cho thấy permission system rất granular. Cần research cách define permissions - XML, database, annotation? Integration với views thế nào?

13. **Deployment options:** Axelor deploy như WAR file lên Tomcat? Hay có embedded server (như Spring Boot)? Có standalone JAR executable không? Kubernetes-friendly hay cần traditional application server?

---

## Sơ đồ kiến trúc tổng quan (Architecture Diagram)

Dựa trên findings từ source code analysis, đây là sơ đồ tổng quan về cấu trúc project Axelor ERP:

```
axelor-erp/                                    (Root Project)
│                                              Version: 8.5.10
│                                              Java: 21 (OpenJDK)
│                                              Build: Gradle 8.x + Multi-module
│
├── build.gradle                               Root build configuration
│   └── allprojects {
│         group = 'com.axelor.apps'
│         java toolchain = Java 21
│       }
│
├── settings.gradle                            Dynamic module discovery
│   └── Traverse modules/ directory
│       → Auto-include modules with build.gradle
│
├── gradle.properties                          JVM settings
│   └── -Xmx2g heap
│       parallel builds enabled
│
├── src/main/resources/
│   └── axelor-config.properties               [518 LINES] Main configuration
│       ├── Database: PostgreSQL + Hibernate
│       ├── Connection Pool: HikariCP (5-20 connections)
│       ├── BPM: Dedicated pool (10-50 connections)
│       ├── Auth: Google, Keycloak, SAML, LDAP, CAS
│       ├── Security: GDPR audit interceptor
│       └── Studio: Auto-install all apps
│
└── modules/
    └── axelor-open-suite/                     Git Submodule (v8.5.9)
        │
        ├── libs.gradle                        External Dependencies
        │   ├── axelor-studio:3.5.1           (Binary addon - No-code platform + BPM)
        │   ├── axelor-message:3.3.0          (Binary addon - Messaging)
        │   ├── axelor-utils:3.5.0            (Binary addon - Utilities)
        │   ├── groovy:3.0.23                 (Scripting language)
        │   ├── pac4j-core:5.7.7              (Multi-provider SSO)
        │   ├── pdfbox, openpdf               (PDF processing)
        │   ├── bouncycastle                  (Cryptography)
        │   └── ical4j, iban4j, zxing, ...    (Business libs)
        │
        ├── version.gradle / version.txt       Version: 8.5.9
        │
        ├── axelor-base/                       ★ FOUNDATION MODULE ★
        │   ├── build.gradle                   Dependencies:
        │   │   ├── axelor-studio             - No-code platform
        │   │   ├── axelor-message            - Messaging
        │   │   ├── axelor-utils              - Utilities
        │   │   ├── Node.js 22.17.1           - Frontend build
        │   │   └── React 19.1                - Map viewer component
        │   │
        │   ├── src/main/java/                 Java business logic
        │   │   └── com/axelor/apps/base/
        │   │       ├── service/              Service layer
        │   │       ├── web/                  Web controllers
        │   │       └── repo/                 Custom repositories
        │   │
        │   ├── src/main/resources/
        │   │   ├── domains/                   ★★★ 189 Domain XML files ★★★
        │   │   │   └── (Entity definitions → Code generation)
        │   │   ├── views/                     ★★★ 192 View XML files ★★★
        │   │   │   └── (UI definitions → Rendered by framework)
        │   │   ├── data-init/                CSV seed data
        │   │   ├── reports/                  BIRT report templates
        │   │   └── i18n/                     Translation files
        │   │
        │   ├── src/main/webapp/               Web resources
        │   │
        │   └── map-viewer/                    ★ React App ★
        │       ├── package.json              React 19.1 + dependencies
        │       ├── src/                      React components
        │       └── dist/                     Build output → bundled into JAR
        │
        ├── axelor-account/                    Accounting Module
        │   ├── build.gradle                   Depends on: axelor-base
        │   └── src/main/resources/
        │       ├── domains/                   ★ 122 entities ★ (2nd largest)
        │       ├── views/                     120 views
        │       ├── l10n/                      Localization (country-specific)
        │       └── reports/                   40 BIRT reports
        │
        ├── axelor-sale/                       Sales Module
        │   ├── build.gradle                   Depends on: axelor-crm
        │   └── src/main/resources/
        │       ├── domains/                   31 entities
        │       └── views/                     30 views
        │
        ├── axelor-crm/                        CRM Module
        ├── axelor-purchase/                   Purchase Module
        ├── axelor-stock/                      Inventory Module
        ├── axelor-supplychain/                Supply Chain (Integration layer)
        ├── axelor-production/                 Manufacturing
        ├── axelor-human-resource/             HR Module
        ├── axelor-project/                    Project Management
        ├── axelor-bank-payment/               Banking & SEPA
        ├── axelor-budget/                     Budget Planning
        └── [18 other business modules...]     Total: 27 modules
```

**Luồng xử lý chính (Main Processing Flow):**

```
1. DEVELOPMENT TIME (Build Phase):
   Domain XML files (189 in base)
       ↓
   Gradle Build Process
       ↓
   Axelor Code Generator (Plugin com.axelor.app:7.4.7)
       ↓
   Generated Java Entity Classes + Repositories
       ↓
   Java Compilation (Java 21)
       ↓
   React Build (map-viewer: Node 22.17.1 + Yarn)
       ↓
   JAR Packaging (All resources bundled)

2. RUNTIME (Application Phase):
   axelor-config.properties → Load configuration
       ↓
   Hibernate initializes with PostgreSQL
       ↓
   ├─ Main Connection Pool: HikariCP (5-20)
   └─ BPM Connection Pool: Dedicated (10-50)
       ↓
   Dynamic Module Loading (27 modules)
       ↓
   View XML Rendering → Web UI
       ↓
   User Interactions → Actions (XML-defined)
       ↓
   ├─ action-method → Java Controllers
   ├─ action-record → Record manipulation
   ├─ action-attrs → UI updates
   └─ action-script → Groovy execution
       ↓
   Service Layer (Business Logic)
       ↓
   Repository Layer (JPA Queries)
       ↓
   Database (PostgreSQL) + Caching (L1 + L2)
```

**Key Architectural Patterns Identified:**

1. **XML-Driven Development (MDD - Model-Driven Development):** Entities và views được định nghĩa bằng XML, sau đó code generator tạo Java classes. Approach này cho phép rapid development và business analyst có thể maintain models.

2. **Gradle Multi-Module với Dynamic Discovery:** Root project tự động phát hiện modules trong `modules/` directory, không cần hardcode. Scalable và dễ add modules mới.

3. **Git Submodule Architecture:** Business logic (axelor-open-suite) tách biệt thành submodule, có thể được fork và customized độc lập.

4. **Binary Addon Model:** Critical features (Studio, BPM, Messaging) được package thành binary addons, có thể cho licensing flexibility.

5. **Hybrid Frontend:** React cho rich components (map), proprietary framework cho main UI generated từ view XMLs.

6. **Dedicated BPM Connection Pool:** BPM workflows có resource requirements khác main app, cần pool riêng để avoid connection starvation.

7. **Comprehensive Security:** Multi-provider authentication (Google, SAML, LDAP, Keycloak), GDPR audit trail, SQL injection protection.

8. **Convention over Configuration với extensive Configuration:** Framework có nhiều conventions (domain XML → entities, views XML → UI) nhưng vẫn có central config file cho fine-tuning.
