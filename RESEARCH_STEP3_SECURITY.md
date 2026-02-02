# BƯỚC 3: PHÂN TÍCH HỆ THỐNG BẢO MẬT VÀ PHÂN QUYỀN - Phân Tích Từ Mã Nguồn

## Phương pháp phân tích

Nghiên cứu kiến trúc bảo mật và hệ thống phân quyền được thực hiện thông qua phân tích các tệp định nghĩa thực thể (Domain XML) liên quan đến xác thực và phân quyền, mã nguồn các dịch vụ xử lý quyền hạn, cùng cấu hình nhà cung cấp xác thực. Phạm vi bao gồm cơ chế xác thực, mô hình phân quyền theo tầng, phân quyền ở cấp đối tượng/trường/bản ghi, và công cụ quản lý quyền hạn.

**Tệp định nghĩa thực thể đã phân tích:**
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/User.xml` - Thực thể người dùng với phần mở rộng nghiệp vụ
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Group.xml` - Thực thể nhóm người dùng
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Role.xml` - Thực thể vai trò
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Permission.xml` - Quyền hạn cấp đối tượng

**Mã nguồn dịch vụ:**
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/auth/service/PermissionServiceImpl.java` - Logic phân quyền
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/auth/service/PermissionAssistantService.java` - Nhập/xuất quyền hạn qua CSV
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/service/pac4j/BaseAuthPac4jUserService.java` - Tích hợp Pac4j

**Phân tích cấu hình:**
- `/src/main/resources/axelor-config.properties` - Nhà cung cấp xác thực, cấu hình phiên làm việc, chính sách mật khẩu

---

## Kết quả chi tiết

### 1. KHUNG BẢO MẬT VÀ KIẾN TRÚC TỔNG QUAN

**Tệp nguồn:** Phân tích câu lệnh nhập (import) trong các lớp dịch vụ, tệp cấu hình [Từ source code]

Axelor áp dụng một cách tiếp cận riêng biệt trong hệ sinh thái ứng dụng Java doanh nghiệp: thay vì sử dụng các khung bảo mật phổ biến như Spring Security (tiêu chuẩn thực tế cho ứng dụng Spring) hay Apache Shiro (khung bảo mật không phụ thuộc nền tảng), Axelor tự xây dựng tầng bảo mật tùy chỉnh (custom security layer) tích hợp chặt chẽ với lõi khung ứng dụng. Quyết định thiết kế này mang lại khả năng kiểm soát tốt hơn đối với mô hình phân quyền phức tạp gồm nhiều cấp (đối tượng, trường, bản ghi), nhưng đổi lại phải tự bảo trì mã bảo mật thay vì dựa vào khung đã được cộng đồng kiểm chứng, đồng thời khó tích hợp với công cụ bảo mật của bên thứ ba vốn được thiết kế cho Spring Security.

Kiến trúc bảo mật của Axelor chia thành hai tầng rõ ràng: tầng xác thực (authentication - "bạn là ai") và tầng phân quyền (authorization - "bạn được làm gì"). Tầng xác thực được triển khai thông qua thư viện Pac4j phiên bản 5.7.7 - một tầng trừu tượng cho phép hỗ trợ nhiều nhà cung cấp xác thực (OAuth 2.0, SAML, LDAP, CAS) mà không cần viết mã riêng cho từng nhà cung cấp. Pac4j là lựa chọn phù hợp cho kịch bản đa nhà cung cấp vì cung cấp giao diện lập trình thống nhất: lập trình viên viết mã dựa trên giao diện của Pac4j, có thể chuyển đổi nhà cung cấp chỉ bằng thay đổi cấu hình mà không cần sửa mã. Phiên bản 5.7.7 (phát hành năm 2023) tương đối mới, cho thấy Axelor duy trì cập nhật các thư viện phụ thuộc.

Tầng phân quyền hoàn toàn do Axelor tự xây dựng, bao gồm mô hình thực thể riêng (User, Group, Role, Permission, MetaPermission) và logic tầng dịch vụ (PermissionServiceImpl, PermissionAssistantService) để giải quyết quyền hạn khi chạy. Khung này không sử dụng các chú thích (annotation) `@PreAuthorize` hay `@Secured` của Spring Security - việc kiểm tra quyền hạn có thể xảy ra trong nội bộ khung (bộ chặn, bộ lọc) một cách trong suốt đối với mã ứng dụng. Cơ chế tiêm phụ thuộc (Dependency Injection) sử dụng Google Guice thay vì Spring - đây là quyết định kiến trúc cơ bản ảnh hưởng đến cách các thành phần được kết nối, cách quản lý phạm vi (scope), và cách triển khai chặn dựa trên AOP cho bảo mật.

**Bằng chứng từ mã nguồn - Câu lệnh nhập trong tầng dịch vụ:**
```java
// Tệp: PermissionAssistantService.java
import com.axelor.auth.db.Group;
import com.axelor.auth.db.Permission;
import com.axelor.auth.db.Role;
import com.axelor.meta.db.MetaPermission;
import com.axelor.meta.db.MetaPermissionRule;
```

**Giải thích mã nguồn:** Các câu lệnh nhập cho thấy cấu trúc gói (package) của thực thể bảo mật: `com.axelor.auth.db` chứa các thực thể phân quyền lõi (Group, Permission, Role), trong khi `com.axelor.meta.db` chứa thực thể phân quyền dựa trên siêu dữ liệu (MetaPermission, MetaPermissionRule). Tiền tố "Meta" gợi ý rằng đây là quyền hạn có thể cấu hình khi chạy (thay vì được định nghĩa lúc biên dịch), phù hợp với triết lý của Axelor về khả năng cấu hình. Các thực thể này không kế thừa từ lớp Spring Security (như GrantedAuthority, UserDetails) - xác nhận đây là triển khai tùy chỉnh hoàn toàn. [Từ source code]

**Bằng chứng từ mã nguồn - Dịch vụ tích hợp Pac4j:**
```java
// Đường dẫn tệp:
/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/service/pac4j/BaseAuthPac4jUserService.java
```

**Giải thích mã nguồn:** Lớp dịch vụ `BaseAuthPac4jUserService` đóng vai trò cầu nối giữa kết quả xác thực từ Pac4j và thực thể User của Axelor. Khi người dùng xác thực thành công qua nhà cung cấp OAuth/SAML/LDAP, Pac4j trả về dữ liệu hồ sơ (email, tên, thuộc tính). Dịch vụ này ánh xạ hồ sơ sang thực thể User của Axelor, xử lý việc cấp phát tài khoản (tạo người dùng mới hoặc liên kết người dùng hiện có), và gán nhóm/vai trò mặc định. Tiền tố "Base" trong tên lớp cho thấy đây là triển khai cơ sở có thể được ghi đè (override) trong dự án khách hàng để tùy chỉnh logic cấp phát tài khoản. [Từ source code]

**Các nhà cung cấp xác thực được hỗ trợ:** [Từ phân tích cấu hình]

Axelor hỗ trợ sáu nhà cung cấp xác thực:
- **Xác thực nội bộ (Local authentication)** - Tên đăng nhập/mật khẩu lưu trong cơ sở dữ liệu
- **Google OAuth 2.0** - Giao thức OpenID Connect
- **Keycloak** - Nền tảng quản lý danh tính và truy cập mã nguồn mở
- **SAML 2.0** - Tiêu chuẩn đăng nhập một lần (SSO) cho doanh nghiệp
- **LDAP** - Tích hợp thư mục doanh nghiệp (Active Directory, OpenLDAP)
- **CAS** - Dịch vụ xác thực tập trung (giao thức SSO cũ)

Sự đa dạng này rất quan trọng cho triển khai doanh nghiệp, nơi các tổ chức khác nhau có hạ tầng xác thực khác nhau. Doanh nghiệp lớn thường đã có LDAP/Active Directory chứa tài khoản nhân viên - khả năng hỗ trợ LDAP của Axelor cho phép tích hợp liền mạch mà không cần quản lý người dùng trùng lặp. Các công ty khởi nghiệp hoặc thiên về đám mây có thể ưu tiên OAuth 2.0 với Google/Keycloak cho luồng xác thực hiện đại. Tổ chức chính phủ hoặc có yêu cầu bảo mật cao có thể cần SAML 2.0 để tuân thủ tiêu chuẩn an ninh. [Từ source code]

---

### 2. MÔ HÌNH PHÂN QUYỀN: PHÂN CẤP BA TẦNG (NGƯỜI DÙNG → NHÓM/VAI TRÒ → QUYỀN HẠN)

**Tệp nguồn:** User.xml, Group.xml, Role.xml, Permission.xml, PermissionAssistantService.java [Từ source code]

Axelor triển khai hệ thống phân quyền phân cấp ba tầng (three-tier authorization hierarchy), tinh vi hơn mô hình kiểm soát truy cập dựa trên vai trò đơn giản (RBAC - Role-Based Access Control) nhưng dễ hiểu hơn hệ thống kiểm soát truy cập dựa trên thuộc tính (ABAC - Attribute-Based Access Control). Mô hình này cân bằng giữa tính linh hoạt (hỗ trợ yêu cầu phân quyền phức tạp của doanh nghiệp) và khả năng quản lý (quản trị viên có thể hiểu và cấu hình quyền hạn mà không cần kiến thức kỹ thuật sâu). Thiết kế phân cấp như sau: **Người dùng** (User - cá nhân) thuộc về một **Nhóm** (Group - nhóm nghiệp vụ: Kinh doanh, Kế toán, Quản lý) và có thể có nhiều **Vai trò** (Role - chức năng: Người duyệt đơn hàng, Người xem báo cáo, Quản trị viên), mỗi Nhóm/Vai trò chứa **Quyền hạn** (Permission - quyền truy cập cụ thể vào đối tượng/trường).

Lý do cho hệ thống kép Nhóm/Vai trò (thay vì chỉ có Vai trò như nhiều hệ thống RBAC) là tách biệt cấu trúc tổ chức khỏi năng lực chức năng. Nhóm thường ánh xạ tới đơn vị kinh doanh hoặc phòng ban (thường ổn định, ít thay đổi), trong khi Vai trò ánh xạ tới chức năng công việc hoặc trách nhiệm (có thể được gán/thu hồi thường xuyên khi nhân sự thay đổi vị trí). Ví dụ: một người dùng thuộc nhóm "Kinh doanh - Miền Bắc" (vị trí tổ chức) và có vai trò "Người tạo đơn hàng", "Người duyệt báo giá" (năng lực chức năng). Khi người dùng chuyển sang khu vực khác, chỉ cần đổi nhóm; khi được thăng chức, thêm vai trò "Quản lý" mà không cần đổi nhóm. Sự tách biệt này giúp quản lý quyền hạn có thể mở rộng trong tổ chức lớn với hàng trăm, hàng nghìn người dùng.

Logic tổng hợp quyền hạn rất quan trọng nhưng không được mô tả rõ ràng trong mã đã phân tích - có thể được triển khai trong lõi khung ứng dụng. Dựa trên các mẫu RBAC phổ biến, hành vi suy luận là: quyền hạn hiệu lực của người dùng là phép hợp (union) của quyền hạn từ nhóm CỘNG VỚI quyền hạn từ tất cả vai trò. Chiến lược hợp nhất theo hướng cấp phép (grant-based): nếu BẤT KỲ nguồn nào (nhóm hoặc vai trò) cấp quyền, người dùng có quyền đó. Không có bằng chứng về quy tắc TỪ CHỐI (DENY) tường minh - sự vắng mặt của quyền đồng nghĩa với từ chối. Cách hợp nhất dựa trên cấp phép đơn giản hơn để suy luận (quản trị viên không phải lo lắng về xung đột quyền) nhưng kém linh hoạt hơn hệ thống có ưu tiên cho phép từ chối ghi đè cấp phép. [Suy luận]

**Bằng chứng từ mã nguồn - Quan hệ Người dùng → Nhóm:**
```xml
<!-- Tệp: User.xml, dòng 45 -->
<many-to-one name="group" ref="Group" column="group_id" massUpdate="true"/>
```

**Giải thích mã nguồn:** Thực thể User có quan hệ nhiều-một (many-to-one) tới Group, nghĩa là mỗi người dùng thuộc về đúng MỘT nhóm chính (hoặc không có nhóm nào nếu chưa gán). Thuộc tính `massUpdate="true"` cho phép thao tác hàng loạt: quản trị viên có thể chọn nhiều người dùng và chuyển tất cả sang nhóm khác cùng lúc - tính năng thực tế cho kịch bản như "chuyển tất cả nhân viên từ phòng ban bị giải thể sang phòng ban mới". Tên cột `group_id` được chỉ định rõ ràng (thay vì dùng quy tắc đặt tên mặc định), có thể để tương thích ngược với lược đồ hiện có hoặc tích hợp với hệ thống bên ngoài. [Từ source code]

**Bằng chứng từ mã nguồn - Quan hệ Nhóm/Vai trò → Quyền hạn:**
```java
// Tệp: PermissionAssistantService.java

// Dòng 636: Nhóm chứa tập hợp MetaPermission
group.addMetaPermission(metaPermission);

// Dòng 657: Vai trò chứa tập hợp MetaPermission
role.addMetaPermission(metaPermission);

// Dòng 715: Nhóm chứa tập hợp Permission
group.addPermission(permission);

// Dòng 744: Vai trò chứa tập hợp Permission
role.addPermission(permission);
```

**Giải thích mã nguồn:** Các lời gọi phương thức `addMetaPermission()` và `addPermission()` cho thấy cả thực thể Group và Role đều có quan hệ một-nhiều (one-to-many) với MetaPermission và Permission. Hai loại quyền hạn phục vụ mục đích khác nhau: **Permission** cho phân quyền cấp đối tượng (người dùng có thể đọc/ghi/xóa thực thể SaleOrder không?), **MetaPermission** cho phân quyền cấp trường (người dùng có thể sửa trường 'discountAmount' trong SaleOrder không?). Sự tách biệt này cho phép kiểm soát chi tiết: có thể cấp quyền truy cập thực thể nhưng hạn chế các trường nhạy cảm cụ thể. [Từ source code]

**Sơ đồ phân cấp:** [Suy luận từ cấu trúc mã]
```
Người dùng (User - cá nhân)
  ├─→ Nhóm (Group, quan hệ nhiều-một) - Nhóm tổ chức chính
  │     ├─→ Permission[] (quyền CRUD cấp đối tượng)
  │     └─→ MetaPermission[] (quyền cấp trường)
  │
  └─→ Vai trò[] (Role, quan hệ nhiều-nhiều, suy luận) - Vai trò chức năng
        ├─→ Permission[] (quyền CRUD cấp đối tượng)
        └─→ MetaPermission[] (quyền cấp trường)
