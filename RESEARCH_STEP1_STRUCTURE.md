# BƯỚC 1: KHÁM PHÁ CẤU TRÚC TỔNG QUAN - Phân Tích Từ Mã Nguồn

## Phương pháp phân tích

Quá trình nghiên cứu được thực hiện bằng cách đọc trực tiếp các tệp cấu hình và mã nguồn tại thư mục `/Volumes/works/code/java/axelor/axelor-erp`. Phạm vi phân tích bao gồm các tệp và thư mục quan trọng sau:

**Tệp cấu hình chính:**
- `/settings.gradle` - Tệp cấu hình cơ chế tải module động của Gradle
- `/build.gradle` - Tệp xây dựng gốc chứa thông tin dự án và cấu hình Java
- `/gradle.properties` - Thiết lập máy ảo Java (JVM) và các tham số xây dựng
- `/src/main/resources/axelor-config.properties` - Tệp cấu hình ứng dụng chính (518 dòng)

**Phụ thuộc và phiên bản:**
- `/modules/axelor-open-suite/libs.gradle` - Khai báo các thư viện bên ngoài
- `/modules/axelor-open-suite/version.gradle` - Quản lý phiên bản
- `/modules/axelor-open-suite/version.txt` - Số phiên bản cụ thể

**Cấu hình xây dựng module:**
- `/modules/axelor-open-suite/axelor-base/build.gradle`
- `/modules/axelor-open-suite/axelor-sale/build.gradle`
- `/modules/axelor-open-suite/axelor-account/build.gradle`

**Mã nguồn mẫu:**
- Tệp XML miền (domain): `Address.xml`, `Company.xml` để hiểu cách định nghĩa thực thể (entity)
- Tệp XML giao diện (view): `Address.xml` để hiểu cách xây dựng giao diện người dùng

---

## Kết quả chi tiết

### 1. CƠ CHẾ TẢI MODULE ĐỘNG

**Nguồn:** `/settings.gradle` [Từ source code]

Axelor ERP sử dụng một cơ chế đặc biệt để quản lý các module - thay vì liệt kê cứng từng module trong tệp cấu hình, hệ thống tự động quét và phát hiện tất cả các module có trong thư mục `modules/`. Cơ chế này được thực hiện thông qua plugin tùy chỉnh của Gradle có tên `com.axelor.app` phiên bản 7.4.7. Khi Gradle chạy, nó sẽ duyệt qua tất cả các thư mục con cấp một bên trong thư mục `modules/`, và nếu thư mục nào chứa tệp `build.gradle`, nó sẽ được nhận diện là một module hợp lệ và được thêm vào danh sách xây dựng.

Cách tiếp cận này mang lại lợi ích lớn về mặt khả năng mở rộng - khi lập trình viên muốn thêm một module mới, họ chỉ cần tạo thư mục mới với tệp `build.gradle` bên trong thư mục `modules/`, không cần phải sửa đổi tệp cấu hình gốc. Hệ thống cũng cấu hình hai kho lưu trữ (repository) chính: Maven Central với điều kiện loại trừ nhóm `com.axelor` để tránh xung đột, và kho lưu trữ Axelor Nexus tại địa chỉ `https://repository.axelor.com/nexus/repository/maven-public/` để tải các thư viện chuyên biệt của Axelor.

**Bằng chứng từ mã:**
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

**Giải thích mã:** Đoạn mã trên sử dụng phương thức `traverse()` của Groovy để duyệt qua các thư mục. Tham số `maxDepth: 1` đảm bảo chỉ quét cấp một, không đệ quy xuống các thư mục con. Biến `modules` thu thập danh sách các thư mục hợp lệ, sau đó được lưu vào `gradle.ext.appModules` để các phần khác của tập lệnh xây dựng có thể truy cập. Vòng lặp `each` cuối cùng thực hiện việc thêm từng module vào dự án với mẫu đặt tên `modules:tên_thư_mục` và ánh xạ vị trí thực tế của module qua thuộc tính `projectDir`.

---

### 2. THÔNG TIN DỰ ÁN GỐC VÀ JAVA

**Nguồn:** `/build.gradle` [Từ source code]

Tệp xây dựng gốc tiết lộ những thông tin quan trọng về dự án: tên chính thức là "Axelor ERP", đang ở phiên bản 8.5.10, và thuộc nhóm `com.axelor.apps`. Một điểm đáng chú ý là có sự chênh lệch phiên bản giữa dự án gốc (8.5.10) và module con axelor-open-suite bên trong (8.5.9 theo tệp version.txt), cho thấy dự án gốc đang ở một phiên bản phát triển mới hơn module con.

Quyết định công nghệ quan trọng nhất được thể hiện qua chuỗi công cụ Java (Java toolchain): dự án sử dụng Java 21, phiên bản hỗ trợ dài hạn (Long-Term Support - LTS) mới nhất của Java tại thời điểm này. Việc sử dụng Java 21 mang lại nhiều lợi thế: hiệu năng (performance) được cải thiện đáng kể nhờ các tối ưu hóa trong máy ảo Java, hỗ trợ so khớp mẫu (pattern matching) và lớp bản ghi (record classes) giúp mã nguồn ngắn gọn hơn, và đặc biệt là các luồng ảo (virtual threads) cho phép xử lý hàng triệu luồng đồng thời mà không tiêu tốn quá nhiều tài nguyên hệ thống.

Dự án cũng định nghĩa nhiều nhiệm vụ Gradle tùy chỉnh phục vụ cho quy trình đảm bảo chất lượng: `checkCsvEOL` kiểm tra ký tự cuối dòng trong tệp CSV, `checkBirtCredentials` đảm bảo không có thông tin đăng nhập nhạy cảm bị lưu trực tiếp trong tệp báo cáo BIRT, và các nhiệm vụ kiểm tra XML để phát hiện lỗi cú pháp hoặc thuộc tính không hợp lệ trước khi chạy ứng dụng.

**Bằng chứng từ mã:**
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

**Giải thích mã:** Khối `allprojects` áp dụng cấu hình cho tất cả các dự án, bao gồm cả dự án gốc và các dự án con. Việc khai báo `toolchain` thay vì chỉ đơn giản đặt `sourceCompatibility` có ý nghĩa quan trọng: Gradle sẽ tự động tải xuống JDK 21 nếu máy xây dựng chưa có, đảm bảo môi trường xây dựng nhất quán trên mọi máy lập trình viên. Thuộc tính `languageVersion` xác định phiên bản Java mà mã nguồn sử dụng, ảnh hưởng đến cả quá trình biên dịch và kiểm tra lỗi của môi trường phát triển tích hợp (IDE).

---

### 3. CẤU HÌNH MÁY ẢO JAVA VÀ HIỆU NĂNG XÂY DỰNG

**Nguồn:** `/gradle.properties` [Từ source code]

Tệp `gradle.properties` chứa các thiết lập quan trọng cho quá trình xây dựng và hiệu năng. Đầu tiên, `org.gradle.parallel=true` cho phép Gradle xây dựng nhiều module đồng thời, thay vì xử lý tuần tự. Điều này quan trọng với một dự án có 27 module như Axelor - thời gian xây dựng có thể giảm từ vài phút xuống còn dưới một phút trên máy đủ mạnh với bộ xử lý nhiều lõi.

Thiết lập `org.gradle.jvmargs=-Xmx2g` cấp phát 2GB bộ nhớ heap cho máy ảo Java chạy daemon của Gradle. Con số 2GB là một lựa chọn cân bằng: đủ lớn để xử lý việc biên dịch và sinh mã tự động của Axelor mà không bị lỗi hết bộ nhớ, nhưng không quá lớn đến mức chiếm dụng quá nhiều bộ nhớ RAM của máy lập trình viên. Đường dẫn thư mục chính của Java được trỏ tới `/usr/local/Cellar/openjdk@21/21.0.9/` - đây là quy ước của Homebrew trên macOS, cho thấy môi trường phát triển đang được thiết lập trên macOS.

**Bằng chứng từ mã:**
```properties
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx2g
org.gradle.java.home=/usr/local/Cellar/openjdk@21/21.0.9/libexec/openjdk.jdk/Contents/Home
```

**Giải thích mã:** Ba dòng cấu hình này làm việc cùng nhau để tối ưu hóa quá trình xây dựng. Dòng đầu tiên bật chế độ song song, hai dòng sau đảm bảo có đủ tài nguyên và đúng phiên bản JDK. Đường dẫn `java.home` ghi đè biến môi trường `JAVA_HOME` của hệ thống, đảm bảo Gradle luôn dùng đúng JDK 21 ngay cả khi lập trình viên có nhiều phiên bản Java được cài đặt.

