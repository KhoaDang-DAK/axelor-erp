# BƯỚC 2: PHÂN TÍCH KIẾN TRÚC CƠ SỞ DỮ LIỆU VÀ MÔ HÌNH DỮ LIỆU - Phân Tích Từ Mã Nguồn

## Phương pháp phân tích

Quá trình nghiên cứu kiến trúc cơ sở dữ liệu và mô hình dữ liệu được thực hiện bằng cách phân tích trực tiếp các tệp XML định nghĩa thực thể (domain XML) từ nhiều mô-đun khác nhau, kiểm tra mã Java được sinh tự động, và đọc các lớp kho lưu trữ tùy chỉnh (custom repository). Phạm vi nghiên cứu bao gồm:

**Các tệp XML định nghĩa thực thể đã phân tích:**
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Address.xml` - Thực thể đơn giản có định vị địa lý
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Company.xml` - Thực thể có bộ đệm và theo dõi thay đổi
- `/modules/axelor-open-suite/axelor-base/src/main/resources/domains/Partner.xml` - Thực thể phức tạp có trường JSON
- `/modules/axelor-open-suite/axelor-sale/src/main/resources/domains/SaleOrder.xml` - Thực thể có nhiều quan hệ và logic nghiệp vụ
- `/modules/axelor-open-suite/axelor-account/src/main/resources/domains/Account.xml` - Thực thể kế toán có ràng buộc
- `/modules/axelor-open-suite/axelor-project/src/main/resources/domains/MetaJsonField.xml` - Mở rộng thực thể lõi

**Mã được sinh tự động đã kiểm tra:**
- `/modules/axelor-open-suite/axelor-base/build/src-gen/java/com/axelor/apps/base/db/Product.java` - Thực thể JPA được sinh tự động
- `/modules/axelor-open-suite/axelor-base/build/src-gen/java/com/axelor/apps/base/db/repo/ProductRepository.java` - Kho lưu trữ được sinh tự động

**Kho lưu trữ tùy chỉnh:**
- `/modules/axelor-open-suite/axelor-base/src/main/java/com/axelor/apps/base/db/repo/ProductBaseRepository.java` - Tầng logic nghiệp vụ tùy chỉnh

**Cấu hình:**
- `/src/main/resources/axelor-config.properties` - Chiến lược tạo bảng tự động của Hibernate và cài đặt cơ sở dữ liệu

---

## Kết quả chi tiết

### 1. CƠ CHẾ ĐỊNH NGHĨA THỰC THỂ BẰNG XML - PHÁT TRIỂN HƯỚNG MÔ HÌNH

**Tệp nguồn:** Tất cả tệp XML định nghĩa thực thể trong các mô-đun [Từ mã nguồn]

Axelor áp dụng một cách tiếp cận độc đáo trong hệ sinh thái Java: thay vì định nghĩa các thực thể JPA trực tiếp bằng mã Java với chú thích (annotation) như cách làm truyền thống của Hibernate hoặc Spring Data JPA, Axelor sử dụng **tệp XML định nghĩa thực thể** (Domain XML) làm nguồn sự thật duy nhất (single source of truth). Đây là một hiện thực hóa của mô hình phát triển hướng mô hình (Model-Driven Development - MDD), trong đó lập trình viên làm việc ở mức trừu tượng cao hơn thông qua lược đồ XML thay vì viết mã khuôn mẫu lặp đi lặp lại (boilerplate). Quyết định thiết kế này mang lại nhiều lợi ích: thứ nhất, những người phân tích nghiệp vụ không cần biết Java vẫn có thể đọc và rà soát định nghĩa thực thể; thứ hai, bộ sinh mã đảm bảo tính nhất quán trong việc tạo các phương thức truy xuất (getter/setter), so sánh (equals/hashCode); thứ ba, XML có thể được kiểm tra bằng lược đồ XSD trước khi biên dịch, giúp phát hiện lỗi sớm; thứ tư, dễ dàng sinh tài liệu tự động từ XML; và cuối cùng, cho phép nền tảng phát triển mà không làm hỏng các định nghĩa thực thể hiện có.

Tệp XML định nghĩa thực thể tuân theo một lược đồ chuẩn được khai báo trong tệp XSD `domain-models_7.4.xsd`, với không gian tên (namespace) `http://axelor.com/xml/ns/domain-models`. Con số 7.4 trong phiên bản lược đồ khớp với phiên bản khung ứng dụng (framework) Axelor, cho thấy lược đồ có thể phát triển theo thời gian. Mỗi tệp XML bắt đầu với một khai báo `<module>` chỉ định tên mô-đun và gói đích (target package) cho mã được sinh - ví dụ `<module name="base" package="com.axelor.apps.base.db"/>` nghĩa là tất cả thực thể trong tệp này sẽ được sinh vào gói `com.axelor.apps.base.db`. Một tệp XML có thể chứa nhiều định nghĩa thực thể, giúp tổ chức các thực thể liên quan cùng nhau (ví dụ: SaleOrder và SaleOrderLine trong cùng một tệp).

**Bằng chứng từ mã nguồn - Cấu trúc XML cơ bản:**
```xml
<domain-models xmlns="http://axelor.com/xml/ns/domain-models">
  <module name="base" package="com.axelor.apps.base.db"/>

  <entity name="EntityName" [attributes]>
    <!-- Định nghĩa các trường -->
  </entity>
</domain-models>
```

**Giải thích mã nguồn:** Khối `<domain-models>` là phần tử gốc, chứa khai báo không gian tên để trình phân tích XML có thể kiểm tra cú pháp. Phần tử `<module>` không chỉ là tài liệu mô tả - nó thực sự điều khiển quá trình sinh mã, xác định cấu trúc gói và cách đặt tên lớp. Một tệp có thể khai báo nhiều mô-đun (ví dụ khi mở rộng thực thể từ khung ứng dụng lõi), nhưng cách làm tốt nhất là mỗi tệp chỉ chứa một mô-đun.

Các thuộc tính cấp thực thể (entity-level attributes) cung cấp siêu dữ liệu (metadata) quan trọng, ảnh hưởng đến cả mã được sinh và lược đồ cơ sở dữ liệu. Thuộc tính `cacheable="true"` bật bộ đệm cấp hai (L2 cache) của Hibernate cho thực thể này - một quyết định quan trọng về hiệu năng. Các thực thể được đưa vào bộ đệm thường là những thực thể được truy cập thường xuyên và ít thay đổi như Company, Currency, Country. Việc đưa thực thể Company vào bộ đệm có ý nghĩa lớn vì hầu hết các giao dịch (hóa đơn, đơn hàng, thanh toán) đều tham chiếu đến công ty - nếu không dùng bộ đệm, hệ thống sẽ phải truy vấn cơ sở dữ liệu hàng nghìn lần mỗi ngày. Tuy nhiên, bộ đệm cũng có những đánh đổi: việc vô hiệu hóa bộ đệm (cache invalidation) phức tạp, tốn bộ nhớ, và có thể gây ra dữ liệu cũ (stale data) nếu không quản lý cẩn thận.

**Bằng chứng từ mã nguồn - Thực thể có bộ đệm:**
```xml
<entity name="Account" cacheable="true">
  <string name="name" title="Name" required="true"/>
  <string name="code" title="Code" required="true" equalsInclude="true"/>
  <!-- ... -->
</entity>
```

**Giải thích mã nguồn:** Thực thể Account được đánh dấu `cacheable="true"`, nghĩa là khi Hibernate tải một đối tượng Account, nó sẽ lưu đối tượng đó vào bộ đệm cấp hai. Các truy vấn tiếp theo cho cùng Account (theo khóa chính) sẽ lấy từ bộ đệm thay vì từ cơ sở dữ liệu. Thuộc tính `equalsInclude="true"` trên trường `code` có ý nghĩa đặc biệt: trường này sẽ được đưa vào các phương thức `equals()` và `hashCode()` được sinh tự động. Thông thường, Axelor chỉ dùng trường `id` để so sánh, nhưng đôi khi logic nghiệp vụ cần so sánh các thực thể dựa trên khóa nghiệp vụ (business key) như mã code thay vì khóa kỹ thuật như id.

Thuộc tính `implements` cho phép lớp thực thể được sinh tự động triển khai (implement) các giao diện Java (interface), từ đó hỗ trợ tính đa hình (polymorphism) và lập trình dựa trên hợp đồng (contract-based programming). Điều này đặc biệt hữu ích trong các ứng dụng nghiệp vụ, nơi nhiều thực thể chia sẻ các hành vi chung - ví dụ: SaleOrder, PurchaseOrder, Invoice đều có giá tiền (giao diện PricedOrder), đều có tiền tệ (giao diện Currenciable), đều có thông tin vận chuyển (giao diện ShippableOrder). Bằng cách khai báo giao diện trong XML, bộ sinh mã sẽ tự động thêm phần triển khai giao diện vào lớp.

**Bằng chứng từ mã nguồn - Thực thể triển khai nhiều giao diện:**
```xml
<entity name="SaleOrder"
  implements="com.axelor.apps.base.interfaces.PricedOrder,
              com.axelor.apps.base.interfaces.Currenciable,
              com.axelor.apps.base.interfaces.ShippableOrder,
              com.axelor.apps.base.interfaces.GlobalDiscounter">
  <!-- ... -->
</entity>
```

**Giải thích mã nguồn:** Thực thể SaleOrder triển khai bốn giao diện cùng lúc. Giao diện PricedOrder có thể định nghĩa các phương thức như `getTotalAmount()`, `getSubTotal()`; Currenciable có thể có `getCurrency()`, `getExchangeRate()`; ShippableOrder chứa các phương thức liên quan đến vận chuyển; GlobalDiscounter xử lý logic chiết khấu. Việc này cho phép tầng dịch vụ (service layer) viết mã tổng quát: một phương thức nhận `PricedOrder` có thể xử lý cả SaleOrder, PurchaseOrder, và Invoice mà không cần biết kiểu cụ thể. Đây là ứng dụng của Nguyên tắc Phân tách Giao diện (Interface Segregation Principle - ISP) trong bộ nguyên tắc SOLID.

Thuộc tính `table` cho phép tùy chỉnh tên bảng trong cơ sở dữ liệu thay vì dùng quy ước đặt tên mặc định. Quy ước mặc định của Axelor là `{MODULE}_{TÊN_THỰC_THỂ}` viết hoa (ví dụ: `BASE_PRODUCT`, `SALE_SALE_ORDER`), nhưng đôi khi cần ghi đè - ví dụ khi mở rộng thực thể từ khung ứng dụng lõi và muốn giữ nguyên tên bảng để đảm bảo tương thích ngược, hoặc khi tích hợp với cơ sở dữ liệu cũ có tên bảng không theo quy ước.

**Bằng chứng từ mã nguồn - Tùy chỉnh tên bảng:**
```xml
<entity name="MetaJsonField" table="META_JSON_FIELD">
  <!-- Mở rộng thực thể MetaJsonField từ lõi -->
</entity>
```

**Giải thích mã nguồn:** Thực thể MetaJsonField được ánh xạ tới bảng `META_JSON_FIELD` thay vì tên mặc định `BASE_META_JSON_FIELD`. Đây có thể là thực thể được mở rộng từ khung ứng dụng lõi, và lập trình viên muốn giữ nguyên tên bảng để không làm hỏng dữ liệu hiện có. Trường hợp này cũng cho thấy Axelor cho phép các mô-đun mở rộng thực thể từ khung lõi - một mẫu thiết kế (pattern) quan trọng cho khả năng mở rộng.

---

### 2. CÁC KIỂU TRƯỜNG VÀ HỆ THỐNG THUỘC TÍNH PHONG PHÚ

**Tệp nguồn:** Tất cả tệp XML định nghĩa thực thể đã phân tích [Từ mã nguồn]

Axelor cung cấp một hệ thống kiểu trường phong phú với hàng chục thuộc tính, cho phép lập trình viên biểu diễn các quy tắc nghiệp vụ phức tạp và hành vi giao diện ngay trong định nghĩa thực thể mà không cần viết mã Java. Sức mạnh của cách tiếp cận này nằm ở sự tập trung hóa: thay vì phân tán logic nghiệp vụ khắp các lớp thực thể, lớp dịch vụ, và bộ điều khiển giao diện, một phần đáng kể logic được tập trung trong tệp XML và được áp dụng ở cả cấp cơ sở dữ liệu (qua ràng buộc - constraint), cấp ứng dụng (qua kiểm tra hợp lệ - validation), và cấp giao diện (qua hiển thị - rendering).

**Các kiểu trường cơ bản** ánh xạ trực tiếp tới các kiểu dữ liệu SQL: `<string>` thành VARCHAR, `<integer>` thành INTEGER, `<decimal>` thành NUMERIC với độ chính xác và thang đo cấu hình được, `<boolean>` thành BOOLEAN, `<date>` thành DATE, `<datetime>` thành TIMESTAMP, `<binary>` thành BLOB. Điểm đặc biệt là Axelor đã trừu tượng hóa sự khác biệt giữa các nhà cung cấp cơ sở dữ liệu - lập trình viên chỉ cần dùng `<decimal precision="20" scale="3">` và Hibernate sẽ sinh đúng kiểu SQL cho PostgreSQL (NUMERIC(20,3)), MySQL (DECIMAL(20,3)), hoặc Oracle (NUMBER(20,3)).

Các thuộc tính cấp trường được chia làm nhiều nhóm phục vụ các mục đích khác nhau. **Nhóm thuộc tính ràng buộc** (constraint attributes) như `required="true"` (ràng buộc NOT NULL), `unique="true"` (ràng buộc UNIQUE), `min`/`max` (kiểm tra phạm vi cho số) được áp dụng ở cả cấp cơ sở dữ liệu và cấp ứng dụng. **Nhóm thuộc tính giao diện** (UI attributes) như `readonly="true"`, `hidden="true"`, `multiline="true"` điều khiển cách hiển thị trên giao diện web - trường chỉ đọc (readonly) vẫn có thể ghi trong mã nhưng hiển thị dạng chỉ đọc trên biểu mẫu, trường ẩn (hidden) không hiển thị nhưng vẫn tồn tại trong mô hình. **Nhóm thuộc tính hành vi** (behavior attributes) như `copy="false"` (loại trừ trường khi sao chép bản ghi), `massUpdate="true"` (cho phép cập nhật hàng loạt), `index="false"` (tắt tạo chỉ mục tự động) giúp tinh chỉnh hành vi ứng dụng mà không cần viết mã.