```

Mô hình này tuân theo nguyên tắc đặc quyền tối thiểu (principle of least privilege): người dùng khởi đầu không có quyền nào (từ chối mặc định), quyền được cấp phát tường minh thông qua thành viên nhóm/vai trò. Chi phí quản trị là gán người dùng vào nhóm/vai trò phù hợp, không phải quản lý quyền theo từng người dùng (cách đó không thể mở rộng). Khi nhân viên mới vào, chỉ cần gán vào nhóm (đơn vị tổ chức) và các vai trò liên quan (chức năng công việc), tự động kế thừa tất cả quyền cần thiết. Thay đổi quyền được tập trung: cập nhật quyền của nhóm/vai trò một lần, áp dụng cho tất cả thành viên ngay lập tức. [Suy luận]

---

### 3. THỰC THỂ NGƯỜI DÙNG: XÁC THỰC VÀ NGỮ CẢNH NGHIỆP VỤ

**Tệp nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/User.xml` [Từ source code]

Thực thể User trong Axelor phục vụ hai mục đích: **thông tin xác thực** (tên đăng nhập, mật khẩu, email) và **ngữ cảnh nghiệp vụ** (công ty, nhóm làm việc, liên kết đối tác, tùy chọn cá nhân). Đây là mẫu thiết kế phổ biến trong ứng dụng kinh doanh, nơi danh tính xác thực phải được ánh xạ tới các khái niệm trong lĩnh vực nghiệp vụ. Ví dụ: khi nhân viên kinh doanh đăng nhập, hệ thống cần biết không chỉ "đây là người dùng 'john.doe'" (xác thực) mà còn "john.doe đại diện cho đối tác Công ty Acme, làm việc trong nhóm Miền Bắc, ngữ cảnh công ty hiện tại là chi nhánh Hà Nội" (ngữ cảnh nghiệp vụ). Ngữ cảnh nghiệp vụ này rất quan trọng cho giải quyết quyền hạn (lọc bản ghi theo công ty/nhóm của người dùng) và hành vi ứng dụng (giá trị mặc định, hành động khả dụng, nội dung bảng điều khiển).

Thực thể User được định nghĩa trong gói `com.axelor.auth.db` (lõi khung ứng dụng), nhưng axelor-open-suite **mở rộng** thực thể cơ sở bằng các trường riêng cho nghiệp vụ thông qua cơ chế mở rộng XML (XML extension). Mẫu mở rộng cho phép khung duy trì logic xác thực lõi (băm mật khẩu, quản lý phiên, đăng nhập/đăng xuất) trong lớp cơ sở ổn định, trong khi ứng dụng thêm tùy chỉnh theo lĩnh vực mà không cần phân nhánh (fork) mã khung. Đổi lại, mở rộng bị giới hạn ở việc thêm trường/phương thức, không thể thay đổi hành vi xác thực lõi mà không sửa đổi khung - chấp nhận được cho hầu hết trường hợp sử dụng nhưng có thể hạn chế tùy chỉnh sâu với yêu cầu xác thực đặc biệt.

Các trường liên quan đến bảo mật trong thực thể User cung cấp kiểm soát truy cập theo thời gian: cờ `blocked` (kiểu boolean) vô hiệu hóa tài khoản ngay lập tức (cho nhân viên đã nghỉ việc hoặc sự cố bảo mật), ngày `activateOn` triển khai kích hoạt trì hoãn (tạo tài khoản trước, kích hoạt vào ngày bắt đầu làm việc), ngày `expiresOn` triển khai hết hạn tự động (nhân viên hợp đồng tạm thời, tài khoản dùng thử). Sự kết hợp ba trường này cho phép quản lý vòng đời tài khoản tinh vi: bộ phận nhân sự tạo tài khoản khi ký hợp đồng với `activateOn` đặt vào ngày bắt đầu và `expiresOn` đặt vào ngày kết thúc, tài khoản tự động kích hoạt/vô hiệu theo lịch mà không cần can thiệp thủ công. Trường `sendEmailUponPasswordChange` triển khai thông báo bảo mật (người dùng nhận email khi mật khẩu thay đổi - phát hiện đặt lại mật khẩu trái phép). [Từ source code]

**Bằng chứng từ mã nguồn - Cấu trúc thực thể User:**
```xml
<entity name="User" sequential="true">
  <!-- Xác thực & Kiểm soát truy cập -->
  <many-to-one name="group" ref="Group" column="group_id" massUpdate="true"/>
  <boolean name="blocked" default="true"
    help="Specify whether to block the user for an indefinite period." massUpdate="true"/>

  <!-- Ngữ cảnh nghiệp vụ - Đa công ty -->
  <many-to-many name="companySet" ref="com.axelor.apps.base.db.Company" title="Company set"/>
  <many-to-one name="activeCompany" ref="com.axelor.apps.base.db.Company"
    title="Active company" massUpdate="true"/>

  <!-- Ngữ cảnh nghiệp vụ - Nhóm làm việc -->
  <many-to-many name="teamSet" ref="com.axelor.apps.base.db.Team" title="Team set"/>
  <many-to-one name="activeTeam" ref="com.axelor.apps.base.db.Team"
    title="Active team" massUpdate="true"/>

  <!-- Ngữ cảnh nghiệp vụ - Liên kết đối tác -->
  <one-to-one name="partner" ref="com.axelor.apps.base.db.Partner"
    title="Partner" mappedBy="linkedUser"/>

  <!-- Bản địa hóa & Tùy chọn -->
  <string name="language" selection="select.language"/>
  <string name="localization"/>

  <!-- Hỗ trợ trường tùy chỉnh -->
  <string name="attrs" json="true"/>
</entity>
```

**Giải thích mã nguồn:** Khai báo thực thể không có lớp cơ sở trong XML (chỉ `<entity name="User">`), cho thấy đây là phần mở rộng của thực thể User hiện có từ lõi khung - bộ sinh mã sẽ hợp nhất các trường này vào lớp cơ sở. Thuộc tính `sequential="true"` chỉ ra thực thể có cơ chế sinh số thứ tự tự động.

Trường `blocked` có `default="true"` - lựa chọn đáng chú ý! Người dùng mới được tạo ở trạng thái bị chặn, phải được mở khóa tường minh trước khi đăng nhập. Lý do: cách tiếp cận an toàn trước (safety-first) ngăn kích hoạt tài khoản vô tình, đảm bảo quản trị viên xem xét và kích hoạt tài khoản tường minh sau khi thiết lập hoàn tất. Nội dung trợ giúp xác nhận mục đích: "chặn người dùng trong thời gian vô thời hạn" (không phải đình chỉ tạm thời mà là vô hiệu hóa hoàn toàn).

Hỗ trợ đa công ty thông qua `companySet` (nhiều-nhiều: người dùng có thể làm việc cho nhiều công ty) và `activeCompany` (ngữ cảnh hiện tại). Mẫu này phổ biến trong kịch bản đa pháp nhân (multi-tenant) khi một tài khoản người dùng trải trên nhiều thực thể pháp lý/chi nhánh. Người dùng chuyển đổi công ty đang hoạt động trong giao diện, ứng dụng lọc dữ liệu/quyền hạn tương ứng. Mẫu tương tự cho nhóm làm việc: `teamSet` (tất cả nhóm người dùng tham gia) và `activeTeam` (ngữ cảnh nhóm hiện tại dùng cho lọc/giá trị mặc định). [Từ source code]

**Bằng chứng từ mã nguồn - Trường tính toán fullName:**
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

**Giải thích mã nguồn:** Trường `fullName` được tính toán động dựa trên việc người dùng có liên kết với thực thể Đối tác (Partner) hay không. Logic ưu tiên tên đối tác (đại diện thực thể nghiệp vụ) hơn tên đăng nhập kỹ thuật: nếu người dùng liên kết với đối tác Nguyễn Văn A, tên hiển thị là "Văn A Nguyễn" thay vì "nguyen.a". Dự phòng sang `name` (tên đăng nhập) nếu không có liên kết đối tác. Thuộc tính `namecolumn="true"` đánh dấu đây là tên hiển thị dùng trong danh sách thả xuống, kết quả tìm kiếm, nhật ký kiểm toán. Thuộc tính `search="partner,name"` cho phép tìm kiếm theo tên đối tác HOẶC tên đăng nhập.

**Ý nghĩa của liên kết Đối tác:** [Suy luận về mẫu tích hợp]

Quan hệ một-một `partner` với `mappedBy="linkedUser"` chỉ ra liên kết hai chiều: thực thể User có tham chiếu đến Partner, thực thể Partner có tham chiếu ngược linkedUser. Trường hợp sử dụng: ứng dụng kinh doanh nơi đối tác bên ngoài (khách hàng, nhà cung cấp) cần truy cập cổng thông tin - người liên hệ của đối tác được cấp tài khoản người dùng liên kết với bản ghi Đối tác. Khi người dùng đối tác đăng nhập, ứng dụng tự động biết họ đại diện cho công ty nào, có thể chỉ hiển thị dữ liệu liên quan (đơn hàng, hóa đơn của chính họ).

---

### 4. THỰC THỂ NHÓM: ĐƠN VỊ TỔ CHỨC VÀ VAI TRÒ NGHIỆP VỤ

**Tệp nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Group.xml` [Từ source code]

Thực thể Group đại diện cho **đơn vị tổ chức** hoặc **nhóm vai trò nghiệp vụ** trong cấu trúc doanh nghiệp - không chỉ là hộp chứa quyền hạn kỹ thuật. Các trường như `isClient`, `isSupplier` cho thấy nhóm có thể đại diện cho cộng đồng người dùng bên ngoài (người dùng cổng khách hàng, người dùng cổng nhà cung cấp), không chỉ nhân viên nội bộ. Trường `technicalStaff` gợi ý danh mục đặc biệt cho nhân viên CNTT/quản trị có đặc quyền nâng cao. Mẫu này mờ ranh giới giữa "nhóm như phòng ban" và "nhóm như nhân cách" (persona) - cùng loại thực thể phục vụ nhiều khái niệm tổ chức. Tính linh hoạt này mạnh mẽ nhưng có rủi ro nhầm lẫn: quản trị viên cần thiết lập quy ước đặt tên rõ ràng để phân biệt loại nhóm.

Thực thể được đánh dấu `cacheable="true"` - tối ưu hiệu suất quan trọng vì dữ liệu nhóm được truy cập thường xuyên (mỗi lần kiểm tra quyền có thể truy vấn thành viên nhóm) nhưng ít thay đổi (tái cơ cấu tổ chức xảy ra theo quý/năm, không phải hàng giờ). Bộ đệm cấp hai (L2 cache) của Hibernate lưu các đối tượng Group đã giải mã hóa trong bộ nhớ, truy vấn sau đó lấy từ bộ đệm thay vì cơ sở dữ liệu. Vô hiệu hóa bộ đệm (cache invalidation) là yếu tố then chốt: khi nhóm bị sửa đổi (thêm/xóa quyền, thay đổi thuộc tính), mục bộ đệm phải được loại bỏ nếu không dữ liệu cũ gây lỗi quyền hạn. Khung Axelor có thể đã tích hợp cơ chế vô hiệu hóa bộ đệm tự động trong bộ lắng nghe vòng đời thực thể (entity lifecycle listener). [Từ source code]

Các trường `navigation` và `homeAction` cho phép tùy chỉnh giao diện theo nhóm: nhóm khác nhau thấy menu điều hướng khác nhau và bảng điều khiển trang chủ khác nhau khi đăng nhập. Ví dụ: nhóm Kinh doanh thấy bảng điều khiển kênh bán hàng, nhóm Kế toán thấy tổng kết tài chính, nhóm Lãnh đạo thấy chỉ số KPI. Việc triển khai có thể: khi đăng nhập, khung tải nhóm của người dùng, đọc thuộc tính homeAction, chuyển hướng đến hành động được chỉ định. Tùy chỉnh này cải thiện trải nghiệm người dùng (người dùng thấy nội dung liên quan ngay lập tức) nhưng tăng độ phức tạp cấu hình (quản trị viên phải duy trì cấu hình giao diện riêng cho từng nhóm). [Suy luận]

**Bằng chứng từ mã nguồn - Các trường của thực thể Group:**
```xml
<entity name="Group" cacheable="true">
  <!-- Cờ vai trò nghiệp vụ -->
  <boolean name="technicalStaff"
    help="Specify whether the members of this group are technical staff." massUpdate="true"/>
  <boolean name="isClient" default="false" massUpdate="true" title="Client"/>
  <boolean name="isSupplier" default="false" massUpdate="true" title="Supplier"/>

  <!-- Tùy chỉnh giao diện -->
  <string name="navigation" selection="select.user.navigation" massUpdate="true"/>
  <string name="homeAction" help="Default home action." massUpdate="true"/>
</entity>
```

**Giải thích mã nguồn:** Thực thể mở rộng Group cơ sở từ gói `com.axelor.auth.db` (lõi khung), thêm các trường riêng cho nghiệp vụ. Tất cả cờ boolean đều có `massUpdate="true"` cho phép thay đổi hàng loạt - hữu ích khi phân loại lại nhóm.

Trường `technicalStaff` có nội dung trợ giúp mô tả gợi ý xử lý đặc biệt: thành viên nhóm kỹ thuật có thể bỏ qua một số hạn chế, xem thông tin gỡ lỗi, truy cập tính năng quản trị. Rủi ro: chỉ định nhóm kỹ thuật quá rộng dẫn đến leo thang đặc quyền quá mức - thực hành tốt nhất là giới hạn cho nhân viên CNTT/quản trị thực sự.

Các trường `isClient` và `isSupplier` hỗ trợ kịch bản cổng thông tin nơi thực thể bên ngoài có quyền truy cập hạn chế. Nhóm khách hàng có thể có quyền chỉ đọc đối với đơn hàng/hóa đơn của họ cùng khả năng tự phục vụ. Nhóm nhà cung cấp có thể cập nhật tình trạng giao hàng, gửi hóa đơn điện tử. Giá trị mặc định `false` nghĩa là nhóm nhân viên nội bộ trừ khi được đánh dấu tường minh - an toàn trước (không vô tình tiết lộ dữ liệu nội bộ cho người dùng bên ngoài). [Từ source code]

---

### 5. THỰC THỂ VAI TRÒ: NĂNG LỰC CHỨC NĂNG VÀ QUYỀN HẠN TRỰC GIAO

**Tệp nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Role.xml` [Từ source code]

Thực thể Role trong Axelor cực kỳ tối giản - chỉ có trường `name` và `description` trong XML mở rộng, cho thấy lõi khung cung cấp hầu hết chức năng. Tính tối giản này có chủ đích: vai trò là nơi chứa quyền hạn thuần túy, không có logic nghiệp vụ hay tùy chỉnh giao diện (khác với Nhóm có navigation/homeAction). Phân tách trách nhiệm: Nhóm = ngữ cảnh tổ chức + tùy chỉnh giao diện + quyền hạn, Vai trò = năng lực chức năng thuần túy (chỉ quyền hạn). Người dùng có thể có một nhóm (vị trí tổ chức) nhưng nhiều vai trò (nhiều năng lực chức năng), cho phép phân quyền chi tiết.