---

### 4. DANH SÁCH MODULE TRONG AXELOR OPEN SUITE

**Nguồn:** Danh sách thư mục `/modules/axelor-open-suite/` [Từ source code]

Cấu trúc dự án sử dụng mô hình git submodule: dự án gốc `axelor-erp` (phiên bản 8.5.10) chứa một submodule tên `axelor-open-suite` (phiên bản 8.5.9) ở bên trong thư mục `modules/`. Mẫu này cho phép quản lý vòng đời của các module nghiệp vụ độc lập với dự án bao bọc bên ngoài - ví dụ, một công ty có thể tạo nhánh riêng (fork) axelor-open-suite để tùy chỉnh logic nghiệp vụ nhưng vẫn giữ nguyên dự án gốc để dễ dàng nâng cấp.

Bên trong axelor-open-suite có tổng cộng **27 module nghiệp vụ**, mỗi module phụ trách một mảng nghiệp vụ cụ thể:

1. **axelor-base** - Module nền tảng quan trọng nhất, chứa các thực thể cơ bản như đối tác (Partner), sản phẩm (Product), công ty (Company), người dùng (User). Hầu hết các module khác đều phụ thuộc vào base.

2. **axelor-account** - Module kế toán, quản lý hóa đơn (Invoice), tài khoản kế toán (Account), thuế (Tax), thanh toán (Payment). Đây là module lớn thứ hai với 122 thực thể miền.

3. **axelor-sale** - Module bán hàng, quản lý đơn hàng (SaleOrder), báo giá (Quotation). Phụ thuộc vào axelor-crm.

4. **axelor-crm** - Quản lý quan hệ khách hàng, bao gồm khách hàng tiềm năng (Lead), cơ hội bán hàng (Opportunity), liên hệ (Contact).

5. **axelor-purchase** - Quản lý mua hàng, gồm đơn mua (PurchaseOrder), nhà cung cấp (Supplier).

6. **axelor-stock** - Quản lý kho hàng, bao gồm hàng tồn (Stock), vị trí kho (Location), xuất nhập kho (Movement).

7. **axelor-supplychain** - Chuỗi cung ứng, tích hợp giữa bán hàng, mua hàng và kho. Đây là lớp tích hợp (integration layer).

8. **axelor-production** - Sản xuất, bao gồm bảng kê vật liệu (Bill of Materials - BOM), trung tâm sản xuất (WorkCenter).

9. **axelor-human-resource** - Nhân sự, quản lý nhân viên (Employee), nghỉ phép (Leave), chi phí (Expense).

10. **axelor-project** - Quản lý dự án, gồm công việc (Task), chấm công (TimeSheet).

Các module còn lại bao gồm: contract (hợp đồng), bank-payment (thanh toán ngân hàng), budget (ngân sách), cash-management (quản lý dòng tiền), fleet (quản lý đội xe), helpdesk (hỗ trợ khách hàng), intervention (can thiệp/sửa chữa), maintenance (bảo trì), marketing (tiếp thị), quality (chất lượng), talent (tuyển dụng), gdpr (tuân thủ quy định bảo vệ dữ liệu), mobile-settings (cấu hình di động), client-portal (cổng thông tin khách hàng), supplier-portal (cổng thông tin nhà cung cấp), supplier-management (quản lý nhà cung cấp), và thư mục changelogs chứa tệp nhật ký thay đổi.

> ⚠️ **Suy luận:** Các module được tổ chức theo dạng cây phân cấp phụ thuộc: axelor-base ở gốc, các module chuyên về miền như account, crm, stock phụ thuộc trực tiếp vào base, và các module tích hợp như supplychain phụ thuộc vào nhiều module khác cùng lúc. Mô hình này đảm bảo rằng logic nghiệp vụ được tổ chức theo module và có thể tái sử dụng.

---

### 5. PHỤ THUỘC BÊN NGOÀI VÀ ADDON

**Nguồn:** `/modules/axelor-open-suite/libs.gradle` [Từ source code]

Tệp này khai báo tất cả các thư viện bên ngoài mà Axelor Open Suite phụ thuộc vào. Điểm đặc biệt đáng chú ý là Axelor sử dụng một kiến trúc addon: ba thành phần quan trọng (axelor-studio, axelor-message, axelor-utils) không phải là module trong cây mã nguồn mà được tải xuống như các phụ thuộc nhị phân (binary dependencies) từ kho lưu trữ Axelor.

**axelor-studio:3.5.1** là addon cực kỳ quan trọng - đây là nền tảng mã thấp/không mã (low-code/no-code platform) cho phép người dùng tạo model, giao diện, và quy trình làm việc qua giao diện đồ họa mà không cần viết mã. Addon này bao gồm cả công cụ quản lý quy trình nghiệp vụ (BPM engine). **axelor-message:3.3.0** cung cấp hệ thống nhắn tin. **axelor-utils:3.5.0** chứa các hàm tiện ích dùng chung.

Về thư viện bên ngoài, danh sách cho thấy Axelor là một nền tảng đầy đủ tính năng:
- **Groovy 3.0.23** - Ngôn ngữ kịch bản cho phép viết logic nghiệp vụ động mà không cần biên dịch lại toàn bộ ứng dụng.
- **Xử lý PDF** (pdfbox 3.0.0, openpdf 1.4.2) - Tạo và xử lý tệp PDF cho báo cáo, hóa đơn.
- **Thư viện bảo mật** (bouncycastle 1.78.1) - Mã hóa và chữ ký số, quan trọng cho hóa đơn điện tử.
- **Lịch** (ical4j 2.2.0) - Xử lý tệp iCalendar cho tích hợp với Google Calendar, Outlook.
- **Kiểm tra IBAN** (iban4j) - Kiểm tra số tài khoản ngân hàng quốc tế.
- **Mã vạch** (zxing 3.5.0) - Tạo và đọc mã vạch và mã QR.
- **WebDAV** (jackrabbit-webdav) - Giao thức để truy cập tệp qua web.
- **OAuth** (google-oauth-client-jetty) - Đăng nhập bằng tài khoản Google.
- **Đăng nhập một lần** (pac4j-core 5.7.7) - Bộ khung cho đăng nhập một lần (Single Sign-On), hỗ trợ nhiều giao thức: OAuth, SAML, CAS, OpenID Connect.
- **SOAP** (groovy-wslite) - Gọi dịch vụ web kiểu cũ.
- **OpenAPI** (swagger-jaxrs2) - Tự động sinh tài liệu API.

**Bằng chứng từ mã:**
```groovy
libs.axelor_studio = 'com.axelor.addons:axelor-studio:3.5.1'
libs.axelor_message = 'com.axelor.addons:axelor-message:3.3.0'
libs.axelor_utils = 'com.axelor.addons:axelor-utils:3.5.0'
libs.groovy = 'org.codehaus.groovy:groovy-all:3.0.23'
libs.pac4j_core = "org.pac4j:pac4j-core:5.7.7"
```

**Giải thích mã:** Cú pháp `libs.tên_biến = 'dependency'` là quy ước của Gradle 7.x trở lên, định nghĩa danh mục phiên bản để quản lý tập trung phiên bản thư viện. Dòng `axelor_studio` cho thấy nhóm là `com.axelor.addons`, tên sản phẩm là `axelor-studio`, phiên bản là `3.5.1`. Tất cả các module khác có thể tham chiếu `libs.axelor_studio` thay vì viết cứng chuỗi phụ thuộc, giúp dễ dàng nâng cấp phiên bản sau này.

---

### 6. CẤU TRÚC AXELOR-BASE - MODULE NỀN TẢNG

**Nguồn:** `/modules/axelor-open-suite/axelor-base/build.gradle` [Từ source code]

axelor-base là module quan trọng và phức tạp nhất trong toàn bộ hệ thống, đóng vai trò như một lớp nền tảng mà tất cả module khác đều xây dựng dựa trên nó. Tệp build.gradle của module này tiết lộ một điểm thú vị: bên cạnh phần phụ trợ Java, module còn có một thành phần giao diện riêng biệt được viết bằng **React 19.1** nằm trong thư mục `map-viewer/`.

Thành phần React này sử dụng **Node.js 22.17.1** và **Yarn 1.22.19** để quản lý các phụ thuộc và xây dựng. Gradle được cấu hình để tự động tải xuống đúng phiên bản Node.js (thông qua `download = true`), sau đó chạy `yarn install` để cài đặt các gói npm, rồi `yarn run build` để biên dịch ứng dụng React thành các tài nguyên tĩnh. Cuối cùng, đầu ra của quá trình xây dựng React được sao chép vào thư mục `webapp/base/map-viewer/` và đóng gói chung vào tệp JAR của module. Cách tiếp cận này có nghĩa ứng dụng React được triển khai như một phần của tệp JAR phụ trợ, không cần phục vụ riêng từ máy chủ Node.js.