**Bằng chứng từ mã nguồn - Trường số thập phân với độ chính xác:**
```xml
<decimal name="exTaxTotal" title="Total W.T." scale="3" precision="20" readonly="true"/>
```

**Giải thích mã nguồn:** Trường này định nghĩa một số thập phân với độ chính xác 20 và thang đo 3, nghĩa là tổng cộng 20 chữ số trong đó 3 chữ số là phần thập phân - có thể lưu số lớn như 99.999.999.999.999.999,999. Độ chính xác cao này cần thiết cho các phép tính tài chính, nơi sai số làm tròn có thể tích lũy thành chênh lệch đáng kể. Thuộc tính `readonly="true"` cho biết đây là trường được tính toán (từ các dòng đơn hàng), người dùng không nhập trực tiếp - giao diện sẽ hiển thị dạng ô nhập bị vô hiệu hóa. Tiêu đề "Total W.T." là viết tắt của "Without Tax" (Tổng chưa thuế), sẽ được dùng làm nhãn trên giao diện.

Thuộc tính đặc biệt `selection` biến một trường số nguyên thành trường có dạng liệt kê (enum-like), tham chiếu đến một định nghĩa danh sách lựa chọn (có thể trong tệp XML riêng hoặc trong cơ sở dữ liệu). Đây là cách Axelor triển khai kiểu liệt kê mà vẫn giữ được tính linh hoạt - thay vì mã hóa cứng (hardcode) kiểu liệt kê trong mã Java (khó thay đổi), danh sách lựa chọn có thể được cấu hình hoặc thậm chí quản lý qua giao diện trong Axelor Studio. Giá trị lựa chọn thường là số nguyên để lưu trữ và đánh chỉ mục hiệu quả, nhưng có nhãn hiển thị cho từng giá trị.

**Bằng chứng từ mã nguồn - Trường số nguyên với danh sách lựa chọn:**
```xml
<integer name="statusSelect" title="Status"
  selection="sale.order.status.select" readonly="true"/>
```

**Giải thích mã nguồn:** Trường `statusSelect` là số nguyên nhưng hoạt động như kiểu liệt kê. Khóa danh sách `"sale.order.status.select"` tham chiếu đến một định nghĩa có thể chứa các giá trị như `{1: "Bản nháp", 2: "Đã xác nhận", 3: "Hoàn thành", 4: "Đã hủy"}`. Trong cơ sở dữ liệu lưu số nguyên (1, 2, 3, 4) nhưng giao diện hiển thị nhãn dễ đọc. Thuộc tính chỉ đọc nghĩa là việc thay đổi trạng thái phải thông qua logic nghiệp vụ (các phương thức quy trình làm việc), không phải cập nhật trực tiếp trường, ngăn chặn các chuyển trạng thái không hợp lệ.

**Trường JSON** là một cải tiến quan trọng của Axelor để hỗ trợ các trường tùy chỉnh/động. Với thuộc tính `json="true"`, một trường chuỗi sẽ lưu dữ liệu được nối tiếp hóa (serialize) dạng JSON thay vì văn bản thuần. Điều này cho phép Axelor Studio tạo trường tùy chỉnh mà không cần câu lệnh ALTER TABLE - tất cả trường tùy chỉnh được nối tiếp hóa thành JSON và lưu trong một cột duy nhất. Đánh đổi rõ ràng: tính linh hoạt cực cao (có thể thêm/xóa trường tùy chỉnh mà không cần dừng hệ thống) so với hiệu năng truy vấn (không thể đánh chỉ mục các trường bên trong JSON, không thể lọc/sắp xếp hiệu quả theo trường tùy chỉnh).

**Bằng chứng từ mã nguồn - Trường JSON cho thuộc tính tùy chỉnh:**
```xml
<string name="partnerAttrs" title="Fields" json="true"/>
```

**Giải thích mã nguồn:** Trường `partnerAttrs` lưu chuỗi JSON chứa các thuộc tính tùy chỉnh của thực thể Partner. Trong cơ sở dữ liệu, cột có thể chứa giá trị như `{"customField1": "value", "vatExempt": true, "loyaltyPoints": 1500}`. Khung ứng dụng Axelor có cơ chế nối tiếp hóa/giải nối tiếp hóa (serialization/deserialization) để chuyển đổi giữa chuỗi JSON và đối tượng Map trong Java. Người dùng nghiệp vụ thông qua Axelor Studio có thể định nghĩa trường tùy chỉnh ("Miễn thuế VAT?", "Điểm thưởng") và chúng được lưu trong đối tượng JSON này. Hạn chế: không thể truy vấn hiệu quả kiểu `SELECT * FROM partner WHERE partnerAttrs->>'vatExempt' = 'true'` (mặc dù PostgreSQL 9.4 trở lên hỗ trợ toán tử JSON, nhưng hiệu năng kém và không có chỉ mục).

**Trường tạm thời** (transient) với thuộc tính `transient="true"` là các trường được tính toán, không được lưu vào cơ sở dữ liệu. Chúng được tính toán tức thời từ các trường khác hoặc từ logic nghiệp vụ. Trường hợp sử dụng: các trường chỉ hiển thị như `fullName = firstName + " " + lastName`, hoặc giá trị dẫn xuất (derived) như `age = ngàyHiệnTại - ngàySinh`. Trường tạm thời không tốn dung lượng cơ sở dữ liệu và không gặp vấn đề dữ liệu cũ, nhưng không thể truy vấn hay lọc theo trường tạm thời.

**Bằng chứng từ mã nguồn - Trường tạm thời được tính toán:**
```xml
<many-to-one name="companyCurrency" transient="true"
  ref="com.axelor.apps.base.db.Currency">
  <![CDATA[
  return company != null ? company.getCurrency() : null;
  ]]>
</many-to-one>
```

**Giải thích mã nguồn:** Trường `companyCurrency` là quan hệ nhiều-một (many-to-one) với thực thể Currency nhưng không có cột khóa ngoại trong cơ sở dữ liệu (vì là trường tạm thời). Khối CDATA chứa biểu thức Java/Groovy để tính giá trị: nếu có công ty thì trả về đơn vị tiền tệ của công ty đó, ngược lại trả về null. Biểu thức này được thực thi mỗi khi trường được truy cập (khi gọi phương thức getter). Trường hợp sử dụng ở đây là tiện lợi: thay vì viết `order.getCompany().getCurrency()` trong mã (có nguy cơ lỗi NullPointerException nếu company là null), có thể viết `order.getCompanyCurrency()` và việc kiểm tra null đã được xử lý sẵn.

**Trường công thức** (formula) với thuộc tính `formula="true"` là các trường được tính toán ở cấp cơ sở dữ liệu thông qua truy vấn con (subquery) trong SQL. Hibernate sẽ sinh câu SQL có truy vấn con trong mệnh đề SELECT để lấy giá trị trường công thức. Trường hợp sử dụng chính là các phép tổng hợp (aggregation): tính tổng, đếm số lượng từ các thực thể liên quan. Trường công thức có đánh đổi về hiệu năng: không tốn dung lượng lưu trữ (không có cột), luôn cập nhật (không có dữ liệu cũ), nhưng mỗi truy vấn phải thực thi truy vấn con (có thể chậm nếu truy vấn con phức tạp).

**Bằng chứng từ mã nguồn - Trường công thức với truy vấn con SQL:**
```xml
<decimal name="exTaxTotalOrdered" title="Total ordered W.T." formula="true"
  precision="20" scale="10">
  <![CDATA[
  SELECT SUM(self.ex_tax_total) FROM sale_sale_order AS self
  WHERE self.origin_sale_quotation = id
  ]]>
</decimal>
```

**Giải thích mã nguồn:** Trường `exTaxTotalOrdered` tính tổng giá trị của tất cả đơn hàng bắt nguồn từ báo giá hiện tại. Truy vấn con SQL sử dụng tương quan: `WHERE self.origin_sale_quotation = id` - `id` ở đây tham chiếu đến khóa chính của báo giá hiện tại (truy vấn ngoài). Hibernate sẽ sinh SQL dạng: `SELECT q.*, (SELECT SUM(o.ex_tax_total) FROM sale_sale_order o WHERE o.origin_sale_quotation = q.id) AS exTaxTotalOrdered FROM sale_quotation q WHERE ...`. Truy vấn con được thực thi cho mỗi dòng, có thể chậm nếu tập kết quả lớn. Cách tiếp cận thay thế là tính toán trước và lưu giá trị vào bộ đệm, nhưng khi đó phải xử lý việc vô hiệu hóa bộ đệm khi các đơn hàng con thay đổi.

Thuộc tính `namecolumn="true"` đánh dấu trường này là "tên hiển thị" của thực thể, sẽ được dùng trong phương thức `toString()` và khi hiển thị thực thể trong danh sách thả xuống (dropdown) hoặc tham chiếu. Kết hợp với thuộc tính `search` (danh sách tên trường phân cách bằng dấu phẩy), Axelor biết phải tìm kiếm trong những trường nào khi người dùng gõ vào ô tìm kiếm.

**Bằng chứng từ mã nguồn - Cột tên với các trường tìm kiếm:**
```xml
<string name="fullName" namecolumn="true"
  search="addressL2,addressL3,addressL4,addressL5,addressL6" title="Address"/>
```

**Giải thích mã nguồn:** Trường `fullName` là tên hiển thị của thực thể Address. Khi người dùng chọn địa chỉ trong danh sách thả xuống, họ sẽ thấy giá trị fullName ("123 Main St, New York, NY 10001") thay vì mã kỹ thuật (12345). Thuộc tính `search` định nghĩa rằng khi người dùng tìm kiếm địa chỉ, hệ thống phải tìm trong các trường: addressL2 (dòng 2), addressL3 (dòng 3), v.v. Axelor sẽ sinh truy vấn dạng: `WHERE fullName LIKE '%từkhóa%' OR addressL2 LIKE '%từkhóa%' OR addressL3 LIKE '%từkhóa%' ...`. Điều này quan trọng cho trải nghiệm người dùng - người dùng không nhớ chính xác fullName nhưng có thể nhớ một phần ở bất kỳ dòng địa chỉ nào.

Một mẫu thiết kế thú vị là **cột tên được tính toán** - trường cột tên có thể là trường được tính toán với biểu thức CDATA thay vì trường tĩnh. Điều này cho phép xây dựng tên hiển thị linh hoạt với logic nghiệp vụ, ví dụ: bao gồm mã công ty trong tên hiển thị nếu hệ thống đa công ty, loại bỏ mã công ty nếu chỉ có một công ty.

**Bằng chứng từ mã nguồn - Cột tên được tính toán với logic điều kiện:**
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

**Giải thích mã nguồn:** Trường `label` được tính toán linh hoạt dựa trên việc thực thể có liên kết với công ty hay không. Nếu có công ty, định dạng nhãn là `MÃ_MÃCÔNGTY - Tên` (ví dụ: `PROD001_NYC - Laptop`), nếu không có thì đơn giản hơn `MÃ - Tên` (ví dụ: `PROD001 - Laptop`). Mẫu thiết kế này hữu ích trong môi trường đa công ty, nơi cùng một mã có thể tồn tại ở các công ty khác nhau - việc đưa mã công ty vào tên hiển thị giúp phân biệt. Khối CDATA chứa mã Java/Groovy sẽ được sinh thành thân phương thức getter.

---

### 3. ÁNH XẠ QUAN HỆ VÀ CHIẾN LƯỢC KHÓA NGOẠI

**Tệp nguồn:** SaleOrder.xml, Partner.xml, Account.xml, Company.xml [Từ mã nguồn]

Axelor hỗ trợ đầy đủ bốn loại quan hệ mà JPA định nghĩa: nhiều-một (many-to-one, sử dụng khóa ngoại), một-nhiều (one-to-many, quan hệ ngược), nhiều-nhiều (many-to-many, bảng trung gian), và một-một (one-to-one, khóa ngoại duy nhất hoặc khóa chính chia sẻ). Mỗi loại quan hệ có những đánh đổi riêng về chuẩn hóa cơ sở dữ liệu (normalization), hiệu năng truy vấn, và mức sử dụng bộ nhớ. Hiểu rõ khi nào dùng loại quan hệ nào là nền tảng cho thiết kế cơ sở dữ liệu.

#### 3.1. Nhiều-Một (Many-to-One): Quan hệ Khóa Ngoại

Nhiều-một là loại quan hệ phổ biến nhất, biểu diễn mối liên kết "thuộc về": một đơn hàng bán hàng (SaleOrder) thuộc về một công ty (Company), một nhân viên (Employee) thuộc về một phòng ban (Department). Cách triển khai trong cơ sở dữ liệu đơn giản: cột khóa ngoại ở bảng phía "nhiều" trỏ đến khóa chính của bảng phía "một". Hibernate mặc định sinh các đối tượng ủy nhiệm tải lười (lazy-loaded proxy) cho trường nhiều-một - khi tải thực thể, thực thể liên quan không được tải ngay (chỉ có ID), chỉ khi trường được truy cập thì đối tượng ủy nhiệm mới kích hoạt truy vấn cơ sở dữ liệu.

**Bằng chứng từ mã nguồn - Các quan hệ nhiều-một cơ bản:**
```xml
<!-- Nhiều-một đơn giản -->
<many-to-one name="company" ref="com.axelor.apps.base.db.Company"
  required="true" title="Company"/>

<!-- Nhiều-một với tên cột tùy chỉnh -->
<many-to-one name="user" column="user_id" ref="com.axelor.auth.db.User"
  title="Assigned to" index="false" massUpdate="true"/>

<!-- Nhiều-một tự tham chiếu (cấu trúc cây) -->
<many-to-one name="parentAccount" ref="Account" title="Parent Account"
  massUpdate="true"/>
```

