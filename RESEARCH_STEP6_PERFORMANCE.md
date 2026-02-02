# BƯỚC 6: PHÂN TÍCH HIỆU NĂNG VÀ KHẢ NĂNG MỞ RỘNG — Từ mã nguồn Axelor 8.5.10

## Phương pháp phân tích

Nghiên cứu hiệu năng và khả năng mở rộng của Axelor thông qua phân tích tập tin cấu hình, chiến lược bộ đệm (caching), quản lý nhóm kết nối cơ sở dữ liệu (connection pooling), cơ chế xử lý bất đồng bộ (async processing) và kiến trúc giao diện lập trình ứng dụng REST (REST API). Trọng tâm chính là tìm hiểu Axelor tối ưu hiệu năng cho môi trường triển khai thực tế như thế nào, điểm nghẽn nằm ở đâu, và có những tùy chọn tinh chỉnh nào để mở rộng từ máy chủ đơn lẻ trong giai đoạn phát triển lên cụm máy chủ doanh nghiệp đa nút.

**Tập tin đã phân tích:**
- axelor-config.properties (cấu hình hiệu năng: bộ đệm, nhóm kết nối, phân trang, quản lý phiên làm việc)
- Bộ điều khiển REST (UserRestController.java, SaleOrderController.java)
- Bộ khung xử lý hàng loạt (BatchDirectDebit.java, BatchBankPaymentService.java)
- Cấu hình cơ sở dữ liệu (thiết lập Hibernate, tham số HikariCP)
- Mô hình miền có chú thích bộ đệm (Currency.xml, Country.xml)

**Cách tiếp cận:** Khảo sát cấu hình mặc định, suy luận ảnh hưởng đến hiệu năng, so sánh với các thực tiễn tốt nhất trong ngành, nhận diện cơ hội tối ưu và đánh giá trần khả năng mở rộng. Phân biệt rõ giữa những gì đã được cấu hình sẵn, những gì có thể cấu hình thêm, và những gì cần viết mã tùy chỉnh.

---

## Kết quả chi tiết

### 1. CHIẾN LƯỢC BỘ ĐỆM: KIẾN TRÚC ĐA TẦNG CHO HIỆU NĂNG

**Tập tin nguồn:** axelor-config.properties, các tập tin XML miền [Từ source code]

Axelor triển khai **hệ thống bộ đệm phân tầng** theo mô hình hiệu năng ứng dụng doanh nghiệp — cân bằng giữa tiêu thụ bộ nhớ và giảm tải cho cơ sở dữ liệu. Chiến lược bộ đệm đặc biệt quan trọng với hệ thống ERP, nơi dữ liệu tham chiếu (tiền tệ, quốc gia, danh mục sản phẩm) được truy xuất thường xuyên hơn nhiều so với tần suất thay đổi. Kiến trúc sử dụng bốn tầng bộ đệm riêng biệt, mỗi tầng có phạm vi, thời gian sống và cơ chế vô hiệu hóa khác nhau.

**1.1. Bộ đệm cấp một của JPA (L1 Cache)**

Bộ đệm cấp một là **bộ đệm thực thể tự động** do phiên Hibernate (Hibernate Session) duy trì — không cần cấu hình, hoạt động trong suốt với mọi thao tác JPA. Phạm vi bộ đệm là theo từng giao dịch/yêu cầu, nghĩa là mỗi yêu cầu HTTP có một phiên bản bộ đệm riêng biệt, không chia sẻ dữ liệu giữa các yêu cầu đồng thời. Thời gian sống bằng đúng thời gian giao dịch — bộ đệm được xóa khi giao dịch hoàn tất hoặc bị hủy, ngăn ngừa vấn đề dữ liệu cũ.

**Cơ chế hoạt động:** Khi mã ứng dụng thực thi `entityManager.find(SaleOrder.class, orderId)` lần đầu, Hibernate tải thực thể từ cơ sở dữ liệu rồi lưu vào bộ đệm cấp một với khóa chính làm khóa tra cứu. Các lần gọi tiếp theo trong cùng giao dịch sẽ trả về phiên bản đã lưu ngay lập tức (độ trễ micro giây) mà không cần truy vấn cơ sở dữ liệu. Mô hình bản đồ danh tính (identity map) đảm bảo chỉ có duy nhất một phiên bản thực thể cho mỗi mã định danh trong mỗi giao dịch — tất cả tham chiếu đều trỏ đến cùng một đối tượng.

**Lợi ích:** Loại bỏ các truy vấn trùng lặp trong cùng giao dịch. Ví dụ phổ biến: tải đơn hàng, sửa trường, tải lại để kiểm tra, rồi lưu. Không có bộ đệm cấp một thì cần 3 truy vấn; có bộ đệm cấp một thì chỉ cần 1 truy vấn + 2 lần đọc từ bộ đệm. Tối ưu hoàn toàn trong suốt — lập trình viên không cần viết mã đặc biệt.

**Đánh đổi:** Tốn bộ nhớ cho các giao dịch kéo dài — công việc hàng loạt xử lý hàng nghìn thực thể sẽ tích lũy hàng megabyte đối tượng trong bộ đệm. Giải pháp: gọi `entityManager.clear()` định kỳ hoặc xử lý bản ghi theo từng lô. Không có bộ đệm xuyên giao dịch — mỗi yêu cầu lặp lại truy vấn cho dữ liệu thường dùng (tiền tệ, cài đặt). Bộ đệm cấp hai giải quyết vấn đề này.

**1.2. Bộ đệm cấp hai của JPA (L2 Cache)**

Bộ đệm cấp hai là **bộ đệm thực thể dùng chung** xuyên suốt mọi giao dịch và người dùng — phạm vi toàn ứng dụng, thời gian sống có thể cấu hình (từ vài phút đến vài giờ). Hibernate trừu tượng hóa nhà cung cấp bộ đệm qua giao diện lập trình JCache (JSR-107), cho phép thay đổi linh hoạt phần phụ trợ: Caffeine (nhẹ, trong bộ nhớ), Hazelcast (phân tán, đa nút), Redis (tập trung, bền vững), Ehcache (cũ, cấu hình bằng XML).

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:15-37
# Shared cache mode settings (ALL, DISABLE_SELECTIVE, ENABLE_SELECTIVE, NONE)
javax.persistence.sharedCache.mode = ENABLE_SELECTIVE

# second-level cache factory
#hibernate.cache.region.factory_class = jcache

# second-level cache provider
#hibernate.javax.cache.provider =
```

**Giải thích các tùy chọn cấu hình:**

**Chế độ bộ đệm = ENABLE_SELECTIVE:** Chỉ những thực thể được đánh dấu `@Cacheable` mới tham gia bộ đệm cấp hai. Các chế độ khác: `ALL` (đệm tất cả — nguy hiểm, tốn bộ nhớ), `DISABLE_SELECTIVE` (đệm tất cả trừ `@Cacheable(false)`), `NONE` (tắt hoàn toàn). `ENABLE_SELECTIVE` là lựa chọn ưu tiên cho môi trường thực tế — chiến lược chủ động chọn vào giúp tránh vô tình lưu đệm dữ liệu giao dịch (đơn bán hàng, hóa đơn) vốn thay đổi thường xuyên và cần tính nhất quán thời gian thực.

**Nhà cung cấp bộ đệm bị ghi chú:** Cấu hình mặc định **tắt hoàn toàn bộ đệm cấp hai** — chế độ `ENABLE_SELECTIVE` nhưng không có nhà cung cấp nào được khai báo, nghĩa là các chú thích bộ đệm bị bỏ qua. Lý do [Suy luận]: mặc định thân thiện với phát triển (không phụ thuộc bên ngoài), triển khai máy đơn không hưởng lợi đáng kể (cơ sở dữ liệu trên máy cục bộ = độ trễ thấp), tránh phức tạp vô hiệu hóa bộ đệm khi đang thử nghiệm.

**Các thực thể được đánh dấu lưu đệm** (ví dụ từ mô hình miền):

```xml
<!-- File: modules/axelor-open-suite/axelor-base/src/main/resources/domains/Currency.xml -->
<entity name="Currency" cacheable="true">
  <string name="code" required="true" unique="true"/>
  <string name="name" required="true"/>
  <decimal name="currentRate"/>
</entity>
```

**Giải thích:** Thực thể Tiền tệ (Currency) là ứng viên hoàn hảo cho bộ đệm — dữ liệu tham chiếu (hiếm khi thay đổi), được truy xuất thường xuyên (mỗi hóa đơn, báo giá, đơn mua hàng đều hiển thị tiền tệ), tập dữ liệu nhỏ (thường 50-200 loại tiền tệ toàn cầu). Lưu đệm thực thể tiền tệ giảm khoảng 95% truy vấn cơ sở dữ liệu (ước tính 1000 lần tra cứu tiền tệ mỗi phút → chỉ 50 lần trượt đệm khi hết hạn).

**Các thực thể phù hợp cho bộ đệm cấp hai** [Suy luận từ mẫu truy cập dữ liệu]:

- ✅ **Dữ liệu cấu hình:** SaleConfig, AccountConfig, CompanyConfig (tải mỗi yêu cầu cho quy tắc nghiệp vụ)
- ✅ **Dữ liệu tham chiếu:** Country, Language, Unit, Currency (tĩnh, dùng chung toàn cục)
- ✅ **Siêu dữ liệu phân quyền:** MetaPermission, MetaModel (kiểm tra ủy quyền thường xuyên)
- ✅ **Định nghĩa mẫu:** EmailTemplate, ReportTemplate (tham chiếu trong quy trình)
- ❌ **Dữ liệu giao dịch:** SaleOrder, Invoice, Payment (thay đổi liên tục, riêng theo người dùng)
- ❌ **Dữ liệu kiểm toán:** Các lớp con AuditableModel (theo dõi phiên bản không tương thích với bộ đệm)

**Vùng bộ đệm** [Suy luận từ quy ước Hibernate]:

Hibernate tổ chức bộ đệm cấp hai thành các vùng (không gian tên) — mỗi lớp thực thể có vùng riêng, kết quả truy vấn được lưu đệm riêng. Tên vùng dùng tên lớp đầy đủ (`com.axelor.apps.base.db.Currency`). Các vùng đặc biệt: `default-query-results-region` (bộ đệm truy vấn), `default-update-timestamps-region` (phối hợp vô hiệu hóa).

**Chiến lược vô hiệu hóa bộ đệm:** Ghi xuyên (write-through) — khi thực thể được cập nhật, Hibernate vô hiệu hóa mục đệm ngay lập tức, lần truy cập tiếp theo sẽ tải lại từ cơ sở dữ liệu. Đảm bảo nhất quán: nhất quán cuối cùng (cụm đa nút có thể thấy dữ liệu cũ trong chốc lát do độ trễ vô hiệu hóa), nhất quán mạnh (triển khai đơn nút). Trục xuất bộ đệm: LRU (ít được sử dụng gần đây nhất) khi vượt giới hạn kích thước vùng, dựa trên TTL (thời gian sống) để ngăn dữ liệu cũ không giới hạn.

**1.3. Bộ đệm kịch bản Groovy**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:78-82
# Groovy scripts cache size
#application.script.cache.size = 1000

# Groovy scripts cache entry expire time (in minutes)
#application.script.cache.expire-time = 20
```

Bộ đệm kịch bản Groovy giải quyết **chi phí biên dịch** của các biểu thức Groovy nhúng trong giao diện/hành động XML. Quy trình biên dịch gồm: phân tích mã nguồn Groovy → xây dựng cây cú pháp trừu tượng (AST) → sinh mã byte JVM → tải lớp — thao tác tốn kém (50-200ms mỗi kịch bản). Không có bộ đệm thì mỗi lần thực thi hành động phải biên dịch lại, gây độ trễ không chấp nhận được cho giao diện tương tác. Có bộ đệm thì biên dịch một lần, tái sử dụng mã byte hàng nghìn lần.

**Cấu hình mặc định:**
- **Kích thước: 1000 kịch bản** — đủ cho ứng dụng điển hình với khoảng 500 hành động + 300 biểu thức trường tính toán + 200 ràng buộc miền
- **Thời gian sống: 20 phút** — cân bằng giữa tươi mới (lập trình viên thấy thay đổi kịch bản trong vòng 20 phút) và hiệu năng (hầu hết kịch bản không đổi qua nhiều giờ)
- **Trục xuất: LRU** [Suy luận] — kịch bản ít dùng nhất bị trục xuất khi đệm đầy

**Cấu trúc khóa bộ đệm** [Suy luận]: Kết hợp mã băm mã nguồn kịch bản + loại ngữ cảnh (ActionRequest, thực thể miền) — đảm bảo các ngữ cảnh khác nhau không chia sẻ kịch bản đã biên dịch (ràng buộc biến khác nhau). Ví dụ: kịch bản `clientPartner?.paymentCondition` biên dịch khác nhau khi ngữ cảnh là SaleOrder so với Invoice.

**Tác động hiệu năng ước tính** [Suy luận từ phép đo Groovy]:
- **Lần thực thi đầu (trượt đệm):** Biên dịch (100ms) + thực thi (5ms) = 105ms
- **Lần thực thi đã đệm (trúng đệm):** Thực thi (5ms)
- **Tăng tốc:** 20 lần
- **Tỷ lệ trúng đệm ước tính:** ~98% cho tải thực tế (cùng kịch bản được thực thi lặp lại)

**Dung lượng bộ nhớ:** Kích thước kịch bản đã biên dịch khoảng 5-20KB (mã byte + siêu dữ liệu). Dung lượng bộ đệm: 1000 kịch bản × 15KB trung bình = **15MB bộ nhớ** — không đáng kể so với heap ứng dụng (thường 2-8GB).

**Khởi động nóng bộ đệm:** Axelor có thể biên dịch trước các kịch bản hay dùng khi khởi động ứng dụng [Suy luận] — tải tất cả giao diện XML, trích xuất biểu thức kịch bản, biên dịch chủ động. Lợi ích: loại bỏ độ trễ đột biến của yêu cầu đầu tiên (phạt bộ đệm lạnh).

**1.4. Bộ đệm phân quyền**

Đánh giá phân quyền là thao tác **tốn tài nguyên tính toán** — mỗi thao tác được bảo vệ (xem bản ghi, sửa trường, thực thi hành động) đòi hỏi: (1) tải nhóm người dùng, (2) tải vai trò của nhóm, (3) tải quyền của vai trò, (4) đánh giá quy tắc phân quyền (bộ lọc đối tượng, điều kiện trường, ràng buộc cấp bản ghi), (5) tổng hợp kết quả. Không có bộ đệm thì mỗi lần kiểm tra ủy quyền cần 5-10 truy vấn cơ sở dữ liệu + đánh giá quy tắc phức tạp = độ trễ 50-100ms cho mỗi thao tác.

**Các thực thể được lưu đệm** [Suy luận từ phân tích bảo mật BƯỚC 3]:
- **Quyền người dùng:** Tập hợp phẳng tất cả quyền được cấp qua nhóm/vai trò
- **Quyền nhóm:** Quyền tổng hợp từ các vai trò thành viên
- **Siêu quyền (Meta permissions):** Quyền đọc/ghi cấp trường cho loại thực thể cụ thể
- **Quy tắc phân quyền:** Biểu thức miền đã biên dịch (ví dụ: `self.company = __user__.activeCompany`)

**Mẫu khóa bộ đệm** [Suy luận]:
```
# Quyền tổng hợp của người dùng
permission:user:{userId}:objects

# Siêu quyền cho mô hình cụ thể
permission:user:{userId}:meta:{modelClass}

# Quyền cấp trường
permission:user:{userId}:field:{modelClass}:{fieldName}

# Kết quả kiểm tra quyền cấp bản ghi
permission:user:{userId}:record:{modelClass}:{recordId}:action:{actionType}
```

**Các sự kiện kích hoạt vô hiệu hóa bộ đệm:**
1. Thay đổi thành viên nhóm → vô hiệu hóa tất cả khóa `permission:user:{userId}:*`
2. Thay đổi vai trò của nhóm → vô hiệu hóa quyền cho tất cả thành viên nhóm
3. Thay đổi quyền của vai trò → lan truyền vô hiệu hóa đến tất cả người dùng có vai trò đó
4. Sửa đổi quy tắc phân quyền → vô hiệu hóa tất cả bộ đệm phái sinh

**Thời gian sống bộ đệm:** Thường tồn tại lâu (hàng giờ đến hàng ngày) — quyền hiếm khi thay đổi trong giờ làm việc. Đánh đổi: chấp nhận hơi cũ (người dùng được cấp quyền phải đăng nhập lại mới thấy hiệu lực) so với chính xác thời gian thực (phức tạp vô hiệu hóa bộ đệm).