> ⚠️ **Suy luận:** Việc có React trong một dự án Java có thể là một widget đặc biệt cho chức năng xem bản đồ - hiển thị địa chỉ khách hàng, vị trí kho hàng trên Google Maps hoặc OpenStreetMap. Việc dùng React cho widget phức tạp như bản đồ có ý nghĩa vì hệ sinh thái React có nhiều thư viện bản đồ tốt và hiệu năng của DOM ảo phù hợp với việc hiển thị hàng nghìn điểm đánh dấu.

**Bằng chứng từ mã (đường ống xây dựng giao diện):**
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

**Giải thích mã:** Khối `node` cấu hình plugin Node của Gradle. Thuộc tính `download = true` quan trọng - nó cho phép Gradle tự động tải Node.js nếu lập trình viên chưa cài, đảm bảo môi trường xây dựng nhất quán trên mọi máy. `nodeModulesDir` chỉ định nơi chạy `yarn install`. Nhiệm vụ `buildFront` phụ thuộc vào `installFrontDeps` (cài các phụ thuộc trước), sau đó chạy xây dựng yarn. Nhiệm vụ `jar` cuối cùng phụ thuộc vào `buildFront`, nghĩa là mỗi lần xây dựng JAR, ứng dụng React đều được xây dựng lại và sao chép vào JAR.

Về cấu trúc thư mục, axelor-base chứa **189 tệp XML miền** (định nghĩa thực thể) và **192 tệp XML giao diện** (định nghĩa giao diện người dùng). Đây là con số ấn tượng cho thấy độ phức tạp của module - nó không chỉ là "nền tảng" đơn giản mà là một ứng dụng nghiệp vụ hoàn chỉnh với đầy đủ thực thể và giao diện người dùng. Cấu trúc thư mục tài nguyên tuân theo quy ước rõ ràng:

```
src/main/resources/
├── apps/                # Cấu hình cấp ứng dụng
├── data-export/         # Mẫu xuất dữ liệu (CSV, Excel)
├── data-import/         # Cấu hình nhập và quy tắc ánh xạ
├── data-init/           # Dữ liệu khởi tạo - tệp CSV
├── domains/             # *** 189 tệp XML miền ***
├── i18n/                # Tệp dịch ngôn ngữ
├── import-configs/      # Cấu hình cho nhập dữ liệu
├── reports/             # Định nghĩa báo cáo BIRT (.rptdesign)
└── views/               # *** 192 tệp XML giao diện ***
```

Thư mục `data-init/` chứa dữ liệu khởi tạo dưới dạng tệp CSV được nhập tự động khi cài đặt lần đầu. Thư mục `i18n/` chứa tệp dịch, cho phép ứng dụng đa ngôn ngữ. Thư mục `reports/` chứa mẫu báo cáo BIRT - BIRT (Business Intelligence and Reporting Tools) là một bộ khung Eclipse cho việc tạo báo cáo phức tạp.

---

### 7. AXELOR-SALE VÀ AXELOR-ACCOUNT - HAI MODULE NGHIỆP VỤ CHÍNH

**Nguồn:** `/modules/axelor-open-suite/axelor-sale/build.gradle`, `/modules/axelor-open-suite/axelor-account/build.gradle` [Từ source code]

**axelor-sale** là module quản lý bán hàng, có một phụ thuộc duy nhất: `api project(":modules:axelor-crm")`. Mẫu này cho thấy kiến trúc phân lớp rõ ràng - module bán hàng không phụ thuộc trực tiếp vào base (mặc dù vẫn nhận được base thông qua phụ thuộc bắc cầu từ CRM), mà phụ thuộc vào CRM. Điều này hợp lý vì quy trình bán hàng thường bắt đầu từ CRM: khách hàng tiềm năng → cơ hội → báo giá → đơn hàng. Module này chứa 31 thực thể miền và 30 tệp giao diện - nhỏ gọn so với base nhưng tập trung vào logic nghiệp vụ bán hàng.

**axelor-account** là module kế toán, phức tạp hơn nhiều với **122 thực thể miền** và **120 tệp giao diện**, khiến nó trở thành module lớn thứ hai sau base. Các phụ thuộc của module kế toán tiết lộ nhiều điều thú vị về chức năng:
- **jdom, xalan, xmlbeans** - Ba thư viện xử lý XML này có thể dùng cho việc sinh và phân tích tệp XML của hóa đơn điện tử, đặc biệt là các chuẩn như ngôn ngữ nghiệp vụ phổ quát (Universal Business Language - UBL), ebXML.
- **bouncycastle (bcprov, bcpkix)** - Nhà cung cấp mã hóa mạnh mẽ, có thể dùng để ký số hóa đơn điện tử, yêu cầu bắt buộc ở nhiều quốc gia như Việt Nam, Italy, Brazil.
- **ical4j** - Tích hợp lịch, có thể dùng cho việc lên lịch thanh toán, nhắc nhở đến hạn hóa đơn.
- **iban4j** - Kiểm tra số tài khoản ngân hàng theo chuẩn số tài khoản ngân hàng quốc tế (International Bank Account Number - IBAN), quan trọng cho thanh toán quốc tế và SEPA trong Liên minh châu Âu.

**Bằng chứng từ mã:**
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

**Giải thích mã:** Từ khóa `api` trong `api project(":modules:axelor-base")` có ý nghĩa: các phụ thuộc của account sẽ phơi bày (expose) module base cho các module khác phụ thuộc vào account. Ngược lại, `implementation` chỉ dùng nội bộ - các module phụ thuộc vào account sẽ không tự động nhận được jdom, xalan, v.v. Đây là thực hành tốt để giảm liên kết và tránh địa ngục phụ thuộc (dependency hell).

Module account cũng có thư mục đặc biệt `l10n/` (địa phương hóa - localization) chứa dữ liệu theo từng quốc gia như bảng tài khoản kế toán, cấu hình thuế, đây là yêu cầu thiết yếu cho một hệ thống kế toán quốc tế vì mỗi nước có quy định kế toán và thuế khác nhau.

---

### 8. MÔ HÌNH MIỀN - CƠ CHẾ ĐỊNH NGHĨA THỰC THỂ BẰNG XML

**Nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Address.xml`, `Company.xml` [Từ source code]

Axelor sử dụng một cách tiếp cận độc đáo: thay vì định nghĩa các thực thể JPA trực tiếp bằng chú thích Java, Axelor dùng **tệp XML miền** như nguồn chân lý. Đây là một ví dụ về phát triển hướng mô hình (Model-Driven Development - MDD) - các lập trình viên định nghĩa model ở mức trừu tượng cao (lược đồ XML), sau đó trình sinh mã của Axelor tự động sinh ra các lớp thực thể Java với đầy đủ chú thích JPA, getter/setter, equals/hashCode, toString.

XML miền tuân theo một lược đồ chuẩn (`domain-models_7.4.xsd`) với không gian tên `http://axelor.com/xml/ns/domain-models`. Mỗi tệp XML có thể chứa nhiều định nghĩa thực thể, và mỗi thực thể có thể có nhiều loại trường: `<string>`, `<integer>`, `<decimal>`, `<boolean>`, `<date>`, `<datetime>`, `<many-to-one>` (khóa ngoại), `<one-to-many>` (quan hệ ngược), `<many-to-many>`.

Điểm đặc biệt của XML miền Axelor là các thuộc tính bổ sung cung cấp siêu dữ liệu phong phú hơn nhiều so với JPA thuần túy:
- **namecolumn="true"** - Đánh dấu trường này là tên hiển thị của thực thể, dùng khi hiển thị trong danh sách thả xuống, tham chiếu.
- **search="field1,field2,field3"** - Danh sách các trường được dùng cho tìm kiếm toàn văn bản khi người dùng gõ vào hộp tìm kiếm.
- **massUpdate="true"** - Cho phép cập nhật hàng loạt trường này cho nhiều bản ghi cùng lúc.
- **cacheable="true"** - Bật bộ đệm cấp thực thể, các thể hiện của thực thể này sẽ được đệm trong bộ đệm cấp hai của Hibernate.
- **readonly="true"** - Trường chỉ đọc, không thể cập nhật sau khi tạo.
- **track** - Tính năng nhật ký kiểm toán, tự động ghi lại lịch sử thay đổi của trường, quan trọng cho tuân thủ và gỡ lỗi.
- **showIf/hideIf** - Hiển thị có điều kiện, trường chỉ hiện khi điều kiện được thỏa mãn.
- **extra-code** - Cho phép nhúng mã Java trực tiếp trong XML bằng phần CDATA, thường dùng để định nghĩa hằng số.