**Giải thích mã nguồn:** Quan hệ đầu tiên rất đơn giản: mỗi thực thể phải thuộc về một công ty (`required="true"`). Câu SQL được sinh sẽ có cột `company` (hoặc `company_id` tùy theo quy ước) với ràng buộc FOREIGN KEY. Thuộc tính `ref="com.axelor.apps.base.db.Company"` chỉ định tên đầy đủ của lớp thực thể đích - bộ sinh mã cần thông tin này để sinh đúng kiểu dữ liệu Java.

Quan hệ thứ hai tùy chỉnh tên cột qua `column="user_id"` (thay vì mặc định `user`). Thuộc tính `index="false"` tắt việc tạo chỉ mục tự động - thông thường Hibernate tự động tạo chỉ mục cho khóa ngoại vì các truy vấn thường lọc/nối theo khóa ngoại, nhưng đôi khi bảng quá nhỏ hoặc khóa ngoại không bao giờ được truy vấn riêng lẻ thì không cần chỉ mục (tiết kiệm dung lượng và cải thiện tốc độ chèn). Thuộc tính `massUpdate="true"` cho phép gán lại hàng loạt: quản trị viên có thể chọn nhiều bản ghi và gán hết cho người dùng khác cùng lúc, hữu ích cho tình huống như "gán lại tất cả công việc của A cho B khi A nghỉ việc".

Quan hệ tự tham chiếu (`parentAccount`) triển khai cấu trúc cây trong SQL - cho phép xây dựng hệ thống phân cấp tài khoản như Hệ thống tài khoản kế toán (Chart of Accounts): Tài sản (cha) → Tài sản ngắn hạn (con) → Tiền mặt (cháu). Cách triển khai đơn giản nhưng việc truy vấn cây phức tạp: để lấy toàn bộ cây con cần truy vấn đệ quy (Biểu thức Bảng Chung - Common Table Expressions trong PostgreSQL) hoặc tải lặp lại (nhiều truy vấn).

#### 3.2. Một-Nhiều (One-to-Many): Quan hệ Tập hợp

Một-nhiều là mặt ngược của quan hệ nhiều-một, không tạo cột mới mà chỉ ánh xạ tới khóa ngoại đã có trong thực thể liên quan. Ví dụ: Công ty có nhiều Nhân viên (một-nhiều) là mặt ngược của Nhân viên thuộc về Công ty (nhiều-một). Thuộc tính `mappedBy` chỉ định tên trường ở phía "nhiều" chứa khóa ngoại - đây là "phía sở hữu" (owning side) của quan hệ. Hibernate chỉ lưu các thay đổi từ phía sở hữu; thay đổi ở phía không sở hữu (một-nhiều) chỉ ảnh hưởng tập hợp trong bộ nhớ, không kích hoạt câu lệnh SQL cập nhật trừ khi đã cấu hình lan truyền (cascade).

Các tập hợp một-nhiều mặc định được tải lười vì việc tải cả tập hợp có thể rất tốn kém (ví dụ: một Công ty có 10.000 nhân viên thì tải hết sẽ gây tràn bộ nhớ - OutOfMemoryError). Thuộc tính `orderBy` chỉ định thứ tự sắp xếp mặc định cho tập hợp - có thể tăng dần (`"sequence"`) hoặc giảm dần (`"-blockingToDate"` với dấu trừ phía trước).

**Bằng chứng từ mã nguồn - Các quan hệ một-nhiều:**
```xml
<!-- Một-nhiều cơ bản với tập hợp có thứ tự -->
<one-to-many name="saleOrderLineList" ref="com.axelor.apps.sale.db.SaleOrderLine"
  mappedBy="saleOrder" title="Sale order lines" orderBy="sequence"/>

<!-- Một-nhiều với thứ tự giảm dần -->
<one-to-many name="blockingList" ref="com.axelor.apps.base.db.Blocking"
  title="Blocking follow-up List" mappedBy="partner" orderBy="-blockingToDate"/>
```

**Giải thích mã nguồn:** Tập hợp `saleOrderLineList` chứa tất cả dòng đơn hàng thuộc về đơn hàng bán hàng này. Thuộc tính `mappedBy="saleOrder"` cho biết thực thể SaleOrderLine có trường nhiều-một tên `saleOrder` trỏ ngược về SaleOrder. Tập hợp được sắp xếp theo trường `sequence` (tăng dần) - các dòng đơn hàng thường có số thứ tự 1, 2, 3... để duy trì thứ tự do người dùng định nghĩa. Nếu không có orderBy, thứ tự tập hợp sẽ phụ thuộc vào cơ sở dữ liệu (không xác định), gây ra các lỗi khó phát hiện.

Tập hợp `blockingList` minh họa thứ tự giảm dần (`"-blockingToDate"`). Các bản ghi chặn (blocking) có thể đại diện cho việc tạm ngưng tín dụng, và nghiệp vụ muốn hiển thị các lần chặn gần nhất trước (ngày chặn mới nhất ở trên cùng). Dấu trừ phía trước là quy ước của Axelor cho sắp xếp giảm dần, tương tự như Django ORM. Câu SQL được sinh sẽ có `ORDER BY blockingToDate DESC`.

**Lưu ý về hiệu năng:** [Suy luận về vấn đề N+1]
Khi truy vấn một tập hợp thực thể, việc truy cập các tập hợp một-nhiều có thể gây ra vấn đề truy vấn N+1: 1 truy vấn lấy N thực thể, sau đó N truy vấn bổ sung lấy tập hợp của mỗi thực thể. Ví dụ: `SELECT * FROM company` (1 truy vấn) → với mỗi công ty, `SELECT * FROM employee WHERE company_id = ?` (N truy vấn). Giải pháp là nối tải trước (join fetch) hoặc tải theo lô (batch fetching), nhưng tệp XML định nghĩa thực thể của Axelor không cung cấp cấu hình này - lập trình viên phải xử lý ở tầng dịch vụ.

#### 3.3. Nhiều-Nhiều (Many-to-Many): Quan hệ Bảng Trung Gian

Nhiều-nhiều triển khai các quan hệ mà cả hai phía đều có nhiều thực thể - ví dụ: một Sản phẩm có nhiều Danh mục, một Danh mục chứa nhiều Sản phẩm. Cách triển khai trong cơ sở dữ liệu yêu cầu bảng trung gian (junction table) với hai khóa ngoại. Hibernate tự động sinh tên bảng trung gian theo quy ước `{thực_thể}_{tên_trường}` - ví dụ: trường `batchSet` trong thực thể Partner sẽ tạo bảng `partner_batch_set` với các cột `partner_id` và `batch_id`.

Nhiều-nhiều có một hạn chế quan trọng: không thể lưu dữ liệu bổ sung trên chính quan hệ. Nếu cần thuộc tính trên liên kết (ví dụ: Product-Category với thuộc tính `displayOrder`), phải chuyển sang hai quan hệ nhiều-một với thực thể trung gian (ProductCategory). Tệp XML định nghĩa thực thể của Axelor không có cú pháp cho lớp liên kết (association class), lập trình viên phải tự tạo thực thể trung gian.

**Bằng chứng từ mã nguồn - Các quan hệ nhiều-nhiều:**
```xml
<!-- Nhiều-nhiều cơ bản -->
<many-to-many name="batchSet" ref="com.axelor.apps.base.db.Batch" title="Batchs"/>

<!-- Nhiều-nhiều tự tham chiếu (mạng lưới liên hệ) -->
<many-to-many name="contactPartnerSet" ref="com.axelor.apps.base.db.Partner"
  title="Contacts"/>

<!-- Nhiều-nhiều trong miền nghiệp vụ -->
<many-to-many name="compatibleAccountSet"
  ref="com.axelor.apps.account.db.Account" title="Compatible Accounts"/>
```

**Giải thích mã nguồn:** Trường `batchSet` tạo quan hệ nhiều-nhiều giữa thực thể hiện tại và các thực thể Batch. Bảng trung gian `{thực_thể}_batch_set` sẽ có các cột `{thực_thể}_id` và `batch_id` cùng với khóa chính tổ hợp (composite primary key) trên cả hai cột để ngăn trùng lặp. Hibernate quản lý bảng trung gian tự động - khi thêm/xóa phần tử từ tập hợp, Hibernate chèn/xóa dòng trong bảng trung gian.

Quan hệ nhiều-nhiều tự tham chiếu (`contactPartnerSet`) triển khai mối quan hệ kiểu mạng xã hội: một Đối tác (Partner) có thể có nhiều Đối tác liên hệ, và mỗi Đối tác liên hệ cũng có các liên hệ riêng. Bảng trung gian `partner_contact_partner_set` ánh xạ Đối tác đến Đối tác. Quan hệ này thường đối xứng (nếu A liên hệ B thì B liên hệ A) nhưng cách triển khai ở đây không bắt buộc tính đối xứng - logic ứng dụng phải tự xử lý.

Trường `compatibleAccountSet` minh họa quan hệ nhiều-nhiều trong miền nghiệp vụ cụ thể. Trong kế toán, một Tài khoản có thể tương thích với một tập hợp Tài khoản khác cho một số thao tác nhất định (ví dụ: chuyển khoản, đối chiếu). Quan hệ này không đối xứng: Tài khoản A tương thích với B không có nghĩa B tương thích với A. Các quy tắc nghiệp vụ về tính tương thích có thể được áp dụng ở tầng dịch vụ thay vì ở ràng buộc cơ sở dữ liệu.

#### 3.4. Một-Một (One-to-One): Quan hệ Duy nhất

Một-một là loại quan hệ ít dùng nhất trong bốn loại vì nó có thể được mô hình hóa bằng bảng riêng (với khóa ngoại duy nhất) hoặc bằng các trường nhúng trong cùng bảng. Trường hợp sử dụng cho bảng riêng một-một: (1) quan hệ tùy chọn có nhiều trường có thể null (tách bảng tiết kiệm dung lượng), (2) tải lười các trường nặng (ví dụ: User có một Profile với trường tiểu sử dài), (3) vòng đời/mẫu truy cập khác nhau.

**Bằng chứng từ mã nguồn - Các quan hệ một-một:**
```xml
<!-- Một-một đơn giản (phía sở hữu) -->
<one-to-one name="emailAddress" ref="com.axelor.message.db.EmailAddress"
  title="Email" unique="true"/>

<!-- Một-một hai chiều (phía không sở hữu) -->
<one-to-one name="linkedUser" ref="com.axelor.auth.db.User"
  title="User" mappedBy="partner"/>
```

**Giải thích mã nguồn:** Trường `emailAddress` tạo quan hệ một-một trong đó thực thể hiện tại "sở hữu" quan hệ (có cột khóa ngoại). Thuộc tính `unique="true"` áp dụng ràng buộc cơ sở dữ liệu ngăn hai thực thể chia sẻ cùng một EmailAddress - đây là điều phân biệt một-một với nhiều-một.

Trường `linkedUser` là mặt ngược của quan hệ một-một hai chiều. Thực thể User (không hiển thị ở đây) có trường `partner` ánh xạ ngược lại. Thuộc tính `mappedBy="partner"` cho biết quan hệ được sở hữu bởi thực thể User, không có cột khóa ngoại trong bảng Partner. Quan hệ này có thể biểu diễn: một Đối tác (công ty/liên hệ) có thể liên kết với một Người dùng hệ thống, và một Người dùng có thể liên kết với một Đối tác. Bản chất hai chiều cho phép điều hướng cả hai hướng: từ Đối tác tới Người dùng hoặc từ Người dùng tới Đối tác.

**Lưu ý về tải lười:** [Suy luận từ hành vi Hibernate]
Quan hệ một-một có hành vi tải lười (lazy loading) phức tạp: nếu là phía không sở hữu (có mappedBy), Hibernate phải truy vấn cơ sở dữ liệu để kiểm tra xem thực thể liên quan có tồn tại không, khiến tải "lười" thực chất không thực sự lười. Phía sở hữu có thể tải lười thực sự vì chỉ cần kiểm tra cột khóa ngoại (không null = thực thể tồn tại). Mã nguồn nhạy cảm về hiệu năng nên kiểm tra xem một-một có thực sự được lợi từ tải lười hay không.

---

### 4. RÀNG BUỘC, CHỈ MỤC VÀ KIỂM SOÁT LƯỢC ĐỒ CƠ SỞ DỮ LIỆU

**Tệp nguồn:** SaleOrder.xml, Account.xml, Company.xml [Từ mã nguồn]

Ràng buộc (constraint) và chỉ mục (index) trong cơ sở dữ liệu rất quan trọng cho tính toàn vẹn dữ liệu và hiệu năng truy vấn, nhưng chúng có những đánh đổi. Ràng buộc (duy nhất, không null, khóa ngoại, kiểm tra) áp dụng quy tắc nghiệp vụ ở cấp cơ sở dữ liệu - tuyến phòng thủ cuối cùng chống lại dữ liệu xấu, ngay cả khi lỗi ứng dụng bỏ qua bước kiểm tra hợp lệ. Chỉ mục tăng tốc truy vấn theo cấp số nhân (O(log n) thay vì O(n)) nhưng làm chậm thao tác chèn/cập nhật/xóa và tiêu tốn dung lượng đĩa. Tệp XML định nghĩa thực thể của Axelor cho phép khai báo ràng buộc và có một số quyền kiểm soát về chỉ mục, mặc dù không toàn diện bằng chú thích JPA thuần.

#### 4.1. Ràng buộc Duy nhất: Đơn Cột và Tổ hợp

Ràng buộc duy nhất (unique constraint) ngăn chặn giá trị trùng lặp, rất quan trọng cho các khóa nghiệp vụ (như mã đơn hàng, mã sản phẩm, địa chỉ email). Theo chuẩn SQL, ràng buộc duy nhất cho phép nhiều giá trị NULL vì NULL khác NULL (giá trị không xác định không bằng giá trị không xác định).