Trường hợp sử dụng cho vai trò so với nhóm: xét nhân viên làm việc bán thời gian nhiều vai: thuộc nhóm "Kinh doanh" (vị trí tổ chức chính) nhưng có vai trò "Người tạo đơn hàng" (có thể tạo đơn hàng), "Người duyệt hóa đơn" (có thể duyệt hóa đơn), "Người xem báo cáo" (có thể xem phân tích). Khi nhân viên được thăng chức, thêm vai trò "Quản lý" (có thể duyệt số tiền cao hơn, xem dữ liệu cấp dưới) mà không đổi thành viên nhóm. Khi nhân viên tạm thay thế đồng nghiệp, thêm vai trò tạm thời (có thể thu hồi sau khi hết thời gian thay thế). Gán/thu hồi vai trò thường xuyên và chi tiết hơn thay đổi thành viên nhóm. [Suy luận]

**Bằng chứng từ mã nguồn - Cấu trúc thực thể Role:**
```xml
<entity name="Role">
  <track>
    <field name="name"/>
    <field name="description"/>
  </track>
</entity>
```

**Giải thích mã nguồn:** XML cực kỳ ngắn gọn - chỉ có cấu hình theo dõi kiểm toán (audit tracking). Không có trường tùy chỉnh, không có logic nghiệp vụ, không có gợi ý giao diện. Điều này củng cố vai trò như bộ tổng hợp quyền hạn nhẹ. Theo dõi ghi lại thay đổi đối với định nghĩa vai trò: khi quản trị viên đổi tên vai trò hoặc cập nhật mô tả, nhật ký kiểm toán ghi nhận ai đã thay đổi và khi nào. Nhật ký kiểm toán quan trọng cho kịch bản tuân thủ (kiểm toán viên hỏi "ai đã thay đổi quyền hạn cho vai trò Tài chính?"). [Từ source code]

**Ma trận quyết định Vai trò và Nhóm:** [Suy luận về thực hành tốt nhất]

Dùng **Nhóm** khi: tập quyền phản ánh cấu trúc tổ chức (phòng ban, bộ phận); người dùng thường có một liên kết chính; cần tùy chỉnh giao diện (màn hình chủ khác nhau theo đơn vị tổ chức); cần cờ nghiệp vụ (isClient, isSupplier, technicalStaff); thay đổi không thường xuyên.

Dùng **Vai trò** khi: tập quyền phản ánh chức năng công việc (người xem, người sửa, người duyệt); người dùng có thể có nhiều năng lực đồng thời; quyền được gán/thu hồi thường xuyên; quan tâm xuyên suốt trải trên nhiều nhóm (tất cả quản lý ở mọi phòng ban cần quyền xem báo cáo); tổ hợp quyền chi tiết (xây dựng quyền phức tạp từ các khối đơn giản).

Cách tiếp cận kết hợp (dùng cả hai) mạnh mẽ nhất: người dùng kế thừa quyền rộng từ nhóm (mức cơ bản phòng ban) cộng thêm năng lực cụ thể từ vai trò (bổ sung chức năng).

---

### 6. THỰC THỂ QUYỀN HẠN: PHÂN QUYỀN CRUD CẤP ĐỐI TƯỢNG

**Tệp nguồn:** `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Permission.xml`, PermissionAssistantService.java [Từ source code]

Thực thể Permission triển khai **phân quyền cấp đối tượng** - kiểm soát quyền truy cập vào toàn bộ thực thể (SaleOrder, Invoice, Product) chứ không phải từng bản ghi riêng lẻ hoặc từng trường. Mức chi tiết này thô hơn phân quyền cấp bản ghi (nơi quyền khác nhau theo bản ghi: "có thể sửa đơn hàng CỦA TÔI nhưng không phải của người khác") nhưng mịn hơn phân quyền cấp ứng dụng (nơi quyền áp dụng cho toàn bộ ứng dụng: "có thể truy cập mô-đun kinh doanh"). Phân quyền cấp đối tượng là điểm cân bằng thực tế: quản trị viên có thể kiểm soát "nhóm Kinh doanh có thể tạo Đơn mua hàng không?" mà không phải quản lý vi mô từng đơn hàng riêng lẻ. Thực thể được đánh dấu `cacheable="true"`, rất quan trọng cho hiệu suất - kiểm tra quyền xảy ra cực kỳ thường xuyên (có thể mỗi truy vấn cơ sở dữ liệu), bộ đệm ngăn cơ sở dữ liệu trở thành nút thắt cổ chai.

Mô hình quyền hỗ trợ năm thao tác đại diện cho CRUD tiêu chuẩn cộng khả năng xuất: **canRead** (xem bản ghi), **canWrite** (sửa bản ghi hiện có), **canCreate** (tạo bản ghi mới), **canRemove** (xóa bản ghi), và **canExport** (xuất ra CSV/Excel/PDF). Tách biệt tạo (create) khỏi ghi (write) quan trọng cho quy trình làm việc: người dùng thực tập có thể tạo đơn hàng nháp (canCreate) nhưng không thể sửa đơn hàng đã gửi (không có canWrite), buộc phải qua quy trình duyệt. Quyền xuất tách riêng vì xuất dữ liệu gây lo ngại rò rỉ dữ liệu: người dùng có thể xem từng bản ghi trên màn hình (canRead) nhưng xuất hàng loạt ra bảng tính (canExport) cho phép chiết xuất dữ liệu - cần kiểm soát chặt hơn. [Từ source code]

Quy ước đặt tên cho quyền hạn tuân theo mẫu `perm.{TênĐốiTượng}.{MãNhómHoặcVaiTrò}` - ví dụ: `perm.SaleOrder.sales_team`, `perm.Product.admins`. Quy ước này cho phép: (1) dò quét trực quan trong danh sách quyền, (2) tránh xung đột tên, (3) sinh tên bằng chương trình. Trường `object` chứa tên lớp đầy đủ (ví dụ: `com.axelor.apps.sale.db.SaleOrder`) hoặc mẫu ký tự đại diện theo gói (ví dụ: `com.axelor.apps.sale.db.*` nghĩa là tất cả thực thể trong gói). Hỗ trợ ký tự đại diện rất quan trọng để mở rộng: quản trị viên có thể cấp "tất cả thực thể kinh doanh" mà không cần liệt kê hàng trăm thực thể riêng lẻ. [Từ source code]

**Bằng chứng từ mã nguồn - Cấu trúc thực thể Permission:**
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

**Giải thích mã nguồn:** Cấu hình theo dõi ghi nhật ký mọi thay đổi trường - thiết yếu cho kiểm toán bảo mật. Khi quyền bị sửa đổi (ví dụ quản trị viên vô tình cấp canRemove khi chỉ nên cấp canRead), nhật ký kiểm toán cho thấy ai đã mắc lỗi và khi nào, cho phép hoàn tác nhanh chóng. Các trường `condition` và `conditionParams` hỗ trợ lọc cấp bản ghi (thảo luận ở mục tiếp theo) - quyền hạn không chỉ cho phép/từ chối nhị phân mà có thể có điều kiện dựa trên thuộc tính bản ghi và ngữ cảnh người dùng. [Từ source code]

**Bằng chứng từ mã nguồn - Gán quyền CRUD:**
```java
// Tệp: PermissionAssistantService.java, dòng 280-285
permission.setCanRead(row[0].equalsIgnoreCase("x"));
permission.setCanWrite(row[1].equalsIgnoreCase("x"));
permission.setCanCreate(row[2].equalsIgnoreCase("x"));
permission.setCanRemove(row[3].equalsIgnoreCase("x"));
permission.setCanExport(row[4].equalsIgnoreCase("x"));
```

**Giải thích mã nguồn:** Mã dịch vụ phân tích dòng CSV nhập vào, mỗi thao tác được biểu diễn bằng dấu "x" (có = cấp phép, không có = từ chối). Các trường boolean được thiết lập dựa trên so sánh không phân biệt hoa thường - chấp nhận cả "X" lẫn "x" cho tiện lợi. Biểu diễn đơn giản này (x so với ô trống) giúp tệp CSV dễ đọc và chỉnh sửa trong công cụ bảng tính - quản trị viên có thể xuất quyền, sửa đổi trong Excel, nhập lại. [Từ source code]

**Bằng chứng từ mã nguồn - Quy ước đặt tên quyền:**
```java
// Tệp: PermissionAssistantService.java, dòng 251-256
protected String getPermissionName(MetaField userField, String objectName, String suffix) {
    String permName = "perm." + objectName + "." + suffix;
    if (userField != null) {
      permName += "." + userField.getName();
    }
    return permName;
}
```

**Giải thích mã nguồn:** Phương thức xây dựng tên quyền từ các thành phần: tiền tố "perm" (nhận diện thực thể quyền), tên đối tượng (thực thể đích), hậu tố (mã nhóm/vai trò), tùy chọn tên trường (cho quyền cấp trường). Định dạng phân tách bằng dấu chấm cho phép tổ chức phân cấp và phân tích cú pháp. Kiểm tra null cho `userField` cho thấy phương thức phục vụ hai mục đích: quyền cấp đối tượng (trường null) và quyền cấp trường (trường được chỉ định). [Từ source code]

**Kiểm tra quyền đối chiếu với siêu mô hình:** [Từ source code, PermissionAssistantService dòng 500-514]
```java
public List<Long> checkPermissionsObject() {
    List<Permission> permissionList = permissionRepository.all().fetch();
    if (ObjectUtils.isEmpty(permissionList)) {
      return null;
    }

    initObjectOrPackages();  // Lấy tất cả gói thực thể từ siêu mô hình JPA

    return permissionList.stream()
        .filter(permission -> !isValidObject(permission.getObject()))
        .map(Permission::getId)
        .collect(Collectors.toList());
}

protected boolean isValidObject(String object) {
    String regex = object.replace("*", ".*");  // Hỗ trợ ký tự đại diện
    return objectOrPackages.stream()
        .anyMatch(entityPackage -> entityPackage.matches(regex));
}
```

**Giải thích mã nguồn:** Dịch vụ kiểm tra phát hiện quyền hạn mồ côi (orphaned) - tức quyền tham chiếu đến thực thể không còn tồn tại trong ứng dụng (mô-đun đã gỡ, thực thể đã đổi tên, lỗi chính tả trong thiết lập quyền). Phương thức `initObjectOrPackages()` truy vấn siêu mô hình JPA (JPA metamodel) để lấy danh sách tất cả lớp thực thể đang đăng ký. Bộ lọc luồng (stream filter) kiểm tra tên đối tượng của mỗi quyền đối chiếu với siêu mô hình, sử dụng khớp biểu thức chính quy (regex) để hỗ trợ ký tự đại diện (wildcard). Quyền không hợp lệ được trả về dưới dạng danh sách mã định danh (ID) để quản trị viên xem xét và xóa. Kiểm tra này rất quan trọng sau khi gỡ mô-đun hoặc tái cấu trúc mã - ngăn tích tụ quyền cũ gây nhầm lẫn. [Từ source code]

---

### 7. BẢO MẬT CẤP BẢN GHI: BỘ LỌC MIỀN VÀ TRUY CẬP CÓ ĐIỀU KIỆN

**Tệp nguồn:** PermissionAssistantService.java dòng 310-343 [Từ source code]

Bảo mật cấp bản ghi (record-level security) — còn gọi là bảo mật cấp dòng (row-level security) trong thuật ngữ cơ sở dữ liệu — là cấp phân quyền tinh vi nhất trong hệ thống: quyền hạn không chỉ kiểm soát "người dùng có được truy cập thực thể SaleOrder không?" (cấp đối tượng) mà còn kiểm soát "người dùng được truy cập những bản ghi SaleOrder CỤ THỂ nào?" (cấp bản ghi). Cơ chế triển khai thông qua **điều kiện** (condition) — các mệnh đề WHERE dạng SQL được chèn động vào truy vấn dựa trên ngữ cảnh người dùng. Mẫu thiết kế này biến đổi kiểm tra quyền đơn giản từ dạng nhị phân có/không thành tập dữ liệu được lọc: người dùng luôn "có quyền" nhưng chỉ nhìn thấy tập con bản ghi phù hợp với ngữ cảnh của họ.

Cú pháp điều kiện giống ngôn ngữ truy vấn JPA: `self.tenTruong` tham chiếu thực thể đang được truy vấn, chỗ giữ chỗ `?` đại diện cho giá trị tham số được cung cấp từ `conditionParams`. Điểm then chốt: điều kiện được thực thi ở tầng cơ sở dữ liệu (là một phần của mệnh đề WHERE trong SQL) chứ không phải ở tầng ứng dụng (lọc trong mã Java), đảm bảo: (1) hiệu năng — chỉ mục cơ sở dữ liệu được sử dụng, chỉ các dòng khớp được trả về; (2) bảo mật — người dùng không thể vượt qua bằng cách thao túng mã ứng dụng; (3) nhất quán — cùng một bộ lọc áp dụng cho mọi đường truy cập (giao diện, API, báo cáo). Đánh đổi (trade-off): điều kiện bị giới hạn ở các biểu thức mà cơ sở dữ liệu có thể đánh giá — logic nghiệp vụ phức tạp cần tính toán ở tầng dịch vụ không thể mã hóa trong điều kiện. [Từ source code]

Các biến ngữ cảnh đặc biệt trong `conditionParams` cho phép lọc động dựa trên thuộc tính của người dùng hiện tại. Mẫu `__user__.{tenTruong}` được giải quyết khi chạy (runtime): bộ khung (framework) đọc giá trị trường của người dùng đang đăng nhập và thay thế vào truy vấn. Ví dụ: điều kiện `self.company = ?` với tham số `__user__.activeCompany` trở thành SQL `WHERE sale_order.company_id = 123` (trong đó 123 là mã công ty đang hoạt động của người dùng hiện tại). Cơ chế giải quyết biến hỗ trợ duyệt quan hệ: `__user__.partner.company` điều hướng từ Người dùng qua Đối tác đến Công ty. Các trường tập hợp hỗ trợ mệnh đề IN: `__user__.teamSet` mở rộng thành `(1,2,3)` cho người dùng thuộc nhóm làm việc 1, 2, 3. [Suy luận]

**Bằng chứng từ mã nguồn - Logic sinh điều kiện:**
```java
// Tệp: PermissionAssistantService.java, dòng 325-342
String condition = "";
String conditionParams = "__user__." + userField.getName();

if (userField.getRelationship().contentEquals("ManyToOne")) {
  condition = "self." + objectField.getName() + " = ?";
} else {
  condition = "self." + objectField.getName() + " in (?)";
}

// Ví dụ kết quả:
// Quan hệ nhiều-một (người dùng có một công ty đang hoạt động):
//   condition: "self.company = ?"
//   conditionParams: "__user__.activeCompany"
//
// Quan hệ nhiều-nhiều (người dùng thuộc nhiều nhóm làm việc):
//   condition: "self.assignedTo in (?)"
//   conditionParams: "__user__.teamSet"
```