**Bằng chứng từ mã (thực thể Address):**
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

**Giải thích mã:** Thực thể Address này minh họa nhiều khái niệm quan trọng. Trường `country` là quan hệ nhiều-một với `required="true"`, nghĩa là mỗi địa chỉ phải thuộc về một quốc gia. Trường `streetName` có thuộc tính `help` - văn bản này sẽ hiển thị như chú giải công cụ khi người dùng rê chuột, giúp hướng dẫn người dùng. Trường `city` có `massUpdate="true"` - trường hợp sử dụng là khi hợp nhất hai thành phố, quản trị viên có thể chọn nhiều địa chỉ và cập nhật thành phố cho tất cả cùng lúc. Các trường `latit` và `longit` dùng độ chính xác cao (38,18) để lưu tọa độ GPS chính xác. Trường `fullName` được đánh dấu `namecolumn="true"` nên khi hiển thị Address trong danh sách thả xuống (ví dụ: chọn địa chỉ giao hàng), sẽ hiển thị fullName thay vì ID. Thuộc tính `search` liệt kê các trường sẽ được tìm kiếm khi người dùng gõ vào hộp tìm kiếm - điều này cho phép tìm địa chỉ theo nhiều trường khác nhau.

**Bằng chứng từ mã (thực thể Company với bộ đệm và theo dõi):**
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

**Giải thích mã:** Thực thể Company có `cacheable="true"` ở cấp thực thể - điều này có ý nghĩa lớn về hiệu năng. Vì các bản ghi Company thường được tham chiếu rất nhiều (mỗi hóa đơn, đơn hàng, đơn mua đều có company), việc đệm chúng trong bộ đệm cấp hai giúp giảm đáng kể số lượng truy vấn cơ sở dữ liệu. Hai trường `name` và `code` đều có `unique="true"` - Hibernate sẽ tạo ràng buộc duy nhất trong cơ sở dữ liệu để đảm bảo không có hai công ty trùng tên hoặc mã. Trường `bankDetailsList` là quan hệ một-nhiều với `mappedBy="company"` - đây là phía ngược của quan hệ, nghĩa là thực thể BankDetails sẽ có trường `company` là khóa ngoại.

Khối `<extra-code>` cho phép nhúng các hằng số Java trực tiếp vào lớp thực thể được sinh. Kỹ thuật này rất hữu ích để tránh số ma thuật - thay vì dùng `company.setCategory(1)`, mã có thể dùng `company.setCategory(Company.CATEGORY_CUSTOMER)`, dễ đọc và ít lỗi hơn.

Khối `<track>` định nghĩa nhật ký kiểm toán: mỗi khi cập nhật một Company và các trường name, code, hoặc currency thay đổi, hệ thống sẽ tự động ghi lại: ai thay đổi, thay đổi lúc nào, giá trị cũ là gì, giá trị mới là gì. Tính năng này cực kỳ quan trọng cho tuân thủ và điều tra khi có sự cố.

> ⚠️ **Suy luận:** Tại sao Axelor chọn XML thay vì chú thích Java? Có nhiều lý do: (1) XML có thể được phân tích và kiểm tra ở thời điểm xây dựng bằng lược đồ XSD, phát hiện lỗi sớm hơn. (2) Trình sinh mã có thể đọc XML dễ dàng hơn phân tích chú thích Java. (3) Các nhà phân tích nghiệp vụ không biết Java vẫn có thể đọc và xem xét model XML. (4) XML cho phép mở rộng và ghi đè dễ dàng qua XPath. Tuy nhiên, nhược điểm là XML dài dòng hơn và không có an toàn kiểu lúc viết (IDE không tự động hoàn thành được như Java).

---

### 9. ĐỊNH NGHĨA GIAO DIỆN - XÂY DỰNG GIAO DIỆN BẰNG XML

**Nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/views/Address.xml` [Từ source code]

Tương tự như mô hình miền, giao diện người dùng trong Axelor cũng được định nghĩa hoàn toàn bằng XML thay vì HTML/JSP/Thymeleaf. Axelor cung cấp một ngôn ngữ chuyên dụng dạng XML cho việc xây dựng giao diện người dùng, với các loại giao diện khác nhau phù hợp với từng màn hình nghiệp vụ:

**Các loại giao diện chính:**
- **`<grid>`** - Giao diện danh sách hoặc bảng, hiển thị nhiều bản ghi trong một bảng với cột, sắp xếp, lọc. Đây là giao diện đầu tiên người dùng thấy khi nhấp vào mục menu.
- **`<form>`** - Giao diện chi tiết hoặc biểu mẫu, hiển thị chi tiết một bản ghi duy nhất với đầy đủ trường, được tổ chức trong các bảng điều khiển, tab. Khi người dùng nhấp vào một hàng trong lưới, giao diện biểu mẫu sẽ mở.
- **`<panel-include>`** - Bao gồm và tái sử dụng các bảng điều khiển từ giao diện khác, giúp tránh trùng lặp.

Hệ thống giao diện Axelor cung cấp nhiều tính năng phức tạp thông qua các thuộc tính XML:

**Móc nối vòng đời hành động:**
- `onLoad` - Thực thi các hành động khi giao diện được tải lần đầu, thường dùng để đặt giá trị mặc định hoặc tải dữ liệu liên quan.
- `onSave` - Thực thi trước khi lưu bản ghi, thường dùng cho kiểm tra hoặc chuyển đổi dữ liệu.
- `onNew` - Thực thi khi người dùng nhấp nút "Mới" để tạo bản ghi mới, dùng để khởi tạo trường.
- `onCopy` - Thực thi khi người dùng sao chép một bản ghi, có thể xóa hoặc sửa một số trường.
- `onChange` - Thực thi khi một trường thay đổi, dùng cho tính toán thời gian thực hoặc cập nhật liên hoàn.

**Các loại hành động:**
Axelor định nghĩa nhiều loại hành động, mỗi loại phục vụ một mục đích khác nhau:
- `action-group-*` - Nhóm nhiều hành động lại và thực thi theo thứ tự.
- `action-*-method-*` - Gọi một phương thức Java trong lớp điều khiển, dùng khi cần logic nghiệp vụ phức tạp không thể biểu diễn bằng XML.
- `action-*-record-*` - Thao tác với bản ghi: đặt giá trị trường, xóa trường, sao chép giá trị. Dùng cho logic đơn giản như "khi chọn khách hàng, tự động điền địa chỉ".
- `action-*-attrs-*` - Cập nhật thuộc tính giao diện người dùng: ẩn/hiện trường, làm chỉ đọc, thay đổi trạng thái bắt buộc, cập nhật tiêu đề/lớp CSS. Dùng cho logic giao diện có điều kiện.
- `action-*-validate-*` - Quy tắc kiểm tra, hiển thị thông báo lỗi/cảnh báo/thông tin khi điều kiện không thỏa mãn.

**Tính năng điều khiển giao diện người dùng:**
- **Hiển thị có điều kiện:** `showIf`, `hideIf`, `readonlyIf` - Các trường có thể tự động ẩn/hiện hoặc chỉ đọc dựa trên biểu thức. Ví dụ: `showIf="status == 'confirmed'"` chỉ hiển thị trường khi đơn hàng đã xác nhận.
- **Hệ thống bố cục:** `colSpan` - Axelor dùng hệ thống lưới 12 cột, mỗi trường có thể trải nhiều cột.
- **Thanh công cụ tùy chỉnh:** `<toolbar>` cho phép thêm nút tùy chỉnh vào giao diện lưới, ví dụ "Nhập hàng loạt", "Tạo báo cáo".
- **Ràng buộc phía khách hàng:** `x-bind` - Ràng buộc dữ liệu hai chiều với chuyển đổi, ví dụ `{{fieldName|uppercase}}` tự động viết hoa đầu vào.
- **Quyền:** `canNew`, `canEdit`, `canRemove` - Kiểm soát xem người dùng có quyền tạo/sửa/xóa bản ghi hay không.

**Bằng chứng từ mã (giao diện lưới với thanh công cụ tùy chỉnh):**
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

**Giải thích mã:** Giao diện lưới này hiển thị danh sách địa chỉ. Thuộc tính `model` chỉ định lớp thực thể với tên gói đầy đủ. Khối `<toolbar>` thêm một nút "Kiểm tra trùng lặp" vào thanh công cụ phía trên lưới - khi nhấp, nó gọi hành động `action-base-method-show-duplicate`. Trường `fullName` được hiển thị trong lưới. Một nút đặc biệt `mapBtn` với biểu tượng Font Awesome `fa-map-marker` xuất hiện trên mỗi hàng, khi nhấp sẽ mở giao diện bản đồ để hiển thị vị trí địa chỉ.

**Bằng chứng từ mã (giao diện biểu mẫu với logic có điều kiện và móc nối vòng đời):**
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

**Giải thích mã:** Biểu mẫu này có `width="large"` - biểu mẫu sẽ chiếm nhiều không gian màn hình hơn độ rộng mặc định. Thuộc tính `onLoad="action-group-base-address-onload"` chỉ định một nhóm hành động chạy khi biểu mẫu tải, có thể dùng để đặt quốc gia mặc định dựa trên vị trí công ty của người dùng. `onSave="action-group-base-address-onsave"` chạy trước khi lưu, có thể kiểm tra địa chỉ hoặc mã hóa địa lý để lấy tọa độ.

Trường `isInvoicingAddr` có hiển thị có điều kiện: `showIf="$popup() && id == null"` - chỉ hiện khi biểu mẫu được mở trong chế độ cửa sổ bật lên VÀ đang tạo mới (id chưa lưu). Hàm `$popup()` là hàm biểu thức tích hợp của Axelor. `&amp;` là cách viết `&&` trong XML (phải thoát).

Bảng điều khiển "actionsPanel" có `sidebar="true"` - đây là bảng điều khiển thanh bên thường xuất hiện bên phải biểu mẫu, chứa các nút hành động. Nhóm nút có `hideIf="$popup()"` - ẩn khi ở chế độ cửa sổ bật lên, và `readonlyIf="!id"` - vô hiệu hóa khi bản ghi chưa được lưu. Trường hợp sử dụng: không thể xem bản đồ cho địa chỉ chưa lưu vì chưa có tọa độ.

**Bằng chứng từ mã (chuyển đổi phía khách hàng):**
```xml
<field name="department" x-bind="{{department|uppercase}}" colSpan="12"/>
<field name="buildingNumber" onChange="action-address-record-change-streetName"
  x-bind="{{buildingNumber|uppercase}}" colSpan="12"/>