Axelor hỗ trợ ràng buộc duy nhất đơn cột qua thuộc tính trường `unique="true"`, và ràng buộc duy nhất tổ hợp (composite) qua phần tử `<unique-constraint>`. Ràng buộc duy nhất tổ hợp áp dụng tính duy nhất trên nhiều cột - ví dụ: (saleOrderSeq, company) duy nhất nghĩa là cùng một mã đơn hàng có thể tồn tại ở các công ty khác nhau nhưng không được trùng trong cùng một công ty. Mẫu thiết kế này là nền tảng cho hệ thống đa công ty.

**Bằng chứng từ mã nguồn - Các ràng buộc duy nhất:**
```xml
<!-- Ràng buộc duy nhất đơn cột (trong thuộc tính trường) -->
<string name="code" title="Code" required="true" unique="true"/>

<!-- Ràng buộc duy nhất tổ hợp nhiều cột -->
<unique-constraint columns="saleOrderSeq,company"/>
<unique-constraint columns="code,company"/>
```

**Giải thích mã nguồn:** Trường `code` có cả `required="true"` và `unique="true"`, tạo ra ràng buộc NOT NULL UNIQUE - sự kết hợp này tương đương với khóa chính tự nhiên (natural primary key), nhưng hệ thống vẫn có khóa thay thế (surrogate ID) vì lý do kỹ thuật. Người dùng nghiệp vụ thường tìm kiếm/tham chiếu theo mã thay vì ID, khiến mẫu thiết kế này rất phổ biến.

Ràng buộc duy nhất tổ hợp `saleOrderSeq,company` rất quan trọng cho môi trường triển khai đa công ty. Bộ sinh mã tuần tự có thể sinh mã độc lập theo từng công ty (SO0001, SO0002 ở Công ty A và SO0001, SO0002 ở Công ty B), hoặc sinh mã duy nhất toàn cục. Ràng buộc này áp dụng chiến lược trước. Ràng buộc thứ hai `code,company` tương tự cho mã tài khoản - cho phép tái sử dụng cùng mã tài khoản giữa các công ty (ví dụ: mọi công ty đều có tài khoản "Tiền mặt" với mã "101").

**Chú thích JPA được sinh:** [Từ mã nguồn - tệp Product.java được sinh]
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

**Giải thích mã nguồn:** Hibernate chuyển đổi ràng buộc duy nhất từ XML thành chú thích `@UniqueConstraint` trong `@Table`. Câu lệnh DDL của cơ sở dữ liệu sẽ có `UNIQUE (saleOrderSeq, company)`. Nếu mã ứng dụng cố chèn bản ghi trùng lặp, cơ sở dữ liệu ném ra ngoại lệ SQLException và Hibernate bọc thành ConstraintViolationException, được đẩy lên tầng dịch vụ để xử lý thân thiện (hiển thị thông báo lỗi cho người dùng thay vì sập ứng dụng).

#### 4.2. Chỉ mục: Tự động, Tắt, và Ảnh hưởng Hiệu năng

Hibernate mặc định tự động tạo chỉ mục cho các cột khóa ngoại vì các thao tác nối (join) và lọc (filter) theo khóa ngoại cực kỳ phổ biến. Chỉ mục tăng tốc truy vấn đáng kể: `SELECT * FROM sale_order WHERE company_id = 123` với chỉ mục trên company_id mất vài micro giây (tra cứu cây B-tree), không có chỉ mục mất vài giây (quét toàn bộ bảng). Tuy nhiên, chỉ mục không miễn phí: mỗi chỉ mục là một cấu trúc cây B-tree riêng biệt chiếm dung lượng đĩa, và mọi thao tác INSERT/UPDATE/DELETE đều phải cập nhật tất cả chỉ mục, làm chậm thao tác ghi.

Axelor cho phép tắt chỉ mục tự động qua thuộc tính `index="false"`. Các trường hợp sử dụng: (1) cột khóa ngoại không bao giờ được truy vấn/nối độc lập (luôn là phần của điều kiện tổ hợp), (2) bảng rất nhỏ (quét toàn bộ đủ nhanh), (3) khối lượng ghi lớn, nơi tốc độ chèn quan trọng hơn tốc độ truy vấn.

**Bằng chứng từ mã nguồn - Kiểm soát chỉ mục:**
```xml
<many-to-one name="user" column="user_id" ref="com.axelor.auth.db.User"
  title="Assigned to" index="false" massUpdate="true"/>
```

**Giải thích mã nguồn:** Khóa ngoại `user_id` được đặt rõ ràng `index="false"`, ngăn Hibernate tạo chỉ mục. Quyết định này có thể dựa trên phân tích: nếu bảng không bao giờ được truy vấn theo người dùng (ví dụ: luôn truy vấn theo khóa chính hoặc các cột đã đánh chỉ mục khác), thì chỉ mục trên user_id lãng phí dung lượng. Tuy nhiên, cần cẩn thận: nếu sau này thêm tính năng lọc theo người dùng, truy vấn sẽ chậm và lập trình viên phải nhớ thêm chỉ mục thủ công.

**Bằng chứng từ mã được sinh - Chỉ mục tự động:**
```java
@Table(
  name = "BASE_PRODUCT",
  indexes = {
    @Index(columnList = "name"),
    @Index(columnList = "picture"),
    @Index(columnList = "product_category"),
    @Index(columnList = "unit"),
    // ... nhiều chỉ mục cho khóa ngoại
  }
)
```

**Giải thích mã nguồn:** Thực thể Product được sinh có nhiều chỉ mục được tạo tự động. Chỉ mục trên `name` phục vụ tìm kiếm theo tên sản phẩm (thao tác phổ biến của người dùng). Chỉ mục trên các khóa ngoại như `product_category`, `unit` phục vụ lọc sản phẩm theo danh mục, theo đơn vị đo lường. Bộ sinh của Hibernate thêm các chỉ mục này dựa trên quy ước và kiểu trường. Lập trình viên không có quyền kiểm soát chi tiết trong tệp XML (không thể chỉ định loại chỉ mục - B-tree hay Hash hay GIN, không thể tạo chỉ mục tổ hợp), phải dựa vào giá trị mặc định của bộ sinh hoặc tạo chỉ mục thủ công sau khi triển khai.

**Ảnh hưởng hiệu năng:** [Suy luận về chiến lược chỉ mục]
Chiến lược chỉ mục tối ưu phụ thuộc vào khối lượng công việc. Hệ thống đọc nhiều (nhiều truy vấn, ít ghi) được hưởng lợi từ việc đánh chỉ mục rộng rãi. Hệ thống ghi nhiều (chèn/cập nhật thường xuyên) nên giảm thiểu chỉ mục. Chỉ mục tổ hợp nhiều cột hữu ích khi truy vấn lọc theo nhiều cột cùng lúc (ví dụ: `WHERE company_id = ? AND date >= ?` được hưởng lợi từ chỉ mục trên (company_id, date)). Axelor không cho phép khai báo chỉ mục tổ hợp trong XML, lập trình viên có thể cần tạo chúng qua tập lệnh di trú (migration script) hoặc trực tiếp trong cơ sở dữ liệu.

---

### 5. PHƯƠNG THỨC TÌM KIẾM: SINH TRUY VẤN KHAI BÁO

**Tệp nguồn:** SaleOrder.xml, Account.xml, Partner.xml [Từ mã nguồn]

Phương thức tìm kiếm (finder method) là một tính năng tiện lợi của Axelor cho phép lập trình viên khai báo các mẫu truy vấn phổ biến trong tệp XML, và bộ sinh mã tự động tạo các phương thức truy vấn trong lớp kho lưu trữ (Repository). Mẫu thiết kế này giảm đáng kể mã khuôn mẫu (boilerplate) - thay vì viết thủ công cú pháp truy vấn trong kho lưu trữ, chỉ cần khai báo `<finder-method name="findByCode" using="code"/>` và bộ sinh sẽ tạo phương thức hoàn chỉnh với ràng buộc tham số, kiểm tra null, và xử lý lỗi đúng cách.

Cú pháp đơn giản nhưng hiệu quả: thuộc tính `name` chỉ định tên phương thức (quy ước: `findBy{TênTrường}`), thuộc tính `using` liệt kê các trường dùng trong truy vấn (phân cách bằng dấu phẩy nếu nhiều trường), và thuộc tính tùy chọn `all="true"` cho biết phương thức trả về danh sách (List) thay vì một thực thể duy nhất. Bộ sinh chuyển đổi các khai báo này thành các lời gọi DSL truy vấn với bộ lọc JPQL và tham số ràng buộc phù hợp.

**Bằng chứng từ mã nguồn - Khai báo phương thức tìm kiếm:**
```xml
<!-- Tìm kiếm theo một trường -->
<finder-method name="findByPartnerSeq" using="partnerSeq"/>

<!-- Tìm kiếm theo nhiều trường -->
<finder-method name="findBySaleOrderSeqAndCompany" using="saleOrderSeq,company"/>
<finder-method name="findByCodeAndCompany" using="code,company"/>

<!-- Tìm tất cả bản ghi khớp (trả về danh sách) -->
<finder-method name="findByAccountType" using="accountType" all="true"/>
```

**Giải thích mã nguồn:** Phương thức `findByPartnerSeq` sinh ra phương thức nhận tham số String partnerSeq và trả về một thực thể Partner duy nhất (hoặc null nếu không tìm thấy). Trường hợp sử dụng: tìm đối tác theo khóa nghiệp vụ duy nhất. Phương thức tìm nhiều trường `findBySaleOrderSeqAndCompany` truy vấn theo khóa nghiệp vụ tổ hợp - cả mã đơn hàng VÀ công ty đều phải khớp. Chữ ký phương thức sẽ là `findBySaleOrderSeqAndCompany(String orderSeq, Company company)` với hai tham số.

Phương thức `findByAccountType` có `all="true"`, sinh ra kiểu trả về `List<Account>` thay vì một Account duy nhất. Trường hợp sử dụng: lấy tất cả tài khoản thuộc một loại nhất định (tài khoản tài sản, tài khoản nợ phải trả, v.v.). Nếu không có `all="true"`, phương thức chỉ trả về kết quả đầu tiên khớp (hoặc null); với `all="true"` trả về tất cả kết quả khớp (danh sách rỗng nếu không tìm thấy).

**Bằng chứng từ mã kho lưu trữ được sinh:**
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

**Giải thích mã nguồn:** Bộ sinh chuyển đổi các khai báo tìm kiếm thành các lời gọi DSL truy vấn. Phương thức `findByCode` tạo đối tượng Query cho lớp Product, thêm bộ lọc với tham số đặt tên (`:code`), ràng buộc giá trị tham số, và gọi `fetchOne()` để lấy một kết quả. Chuỗi bộ lọc `"self.code = :code"` sử dụng quy ước Axelor: `self` là bí danh (alias) cho thực thể hiện tại. DSL truy vấn chuyển đổi thành JPQL: `SELECT self FROM Product self WHERE self.code = :code`. Tham số đặt tên (`:code`) an toàn hơn tham số vị trí vì không thể nhầm lẫn thứ tự, và dễ đọc hơn.

Phương thức trả về thực thể trực tiếp (không phải Optional) - giá trị null nghĩa là không tìm thấy. Cách tiếp cận này đơn giản hơn mẫu Optional trong Java 8+, nhưng có nguy cơ lỗi NullPointerException nếu mã gọi không kiểm tra null.

**Hạn chế:** [Suy luận về khả năng phương thức tìm kiếm]
Phương thức tìm kiếm chỉ hỗ trợ truy vấn bằng (equality query, `trường = giá_trị`), không hỗ trợ truy vấn phạm vi (`trường > giá_trị`), truy vấn LIKE (`trường LIKE '%mẫu%'`), điều kiện OR, sắp xếp, hoặc phân trang. Các truy vấn phức tạp yêu cầu phương thức viết thủ công trong kho lưu trữ tùy chỉnh. Đánh đổi: các phương thức tìm kiếm khai báo đáp ứng 80% trường hợp phổ biến (lấy theo ID, lấy theo khóa nghiệp vụ) mà không cần viết mã, 20% phức tạp còn lại vẫn cần mã tùy chỉnh.

---

### 6. MÃ NHÚNG BỔ SUNG VÀ HẰNG SỐ: NHÚNG JAVA VÀO XML

**Tệp nguồn:** SaleOrder.xml, Account.xml, Partner.xml, Company.xml [Từ mã nguồn]

Một tính năng mạnh mẽ của tệp XML định nghĩa thực thể trong Axelor là khả năng nhúng mã Java trực tiếp vào định nghĩa thực thể qua các phần tử `<extra-code>` và `<extra-imports>`. Cơ chế này cho phép bộ sinh tạo ra các lớp thực thể hoạt động đầy đủ với hằng số, phương thức hỗ trợ, và logic nghiệp vụ nội tuyến (inline), mà không cần tệp Java riêng biệt. Trường hợp sử dụng chính là định nghĩa hằng số cho các trường số nguyên dạng liệt kê (mã trạng thái, mã loại) - các hằng số này giúp mã dễ đọc và an toàn khi tái cấu trúc (refactor).

Mẫu thiết kế phổ biến: định nghĩa các trường danh sách lựa chọn số nguyên (`statusSelect`, `typeSelect`) cùng với các hằng số tương ứng trong khối mã bổ sung (extra-code). Mã ứng dụng sau đó tham chiếu hằng số (`SaleOrder.STATUS_CONFIRMED`) thay vì số ma thuật (magic number) (`3`), cải thiện đáng kể khả năng đọc hiểu. Khi nghiệp vụ thay đổi mã trạng thái (ví dụ: chèn trạng thái mới giữa các trạng thái hiện có), chỉ cần cập nhật hằng số, không cần tìm kiếm khắp mã nguồn.