**Giải thích mã nguồn:** Mã sinh cú pháp điều kiện một cách động dựa trên loại quan hệ được phát hiện qua nội suy siêu mô hình (metamodel introspection). Quan hệ nhiều-một (giá trị đơn) dùng toán tử bằng `=`, quan hệ tập hợp (nhiều-nhiều, một-nhiều) dùng toán tử thuộc tập `in`. Tên trường được trích xuất từ siêu dữ liệu (`userField.getName()`, `objectField.getName()`) đảm bảo điều kiện hợp lệ đối với lược đồ thực tế — ngăn lỗi chính tả. Điều kiện được sinh ra lưu trong thực thể Permission để đánh giá khi chạy, không phải khi biên dịch — cho phép quản trị viên sửa đổi quy tắc lọc mà không cần triển khai lại mã. [Từ source code]

**Ví dụ minh họa sức mạnh cơ chế:**

**Kịch bản 1: Cách ly dữ liệu đa công ty**
```
Người dùng: Minh (activeCompany: Chi nhánh Hà Nội)
Quyền trên SaleOrder:
  condition: "self.company = ?"
  conditionParams: "__user__.activeCompany"

Kết quả: Minh chỉ thấy đơn hàng có company_id = mã Chi nhánh Hà Nội
Mọi truy vấn được lọc tự động, trong suốt đối với mã ứng dụng.
```

**Kịch bản 2: Phân bổ bản ghi theo nhóm làm việc**
```
Người dùng: Lan (teamSet: [Kinh doanh Bắc, Kinh doanh Tây])
Quyền trên Lead:
  condition: "self.assignedTeam in (?)"
  conditionParams: "__user__.teamSet"

Kết quả: Lan thấy khách hàng tiềm năng gán cho Kinh doanh Bắc HOẶC Kinh doanh Tây.
Khi Lan được chuyển sang nhóm khác, dữ liệu hiển thị thay đổi tự động.
```

**Kịch bản 3: Truy cập phân cấp (quản lý xem dữ liệu cấp dưới)**
```
Người dùng: Quản lý Tuấn (teamSet bao gồm tất cả nhóm quản lý)
Quyền trên TimeSheet:
  condition: "self.employee.team in (?)"
  conditionParams: "__user__.teamSet"

Kết quả: Tuấn thấy bảng chấm công của tất cả nhân viên trong các nhóm do mình quản lý.
Biểu thức đường dẫn "self.employee.team" duyệt từ BảngChấmCông → NhânViên → NhómLàmViệc.
```

**Thách thức triển khai suy luận:** [Suy luận về độ phức tạp kỹ thuật]

Triển khai bảo mật cấp bản ghi đúng cách đòi hỏi bộ khung xử lý: (1) viết lại truy vấn — chặn mọi truy vấn (JPA Criteria, JPQL, DSL truy vấn) và chèn mệnh đề WHERE điều kiện; (2) ràng buộc tham số — giải quyết biến `__user__`, xử lý chuyển đổi kiểu (tham chiếu thực thể → mã định danh), mở rộng tập hợp; (3) tối ưu phép nối (JOIN) — điều kiện có biểu thức đường dẫn (`self.employee.team`) yêu cầu phép nối tự động, phải tránh vấn đề N+1 truy vấn; (4) tổ hợp quyền — nhiều quyền có điều kiện khác nhau (từ nhóm + vai trò) phải được kết hợp bằng phép HOẶC (OR) đúng cách; (5) độ phức tạp bộ đệm — kết quả điều kiện phụ thuộc người dùng, không thể đệm toàn cục mà chỉ theo phiên làm việc.

**Bằng chứng từ mã nguồn - Xuất CSV bao gồm điều kiện:**
```java
// Tệp: PermissionAssistantService.java, dòng 310-312
row[colIndex++] = Strings.isNullOrEmpty(perm.getCondition()) ? "" : perm.getCondition();
row[colIndex++] = Strings.isNullOrEmpty(perm.getConditionParams()) ? "" : perm.getConditionParams();
```

**Giải thích mã nguồn:** Xuất CSV bao gồm cả điều kiện và tham số điều kiện như các cột thông thường — quản trị viên có thể chỉnh sửa điều kiện trong công cụ bảng tính. Xử lý chuỗi rỗng (kiểm tra `Strings.isNullOrEmpty`) phân biệt giữa không có điều kiện (ô trống = tất cả bản ghi hiển thị) và chuỗi điều kiện rỗng (có thể là lỗi phân tích). Định dạng xuất cho phép cập nhật điều kiện hàng loạt: quản trị viên xuất tất cả quyền, dùng công thức Excel để sinh điều kiện cho nhiều nhóm/thực thể, rồi nhập lại. [Từ source code]

---

### 8. QUYỀN CẤP TRƯỜNG: METAPERMISSION VÀ METAPERMISSIONRULE

**Tệp nguồn:** PermissionAssistantService.java dòng 621-689 [Từ source code]

Quyền cấp trường dữ liệu (field-level permission) cung cấp kiểm soát phân quyền chi tiết nhất: ngay cả khi người dùng có quyền cấp đối tượng để đọc thực thể SaleOrder, các trường nhạy cảm cụ thể (discountAmount, costPrice, marginPercentage) vẫn có thể bị ẩn hoặc chỉ đọc. Kiến trúc sử dụng **cấu trúc hai tầng**: **MetaPermission** đóng vai trò thùng chứa/nhóm gom cho quyền trường của một đối tượng, **MetaPermissionRule** định nghĩa quy tắc truy cập thực tế cho từng trường. Thiết kế hai tầng cho phép truy vấn hiệu quả (tải MetaPermission một lần, lấy tất cả quy tắc trường cùng nhau) và tổ chức rõ ràng (quy tắc trường được nhóm theo đối tượng, không phân tán khắp cơ sở dữ liệu).

Thực thể MetaPermission tuân theo cùng quy ước đặt tên như Permission (`perm.{ĐốiTượng}.{NhómHoặcVaiTrò}`), thuộc sở hữu của Nhóm hoặc Vai trò qua quan hệ một-nhiều (one-to-many). Một MetaPermission có thể chứa hàng chục thực thể MetaPermissionRule (mỗi quy tắc ứng với một trường). Sự tương đồng cấu trúc với thực thể Permission là có chủ đích — quản trị viên hiểu "Permission kiểm soát đối tượng, MetaPermission kiểm soát trường" mà không cần học khái niệm hoàn toàn khác.

MetaPermissionRule cung cấp ba cờ boolean (canRead, canWrite, canExport) song song với cờ của Permission nhưng đáng chú ý là THIẾU canCreate và canRemove — hợp lý vì trường không tồn tại độc lập với thực thể cha, không thể "tạo trường" mà không tạo toàn bộ thực thể. Quyền xuất tách riêng (giống cấp đối tượng) vì dữ liệu xuất hiển thị giá trị trường ngay cả khi giao diện ẩn chúng. Phân biệt giữa chỉ đọc (readonly) và ẩn (hidden) rất quan trọng: **trường chỉ đọc hiển thị nhưng không chỉnh sửa được** (người dùng thấy giá trị, hiểu logic nghiệp vụ, nhưng không thể thay đổi — ví dụ tổng tính toán), **trường ẩn hoàn toàn vô hình** (người dùng không biết trường tồn tại — ví dụ dữ liệu giá vốn nội bộ không hiển thị cho nhóm kinh doanh). [Từ source code]

**Bằng chứng từ mã nguồn - Truy xuất/tạo MetaPermission:**
```java
// Tệp: PermissionAssistantService.java, dòng 621-640
public MetaPermission getMetaPermission(Group group, String objectName) {
    String permName = getPermissionName(null, objectNames[objectNames.length - 1], group.getCode());
    MetaPermission metaPermission = metaPermissionRepository.all()
        .filter("self.name = ?1", permName)
        .fetchOne();

    if (metaPermission == null) {
      metaPermission = new MetaPermission();
      metaPermission.setName(permName);
      metaPermission.setObject(objectName);

      group.addMetaPermission(metaPermission);  // ← Quan hệ hai chiều
    }

    return metaPermission;
}
```

**Giải thích mã nguồn:** Phương thức triển khai mẫu lấy-hoặc-tạo (get-or-create): truy vấn MetaPermission hiện có theo tên, nếu không tìm thấy thì tạo phiên bản mới. Mẫu này phổ biến trong quản lý quyền hạn — tránh tạo bản ghi quyền trùng lặp (sẽ gây phân quyền mơ hồ). Lời gọi `group.addMetaPermission()` thiết lập quan hệ hai chiều (bidirectional): MetaPermission tham chiếu đến Nhóm, tập hợp của Nhóm bao gồm MetaPermission. Liên kết hai chiều cho phép duyệt cả hai hướng: "nhóm Kinh doanh có những quyền nào?" và "quyền này thuộc nhóm nào?". [Từ source code]

**Bằng chứng từ mã nguồn - Tạo MetaPermissionRule với logic có điều kiện:**
```java
// Tệp: PermissionAssistantService.java, dòng 663-689
public MetaPermission updateFieldPermission(
    MetaPermission metaPermission, String field, String[] row) {

    MetaPermissionRule permissionRule = ruleRepository.all()
        .filter("self.field = ?1 and self.metaPermission.name = ?2",
                field, metaPermission.getName())
        .fetchOne();

    if (permissionRule == null) {
      permissionRule = new MetaPermissionRule();
      permissionRule.setMetaPermission(metaPermission);
      permissionRule.setField(field);  // ← Tên trường dưới dạng chuỗi
    }

    permissionRule.setCanRead(row[0].equalsIgnoreCase("x"));
    permissionRule.setCanWrite(row[1].equalsIgnoreCase("x"));
    permissionRule.setCanExport(row[4].equalsIgnoreCase("x"));
    permissionRule.setReadonlyIf(row[5]);  // ← Biểu thức chỉ đọc có điều kiện
    permissionRule.setHideIf(row[6]);      // ← Biểu thức ẩn có điều kiện

    metaPermission.addRule(permissionRule);

    return metaPermission;
}
```

**Giải thích mã nguồn:** Mẫu lấy-hoặc-tạo tương tự cho MetaPermissionRule. Truy vấn sử dụng bộ lọc kết hợp khớp cả tên trường VÀ MetaPermission cha — cần thiết vì cùng tên trường xuất hiện ở nhiều đối tượng (mọi thực thể đều có trường "id", phải phân biệt quy tắc SaleOrder.id với Invoice.id). Tên trường được lưu dưới dạng chuỗi (`permissionRule.setField(field)`) thay vì tham chiếu đến thực thể MetaField — liên kết lỏng hơn cho phép quyền trường tồn tại qua các thay đổi lược đồ (đổi tên trường trong mã, cập nhật tên quyền trường riêng).

Chỉ số dòng `[0]`, `[1]`, `[4]`, `[5]`, `[6]` cho thấy bố cục cột CSV: cột 0-4 là canRead/canWrite/canCreate/canRemove/canExport (song song với quyền đối tượng), cột 5-6 là readonlyIf/hideIf dành riêng cho trường. Khoảng trống (bỏ qua canCreate/canRemove cho trường) duy trì căn chỉnh cột với dòng quyền đối tượng — CSV có cấu trúc đồng nhất dù dòng biểu diễn đối tượng hay trường. [Từ source code]

**Hiển thị trường có điều kiện:** Các trường `readonlyIf` và `hideIf` chứa **biểu thức** được đánh giá khi chạy để kiểm soát hiển thị trường một cách động dựa trên trạng thái bản ghi và ngữ cảnh người dùng. Ngôn ngữ biểu thức có thể là Groovy (ngôn ngữ kịch bản của Axelor) hoặc JavaScript. Ví dụ về logic có điều kiện mạnh mẽ:

```groovy
// readonlyIf: Trường chiết khấu chỉ đọc sau khi đơn hàng được xác nhận
"statusSelect > 1"  // Trạng thái > Nháp

// hideIf: Ẩn giá vốn nội bộ với người không phải quản trị
"!__user__.group.technicalStaff"  // Nhóm người dùng không phải nhân viên kỹ thuật

// readonlyIf: Trường chỉ đọc trừ khi người dùng là người tạo
"createdBy.id != __user__.id"  // Được tạo bởi người dùng khác

// hideIf: Ẩn trường dựa trên loại đơn hàng
"typeSelect != 3"  // Không phải loại đơn hàng đặc biệt
```

Biểu thức truy cập cả trường bản ghi (`statusSelect`, `createdBy.id`, `typeSelect`) lẫn ngữ cảnh người dùng (biến `__user__`), cho phép giao diện nhạy ngữ cảnh. Việc triển khai yêu cầu bộ khung đánh giá biểu thức cho từng trường trên từng bản ghi trong quá trình hiển thị — cần cân nhắc hiệu năng cho biểu mẫu hoặc lưới dữ liệu lớn. Đệm kết quả biên dịch biểu thức (phân tích một lần, đánh giá nhiều lần) là yếu tố then chốt. [Suy luận]

**Ví dụ trường hợp sử dụng - Ma trận quyền nhóm kinh doanh:**
```
Đối tượng: SaleOrder (Đơn hàng bán)
Quyền trường cho nhóm "sales_team":
- clientPartner: canRead=CÓ, canWrite=CÓ (có thể sửa khách hàng)
- ourCompany: canRead=CÓ, canWrite=KHÔNG (xem công ty, không thể đổi)
- exTaxTotal: canRead=CÓ, canWrite=KHÔNG (xem tổng trước thuế, không thể thao túng)
- inTaxTotal: canRead=CÓ, canWrite=KHÔNG (xem tổng sau thuế, không thể thao túng)
- discountAmount: canRead=CÓ, canWrite=CÓ, readonlyIf="statusSelect > 2"
    (chỉnh sửa được khi nháp/xác nhận, chỉ đọc sau đó)
- costPrice: canRead=KHÔNG, hideIf="true" (ẩn hoàn toàn — bảo vệ biên lợi nhuận)
- internalNotes: canRead=CÓ, canWrite=CÓ (có thể thêm ghi chú nội bộ)
```

Ma trận này đảm bảo: nhóm kinh doanh có thể tạo/sửa đơn hàng nhưng không thể thao túng tổng tính toán (ngăn gian lận), không thể xem giá vốn (ngăn tiết lộ biên lợi nhuận), không thể sửa chiết khấu sau giai đoạn quy trình nhất định (ngăn thay đổi sau khi duyệt).

---