```

**Giải thích mã:** Thuộc tính `x-bind` thực hiện ràng buộc dữ liệu hai chiều với chuyển đổi bằng ống. `{{department|uppercase}}` tự động viết hoa mọi đầu vào vào trường department - người dùng gõ "sales" sẽ tự động thành "SALES". Trường `buildingNumber` vừa có ràng buộc viết hoa VÀ hành động `onChange` - khi người dùng nhập số nhà, nó viết hoa, sau đó kích hoạt hành động `action-address-record-change-streetName` có thể tự động xây dựng tên đường từ các trường số nhà + tên đường.

**Bằng chứng từ mã (tích hợp điều khiển Java):**
```xml
<button name="validateBtn" title="Validate"
  onClick="com.axelor.apps.base.web.AddressController:validate,save"/>
```

**Giải thích mã:** Thuộc tính `onClick` có thể tham chiếu trực tiếp một phương thức điều khiển Java bằng cú pháp `package.ClassName:methodName`. Có thể xâu chuỗi nhiều hành động bằng dấu phẩy: `,save` là hành động tích hợp để lưu bản ghi. Luồng sẽ là: gọi `AddressController.validate()` (có thể kiểm tra định dạng địa chỉ, kiểm tra với API dịch vụ bưu điện), nếu kiểm tra thành công thì lưu bản ghi.

> ⚠️ **Suy luận:** Tại sao Axelor dùng XML cho giao diện? (1) Cách tiếp cận khai báo dễ duy trì hơn mã giao diện người dùng mệnh lệnh. (2) Những người không phải lập trình viên (nhà phân tích nghiệp vụ) có thể tùy chỉnh giao diện. (3) Giao diện có thể được mở rộng và ghi đè bởi các module khác qua XPath. (4) Công cụ giao diện Axelor có thể sinh mã giao diện tối ưu từ XML. Nhược điểm là: (1) XML dài dòng. (2) Khó gỡ lỗi vì không có dấu vết ngăn xếp như mã thông thường. (3) Hỗ trợ IDE hạn chế (không tự động hoàn thành). (4) Logic giao diện phức tạp vẫn phải dùng phương thức điều khiển Java.

---

### 10. TẬP TIN CẤU HÌNH ỨNG DỤNG CHÍNH

**Nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Tệp `axelor-config.properties` với 518 dòng là trung tâm cấu hình của toàn bộ ứng dụng Axelor. Tệp này tổng hợp mọi khía cạnh: cơ sở dữ liệu, bảo mật, hiệu năng, giao diện người dùng, báo cáo, mã hóa.

#### 10.1. Cấu hình cơ sở dữ liệu và Hibernate

```properties
db.default.driver = org.postgresql.Driver
db.default.ddl = update
db.default.url = jdbc:postgresql://10.10.1.31:5432/axelor_demo_staging
db.default.user = odooclau
db.default.password = odoo
```

Kết nối cơ sở dữ liệu sử dụng trình điều khiển PostgreSQL kết nối đến địa chỉ IP `10.10.1.31` cổng `5432`, tên cơ sở dữ liệu `axelor_demo_staging`. Điểm đáng chú ý nhất là `db.default.ddl = update` - đây là chiến lược tự động ngôn ngữ định nghĩa dữ liệu (Data Definition Language - DDL) của Hibernate. Với cài đặt này, mỗi khi ứng dụng khởi động, Hibernate sẽ so sánh cấu trúc của các lớp thực thể Java với lược đồ hiện tại trong cơ sở dữ liệu. Nếu có sự khác biệt (ví dụ: thêm trường mới vào thực thể), Hibernate sẽ tự động thực thi các câu lệnh `ALTER TABLE` để cập nhật lược đồ.

> ⚠️ **Suy luận:** Cách tiếp cận này rất tiện lợi cho phát triển vì không cần viết tập lệnh di chuyển thủ công, nhưng cực kỳ nguy hiểm cho môi trường sản xuất vì: (1) Không có kiểm soát phiên bản của thay đổi lược đồ. (2) Không thể hoàn tác nếu di chuyển thất bại. (3) Nguy cơ mất dữ liệu nếu xóa cột. Thực hành tốt nhất trong sản xuất là dùng `ddl=validate` (chỉ kiểm tra, không tự động sửa) và dùng công cụ di chuyển như Flyway hoặc Liquibase, nhưng Axelor không có công cụ di chuyển này.

```properties
javax.persistence.sharedCache.mode = ENABLE_SELECTIVE
hibernate.search.default.directory_provider = none
hibernate.hikari.minimumIdle = 5
hibernate.hikari.maximumPoolSize = 20
hibernate.hikari.idleTimeout = 300000
```

Bộ đệm chia sẻ JPA (L2 cache) được đặt `ENABLE_SELECTIVE` nghĩa là chỉ các thực thể được đánh dấu `@Cacheable` (hoặc trong Axelor là `cacheable="true"` trong XML miền) mới được đệm. Đây là cách tiếp cận bảo thủ và an toàn - tránh đệm toàn bộ (tốn bộ nhớ) nhưng vẫn đệm những thực thể được truy cập thường xuyên như công ty, tiền tệ, quốc gia.

Nhóm kết nối HikariCP được cấu hình với tối thiểu 5 kết nối nhàn rỗi và tối đa 20 kết nối. Thời gian chờ nhàn rỗi là 300.000 mili giây (5 phút) - các kết nối không sử dụng quá 5 phút sẽ bị đóng để giải phóng tài nguyên. Con số 20 kết nối tối đa là bảo thủ, phù hợp cho triển khai nhỏ đến trung bình.

> ⚠️ **Suy luận:** Tại sao cần nhóm kết nối? Các kết nối cơ sở dữ liệu rất "đắt" để tạo (mất vài chục mili giây), nếu mỗi yêu cầu đều tạo kết nối mới sẽ rất chậm. Nhóm tái sử dụng kết nối, giảm độ trễ và tải cơ sở dữ liệu.

#### 10.2. Thông tin ứng dụng

```properties
application.name = Axelor Open Suite
application.version = 8.5.10
application.mode = dev
application.theme = Modern
application.locale = fr
```

Ứng dụng đang chạy ở `mode = dev` (chế độ phát triển) - ở chế độ này, Axelor có thể bật tải lại nóng (hot reload), thông báo lỗi chi tiết, vô hiệu hóa bộ đệm để dễ gỡ lỗi. Khi triển khai sản xuất phải đổi sang `mode = prod`. Chủ đề là "Modern" - Axelor có nhiều chủ đề giao diện người dùng. Ngôn ngữ mặc định là `fr` (tiếng Pháp) - toàn bộ giao diện người dùng mặc định sẽ hiển thị tiếng Pháp (Axelor là công ty Pháp).

#### 10.3. Bảo mật và bảo vệ chèn SQL

```properties
#application.domain-blocklist-pattern = (\\(\\s*(SELECT|DELETE|UPDATE)\\s+)|query_to_xml
```

Lọc biểu thức miền - đây là một biện pháp bảo mật quan trọng. Axelor cho phép người dùng (đặc biệt là quản trị viên) định nghĩa biểu thức lọc cho bản ghi. Nếu không kiểm tra cẩn thận, các biểu thức này có thể chứa tấn công chèn SQL. Mẫu này chặn các biểu thức chứa từ khóa SQL nguy hiểm như SELECT, DELETE, UPDATE bên trong dấu ngoặc đơn, hoặc hàm `query_to_xml` có thể rò rỉ dữ liệu. Hiện đang được chú thích - có thể đang dùng mẫu mặc định hoặc đã di chuyển sang cách tiếp cận kiểm tra khác.

```properties
#application.script.cache.size = 1000
#application.script.cache.expire-time = 20
```

Bộ đệm kịch bản Groovy - Axelor cho phép viết kịch bản Groovy trong các hành động, nhiệm vụ BPM. Các kịch bản Groovy phải được biên dịch trước khi thực thi, quá trình này tốn CPU. Bộ đệm các kịch bản đã biên dịch với kích thước 1000 (1000 kịch bản duy nhất) và thời gian hết hạn 20 phút giúp tránh biên dịch lại.

#### 10.4. Cấu hình giao diện

```properties
view.max-tabs = 10
view.menubar.location = both
```

Giao diện người dùng chỉ cho phép mở tối đa 10 tab cùng lúc - tránh người dùng mở quá nhiều tab làm chậm trình duyệt. Thanh menu có thể hiển thị ở `left` (thanh bên), `top` (thanh trên), hoặc `both` - cấu hình này cho phép người dùng chọn cách điều hướng ưa thích.

#### 10.5. Cấu hình BPM Studio (QUAN TRỌNG)

```properties
studio.bpm.logging = false
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
studio.bpm.history.time.to.live = P180D
```

Đây là phát hiện cực kỳ quan trọng - Axelor có tích hợp công cụ quản lý quy trình nghiệp vụ! Công cụ BPM có nhóm kết nối riêng biệt với nhóm ứng dụng chính. Kết nối nhàn rỗi: 10, kết nối hoạt động tối đa: 50 - lớn hơn nhiều so với nhóm chính (20), cho thấy các quy trình BPM có tính đồng thời cao.

`history.time.to.live = P180D` - định dạng thời lượng ISO 8601, P180D = 180 ngày. Các thể hiện quy trình đã hoàn thành và dữ liệu lịch sử sẽ được giữ trong 180 ngày rồi tự động dọn dẹp. Điều này quan trọng vì các bảng BPM phình to rất nhanh - mỗi quy trình tạo nhiều hàng (thể hiện quy trình, hoạt động, biến, lịch sử), nếu không dọn dẹp định kỳ cơ sở dữ liệu sẽ đầy. 180 ngày là cân bằng tốt giữa yêu cầu kiểm toán và kích thước cơ sở dữ liệu.

> ⚠️ **Suy luận:** Dựa trên các mẫu cấu hình (nhóm kết nối, TTL lịch sử), công cụ BPM có thể là Camunda - một trong những công cụ BPM phổ biến nhất cho Java. Camunda có cấu hình tương tự.

#### 10.6. Xác thực và đăng nhập một lần

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

Axelor hỗ trợ đa dạng phương thức xác thực: Google OAuth, Keycloak (quản lý danh tính doanh nghiệp), SAML 2.0 (chuẩn đăng nhập một lần), LDAP (tích hợp Active Directory), CAS (dịch vụ xác thực tập trung - Central Authentication Service). Đây là danh sách ấn tượng - phần lớn ứng dụng nghiệp vụ chỉ hỗ trợ một hoặc hai phương thức.

`auth.user.provisioning = none` kiểm soát việc tự động tạo tài khoản người dùng. Các tùy chọn có thể là: `create` (tự động tạo người dùng trong cơ sở dữ liệu khi đăng nhập qua đăng nhập một lần lần đầu), `link` (liên kết tài khoản đăng nhập một lần với người dùng hiện có), `none` (không tự động, phải tạo người dùng trước). `none` là cách tiếp cận an toàn nhất nhưng kém tiện lợi.

#### 10.7. Quản lý dữ liệu

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

Tải lên tệp được giới hạn 5MB - đủ cho tài liệu, hình ảnh nhưng không đủ cho video. Thư mục tải lên là `{java.io.tmpdir}/axelor` - `{java.io.tmpdir}` là chỗ giữ chỗ được thay thế bằng thư mục tạm thời của hệ thống. Các mẫu danh sách cho phép/chặn kiểm soát loại tệp: có thể cho phép XML, HTML, hình ảnh, PDF nhưng chặn SVG (SVG có thể chứa JavaScript - nguy cơ tấn công kịch bản chéo trang - XSS).

Kích thước tối đa xuất dữ liệu 5000 bản ghi - giới hạn này tránh người dùng xuất hàng triệu bản ghi làm cơ sở dữ liệu quá tải hoặc tạo tệp quá lớn không thể mở. Mã hóa UTF-8 đảm bảo hỗ trợ đầy đủ các ký tự Unicode (tiếng Việt, tiếng Trung, biểu tượng cảm xúc).

`data.import.demo-data = false` - không nhập dữ liệu demo khi khởi động. Dữ liệu demo hữu ích cho kiểm thử nhưng phải vô hiệu hóa trong sản xuất.

#### 10.8. Bộ lập lịch Quartz

```properties
#quartz.enable = true
#quartz.thread-count = 3
```

Quartz là thư viện lập lịch công việc cho Java, cho phép chạy các nhiệm vụ theo lịch hoặc định kỳ. Số lượng luồng = 3 nghĩa là tối đa 3 công việc có thể chạy đồng thời. Con số nhỏ này hợp lý vì các công việc theo lịch thường là nặng - nếu cho phép quá nhiều công việc chạy cùng lúc sẽ quá tải hệ thống.

> ⚠️ **Suy luận:** Ví dụ về các công việc theo lịch: gửi email nhắc nhở, tạo báo cáo hàng ngày, dọn dẹp dữ liệu cũ, đồng bộ với hệ thống bên ngoài.

#### 10.9. Tuân thủ GDPR và nhật ký kiểm toán

```properties
hibernate.session_factory.interceptor = com.axelor.apps.base.tracking.GlobalAuditInterceptor
```

Đây là một bộ chặn Hibernate tùy chỉnh được gọi cho mọi thao tác cơ sở dữ liệu (chèn, cập nhật, xóa). `GlobalAuditInterceptor` có thể ghi lại ai đã làm gì và khi nào - quan trọng cho tuân thủ GDPR (quy định của Liên minh châu Âu yêu cầu theo dõi mọi truy cập và sửa đổi dữ liệu cá nhân). Bộ chặn có chi phí hiệu năng (mỗi thao tác phải ghi thêm bản ghi kiểm toán) nhưng cần thiết cho tuân thủ.

#### 10.10. Nhà cung cấp ngữ cảnh

```properties
context.date = com.axelor.apps.base.service.DateService:date
context.appLogo = com.axelor.apps.base.service.user.UserService:getUserActiveCompanyLogoLink
context.app = com.axelor.studio.app.service.AppService
```

Các nhà cung cấp ngữ cảnh tiêm các giá trị động vào biểu thức và mẫu. Ví dụ: trong biểu thức giao diện, có thể dùng `$date` sẽ gọi `DateService.date()` để lấy ngày hiện tại. `$appLogo` trả về địa chỉ URL của logo công ty để hiển thị trong giao diện người dùng. Đây là mẫu tiêm phụ thuộc được áp dụng cho cấu hình.

#### 10.11. Cài đặt ứng dụng Studio

```properties
studio.apps.install = all
```

`studio.apps.install = all` tự động cài đặt tất cả ứng dụng Axelor Studio khi ứng dụng khởi động lần đầu. Thay thế là danh sách tên module phân tách bằng dấu phẩy (ví dụ: `sale,account,stock`). Cài đặt `all` tiện lợi cho phát triển nhưng có thể không mong muốn trong sản xuất nếu chỉ cần tập con chức năng.

---

## Những điều không tìm thấy trong mã nguồn

Sau quá trình nghiên cứu toàn diện mã nguồn, có một số thành phần và công nghệ mà chúng tôi không tìm thấy, điều này cũng tiết lộ nhiều thông tin về kiến trúc của Axelor:

1. **Không tìm thấy phụ thuộc bộ khung Spring** - Axelor không sử dụng Spring Boot hay Spring Framework, điều này ngạc nhiên vì phần lớn ứng dụng doanh nghiệp Java hiện đại đều dùng Spring. Axelor có bộ khung độc quyền riêng với tiêm phụ thuộc dựa trên Google Guice (nhẹ hơn Spring nhiều). [Từ source code: không có spring-boot-starter hay spring-context trong các phụ thuộc]

2. **Không tìm thấy tệp `persistence.xml`** - Chuẩn JPA thường yêu cầu tệp này để cấu hình đơn vị lưu trữ, nhưng Axelor quản lý cấu hình JPA qua bộ khung riêng, có thể theo chương trình hoặc qua axelor-config.properties. [Từ source code: tìm kiếm toàn bộ dự án không có tệp này]

3. **Không tìm thấy Liquibase hoặc Flyway** - Đây là hai công cụ di chuyển phổ biến nhất cho quản lý phiên bản cơ sở dữ liệu Java. Việc thiếu chúng nghĩa là Axelor hoàn toàn dựa vào cập nhật tự động DDL của Hibernate (`ddl=update`), đây là một quyết định kiến trúc có rủi ro cho môi trường sản xuất. [Từ source code: không có phụ thuộc và không có thư mục tập lệnh di chuyển]

4. **Không tìm thấy cấu hình Docker** - Không có `Dockerfile`, `docker-compose.yml` ở gốc dự án. [Từ source code: không có ở cấp gốc]

5. **Không tìm thấy tệp cấu hình tích hợp liên tục/triển khai liên tục (CI/CD)** - Không có `.gitlab-ci.yml`, `.github/workflows`, `Jenkinsfile` ở gốc dự án. [Từ source code: không có ở cấp gốc]

6. **Không tìm thấy bộ khung giao diện hiện đại (ngoài React map-viewer)** - Không có dự án Angular, Vue.js ở giao diện. Giao diện người dùng chính của Axelor có thể dùng bộ khung JavaScript tùy chỉnh (độc quyền) được sinh từ XML giao diện, hoặc có thể là bộ khung kế thừa. React chỉ dùng cho thành phần trình xem bản đồ. [Từ source code: chỉ có React trong axelor-base/map-viewer]

7. **Không tìm thấy hệ thống hàng đợi thông điệp** - Không có phụ thuộc RabbitMQ, Kafka, ActiveMQ. Xử lý bất đồng bộ có thể dựa hoàn toàn vào bộ lập lịch Quartz, không có kiến trúc thực sự hướng thông điệp. [Từ source code: không có JMS hoặc thư viện nhắn tin ngoài addon axelor-message]

8. **Không tìm thấy công cụ giám sát/quan sát tích hợp** - Không có tích hợp Prometheus, Micrometer, ELK stack. [Từ source code: không có thư viện số liệu]

9. **Không tìm thấy công cụ độ bao phủ kiểm thử** - Có phụ thuộc JUnit nhưng không có Jacoco, Cobertura cho báo cáo độ bao phủ kiểm thử. [Từ source code: chỉ có mockito và nền tảng junit]

---

## Câu hỏi mở cần điều tra thêm

Sau bước nghiên cứu đầu tiên này, xuất hiện nhiều câu hỏi kỹ thuật cần điều tra sâu hơn ở các bước tiếp theo:

1. **Cơ chế sinh mã chi tiết:** Các XML miền được biến đổi thành các lớp thực thể Java bằng cách nào? Là bộ xử lý chú thích chạy lúc biên dịch, hay là plugin Gradle sinh mã trước khi biên dịch? Mã được sinh được lưu ở đâu? Làm thế nào để kiểm tra các thực thể được sinh khi gỡ lỗi?

2. **Công cụ hiển thị giao diện:** Các XML giao diện được hiển thị ra HTML/JavaScript thế nào? Bộ khung giao diện là gì - bộ khung tùy chỉnh, GWT kế thừa (Google Web Toolkit), hoặc một công cụ mẫu nào đó? React chỉ dùng cho bản đồ, vậy phần còn lại dùng gì?

3. **Công cụ BPM cụ thể:** Cấu hình cho thấy có BPM nhưng không rõ công cụ. Cần tìm trong addon axelor-studio (phụ thuộc nhị phân) xem có tham chiếu đến Camunda, Activiti, Flowable hay triển khai tùy chỉnh không? Có hỗ trợ chuẩn BPMN 2.0 hay là ký hiệu độc quyền?

4. **Triển khai mẫu kho lưu trữ:** XML miền không định nghĩa các phương thức kho lưu trữ như `findByName()`. Vậy các kho lưu trữ được sinh tự động với các phương thức CRUD? Hay các lập trình viên phải viết các lớp kho lưu trữ tùy chỉnh? Có ngôn ngữ chuyên dụng truy vấn (Query DSL) hay API tiêu chí nào không?

5. **Công cụ thực thi hành động:** Khi nút trong XML giao diện kích hoạt `action-method`, hệ thống định tuyến yêu cầu tới phương thức điều khiển Java như thế nào? Có ánh xạ dựa trên quy ước (như Rails) hay dựa trên chú thích (như Spring)? Hiệu năng của phân phối hành động ra sao với hàng nghìn hành động?

6. **MetaJsonModel và mô hình tùy chỉnh:** Yêu cầu nghiên cứu đề cập "mô hình tùy chỉnh" và "mô hình JSON". Cần tìm các thực thể như `MetaJsonRecord`, `MetaJsonModel`, `MetaJsonField` để hiểu cơ chế. Dữ liệu được tuần tự hóa/giải tuần tự như thế nào? Hiệu năng truy vấn với các cột JSON?

7. **Mã nguồn axelor-studio:** Addon Studio là phụ thuộc nhị phân (`axelor-studio:3.5.1`), không có nguồn trong kho lưu trữ open-suite. Đây có phải là addon trả phí/độc quyền? Có tài liệu nào về API Studio không? Có thể dịch ngược để nghiên cứu không?

8. **Kế thừa và mở rộng giao diện:** Đã thấy `extension="true"` trong XML giao diện. Cơ chế cụ thể là vá dựa trên XPath? Có giải quyết xung đột nào khi nhiều module mở rộng cùng giao diện? Thứ tự mở rộng?

9. **Triển khai đa thuê bao:** Cấu hình có đề cập `application.multi-tenancy` nhưng đang bị vô hiệu hóa. Khi bật, triển khai là mỗi lược đồ (mỗi thuê bao một lược đồ cơ sở dữ liệu) hay mỗi hàng (bảng chia sẻ với cột tenant_id)? Cách ly bảo mật thế nào?

10. **Các lựa chọn thay thế tìm kiếm Hibernate:** Cấu hình có `hibernate.search.default.directory_provider = none`. Vậy tìm kiếm toàn văn bản được triển khai bằng gì? Tìm kiếm toàn văn bản PostgreSQL? Tích hợp Elasticsearch? Hay không có khả năng tìm kiếm?

11. **Các điểm cuối API được sinh:** Mỗi thực thể miền tự động có các điểm cuối REST API không? Định dạng là gì - `/api/com.axelor.apps.base.db.Address` hay dạng ngắn `/api/address`? Có tài liệu Swagger/OpenAPI được sinh tự động không?

12. **Độ chi tiết hệ thống quyền:** Cấu hình có đề cập `application.permission.disable-action` và `disable-relational-field`. Điều này cho thấy hệ thống quyền rất chi tiết. Cần nghiên cứu cách định nghĩa quyền - XML, cơ sở dữ liệu, chú thích? Tích hợp với giao diện thế nào?

13. **Các tùy chọn triển khai:** Axelor triển khai như tệp WAR lên Tomcat? Hay có máy chủ nhúng (như Spring Boot)? Có tệp JAR độc lập thực thi không? Thân thiện với Kubernetes hay cần máy chủ ứng dụng truyền thống?

---

## Sơ đồ kiến trúc tổng quan

Dựa trên các phát hiện từ phân tích mã nguồn, đây là sơ đồ tổng quan về cấu trúc dự án Axelor ERP:

```
axelor-erp/                                    (Dự án gốc)
│                                              Phiên bản: 8.5.10
│                                              Java: 21 (OpenJDK)
│                                              Xây dựng: Gradle 8.x + Đa module
│
├── build.gradle                               Cấu hình xây dựng gốc
│   └── allprojects {
│         group = 'com.axelor.apps'
│         java toolchain = Java 21
│       }
│
├── settings.gradle                            Phát hiện module động
│   └── Duyệt thư mục modules/
│       → Tự động thêm module có build.gradle
│
├── gradle.properties                          Cài đặt JVM
│   └── -Xmx2g heap
│       xây dựng song song được bật
│
├── src/main/resources/
│   └── axelor-config.properties               [518 DÒNG] Cấu hình chính
│       ├── Cơ sở dữ liệu: PostgreSQL + Hibernate
│       ├── Nhóm kết nối: HikariCP (5-20 kết nối)
│       ├── BPM: Nhóm riêng (10-50 kết nối)
│       ├── Xác thực: Google, Keycloak, SAML, LDAP, CAS
│       ├── Bảo mật: Bộ chặn kiểm toán GDPR
│       └── Studio: Tự động cài đặt tất cả ứng dụng
│
└── modules/
    └── axelor-open-suite/                     Git Submodule (v8.5.9)
        │
        ├── libs.gradle                        Phụ thuộc bên ngoài
        │   ├── axelor-studio:3.5.1           (Addon nhị phân - Nền tảng không mã + BPM)
        │   ├── axelor-message:3.3.0          (Addon nhị phân - Nhắn tin)
        │   ├── axelor-utils:3.5.0            (Addon nhị phân - Tiện ích)
        │   ├── groovy:3.0.23                 (Ngôn ngữ kịch bản)
        │   ├── pac4j-core:5.7.7              (Đăng nhập một lần đa nhà cung cấp)
        │   ├── pdfbox, openpdf               (Xử lý PDF)
        │   ├── bouncycastle                  (Mã hóa)
        │   └── ical4j, iban4j, zxing, ...    (Thư viện nghiệp vụ)
        │
        ├── version.gradle / version.txt       Phiên bản: 8.5.9
        │
        ├── axelor-base/                       ★ MODULE NỀN TẢNG ★
        │   ├── build.gradle                   Phụ thuộc:
        │   │   ├── axelor-studio             - Nền tảng không mã
        │   │   ├── axelor-message            - Nhắn tin
        │   │   ├── axelor-utils              - Tiện ích
        │   │   ├── Node.js 22.17.1           - Xây dựng giao diện
        │   │   └── React 19.1                - Thành phần trình xem bản đồ
        │   │
        │   ├── src/main/java/                 Logic nghiệp vụ Java
        │   │   └── com/axelor/apps/base/
        │   │       ├── service/              Lớp dịch vụ
        │   │       ├── web/                  Điều khiển web
        │   │       └── repo/                 Kho lưu trữ tùy chỉnh
        │   │
        │   ├── src/main/resources/
        │   │   ├── domains/                   ★★★ 189 tệp XML miền ★★★
        │   │   │   └── (Định nghĩa thực thể → Sinh mã)
        │   │   ├── views/                     ★★★ 192 tệp XML giao diện ★★★
        │   │   │   └── (Định nghĩa giao diện → Hiển thị bởi bộ khung)
        │   │   ├── data-init/                Dữ liệu khởi tạo CSV
        │   │   ├── reports/                  Mẫu báo cáo BIRT
        │   │   └── i18n/                     Tệp dịch
        │   │
        │   ├── src/main/webapp/               Tài nguyên web
        │   │
        │   └── map-viewer/                    ★ Ứng dụng React ★
        │       ├── package.json              React 19.1 + phụ thuộc
        │       ├── src/                      Thành phần React
        │       └── dist/                     Đầu ra xây dựng → đóng gói vào JAR
        │
        ├── axelor-account/                    Module kế toán
        │   ├── build.gradle                   Phụ thuộc vào: axelor-base
        │   └── src/main/resources/
        │       ├── domains/                   ★ 122 thực thể ★ (lớn thứ hai)
        │       ├── views/                     120 giao diện
        │       ├── l10n/                      Địa phương hóa (theo quốc gia)
        │       └── reports/                   40 báo cáo BIRT
        │
        ├── axelor-sale/                       Module bán hàng
        │   ├── build.gradle                   Phụ thuộc vào: axelor-crm
        │   └── src/main/resources/
        │       ├── domains/                   31 thực thể
        │       └── views/                     30 giao diện
        │
        ├── axelor-crm/                        Module CRM
        ├── axelor-purchase/                   Module mua hàng
        ├── axelor-stock/                      Module kho
        ├── axelor-supplychain/                Chuỗi cung ứng (lớp tích hợp)
        ├── axelor-production/                 Sản xuất
        ├── axelor-human-resource/             Module nhân sự
        ├── axelor-project/                    Quản lý dự án
        ├── axelor-bank-payment/               Ngân hàng & SEPA
        ├── axelor-budget/                     Lập kế hoạch ngân sách
        └── [18 module nghiệp vụ khác...]     Tổng cộng: 27 module