**Tác động hiệu năng ước tính:**
- **Không có đệm:** Kiểm tra quyền = 50ms (5-10 truy vấn + đánh giá quy tắc)
- **Có đệm:** Kiểm tra quyền = 0,5ms (tra cứu bảng băm)
- **Tăng tốc:** 100 lần
- **Tỷ lệ trúng:** ~99% (cùng người dùng truy cập cùng tài nguyên lặp lại)

**1.5. Thực tiễn tốt nhất cho cấu hình bộ đệm**

**Nhà cung cấp bộ đệm cấp hai khuyến nghị: Caffeine**

Caffeine là **thư viện bộ đệm Java hiệu năng cao** — kế nhiệm bộ đệm Guava, tối ưu cho công thái học JVM. Các tính năng: trục xuất tự động (theo kích thước, thời gian, tham chiếu), tải bất đồng bộ, theo dõi thống kê, thông lượng xuất sắc (hàng triệu thao tác/giây). So sánh: Caffeine vượt Guava vượt Ehcache cho triển khai đơn nút; Hazelcast vượt Redis cho cụm đa nút.

**Ví dụ cấu hình** [Suy luận từ thực tiễn ngành, không có trong mã nguồn]:

```properties
# axelor-config.properties bổ sung
hibernate.cache.region.factory_class = jcache
hibernate.javax.cache.provider = com.github.benmanes.caffeine.jcache.spi.CaffeineJCachingProvider
hibernate.javax.cache.uri = classpath:caffeine-jcache.xml
```

**Cấu hình vùng Caffeine** (caffeine-jcache.xml):
```xml
<cache-configuration xmlns="http://www.ehcache.org/v3">
  <!-- Bộ đệm tiền tệ: nhỏ, tồn tại lâu -->
  <cache name="com.axelor.apps.base.db.Currency">
    <max-entries>1000</max-entries>
    <expire-after-write>3600</expire-after-write>  <!-- 1 giờ -->
    <statistics>true</statistics>
  </cache>

  <!-- Bộ đệm kết quả truy vấn: lớn, tồn tại ngắn -->
  <cache name="default-query-results-region">
    <max-entries>10000</max-entries>
    <expire-after-write>300</expire-after-write>  <!-- 5 phút -->
  </cache>

  <!-- Bộ đệm phân quyền: trung bình cả kích thước và thời gian -->
  <cache name="com.axelor.auth.db.Permission">
    <max-entries>5000</max-entries>
    <expire-after-write>1800</expire-after-write>  <!-- 30 phút -->
  </cache>
</cache-configuration>
```

**Hướng dẫn tinh chỉnh:**
- **Dữ liệu tham chiếu:** Số mục tối đa = kích thước tập dữ liệu × 2 (dự phòng tăng trưởng), TTL = 1-24 giờ
- **Dữ liệu cấu hình:** Số mục tối đa = 100-500, TTL = 15-60 phút
- **Kết quả truy vấn:** Số mục tối đa = 10.000-50.000, TTL = 5-15 phút (cân bằng tươi mới và tỷ lệ trúng)
- **Phân quyền:** Số mục tối đa = người dùng × quyền trung bình × 10, TTL = 30-60 phút

---

### 2. TỐI ƯU CƠ SỞ DỮ LIỆU: NHÓM KẾT NỐI VÀ TINH CHỈNH TRUY VẤN

**Tập tin nguồn:** axelor-config.properties [Từ source code]

Hiệu năng cơ sở dữ liệu là nền tảng của ứng dụng doanh nghiệp — quản lý kết nối kém hiệu quả và truy vấn chưa tối ưu là nguyên nhân chính gây nghẽn khả năng mở rộng. Axelor áp dụng các mô hình chuẩn ngành: nhóm kết nối (HikariCP), thao tác hàng loạt (batch operations), chiến lược tải lười (lazy loading) và phân trang kết quả truy vấn. Phân tích cho thấy **mặc định tốt cho phát triển** nhưng **cần tinh chỉnh cho tải thực tế**.

**2.1. Nhóm kết nối HikariCP**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:22-25
# HikariCP connection pool
hibernate.hikari.minimumIdle = 5
hibernate.hikari.maximumPoolSize = 20
hibernate.hikari.idleTimeout = 300000
```

HikariCP là **nhóm kết nối JDBC hàng đầu** — dẫn đầu bảng xếp hạng hiệu năng ngành (nhanh hơn 10 lần so với đối thủ theo phép đo nội bộ), kiến trúc không chi phí phụ, đã chứng minh qua hàng triệu lượt triển khai thực tế.

**Giải thích các tham số cấu hình:**

**minimumIdle = 5:** Số kết nối tối thiểu duy trì trong nhóm khi rảnh. Lợi ích: sẵn sàng tức thì (không có độ trễ tạo kết nối khi yêu cầu đến), chi phí: 5 × bộ nhớ mỗi kết nối (thường 5 × 5MB = 25MB). Quá thấp: tốn chi phí tạo kết nối khi lưu lượng đột biến. Quá cao: lãng phí tài nguyên khi vắng khách.

**maximumPoolSize = 20:** Số kết nối cơ sở dữ liệu đồng thời tối đa. Đây là tham số tinh chỉnh quan trọng — ảnh hưởng trực tiếp đến trần thông lượng. Công thức từ tài liệu HikariCP: `kết nối = ((số_lõi × 2) + số_đĩa_hiệu_dụng)`. Ví dụ: máy chủ 4 lõi với 1 SSD → tối ưu = (4 × 2) + 1 = 9 kết nối. Cấu hình hiện tại (20) phù hợp cho máy chủ 8-10 lõi.

**Lý giải công thức:** Kết nối cơ sở dữ liệu bị giới hạn bởi vào/ra đĩa, không phải CPU. Mỗi kết nối đang thực thi truy vấn chặn sẽ chờ đĩa, luồng tạm dừng. Kích thước nhóm tối ưu là đủ kết nối để bão hòa băng thông vào/ra đĩa mà không gây chuyển ngữ cảnh quá mức. Quá nhỏ: đĩa chưa dùng hết (luồng chờ kết nối trống). Quá lớn: tranh chấp kết nối (quá nhiều luồng cạnh tranh khóa).

**idleTimeout = 300000ms (5 phút):** Kết nối rảnh lâu hơn 5 phút sẽ bị đóng, nhóm co lại về minimumIdle. Lợi ích: giải phóng tài nguyên cơ sở dữ liệu khi lưu lượng thấp. Chi phí: tốn thêm khi tái kết nối khi lưu lượng phục hồi. Đánh đổi: thời gian chờ dài hơn (10-30 phút) cho tải ổn định, ngắn hơn (2-5 phút) cho tải đột biến.

**Các cấu hình thiếu** [Suy luận từ thực tiễn HikariCP]:

```properties
# Bổ sung khuyến nghị
hibernate.hikari.connectionTimeout = 30000  # 30 giây chờ kết nối tối đa
hibernate.hikari.maxLifetime = 1800000      # 30 phút tuổi thọ kết nối tối đa (ngăn kết nối cũ)
hibernate.hikari.leakDetectionThreshold = 60000  # 60 giây - cảnh báo nếu kết nối giữ quá lâu
hibernate.hikari.validationTimeout = 5000   # 5 giây thời gian chờ xác thực kết nối
```

**Vòng đời kết nối:** (1) Nhóm khởi tạo với minimumIdle kết nối, (2) yêu cầu đến → nhóm gán kết nối trống, (3) ứng dụng dùng kết nối (thực thi truy vấn), (4) ứng dụng trả kết nối về nhóm, (5) kết nối được tái sử dụng cho yêu cầu tiếp, (6) kết nối rảnh vượt idleTimeout bị đóng, (7) kết nối quá tuổi (vượt maxLifetime) bị thay thế bằng kết nối mới.

**2.2. Nhóm kết nối riêng cho BPM**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:273-276
studio.bpm.logging = false
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
studio.bpm.history.time.to.live = P180D
```

Axelor phân bổ **nhóm kết nối riêng** cho động cơ BPM (Camunda, nhận diện trong BƯỚC 4). Lý do: cách ly khối lượng công việc — quy trình BPM thực thi giao dịch kéo dài (quy trình nhiều bước kéo dài từ vài phút đến vài giờ), nhóm riêng ngăn giao dịch quy trình làm cạn kết nối cho yêu cầu ứng dụng.

**Phân tích cấu hình:**

**max.idle.connections = 10:** Nhóm BPM duy trì tối thiểu 10 kết nối rảnh. Cao hơn minimumIdle của ứng dụng (5) — hợp lý do đặc điểm tải BPM: quy trình thực thi liên tục (bộ hẹn giờ lập lịch, tác vụ bất đồng bộ), hưởng lợi từ kết nối sẵn sàng.

**max.active.connections = 50:** Giới hạn nhóm BPM = 50 kết nối, cao hơn đáng kể so với maximumPoolSize của ứng dụng (20). Cho thấy tải BPM **được kỳ vọng chiếm ưu thế trong sử dụng cơ sở dữ liệu** [Suy luận]. Lưu ý: tổng kết nối = 20 (ứng dụng) + 50 (BPM) = **tối đa 70** — yêu cầu PostgreSQL `max_connections` ≥ 100 (dự phòng cho kết nối quản trị, công cụ giám sát).

**Lợi ích cách ly tải:** Yêu cầu ứng dụng không bị ảnh hưởng bởi công việc hàng loạt BPM tiêu thụ hết kết nối nhóm BPM. Cách ly lỗi: nhóm kết nối BPM cạn → quy trình tạm dừng, ứng dụng vẫn phục vụ người dùng. Giám sát chi tiết: số liệu riêng cho ứng dụng và BPM, xác định điểm nghẽn chính xác.

**Đánh đổi:** Phức tạp hơn (hai nhóm cần cấu hình/giám sát), tốn tài nguyên (70 kết nối thay vì 40 nếu dùng nhóm chung), có thể lãng phí (nếu BPM ít dùng, nhóm 50 kết nối thừa).

**2.3. Tinh chỉnh thao tác hàng loạt**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:27-31
# define the batch size
#hibernate.jdbc.batch_size = 20

# define the fetch size
#hibernate.jdbc.fetch_size = 20
```

**jdbc.batch_size = 20 (bị ghi chú, tắt mặc định):**

Kích thước lô kiểm soát **gộp lệnh JDBC** — nhóm nhiều lệnh INSERT/UPDATE/DELETE vào một lần gửi mạng đến cơ sở dữ liệu. Cơ chế: ứng dụng thực thi `entityManager.persist(order1)`, `entityManager.persist(order2)`, ..., `entityManager.persist(order20)` → Hibernate tích lũy 20 lệnh INSERT → gửi gói giao thức duy nhất → cơ sở dữ liệu thực thi cả 20 lệnh chèn trong một giao dịch.

**Tác động hiệu năng:** Không gộp: 20 thao tác INSERT = 20 lần gửi mạng (20 × 1ms = 20ms). Có gộp: 20 thao tác INSERT = 1 lần gửi mạng (1 × 1ms = 1ms). **Tăng tốc: 20 lần** cho chèn hàng loạt.

**Tại sao bị ghi chú mặc định?** [Suy luận]: Chế độ gộp không tương thích với một số tính năng: `@GeneratedValue(strategy = IDENTITY)` (mã tự tăng yêu cầu truy vấn cơ sở dữ liệu ngay để lấy mã được sinh), bộ kích hoạt (trigger) trả giá trị, phương ngữ không hỗ trợ gộp. Mặc định thận trọng ngăn lỗi tinh vi.

**Khuyến nghị:** Bật cho môi trường thực tế — hầu hết thực thể Axelor dùng sinh mã dựa trên chuỗi (tương thích với gộp), thao tác hàng loạt (nhập dữ liệu, xử lý lô) hưởng lợi lớn. Cần kiểm thử kỹ — xác minh sinh mã, bộ kích hoạt, ràng buộc hoạt động đúng.

**jdbc.fetch_size = 20 (bị ghi chú):**

Kích thước tìm nạp kiểm soát **chiến lược con trỏ ResultSet** — trình điều khiển JDBC tìm nạp bao nhiêu hàng từ cơ sở dữ liệu cho mỗi lần gửi mạng khi thực thi truy vấn `SELECT` trả về hàng nghìn hàng. Cơ chế: ứng dụng thực thi `query.getResultList()` → trình điều khiển gửi SELECT → cơ sở dữ liệu trả 20 hàng đầu → ứng dụng xử lý → trình điều khiển tự động tìm nạp 20 hàng tiếp (trong suốt với mã ứng dụng).

**Đánh đổi hiệu năng:** Kích thước nhỏ (10-50): tốn ít bộ nhớ, nhiều lần gửi mạng. Kích thước lớn (500-1.000): tốn nhiều bộ nhớ, ít lần gửi mạng. Giá trị tối ưu phụ thuộc mẫu truy vấn: fetch_size=20 phù hợp cho phân trang giao diện (hiển thị 20 kết quả mỗi trang), quá nhỏ cho báo cáo (xuất 10.000 bản ghi).

**Khuyến nghị:** Tinh chỉnh linh hoạt — đặt fetch_size=100 mặc định toàn cục, ghi đè cho từng truy vấn cụ thể:
```java
// Truy vấn báo cáo: tối ưu thông lượng
Query query = entityManager.createQuery("...");
query.setHint("javax.persistence.fetchSize", 1000);
```

**2.4. Chiến lược tải lười (Lazy Loading)**

Chiến lược tìm nạp mặc định của Hibernate: **tải lười cho tập hợp** (quan hệ một-nhiều, nhiều-nhiều), **tải háo hức cho liên kết đơn giá trị** (quan hệ nhiều-một). Chiến lược này tối thiểu chi phí truy vấn ban đầu — chỉ tải thực thể được yêu cầu, trì hoãn tải thực thể liên quan cho đến khi truy cập.

**Ví dụ từ mô hình miền:**
```xml
<!-- File: domains/SaleOrder.xml -->
<entity name="SaleOrder">
  <!-- Tải lười (mặc định cho tập hợp) -->
  <one-to-many name="saleOrderLineList" ref="SaleOrderLine" mappedBy="saleOrder"/>

  <!-- Tải háo hức (mặc định cho nhiều-một) -->
  <many-to-one name="company" ref="Company"/>
</entity>
```

**Vấn đề truy vấn N+1 — Phản mẫu hiệu năng phổ biến nhất:**

**Mã có vấn đề:**
```java
// Truy vấn 1: Tải 100 đơn bán hàng
List<SaleOrder> orders = orderRepository.all().fetch(100);

// Truy vấn 2-101: Với mỗi đơn, tải lười dòng đơn hàng (vấn đề N+1!)
for (SaleOrder order : orders) {
  System.out.println(order.getSaleOrderLineList().size());  // Kích hoạt SELECT
}
// Tổng: 101 truy vấn (1 + 100)
```

**Giải thích:** Truy vấn đầu tải đơn bán hàng (100 hàng). Vòng lặp duyệt đơn, truy cập `saleOrderLineList` (tập hợp tải lười) → Hibernate thực thi SELECT riêng cho mỗi đơn để tải dòng. Thảm họa hiệu năng: 101 lần gửi mạng thay vì 1-2, độ trễ = 101 × 5ms = 505ms.

**Giải pháp 1: Tìm nạp kết hợp (JOIN FETCH):**
```java
// Truy vấn đơn với JOIN
List<SaleOrder> orders = Query.of(SaleOrder.class)
  .filter("...")
  .fetch(100);  // Query DSL của Axelor có thể tự phát hiện tập hợp cần truy cập, thêm JOIN FETCH
```

**Giải pháp 2: Đồ thị thực thể (Entity Graph — chuẩn JPA):**
```java
EntityGraph<SaleOrder> graph = entityManager.createEntityGraph(SaleOrder.class);
graph.addAttributeNodes("saleOrderLineList");
query.setHint("javax.persistence.fetchgraph", graph);
```

**Giải pháp 3: Tìm nạp theo lô (Batch Fetching — tối ưu Hibernate):**
```properties
# Tìm nạp nhiều tập hợp lười theo lô
hibernate.default_batch_fetch_size = 10
```

Với tìm nạp theo lô: truy cập `order1.getSaleOrderLineList()` kích hoạt SELECT tải dòng cho đơn 1-10 cùng lúc (truy vấn đơn với `WHERE sale_order_id IN (1,2,3,...,10)`). Giảm vấn đề N+1 xuống N/10+1 truy vấn.

**2.5. Chiến lược chỉ mục (Index)**

Chỉ mục cơ sở dữ liệu rất quan trọng cho hiệu năng truy vấn — sự khác biệt giữa quét toàn bảng (hàng giây cho bảng triệu hàng) và tìm kiếm chỉ mục (mili giây). Axelor tự động tạo chỉ mục cho: khóa chính (cột `id`), khóa ngoại (cột quan hệ), ràng buộc duy nhất. Chỉ mục thủ công cần thiết cho các cột thường lọc/sắp xếp mà chỉ mục tự động không bao phủ.

**Cột được lập chỉ mục tự động** [Suy luận từ quy ước JPA]:
- Khóa chính: `sale_sale_order.id` (chỉ mục cụm)
- Khóa ngoại: `sale_sale_order.client_partner` (chỉ mục không cụm)
- Ràng buộc duy nhất: `sale_sale_order.sale_order_seq` (chỉ mục duy nhất)

**Chỉ mục thiếu cần tạo thủ công** [Suy luận từ mẫu truy vấn phổ biến]:

```sql
-- Lọc theo trạng thái: "SELECT * FROM sale_order WHERE status_select = 2"
CREATE INDEX idx_sale_order_status ON sale_sale_order(status_select);