**Bằng chứng từ mã nguồn - Phần nhập bổ sung và hằng số:**
```xml
<extra-imports>
  import com.axelor.apps.base.interfaces.GlobalDiscounterLine;
</extra-imports>

<extra-code>
  <![CDATA[
  // TRẠNG THÁI
  public static final int STATUS_DRAFT_QUOTATION = 1;
  public static final int STATUS_FINALIZED_QUOTATION = 2;
  public static final int STATUS_ORDER_CONFIRMED = 3;
  public static final int STATUS_ORDER_COMPLETED = 4;
  public static final int STATUS_CANCELED = 5;

  // TRẠNG THÁI ĐẶT HÀNG
  public static final int ORDERING_STATUS_PARTIALLY_ORDERED = 1;
  public static final int ORDERING_STATUS_CLOSED = 2;
  ]]>
</extra-code>
```

**Giải thích mã nguồn:** Khối `<extra-imports>` thêm các câu lệnh nhập (import) sẽ xuất hiện ở đầu tệp Java được sinh. Cần thiết khi mã bổ sung tham chiếu các lớp từ gói khác. Phần CDATA trong `<extra-code>` chứa mã Java nguyên bản được sao chép trực tiếp vào thân lớp được sinh. Các hằng số định nghĩa ở đây trở thành thành viên tĩnh (static member) của lớp thực thể được sinh, có thể truy cập dạng `SaleOrder.STATUS_CONFIRMED`.

Các chú thích nhóm (`// TRẠNG THÁI`, `// TRẠNG THÁI ĐẶT HÀNG`) tổ chức hằng số thành các phần logic, cải thiện khả năng bảo trì. Mẫu thiết kế này đặc biệt quan trọng cho các thực thể có nhiều trường danh sách lựa chọn (hơn 10 hằng số trạng thái, hơn 5 hằng số loại, v.v.).

**Ví dụ khác từ Account.xml:**
```xml
<extra-code><![CDATA[
  // VỊ TRÍ THÔNG THƯỜNG
  public static final int COMMON_POSITION_NONE = 0;
  public static final int COMMON_POSITION_CREDIT = 1;
  public static final int COMMON_POSITION_DEBIT = 2;

  // TRẠNG THÁI
  public static final int STATUS_INACTIVE = 0;
  public static final int STATUS_ACTIVE = 1;

  // HỆ THỐNG THUẾ GTGT
  public static final int VAT_SYSTEM_DEFAULT = 0;
  public static final int VAT_SYSTEM_GOODS = 1;
  public static final int VAT_SYSTEM_SERVICE = 2;
]]></extra-code>
```

**Giải thích mã nguồn:** Thực thể Account có ba nhóm hằng số cho các trường danh sách lựa chọn khác nhau. Hằng số VAT_SYSTEM phân biệt các quy tắc xử lý thuế giá trị gia tăng khác nhau (hàng hóa so với dịch vụ có thuế suất khác nhau ở nhiều quốc gia). COMMON_POSITION (nợ/có) là nền tảng trong kế toán kép (double-entry bookkeeping). STATUS kiểm soát tài khoản đang hoạt động hay đã lưu trữ.

Tầng dịch vụ sử dụng hằng số dạng: `if (order.getStatusSelect() == SaleOrder.STATUS_CONFIRMED) { ... }`. Mẫu này tốt hơn đáng kể so với số ma thuật: `if (order.getStatusSelect() == 3) { ... }` - không thể hiểu được nếu không tra cứu ý nghĩa mã trạng thái.

---

### 7. THEO DÕI KIỂM TOÁN: LỊCH SỬ THAY ĐỔI TÍCH HỢP SẴN

**Tệp nguồn:** SaleOrder.xml, Account.xml, Partner.xml, Company.xml [Từ mã nguồn]

Axelor cung cấp một hệ thống dấu vết kiểm toán (audit trail) toàn diện được khai báo trực tiếp trong tệp XML qua phần tử `<track>`. Hệ thống này tự động ghi lại AI đã thay đổi CÁI GÌ vào KHI NÀO, tạo ra nhật ký lịch sử không thể sửa đổi cho mọi trường được theo dõi. Tính năng này rất quan trọng cho các yêu cầu tuân thủ (ví dụ: GDPR ở Liên minh Châu Âu yêu cầu nhật ký kiểm toán chi tiết về việc truy cập/sửa đổi dữ liệu cá nhân), bảo mật (điều tra pháp y khi có sự cố), và nghiệp vụ (giải quyết tranh chấp khi có câu hỏi về các thay đổi dữ liệu).

Cơ chế theo dõi có thể chọn lọc: theo dõi các trường cụ thể thay vì toàn bộ thực thể (giảm khối lượng nhật ký), chỉ theo dõi khi TẠO hoặc chỉ khi CẬP NHẬT (kiểm soát chi tiết), và thậm chí sinh thông điệp dễ đọc về các thay đổi trạng thái (ví dụ: "Đơn hàng đã xác nhận" khi trạng thái thay đổi thành 3). Thông điệp có thể có điều kiện dựa trên giá trị trường và có nhãn trực quan (quan trọng, thông tin, thành công, cảnh báo) để tô sáng trên giao diện.

**Bằng chứng từ mã nguồn - Cấu hình theo dõi toàn diện:**
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

**Giải thích mã nguồn:** Cấu hình theo dõi cho thực thể SaleOrder định nghĩa các trường nào được theo dõi và các thông điệp cần sinh. Các trường `saleOrderSeq`, `clientPartner`, `statusSelect`, `inTaxTotal` được theo dõi trên cả sự kiện TẠO và CẬP NHẬT (không có thuộc tính `on` nghĩa là cả hai). Trường `creationDate` chỉ theo dõi `on="CREATE"` - hợp lý vì ngày tạo chỉ được đặt một lần, các thay đổi tiếp theo sẽ là lỗi. Ngược lại, `confirmationDateTime` chỉ theo dõi `on="UPDATE"` - trường này null khi tạo, chỉ được điền khi đơn hàng được xác nhận sau đó.

Phần thông điệp tạo các mục nhật ký kiểm toán dễ đọc. Thông điệp đầu tiên `if="true"` luôn kích hoạt khi TẠO, ghi "Báo giá/đơn hàng đã được tạo". Các thông điệp tiếp theo có điều kiện theo giá trị statusSelect: khi trạng thái là 1, ghi "Bản nháp báo giá" với nhãn quan trọng (tô sáng đỏ/cam trên giao diện), khi trạng thái là 3, ghi "Đơn hàng đã xác nhận" với nhãn thành công (tô sáng xanh). Nhãn hoàn toàn phục vụ hiển thị nhưng giúp người dùng quét nhanh nhật ký kiểm toán, phát hiện các sự kiện quan trọng.

---

### 8. BỘ LẮNG NGHE THỰC THỂ: MẪU MÓC NỐI VÒNG ĐỜI

**Tệp nguồn:** Account.xml [Từ mã nguồn]

Bộ lắng nghe thực thể JPA (entity listener) cung cấp các móc nối (hook) vào các sự kiện vòng đời của thực thể (trước khi lưu - prePersist, trước khi cập nhật - preUpdate, sau khi tải - postLoad, v.v.), cho phép chạy logic tùy chỉnh tại các thời điểm cụ thể mà không cần sửa đổi trực tiếp lớp thực thể. Axelor cho phép khai báo bộ lắng nghe qua phần tử `<entity-listener>` trong tệp XML, liên kết thực thể được sinh với lớp lắng nghe. Mẫu thiết kế này được ưu tiên hơn việc đặt logic trực tiếp trong lớp thực thể vì: (1) thực thể chỉ nên chứa trạng thái, không chứa logic (mẫu mô hình miền thiếu máu - anemic domain model), (2) bộ lắng nghe có thể tiêm dịch vụ (inject service) qua cơ chế tiêm phụ thuộc (dependency injection) trong khi thực thể không nên có phụ thuộc, (3) nhiều bộ lắng nghe có thể được thêm mà không sửa đổi thực thể.

**Bằng chứng từ mã nguồn - Khai báo bộ lắng nghe thực thể:**
```xml
<entity-listener
  class="com.axelor.apps.account.db.repo.listener.AccountListener"/>
```

**Giải thích mã nguồn:** Thực thể Account được liên kết với lớp AccountListener. Bộ sinh thêm chú thích `@EntityListeners(AccountListener.class)` vào thực thể Account được sinh. Lớp AccountListener phải triển khai các phương thức hồi gọi (callback) của bộ lắng nghe JPA như:

```java
public class AccountListener {
  @PrePersist
  public void prePersist(Account account) {
    // Logic trước khi chèn (INSERT)
  }

  @PreUpdate
  public void preUpdate(Account account) {
    // Logic trước khi cập nhật (UPDATE)
  }

  @PostLoad
  public void postLoad(Account account) {
    // Logic sau khi truy vấn (SELECT)
  }
}
```

**Trường hợp sử dụng cho bộ lắng nghe:** [Suy luận về các mẫu phổ biến]
- **Kiểm tra hợp lệ:** Kiểm tra nghiệp vụ phức tạp không thể biểu diễn qua ràng buộc (ví dụ: "tài khoản nợ không được có số dư bên có")
- **Trường dẫn xuất:** Tính toán trường trước khi lưu (ví dụ: `fullName = firstName + lastName`, `total = đơnGiá * sốLượng`)
- **Ghi nhật ký kiểm toán tùy chỉnh:** Logic kiểm toán vượt quá khả năng theo dõi tích hợp sẵn
- **Tích hợp hệ thống bên ngoài:** Thông báo hệ thống bên ngoài khi thực thể thay đổi (webhook, hàng đợi tin nhắn)
- **Vô hiệu hóa bộ đệm:** Xóa bộ đệm khi thực thể bị sửa đổi

**Hạn chế:** [Suy luận về hiệu năng và độ phức tạp]
Bộ lắng nghe có chi phí hiệu năng - mọi sự kiện vòng đời thực thể đều phải gọi các phương thức lắng nghe, ngay cả khi không cần logic tùy chỉnh. Trong các thao tác hàng loạt (chèn hàng nghìn bản ghi), chi phí bộ lắng nghe có thể nhân lên. Cách tiếp cận thay thế: xử lý logic rõ ràng trong các phương thức tầng dịch vụ, gọi các phương thức tiện ích khi cần, cho nhiều quyền kiểm soát hơn về thời điểm logic được thực thi.

---

### 9. CƠ CHẾ SINH MÃ: TỪ XML ĐỊNH NGHĨA ĐẾN THỰC THỂ JPA

**Tệp nguồn:** `settings.gradle` (cấu hình trình cắm Gradle), mã được sinh trong `build/src-gen/` [Từ mã nguồn]

Quy trình sinh mã là trái tim của cách tiếp cận phát triển hướng mô hình (Model-Driven Development) trong Axelor. Thay vì viết và bảo trì thủ công hàng trăm lớp thực thể với mã khuôn mẫu lặp lại (getter/setter, equals/hashCode, chú thích JPA), lập trình viên chỉ cần bảo trì các tệp XML và trình cắm Gradle tự động sinh mã Java sẵn sàng cho môi trường sản xuất. Cơ chế này không chỉ tiết kiệm công sức mà còn đảm bảo tính nhất quán - tất cả thực thể tuân theo cùng các mẫu thiết kế, quy chuẩn mã, và cách thực hành tốt nhất mà không phụ thuộc vào kỷ luật của lập trình viên.

Quá trình sinh mã được thực hiện bởi một **trình cắm Gradle** (Gradle plugin) được cấu hình trong hệ thống xây dựng (build system). Trình cắm này là một phần của khung ứng dụng lõi Axelor, chứa bộ phân tích cú pháp cây trừu tượng (AST parser) cho lược đồ XML và các mẫu mã (template) cho việc sinh thực thể Java. Quy trình xây dựng có một bước rõ ràng "generateCode" chạy trước khi biên dịch - Gradle trước tiên phân tích tất cả tệp XML từ `src/main/resources/domains/`, kiểm tra dựa trên lược đồ XSD để bắt lỗi sớm, xây dựng biểu diễn mô hình nội bộ, rồi sinh các tệp mã Java vào thư mục `build/src-gen/`. Mã được sinh sau đó được biên dịch cùng với mã viết tay thành các tệp JAR cuối cùng.

**Cấu trúc mã được sinh:**
Các thực thể được sinh đặt trong gói được chỉ định trong thuộc tính `<module package="..."/>`. Mỗi khai báo `<entity>` sinh ra HAI lớp:

1. **Lớp thực thể** - `{TênThựcThể}.java` trong gói `com.axelor.apps.{module}.db`
2. **Lớp kho lưu trữ** - `{TênThựcThể}Repository.java` trong gói `com.axelor.apps.{module}.db.repo`

Mẫu thiết kế này phân tách mô hình dữ liệu (thực thể) khỏi tầng truy cập dữ liệu (kho lưu trữ), tuân theo mẫu Kho lưu trữ (Repository Pattern) và khuyến khích phân tầng đúng cách. Lớp thực thể kế thừa `AuditableModel` (lớp cơ sở cung cấp các trường id, version, createdBy, updatedBy, createdOn, updatedOn) và chứa tất cả khai báo trường, getter/setter, triển khai equals/hashCode. Lớp kho lưu trữ chứa phương thức tìm kiếm, tiện ích DSL truy vấn, và hằng số (nếu có trong extra-code).

**Bằng chứng từ mã nguồn - Cấu trúc lớp thực thể được sinh:**
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

**Giải thích mã nguồn:** Thực thể được sinh là thực thể JPA chuẩn với tất cả chú thích cần thiết. `@Entity` đánh dấu lớp là thực thể JPA, `@Table` chỉ định tên bảng cơ sở dữ liệu (theo quy ước CHỮ_HOA). Khai báo trường ánh xạ từ định nghĩa XML: `<string name="code" unique="true" required="true">` thành `@Column(name = "code", unique = true, nullable = false) private String code;`. Trường quan hệ `currency` có chú thích `@ManyToOne` với `fetch = FetchType.LAZY` - Axelor mặc định tải lười để tối ưu hiệu năng. Các phương thức `equals()` và `hashCode()` chỉ dùng trường ID - mẫu JPA chuẩn tránh vấn đề với tập hợp được ủy nhiệm (proxied collection) và thực thể đã tách rời (detached entity).