### 9. QUẢN LÝ QUYỀN HẠN: CÔNG CỤ NHẬP/XUẤT CSV

**Tệp nguồn:** PermissionAssistantService.java - Phân tích toàn bộ lớp [Từ source code]

Quản lý hàng trăm quyền hạn xuyên suốt hàng chục nhóm/vai trò và hàng trăm thực thể thông qua biểu mẫu giao diện sẽ rất tẻ nhạt và dễ sai sót. Axelor cung cấp **Trợ lý quyền hạn** (Permission Assistant) — công cụ nhập/xuất CSV tinh vi cho phép quản lý quyền hàng loạt bằng phần mềm bảng tính quen thuộc. Kiến trúc công cụ tuân theo mẫu ETL (Trích xuất - Biến đổi - Nạp): xuất quyền hiện tại ra CSV (Trích xuất), chỉnh sửa trong Excel/LibreOffice (Biến đổi), nhập lại CSV đã sửa (Nạp). Cách tiếp cận này tận dụng kỹ năng bảng tính của quản trị viên — hầu hết quản trị viên thành thạo Excel, có thể dùng công thức/điền/lọc để cấu hình quyền nhanh chóng.

Định dạng CSV mã hóa khéo léo cấu trúc quyền phân cấp (đối tượng → trường) trong định dạng bảng phẳng. Các dòng tiêu đề xác định nhóm/vai trò (mỗi nhóm có nhiều cột: Đọc/Ghi/Tạo/Xóa/Xuất/ĐiềuKiện/ThamSố/ChỉĐọcNếu/ẨnNếu). Các dòng dữ liệu biểu diễn đối tượng (quyền cấp thực thể) hoặc trường (thụt lề dưới đối tượng cha). Ví dụ định dạng:

```csv
;;;;;Nhóm: Kinh doanh;;;;;;;Nhóm: Quản lý;;;;;;;
Đối tượng;Trường;Tiêu đề;;Đ;G;T;X;Xu;ĐK;TS;ĐN;Ẩn;;Đ;G;T;X;Xu;ĐK;TS;ĐN;Ẩn
com.axelor.apps.sale.db.SaleOrder;;;x;x;x;;x;self.company=?;__user__.activeCompany;;;x;x;x;x;x;;;;
;clientPartner;Khách hàng;;;x;x;;;;;;;x;x;;;;;;
;discountAmount;Chiết khấu;;;x;x;;;;;;statusSelect>2;;x;x;;;;;;
```

Dòng đầu (tiêu đề nhóm) trải trên nhiều cột mỗi nhóm. Dòng thứ hai (tiêu đề cột) định nghĩa ý nghĩa từng cột. Các dòng sau trộn quyền cấp đối tượng (không có tên trường) và cấp trường (có tên trường). Dấu chấm phẩy phân tách cột, ô trống biểu diễn "không có quyền" hoặc "không áp dụng". [Từ source code]

**Quy trình nhập cung cấp kiểm tra hợp lệ và cập nhật theo giao dịch:**

Các kiểm tra hợp lệ khi nhập bao gồm: (1) **kiểm tra tiêu đề** — tên nhóm/vai trò trong CSV phải khớp với bản ghi cơ sở dữ liệu hiện có, ngăn lỗi chính tả tạo quyền mồ côi; (2) **kiểm tra đối tượng** — tên lớp thực thể phải tồn tại trong siêu mô hình JPA (kiểm tra qua `isValidObject()`), ngăn quyền cho thực thể không tồn tại; (3) **kiểm tra trường** — tên trường phải tồn tại trên đối tượng được chỉ định, ngăn lỗi chính tả trong quyền trường; (4) **kiểm tra cú pháp** — biểu thức điều kiện phải phân tích được (dù kiểm tra có thể không nghiêm ngặt — lỗi bị bắt khi chạy).

Hành vi giao dịch là yếu tố then chốt: toàn bộ cập nhật quyền được bọc trong một giao dịch cơ sở dữ liệu duy nhất. Nếu bất kỳ kiểm tra nào thất bại hoặc lỗi xảy ra giữa chừng, toàn bộ nhập được hoàn tác — ngăn cập nhật quyền một phần khiến hệ thống ở trạng thái không nhất quán. Cách tiếp cận không có giao dịch sẽ tạo quyền cho N đối tượng đầu rồi thất bại ở đối tượng N+1, để lại một số nhóm được cấu hình còn nhóm khác thì không. [Từ source code]

**Các trường hợp sử dụng nâng cao nhờ cách tiếp cận CSV:**

**Trường hợp 1: Sao chép quyền từ nhóm này sang nhóm khác** — Xuất quyền với cả nhóm "nguồn" và "đích", trong Excel sao chép cột nhóm nguồn sang nhóm đích, điều chỉnh nhỏ (đích hạn chế hơn một chút), nhập lại — nhóm đích có cùng quyền như nguồn.

**Trường hợp 2: Sinh quyền hàng loạt bằng công thức** — Xuất ma trận quyền rỗng, dùng công thức Excel: `=IF(ISNUMBER(SEARCH("sale", B2)), "x", "")` cấp tất cả đối tượng liên quan đến kinh doanh; dùng VLOOKUP áp dụng mẫu quyền chuẩn (mẫu người xem, mẫu người sửa, mẫu quản trị), nhập lại — hàng trăm quyền được cấu hình bằng công thức.

**Trường hợp 3: Kiểm toán và rà soát quyền** — Xuất quyền hiện tại, dùng định dạng có điều kiện trong Excel để làm nổi bật tổ hợp nguy hiểm (ví dụ: "canRemove=x trên thực thể tài chính"), dùng bảng tổng hợp (pivot table) phân tích: nhóm nào có quyền xóa? đối tượng nào mở hoàn toàn cho mọi nhóm? — nhận diện nhóm được cấp quyền quá mức.

**Trường hợp 4: Kiểm soát phiên bản và theo dõi thay đổi** — Xuất quyền ra CSV hàng tháng, lưu tệp CSV vào kho Git, so sánh giữa các phiên bản cho thấy thay đổi quyền theo thời gian, hoàn tác về trạng thái quyền trước đó bằng cách nhập CSV lịch sử.

**Cân nhắc bảo mật:** Nhập CSV mạnh mẽ nhưng nguy hiểm — nhập CSV độc hại có thể cấp quyền quá mức hoặc xóa sạch quyền hiện có. Truy cập Trợ lý quyền hạn nên được giới hạn cho quản trị viên bảo mật. Nhập nên ghi nhật ký mọi thay đổi (ai nhập, khi nào, tệp nào, thay đổi gì) cho nhật ký kiểm toán. Thực hành tốt: xuất trước khi nhập (sao lưu), rà soát thay đổi trong công cụ so sánh CSV trước khi nhập. [Suy luận]

---

### 10. CẤU HÌNH XÁC THỰC: HỖ TRỢ ĐA NHÀ CUNG CẤP PAC4J

**Tệp nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Cấu hình xác thực của Axelor thể hiện tính linh hoạt cấp doanh nghiệp thông qua tích hợp thư viện Pac4j, hỗ trợ đồng thời sáu nhà cung cấp xác thực khác nhau. Cấu hình tuân theo nguyên tắc quy ước hơn cấu hình (convention-over-configuration): hầu hết thiết lập ở dạng ghi chú (bị tắt mặc định), quản trị viên bỏ ghi chú và điền thông tin chỉ cho nhà cung cấp cần dùng. Cấu trúc cấu hình không phụ thuộc nhà cung cấp (`auth.provider.{tên}.{thuộc_tính}`) cho phép thêm nhà cung cấp tùy chỉnh mà không cần thay đổi mã bộ khung.

Cấu hình phiên làm việc (session) kiểm soát vòng đời phiên người dùng — các tham số bảo mật quan trọng. Thiết lập `session.timeout = 480` (8 giờ) cân bằng giữa bảo mật (tự động đăng xuất sau thời gian không hoạt động giảm rủi ro truy cập trái phép vào phiên bị bỏ rơi) và tính tiện dụng (người dùng không bị đăng xuất giữa ngày làm việc). Thiết lập `session.cookie.secure = true` đang ở dạng ghi chú nhưng nên được bật trong môi trường chính thức để truyền cookie chỉ qua HTTPS — ngăn chiếm đoạt phiên qua kết nối không mã hóa. [Từ source code]

**Bằng chứng từ mã nguồn - Cấu hình phiên làm việc:**
```properties
session.timeout = 480                    # 8 giờ (tính bằng phút)
#session.cookie.secure = true            # Chỉ HTTPS (bỏ ghi chú trong sản xuất!)
```

**Giải thích mã nguồn:** Giá trị hết hạn 480 phút = 8 giờ giả định ngày làm việc tiêu chuẩn — người dùng đăng nhập buổi sáng, có thể làm việc cả ngày mà không cần xác thực lại. Các giá trị hết hạn ngắn hơn (30-60 phút) phù hợp cho môi trường bảo mật cao (ngân hàng, y tế) chấp nhận đánh đổi tính tiện dụng. Cờ bảo mật cookie ở dạng ghi chú cho thấy giá trị mặc định thân thiện với phát triển (cho phép kiểm thử HTTP) — nguy hiểm nếu quên bật trong sản xuất. Danh sách kiểm tra kiểm toán bảo mật nên xác minh thiết lập này được bật cho triển khai công khai trên Internet. [Từ source code]

**Bằng chứng từ mã nguồn - Thiết lập xác thực toàn cục:**
```properties
#auth.provider-order =                   # Tên nhà cung cấp phân cách bằng dấu phẩy
#auth.callback-url =                     # URL gọi ngược OAuth
#auth.user.provisioning = none           # create / link / none
#auth.user.default-group = users         # Nhóm mặc định cho người dùng mới
#auth.user.principal-attribute = email   # Thuộc tính dùng làm tên chủ thể
```

**Giải thích mã nguồn:** Thứ tự nhà cung cấp quyết định chuỗi xác thực phân tầng: nếu nhà cung cấp đầu tiên từ chối (không tìm thấy người dùng), thử nhà cung cấp thứ hai, v.v. Ví dụ: `auth.provider-order = ldap,google,local` thử LDAP trước (thư mục doanh nghiệp), dự phòng Google OAuth (đối tác bên ngoài), cuối cùng xác thực nội bộ (truy cập khẩn cấp). URL gọi ngược (callback URL) quan trọng cho luồng OAuth/SAML — nhà cung cấp chuyển hướng người dùng về URL này sau xác thực, phải truy cập công khai được và khớp chính xác với đăng ký nhà cung cấp (sai khớp gây thất bại xác thực).

Thiết lập cấp phát tài khoản người dùng (provisioning) kiểm soát việc xác thực bên ngoài có tự động tạo người dùng hay không: `create` cho phép đăng ký tự phục vụ (người dùng OAuth Google xác thực, tài khoản tự tạo trong Axelor) — tiện nhưng có rủi ro bảo mật nếu miền OAuth công khai; `link` yêu cầu quản trị viên tạo tài khoản trước (kiểm soát chặt) nhưng cho phép xác thực linh hoạt (người dùng có thể đăng nhập qua LDAP HOẶC Google dùng cùng tài khoản); `none` nghiêm ngặt nhất — chỉ người dùng được tạo tường minh mới đăng nhập được. [Từ source code]

**Bằng chứng từ mã nguồn - Xác thực nội bộ với xác thực cơ bản (Basic Auth):**
```properties
#auth.local.basic-auth = indirect, direct  # Bật xác thực cơ bản HTTP
```

**Giải thích mã nguồn:** Xác thực cơ bản (Basic Auth) cho phép xác thực API mà không cần phiên trình duyệt — máy khách gửi tiêu đề `Authorization: Basic <base64(tên:mật_khẩu)>` mỗi yêu cầu. Chế độ `indirect` yêu cầu chuyển hướng đến trang đăng nhập cho trình duyệt, `direct` cho phép xác thực trực tiếp cho máy khách lập trình. Cảnh báo bảo mật: xác thực cơ bản truyền thông tin đăng nhập với mọi yêu cầu (dù đã mã hóa base64), HTTPS bắt buộc để ngăn chặn đánh cắp thông tin. Các phương thức thay thế hiện đại (mã thông báo JWT, luồng thông tin OAuth 2.0) an toàn hơn cho API. [Từ source code]

**Bằng chứng từ mã nguồn - Tích hợp Google OpenID Connect:**
```properties
#auth.provider.google.client-id =        # Google Cloud Console - Mã khách OAuth
#auth.provider.google.secret =           # Bí mật khách (giữ bí mật!)
```

**Bằng chứng từ mã nguồn - Tích hợp Keycloak cho SSO doanh nghiệp:**
```properties
#auth.provider.keycloak.client-id = demo-app
#auth.provider.keycloak.secret = 233d1690-4498-490c-a60d-5d12bb685557
#auth.provider.keycloak.realm = demo-app
#auth.provider.keycloak.base-uri = http://localhost:8083/auth
```

**Giải thích mã nguồn:** Keycloak là nền tảng quản lý danh tính và truy cập mã nguồn mở phổ biến trong hệ sinh thái Java doanh nghiệp. Khái niệm vương quốc (realm) cho phép đa thuê bao (multi-tenancy) trong Keycloak (vương quốc demo-app cách ly người dùng kiểm thử khỏi sản xuất). URI cơ sở trỏ đến máy chủ Keycloak — có thể là máy chủ nội bộ doanh nghiệp hoặc đám mây. Lợi ích so với Google OAuth: toàn quyền kiểm soát (tự lưu trữ), tích hợp với hệ thống doanh nghiệp (đồng bộ LDAP, liên kết Active Directory), tính năng nâng cao (xác thực hai yếu tố, chính sách mật khẩu, liên kết người dùng). [Từ source code]

**Bằng chứng từ mã nguồn - Cấu hình SAML 2.0:**
```properties
#auth.provider.saml.keystore-path = {java.io.tmpdir}/samlKeystore.jks
#auth.provider.saml.keystore-password = open-platform-demo-passwd
#auth.provider.saml.private-key-password = open-platform-demo-passwd
#auth.provider.saml.identity-provider-metadata-path = http://localhost:9012/simplesaml/saml2/idp/metadata.php
#auth.provider.saml.service-provider-metadata-path = {java.io.tmpdir}/sp-metadata.xml
#auth.provider.saml.service-provider-entity-id = sp.test.pac4j
```