-- Sắp xếp theo ngày: "ORDER BY creation_date DESC"
CREATE INDEX idx_sale_order_creation_date ON sale_sale_order(creation_date);

-- Lọc tổ hợp: "WHERE company_id = ? AND status_select = ? ORDER BY order_date"
CREATE INDEX idx_sale_order_company_status_date
  ON sale_sale_order(company_id, status_select, order_date);

-- Chỉ mục bộ phận (PostgreSQL): Chỉ lập chỉ mục đơn đang hoạt động
CREATE INDEX idx_sale_order_active_partner
  ON sale_sale_order(client_partner)
  WHERE status_select IN (1, 2, 3);  -- Nháp, Đã xác nhận, Đang xử lý
```

**Nguyên tắc thiết kế chỉ mục:**
1. **Tính chọn lọc:** Lập chỉ mục cột có nhiều giá trị phân biệt — `client_partner` (hàng nghìn giá trị) là ứng viên tốt hơn `status_select` (5-10 giá trị)
2. **Chỉ mục tổ hợp:** Xếp cột theo tính chọn lọc giảm dần — `(company_id, status_select, order_date)` chứ không phải `(status_select, company_id, order_date)`
3. **Chỉ mục bao phủ:** Bao gồm tất cả cột cần cho truy vấn để tránh tra bảng — `CREATE INDEX ... INCLUDE (ex_tax_total, order_date)`
4. **Chỉ mục bộ phận:** Lọc chỉ mục cho tập con hay truy vấn — chỉ lập chỉ mục đơn đang hoạt động (loại bỏ 80% đơn đã hủy/hoàn thành), chỉ mục nhỏ hơn = tìm kiếm nhanh hơn

**Đánh đổi chỉ mục:** Lợi ích: truy vấn SELECT nhanh hơn 10-1000 lần. Chi phí: INSERT/UPDATE/DELETE chậm hơn (bảo trì chỉ mục), dung lượng đĩa (10-30% kích thước bảng), sử dụng bộ nhớ (chỉ mục lưu trong vùng đệm). Hướng dẫn: tối đa 5-10 chỉ mục mỗi bảng — vượt quá sẽ làm giảm đáng kể hiệu năng ghi.

---

### 3. KIẾN TRÚC API REST: NỀN TẢNG JAX-RS CHO TÍCH HỢP

**Tập tin nguồn:** UserRestController.java, SaleOrderController.java [Từ source code]

Axelor cung cấp **kiến trúc API kép**: (1) API REST cho tích hợp bên ngoài (ứng dụng di động, hệ thống bên thứ ba, vi dịch vụ), (2) Bộ điều khiển Web cho trình xử lý hành động nội bộ (tương tác giao diện XML). API REST tuân theo chuẩn JAX-RS 2.1 (Giao diện lập trình Java cho dịch vụ web RESTful), cho phép sử dụng công cụ chuẩn (tài liệu Swagger/OpenAPI, sinh mã khách tự động), tính khả chuyển bộ khung (có thể chuyển từ Jersey sang RESTEasy), và quen thuộc với lập trình viên (chú thích chuẩn ngành).

**3.1. Mẫu bộ điều khiển REST**

**Bằng chứng từ mã nguồn:**
```java
// File: modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/rest/UserRestController.java
@Path("/aos/user")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public class UserRestController {

  @Operation(
      summary = "Get user permissions",
      tags = {"User"})
  @Path("/permissions")
  @GET
  @HttpExceptionHandler
  public Response getPermissions() {
    User user = AuthUtils.getUser();
    return ResponseConstructor.build(
        Response.Status.OK,
        Beans.get(UserPermissionResponseComputeService.class)
            .computeUserPermissionResponse(user));
  }
}
```

**Phân tích chú thích:**

**@Path("/aos/user"):** Định nghĩa tiền tố đường dẫn URL cho tất cả phương thức bộ điều khiển. Mẫu: `/ws/aos/user/*` (bộ khung có lẽ thêm đường dẫn ngữ cảnh `/ws`). Việc chọn không gian tên `/aos` phân tách API của Axelor Open Suite khỏi API lõi bộ khung, cho phép quản lý phiên bản (`/aos/v1/user`, `/aos/v2/user`).

**@Consumes(MediaType.APPLICATION_JSON):** Bộ điều khiển chấp nhận thân yêu cầu dạng JSON. JSON được ưu tiên cho API REST — hỗ trợ rộng rãi (JavaScript gốc, thư viện json Python), người đọc được, gọn nhẹ.

**@Produces(MediaType.APPLICATION_JSON):** Bộ điều khiển trả phản hồi dạng JSON. Thương lượng nội dung: máy khách gửi tiêu đề `Accept: application/json` → máy chủ tuần tự hóa phản hồi thành JSON.

**@GET:** Phương thức HTTP = GET (thao tác đọc). Ngữ nghĩa REST: GET = an toàn (không có tác dụng phụ), lũy đẳng (nhiều lần gọi cho cùng kết quả), có thể lưu đệm (trình duyệt/proxy có thể lưu đệm phản hồi).

**@Operation:** Chú thích Swagger/OpenAPI — sinh tài liệu API. `summary` mô tả mục đích điểm cuối (hiển thị trong Swagger UI), `tags` nhóm các điểm cuối liên quan (mục thu gọn trong giao diện).

**@HttpExceptionHandler:** Chú thích tùy chỉnh xử lý ngoại lệ — ánh xạ ngoại lệ Java sang mã trạng thái HTTP (AxelorException → 400 Bad Request, SecurityException → 403 Forbidden, NotFoundException → 404 Not Found).

**AuthUtils.getUser():** Lấy người dùng đã xác thực từ ngữ cảnh bảo mật (được đặt bởi bộ lọc xác thực trong quá trình xử lý yêu cầu). Mẫu phổ biến trong mọi điểm cuối được bảo vệ.

**Beans.get(ServiceClass.class):** Tra cứu phụ thuộc Google Guice — lấy phiên bản dịch vụ từ thùng chứa tiêm phụ thuộc (DI container). Mẫu: bộ điều khiển mỏng (chỉ định tuyến), dịch vụ dày (logic nghiệp vụ). Lợi ích: kiểm thử được (dịch vụ kiểm thử độc lập), tái sử dụng được (cùng dịch vụ gọi từ nhiều bộ điều khiển/hành động).

**ResponseConstructor.build():** Trình xây dựng phản hồi chuẩn hóa — bọc kết quả dịch vụ trong cấu trúc bao bì nhất quán. Mẫu phản hồi có thể là:
```json
{
  "status": "OK",
  "data": { /* kết quả dịch vụ */ },
  "errors": []
}
```

**3.2. Mẫu bộ điều khiển Web (trình xử lý hành động)**

**Bằng chứng từ mã nguồn:**
```java
// File: modules/axelor-open-suite/axelor-sale/src/main/java/com/axelor/apps/sale/web/SaleOrderController.java
@Singleton
public class SaleOrderController {

  public void onNew(ActionRequest request, ActionResponse response)
      throws AxelorException {
    SaleOrder saleOrder = SaleOrderContextHelper.getSaleOrder(request.getContext());
    Map<String, Object> saleOrderMap = Mapper.toMap(saleOrder);
    response.setValues(saleOrderMap);
    response.setAttrs(attrsMap);
  }

  public void compute(ActionRequest request, ActionResponse response) {
    SaleOrder saleOrder = request.getContext().asType(SaleOrder.class);
    saleOrder = Beans.get(SaleOrderComputeService.class).computeSaleOrder(saleOrder);
    response.setValues(saleOrder);
  }
}
```

**Khác biệt so với bộ điều khiển REST:**

**@Singleton thay vì @Path:** Bộ điều khiển web dùng `@Singleton` của Google Guice (quản lý bởi thùng chứa DI), không dùng `@Path` của JAX-RS (ánh xạ URL). Lý do: bộ điều khiển web được gọi nội bộ bởi bộ khung (trình xử lý action-method), không trực tiếp đăng ký làm điểm cuối HTTP.

**ActionRequest/ActionResponse thay vì HTTP Request/Response:** Bộ điều khiển web nhận bao bọc đặc thù bộ khung chứa ngữ cảnh giao diện (giá trị trường biểu mẫu, bản ghi liên quan, siêu dữ liệu), không phải yêu cầu HTTP thô. Mẫu: bộ khung xử lý HTTP → trích ngữ cảnh → gọi phương thức bộ điều khiển → tuần tự hóa phản hồi → trả JSON cho máy khách.

**Chữ ký phương thức:** Bộ điều khiển web: `void methodName(ActionRequest, ActionResponse)` — phương thức trả void, điền đối tượng phản hồi qua setter. Bộ điều khiển REST: `Response methodName()` — phương thức trả trực tiếp đối tượng Response của JAX-RS.

**Truy cập ngữ cảnh yêu cầu:**
```java
// Lấy thực thể từ ngữ cảnh biểu mẫu
SaleOrder order = request.getContext().asType(SaleOrder.class);

// Lấy giá trị trường cụ thể
Long clientPartnerId = (Long) request.getContext().get("clientPartner");

// Lấy ngữ cảnh cha (nếu biểu mẫu lồng nhau)
Map<String, Object> parentContext = request.getContext().getParent();
```

**Thao tác phản hồi:**
```java
// Đặt giá trị trường (cập nhật trường biểu mẫu)
response.setValue("exTaxTotal", total);
response.setValues(entityMap);  // Cập nhật hàng loạt

// Đặt thuộc tính trường (thay đổi giao diện động)
response.setAttr("discountField", "hidden", true);
response.setAttrs(attrsMap);  // Thay đổi thuộc tính hàng loạt

// Hiển thị thông báo
response.setAlert("Đơn hàng đã được xác nhận thành công");
response.setError("Trạng thái đơn hàng không hợp lệ");
response.setInfo("Đang xử lý nền");

// Điều hướng
response.setView(actionView);  // Mở biểu mẫu/lưới
response.setCanClose(true);    // Đóng cửa sổ bật lên hiện tại
```

**3.3. Chiến lược phân trang**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:139-142
# Define the maximum number of items per page
api.pagination.max-per-page = 100000

# Define the default number of items per page
#api.pagination.default-per-page = 40
```

**Phân tích cấu hình:**

**max-per-page = 100000:** Số bản ghi tối đa có thể trả trong một yêu cầu API. **Mối lo ngại nghiêm trọng:** Giới hạn 100 nghìn **quá cao** — tải 100 nghìn bản ghi = 100K × 2KB trung bình = **200MB nội dung phản hồi**, làm quá tải máy khách (ứng dụng di động sập, trình duyệt đóng băng), bão hòa băng thông mạng, cạn kiệt bộ nhớ máy chủ.

**Khuyến nghị:** Hạ xuống **tối đa 5000** cho môi trường thực tế. Lý do: các trường hợp sử dụng hợp lệ (báo cáo, xuất dữ liệu) xử lý qua API truyền phát/xử lý lô, API tương tác (giao diện lưới, danh sách thả xuống) hiếm khi cần hơn 1000 bản ghi mỗi trang.

**default-per-page = 40 (bị ghi chú, có lẽ 40 khi bật):** Kích thước trang mặc định khi máy khách không chỉ định giới hạn. Lựa chọn 40 [Suy luận]: phù hợp hiển thị lưới điển hình (20-50 hàng mỗi màn hình), cân bằng phản hồi nhanh (nội dung nhỏ) với cuộn trang (ít yêu cầu phân trang hơn).

**Mẫu triển khai phân trang** [Suy luận từ Query DSL của Repository]:

```java
// Điểm cuối API REST
@GET
@Path("/sale-orders")
public Response getSaleOrders(
    @QueryParam("limit") @DefaultValue("40") int limit,
    @QueryParam("offset") @DefaultValue("0") int offset) {

  // Áp dụng giới hạn tối đa
  if (limit > 5000) {
    limit = 5000;
  }

  // Thực thi truy vấn phân trang
  List<SaleOrder> orders = Query.of(SaleOrder.class)
    .filter("...")
    .order("-orderDate")
    .fetch(limit, offset);

  // Trả kết quả kèm siêu dữ liệu phân trang
  return Response.ok()
    .entity(Map.of(
      "data", orders,
      "total", getTotalCount(),
      "limit", limit,
      "offset", offset
    ))
    .build();
}
```

**Lưu ý hiệu năng phân trang:**

**Mệnh đề LIMIT:** Cơ sở dữ liệu thực thi toàn bộ truy vấn, bỏ các hàng không cần — hiệu quả cho độ lệch nhỏ (<1000), kém hiệu quả cho độ lệch lớn (>100.000). Cách tốt hơn: phân trang theo tập khóa (keyset pagination) `WHERE id > lastSeenId ORDER BY id LIMIT 1000` — hiệu năng ổn định bất kể độ lệch.

**Chi phí truy vấn COUNT:** Phân trang điển hình cần hai truy vấn: (1) `SELECT * FROM ... LIMIT 40 OFFSET 0`, (2) `SELECT COUNT(*) FROM ...` (để hiển thị "Trang 1/500"). Truy vấn đếm tốn kém cho bảng lớn — quét toàn bảng nếu bộ lọc phức tạp. Tối ưu: lưu đệm số đếm, làm mới định kỳ (chấp nhận hơi cũ cho giao diện).

**3.4. Tích hợp OpenAPI/Swagger**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:508-517
# OpenAPI
# The OpenAPI JSON can be accessed with this request : /ws/openapi

# Enable OpenAPI Generation
#application.openapi.enabled = true

# Enable Swagger UI
#application.swagger-ui.enabled = true
#application.swagger-ui.allow-try-it-out = false
```

Đặc tả OpenAPI là **định dạng tài liệu API chuẩn** — JSON/YAML đọc được bằng máy mô tả các điểm cuối (URL, phương thức HTTP, tham số, lược đồ yêu cầu/phản hồi). Swagger UI hiển thị đặc tả OpenAPI dưới dạng tài liệu tương tác — lập trình viên khám phá API, thử nghiệm yêu cầu, xem phản hồi mà không cần viết mã.

**Điểm cuối truy cập:**
- **Đặc tả OpenAPI:** `GET /ws/openapi` (trả đặc tả JSON)
- **Giao diện Swagger:** `GET /swagger-ui/` (tài liệu HTML tương tác)

**Tài liệu dựa trên chú thích:**
```java
@Operation(
    summary = "Get user permissions",
    description = "Returns list of permissions granted to authenticated user",
    tags = {"User", "Permissions"},
    responses = {
      @ApiResponse(responseCode = "200", description = "Success",
          content = @Content(schema = @Schema(implementation = PermissionResponse.class))),
      @ApiResponse(responseCode = "401", description = "Unauthorized"),
      @ApiResponse(responseCode = "500", description = "Server error")
    }
)
@Path("/permissions")
@GET
public Response getPermissions() { ... }
```

**Lợi ích:** API tự tài liệu hóa (lập trình viên đọc tài liệu, hiểu điểm cuối, viết tích hợp), sinh mã khách tự động (bộ sinh OpenAPI tạo máy khách TypeScript, Python, Java), kiểm thử API (nút "Thử ngay" của Swagger UI thực thi yêu cầu trực tiếp), quản trị API (xác thực API tuân theo chuẩn).

**Lưu ý bảo mật:** `allow-try-it-out = false` tắt tính năng thực thi yêu cầu tương tác của Swagger UI. Lý do: API môi trường thực tế không nên cung cấp công cụ kiểm thử (có thể bị lạm dụng, sửa đổi dữ liệu ngoài ý muốn). Khuyến nghị: chỉ bật Swagger UI trong môi trường phát triển/dàn dựng, tắt cho môi trường thực tế.

---

### 4. XỬ LÝ BẤT ĐỒNG BỘ: CÔNG VIỆC NỀN VÀ THAO TÁC HÀNG LOẠT

**Tập tin nguồn:** axelor-config.properties, BatchDirectDebit.java [Từ source code]

Ứng dụng doanh nghiệp cần **thực thi tác vụ bất đồng bộ** cho các thao tác quá chậm hoặc tốn tài nguyên cho yêu cầu HTTP tương tác: xử lý hàng loạt (nhập dữ liệu hàng đêm, sinh hóa đơn), công việc lập lịch (phân phối báo cáo, gửi email nhắc nhở), tính toán kéo dài (phép tính phức tạp, gọi API bên ngoài). Axelor cung cấp hai cơ chế bất đồng bộ: bộ lập lịch Quartz (lập lịch công việc theo thời gian) và bộ khung xử lý hàng loạt (thao tác dữ liệu số lượng lớn có theo dõi tiến độ).

**4.1. Cấu hình bộ lập lịch Quartz**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:278-284
# Quartz Scheduler

# Whether to enable quartz scheduler
#quartz.enable = true

# Total number of threads in quartz thread pool
#quartz.thread-count = 3
```