```

**Luồng xử lý chính:**

```
1. THỜI GIAN PHÁT TRIỂN (Giai đoạn xây dựng):
   Tệp XML miền (189 trong base)
       ↓
   Quá trình xây dựng Gradle
       ↓
   Trình sinh mã Axelor (Plugin com.axelor.app:7.4.7)
       ↓
   Các lớp thực thể Java + kho lưu trữ được sinh
       ↓
   Biên dịch Java (Java 21)
       ↓
   Xây dựng React (map-viewer: Node 22.17.1 + Yarn)
       ↓
   Đóng gói JAR (Tất cả tài nguyên được đóng gói)

2. THỜI GIAN CHẠY (Giai đoạn ứng dụng):
   axelor-config.properties → Tải cấu hình
       ↓
   Hibernate khởi tạo với PostgreSQL
       ↓
   ├─ Nhóm kết nối chính: HikariCP (5-20)
   └─ Nhóm kết nối BPM: Riêng biệt (10-50)
       ↓
   Tải module động (27 module)
       ↓
   Hiển thị XML giao diện → Giao diện web
       ↓
   Tương tác người dùng → Hành động (định nghĩa XML)
       ↓
   ├─ action-method → Điều khiển Java
   ├─ action-record → Thao tác bản ghi
   ├─ action-attrs → Cập nhật giao diện
   └─ action-script → Thực thi Groovy
       ↓
   Lớp dịch vụ (Logic nghiệp vụ)
       ↓
   Lớp kho lưu trữ (Truy vấn JPA)
       ↓
   Cơ sở dữ liệu (PostgreSQL) + Bộ đệm (L1 + L2)