**Giải thích mã nguồn:** SAML phức tạp hơn OAuth — đòi hỏi trao đổi siêu dữ liệu lẫn nhau và khóa mật mã để ký/xác minh xác nhận (assertion). Kho khóa (keystore) chứa chứng chỉ để ký yêu cầu SAML và giải mã phản hồi. Siêu dữ liệu nhà cung cấp danh tính (từ IdP doanh nghiệp như Okta, Azure AD, ADFS) mô tả điểm cuối (endpoint) và chứng chỉ của IdP. Siêu dữ liệu nhà cung cấp dịch vụ (siêu dữ liệu của Axelor) được gửi cho IdP mô tả URL gọi ngược và định dạng xác nhận mong đợi. Độ phức tạp SAML được biện minh cho doanh nghiệp yêu cầu: giao thức chuẩn hóa (không bị khóa nhà cung cấp), kiểm soát phát hành thuộc tính (thuộc tính người dùng nào được chia sẻ), truyền đăng xuất (đăng xuất từ IdP đăng xuất khỏi tất cả SP). [Từ source code]

**Bằng chứng từ mã nguồn - Tích hợp LDAP:**
```properties
#auth.ldap.server.url = ldap://localhost:389
#auth.ldap.server.starttls = false              # Mã hóa kết nối
#auth.ldap.server.auth.type = simple            # simple / CRAM-MD5 / DIGEST-MD5 / EXTERNAL / GSSAPI
#auth.ldap.server.auth.user = cn=admin,dc=test,dc=com
#auth.ldap.server.auth.password = admin
#auth.ldap.group.base = ou=groups,dc=test,dc=com
#auth.ldap.group.filter = (uniqueMember=uid={0})
#auth.ldap.user.base = ou=users,dc=test,dc=com
#auth.ldap.user.filter = (uid={0})
#auth.ldap.user.id-attribute = uid
```

**Giải thích mã nguồn:** Cấu hình LDAP tuân theo quy ước cấu trúc thư mục (Tên phân biệt - Distinguished Name). Thiết lập xác thực máy chủ định nghĩa thông tin đăng nhập của Axelor để kết nối máy chủ LDAP (tài khoản dịch vụ). Cơ sở và bộ lọc người dùng mô tả nơi lưu trữ người dùng và cách truy vấn (chỗ giữ chỗ `{0}` được thay thế bằng tên đăng nhập). Cơ sở và bộ lọc nhóm cho phép tra cứu thành viên nhóm (gán nhóm Axelor dựa trên nhóm LDAP). Thuộc tính mã định danh chỉ định thuộc tính LDAP nào ánh xạ sang tên đăng nhập Axelor (uid, sAMAccountName, mail tùy lược đồ thư mục). Thiết lập STARTTLS nên được bật (true) cho sản xuất — mã hóa lưu lượng LDAP ngăn đánh cắp mật khẩu. [Từ source code]

**Bằng chứng từ mã nguồn - Cấu hình CAS:**
```properties
#auth.provider.cas.login-url = https://localhost:8443/cas/login
#auth.provider.cas.prefix-url = https://localhost:8443/cas
#auth.provider.cas.protocol = CAS30       # CAS10 / CAS20 / CAS20_PROXY / CAS30 / CAS30_PROXY / SAML
```

**Giải thích mã nguồn:** CAS (Dịch vụ xác thực tập trung - Central Authentication Service) là giao thức SSO cũ do Đại học Yale phát triển, vẫn được dùng trong tổ chức học thuật và môi trường doanh nghiệp kế thừa. Lựa chọn phiên bản giao thức quan trọng — CAS 3.0 hỗ trợ phát hành thuộc tính (siêu dữ liệu người dùng ngoài tên đăng nhập), CAS 1.0/2.0 chỉ cung cấp tên đăng nhập. Biến thể proxy cho phép xác thực dịch vụ-sang-dịch vụ (Axelor có thể lấy vé để gọi dịch vụ khác được CAS bảo vệ thay mặt người dùng). Tổ chức đang chuyển từ CAS sang OAuth/SAML hiện đại có thể chạy đồng thời cả hai trong thời gian chuyển đổi. [Từ source code]

**Bằng chứng từ mã nguồn - Cấu hình đăng xuất:**
```properties
#auth.logout.default-url =                # Chuyển hướng sau đăng xuất
#auth.logout.url-pattern =                # Mẫu URL kích hoạt đăng xuất
#auth.logout.local = true                 # Xóa hồ sơ khỏi phiên
#auth.logout.central = false              # Gọi điểm cuối đăng xuất IdP (đăng xuất SSO)
```

**Giải thích mã nguồn:** Đăng xuất cục bộ (mặc định) chỉ xóa phiên Axelor, phiên IdP vẫn hoạt động — người dùng có thể đăng nhập lại ngay mà không cần nhập lại mật khẩu. Đăng xuất tập trung gọi điểm cuối đăng xuất của IdP (Đăng xuất đơn SAML, thu hồi OAuth), kết thúc phiên SSO toàn cục — người dùng bị đăng xuất khỏi tất cả ứng dụng. Đánh đổi: đăng xuất tập trung bảo mật hơn (người dùng cố ý đăng xuất, phiên nên được kết thúc ở mọi nơi) nhưng phức tạp trong triển khai (đòi hỏi IdP hỗ trợ, truyền đăng xuất đáng tin cậy) và có thể gây bất tiện cho người dùng (đăng xuất khỏi một ứng dụng đăng xuất luôn email, lịch, mọi thứ). [Từ source code]

---

### 11. ĐA THUÊ BAO VÀ CÁCH LY DỮ LIỆU

**Tệp nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Tùy chọn cấu hình đa thuê bao (multi-tenancy) tồn tại trong tệp cấu hình Axelor nhưng chi tiết triển khai hầu như không có trong mã nguồn ứng dụng đã phân tích — cho thấy tính năng được triển khai chủ yếu ở tầng lõi bộ khung thay vì tầng ứng dụng. Thuộc tính cấu hình `application.multi-tenancy` kiểm soát bật/tắt, với giá trị mặc định `false` chỉ ra đa thuê bao là tính năng tùy chọn (phải bật tường minh).

**Bằng chứng từ mã nguồn - Cấu hình đa thuê bao:**
```properties
# Bật đa thuê bao
#application.multi-tenancy = false
```

**Giải thích mã nguồn:** Cấu hình ở dạng ghi chú với giá trị mặc định `false` gợi ý hầu hết triển khai Axelor chạy chế độ đơn thuê bao. Độ phức tạp đa thuê bao (cách ly thuê bao, truy vấn xuyên thuê bao, tùy chỉnh riêng thuê bao) đòi hỏi hạ tầng đáng kể — chỉ bật khi cần giúp giảm phức tạp vận hành. Sự thiếu vắng cấu hình đa thuê bao bổ sung (chiến lược xác định thuê bao, ánh xạ cơ sở dữ liệu thuê bao, đường dẫn tùy chỉnh thuê bao) gợi ý hoặc yêu cầu cấu hình tối thiểu (bộ khung xử lý nội bộ) hoặc tính năng chưa phát triển đầy đủ. [Từ source code]

**Chi tiết triển khai không tìm thấy:** [Không tìm thấy trong mã nguồn]

Mã nguồn axelor-open-suite đã phân tích KHÔNG chứa: lớp giải quyết thuê bao (cách bộ khung xác định thuê bao hiện tại từ yêu cầu HTTP), quản lý ngữ cảnh thuê bao (lưu trữ thuê bao theo luồng cục bộ, truyền ngữ cảnh), bộ chặn lọc thuê bao (lọc truy vấn tự động theo mã thuê bao), tùy chỉnh lược đồ riêng thuê bao (trường/thực thể khác nhau mỗi thuê bao).

**Cách tiếp cận đa thuê bao suy luận:** [Suy luận từ tính năng hiện có]

Xét thực thể User có trường `activeCompany` và quyền hỗ trợ `self.company = ?` với `__user__.activeCompany`, cách triển khai đa thuê bao có thể là **đa thuê bao mềm** (lược đồ chung, lọc cấp dòng) thay vì **đa thuê bao cứng** (lược đồ riêng mỗi thuê bao). Lý do: (1) Công ty đóng vai trò bộ phân biệt thuê bao — ngữ cảnh công ty đang hoạt động của người dùng đóng vai trò ngữ cảnh thuê bao; (2) lọc truy vấn tự động — điều kiện quyền chèn bộ lọc công ty vào truy vấn; (3) lược đồ cơ sở dữ liệu chung — dữ liệu mọi thuê bao trong cùng bảng, phân biệt bằng khóa ngoại công ty; (4) cách ly ở tầng ứng dụng — mã bộ khung đảm bảo người dùng chỉ thấy/sửa dữ liệu công ty mình.

Đánh đổi đa thuê bao mềm: **Ưu điểm** gồm triển khai đơn giản (một cơ sở dữ liệu), truy vấn xuyên thuê bao dễ dàng (cho siêu quản trị), sử dụng tài nguyên hiệu quả (hạ tầng chung). **Nhược điểm** gồm cách ly yếu hơn (lỗi ứng dụng có thể lộ dữ liệu thuê bao), tuân thủ khó hơn (dữ liệu trộn lẫn vật lý), tối ưu truy vấn phức tạp (mọi truy vấn cần bộ lọc công ty). [Suy luận]

---

### 12. LUỒNG GIẢI QUYẾT QUYỀN HẠN VÀ ĐÁNH GIÁ KHI CHẠY

**Tệp nguồn:** Phân tích quan hệ thực thể và logic dịch vụ [Suy luận từ các mẫu mã]

Luồng giải quyết quyền hạn biểu diễn quy trình khi chạy khi người dùng thực hiện thao tác và bộ khung quyết định cho phép hay từ chối. Luồng không được mô tả tường minh trong mã đã phân tích (triển khai trong lõi bộ khung) nhưng có thể suy luận từ quan hệ thực thể và mẫu dịch vụ quyền. Hiểu luồng này rất quan trọng cho gỡ lỗi vấn đề quyền hạn và thiết kế sơ đồ phân quyền hiệu quả.

**Luồng giải quyết suy luận:**

```
1. Xác thực người dùng
   ↓
2. Tải thực thể User
   - Truy vấn User theo tên đăng nhập/email
   - Tải háo hức (eager load) Nhóm (quan hệ nhiều-một)
   - Tải lười (lazy load) Vai trò (quan hệ nhiều-nhiều, tải khi cần)
   ↓
3. Tổng hợp quyền cấp đối tượng
   - Truy vấn thực thể Permission thuộc Nhóm của người dùng
   - Truy vấn thực thể Permission thuộc các Vai trò của người dùng
   - Hợp nhất tất cả quyền (phép hợp dựa trên cấp phép)
   ↓
4. Tổng hợp quyền cấp trường
   - Truy vấn thực thể MetaPermission thuộc Nhóm của người dùng
   - Truy vấn thực thể MetaPermission thuộc các Vai trò của người dùng
   - Với mỗi MetaPermission, tải các MetaPermissionRule liên quan
   - Hợp nhất tất cả quy tắc trường (phép hợp dựa trên cấp phép)
   ↓
5. Đệm kết quả quyền
   - Lưu quyền đã tổng hợp trong phiên người dùng
   - Yêu cầu sau dùng quyền đã đệm (không truy vấn lại)
   - Vô hiệu bộ đệm khi quyền thay đổi hoặc phiên hết hạn
   ↓
6. Kiểm tra quyền khi chạy (mỗi thao tác)
   - Xác định đối tượng đích và thao tác (ĐỌC/GHI/TẠO/XÓA/XUẤT)
   - Tra cứu quyền trong bộ đệm: người dùng có quyền cho đối tượng+thao tác?
   - Nếu không có quyền trực tiếp, kiểm tra quyền ký tự đại diện (cấp gói)
   - Nếu quyền có điều kiện, đánh giá điều kiện
   ↓
7. Đánh giá điều kiện (nếu có)
   - Phân tích conditionParams, giải quyết biến __user__
   - Chèn điều kiện vào mệnh đề WHERE của truy vấn
   - Thực thi truy vấn đã lọc (cơ sở dữ liệu chỉ trả dòng khớp)
   ↓
8. Áp dụng quyền trường (cho hiển thị giao diện)
   - Với mỗi trường trong biểu mẫu/lưới, kiểm tra MetaPermissionRule
   - Áp dụng canRead: ẩn trường hoàn toàn nếu false
   - Áp dụng canWrite: đặt trường chỉ đọc nếu false
   - Đánh giá biểu thức readonlyIf/hideIf với ngữ cảnh bản ghi
   - Hiển thị giao diện cuối cùng với trường hiển thị/chỉnh sửa phù hợp
   ↓
9. Trả kết quả đã lọc
   - Người dùng chỉ thấy bản ghi khớp điều kiện quyền
   - Người dùng chỉ thấy trường được phép bởi quyền trường
   - Người dùng chỉ thực hiện được thao tác được cấp phép
```

**Ưu tiên quyền và chiến lược hợp nhất:** [Suy luận từ mẫu dựa trên cấp phép]

Khi người dùng thuộc Nhóm có một số quyền VÀ có Vai trò với quyền bổ sung, quyền hiệu lực là **phép hợp** (phép HOẶC logic): nếu Nhóm cấp canRead, người dùng có canRead (dù Vai trò không cấp); nếu Vai trò cấp canWrite, người dùng có canWrite (dù Nhóm không cấp). Không quan sát thấy quy tắc TỪ CHỐI tường minh — từ chối thông qua sự vắng mặt của cấp phép. [Suy luận]

**Cân nhắc hiệu năng:** [Suy luận về chiến lược tối ưu]

Kiểm tra quyền trên mỗi truy vấn cơ sở dữ liệu có thể ảnh hưởng nghiêm trọng đến hiệu năng nếu không được tối ưu. Bộ khung có thể triển khai: (1) đệm cấp phiên — tải quyền một lần mỗi lần đăng nhập, tái sử dụng đến khi đăng xuất; (2) đệm kế hoạch truy vấn — biên dịch biểu thức điều kiện một lần, tái sử dụng cho nhiều truy vấn; (3) kiểm tra quyền theo lô — kiểm tra quyền cho tập hợp đối tượng cùng lúc, không kiểm tra riêng lẻ; (4) đánh giá lười — chỉ kiểm tra quyền khi thực sự cần (không kiểm tra đầu cơ). [Suy luận]

---

### 13. BIẾN NGỮ CẢNH VÀ THAM SỐ QUYỀN ĐỘNG

**Tệp nguồn:** PermissionAssistantService.java dòng 326 [Từ source code]

Biến ngữ cảnh trong điều kiện quyền cho phép đưa ra quyết định phân quyền động dựa trên thuộc tính của người dùng hiện tại — biến đổi quy tắc quyền tĩnh thành kiểm soát truy cập thích ứng. Mẫu biến đặc biệt `__user__.{tenTruong}` cung cấp truy cập trực tiếp đến các trường thực thể của người dùng đang đăng nhập, bộ khung xử lý giải quyết khi chạy và chuyển đổi kiểu một cách trong suốt.

**Mẫu biến ngữ cảnh được ghi nhận:**
```
__user__.{tenTruong}
```