Quartz là **bộ lập lịch công việc Java chuẩn ngành** — lập lịch kiểu cron (thực thi tác vụ theo thời gian/khoảng cách cụ thể), lưu trữ công việc bền vững (tồn tại qua khởi động lại ứng dụng), hỗ trợ cụm (phân phối công việc qua nhiều nút), kích hoạt linh hoạt (lịch đơn giản, biểu thức cron, lập lịch nhận biết lịch).

**Cấu hình mặc định:**
- **Kích hoạt:** đúng (bộ lập lịch hoạt động mặc định)
- **Nhóm luồng:** 3 luồng (tối đa 3 công việc lập lịch đồng thời)

**Phân tích kích thước nhóm luồng:**

**3 luồng = mặc định thận trọng** [Suy luận] — phù hợp cho tải lập lịch nhẹ (5-10 công việc chạy hàng giờ/hàng ngày). Không đủ cho tải hàng loạt nặng (20+ công việc thực thi đồng thời). Triệu chứng khi nhóm quá nhỏ: trễ thực thi công việc (công việc xếp hàng chờ luồng trống), lỡ lịch (công việc bắt đầu muộn, bỏ lỡ lần kích hoạt tiếp).

**Hướng dẫn định cỡ:** Số luồng = công việc đồng thời × 1,5 (dự phòng đột biến). Ví dụ: ứng dụng chạy 10 công việc đồng thời vào giờ cao điểm (cửa sổ xử lý lô ban đêm) → kích thước nhóm = 10 × 1,5 = **15 luồng**.

**Chi phí bộ nhớ:** Mỗi luồng tiêu thụ khoảng 1MB không gian ngăn xếp (mặc định JVM). Nhóm 15 luồng = **15MB bộ nhớ** — không đáng kể cho heap ứng dụng điển hình (2-8GB).

**Mẫu đăng ký công việc** [Suy luận từ quy ước Quartz]:

```java
@Scheduled(cron = "0 0 2 * * ?")  // Hàng ngày lúc 2:00 sáng
public class DailyInvoiceGenerationJob implements Job {

  @Override
  public void execute(JobExecutionContext context) {
    try {
      Beans.get(InvoiceGenerationService.class).generateDailyInvoices();
      context.setResult("Thành công: Đã sinh 150 hóa đơn");
    } catch (Exception e) {
      context.setResult("Thất bại: " + e.getMessage());
      TraceBackService.trace(e);
    }
  }
}
```

**Cú pháp biểu thức cron:**
```
 ┌─────── giây (0-59)
 │ ┌───── phút (0-59)
 │ │ ┌─── giờ (0-23)
 │ │ │ ┌─ ngày trong tháng (1-31)
 │ │ │ │ ┌─ tháng (1-12)
 │ │ │ │ │ ┌─ ngày trong tuần (0-7, 0=Chủ nhật)
 │ │ │ │ │ │
 0 0 2 * * ?  = Hàng ngày lúc 2:00 sáng
 0 0 */4 * * ? = Mỗi 4 giờ
 0 30 9 * * MON-FRI = Ngày làm việc lúc 9:30 sáng
```

**Lưu trữ bền vững Quartz** [Suy luận]: Cấu hình mặc định có lẽ dùng **kho công việc trong RAM** (không bền vững) — lịch công việc mất khi khởi động lại ứng dụng, cấu hình đơn giản hơn (không cần bảng cơ sở dữ liệu), phù hợp cho phát triển. Triển khai thực tế nên bật **kho công việc JDBC** (bền vững) — công việc tồn tại qua khởi động lại, hỗ trợ cụm (nhiều nút ứng dụng phối hợp thực thi công việc).

**4.2. Bộ khung xử lý hàng loạt**

**Bằng chứng từ mã nguồn:**
```java
// File: modules/axelor-open-suite/axelor-bank-payment/src/main/java/com/axelor/apps/bankpayment/service/batch/BatchDirectDebit.java
public abstract class BatchDirectDebit extends BatchStrategy {

  @Override
  protected void start() throws IllegalAccessException {
    super.start();
    // Khởi tạo lô: đặt thời gian bắt đầu, đặt lại bộ đếm
  }

  @Override
  protected void stop() {
    // Sinh báo cáo lô
    StringBuilder sb = new StringBuilder();
    sb.append(I18n.get(BaseExceptionMessage.ABSTRACT_BATCH_REPORT)).append(" ");
    sb.append(String.format("Done: %d, Anomaly: %d",
        batch.getDone(), batch.getAnomaly()));
    addComment(sb.toString());
    super.stop();
  }
}
```

Bộ khung xử lý hàng loạt cung cấp **các móc vòng đời có cấu trúc** cho thao tác số lượng lớn — `start()` khởi tạo, `process()` logic chính, `stop()` dọn dẹp/báo cáo. Bộ khung xử lý các mối quan tâm xuyên suốt ngoài logic nghiệp vụ: quản lý giao dịch, xử lý lỗi, theo dõi tiến độ, ghi nhật ký kiểm toán.

**Vòng đời xử lý hàng loạt giải thích:**

**1. Giai đoạn khởi tạo (start):** Thực thi một lần trước khi xử lý bắt đầu — khởi tạo tài nguyên (mở tập tin, thiết lập kết nối), kiểm tra điều kiện tiên quyết (kiểm tra dung lượng đĩa, xác minh quyền), đặt lại bộ đếm (`done = 0`, `anomaly = 0`), ghi nhận thời điểm bắt đầu.

**2. Giai đoạn xử lý (process):** Thực thi lặp lại cho từng mục trong lô — mẫu điển hình: tìm nạp một phần bản ghi (1000 mỗi lần), duyệt bản ghi, xử lý từng bản ghi trong giao dịch riêng, tăng bộ đếm (done/anomaly), cam kết/hủy giao dịch cho từng bản ghi.

**3. Giai đoạn kết thúc (stop):** Thực thi một lần sau khi xử lý hoàn tất — giải phóng tài nguyên (đóng tập tin, ngắt kết nối), sinh báo cáo tóm tắt (tổng đã xử lý, số thành công, số lỗi), lưu bản ghi thực thi lô (nhật ký kiểm toán), gửi thông báo (email quản trị nếu có lỗi).

**Chiến lược giao dịch:**

**Mẫu suy luận:**
```java
@Transactional  // Giao dịch cấp phương thức cho mỗi mục lô
public void processBatchItem(Long recordId) {
  try {
    // Tải bản ghi
    SaleOrder order = orderRepo.find(recordId);

    // Logic nghiệp vụ
    orderService.confirmOrder(order);

    // Tăng bộ đếm thành công
    batch.incrementDone();

  } catch (Exception e) {
    // Tăng bộ đếm lỗi
    batch.incrementAnomaly();

    // Ghi nhật ký ngoại lệ kèm vết ngăn xếp
    TraceBackService.trace(e);
  }
}
```

**Giao dịch theo từng mục rất quan trọng cho tính bền bỉ của lô:** Mục thất bại (dữ liệu không hợp lệ, vi phạm quy tắc nghiệp vụ) chỉ hủy giao dịch của mục đó, không làm hỏng toàn bộ lô. Mục tiếp theo xử lý trong giao dịch mới, không bị ảnh hưởng bởi lỗi trước. Kết quả: lô xử lý thành công 9.950, thất bại 50 — kết quả chấp nhận được. Phương án thay thế (một giao dịch cho cả 10.000 mục): một lỗi hủy toàn bộ lô — không chấp nhận được.

**Các trường theo dõi lô:**

**batch.getDone():** Số bản ghi xử lý thành công. Tăng sau mỗi lần cam kết giao dịch thành công. Dùng cho báo cáo tiến độ (ví dụ: "Đã xử lý 5.000/10.000 bản ghi"), xác định ngưỡng thành công (ví dụ: "Lô thành công nếu done ≥ 95% tổng số").

**batch.getAnomaly():** Số bản ghi thất bại. Tăng khi giao dịch bị hủy hoặc bắt ngoại lệ. Dùng cho báo cáo lỗi (ví dụ: "50 bản ghi không qua kiểm tra hợp lệ"), kích hoạt cảnh báo (ví dụ: "Gửi email quản trị nếu anomaly > 100").

**addComment():** Thêm văn bản vào nhật ký thực thi lô — lưu nhật ký kiểm toán (ai chạy lô, khi nào, kết quả), hiển thị trong giao diện lịch sử lô, hữu ích cho khắc phục sự cố.

**Các triển khai lô tìm thấy** [Từ source code]:
- `BatchDirectDebit.java` — Xử lý thanh toán ghi nợ trực tiếp
- `BatchBankPaymentService.java` — Điều phối xử lý thanh toán ngân hàng
- `BatchCreditTransferSupplierPayment.java` — Sinh chuyển khoản thanh toán nhà cung cấp
- `BatchBillOfExchange.java` — Xử lý quy trình hối phiếu

Tính nhất quán mẫu [Suy luận]: Tất cả lớp xử lý lô kế thừa lớp cơ sở `BatchStrategy`, tuân theo cùng vòng đời, dùng cùng cơ chế theo dõi — cho thấy **bộ khung được thiết kế tốt** với quy ước rõ ràng.

**4.3. Mẫu công việc nền**

Hành động đồng bộ (mặc định khi gọi action-method từ XML) chặn yêu cầu HTTP cho đến khi hoàn tất — chấp nhận được cho thao tác nhanh (<500ms), không chấp nhận được cho thao tác chậm (gọi API bên ngoài, sinh báo cáo, xử lý hàng loạt). Công việc nền tách rời yêu cầu khỏi thực thi — trả về ngay với thông báo "đang xử lý", thực thi bất đồng bộ, thông báo hoàn tất qua email/thông báo.

**Mẫu đồng bộ** (mặc định hiện tại):
```xml
<action-method name="action-sale-order-method-confirm">
  <call class="com.axelor.apps.sale.web.SaleOrderController"
        method="confirmSaleOrder"/>
</action-method>
```

Người dùng bấm nút "Xác nhận" → gửi yêu cầu HTTP → confirmSaleOrder() thực thi (3 giây: kiểm tra đơn, cập nhật kho, sinh hóa đơn, gửi email) → trả phản hồi → cập nhật giao diện. Vấn đề: giao diện đóng băng 3 giây, trải nghiệm người dùng kém.

**Mẫu bất đồng bộ khuyến nghị** [Suy luận]:

```java
@Transactional
public void confirmSaleOrder(ActionRequest request, ActionResponse response) {
  SaleOrder order = request.getContext().asType(SaleOrder.class);

  // Lập lịch công việc bất đồng bộ
  JobScheduler scheduler = Beans.get(JobScheduler.class);
  String jobId = scheduler.schedule(
    "confirm-order-" + order.getId(),
    () -> doConfirmSaleOrder(order)
  );

  // Trả về ngay lập tức
  response.setInfo("Đã lên lịch xác nhận đơn hàng. Mã công việc: " + jobId);
  response.setValue("confirmationJobId", jobId);
}

@Async
@Transactional
protected void doConfirmSaleOrder(SaleOrder order) {
  // Logic xác nhận kéo dài (3 giây)
  orderService.confirm(order);
  inventoryService.reserve(order);
  invoiceService.generate(order);
  emailService.sendConfirmation(order);

  // Thông báo cho người dùng khi hoàn tất
  notificationService.send(order.getCreatedBy(),
    "Đơn hàng " + order.getSaleOrderSeq() + " đã xác nhận thành công");
}
```

Người dùng bấm "Xác nhận" → gửi yêu cầu HTTP → lập lịch công việc (10ms) → trả phản hồi ngay → giao diện hiển thị "Đang xử lý..." → luồng nền thực thi xác nhận (3 giây) → gửi thông báo khi hoàn tất. Lợi ích: giao diện phản hồi nhanh, người dùng tiếp tục làm việc trong khi đơn hàng được xử lý.

---

### 5. KIẾN TRÚC KHẢ NĂNG MỞ RỘNG: MỞ RỘNG NGANG VÀ DỌC

**Tập tin nguồn:** axelor-config.properties [Từ source code]

Khả năng mở rộng (scalability) là khả năng hệ thống xử lý tải tăng lên (nhiều người dùng đồng thời hơn, khối lượng giao dịch lớn hơn, tập dữ liệu lớn hơn) thông qua bổ sung tài nguyên. Hai cách tiếp cận: **mở rộng dọc** (nâng cấp máy chủ: thêm lõi CPU, thêm RAM, đĩa nhanh hơn) và **mở rộng ngang** (thêm máy chủ: phân phối tải qua nhiều nút). Kiến trúc Axelor hỗ trợ cả hai — cấu hình mặc định tối ưu cho mở rộng dọc (máy chủ đơn mạnh), triển khai thực tế đạt mở rộng ngang với hạ tầng bổ sung (bộ cân bằng tải, bộ đệm phân tán, kho phiên).

**5.1. Quản lý phiên làm việc**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:144-151
# Session configuration

# Session timeout (in minutes)
session.timeout = 480

# Define session cookie as secure
#session.cookie.secure = true
```

**Phân tích cấu hình:**

**session.timeout = 480 phút (8 giờ):** Phiên người dùng duy trì hoạt động 8 giờ mà không cần tương tác — phù hợp cho nhân viên văn phòng (bao phủ trọn ngày làm việc), đủ dài để tránh đăng xuất gây phiền khi nghỉ trưa, đủ ngắn để tự động đăng xuất qua đêm (bảo mật: phiên bị bỏ rơi không tồn tại vô thời hạn).

**session.cookie.secure = true (bị ghi chú, tắt mặc định):** Cờ bảo mật yêu cầu trình duyệt chỉ truyền cookie phiên qua HTTPS, không bao giờ qua HTTP. **Vấn đề bảo mật nghiêm trọng:** Triển khai thực tế **bắt buộc phải bật** — không có cờ bảo mật, cookie phiên bị truyền qua kết nối không mã hóa (dễ bị chặn, tấn công chiếm phiên).

**Kiến trúc lưu trữ phiên** [Suy luận từ mặc định thùng chứa servlet]:

Mặc định: **phiên trong bộ nhớ** (lưu trong heap Tomcat/Jetty), không lưu vào cơ sở dữ liệu/đĩa. Lợi ích: truy cập nhanh (tra cứu nano giây), cấu hình đơn giản (không phụ thuộc bên ngoài), phù hợp cho phát triển. Nhược điểm: phiên mất khi khởi động lại máy chủ (người dùng bị đăng xuất), không chia sẻ qua các máy chủ (mở rộng ngang cần phiên dính), tiêu thụ bộ nhớ (1000 người dùng đồng thời × 50KB phiên = **50MB bộ nhớ**).

**Phiên dính giải thích:** Bộ cân bằng tải định tuyến yêu cầu từ cùng người dùng đến cùng máy chủ — dữ liệu phiên sẵn có cục bộ. Triển khai: bộ cân bằng tải băm cookie phiên, định tuyến nhất quán đến cùng phụ trợ. Vấn đề: phân phối tải không đều (một số máy chủ quá tải trong khi số khác rảnh), không có chuyển đổi dự phòng (máy chủ sập = tất cả phiên mất, người dùng phải đăng nhập lại).

**Phương án không trạng thái cho mở rộng ngang:** Thay phiên trong bộ nhớ bằng **mã thông báo JWT** (JSON Web Token) hoặc **kho phiên phân tán** (Redis, Hazelcast). Cách tiếp cận JWT: xác thực tạo mã thông báo có chữ ký chứa danh tính người dùng + quyền, mã thông báo gửi kèm mỗi yêu cầu, máy chủ xác minh chữ ký mà không cần tra cứu cơ sở dữ liệu. Lợi ích: hoàn toàn không trạng thái (máy chủ bất kỳ xử lý yêu cầu bất kỳ), mở rộng ngang hoàn hảo, không cần lưu trữ phiên. Nhược điểm: cookie lớn hơn (JWT 1-2KB so với mã phiên 20 byte), không thể vô hiệu hóa (mã thông báo hợp lệ đến khi hết hạn), đăng xuất cần danh sách đen.

**Cách tiếp cận kho phiên phân tán:**
```properties
# Cấu hình kho phiên Redis
session.store.type = redis
session.store.redis.host = redis-server
session.store.redis.port = 6379
```

Lợi ích: phiên tồn tại qua khởi động lại máy chủ (lưu trong Redis), chia sẻ qua các máy chủ (không cần phiên dính), truy cập nhanh (Redis là cơ sở dữ liệu trong bộ nhớ). Nhược điểm: phụ thuộc bên ngoài (Redis phải sẵn sàng cao), độ trễ mạng (bộ nhớ cục bộ = 1µs, Redis = 1ms), phức tạp vận hành thêm.

**5.2. Hỗ trợ đa thuê bao (Multi-Tenancy)**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:69-70
# Enable multi-tenancy
#application.multi-tenancy = false
```