**Bằng chứng từ mã nguồn - Lớp kho lưu trữ được sinh:**
```java
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

  public static final String PRODUCT_TYPE_SERVICE = "service";
  public static final String PRODUCT_TYPE_STORABLE = "storable";
}
```

**Giải thích mã nguồn:** Kho lưu trữ được sinh kế thừa lớp cơ sở `JpaRepository<T>` cung cấp các thao tác CRUD (lưu, tìm, xóa, tất cả). Hàm tạo gọi super với kiểu lớp thực thể. Phương thức tìm kiếm khai báo trong XML `<finder-method>` xuất hiện ở đây dưới dạng lời gọi DSL truy vấn. Hằng số từ các phần `<extra-code>` được đưa vào ở cấp lớp. Tầng dịch vụ tiêm đối tượng kho lưu trữ qua cơ chế tiêm phụ thuộc: `@Inject ProductRepository productRepo;` rồi gọi `productRepo.findByCode("PROD001")`.

---

### 10. MẪU KHO LƯU TRỮ: KIẾN TRÚC HAI TẦNG (SINH TỰ ĐỘNG + TÙY CHỈNH)

**Tệp nguồn:** `ProductRepository.java` được sinh, `ProductBaseRepository.java` tùy chỉnh [Từ mã nguồn]

Axelor triển khai một mẫu kho lưu trữ hai tầng tinh vi, phân tách mã truy cập dữ liệu được sinh tự động khỏi logic nghiệp vụ tùy chỉnh. Tầng 1 là **kho lưu trữ được sinh** chứa phương thức tìm kiếm và CRUD cơ bản từ định nghĩa XML - các tệp này được sinh lại mỗi lần xây dựng và không nên sửa thủ công. Tầng 2 là **kho lưu trữ tùy chỉnh** - các lớp viết tay kế thừa kho lưu trữ được sinh, chứa truy vấn phức tạp, kiểm tra hợp lệ nghiệp vụ, trường tính toán, và logic tích hợp. Mẫu thiết kế này giải quyết một cách tinh tế thách thức cốt lõi của sinh mã tự động: làm thế nào để bảo toàn các tùy chỉnh khi sinh lại mã.

Quy ước đặt tên phân biệt rõ hai tầng: kho lưu trữ được sinh có tên `{ThựcThể}Repository` (ví dụ: `ProductRepository`), kho lưu trữ tùy chỉnh có tên `{ThựcThể}BaseRepository` hoặc `{ThựcThể}ManagementRepository` (ví dụ: `ProductBaseRepository`). Khung tiêm phụ thuộc (dependency injection) được cấu hình để tiêm đối tượng kho lưu trữ tùy chỉnh khi mã yêu cầu giao diện kho lưu trữ - tầng dịch vụ không cần biết về sự phân biệt giữa hai tầng.

**Bằng chứng từ mã nguồn - Kho lưu trữ tùy chỉnh (Tầng 2):**
```java
// Tệp: src/main/java/.../db/repo/ProductBaseRepository.java
// VIẾT TAY - an toàn để sửa
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
    // Kiểm tra hợp lệ tùy chỉnh trước khi lưu
    if (product.getCode() == null || product.getCode().isEmpty()) {
      throw new PersistenceException("Product code is required");
    }

    // Gọi dịch vụ để tính các trường dẫn xuất
    productService.computeSalePrice(product);

    // Gọi save của lớp cha (thực hiện lưu vào CSDL)
    return super.save(product);
  }

  public Product copy(Product product, boolean deep) {
    Product copy = super.copy(product, deep);

    // Logic sao chép tùy chỉnh
    copy.setCode(null); // Buộc người dùng nhập mã mới
    copy.setStatusSelect(STATUS_DRAFT); // Đặt lại về trạng thái nháp

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

**Giải thích mã nguồn:** Kho lưu trữ tùy chỉnh nằm trong thư mục `src/main/java/` (thư mục mã viết tay) và kế thừa `ProductRepository` được sinh, thừa hưởng tất cả phương thức tìm kiếm và hằng số. Lớp có thể tiêm dịch vụ qua chú thích `@Inject` - kho lưu trữ được sinh không có phụ thuộc để giữ chúng đơn giản, nhưng kho lưu trữ tùy chỉnh có thể có đồ thị phụ thuộc phức tạp.

Phương thức `save()` được ghi đè minh họa mẫu kiểm tra hợp lệ + tính toán trường dẫn xuất. Trước khi gọi `super.save()` (thực hiện INSERT/UPDATE thực tế vào cơ sở dữ liệu), logic tùy chỉnh kiểm tra các trường bắt buộc và gọi dịch vụ để tính các giá trị phụ thuộc. Điều này đảm bảo quy tắc nghiệp vụ được áp dụng nhất quán bất kể sản phẩm được lưu bằng cách nào (giao diện web, API, tác vụ hàng loạt).

Phương thức `copy()` tùy chỉnh xử lý sao chép thực thể với logic nghiệp vụ. Sao chép đơn giản sẽ giữ nguyên mã và trạng thái, gây vi phạm ràng buộc (mã phải duy nhất) và trạng thái nghiệp vụ sai (bản sao nên bắt đầu ở trạng thái nháp). Logic tùy chỉnh đặt lại các trường này về giá trị mặc định an toàn. Phương thức `findExpiredProducts()` minh họa truy vấn phức tạp không thể biểu diễn qua phương thức tìm kiếm khai báo trong XML - yêu cầu nhiều bộ lọc, so sánh ngày, lọc trạng thái.

**Ràng buộc tiêm phụ thuộc:** [Suy luận về cấu hình Guice]
Axelor phải cấu hình bộ chứa Guice DI để tiêm kho lưu trữ tùy chỉnh khi mã yêu cầu kho lưu trữ:
```java
bind(ProductRepository.class).to(ProductBaseRepository.class);
```
Lệnh này nói với Guice: khi thành phần yêu cầu `ProductRepository`, hãy tiêm đối tượng `ProductBaseRepository` thay thế. Các dịch vụ có thể tiêm `ProductRepository` (giao diện/lớp cơ sở) mà không cần biết về triển khai tùy chỉnh - nguyên tắc đảo ngược phụ thuộc (dependency inversion principle).

**Lợi ích của mẫu hai tầng:**
1. **Phân tách mối quan tâm:** Mã được sinh chỉ truy cập dữ liệu thuần túy, mã tùy chỉnh chứa logic nghiệp vụ
2. **An toàn khi nâng cấp:** Nâng cấp khung ứng dụng sinh lại tầng 1 mà không ảnh hưởng tầng 2
3. **Nhất quán:** Tất cả kho lưu trữ có cùng cấu trúc cơ bản (tầng 1), logic tùy chỉnh là phần bổ sung
4. **Khả năng kiểm thử:** Có thể kiểm thử logic tầng 2 bằng cách giả lập (mock) thao tác tầng 1
5. **Dễ tìm kiếm:** Lập trình viên biết tìm ở đâu - truy vấn đơn giản ở kho được sinh, logic phức tạp ở kho tùy chỉnh

---

### 11. CÁC MẪU TRUY VẤN VÀ DSL TRUY VẤN AXELOR

**Tệp nguồn:** Các lớp kho lưu trữ tùy chỉnh, mã tầng dịch vụ [Từ mã nguồn kho lưu trữ]

Axelor cung cấp DSL truy vấn (Query DSL) riêng được xây dựng trên nền tảng API tiêu chí (Criteria API) của JPA, mang lại giao diện nối chuỗi liền mạch (fluent interface) để xây dựng truy vấn an toàn về kiểu dữ liệu (type-safe). DSL này trừu tượng hóa API tiêu chí dài dòng của Hibernate và các truy vấn chuỗi JPQL, cung cấp cú pháp thân thiện với lập trình viên, đồng thời đảm bảo an toàn tại thời điểm biên dịch. Mẫu chung là `Query.of(LớpThựcThể.class).filter(...).bind(...).fetch()` - phong cách khai báo giảm thiểu mã khuôn mẫu và giảm nguy cơ tấn công chèn SQL (SQL injection) vì tất cả tham số được thoát ký tự (escape) đúng cách.

DSL truy vấn có một số tính năng đặc biệt: chuỗi bộ lọc dùng bí danh `self` tham chiếu đến thực thể được truy vấn (mượn từ quy ước JPQL), tham số đặt tên (`:tenThamSo`) thay vì tham số vị trí (`?1`), nối chuỗi phương thức (method chaining) cho khả năng tổ hợp, và hỗ trợ phân trang, sắp xếp, và chế độ tải dữ liệu. Mẫu xây dựng (builder pattern) cho phép truy vấn được xây dựng dần dần - có thể truyền đối tượng truy vấn giữa các phương thức, thêm bộ lọc có điều kiện, tái sử dụng truy vấn gốc với các tham số khác nhau.

**Bằng chứng từ mã nguồn - Các mẫu DSL truy vấn cơ bản:**
```java
// Truy vấn đơn giản với một bộ lọc
Product product = Query.of(Product.class)
  .filter("self.code = :code")
  .bind("code", "PROD001")
  .fetchOne();

// Nhiều bộ lọc (logic AND)
List<Product> products = Query.of(Product.class)
  .filter("self.productCategory = :category")
  .filter("self.salePrice > :minPrice")
  .bind("category", category)
  .bind("minPrice", 100.0)
  .fetch();

// Sắp xếp và phân trang
List<Product> products = Query.of(Product.class)
  .filter("self.statusSelect = :status")
  .bind("status", ProductRepository.STATUS_ACTIVE)
  .order("-createdOn") // Giảm dần (dấu trừ phía trước)
  .fetchLimit(20, 0); // Giới hạn 20, bắt đầu từ 0
```

**Giải thích mã nguồn:** Việc xây dựng truy vấn bắt đầu với `Query.of(Product.class)` thiết lập kiểu thực thể (đảm bảo an toàn kiểu). Phương thức filter thêm mệnh đề WHERE - nhiều lời gọi filter được nối bằng AND (không cần từ khóa AND rõ ràng). Chuỗi `"self.code = :code"` là đoạn JPQL trong đó `self` tham chiếu đến thực thể Product và `:code` là chỗ giữ chỗ (placeholder) cho tham số đặt tên.

Phương thức bind cung cấp giá trị tham số - tên phải khớp với chỗ giữ chỗ. Tham số đặt tên ưu việt hơn tham số vị trí vì: (1) dễ đọc - `bind("code", value)` rõ ràng hơn `bind(1, value)`, (2) an toàn khi tái cấu trúc - có thể sắp xếp lại bộ lọc mà không làm hỏng vị trí tham số, (3) tái sử dụng - có thể ràng buộc cùng tham số nhiều lần trong truy vấn phức tạp.

Phương thức order với dấu trừ (`"-createdOn"`) chỉ định sắp xếp giảm dần - quy ước mượn từ Django ORM. Không có dấu trừ thì mặc định tăng dần. Phương thức `fetchLimit(limit, offset)` triển khai phân trang - rất quan trọng cho tập kết quả lớn, ngăn tải hàng nghìn bản ghi vào bộ nhớ. Các tham số ánh xạ tới mệnh đề `LIMIT` và `OFFSET` trong SQL.

**Truy vấn phức tạp với phép nối và truy vấn con:**
```java
// Truy vấn nối rõ ràng
List<SaleOrderLine> lines = Query.of(SaleOrderLine.class)
  .filter("self.saleOrder.company = :company")
  .filter("self.saleOrder.statusSelect = :status")
  .bind("company", company)
  .bind("status", SaleOrder.STATUS_CONFIRMED)
  .fetch();

// Truy vấn đếm
long count = Query.of(Product.class)
  .filter("self.productCategory = :category")
  .bind("category", category)
  .count();