```

**Các mẫu kiến trúc chính được xác định:**

1. **Phát triển hướng XML (Phát triển hướng mô hình - MDD):** Các thực thể và giao diện được định nghĩa bằng XML, sau đó trình sinh mã tạo các lớp Java. Cách tiếp cận này cho phép phát triển nhanh và nhà phân tích nghiệp vụ có thể duy trì model.

2. **Gradle đa module với phát hiện động:** Dự án gốc tự động phát hiện các module trong thư mục `modules/`, không cần viết cứng. Có thể mở rộng và dễ thêm module mới.

3. **Kiến trúc Git Submodule:** Logic nghiệp vụ (axelor-open-suite) tách biệt thành submodule, có thể được tạo nhánh và tùy chỉnh độc lập.

4. **Mô hình addon nhị phân:** Các tính năng quan trọng (Studio, BPM, Messaging) được đóng gói thành các addon nhị phân, có thể cho tính linh hoạt cấp phép.

5. **Giao diện lai:** React cho các thành phần phong phú (bản đồ), bộ khung độc quyền cho giao diện chính được sinh từ XML giao diện.

6. **Nhóm kết nối BPM riêng biệt:** Quy trình làm việc BPM có yêu cầu tài nguyên khác ứng dụng chính, cần nhóm riêng để tránh thiếu kết nối.

7. **Bảo mật toàn diện:** Xác thực đa nhà cung cấp (Google, SAML, LDAP, Keycloak), nhật ký kiểm toán GDPR, bảo vệ chèn SQL.

8. **Quy ước hơn cấu hình với cấu hình mở rộng:** Bộ khung có nhiều quy ước (XML miền → thực thể, XML giao diện → giao diện) nhưng vẫn có tệp cấu hình tập trung cho tinh chỉnh.