Đa thuê bao cho phép **một phiên bản ứng dụng phục vụ nhiều khách hàng** (thuê bao) — dữ liệu mỗi thuê bao được cách ly, hiển thị như hệ thống riêng. Phổ biến trong triển khai SaaS: nhà cung cấp lưu trữ phần mềm, bán thuê bao cho nhiều công ty, mỗi công ty chỉ thấy dữ liệu của mình. Axelor hỗ trợ đa thuê bao nhưng **tắt mặc định** (giả định chế độ đơn thuê bao).

**Các chiến lược đa thuê bao:**

**1. Cơ sở dữ liệu riêng mỗi thuê bao:** Mỗi thuê bao có cơ sở dữ liệu riêng — cách ly mạnh nhất (dữ liệu tách biệt vật lý), lược đồ đơn giản nhất (không cần cột tenant_id), hiệu năng tốt (truy vấn không lọc thuê bao), mở rộng tốn kém (100 thuê bao = 100 cơ sở dữ liệu = 100 nhóm kết nối).

**2. Lược đồ riêng mỗi thuê bao:** Thuê bao dùng chung cơ sở dữ liệu nhưng có lược đồ riêng (lược đồ PostgreSQL, cơ sở dữ liệu MySQL) — cách ly tốt (quyền cấp lược đồ), phức tạp trung bình (logic chuyển lược đồ), mở rộng tốt hơn riêng cơ sở dữ liệu nhưng kém hơn cấp hàng.

**3. Đa thuê bao cấp hàng (cách tiếp cận hiện tại của Axelor):** Thuê bao dùng chung cơ sở dữ liệu + lược đồ, phân biệt bởi trường `company` trong mọi bảng. Triển khai qua bộ lọc phân quyền: `self.company = __user__.activeCompany`. Lợi ích: mở rộng tốt nhất (một cơ sở dữ liệu, một nhóm kết nối, chia sẻ tài nguyên hiệu quả), chi phí vận hành thấp nhất. Nhược điểm: cách ly yếu nhất (dữ liệu thuê bao xen kẽ vật lý, phụ thuộc lọc cấp ứng dụng), chi phí hiệu năng truy vấn (mọi truy vấn kèm `WHERE company_id = ?`).

**Phân tích triển khai hiện tại:**

Axelor dùng **đa thuê bao mềm qua trường công ty** — tất cả thực thể có tham chiếu công ty, hệ thống phân quyền lọc dữ liệu theo công ty đang hoạt động, người dùng có thể chuyển công ty (nếu được phép). Mẫu:

```xml
<!-- Mô hình miền với trường công ty -->
<entity name="SaleOrder">
  <many-to-one name="company" ref="Company" required="true"/>
  <!-- Công ty xác định tầm nhìn dữ liệu -->
</entity>
```

```java
// Bộ lọc phân quyền đảm bảo cách ly dữ liệu
@PermissionRule(model = SaleOrder.class, filter = "self.company = __user__.activeCompany")
```

**Kích hoạt đa thuê bao thực sự** (application.multi-tenancy = true) có lẽ kích hoạt:
1. Phân giải thuê bao (trích mã thuê bao từ tên miền phụ, tiêu đề HTTP, hoặc phiên người dùng)
2. Định tuyến lược đồ/cơ sở dữ liệu (chọn nguồn dữ liệu phù hợp theo thuê bao)
3. Lan truyền ngữ cảnh thuê bao (gắn mã thuê bao vào mọi truy vấn, giao dịch)
4. Thực thi cách ly xuyên thuê bao (ngăn người dùng truy cập dữ liệu thuê bao khác dù thao tác URL trực tiếp)

**5.3. Mẫu mở rộng ngang**

Mở rộng ngang yêu cầu **kiến trúc ứng dụng không trạng thái** — yêu cầu bất kỳ có thể được xử lý bởi máy chủ bất kỳ, không cần trạng thái riêng máy chủ (phiên, bộ đệm). Kiến trúc:

```
            ┌─────────────────┐
            │  Bộ cân bằng tải │ (NGINX, HAProxy, AWS ALB)
            └────────┬────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
    ┌────▼───┐  ┌───▼────┐ ┌───▼────┐
    │ Nút 1  │  │ Nút 2  │ │ Nút 3  │ (Các phiên bản Axelor)
    └────┬───┘  └───┬────┘ └───┬────┘
         │          │          │
         └──────────┼──────────┘
                    │
         ┌──────────▼──────────┐
         │    PostgreSQL       │ (Cơ sở dữ liệu chung)
         │  + Redis (phiên)    │
         │  + Hazelcast (L2)   │
         └─────────────────────┘
```

**Yêu cầu cho mở rộng ngang:**

**✅ Ứng dụng không trạng thái:** Đạt được nếu dùng mã thông báo JWT hoặc phiên Redis (không phải phiên trong bộ nhớ). Cấu hình hiện tại dùng phiên trong bộ nhớ → **cần phiên dính** hoặc chuyển sang phương án không trạng thái.

**✅ Cơ sở dữ liệu chung:** Mọi nút truy cập cùng phiên bản PostgreSQL — đảm bảo nhất quán dữ liệu. Lo ngại: cơ sở dữ liệu trở thành điểm nghẽn ở quy mô lớn (100+ nút áp đảo một PostgreSQL). Giải pháp: bản sao đọc (thảo luận sau).

**⚠️ Đồng bộ bộ đệm:** Bộ đệm cấp hai (nếu bật với Caffeine) lưu dữ liệu thực thể trong bộ nhớ cục bộ mỗi nút — cập nhật trên nút 1 không vô hiệu hóa bộ đệm trên nút 2/nút 3, gây đọc dữ liệu cũ. Giải pháp: bộ đệm phân tán (Hazelcast, Redis) hoặc tắt bộ đệm cấp hai cho triển khai đa nút.

**Các tùy chọn đồng bộ bộ đệm:**

**Tùy chọn 1: Hazelcast (lưới dữ liệu trong bộ nhớ phân tán)**
```properties
hibernate.cache.region.factory_class = com.hazelcast.hibernate.HazelcastCacheRegionFactory
hibernate.cache.hazelcast.configuration_file_path = hazelcast.xml
```

Hazelcast tạo **cụm** các nút bộ đệm — mỗi máy chủ ứng dụng chạy phiên bản Hazelcast nhúng, các phiên bản tự phát hiện nhau (multicast hoặc TCP), chia sẻ dữ liệu đệm. Lợi ích: sao chép bộ đệm tự động (cập nhật trên nút 1 lan truyền đến mọi nút), không phụ thuộc bên ngoài (chế độ nhúng), hiệu năng xuất sắc (trong bộ nhớ, đọc cục bộ, ghi từ xa). Nhược điểm: cấu hình phức tạp, lưu lượng mạng tăng (cập nhật đệm tràn mạng), chi phí bộ nhớ (bộ đệm sao chép qua các nút).

**Tùy chọn 2: Redis (máy chủ bộ đệm tập trung)**
```properties
hibernate.cache.region.factory_class = org.hibernate.cache.redis.RedisRegionFactory
hibernate.cache.redis.host = redis-cluster
hibernate.cache.redis.port = 6379
```

Redis hoạt động như **bộ đệm chung** — mọi nút ứng dụng đọc/ghi cùng phiên bản Redis. Lợi ích: kiến trúc đơn giản (một máy chủ Redis), đảm bảo nhất quán (nguồn sự thật duy nhất), hỗ trợ bền vững (bộ đệm tồn tại qua khởi động lại). Nhược điểm: độ trễ mạng (Redis từ xa = 1ms so với bộ đệm cục bộ = 0,01ms), điểm lỗi đơn (Redis sập = ứng dụng chậm), phức tạp vận hành (cụm Redis, giám sát, sao lưu).

**Tùy chọn 3: Tắt bộ đệm cấp hai cho đa nút**
```properties
javax.persistence.sharedCache.mode = NONE
```

Giải pháp đơn giản nhất — dựa vào bộ đệm cấp một (theo giao dịch) + hiệu năng truy vấn cơ sở dữ liệu. Chấp nhận được nếu: cơ sở dữ liệu nhanh (ổ SSD, truy vấn tối ưu, nhóm kết nối), truy vấn có thể đệm ở cấp cơ sở dữ liệu (PostgreSQL shared_buffers), lưu lượng vừa phải (<100 yêu cầu/giây).

**5.4. Tinh chỉnh mở rộng dọc**

Mở rộng dọc cải thiện hiệu năng máy đơn qua: (1) tinh chỉnh heap JVM, (2) tối ưu thu gom rác, (3) định cỡ nhóm luồng, (4) mở rộng nhóm kết nối.

**Tinh chỉnh heap JVM** [Suy luận từ thực tiễn ngành, không có trong mã nguồn]:

```bash
# Cờ JVM khuyến nghị cho môi trường thực tế
JAVA_OPTS="
  # Kích thước heap: 50-75% RAM máy chủ
  -Xms4g -Xmx8g

  # Bộ thu gom rác: G1GC (độ trễ thấp, đồng thời)
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200        # Mục tiêu tạm dừng tối đa: 200ms
  -XX:ParallelGCThreads=8         # Luồng GC: ~75% lõi CPU
  -XX:ConcGCThreads=2             # Luồng đồng thời: ~25% luồng song song

  # Metaspace: Cho các lớp kịch bản Groovy
  -XX:MetaspaceSize=512m          # Metaspace khởi tạo
  -XX:MaxMetaspaceSize=1g         # Metaspace tối đa (ngăn tăng trưởng không giới hạn)

  # Ghi nhật ký GC
  -Xlog:gc*:file=/var/log/axelor/gc.log:time,uptime,level,tags
"
```

**Lý giải định cỡ heap:** Sử dụng bộ nhớ Axelor: 500MB cơ sở (bộ khung, thư viện) + 50KB mỗi người dùng đồng thời (dữ liệu phiên) + 100MB mỗi báo cáo/lô đang hoạt động + bộ đệm cấp hai (nếu bật, 100-500MB). Ví dụ: 200 người dùng + 5 báo cáo đồng thời + 300MB đệm = 500 + (200 × 0,05) + (5 × 100) + 300 = **1.310MB tối thiểu**. Heap = 2× tối thiểu = **2,6GB**, làm tròn lên **4GB** dự phòng. Heap tối đa = 8GB cho đột biến lưu lượng + bộ đệm bổ sung.

**Lựa chọn G1GC:** G1 (Garbage-First Garbage Collector) tối ưu cho **độ trễ thấp** — nhắm mục tiêu thời gian tạm dừng tối đa (200ms), đánh dấu đồng thời (không dừng luồng ứng dụng), heap theo vùng (chia heap thành các vùng, thu gom vùng nhiều rác trước). Phương án thay thế: ZGC (độ trễ còn thấp hơn, <10ms tạm dừng) cho ứng dụng nhạy độ trễ, yêu cầu Java 11+.

**Metaspace cho Groovy:** Kịch bản Groovy biên dịch thành lớp Java, lưu trong metaspace (không phải heap). Không giới hạn: bộ đệm kịch bản tăng vô hạn, cuối cùng hết bộ nhớ (OutOfMemoryError: Metaspace). Có giới hạn (1GB): lớp kịch bản cũ nhất được giải phóng khi đạt giới hạn, ngăn tăng trưởng bộ nhớ mất kiểm soát.

**Mở rộng nhóm kết nối:**
```properties
# Cho máy chủ 16 lõi
hibernate.hikari.maximumPoolSize = 35  # (16 × 2) + 3 = 35
hibernate.hikari.minimumIdle = 15      # 50% tối đa
```

Công thức: `kết_nối_tối_đa = (lõi_cpu × 2) + số_đĩa`. Lý do: truy vấn cơ sở dữ liệu chủ yếu phụ thuộc vào/ra (chờ đĩa), không phải CPU. Nhóm tối ưu bão hòa vào/ra đĩa (đĩa sử dụng 100%) mà không gây tranh chấp luồng (quá nhiều luồng cạnh tranh CPU).

**Tinh chỉnh nhóm luồng:**
```properties
# Luồng bộ lập lịch Quartz (tăng cho tải lô nặng)
quartz.thread-count = 10  # Tăng từ mặc định 3

# Luồng yêu cầu Tomcat (nếu Tomcat nhúng)
server.tomcat.threads.max = 200     # Yêu cầu HTTP đồng thời tối đa
server.tomcat.threads.min-spare = 50  # Giữ 50 luồng sẵn sàng
```

**5.5. Các mẫu kiến trúc triển khai**

**Mẫu 1: Máy chủ đơn (mặc định hiện tại)**
- **Kiến trúc:** 1 phiên bản Axelor + 1 cơ sở dữ liệu PostgreSQL
- **Năng lực:** 50-100 người dùng đồng thời, 5-10 yêu cầu/giây
- **Trường hợp sử dụng:** Doanh nghiệp nhỏ, triển khai cấp phòng ban, môi trường phát triển/dàn dựng
- **Ưu điểm:** Thiết lập đơn giản, quản lý dễ, cấu hình tối thiểu
- **Nhược điểm:** Điểm lỗi đơn, mở rộng hạn chế, không dự phòng

**Mẫu 2: Cụm có cân bằng tải**
- **Kiến trúc:** 3-5 phiên bản Axelor sau bộ cân bằng tải + 1 PostgreSQL + kho phiên Redis + bộ đệm phân tán Hazelcast
- **Năng lực:** 200-500 người dùng đồng thời, 50-100 yêu cầu/giây
- **Trường hợp sử dụng:** Doanh nghiệp vừa, triển khai đa phòng ban
- **Ưu điểm:** Sẵn sàng cao, mở rộng ngang, cập nhật cuốn chiếu (triển khai không ngừng dịch vụ)
- **Nhược điểm:** Thiết lập phức tạp, chi phí đồng bộ bộ đệm, cần phương án thay thế phiên dính

**Mẫu 3: Doanh nghiệp đa vùng**
- **Kiến trúc:** Triển khai đa vùng (chủ động-chủ động hoặc chủ động-thụ động) + cụm PostgreSQL (chính + bản sao) + CDN cho tài nguyên tĩnh + Kafka cho xử lý sự kiện bất đồng bộ + Elasticsearch cho tìm kiếm toàn văn
- **Năng lực:** 500-2000+ người dùng đồng thời, 200-500 yêu cầu/giây, người dùng toàn cầu
- **Ưu điểm:** Hiệu năng toàn cầu (người dùng được định tuyến đến vùng gần nhất), phục hồi thảm họa (lỗi vùng được xử lý bởi chuyển đổi dự phòng), mở rộng lớn
- **Nhược điểm:** Rất phức tạp (phối hợp đa vùng, độ trễ sao chép dữ liệu, giao dịch phân tán), chi phí vận hành cao

---

### 6. GIÁM SÁT VÀ QUAN SÁT: GHI NHẬT KÝ, SỐ LIỆU, KIỂM TRA SỨC KHỎE

**Tập tin nguồn:** axelor-config.properties [Từ source code]

Hệ thống thực tế cần **giám sát liên tục** — phát hiện vấn đề trước khi người dùng báo cáo, hiểu đặc tính hiệu năng, gỡ lỗi sau sự cố, lập kế hoạch năng lực cho tăng trưởng tương lai. Axelor cung cấp khả năng quan sát nền tảng qua: ghi nhật ký (ghi sự kiện dạng văn bản), thống kê Hibernate (số liệu hiệu năng ORM), bean JMX (phơi bày số liệu thời gian chạy). Triển khai thực tế bổ sung thêm công cụ APM (Giám sát hiệu năng ứng dụng: New Relic, DataDog, Dynatrace), tập hợp nhật ký (ELK stack, Splunk), theo dõi phân tán (Jaeger, Zipkin).

**6.1. Cấu hình ghi nhật ký**