```

**Giải thích mã nguồn:** Bộ lọc `"self.saleOrder.company = :company"` minh họa biểu thức đường dẫn (path expression) - DSL tự động thực hiện phép nối khi duyệt qua quan hệ. Bên trong, sinh câu SQL JOIN giữa bảng sale_order_line và sale_order. Lập trình viên không cần khai báo phép nối rõ ràng - cú pháp gọn gàng hơn, mặc dù ít quyền kiểm soát hơn về loại phép nối (LEFT hay INNER) và chiến lược tải dữ liệu.

**Xây dựng truy vấn động:**
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

**Giải thích mã nguồn:** Đối tượng Query có thể thay đổi - mỗi lời gọi filter/bind trả về cùng hoặc đối tượng mới, cho phép xây dựng dần dần. Mẫu thiết kế này rất hiệu quả cho biểu mẫu tìm kiếm với bộ lọc tùy chọn - chỉ thêm bộ lọc khi tham số được cung cấp, tránh phải nối chuỗi SQL có điều kiện phức tạp. Mã sạch và dễ bảo trì hơn so với xây dựng chuỗi JPQL: `String jpql = "SELECT p FROM Product p WHERE 1=1"; if (category != null) jpql += " AND p.category = :category"; ...` (cách làm xấu cần tránh).

**Hạn chế của DSL truy vấn:** [Suy luận về các tính năng thiếu]
DSL truy vấn đáp ứng các trường hợp phổ biến nhưng các tình huống phức tạp có thể cần JPQL thuần hoặc SQL gốc (native SQL):
- **Phép nối phức tạp:** Không thể chỉ định loại phép nối (LEFT JOIN, RIGHT JOIN, OUTER JOIN)
- **Phép tổng hợp:** Không hỗ trợ tích hợp cho GROUP BY, HAVING, các hàm tổng hợp (SUM, AVG, MAX)
- **Truy vấn con:** Không thể nhúng truy vấn con trong bộ lọc (ví dụ: WHERE id IN (SELECT ...))
- **Truy vấn hợp nhất:** Không thể UNION nhiều truy vấn
- **Phép chiếu tùy chỉnh:** Luôn lấy toàn bộ thực thể, không thể SELECT các cột cụ thể

Cho các tình huống này, Axelor cho phép thực thi JPQL thuần hoặc SQL gốc:
```java
String sql = "SELECT * FROM product WHERE code ~* :regex"; // Biểu thức chính quy PostgreSQL
Query query = JPA.em().createNativeQuery(sql, Product.class);
query.setParameter("regex", "^PROD-.*");
List<Product> results = query.getResultList();
```

Đánh đổi: truy vấn thuần mạnh mẽ hơn nhưng mất tính an toàn kiểu, dài dòng hơn, và phụ thuộc cơ sở dữ liệu cụ thể (vấn đề khả năng di chuyển).

---

### 12. TRƯỜNG JSON VÀ CƠ CHẾ TRƯỜNG TÙY CHỈNH

**Tệp nguồn:** Partner.xml, SaleOrder.xml, phân tích tích hợp Axelor Studio [Từ mã nguồn và suy luận]

Trường JSON trong Axelor phục vụ một trường hợp sử dụng rất cụ thể: cho phép người dùng nghiệp vụ thêm trường tùy chỉnh vào thực thể thông qua Axelor Studio (công cụ không cần viết mã) mà không cần sự can thiệp của lập trình viên và không cần di trú cơ sở dữ liệu (database migration). Đây là sự căng thẳng kinh điển trong phần mềm doanh nghiệp - cân bằng giữa cấu trúc (lược đồ cứng đảm bảo toàn vẹn dữ liệu) và tính linh hoạt (người dùng nghiệp vụ muốn tùy chỉnh mà không cần chờ bộ phận kỹ thuật). Giải pháp của Axelor là kết hợp: các trường lõi có kiểu mạnh trong lược đồ, các trường tùy chỉnh có kiểu lỏng trong khối JSON.

Cơ chế hoạt động như sau: định nghĩa thực thể bao gồm một trường chuỗi với thuộc tính `json="true"`, ví dụ `<string name="attrs" json="true"/>`. Kiểu cột cơ sở dữ liệu vẫn là TEXT hoặc VARCHAR (tùy cơ sở dữ liệu), nhưng khung ứng dụng Axelor chặn các phương thức getter/setter để nối tiếp hóa/giải nối tiếp hóa JSON. Mã ứng dụng không làm việc với chuỗi JSON thô - thay vào đó làm việc với kiểu trừu tượng `Map<String, Object>`. Giao diện Axelor Studio cho phép người dùng nghiệp vụ định nghĩa trường tùy chỉnh (tên, kiểu, nhãn, giá trị mặc định, quy tắc kiểm tra) và lưu siêu dữ liệu (metadata) trong bảng `meta_json_field`, trong khi giá trị thực tế của trường được lưu trong cột JSON của bản ghi thực thể.

**Bằng chứng từ mã nguồn - Khai báo trường JSON:**
```xml
<!-- Trong Partner.xml -->
<string name="attrs" title="Custom attributes" json="true"/>

<!-- Trong SaleOrder.xml -->
<string name="partnerAttrs" title="Fields" json="true"/>
```

**Sử dụng trường JSON trong mã thực thi:**
```java
// Đọc trường JSON (giải nối tiếp hóa thành Map)
Product product = productRepo.find(123L);
Map<String, Object> attrs = product.getAttrs();

String customField1 = (String) attrs.get("customField1");
Boolean isSpecial = (Boolean) attrs.get("specialProduct");
Integer loyaltyPoints = (Integer) attrs.get("loyaltyPoints");

// Ghi trường JSON
attrs.put("customField1", "giá trị mới");
attrs.put("newCustomField", 42);
product.setAttrs(attrs);
productRepo.save(product);
```

**Giải thích mã nguồn:** Khung ứng dụng cung cấp giao diện Map tiện lợi che giấu sự phức tạp của JSON. Lập trình viên không cần gọi thủ công thư viện JSON - getter trả về Map, setter nhận Map. Nhược điểm: không có tính an toàn kiểu - phải ép kiểu (`(String) attrs.get(...)`) nên có nguy cơ lỗi ClassCastException lúc chạy nếu kiểu sai.

**Định dạng lưu trữ trong cơ sở dữ liệu:**
```sql
-- Dữ liệu mẫu từ bảng partner
SELECT id, name, attrs FROM partner WHERE id = 123;

-- Kết quả:
-- id  | name          | attrs
-- 123 | Acme Corp     | {"customField1": "value", "vatExempt": true, "loyaltyPoints": 1500}
```

**Giải thích:** JSON được lưu dưới dạng chuỗi văn bản trong cơ sở dữ liệu. Các cơ sở dữ liệu hiện đại (PostgreSQL 9.2+, MySQL 5.7+) có kiểu JSON gốc hỗ trợ đánh chỉ mục và truy vấn, nhưng Axelor sử dụng cột TEXT để đảm bảo khả năng di chuyển giữa các hệ quản trị. Hệ quả: không thể truy vấn hiệu quả các trường tùy chỉnh.

**Lưu trữ siêu dữ liệu trong MetaJsonField:**
```sql
-- Cấu trúc bảng meta_json_field (suy luận)
CREATE TABLE meta_json_field (
  id BIGINT PRIMARY KEY,
  model VARCHAR(255),       -- Lớp thực thể đích (com.axelor.apps.base.db.Partner)
  model_field VARCHAR(255),  -- Tên trường JSON (attrs)
  name VARCHAR(255),         -- Tên trường tùy chỉnh (loyaltyPoints)
  type VARCHAR(50),          -- Kiểu trường (integer, string, boolean, decimal, date)
  title VARCHAR(255),        -- Nhãn hiển thị (Điểm thưởng)
  default_value TEXT,        -- Giá trị mặc định cho bản ghi mới
  required BOOLEAN,          -- Kiểm tra hợp lệ: trường có bắt buộc?
  ...
);
```

**Giải thích:** Bảng siêu dữ liệu mô tả cấu trúc của các trường tùy chỉnh. Khi người dùng Axelor Studio tạo trường tùy chỉnh "Điểm thưởng" (kiểu số nguyên) trên thực thể Partner, một dòng được chèn vào bảng này. Mã hiển thị giao diện truy vấn siêu dữ liệu để biết hiển thị trường tùy chỉnh nào trong biểu mẫu, kiểu dữ liệu của chúng để chọn thành phần giao diện phù hợp (ô nhập văn bản, hộp chọn, bộ chọn ngày), và quy tắc kiểm tra cần áp dụng.

**Lợi ích của cách tiếp cận trường JSON:**
1. **Tùy chỉnh không cần dừng hệ thống:** Thêm trường mà không cần ALTER TABLE, không khóa cơ sở dữ liệu, không cần triển khai lại
2. **Thân thiện với đa thuê bao (multi-tenancy):** Các thuê bao khác nhau có thể có trường tùy chỉnh khác nhau trong cùng cơ sở dữ liệu
3. **Tạo mẫu nhanh:** Người dùng nghiệp vụ có thể thử nghiệm ý tưởng nhanh chóng, xóa trường dễ dàng nếu không hữu ích

**Nhược điểm:**
1. **Hiệu năng truy vấn:** Không thể đánh chỉ mục trường tùy chỉnh, lọc/sắp xếp yêu cầu quét toàn bộ bảng
2. **Toàn vẹn dữ liệu:** Không có ràng buộc khóa ngoại, ràng buộc kiểm tra, hoặc kiểm tra kiểu ở cấp cơ sở dữ liệu
3. **Thách thức báo cáo:** Các công cụ phân tích nghiệp vụ (BI) và truy vấn SQL báo cáo không dễ truy cập trường JSON
4. **An toàn kiểu:** Lỗi kiểu xảy ra lúc chạy, không có kiểm tra tại thời điểm biên dịch
5. **Quan hệ phức tạp:** Không thể định nghĩa quan hệ nhiều-một, một-nhiều với trường tùy chỉnh (chỉ hỗ trợ kiểu nguyên thủy và chuỗi)

---

### 13. CHIẾN LƯỢC DDL CỦA HIBERNATE VÀ QUẢN LÝ LƯỢC ĐỒ CƠ SỞ DỮ LIỆU

**Tệp nguồn:** `src/main/resources/axelor-config.properties` [Từ mã nguồn]

Quản lý lược đồ cơ sở dữ liệu (database schema management) là một trong những quyết định quan trọng nhất trong kiến trúc ứng dụng - cách xử lý việc phát triển lược đồ (thêm bảng, thay đổi cột, di trú dữ liệu) qua các môi trường phát triển, kiểm thử, dàn dựng (staging), và sản xuất (production). Axelor áp dụng một cách tiếp cận khác biệt so với hầu hết ứng dụng Java hiện đại: **sinh DDL tự động của Hibernate** thay vì công cụ di trú như Flyway hoặc Liquibase. Cấu hình này được tìm thấy trong tệp axelor-config.properties với khóa `hibernate.hbm2ddl.auto`.

**Bằng chứng từ mã nguồn - Cấu hình DDL Hibernate:**
```properties
# Cài đặt cơ sở dữ liệu
db.default.driver = org.postgresql.Driver
db.default.ddl = update
db.default.url = jdbc:postgresql://localhost:5432/axelor_erp_db
db.default.user = axelor
db.default.password = axelor

# Cài đặt Hibernate
hibernate.hbm2ddl.auto = update
hibernate.show_sql = false
hibernate.format_sql = true
```

**Giải thích mã nguồn:** Thuộc tính `hibernate.hbm2ddl.auto = update` chỉ thị Hibernate tự động đồng bộ lược đồ cơ sở dữ liệu với các định nghĩa thực thể JPA mỗi khi ứng dụng khởi động. Giá trị "update" nghĩa là: (1) kiểm tra lược đồ hiện tại, (2) so sánh với ánh xạ thực thể, (3) thực thi câu lệnh ALTER TABLE để thêm bảng/cột thiếu, (4) KHÔNG BAO GIỜ xóa bảng/cột hiện có. Cài đặt `hibernate.show_sql = false` tắt ghi nhật ký câu lệnh SQL ra bảng điều khiển (sẽ cực kỳ dài dòng với hàng nghìn truy vấn).

**Các tùy chọn hibernate.hbm2ddl.auto và ý nghĩa:**

| Giá trị | Hành vi | Trường hợp sử dụng | Rủi ro |
|---------|---------|---------------------|--------|
| `create` | XÓA tất cả bảng, rồi TẠO từ đầu | Phát triển cục bộ, kiểm thử tự động | **MẤT TOÀN BỘ DỮ LIỆU** - không bao giờ dùng trên production |
| `create-drop` | TẠO khi khởi động, XÓA khi tắt | Kiểm thử tích hợp (bắt đầu sạch mỗi lần) | **MẤT TOÀN BỘ DỮ LIỆU** - không bao giờ dùng trên production |
| `update` | SỬA bảng cho khớp thực thể, không bao giờ XÓA | Phát triển, môi trường dàn dựng | Trôi lược đồ, không hoàn tác, không đổi tên cột |
| `validate` | KIỂM TRA lược đồ khớp thực thể, ném ngoại lệ nếu không | Production (sau khi di trú thủ công) | Ứng dụng không khởi động được nếu lược đồ không khớp |
| `none` | Không làm gì, giả định lược đồ đã đúng | Production (với công cụ di trú) | Lập trình viên phải quản lý lược đồ thủ công |

**Tại sao Axelor chọn chiến lược "update":** [Suy luận về lý do thiết kế]

Axelor nhắm đến ứng dụng nghiệp vụ nơi lược đồ phát triển thường xuyên - thêm trường tùy chỉnh (qua Studio), cài đặt mô-đun mới (với thực thể mới), nâng cấp phiên bản khung ứng dụng (bảng lõi mới). Chế độ "update" mang lại sự tiện lợi: lập trình viên sửa XML, chạy ứng dụng, lược đồ tự động cập nhật - không cần viết tập lệnh DDL thủ công. Cho môi trường phát triển và dàn dựng, đây là cải thiện năng suất đáng kể.

Tuy nhiên, "update" có những hạn chế nghiêm trọng cho môi trường sản xuất:

1. **Không thể hoàn tác:** Không thể quay lại thay đổi lược đồ nếu triển khai thất bại
2. **Không thể đổi tên cột:** Hibernate thấy cột cũ thiếu + cột mới thiếu → thêm cột mới, giữ cột cũ (dữ liệu trùng lặp)
3. **Không thể thay đổi kiểu cột:** Câu lệnh ALTER COLUMN TYPE nguy hiểm (nguy cơ mất dữ liệu), Hibernate thận trọng không thử
4. **Không di trú dữ liệu:** Lược đồ thay đổi nhưng dữ liệu hiện có không được chuyển đổi
5. **Không phát hiện trường bị xóa:** Hibernate chỉ thêm, không bao giờ xóa - chế độ "update" không bao giờ xóa cột ngay cả khi đã bị xóa khỏi thực thể

**Cách thực hành tốt cho môi trường sản xuất:** [Suy luận từ tiêu chuẩn ngành]
Môi trường sản xuất nên dùng `hibernate.hbm2ddl.auto = validate` hoặc `none` kết hợp với công cụ di trú như Flyway hoặc Liquibase. Các công cụ di trú cung cấp: quản lý phiên bản cho thay đổi lược đồ (mỗi tập lệnh được đánh số/đặt tên), khả năng hoàn tác (có thể quay lại di trú theo cách có kiểm soát), chuyển đổi dữ liệu (câu lệnh UPDATE di chuyển dữ liệu giữa cột cũ/mới), nhất quán môi trường (cùng tập lệnh di trú áp dụng cho phát triển/kiểm thử/dàn dựng/sản xuất), và dấu vết kiểm toán (bảng lịch sử di trú cho biết tập lệnh nào đã chạy khi nào). Tuy nhiên, mã nguồn Axelor **không bao gồm** Flyway hay Liquibase.

---

### 14. VỊ TRÍ MÃ ĐƯỢC SINH VÀ TÍCH HỢP VỚI HỆ THỐNG XÂY DỰNG

**Tệp nguồn:** Các tập lệnh xây dựng Gradle, phân tích thư mục đầu ra xây dựng [Từ mã nguồn]

Hiểu rõ mã được sinh nằm ở đâu và cách tích hợp vào quy trình xây dựng rất quan trọng cho việc gỡ lỗi, quản lý phiên bản, và cộng tác. Axelor tuân theo quy ước Gradle với một số tùy chỉnh: mã được sinh đặt trong thư mục `build/` (bị bỏ qua bởi git, tạm thời), phân tách rõ ràng khỏi mã viết tay trong thư mục `src/` (được quản lý phiên bản, lâu dài).

**Mẫu cấu trúc thư mục:**
```
modules/axelor-open-suite/axelor-base/
├── src/
│   ├── main/
│   │   ├── java/              # Mã viết tay
│   │   │   └── com/axelor/apps/base/
│   │   │       ├── service/   # Dịch vụ logic nghiệp vụ
│   │   │       ├── web/       # Bộ điều khiển
│   │   │       └── db/repo/   # Kho lưu trữ tùy chỉnh (Tầng 2)
│   │   └── resources/
│   │       ├── domains/       # Tệp XML định nghĩa (nguồn cho sinh mã)
│   │       └── views/         # Tệp XML giao diện
│   └── test/                  # Kiểm thử đơn vị
└── build/
    ├── src-gen/
    │   └── java/              # Mã được sinh (KHÔNG SỬA)
    │       └── com/axelor/apps/base/db/
    │           ├── Product.java              # Thực thể được sinh
    │           ├── Company.java
    │           └── repo/
    │               ├── ProductRepository.java    # Kho sinh (Tầng 1)
    │               └── CompanyRepository.java
    ├── classes/               # Tệp .class đã biên dịch
    └── libs/                  # Tệp JAR đầu ra