Trong đó `{tenTruong}` có thể là bất kỳ trường nào trên thực thể User, bao gồm:
- Trường trực tiếp: `__user__.code`, `__user__.name`, `__user__.language`
- Thực thể liên quan: `__user__.activeCompany`, `__user__.activeTeam`, `__user__.group`, `__user__.partner`
- Tập hợp: `__user__.companySet`, `__user__.teamSet`
- Đường dẫn lồng nhau: `__user__.partner.company`, `__user__.group.technicalStaff`, `__user__.activeCompany.currency`

**Bằng chứng từ mã nguồn - Xây dựng biến:**
```java
// Tệp: PermissionAssistantService.java, dòng 326
String conditionParams = "__user__." + userField.getName();
```

**Giải thích mã nguồn:** Phép nối chuỗi đơn giản xây dựng tham chiếu biến đặc biệt. Khi chạy, bộ khung phải phân tích chuỗi, tách theo dấu chấm (`.`), duyệt đồ thị đối tượng từ thực thể User qua đường dẫn được chỉ định, trích xuất giá trị cuối cùng. Triển khai có thể dùng phản chiếu (reflection) hoặc bộ truy cập thuộc tính (property accessor) để duyệt quan hệ một cách động. Xử lý lỗi rất quan trọng: nếu đường dẫn không hợp lệ (lỗi chính tả tên trường) hoặc gặp giá trị null (user.partner là null vì người dùng chưa liên kết đối tác), bộ khung phải trả về null một cách nhẹ nhàng hoặc ném lỗi rõ ràng. [Từ source code]

**Các kịch bản sử dụng minh họa tính linh hoạt:**

**Kịch bản 1: So sánh đơn giá trị**
```
Điều kiện: "self.createdBy = ?"
Tham số: "__user__"
→ SQL: WHERE created_by_id = {mã người dùng hiện tại}
Trường hợp sử dụng: Người dùng chỉ thấy bản ghi do mình tạo
```

**Kịch bản 2: Lọc theo khóa ngoại**
```
Điều kiện: "self.company = ?"
Tham số: "__user__.activeCompany"
→ SQL: WHERE company_id = {mã công ty đang hoạt động của người dùng}
Trường hợp sử dụng: Cách ly dữ liệu đa công ty
```

**Kịch bản 3: Thuộc tập hợp (mệnh đề IN)**
```
Điều kiện: "self.assignedTeam in (?)"
Tham số: "__user__.teamSet"
→ SQL: WHERE assigned_team_id IN (1,2,3)  -- các nhóm của người dùng
Trường hợp sử dụng: Hiển thị bản ghi theo nhóm làm việc
```

**Kịch bản 4: Duyệt đường dẫn lồng nhau**
```
Điều kiện: "self.currency = ?"
Tham số: "__user__.activeCompany.currency"
→ SQL: WHERE currency_id = {mã tiền tệ của công ty người dùng}
Trường hợp sử dụng: Bản ghi theo tiền tệ khớp tiền tệ công ty người dùng
```

**Kịch bản 5: Kiểm tra cờ boolean**
```
Điều kiện: "self.confidential = false OR ? = true"
Tham số: "__user__.group.technicalStaff"
→ SQL: WHERE (confidential = false OR {là nhân viên kỹ thuật})
Trường hợp sử dụng: Bản ghi mật chỉ hiển thị với nhân viên kỹ thuật
```

**Chuyển đổi kiểu và giải quyết giá trị:** [Suy luận về triển khai]

Bộ khung phải xử lý chuyển đổi kiểu khi giải quyết biến ngữ cảnh: tham chiếu thực thể chuyển sang mã định danh (đối tượng User → giá trị user.id kiểu Long); tập hợp mở rộng thành danh sách mã định danh (Set<Team> → List<Long> mã nhóm); kiểu nguyên thủy dùng trực tiếp (String, Integer, Boolean); giá trị null xử lý nhẹ nhàng (công ty null → điều kiện loại trừ mọi bản ghi HOẶC ném lỗi).

**Ý nghĩa bảo mật của biến ngữ cảnh:** Biến ngữ cảnh mạnh mẽ nhưng cần sử dụng cẩn thận: rò rỉ thông tin — điều kiện tham chiếu trường `__user__` có thể vô tình lộ trường đó trong nhật ký gỡ lỗi/thông báo lỗi; leo thang đặc quyền — điều kiện sai định dạng có thể vô tình cấp quyền truy cập (ví dụ lỗi chính tả `self.company != ?` thay vì `self.company = ?` đảo ngược bộ lọc); hiệu năng — đường dẫn lồng sâu (`__user__.partner.company.parent.currency`) đòi hỏi nhiều phép nối, truy vấn chậm; bảo trì — đổi tên trường thực thể User làm hỏng điều kiện tham chiếu trường đó (không có kiểm tra khi biên dịch cho chuỗi điều kiện). [Suy luận]

---

### 14. ĐỆM QUYỀN HẠN VÀ TỐI ƯU HIỆU NĂNG

**Tệp nguồn:** Permission.xml, Group.xml — định nghĩa thực thể [Từ source code]

Các thực thể Permission và Group được đánh dấu `cacheable="true"` — tối ưu hiệu năng then chốt vì kiểm tra quyền xảy ra cực kỳ thường xuyên trong suốt quá trình thực thi ứng dụng. Chiến lược đệm giảm tải cơ sở dữ liệu (quyền được truy vấn một lần mỗi phiên thay vì mỗi thao tác) và cải thiện thời gian phản hồi (truy cập bộ đệm tính bằng micro giây so với truy vấn cơ sở dữ liệu tính bằng mili giây).

**Bằng chứng từ mã nguồn - Khai báo thực thể có đệm:**
```xml
<entity name="Permission" cacheable="true">
<entity name="Group" cacheable="true">
```

**Giải thích mã nguồn:** Thuộc tính cacheable hướng dẫn Hibernate lưu phiên bản thực thể trong bộ đệm cấp hai (L2 cache) — bộ đệm chia sẻ giữa tất cả phiên (không chỉ một người dùng). Bộ đệm cấp hai được cấu hình toàn cục với chế độ `ENABLE_SELECTIVE` (từ phát hiện BƯỚC 1), nghĩa là chỉ thực thể được đánh dấu cacheable tường minh mới được đệm (ngăn ô nhiễm bộ đệm từ thực thể ít truy cập). [Từ source code]

**Lợi ích đệm cho hệ thống quyền:**

1. **Giảm tải truy vấn**: Kiểm tra quyền trên mỗi yêu cầu HTTP, đệm ngăn hàng nghìn truy vấn cơ sở dữ liệu mỗi giây
2. **Cải thiện độ trễ**: Truy cập bộ đệm ~0.1ms so với truy vấn cơ sở dữ liệu ~10-50ms — nhanh hơn 100-500 lần
3. **Khả năng mở rộng**: Máy chủ ứng dụng có thể mở rộng ngang mà không quá tải cơ sở dữ liệu với truy vấn quyền
4. **Nhất quán**: Tất cả máy chủ ứng dụng chia sẻ cùng dữ liệu quyền (nếu dùng bộ đệm phân tán)

**Thách thức vô hiệu bộ đệm:**

Bộ đệm cấp hai đặt ra thách thức nhất quán — khi quyền bị sửa đổi, làm sao đảm bảo tất cả bản sao đệm được cập nhật? Các chiến lược: (1) hết hạn theo thời gian — mục bộ đệm hết hạn sau N phút, buộc làm mới — đơn giản nhưng có thể phục vụ quyền cũ; (2) vô hiệu dựa trên sự kiện — khi thực thể Permission được cập nhật, bộ khung phát sự kiện vô hiệu đến tất cả bộ đệm — phức tạp nhưng đảm bảo nhất quán; (3) vô hiệu dựa trên phiên bản — Hibernate phát hiện thay đổi phiên bản và vô hiệu tự động — có hỗ trợ sẵn. Axelor có thể dùng kết hợp: vô hiệu tự động của Hibernate (dựa trên phiên bản) cộng tổng hợp quyền cấp phiên (quyền tải một lần mỗi đăng nhập, đệm trong phiên HTTP). [Suy luận]

**Tổng hợp quyền cấp phiên:** [Suy luận về chiến lược đệm]

Ngoài bộ đệm cấp hai ở mức thực thể, ứng dụng có thể thực hiện **tổng hợp quyền cấp phiên**: tải tất cả quyền từ nhóm và vai trò của người dùng một lần khi đăng nhập, đệm kết quả đã hợp nhất trong phiên HTTP, tái sử dụng cho mọi yêu cầu tiếp theo. Bộ đệm cấp phiên lý tưởng vì quyền hiếm khi thay đổi trong một phiên người dùng. Đánh đổi: thay đổi quyền không có hiệu lực cho đến khi người dùng đăng xuất/đăng nhập lại hoặc phiên hết hạn. [Suy luận]

---

### 15. TÙY CHỌN CẤU HÌNH BẢO MẬT VÀ GIA CỐ

**Tệp nguồn:** `/src/main/resources/axelor-config.properties` [Từ source code]

Cấu hình bảo mật mở rộng ra ngoài phạm vi xác thực/phân quyền, bao gồm chính sách mật khẩu, kiểm soát quyền hạn, và bảo vệ chống tiêm mã (injection). Các thuộc tính cung cấp cách tiếp cận phòng thủ nhiều lớp (defense-in-depth) — nhiều tầng bảo mật bảo vệ trước các hướng tấn công khác nhau.

**Áp dụng chính sách mật khẩu:**

**Bằng chứng từ mã nguồn - Mẫu biểu thức chính quy mật khẩu:**
```properties
user.password.pattern = (((?=.*[a-z])(?=.*[A-Z])(?=.*\\d))|((?=.*[a-z])(?=.*[A-Z])(?=.*\\W))|((?=.*[a-z])(?=.*\\d)(?=.*\\W))|((?=.*[A-Z])(?=.*\\d)(?=.*\\W))).{8,}

#user.password.pattern-title = Custom password requirements message
```

**Giải thích mã nguồn:** Biểu thức chính quy phức tạp áp dụng yêu cầu độ mạnh mật khẩu: tối thiểu 8 ký tự VÀ ít nhất 3 trong 4 loại ký tự (chữ thường, chữ hoa, chữ số, ký tự đặc biệt). Chi tiết mẫu: `(?=.*[a-z])` — nhìn trước tích cực cho chữ thường; `(?=.*[A-Z])` — cho chữ hoa; `(?=.*\\d)` — cho chữ số; `(?=.*\\W)` — cho ký tự đặc biệt; `.{8,}` — tối thiểu 8 ký tự. Các nhóm biểu thức chính quy kết hợp bằng phép HOẶC (`|`) yêu cầu bất kỳ 3 trong 4 loại. Cách tiếp cận cân bằng bảo mật (ngăn mật khẩu yếu như "password123") với tính tiện dụng (không yêu cầu cả 4 loại cho phép linh hoạt). Thuộc tính `password-pattern-title` tùy chọn cung cấp thông báo lỗi tùy chỉnh hiển thị cho người dùng khi mật khẩu bị từ chối. [Từ source code]

**Kiểm soát hệ thống quyền (nguy hiểm!):**

**Bằng chứng từ mã nguồn - Tùy chọn bỏ qua quyền:**
```properties
# Tắt kiểm tra quyền hành động (NGUY HIỂM!)
#application.permission.disable-action = false

# Tắt kiểm tra quyền trường quan hệ
#application.permission.disable-relational-field = false
```

**Giải thích mã nguồn:** Các thuộc tính cho phép tắt hoàn toàn kiểm tra quyền — dành cho môi trường kiểm thử/phát triển nơi thiết lập quyền gây phiền phức. CẢNH BÁO: bật trong sản xuất (đặt thành `true`) tạo lỗ hổng bảo mật — bất kỳ người dùng nào đều thực hiện được bất kỳ hành động nào. Các thuộc tính ở dạng ghi chú theo mặc định (an toàn) nhưng lập trình viên có thể bỏ ghi chú trong kiểm thử rồi quên bật lại trước khi triển khai sản xuất. Danh sách kiểm tra kiểm toán bảo mật phải xác minh các thuộc tính này vẫn là `false` (hoặc ở dạng ghi chú) trong cấu hình sản xuất. [Từ source code]

**Bảo vệ chống tiêm SQL:**

**Bằng chứng từ mã nguồn - Lọc biểu thức miền:**
```properties
# Mẫu danh sách chặn cho biểu thức miền
#application.domain-blocklist-pattern = (\\(\\s*(SELECT|DELETE|UPDATE)\\s+)|query_to_xml|some_another_function
```

**Giải thích mã nguồn:** Mẫu biểu thức chính quy chặn các đoạn SQL nguy hiểm xuất hiện trong biểu thức lọc miền (điều kiện quyền). Mẫu cụ thể chặn: truy vấn con bắt đầu bằng SELECT/DELETE/UPDATE (tiềm ẩn tiêm SQL), hàm `query_to_xml` của PostgreSQL (có thể rò rỉ thông tin lược đồ), chỗ giữ chỗ cho thêm hàm nguy hiểm tùy chỉnh. Bảo vệ rất quan trọng vì điều kiện quyền cho phép quản trị viên viết biểu thức dạng SQL — biểu thức nguy hiểm do cố ý hoặc vô tình có thể xâm phạm cơ sở dữ liệu. Ví dụ bị chặn: `self.id = 1 OR (SELECT password FROM auth_user LIMIT 1) IS NOT NULL` — điều này sẽ vượt qua lọc quyền VÀ rò rỉ mật khẩu. [Từ source code]

---

### 16. NHỮNG ĐIỀU KHÔNG TÌM THẤY TRONG MÃ NGUỒN

Sau quá trình phân tích sâu mã bảo mật và phân quyền, một số tính năng phổ biến trong hệ thống bảo mật doanh nghiệp **KHÔNG** xuất hiện hoặc không rõ ràng trong mã nguồn Axelor. Ghi nhận những "tính năng vắng mặt" giúp thiết lập kỳ vọng thực tế và nhận diện hạn chế tiềm năng:

**1. Tích hợp Spring Security hoặc Apache Shiro** [Không tìm thấy]
Xác nhận Axelor **KHÔNG** sử dụng các bộ khung bảo mật Java phổ biến — thay vào đó triển khai tầng bảo mật tùy chỉnh. Hệ quả: không thể tận dụng hệ sinh thái rộng lớn của Spring Security (máy chủ tài nguyên OAuth, bảo mật cấp phương thức, tiện ích kiểm thử bảo mật), phải dựa vào API riêng của Axelor.

**2. Chi tiết triển khai đa thuê bao** [Không rõ]
Tùy chọn cấu hình tồn tại (`application.multi-tenancy`) nhưng mã triển khai không có trong tệp đã phân tích. Không tìm thấy: lớp TenantResolver, TenantContext, tùy chỉnh lược đồ riêng thuê bao, API truy vấn xuyên thuê bao. Có thể được triển khai trong lõi bộ khung, không tiếp cận được ở tầng ứng dụng.