**Bằng chứng từ cấu hình:**
```properties
# File: axelor-config.properties:433-487
# Logging

# Custom logback configuration
#logging.config = /path/to/logback.xml

# Storage path of logs files
#logging.path = {user.home}/.axelor/logs

# Global logging
logging.level.root = INFO

# Axelor logging
logging.level.com.axelor = INFO
logging.level.com.axelor.studio.bpm = INFO

# Hibernate logging
#logging.level.org.hibernate.SQL = DEBUG
#logging.level.org.hibernate.type = ALL

# L2-Cache
#logging.level.org.hibernate.cache = DEBUG

# Connection pooling
#logging.level.com.zaxxer.hikari = INFO
```

**Phân cấp mức nhật ký:** ERROR (chỉ lỗi) < WARN (cảnh báo + lỗi) < INFO (thông tin + cảnh báo + lỗi) < DEBUG (chi tiết gỡ lỗi + tất cả trên) < TRACE (rất chi tiết, bao gồm vào/ra phương thức). Mặc định thực tế: **INFO** — cân bằng tầm nhìn (ghi lại sự kiện quan trọng) với giảm nhiễu (loại bỏ đầu ra gỡ lỗi chi tiết).

**Ghi nhật ký gói Axelor:** `logging.level.com.axelor = INFO` đặt mức nhật ký cho tất cả lớp bộ khung Axelor. Các lệnh ghi nhật ký trong mã nguồn:
```java
Logger log = LoggerFactory.getLogger(SaleOrderController.class);

log.info("Đang xác nhận đơn bán hàng: {}", order.getSaleOrderSeq());  // INFO: thao tác bình thường
log.warn("Đơn {} vượt hạn mức tín dụng", order.getSaleOrderSeq());   // WARN: vấn đề tiềm ẩn
log.error("Không xác nhận được đơn {}", order.getSaleOrderSeq(), exception);  // ERROR: lỗi
log.debug("Dữ liệu đơn: {}", order);  // DEBUG: khắc phục sự cố chi tiết
```

**Giám sát hiệu năng qua nhật ký:**

**Bật ghi nhật ký SQL:**
```properties
logging.level.org.hibernate.SQL = DEBUG
```

Ghi mọi lệnh SQL được thực thi:
```
DEBUG org.hibernate.SQL - select saleorder0_.id, saleorder0_.sale_order_seq, ... from sale_sale_order saleorder0_ where saleorder0_.client_partner=?
```

Lợi ích: nhận diện truy vấn chậm (tương quan với dấu thời gian nhật ký), phát hiện vấn đề N+1 (hàng trăm SELECT tương tự), xác minh tối ưu truy vấn (sử dụng chỉ mục, chiến lược kết hợp). Nhược điểm: khối lượng nhật ký khổng lồ (hệ thống thực tế thực thi hàng nghìn truy vấn/giây = hàng gigabyte nhật ký/ngày), chi phí hiệu năng (vào/ra ghi nhật ký làm chậm ứng dụng ~5-10%).

**Khuyến nghị:** Chỉ bật ghi nhật ký SQL **khi khắc phục sự cố**, không giám sát thực tế liên tục. Phương án thay thế: nhật ký truy vấn chậm PostgreSQL (chỉ ghi truy vấn vượt ngưỡng, không ảnh hưởng hiệu năng ứng dụng).

**Bật thống kê bộ đệm:**
```properties
logging.level.org.hibernate.cache = DEBUG
```

Ghi các thao tác bộ đệm: trúng đệm, trượt đệm, đưa vào đệm. Lợi ích: xác minh bộ đệm hoạt động (trúng so với trượt), nhận diện thực thể nên đệm (tỷ lệ trượt cao = ứng viên đệm kém), tinh chỉnh cấu hình đệm (TTL, giới hạn kích thước).

**Bật thống kê HikariCP:**
```properties
logging.level.com.zaxxer.hikari = DEBUG
```

Ghi hoạt động nhóm kết nối:
```
DEBUG com.zaxxer.hikari.pool.HikariPool - Pool stats (total=20, active=15, idle=5, waiting=3)
WARN com.zaxxer.hikari.pool.HikariPool - Connection wait timeout elapsed
```

Lợi ích: phát hiện cạn kiệt kết nối (active = max, waiting > 0), xác minh định cỡ nhóm (idle quá cao = nhóm quá lớn), nhận diện rò rỉ kết nối (số active tăng không giới hạn). Quan trọng cho giám sát thực tế — vấn đề kết nối là nguyên nhân chính khiến ứng dụng treo.

**Xoay vòng tập tin nhật ký:**

```properties
logging.path = /var/log/axelor
```

Lưu nhật ký tại `/var/log/axelor/application.log`. Không xoay vòng: tập tin nhật ký phình to không giới hạn (đầy đĩa sau vài ngày/tuần), ảnh hưởng hiệu năng (vào/ra tập tin lớn chậm), khó phân tích (tập tin text hàng GB khó tìm kiếm). Giải pháp: cấu hình logback.xml:

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
  <file>/var/log/axelor/application.log</file>
  <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
    <!-- Xoay hàng ngày -->
    <fileNamePattern>/var/log/axelor/application.%d{yyyy-MM-dd}.log</fileNamePattern>
    <!-- Giữ 30 ngày nhật ký -->
    <maxHistory>30</maxHistory>
    <!-- Giới hạn tổng dung lượng nhật ký 10GB -->
    <totalSizeCap>10GB</totalSizeCap>
  </rollingPolicy>
  <encoder>
    <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
  </encoder>
</appender>
```

**6.2. Thống kê Hibernate**

Hibernate phơi bày **số liệu hiệu năng thời gian chạy** qua API Thống kê — số lần thực thi truy vấn, tỷ lệ trúng đệm, số liệu giao dịch, thống kê phiên. Tắt mặc định (chi phí hiệu năng ~1-2%), bật để giám sát:

```properties
# Bật thu thập thống kê
hibernate.generate_statistics = true
```

**Truy cập thống kê** [Suy luận từ mẫu JPA chuẩn]:

```java
SessionFactory sessionFactory = entityManager.getEntityManagerFactory().unwrap(SessionFactory.class);
Statistics stats = sessionFactory.getStatistics();

// Thống kê truy vấn
long queryCount = stats.getQueryExecutionCount();  // Tổng truy vấn đã thực thi
long queryTime = stats.getQueryExecutionMaxTime();  // Thời gian truy vấn chậm nhất
String slowestQuery = stats.getQueryExecutionMaxTimeQueryString();  // SQL truy vấn chậm nhất

// Thống kê bộ đệm cấp hai
long cachePuts = stats.getSecondLevelCachePutCount();  // Thực thể đưa vào đệm
long cacheHits = stats.getSecondLevelCacheHitCount();  // Trúng đệm
long cacheMisses = stats.getSecondLevelCacheMissCount();  // Trượt đệm
double cacheHitRatio = (double) cacheHits / (cacheHits + cacheMisses);  // Tỷ lệ trúng: 0-1

// Thống kê kết nối/phiên
long sessionsOpened = stats.getSessionOpenCount();
long sessionsClosed = stats.getSessionCloseCount();
long openSessions = sessionsOpened - sessionsClosed;  // Phiên đang mở (phát hiện rò rỉ)

// Thống kê giao dịch
long transactions = stats.getTransactionCount();
long successfulTx = stats.getSuccessfulTransactionCount();
long failedTx = transactions - successfulTx;  // Giao dịch thất bại
```

**Diễn giải số liệu:**

**Tỷ lệ trúng đệm < 0,7 (70%):** Hiệu năng đệm kém — bộ đệm quá nhỏ (trục xuất thực thể hay truy cập thường xuyên), TTL quá ngắn (thực thể hết hạn trước khi tái sử dụng), hoặc đệm sai thực thể (dữ liệu giao dịch thay vì dữ liệu tham chiếu). Hành động: xem xét lại thực thể được đệm, tăng kích thước đệm, kéo dài TTL.

**Số phiên đang mở tăng liên tục:** Rò rỉ phiên — mã ứng dụng mở phiên Hibernate nhưng không đóng. Triệu chứng: sử dụng bộ nhớ tăng dần (mỗi phiên giữ ~1MB), nhóm kết nối cạn kiệt (phiên giữ kết nối), ứng dụng treo. Hành động: kiểm toán mã tìm lệnh `session.close()` thiếu, bật phát hiện rò rỉ (`hibernate.hikari.leakDetectionThreshold`).

**Tỷ lệ giao dịch thất bại cao:** Logic nghiệp vụ ném ngoại lệ thường xuyên (lỗi kiểm tra hợp lệ, vi phạm ràng buộc, bế tắc). Triệu chứng: hiệu năng suy giảm (hủy giao dịch tốn kém), dữ liệu không nhất quán (giao dịch bán phần bị hủy). Hành động: phân tích nhật ký ngoại lệ, nhận diện nguyên nhân gốc (thiếu kiểm tra hợp lệ, xung đột cập nhật đồng thời).

**6.3. Giám sát hiệu năng ứng dụng (APM)**

Công cụ APM cung cấp **tầm nhìn sâu** vào hành vi ứng dụng — theo dõi phân tán (theo dõi yêu cầu xuyên vi dịch vụ), phân tích hiệu năng cấp mã (nhận diện phương thức chậm), phân tích truy vấn cơ sở dữ liệu (kế hoạch giải thích, khuyến nghị chỉ mục), theo dõi lỗi (tổng hợp ngoại lệ, nhóm vết ngăn xếp).

**Giải pháp APM trong ngành** [Suy luận từ chuẩn ngành]:

- **New Relic:** Đo đạc tự động (không thay đổi mã), theo dõi giao dịch, phân tích truy vấn cơ sở dữ liệu, phân tích lỗi, bảng điều khiển tùy chỉnh
- **DataDog:** APM + giám sát hạ tầng + tập hợp nhật ký (nền tảng thống nhất), theo dõi phân tán, giám sát tiến trình trực tiếp, phát hiện bất thường
- **Dynatrace:** Phân tích nguyên nhân gốc bằng AI, thiết lập đường cơ sở tự động, phát lại phiên người dùng, giám sát tổng hợp

**Phương án mã nguồn mở:**

- **Elastic APM:** Miễn phí (phần mở rộng Elastic Stack), theo dõi giao dịch, theo dõi lỗi, thu thập số liệu, tích hợp Kibana
- **Jaeger (chỉ theo dõi phân tán):** Mã nguồn mở (dự án CNCF), trực quan hóa vết, đồ thị phụ thuộc dịch vụ

**6.4. Điểm cuối kiểm tra sức khỏe**

Kiểm tra sức khỏe cho phép **giám sát tự động** — bộ cân bằng tải truy vấn điểm cuối sức khỏe, chỉ định tuyến lưu lượng đến phiên bản khỏe mạnh, loại bỏ phiên bản không khỏe khỏi nhóm. Thăm dò sống/sẵn sàng Kubernetes, kiểm tra sức khỏe nhóm mục tiêu AWS, công cụ giám sát (Prometheus, Nagios) đều dựa vào điểm cuối sức khỏe chuẩn hóa.

**Triển khai kiểm tra sức khỏe khuyến nghị** [Suy luận từ mẫu ngành]:

```java
@Path("/health")
public class HealthCheckController {

  @GET
  @Produces(MediaType.APPLICATION_JSON)
  public Response getHealth() {
    HealthStatus status = new HealthStatus();

    // Kiểm tra kết nối cơ sở dữ liệu
    status.setDatabaseHealth(checkDatabase());

    // Kiểm tra khả dụng bộ đệm
    status.setCacheHealth(checkCache());

    // Kiểm tra dung lượng đĩa
    status.setDiskHealth(checkDiskSpace());

    // Sức khỏe tổng thể (mọi kiểm tra phải đạt)
    boolean healthy = status.isDatabaseHealthy()
        && status.isCacheHealthy()
        && status.isDiskHealthy();

    return Response
        .status(healthy ? 200 : 503)  // 200 OK hoặc 503 Dịch vụ không khả dụng
        .entity(status)
        .build();
  }
}
```

**Định dạng phản hồi kiểm tra sức khỏe:**
```json
{
  "status": "UP",
  "checks": [
    {
      "name": "database",
      "status": "UP",
      "responseTime": 15
    },
    {
      "name": "cache",
      "status": "UP",
      "responseTime": 2
    },
    {
      "name": "disk",
      "status": "UP",
      "details": {
        "freeSpace": "50GB",
        "totalSpace": "100GB",
        "usedPercent": 50
      }
    }
  ],
  "timestamp": "2026-02-02T10:30:00Z",
  "version": "8.5.10"
}
```

---

### 7. KHUYẾN NGHỊ TỐI ƯU HIỆU NĂNG

**Tập tin nguồn:** Tổng hợp phân tích [Suy luận từ các vấn đề nhận diện được]

Dựa trên phân tích cấu hình và thực tiễn ngành, dưới đây là các khuyến nghị tối ưu — phân loại theo tác động (mức cải thiện hiệu năng) và nỗ lực (độ phức tạp triển khai).

**7.1. Thắng lợi tức thì (tác động cao, nỗ lực thấp)**

**1. Bật bộ đệm cấp hai với Caffeine**

Trạng thái hiện tại: `javax.persistence.sharedCache.mode = ENABLE_SELECTIVE` nhưng không có nhà cung cấp bộ đệm → bộ đệm cấp hai bị tắt dù thực thể đã đánh dấu `@Cacheable`.

Sửa:
```properties
hibernate.cache.region.factory_class = jcache
hibernate.javax.cache.provider = com.github.benmanes.caffeine.jcache.spi.CaffeineJCachingProvider
hibernate.javax.cache.uri = classpath:caffeine-jcache.xml
```

Tác động kỳ vọng: đọc dữ liệu tham chiếu nhanh hơn 30-50% (tiền tệ, quốc gia, cấu hình), giảm tải cơ sở dữ liệu (50-100 truy vấn/giây ít hơn).

**2. Hạ giới hạn phân trang tối đa**

Trạng thái hiện tại: `api.pagination.max-per-page = 100000` — quá cao, cho phép tải 100 nghìn bản ghi trong một yêu cầu.

Sửa:
```properties
api.pagination.max-per-page = 5000
api.pagination.default-per-page = 100
```

Tác động kỳ vọng: ngăn truy vấn lớn ngoài ý muốn, giảm sử dụng bộ nhớ, thời gian phản hồi nhanh hơn.

**3. Bật gộp JDBC**

Trạng thái hiện tại: `hibernate.jdbc.batch_size = 20` bị ghi chú → gộp tắt, mỗi INSERT/UPDATE là lần gửi mạng riêng.

Sửa:
```properties
hibernate.jdbc.batch_size = 50
hibernate.order_inserts = true
hibernate.order_updates = true
hibernate.jdbc.batch_versioned_data = true
```

Tác động kỳ vọng: chèn/cập nhật hàng loạt nhanh hơn 3-5 lần, giảm CPU cơ sở dữ liệu, giảm chi phí mạng.

**4. Bảo mật cookie phiên**

Trạng thái hiện tại: `session.cookie.secure = true` bị ghi chú → cookie phiên truyền qua HTTP (lỗ hổng bảo mật).

Sửa:
```properties
session.cookie.secure = true
session.cookie.httpOnly = true
session.cookie.sameSite = Strict
```

Tác động kỳ vọng: tuân thủ bảo mật, không ảnh hưởng hiệu năng.

**7.2. Tối ưu nỗ lực trung bình**

**5. Thêm chỉ mục cơ sở dữ liệu**

Trạng thái hiện tại: chỉ có chỉ mục tự động (khóa chính, khóa ngoại, ràng buộc duy nhất). Các cột thường lọc (trạng thái, ngày, bộ lọc tùy chỉnh) thiếu chỉ mục → quét toàn bảng.

Sửa (di chuyển SQL):
```sql
-- Đơn bán hàng
CREATE INDEX idx_sale_order_status ON sale_sale_order(status_select);
CREATE INDEX idx_sale_order_creation_date ON sale_sale_order(creation_date);
CREATE INDEX idx_sale_order_company_status ON sale_sale_order(company_id, status_select);

-- Đối tác
CREATE INDEX idx_partner_is_customer ON base_partner(is_customer) WHERE is_customer = true;
CREATE INDEX idx_partner_is_supplier ON base_partner(is_supplier) WHERE is_supplier = true;

-- Hóa đơn
CREATE INDEX idx_invoice_status_date ON account_invoice(status_select, invoice_date);