```

**Giải thích cấu trúc:** Sự phân tách rõ ràng giữ mọi thứ có trật tự: lập trình viên làm việc trong `src/`, bộ sinh xuất ra `build/`. Cấu hình IDE phải bao gồm cả hai tập nguồn: tập nguồn chính (`src/main/java`) + tập nguồn được sinh (`build/src-gen/java`) - nếu không, các lớp được sinh sẽ không được nhận diện trong quá trình biên dịch. Hệ thống quản lý phiên bản bỏ qua toàn bộ thư mục `build/` - mã được sinh không bao giờ được commit (sẽ gây xung đột hợp nhất, lãng phí dung lượng kho, gây nhầm lẫn về nguồn sự thật).

Phụ thuộc `compileJava.dependsOn generateCode` đảm bảo việc sinh mã luôn chạy trước khi biên dịch - rất quan trọng vì biên dịch mã viết tay (dịch vụ, bộ điều khiển) tham chiếu đến thực thể được sinh, biên dịch sẽ thất bại nếu thực thể chưa được sinh. Thứ tự xây dựng: clean → generateCode → compileJava → processResources → classes → jar.

**Cạm bẫy phổ biến:** Lập trình viên sửa trực tiếp lớp được sinh (tìm lỗi, sửa nhanh) quên rằng các sửa đổi sẽ bị mất lần xây dựng tiếp theo. Cách phòng ngừa: bộ sinh thêm chú thích đầu tệp `// TỰ ĐỘNG SINH - KHÔNG SỬA`, và quy trình rà soát mã (code review) nên phát hiện các thay đổi trong thư mục `build/`.

---

### 15. NHỮNG ĐIỀU KHÔNG TÌM THẤY TRONG MÃ NGUỒN

Sau quá trình phân tích sâu các tệp XML, mã được sinh, và cấu hình, một số tính năng cơ sở dữ liệu/ORM phổ biến trong ứng dụng doanh nghiệp **KHÔNG** xuất hiện trong mã nguồn Axelor. Việc ghi nhận những "tính năng vắng mặt" quan trọng vì: (1) giúp đặt kỳ vọng đúng về khả năng nền tảng, (2) xác định các khoảng trống có thể cần giải pháp thay thế, (3) hiểu triết lý kiến trúc (những gì Axelor cố ý không đưa vào).

**1. Công cụ di trú cơ sở dữ liệu (Flyway, Liquibase)** [Không tìm thấy]
Không có phụ thuộc hoặc cấu hình cho Flyway (`org.flywaydb`) hoặc Liquibase (`org.liquibase`) trong tệp xây dựng. Không có thư mục tập lệnh di trú (`db/migration/`, `liquibase/changelogs/`). Axelor dựa hoàn toàn vào DDL tự động của Hibernate, như đã xác nhận từ cấu hình `hibernate.hbm2ddl.auto=update`.

**2. Hỗ trợ nhiều nhà cung cấp cơ sở dữ liệu** [Không rõ ràng]
Tệp cấu hình chỉ hiển thị PostgreSQL (`org.postgresql.Driver`, URL JDBC dành riêng cho PostgreSQL). Không tìm thấy hồ sơ (profile) hoặc cấu hình cho MySQL, Oracle, SQL Server. Tệp XML trừu tượng hóa sự khác biệt giữa các cơ sở dữ liệu (không có kiểu dữ liệu riêng cho từng nhà cung cấp), gợi ý hỗ trợ đa cơ sở dữ liệu khả thi về mặt kỹ thuật, nhưng không có bằng chứng về kiểm thử/chứng nhận với cơ sở dữ liệu khác.

**3. Phân mảnh cơ sở dữ liệu (Database sharding)** [Không tìm thấy]
Không có cấu hình cho phân mảnh (chia dữ liệu qua nhiều cơ sở dữ liệu). Định nghĩa nguồn dữ liệu duy nhất (`db.default.*`) gợi ý một thể hiện cơ sở dữ liệu duy nhất.

**4. Xóa mềm (Soft delete)** [Không tìm thấy trong tệp XML]
Không có trường `deleted` hoặc `archived` dạng boolean phổ quát trong lớp thực thể cơ sở. Không có chú thích `@Where(clause = "deleted = false")` trong mã được sinh. Một số thực thể có thể triển khai xóa mềm thủ công với trường trạng thái (`STATUS_ARCHIVED = 9`) nhưng không có hỗ trợ ở cấp khung ứng dụng.

**5. Khóa lạc quan với phiên bản (Optimistic locking)** [Có thể có nhưng không rõ]
Lớp cơ sở `AuditableModel` có trường `version` gợi ý hỗ trợ khóa lạc quan (chú thích JPA @Version). Nhưng không thấy xử lý phiên bản rõ ràng trong tệp XML hoặc mã kho lưu trữ.

**6. Cấu hình nhóm kết nối cơ sở dữ liệu (Connection pooling)** [Có nhưng tối thiểu]
Tệp cấu hình đề cập HikariCP (nhóm kết nối tiêu chuẩn ngành) nhưng không có tham số tinh chỉnh: kích thước nhóm, thời gian chờ, truy vấn kiểm tra, phát hiện rò rỉ. Sử dụng giá trị mặc định (có thể tối đa 10 kết nối).

**7. Bản sao đọc (Read replica)** [Không tìm thấy]
Định nghĩa nguồn dữ liệu duy nhất, không có phân tách đọc/ghi. Tất cả truy vấn đều truy cập cơ sở dữ liệu chính.

**8. Khóa chính tổ hợp (Composite primary key)** [Không tìm thấy]
Tất cả thực thể dùng khóa chính thay thế (surrogate) duy nhất (`id BIGINT`). Không có mẫu `@IdClass` hoặc `@EmbeddedId`.

**9. Chiến lược kế thừa (Inheritance strategy)** [Không rõ]
JPA hỗ trợ ba chiến lược kế thừa: SINGLE_TABLE, JOINED, TABLE_PER_CLASS. Tệp XML của Axelor không cung cấp cấu hình kế thừa - không thấy phần tử `<inheritance>` hoặc cài đặt bộ phân biệt (discriminator).

**10. Thủ tục lưu sẵn (Stored procedure)** [Không tìm thấy]
Không có chú thích `@NamedStoredProcedureQuery` hoặc tương đương XML. Logic nghiệp vụ hoàn toàn ở tầng Java/Groovy, không có logic ở tầng cơ sở dữ liệu. Triết lý của Axelor rõ ràng ưu tiên logic tầng ứng dụng vì lý do khả năng di chuyển và kiểm thử.

---

### 16. CÂU HỎI MỞ VÀ ĐIỂM CẦN NGHIÊN CỨU THÊM

Phân tích tệp XML và mã được sinh cho bức tranh toàn diện về kiến trúc cơ sở dữ liệu của Axelor, nhưng một số câu hỏi vẫn cần điều tra sâu hơn hoặc xem tài liệu:

1. **Thứ tự thực thi hồi gọi vòng đời thực thể:** Khi thực thể có nhiều hồi gọi (@PrePersist, @PreUpdate, @PostLoad) cộng bộ lắng nghe thực thể cộng theo dõi kiểm toán, thứ tự thực thi chính xác là gì?

2. **Quản lý giao dịch và mức cô lập:** Không thấy cấu hình giao dịch rõ ràng. Mức cô lập mặc định (READ_COMMITTED hay REPEATABLE_READ)? Giao dịch được phân ranh giới thế nào (cấp phương thức @Transactional, lập trình, khai báo)?

3. **Chi tiết chiến lược bộ đệm:** Cấu hình cho thấy chế độ bộ đệm chia sẻ (shared cache) nhưng không thấy cấu hình nhà cung cấp bộ đệm. Triển khai bộ đệm nào (EHCache, Infinispan, Redis)? Chính sách đào thải bộ đệm?

4. **Chiến lược tải lười so với tải ngay:** Thực thể được sinh dùng `@ManyToOne(fetch = FetchType.LAZY)` mặc định, nhưng có tình huống nào cấu hình tải ngay? Khung ứng dụng có cung cấp mẫu OpenSessionInView không?

5. **Xử lý hàng loạt:** Mã nguồn cho thấy thao tác CRUD từng thực thể, nhưng Axelor xử lý chèn/cập nhật hàng loạt (hàng nghìn bản ghi) thế nào?

6. **Triển khai đa thuê bao (Multi-tenancy):** Cấu hình và mã gợi ý hỗ trợ đa công ty, nhưng triển khai kỹ thuật chưa rõ. Bảo mật cấp dòng (lọc theo company_id) hay lược đồ riêng theo thuê bao?

---

## TÓM TẮT KIẾN TRÚC CƠ SỞ DỮ LIỆU VÀ MÔ HÌNH DỮ LIỆU

Sau quá trình phân tích chi tiết từ mã nguồn, có thể tóm lược kiến trúc cơ sở dữ liệu và mô hình dữ liệu của Axelor qua những điểm chính sau:

### Mô hình phát triển: Phát triển hướng mô hình (Model-Driven Development)

Axelor áp dụng cách tiếp cận phát triển dựa trên XML - tệp XML định nghĩa thực thể là nguồn sự thật duy nhất, trình cắm Gradle sinh tự động thực thể JPA và kho lưu trữ. Mô hình này chuyển nỗ lực từ việc viết mã khuôn mẫu sang mô hình hóa khai báo, cải thiện đáng kể năng suất và tính nhất quán.

### Định nghĩa thực thể và Sinh mã

Tệp XML (`domains/*.xml`) định nghĩa thực thể với siêu dữ liệu phong phú: trường (kiểu, ràng buộc, hành vi), quan hệ (nhiều-một, một-nhiều, nhiều-nhiều, một-một), trường tính toán (tạm thời, công thức), theo dõi (nhật ký kiểm toán), bộ lắng nghe (móc nối vòng đời), phương thức tìm kiếm (truy vấn khai báo). Bộ sinh tạo mẫu kho lưu trữ hai tầng: kho sinh (tầng 1) với CRUD cơ bản + phương thức tìm, kho tùy chỉnh (tầng 2) với logic nghiệp vụ.

### Quản lý lược đồ cơ sở dữ liệu

DDL tự động Hibernate (`hibernate.hbm2ddl.auto=update`) quản lý lược đồ - tiện lợi cho phát triển (đồng bộ tự động) nhưng rủi ro cho sản xuất (không hoàn tác, không đổi tên cột, không di trú dữ liệu). Môi trường sản xuất nên dùng chế độ `validate` với công cụ di trú (Flyway/Liquibase), mặc dù mã nguồn Axelor không bao gồm các công cụ này. PostgreSQL là cơ sở dữ liệu chính.

### Mẫu truy vấn

DSL truy vấn Axelor cung cấp giao diện nối chuỗi bọc API tiêu chí Hibernate: `Query.of(ThựcThể.class).filter().bind().fetch()`. Hỗ trợ tham số đặt tên, biểu thức đường dẫn (nối tự động), sắp xếp, phân trang. Đáp ứng các trường hợp phổ biến (truy vấn bằng, lọc cơ bản) nhưng tình huống phức tạp (tổng hợp, truy vấn con, hợp nhất) cần JPQL/SQL thuần.

### Triết lý kiến trúc

Axelor ưu tiên **năng suất lập trình viên** (ít mã khuôn mẫu, sinh tự động, cấu hình khai báo) và **trao quyền người dùng nghiệp vụ** (trường tùy chỉnh, công cụ không cần viết mã) hơn là **tối ưu hiệu năng cơ sở dữ liệu** (kiểm soát chỉ mục hạn chế, không phân mảnh, không bản sao đọc) và **tính năng ORM nâng cao** (không chiến lược kế thừa, không khóa tổ hợp, không xóa mềm). Phù hợp cho triển khai vừa và nhỏ (hàng trăm người dùng, hàng triệu bản ghi) nơi tốc độ phát triển quan trọng hơn khả năng mở rộng cực đại.

---

**Tổng số dòng mã đã phân tích:**
- Tệp XML định nghĩa: ~15 tệp, ~3.000 dòng tổng cộng
- Mã thực thể được sinh: ~10 tệp đã xem xét, mẫu đại diện
- Tệp cấu hình: axelor-config.properties, build.gradle

**Nguồn:** Tất cả phát hiện từ phân tích trực tiếp mã nguồn, bổ sung bằng suy luận dựa trên hành vi chuẩn của JPA/Hibernate và các mẫu ứng dụng doanh nghiệp phổ biến.

---

*Kết thúc RESEARCH_STEP2_DATABASE.md*