**3. Thuật toán băm mật khẩu** [Không rõ]
Trường mật khẩu không hiển thị trong XML mở rộng User đã phân tích (phải nằm trong lõi bộ khung). Không biết thuật toán sử dụng: BCrypt (tiêu chuẩn ngành)? PBKDF2? SCrypt? Argon2? Số vòng lặp băm? Cơ chế sinh muối (salt)? Quan trọng để đánh giá bảo mật mật khẩu nhưng không có tài liệu trong mã tiếp cận được.

**4. Cơ chế lưu trữ phiên** [Không rõ]
Cấu hình đặt thời gian hết phiên nhưng không chỉ định lưu trữ: trong bộ nhớ (mất khi khởi động lại)? cơ sở dữ liệu (bền vững)? Redis (phân tán)? Cho triển khai cụm, lưu trữ phiên phân tán là thiết yếu — không rõ Axelor hỗ trợ sẵn hay cần cấu hình tùy chỉnh.

**5. Quy tắc ưu tiên quyền khi xung đột** [Không rõ]
Khi Nhóm cấp canRead nhưng Vai trò từ chối (giả thuyết), bên nào thắng? Không quan sát thấy quy tắc từ chối tường minh, gợi ý hợp nhất thuần dựa trên cấp phép (không có xung đột). Nhưng nếu phiên bản tương lai thêm quy tắc từ chối, thứ tự ưu tiên chưa được định nghĩa trong mã.

**6. Xác thực hai yếu tố (2FA)** [Không tìm thấy]
Không có cấu hình hoặc mã cho 2FA, TOTP, xác minh SMS, khóa bảo mật. Thực hành bảo mật hiện đại đặc biệt cho tài khoản quản trị, nhưng phải triển khai dưới dạng phần mở rộng tùy chỉnh nếu cần.

**7. Xác thực API (JWT, khóa API)** [Không rõ]
Cơ chế xác thực API REST không rõ ràng. Xác thực cơ bản (Basic Auth) được đề cập nhưng không có sinh mã thông báo JWT, quản lý khóa API, luồng thông tin khách OAuth 2.0. Máy khách API có thể cần cookie phiên (không lý tưởng cho truy cập lập trình).

**8. Kế thừa hoặc phân cấp quyền** [Không rõ]
Nhóm có thể có nhóm cha (phân cấp tổ chức) không? Vai trò có thể kế thừa từ vai trò khác (phân cấp vai trò) không? Không có bằng chứng trong mã — có vẻ là cấu trúc phẳng. Tổ chức doanh nghiệp thường cần quyền phân cấp (ví dụ: vai trò Quản lý kế thừa mọi quyền của vai trò Nhân viên cộng thêm quyền bổ sung).

**9. Nhật ký kiểm toán cho sự kiện bảo mật** [Không đầy đủ]
Hệ thống theo dõi ghi nhật ký thay đổi thực thể nhưng không rõ có ghi: nỗ lực đăng nhập (thành công/thất bại), sự kiện đăng xuất, kiểm tra quyền (nỗ lực truy cập bị từ chối), tạo/hủy phiên, thay đổi mật khẩu hay không. Kiểm toán bảo mật và ứng phó sự cố đòi hỏi ghi nhật ký sự kiện bảo mật toàn diện.

**10. Quản lý mã thông báo OAuth/OIDC** [Không rõ]
Tích hợp Pac4j xử lý xác thực OAuth nhưng không rõ: mã thông báo truy cập/làm mới (access/refresh token) lưu ở đâu? Mã thông báo làm mới được xoay vòng như thế nào? Hỗ trợ thu hồi mã thông báo? Truy cập ngoại tuyến (mã thông báo làm mới với thời hạn kéo dài)? Quan trọng cho triển khai OAuth trong sản xuất.

---

### 17. CÂU HỎI MỞ VÀ ĐIỂM CẦN NGHIÊN CỨU THÊM

Phân tích hệ thống bảo mật đặt ra nhiều câu hỏi đòi hỏi nghiên cứu sâu hơn hoặc rà soát tài liệu:

**1. Hiệu năng giải quyết quyền:** Tỉ lệ trúng bộ đệm trong sản xuất là bao nhiêu? Mỗi yêu cầu có bao nhiêu truy vấn cơ sở dữ liệu thuộc về kiểm tra quyền? Có vấn đề N+1 truy vấn khi tải quyền cho nhiều đối tượng không? Cần đo hiệu năng thực tế.

**2. Cập nhật quyền động:** Khi quản trị viên sửa quyền, thay đổi có hiệu lực khi nào? Ngay lập tức cho mọi người dùng? Chỉ sau đăng xuất/đăng nhập? Bộ khung có phát sự kiện vô hiệu trong triển khai cụm không? Cập nhật quyền thời gian thực rất quan trọng cho sự cố bảo mật (thu hồi truy cập ngay cho tài khoản bị xâm phạm).

**3. Chi tiết ngôn ngữ biểu thức điều kiện:** Các trường `readonlyIf`, `hideIf` dùng ngôn ngữ biểu thức gì chính xác? Groovy? JavaScript? DSL tùy chỉnh? Biến/hàm nào khả dụng trong ngữ cảnh biểu thức? Biểu thức được biên dịch hay thông dịch?

**4. Truy vấn xuyên thuê bao và chia sẻ dữ liệu:** Trong chế độ đa thuê bao, siêu quản trị có thể truy vấn xuyên mọi thuê bao không? Thuê bao có thể chia sẻ dữ liệu nhất định (danh mục sản phẩm chung) trong khi giữ dữ liệu giao dịch cách ly không?

**5. Tích hợp phân quyền bên ngoài:** Axelor có thể tích hợp với dịch vụ phân quyền bên ngoài (AWS IAM, truy cập có điều kiện Azure AD, OPA) không?

**6. Bảo mật API ngoài xác thực cơ bản:** Cho máy khách API REST, phương thức xác thực nào được khuyến nghị? Có giới hạn tốc độ (rate limit) không? Làm sao triển khai khóa API cho tài khoản dịch vụ?

**7. Quyền mặc định cho thực thể mới:** Khi mô-đun cài đặt với thực thể mới, quyền mặc định nào được áp dụng? Mọi thực thể bị từ chối mặc định (an toàn) hay cấp cho tất cả người dùng (tiện lợi)?

**8. Tiện ích kiểm thử quyền:** Có công cụ kiểm thử cấu hình quyền không? Mô phỏng người dùng với nhóm/vai trò cụ thể và xác minh họ có/không thể truy cập đối tượng/trường nhất định?

**9. Di chuyển quyền và kiểm soát phiên bản:** Khi thực thể đổi tên hoặc gói bị tái cấu trúc, làm sao di chuyển quyền? Có kịch bản hỗ trợ không? Nhập/xuất CSV giúp ích nhưng không tự động hóa thay đổi cấu trúc.

**10. Liên kết và SSO xuyên ứng dụng:** Nhiều phiên bản Axelor có thể chia sẻ xác thực (SSO) không? Axelor có thể tham gia liên kết SSO doanh nghiệp với ứng dụng ngoài Axelor không? SAML/OAuth cho phép điều này về lý thuyết, nhưng thiết lập thực tế chưa rõ.

Những câu hỏi này đại diện cho các lĩnh vực cần kiểm thử thực tế, rà soát tài liệu bộ khung, hoặc trao đổi trực tiếp với cộng đồng Axelor để làm rõ.

---

## TÓM TẮT KIẾN TRÚC BẢO MẬT

Sau quá trình phân tích chi tiết từ mã nguồn, kiến trúc bảo mật của Axelor có thể tóm lược qua các điểm sau:

### Kiến trúc bảo mật nhiều tầng

Axelor triển khai **bảo mật đa tầng toàn diện** trải dài từ xác thực, phân quyền (cấp đối tượng/trường/bản ghi), đến nhật ký kiểm toán. Kiến trúc không dựa vào bộ khung tiêu chuẩn (Spring Security/Shiro) mà xây dựng giải pháp tùy chỉnh tích hợp chặt với lõi bộ khung. Quyết định này mang lại tích hợp sâu với cách tiếp cận hướng mô hình của Axelor (quyền định nghĩa trong XML, quản lý qua CSV) nhưng hy sinh tính tương thích hệ sinh thái (công cụ bảo mật bên thứ ba thiết kế cho Spring Security không hoạt động trực tiếp).

### Xác thực: Hỗ trợ đa nhà cung cấp Pac4j

Tầng xác thực được cung cấp bởi **Pac4j 5.7.7**, hỗ trợ sáu nhà cung cấp xác thực: nội bộ (tên đăng nhập/mật khẩu), Google OAuth, Keycloak, SAML 2.0, LDAP, và CAS. Kiến trúc không phụ thuộc nhà cung cấp cho phép kết hợp (nhân viên doanh nghiệp qua LDAP, đối tác qua Google OAuth) trong cùng triển khai. Cấp phát tài khoản người dùng (tự tạo, liên kết, hoặc từ chối người dùng chưa biết) có thể cấu hình theo yêu cầu bảo mật của từng triển khai.

### Phân quyền: Phân cấp ba tầng

Mô hình phân quyền tuân theo phân cấp **Người dùng → Nhóm/Vai trò → Quyền hạn**. Người dùng thuộc một Nhóm (đơn vị tổ chức) và nhiều Vai trò (năng lực chức năng). Quyền tổng hợp từ cả Nhóm và mọi Vai trò bằng hợp nhất dựa trên cấp phép (bất kỳ nguồn quyền nào cấp = cho phép). Hệ thống kép Nhóm/Vai trò tách biệt ngữ cảnh tổ chức (nhóm) khỏi quyền chức năng (vai trò), cho phép gán quyền linh hoạt mà không cần bảo trì ma trận phức tạp.

### Quyền cấp đối tượng: CRUD + Xuất

Thực thể Permission kiểm soát truy cập vào toàn bộ lớp thực thể (SaleOrder, Invoice, Product) với năm thao tác: canRead, canWrite, canCreate, canRemove, canExport. Hỗ trợ ký tự đại diện cho phép quyền cấp gói (`com.axelor.apps.sale.*`) giảm gánh nặng cấu hình. Kiểm tra đối chiếu siêu mô hình JPA ngăn quyền mồ côi.

### Bảo mật cấp bản ghi: Bộ lọc miền

Tính năng phân quyền tinh vi nhất: **quyền có điều kiện** thông qua mệnh đề WHERE dạng SQL được chèn vào truy vấn. Biến đặc biệt (`__user__.{trường}`) cho phép lọc động dựa trên thuộc tính người dùng hiện tại. Ví dụ: `self.company = ?` + `__user__.activeCompany` triển khai cách ly đa công ty; `self.assignedTeam in (?)` + `__user__.teamSet` cho phép hiển thị theo nhóm làm việc. Điều kiện được đánh giá ở tầng cơ sở dữ liệu (không phải ứng dụng), đảm bảo cả bảo mật lẫn hiệu năng.

### Quyền cấp trường: Kiểm soát chi tiết

Thực thể MetaPermission và MetaPermissionRule cung cấp phân quyền cấp trường: ẩn trường nhạy cảm (costPrice) khỏi nhóm nhất định, đặt trường chỉ đọc dựa trên trạng thái quy trình (`readonlyIf="statusSelect > 2"`). Biểu thức có điều kiện (`hideIf`, `readonlyIf`) cho phép giao diện nhạy ngữ cảnh — cùng trường hiển thị với một số người dùng, ẩn với người khác, chỉ đọc dựa trên trạng thái bản ghi.

### Quản lý quyền dựa trên CSV

Công cụ Trợ lý quyền hạn cho phép cấu hình quyền hàng loạt qua nhập/xuất CSV. Quản trị viên tận dụng công cụ bảng tính (công thức Excel, bảng tổng hợp, định dạng có điều kiện) để cấu hình nhanh hàng trăm quyền. Nhập theo giao dịch đảm bảo nhất quán. Cách tiếp cận CSV hiệu quả hơn đáng kể so với cấu hình quyền từng cái qua giao diện cho ma trận quyền lớn.

### Tối ưu hiệu năng: Đệm nhiều cấp

Thực thể quyền được đánh dấu `cacheable="true"` tận dụng bộ đệm cấp hai Hibernate. Tổng hợp quyền cấp phiên (tải một lần khi đăng nhập, đệm suốt phiên) ngăn truy vấn cơ sở dữ liệu lặp lại. Đệm rất quan trọng vì kiểm tra quyền xảy ra trên mọi thao tác — không có đệm, hệ thống sẽ bị nghẽn cơ sở dữ liệu.

### Tùy chọn gia cố bảo mật

Biểu thức chính quy chính sách mật khẩu áp dụng mật khẩu mạnh (8+ ký tự, 3 trong 4 loại). Bảo vệ chống tiêm SQL chặn biểu thức nguy hiểm trong điều kiện quyền. Hệ thống quyền có thể tắt cho kiểm thử (nguy hiểm nếu để bật trong sản xuất).

### Thiếu sót và hạn chế

Các vắng mặt đáng chú ý: không có xác thực hai yếu tố; xác thực API không rõ (ngoài xác thực cơ bản); không có kế thừa/phân cấp quyền; ghi nhật ký sự kiện bảo mật tối thiểu; cấu hình đa thuê bao tồn tại nhưng triển khai không rõ. Thuật toán băm mật khẩu và cơ chế lưu trữ phiên không có tài liệu trong mã tiếp cận được. Quy tắc ưu tiên quyền trong kịch bản xung đột chưa được định nghĩa.

### Triết lý kiến trúc

Axelor ưu tiên **khả năng cấu hình hơn bảo mật lập trình** — quyền được định nghĩa trong XML/CSV thay vì chú thích, cho phép người dùng nghiệp vụ cấu hình bảo mật mà không cần thay đổi mã. Đánh đổi: kém an toàn khi biên dịch (lỗi chính tả trong điều kiện quyền chỉ bị bắt khi chạy), linh hoạt hơn trong vận hành. Phù hợp cho môi trường nơi yêu cầu bảo mật thay đổi thường xuyên và người không phải lập trình viên cần quản lý quyền hạn.

---

**Tổng số dòng mã/cấu hình đã phân tích:**
- Thực thể miền: 4 tệp (User, Group, Role, Permission)
- Mã dịch vụ: 2 tệp (PermissionServiceImpl, PermissionAssistantService)
- Cấu hình: axelor-config.properties (phần xác thực)

**Nguồn:** Tất cả phát hiện từ phân tích mã nguồn trực tiếp, bổ sung bằng suy luận dựa trên mẫu bảo mật tiêu chuẩn, hành vi JPA/Hibernate, và thực hành tốt nhất ứng dụng doanh nghiệp.

---

*Kết thúc RESEARCH_STEP3_SECURITY.md*