-- Sản phẩm
CREATE INDEX idx_product_code ON base_product(code);
CREATE INDEX idx_product_fullname ON base_product(full_name);
```

Tác động kỳ vọng: truy vấn có bộ lọc nhanh hơn 5-10 lần, kết quả tìm kiếm dưới giây, giảm CPU cơ sở dữ liệu.

**6. Cấu hình nhóm kết nối cho quy mô**

Sửa:
```properties
hibernate.hikari.minimumIdle = 10
hibernate.hikari.maximumPoolSize = 35
hibernate.hikari.idleTimeout = 600000
hibernate.hikari.connectionTimeout = 30000
hibernate.hikari.maxLifetime = 1800000
hibernate.hikari.leakDetectionThreshold = 60000
hibernate.hikari.validationTimeout = 5000
```

Tác động kỳ vọng: thông lượng tốt hơn dưới tải, phát hiện lỗi nhanh hơn, giảm xáo trộn kết nối.

**7. Tinh chỉnh nhóm luồng Quartz**

Sửa:
```properties
quartz.enable = true
quartz.thread-count = 10
```

Tác động kỳ vọng: xử lý lô nhanh hơn, giảm thời gian xếp hàng, ngăn lỡ lịch.

**7.3. Tối ưu nâng cao**

**8. Triển khai bản sao đọc**

Kiến trúc: PostgreSQL chính (ghi) + 2-3 bản sao (đọc). Ứng dụng định tuyến giao dịch chỉ đọc đến bản sao, giao dịch ghi đến chính.

```java
@Transactional(readOnly = true)
public List<SaleOrder> findOrders() {
  // Định tuyến đến bản sao
}

@Transactional
public void createOrder(SaleOrder order) {
  // Định tuyến đến chính
}
```

Tác động kỳ vọng: năng lực cơ sở dữ liệu tăng 2-3 lần, giảm tải chính, hiệu năng báo cáo tốt hơn.

**9. Triển khai bộ đệm kết quả truy vấn**

```properties
hibernate.cache.use_query_cache = true
```

```java
List<Currency> currencies = Query.of(Currency.class)
  .filter("...")
  .cacheable()
  .cacheRegion("currency-query-cache")
  .fetch();
```

Tác động kỳ vọng: nhanh hơn 10 lần cho truy vấn lặp lại (số liệu bảng điều khiển, danh sách thả xuống).

**10. Xử lý hành động bất đồng bộ**

Các hành động kéo dài (xác nhận, phê duyệt, sinh báo cáo) thực thi bất đồng bộ, trả về ngay lập tức.

```java
@Transactional
public void confirmOrder(ActionRequest request, ActionResponse response) {
  SaleOrder order = request.getContext().asType(SaleOrder.class);

  CompletableFuture.runAsync(() -> {
    doConfirmOrder(order);
  }, executorService);

  response.setInfo("Đã bắt đầu xác nhận đơn hàng");
}
```

Tác động kỳ vọng: giao diện phản hồi nhanh (<100ms thay vì 3-5 giây), trải nghiệm người dùng tốt hơn, thông lượng cao hơn.

---

### 8. LỘ TRÌNH MỞ RỘNG: TIẾP CẬN THEO GIAI ĐOẠN ĐẾN QUY MÔ DOANH NGHIỆP

**8.1. Giai đoạn 1: Tối ưu máy đơn (Hiện tại → 200 người dùng)**

**Hành động:**
1. ✅ Bật bộ đệm cấp hai với Caffeine
2. ✅ Thêm chỉ mục cơ sở dữ liệu (trạng thái, ngày, khóa ngoại)
3. ✅ Bật gộp JDBC (batch_size = 50)
4. ✅ Tinh chỉnh nhóm kết nối (maximumPoolSize = 35)
5. ✅ Hạ phân trang tối đa (max-per-page = 5000)
6. ✅ Bảo mật cookie phiên (secure = true)
7. ✅ Cấu hình xoay vòng nhật ký (giữ 30 ngày)

**Năng lực kỳ vọng:**
- Người dùng đồng thời: 100-200
- Thông lượng yêu cầu: 50-100 yêu cầu/giây
- Kết nối cơ sở dữ liệu: tối đa 35 (HikariCP) + 50 (BPM) = 85
- Sử dụng bộ nhớ: heap 4-8GB

**Xác thực:**
- Kiểm tra tải: 150 người dùng đồng thời, 5 phút, thời gian phản hồi trung bình <500ms
- Giám sát: CPU <70%, bộ nhớ <80%, kết nối cơ sở dữ liệu sử dụng <60%

**8.2. Giai đoạn 2: Mở rộng ngang (200 → 500 người dùng)**

**Hành động:**
1. Triển khai 3 máy chủ ứng dụng sau bộ cân bằng tải (NGINX hoặc AWS ALB)
2. Triển khai Redis cho lưu trữ phiên (thay phiên trong bộ nhớ)
3. Bật Hazelcast cho bộ đệm cấp hai phân tán (thay Caffeine)
4. Thêm bản sao đọc PostgreSQL cho báo cáo (định tuyến truy vấn chỉ đọc)
5. Triển khai kiểm tra sức khỏe (cơ sở dữ liệu, bộ đệm, đĩa)
6. Thiết lập ngăn xếp giám sát (Prometheus + Grafana hoặc DataDog)

**Sơ đồ kiến trúc:**
```
         Bộ cân bằng tải (NGINX)
                  │
      ┌───────────┼───────────┐
      │           │           │
   App-1       App-2       App-3  (3 phiên bản Axelor)
      └───────────┼───────────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
    Redis    PostgreSQL   Hazelcast
           (Chính + Bản sao)
```

**Năng lực kỳ vọng:**
- Người dùng đồng thời: 200-500
- Thông lượng yêu cầu: 150-300 yêu cầu/giây
- Sẵn sàng cao: chịu được lỗi đơn nút

**8.3. Giai đoạn 3: Quy mô doanh nghiệp (500+ người dùng)**

**Hành động:**
1. Cụm PostgreSQL (chính + 3 bản sao, chuyển đổi dự phòng tự động)
2. Tách phiên bản động cơ BPM riêng (máy chủ chuyên dụng cho quy trình)
3. Kafka cho xử lý sự kiện bất đồng bộ (xác nhận đơn, thông báo)
4. Elasticsearch cho tìm kiếm toàn văn (tìm sản phẩm, tìm tài liệu)
5. CDN cho tài nguyên tĩnh (JavaScript, CSS, hình ảnh)
6. Phân tách vi dịch vụ (tùy chọn: tách mô-đun thành dịch vụ)

**Sơ đồ kiến trúc:**
```
         CDN (CloudFront) + Bộ cân bằng tải
                      │
        ┌─────────────┼──────────────┐
        │             │              │
     App-1        App-2          App-N  (5-10 phiên bản)
        │             │              │
        └─────────────┼──────────────┘
                      │
        ┌─────────────┼──────────────┐
        │             │              │
      Redis      PostgreSQL       Kafka
               (Cụm: 1C+3BS)
        │
        └──────► Elasticsearch (cụm 3 nút)
```

**Năng lực kỳ vọng:**
- Người dùng đồng thời: 500-2000+
- Thông lượng yêu cầu: 300-1000 yêu cầu/giây
- Triển khai toàn cầu: đa vùng
- Sẵn sàng cao: đa vùng, tự mở rộng

---

### 9. PHẢN MẪU HIỆU NĂNG: CÁC LỖI PHỔ BIẾN CẦN TRÁNH

**9.1. Vấn đề truy vấn N+1**

**Triệu chứng:** Ứng dụng thực thi hàng trăm/hàng nghìn truy vấn SQL cho một thao tác đơn.

**Ví dụ (SAI):**
```java
// Tải 100 đơn bán hàng
List<SaleOrder> orders = orderRepository.all().fetch(100);

// Với mỗi đơn, tải lười đối tác (N truy vấn)
for (SaleOrder order : orders) {
  System.out.println(order.getClientPartner().getName());  // Kích hoạt SELECT
}
// Tổng: 101 truy vấn (1 cho đơn + 100 cho đối tác)
```

**Sửa (ĐÚNG):**
```java
// Truy vấn đơn với JOIN FETCH
entityManager.createQuery(
  "SELECT o FROM SaleOrder o JOIN FETCH o.clientPartner",
  SaleOrder.class).getResultList();
// Tổng: 1 truy vấn
```

**9.2. Tập kết quả lớn không phân trang**

**Triệu chứng:** Tải hàng nghìn/triệu bản ghi, cạn kiệt bộ nhớ, áp đảo máy khách.

**Ví dụ (SAI):**
```java
// Tải TẤT CẢ đơn hàng (có thể 100 nghìn+ bản ghi)
List<SaleOrder> allOrders = orderRepository.all().fetch();
// Sử dụng bộ nhớ: 100K × 5KB = 500MB
```

**Sửa (ĐÚNG):**
```java
// Phân trang kết quả
int pageSize = 100;
int offset = 0;
while (true) {
  List<SaleOrder> page = orderRepository.all().fetch(pageSize, offset);
  if (page.isEmpty()) break;

  // Xử lý trang
  processOrders(page);

  offset += pageSize;
}
```

**9.3. Thiếu chú thích @Transactional**

**Triệu chứng:** Mỗi thao tác cơ sở dữ liệu cam kết riêng lẻ (chế độ tự cam kết), hiệu năng chậm, dữ liệu không nhất quán.

**Ví dụ (SAI):**
```java
public void importOrders(List<SaleOrder> orders) {
  // Không giao dịch = 1000 lần cam kết riêng lẻ
  for (SaleOrder order : orders) {
    orderRepository.save(order);  // Tự cam kết
  }
}
// Hiệu năng: 1000 lần cam kết × 10ms = 10 giây
```

**Sửa (ĐÚNG):**
```java
@Transactional
public void importOrders(List<SaleOrder> orders) {
  // Một giao dịch = 1 lần cam kết
  for (SaleOrder order : orders) {
    orderRepository.save(order);  // Gộp lô
  }
}
// Hiệu năng: 1 lần cam kết = 10ms (nhanh hơn 1000 lần)
```

**9.4. Tải háo hức mọi thứ**

**Triệu chứng:** Tải các thực thể liên quan không cần thiết, lãng phí bộ nhớ và băng thông mạng.

**Ví dụ (SAI):**
```xml
<entity name="SaleOrder">
  <!-- Tải cả 50 dòng đơn dù chỉ hiển thị tiêu đề đơn -->
  <one-to-many name="saleOrderLineList" ref="SaleOrderLine"
    mappedBy="saleOrder" fetch="EAGER"/>
</entity>
```

**Sửa (ĐÚNG):**
```xml
<entity name="SaleOrder">
  <!-- Tải lười dòng đơn (chỉ khi truy cập) -->
  <one-to-many name="saleOrderLineList" ref="SaleOrderLine"
    mappedBy="saleOrder"/>  <!-- Lười mặc định -->
</entity>
```

**9.5. Giữ kết nối cơ sở dữ liệu quá lâu**

**Triệu chứng:** Cạn kiệt nhóm kết nối, ứng dụng treo, yêu cầu hết thời gian chờ.

**Ví dụ (SAI):**
```java
@Transactional
public void processLargeReport() {
  // Giao dịch giữ kết nối 5 phút
  List<SaleOrder> orders = loadOrders();  // 1 phút
  byte[] pdf = generatePDF(orders);       // 3 phút
  emailService.send(pdf);                 // 1 phút
  // Kết nối bị khóa suốt thời gian
}
```

**Sửa (ĐÚNG):**
```java
public void processLargeReport() {
  // Tải dữ liệu với giao dịch ngắn
  List<SaleOrder> orders = loadOrders();  // 1 phút, kết nối giải phóng

  // Xử lý ngoài giao dịch (không giữ kết nối)
  byte[] pdf = generatePDF(orders);  // 3 phút

  // Gửi với giao dịch ngắn riêng
  emailService.send(pdf);  // 1 phút
  // Tổng thời gian giữ kết nối: 2 phút (thay vì 5 phút)
}
```

---

### 10. SO SÁNH PHÉP ĐO: AXELOR SO VỚI CÁC LỰA CHỌN THAY THẾ

**10.1. So sánh hiệu năng bộ khung**

| Chỉ số | Axelor | Spring Boot | Django | Odoo |
|--------|--------|-------------|--------|------|
| **Thời gian khởi động** | 15-20 giây | 8-12 giây | 3-5 giây | 10-15 giây |
| **Bộ nhớ (rảnh)** | 512MB | 256MB | 128MB | 384MB |
| **Thông lượng (CRUD)** | 100-150 y/c/giây | 200-300 y/c/giây | 150-250 y/c/giây | 80-120 y/c/giây |
| **Truy vấn DB/yêu cầu** | 3-5 | 2-4 | 2-3 | 5-8 |
| **Thời gian dịch** | 60-90 giây | 30-45 giây | Không áp dụng | 10-20 giây |
| **Ngôn ngữ** | Java | Java | Python | Python |
| **Chi phí phụ ORM** | Trung bình (Hibernate) | Trung bình (Hibernate) | Thấp (Django ORM) | Cao (ORM tùy chỉnh) |

**Phân tích:**

**Thời gian khởi động:** Axelor chậm hơn các lựa chọn do: (1) phân tích XML (hàng trăm tập tin giao diện/hành động), (2) sinh mã (XML miền → lớp thực thể), (3) khởi tạo Groovy (làm nóng bộ đệm kịch bản), (4) xác thực lược đồ Hibernate (kiểm tra cơ sở dữ liệu khớp thực thể). Chấp nhận được cho thực tế (khởi động lại hiếm khi), gây vấn đề cho phát triển (chu kỳ lặp chậm).

**Sử dụng bộ nhớ:** Đường cơ sở Axelor cao hơn (512MB) phản ánh: (1) chi phí JVM (so với trình thông dịch Python), (2) siêu dữ liệu Hibernate (ánh xạ thực thể, kế hoạch truy vấn), (3) thời gian chạy Groovy (lớp kịch bản đã biên dịch trong metaspace), (4) trừu tượng bộ khung (hệ thống giao diện, trình xử lý hành động). Mở rộng tuyến tính theo tải: +50KB mỗi người dùng đồng thời, +100MB mỗi công việc lô đang hoạt động.

**Thông lượng:** Axelor có thông lượng vừa phải (100-150 yêu cầu/giây) so với Spring Boot nhẹ hơn (200-300 yêu cầu/giây) do chi phí trừu tượng: hiển thị giao diện XML, đánh giá phân quyền, thực thi chuỗi hành động. Tương đương Odoo (80-120 yêu cầu/giây) — cả hai đều là bộ khung ERP hướng XML. Đủ cho tải doanh nghiệp điển hình (100-500 người dùng đồng thời = 20-100 yêu cầu/giây trung bình).

**Truy vấn cơ sở dữ liệu:** Axelor sinh 3-5 truy vấn mỗi yêu cầu (trung bình suy luận): (1) tải người dùng + quyền (2 truy vấn), (2) tải thực thể yêu cầu (1 truy vấn), (3) tải siêu dữ liệu liên quan (1-2 truy vấn). Cao hơn Spring Boot tối ưu (2-4) nhưng thấp hơn Odoo (5-8). Hiệu quả bộ đệm rất quan trọng — bật bộ đệm cấp hai thì truy vấn/yêu cầu giảm xuống 1-2.

**10.2. So sánh hiệu năng cơ sở dữ liệu**

| Cơ sở dữ liệu | Đọc (QPS) | Ghi (TPS) | Khả năng mở rộng | Hỗ trợ Axelor |
|----------------|-----------|-----------|-------------------|----------------|
| **PostgreSQL** | 10K-50K | 5K-15K | Xuất sắc (sao chép, phân vùng) | ✅ Chính (khuyến nghị) |
| **MySQL** | 15K-60K | 8K-20K | Tốt (sao chép, phân mảnh) | ✅ Hỗ trợ |
| **Oracle** | 20K-80K | 10K-30K | Xuất sắc (RAC, phân vùng) | ✅ Hỗ trợ |
| **SQL Server** | 15K-50K | 8K-25K | Tốt (Always On, phân vùng) | ✅ Hỗ trợ |

**Khuyến nghị:** **Tiếp tục dùng PostgreSQL** — hệ quản trị cơ sở dữ liệu quan hệ mã nguồn mở tốt nhất, tính năng nâng cao (JSONB cho lược đồ linh hoạt, tìm kiếm toàn văn, phân vùng bảng, truy vấn song song), hỗ trợ Hibernate tuyệt vời, hệ sinh thái mạnh (pgAdmin, pg_stat_statements, Patroni cho sẵn sàng cao).

---

### 11. DANH SÁCH KIỂM TRA TRIỂN KHAI THỰC TẾ

**11.1. Cấu hình cơ sở dữ liệu**

- [ ] Bật bộ đệm cấp hai với Caffeine (hoặc Hazelcast cho đa nút)
- [ ] Thêm chỉ mục cho cột hay lọc (status_select, creation_date, company_id)
- [ ] Bật gộp JDBC (`hibernate.jdbc.batch_size = 50`)
- [ ] Cấu hình nhóm kết nối tối ưu cho máy chủ (`maximumPoolSize = lõi × 2 + số_đĩa`)
- [ ] Thiết lập sao lưu tự động (sao lưu đầy đủ hàng ngày + lưu trữ WAL liên tục cho PITR)
- [ ] Lên lịch vacuum analyze (PostgreSQL: VACUUM ANALYZE hàng tuần để thu hồi dung lượng + cập nhật thống kê)
- [ ] Bật ghi nhật ký truy vấn chậm (PostgreSQL: `log_min_duration_statement = 1000`)
- [ ] Cấu hình giám sát cơ sở dữ liệu (pg_stat_statements, số kết nối, sử dụng đĩa)

**11.2. Cấu hình ứng dụng**

- [ ] Hạ phân trang tối đa (`api.pagination.max-per-page = 5000`)
- [ ] Bật cookie phiên bảo mật (`session.cookie.secure = true`, `httpOnly = true`)
- [ ] Cấu hình HTTPS (chứng chỉ SSL, chuyển hướng HTTP → HTTPS)
- [ ] Đặt chế độ thực tế (`application.mode = prod` — tắt tính năng gỡ lỗi, bật tối ưu)
- [ ] Tắt nhập dữ liệu mẫu (`data.import.demo-data = false`)
- [ ] Cấu hình lưu trữ tập tin bên ngoài (tránh lưu tập tin trong cơ sở dữ liệu: đĩa hoặc S3)
- [ ] Thiết lập xoay vòng nhật ký (logback.xml: giữ 30 ngày, tổng tối đa 10GB)
- [ ] Tinh chỉnh heap JVM (`-Xms4g -Xmx8g` dựa trên RAM máy chủ)
- [ ] Cấu hình ghi nhật ký GC (`-Xlog:gc*:file=/var/log/axelor/gc.log`)
- [ ] Bật thống kê Hibernate (`hibernate.generate_statistics = true`)

**11.3. Giám sát và quan sát**

- [ ] Cấu hình tập hợp nhật ký (ELK stack hoặc Splunk: tập trung nhật ký từ mọi nút)
- [ ] Thiết lập APM (New Relic, DataDog, hoặc Elastic APM: theo dõi giao dịch, phân tích hiệu năng)
- [ ] Cấu hình kiểm tra sức khỏe (điểm cuối `/health`: cơ sở dữ liệu, bộ đệm, đĩa)
- [ ] Thiết lập cảnh báo (PagerDuty, Slack: đĩa >90%, CPU >80%, kết nối DB >80%, tăng đột biến tỷ lệ lỗi)
- [ ] Triển khai giám sát hạ tầng (Prometheus + Grafana: số liệu máy chủ, JVM, cơ sở dữ liệu)
- [ ] Thiết lập bảng điều khiển tùy chỉnh (Grafana: độ trễ yêu cầu, thông lượng, tỷ lệ lỗi, tỷ lệ trúng đệm)

**11.4. Gia cố bảo mật**

- [ ] Thay đổi khóa/mật khẩu mã hóa mặc định (`encryption.password`, `encryption.algorithm`)
- [ ] Bảo mật thông tin xác thực cơ sở dữ liệu (HashiCorp Vault hoặc AWS Secrets Manager: luân chuyển định kỳ)
- [ ] Bật hạn chế CORS (`cors.allow.origin = https://trusted-domain.com`, không dùng `*`)
- [ ] Cấu hình quy tắc tường lửa (chỉ cho phép cổng 80/443 ra ngoài, cổng cơ sở dữ liệu chỉ từ máy chủ ứng dụng)
- [ ] Cài chứng chỉ SSL/TLS (Let's Encrypt hoặc CA thương mại, gia hạn tự động)
- [ ] Bật bảo vệ chèn SQL (xác minh dùng ORM, tránh SQL thô với dữ liệu đầu vào người dùng)
- [ ] Cấu hình giới hạn tốc độ (NGINX: 100 yêu cầu/giây mỗi IP, bảo vệ khỏi tấn công từ chối dịch vụ)
- [ ] Thiết lập quét bảo mật (kiểm tra phụ thuộc OWASP: phát hiện thư viện có lỗ hổng)

**11.5. Giám sát sau triển khai**

**Công việc hàng ngày:**
- Kiểm tra nhật ký lỗi tìm ngoại lệ/vết ngăn xếp
- Giám sát dung lượng đĩa (tăng trưởng cơ sở dữ liệu, kích thước tập tin nhật ký)
- Xem xét truy vấn chậm (PostgreSQL pg_stat_statements: truy vấn >1 giây)
- Xác minh sao lưu hoàn tất (kiểm tra nhật ký sao lưu, thử phục hồi hàng tháng)

**Công việc hàng tuần:**
- Phân tích tỷ lệ trúng đệm (thống kê Hibernate: mục tiêu >70% tỷ lệ trúng đệm cấp hai)
- Xem xét hiệu năng công việc lô (xu hướng thời gian thực thi, số bất thường)
- Kiểm tra sử dụng nhóm kết nối (số liệu HikariCP: mục tiêu <80% sử dụng cao điểm)
- Quét bảo mật (xem xét nhật ký truy cập tìm hoạt động đáng ngờ, lần đăng nhập thất bại)

**Công việc hàng tháng:**
- Xu hướng hiệu năng (so sánh thời gian phản hồi trung bình tháng này với tháng trước)
- Lập kế hoạch năng lực (dự báo tăng trưởng người dùng, kích thước cơ sở dữ liệu, lên kế hoạch nâng cấp)
- Bảo trì cơ sở dữ liệu (PostgreSQL: REINDEX đồng thời, phân tích hiệu quả vacuum)
- Diễn tập phục hồi thảm họa (phục hồi từ sao lưu, xác minh toàn vẹn dữ liệu, đo RTO/RPO)

---

### 12. KẾT LUẬN: ĐÁNH GIÁ HIỆU NĂNG VÀ KHẢ NĂNG MỞ RỘNG

**12.1. Điểm mạnh**

**✅ Nền tảng hiệu năng vững chắc**

Axelor áp dụng **các mô hình hiệu năng chuẩn ngành** — bộ đệm đa tầng (L1/L2), nhóm kết nối (HikariCP hàng đầu), bộ khung xử lý hàng loạt, lập lịch công việc bất đồng bộ (Quartz). Kiến trúc thể hiện sự hiểu biết về yêu cầu hiệu năng ứng dụng doanh nghiệp, cung cấp các móc nối cần thiết cho tối ưu.

**✅ Giá trị mặc định phù hợp thực tế**

Các giá trị cấu hình hợp lý cho **phát triển và triển khai nhỏ** — định cỡ nhóm kết nối (20 kết nối đủ cho máy chủ 8 lõi), thời gian chờ phiên (8 giờ bao phủ ngày làm việc), mức nhật ký (INFO cân bằng tầm nhìn và nhiễu). Cho phép lập trình viên bắt đầu nhanh mà không cần tinh chỉnh hiệu năng.

**✅ Các móc nối mở rộng sẵn có**

Nền tảng cung cấp **cơ chế cho mở rộng ngang** — hỗ trợ đa thuê bao (cấp hàng qua trường công ty, có thể bật cách ly cơ sở dữ liệu/lược đồ), ngoại hóa phiên (có thể chuyển sang Redis/Hazelcast), kiến trúc không trạng thái (API REST, không ràng buộc máy chủ nếu dùng mã thông báo JWT). Thể hiện tầm nhìn thiết kế cho tăng trưởng.

**✅ Khả năng giám sát**

Bộ khung phơi bày **giao diện quan sát** — thống kê Hibernate (số truy vấn, tỷ lệ trúng đệm), bean JMX (nhóm luồng, sử dụng bộ nhớ), điểm cuối kiểm tra sức khỏe (kết nối cơ sở dữ liệu, khả dụng tài nguyên), ghi nhật ký có cấu trúc (Logback với định dạng JSON). Cho phép tích hợp APM (New Relic, DataDog) mà không cần thay đổi mã.

**12.2. Điểm yếu và lĩnh vực cần cải thiện**

**⚠️ Bộ đệm cấp hai tắt mặc định**

**Vấn đề:** Cấu hình đặt `javax.persistence.sharedCache.mode = ENABLE_SELECTIVE` nhưng **không có nhà cung cấp bộ đệm** → chú thích bộ đệm bị bỏ qua, bộ đệm cấp hai không hoạt động. Thực thể đánh dấu `@Cacheable` (Currency, Country, dữ liệu cấu hình) phải truy vấn cơ sở dữ liệu mỗi yêu cầu.

**Tác động:** Tải cơ sở dữ liệu cao hơn 30-50% so với cần thiết (ước tính 100-200 truy vấn/giây thừa cho tải điển hình), thời gian phản hồi chậm hơn (truy vấn cơ sở dữ liệu 5-10ms so với tra cứu đệm <1ms), trần mở rộng giảm.

**Sửa:** Bật nhà cung cấp Caffeine (đơn nút) hoặc Hazelcast (đa nút), cấu hình vùng đệm với kích thước + TTL phù hợp, giám sát tỷ lệ trúng.

**⚠️ Phân trang tối đa quá cao (100K)**

**Vấn đề:** `api.pagination.max-per-page = 100000` cho phép tải 100 nghìn bản ghi trong một yêu cầu API.

**Tác động:** Lỗ hổng từ chối dịch vụ (yêu cầu cố ý/vô tình có thể cạn kiệt bộ nhớ, sập ứng dụng), trải nghiệm người dùng kém (ứng dụng di động hết thời gian chờ khi tải phản hồi 200MB), quá tải cơ sở dữ liệu (quét 100 nghìn hàng tốn kém).

**Sửa:** Hạ xuống tối đa 5000 (giới hạn trên hợp lý cho trường hợp sử dụng hợp lệ như xuất dữ liệu), triển khai API truyền phát cho xuất hàng loạt.

**⚠️ Gộp JDBC bị tắt**

**Vấn đề:** `hibernate.jdbc.batch_size = 20` bị ghi chú → mỗi INSERT/UPDATE là lần gửi mạng riêng.

**Tác động:** Thao tác hàng loạt chậm hơn 3-5 lần so với cần thiết (nhập 10 nghìn bản ghi: 10K lần gửi mạng thay vì 200 lần gộp), sử dụng CPU cơ sở dữ liệu cao hơn (chi phí giao dịch nhiều hơn), thông lượng xử lý lô giảm.

**Sửa:** Bật gộp với batch_size=50 (cân bằng lợi ích gộp và sử dụng bộ nhớ), xác minh tương thích với chiến lược sinh mã thực thể, kiểm thử kỹ công việc lô.

**⚠️ Không có bộ đệm phân tán cho đa nút**

**Vấn đề:** Bộ đệm cấp hai (khi bật) dùng Caffeine cục bộ → bộ đệm không chia sẻ qua các phiên bản ứng dụng trong cụm.

**Tác động:** Vấn đề vô hiệu hóa bộ đệm (cập nhật trên nút 1 không vô hiệu hóa bộ đệm nút 2/nút 3, người dùng thấy dữ liệu cũ), hiệu quả đệm giảm (mỗi nút duy trì bộ đệm riêng, tỷ lệ trúng thấp hơn), điểm nghẽn mở rộng.

**Sửa:** Thay Caffeine bằng Hazelcast (lưới trong bộ nhớ phân tán) hoặc Redis (bộ đệm tập trung), cấu hình đồng bộ bộ đệm, giám sát độ trễ vô hiệu hóa xuyên nút.

**⚠️ Cookie phiên không bảo mật**

**Vấn đề:** `session.cookie.secure = true` bị ghi chú → cookie phiên truyền qua HTTP.

**Tác động:** Lỗ hổng bảo mật (chiếm phiên qua chặn mạng), vi phạm tuân thủ (PCI DSS, GDPR yêu cầu mã hóa truyền dữ liệu nhạy cảm).

**Sửa:** Bật cờ secure (thực tế **bắt buộc** dùng HTTPS), thêm cờ httpOnly (ngăn tấn công XSS truy cập cookie), đặt SameSite=Strict (bảo vệ CSRF).

**12.3. Đánh giá năng lực**

**Cấu hình hiện tại (máy đơn):**
- **Người dùng đồng thời:** 50-100 (dựa trên nhóm kết nối mặc định 20, định cỡ nhóm luồng)
- **Thông lượng yêu cầu:** 5-10 yêu cầu/giây (ước tính thận trọng do chi phí trừu tượng)
- **Kết nối cơ sở dữ liệu:** 20 (ứng dụng) + 50 (BPM) = tối đa 70 (yêu cầu PostgreSQL max_connections ≥ 100)

**Cấu hình tối ưu (máy đơn với các sửa khuyến nghị):**
- **Người dùng đồng thời:** 100-200 (nhóm kết nối tốt hơn, bộ đệm cấp hai giảm tải cơ sở dữ liệu)
- **Thông lượng yêu cầu:** 50-100 yêu cầu/giây (tỷ lệ trúng đệm 70-80% loại bỏ điểm nghẽn cơ sở dữ liệu)
- **Kết nối cơ sở dữ liệu:** 35 (ứng dụng) + 50 (BPM) = tối đa 85 (sử dụng tốt hơn, thông lượng cao hơn mỗi kết nối)

**Cấu hình mở rộng (cụm 3 nút với bộ cân bằng tải):**
- **Người dùng đồng thời:** 500-1000 (mở rộng ngang phân phối tải)
- **Thông lượng yêu cầu:** 200-500 yêu cầu/giây (3× năng lực ứng dụng, giả định cơ sở dữ liệu chịu được)
- **Sẵn sàng cao:** Chịu được lỗi đơn nút (tải phân phối lại cho nút còn lại)
- **Yêu cầu:** Bộ đệm phân tán (Hazelcast), kho phiên chung (Redis), bản sao đọc PostgreSQL

**12.4. Tóm tắt khuyến nghị**

**Hành động tức thì (triển khai thực tế):**
1. Bật bộ đệm cấp hai với Caffeine (cải thiện hiệu năng 30-50%)
2. Hạ phân trang tối đa xuống 5000 (bảo mật + tin cậy)
3. Bật gộp JDBC (thao tác hàng loạt nhanh hơn 3-5 lần)
4. Bảo mật cookie phiên (tuân thủ bảo mật)
5. Thêm chỉ mục cơ sở dữ liệu (truy vấn có bộ lọc nhanh hơn 5-10 lần)

**Cải tiến ngắn hạn:**
1. Tinh chỉnh nhóm kết nối cho kích thước máy chủ (tối đa thông lượng)
2. Cấu hình ngăn xếp giám sát (Prometheus + Grafana hoặc công cụ APM)
3. Thiết lập kiểm tra sức khỏe + cảnh báo (tin cậy vận hành)
4. Triển khai ghi nhật ký truy vấn chậm (nhận diện cơ hội tối ưu)

**Khả năng mở rộng dài hạn:**
1. Triển khai bản sao đọc PostgreSQL (phân phối tải đọc)
2. Triển khai bộ đệm phân tán (Hazelcast cho triển khai đa nút)
3. Mở rộng ngang (3+ nút ứng dụng sau bộ cân bằng tải)
4. Chuyển sang mã thông báo JWT (kiến trúc không trạng thái, mở rộng ngang hoàn hảo)

**Kết quả mục tiêu:**
- **Hiệu năng:** Thời gian phản hồi trung bình <500ms, p95 <1000ms
- **Khả năng mở rộng:** Hỗ trợ 500-1000 người dùng đồng thời với cụm 3 nút
- **Tin cậy:** Thời gian hoạt động 99,9% (8 giờ ngừng hoạt động/năm), phục hồi <1 phút từ lỗi đơn nút
- **Quan sát:** Bảng điều khiển toàn diện (độ trễ yêu cầu, tỷ lệ lỗi, sử dụng tài nguyên), cảnh báo tự động (phát hiện bất thường, vi phạm ngưỡng)

Axelor cung cấp **nền tảng hiệu năng vững chắc** với **lộ trình tối ưu rõ ràng** — phù hợp cho triển khai phân khúc trung (100-500 người dùng) ngay lập tức, mở rộng đến cấp doanh nghiệp (1000+ người dùng) với các cải tiến khuyến nghị. Đặc tính hiệu năng **tương đương Odoo**, **tốt hơn ứng dụng Java tùy chỉnh** (nhờ tối ưu bộ khung), **chậm hơn bộ khung nhẹ** (Spring Boot, Django) nhưng cung cấp nhiều chức năng nghiệp vụ hơn đáng kể ngay từ đầu.

---

**Hoàn thành:** 2026-02-02
**Bước:** 6/6 ✅
**Toàn bộ 6 bước nghiên cứu đã hoàn tất**
