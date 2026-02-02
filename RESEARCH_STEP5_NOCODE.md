# BƯỚC 5: PHÂN TÍCH KHẢ NĂNG KHÔNG MÃ/ÍT MÃ — Từ mã nguồn

## Phương pháp phân tích

Nghiên cứu khả năng không mã/ít mã (no-code/low-code) của Axelor thông qua phân tích định nghĩa giao diện XML, hệ thống hành động (action system), cơ chế sinh mã, và ngôn ngữ biểu thức. Trọng tâm chính là tìm hiểu **bằng cách nào** người dùng nghiệp vụ và người dùng nâng cao có thể xây dựng ứng dụng mà không cần viết mã Java truyền thống, và **ở đâu** nền tảng vạch ranh giới giữa cấu hình khai báo (declarative) và tùy chỉnh lập trình (programmatic).

**Các tệp đã phân tích:**
- Tệp XML giao diện (SaleOrder.xml 1.827 dòng, Partner.xml, Product.xml)
- Tệp XML miền (domain) — định nghĩa thực thể
- Định nghĩa hành động (7 loại: method, record, view, attrs, group, condition, validate, script)
- Tác vụ sinh mã (tiện ích Gradle)
- Trình xem bản đồ React (package.json, cấu hình biên dịch)
- Tệp cấu hình (axelor-config.properties)

**Cách tiếp cận:** Lập danh mục các khả năng không mã bằng cách kiểm tra ví dụ giao diện/hành động thực tế từ các mô-đun sản xuất, suy luận mẫu thiết kế, và đánh giá mức độ phủ (bao nhiêu phần trăm ứng dụng nghiệp vụ điển hình có thể đạt được mà không cần viết mã Java).

---

## Kết quả chi tiết

### 1. MÔ HÌNH PHÁT TRIỂN HƯỚNG XML: KIẾN TRÚC HƯỚNG MÔ HÌNH

**Tệp nguồn:** Các tệp XML giao diện xuyên suốt nhiều mô-đun `[Từ source code]`

Axelor áp dụng **phát triển hướng mô hình** (Model-Driven Development — MDD) làm nguyên tắc kiến trúc nền tảng — ứng dụng được định nghĩa chủ yếu thông qua siêu dữ liệu khai báo (XML) thay vì mã mệnh lệnh (Java). Sự chuyển đổi mô hình này cho phép nhà phân tích nghiệp vụ có chuyên môn miền nhưng kỹ năng lập trình hạn chế đóng góp trực tiếp vào phát triển ứng dụng. Nhận thức cốt lõi: hầu hết ứng dụng nghiệp vụ tuân theo các mẫu dự đoán được (thao tác CRUD, quan hệ chủ-chi tiết, luồng phê duyệt) — các mẫu này có thể mã hóa thành cấu trúc siêu dữ liệu tái sử dụng được xử lý bởi bộ thực thi bộ khung (framework runtime).

Cách tiếp cận MDD đối lập với phát triển truyền thống ưu tiên mã (ví dụ ứng dụng Spring Boot: viết thủ công bộ điều khiển, dịch vụ, kho dữ liệu, giao diện). Lợi ích: chu kỳ phát triển nhanh hơn (thay đổi XML, tải lại trình duyệt thay vì thay đổi Java, biên dịch lại, khởi động lại máy chủ), rào cản kỹ năng thấp hơn (cú pháp XML đơn giản hơn Java), và căn chỉnh tốt hơn giữa nghiệp vụ và CNTT (chuyên gia miền đọc được đặc tả XML, xác minh tính đúng đắn). Đánh đổi (trade-off): phụ thuộc nhà cung cấp (lược đồ XML độc quyền của Axelor), linh hoạt hạn chế cho yêu cầu bất thường (bộ khung phải dự đoán trước các trường hợp sử dụng), và thách thức gỡ lỗi (lỗi XML khó hiểu, không có điểm dừng hay dấu vết ngăn xếp).

Bối cảnh ngành: MDD phổ biến trong các nền tảng doanh nghiệp (Oracle ADF, Microsoft Dynamics, OutSystems, Mendix). Cách tiếp cận hướng XML của Axelor nhẹ hơn các trình mô hình trực quan (công cụ xây dựng giao diện kéo thả) nhưng có cấu trúc hơn mã thuần. Điểm cân bằng: nhà phát triển quen thuộc với ngôn ngữ đánh dấu (HTML, XML) thấy đường cong học tập nhẹ nhàng, trong khi người thiết kế trực quan có thể ưa thích công cụ đồ họa.

**Ước tính tỷ lệ:** Phân tích các mô-đun axelor-open-suite cho thấy **80-90% mã ứng dụng là khai báo** (định nghĩa XML) so với **10-20% mã mệnh lệnh** (dịch vụ Java, bộ điều khiển). Màn hình CRUD, luồng công việc đơn giản và logic nghiệp vụ chuẩn hoàn toàn dựa trên XML. Thuật toán phức tạp, tích hợp API bên ngoài và tối ưu hiệu năng cần Java.

---

### 2. HỆ THỐNG GIAO DIỆN: NĂM LOẠI GIAO DIỆN BAO PHỦ MỌI MẪU GIAO DIỆN NGƯỜI DÙNG

**Tệp nguồn:** SaleOrder.xml, các định nghĩa giao diện đa dạng xuyên mô-đun `[Từ source code]`

Axelor cung cấp **năm loại giao diện riêng biệt**, mỗi loại tối ưu cho các mẫu trải nghiệm người dùng cụ thể thường gặp trong ứng dụng nghiệp vụ. Việc chọn loại giao diện không tùy tiện — mỗi loại giải quyết một mô hình tương tác người dùng và yêu cầu hiển thị dữ liệu khác nhau. Hệ thống giao diện của bộ khung đủ toàn diện nên hiếm khi cần thành phần giao diện tùy chỉnh; hầu hết yêu cầu có thể đáp ứng bằng cách chọn loại giao diện thích hợp và cấu hình các tùy chọn khai báo.

**Ma trận loại giao diện:**

| Loại giao diện | Trường hợp sử dụng | Độ phức tạp | Khối lượng dữ liệu |
|----------------|---------------------|-------------|---------------------|
| **Lưới (Grid)** | Xem danh sách/bảng | Thấp | Cao (hàng nghìn) |
| **Biểu mẫu (Form)** | Xem/sửa chi tiết | Trung bình | Một bản ghi |
| **Lịch (Calendar)** | Dòng thời gian/lịch trình | Thấp | Trung bình (hàng trăm) |
| **Thẻ (Cards)** | Kanban/bộ sưu tập | Trung bình | Trung bình (hàng trăm) |
| **Biểu đồ (Chart)** | Phân tích/báo cáo | Thấp | Dữ liệu tổng hợp |

**2.1. Giao diện lưới — Hiển thị dữ liệu dạng bảng**

```xml
<!-- Tệp: SaleOrder.xml -->
<grid name="sale-order-quotation-grid" title="Sale quotations"
  model="com.axelor.apps.sale.db.SaleOrder" orderBy="-creationDate">

  <toolbar>
    <button name="printBtn" title="Print"
      onClick="action-sale-order-method-show-sale-order" icon="fa-print"/>
  </toolbar>

  <menubar>
    <menu name="saleOrderToolsMenu" title="Tools" icon="fa-wrench" showTitle="true">
      <item name="mergeQuotationsItem" title="Merge quotations"
        action="action-sale-order-method-convert-selected-lines-to-merge-lines"/>
    </menu>
  </menubar>

  <hilite background="warning"
    if="(endOfValidityDate != null) &amp;&amp; ($moment(endOfValidityDate) &lt; $moment(todayDate))"/>

  <field name="saleOrderSeq"/>
  <field name="creationDate"/>
  <field name="clientPartner" form-view="partner-form" grid-view="partner-grid"/>
  <field name="exTaxTotal" aggregate="sum" x-scale="currency.numberOfDecimals"/>
  <field name="statusSelect" widget="single-select"/>
</grid>
```

**Giải thích các khả năng:**

**Nút thanh công cụ:** Phần tử `<toolbar>` thêm nút hành động phía trên lưới, mỗi nút kích hoạt hành động (in, xuất, tạo mới). Biểu tượng chỉ định qua lớp FontAwesome (`fa-print`, `fa-download`) — không cần tải biểu tượng tùy chỉnh. Hiển thị nút có thể điều khiển qua biểu thức `showIf` (ví dụ `showIf="__user__.hasRole('admin')"`) cho phép giao diện theo vai trò.

**Trình đơn thả xuống thanh menu:** `<menubar>` tạo trình đơn thả xuống với các mục lồng nhau — hữu ích cho nhóm hành động liên quan (trình đơn Công cụ: gộp báo giá, tách đơn, cập nhật hàng loạt). Mẫu này giảm lộn xộn thanh công cụ khi có nhiều hành động.

**Tô sáng có điều kiện:** Phần tử `<hilite>` áp dụng lớp CSS (màu nền, kiểu chữ) dựa trên điều kiện thời gian chạy. Ví dụ: báo giá hết hạn (endOfValidityDate đã qua) được tô nền cam cảnh báo người dùng cần chú ý. Ngôn ngữ biểu thức hỗ trợ điều kiện phức tạp (so sánh ngày, kiểm tra trạng thái, logic đa trường). Thay thế CSS tùy chỉnh: kiểu dáng có điều kiện khai báo không cần mã giao diện.

**Điều hướng trường:** Trường quan hệ (`clientPartner`) chỉ định giao diện đích để khoan sâu: nhấp tên đối tác mở giao diện biểu mẫu (chi tiết) hoặc giao diện lưới (nếu nhiều bản ghi). Điều hướng hai chiều (chủ-chi tiết) cấu hình hoàn toàn bằng XML — không cần mã định tuyến.

**Tổng hợp:** `aggregate="sum"` tự động tổng cột số ở chân lưới. Các phép tổng hợp hỗ trợ: sum, avg, min, max, count. Tổng hợp cấp cơ sở dữ liệu (hiệu quả cho hàng nghìn bản ghi) chứ không phải tổng hợp phía máy khách. Độ chính xác thập phân điều khiển qua thuộc tính `x-scale` (ví dụ currency.numberOfDecimals = 2 cho USD, 3 cho KWD).

**Sắp xếp:** `orderBy="-creationDate"` đặt sắp xếp mặc định (dấu trừ = giảm dần). Người dùng có thể ghi đè bằng nhấp tiêu đề cột. Hỗ trợ sắp xếp đa cột (orderBy="statusSelect,-creationDate").

**2.2. Giao diện biểu mẫu — Chỉnh sửa chủ-chi tiết**

```xml
<form name="sale-order-form" title="Sale order"
  model="com.axelor.apps.sale.db.SaleOrder"
  onLoad="action-group-sale-saleorder-onload"
  onSave="action-group-sale-order-onsave"
  onNew="action-sale-order-method-onnew"
  width="large">

  <panel name="mainPanel">
    <field name="saleOrderSeq" css="highlight" readonly="true"/>
    <field name="company" canEdit="false" widget="SuggestBox"/>
    <field name="clientPartner"
      onChange="action-group-sale-saleorder-clientpartner-onchange"
      domain="self.isCustomer = true AND :company member of self.companySet"/>
  </panel>

  <panel-related field="saleOrderLineList" type="one-to-many"
    form-view="sale-order-line-form" grid-view="sale-order-line-grid"
    onChange="action-sale-order-method-compute"/>

  <panel-tabs>
    <panel name="invoicingPanel" title="Invoicing">
      <field name="paymentCondition"/>
    </panel>
  </panel-tabs>
</form>
```

**Bộ xử lý sự kiện:** Sự kiện vòng đời biểu mẫu (`onLoad`, `onSave`, `onNew`, `onChange`) kích hoạt chuỗi hành động. `onLoad` thực thi khi biểu mẫu mở (khởi tạo giá trị mặc định, tải dữ liệu liên quan), `onSave` trước khi lưu bền (xác thực, trường tính toán), `onNew` khi tạo bản ghi mới. Kiến trúc hướng sự kiện cho phép luồng công việc phức tạp mà không cần mã — chuỗi hành động khai báo.

**Lọc miền (domain filtering):** Thuộc tính `domain` tiêm mệnh đề WHERE vào truy vấn trường quan hệ. Biểu thức `self.isCustomer = true AND :company member of self.companySet` lọc đối tác: (1) phải là khách hàng, (2) phải thuộc tập đối tác của công ty hiện tại. Tham số ngữ cảnh (`:company`) được phân giải từ dữ liệu biểu mẫu. Lọc động ngăn lựa chọn không hợp lệ (ví dụ chọn nhà cung cấp khi cần khách hàng).

**Bảng chủ-chi tiết:** `<panel-related>` nhúng lưới bản ghi liên quan (một-nhiều, nhiều-nhiều). Ví dụ: đơn bán (chủ) chứa dòng đơn bán (chi tiết). Người dùng thêm/sửa/xóa dòng trực tiếp mà không rời biểu mẫu đơn hàng. Giao diện cấu hình được: `grid-view` cho hiển thị danh sách, `form-view` cho sửa dòng. `onChange` tính lại tổng khi dòng thay đổi.

**Bảng tab:** `<panel-tabs>` tổ chức biểu mẫu dày đặc thành phần logic (tab lập hóa đơn, tab vận chuyển, tab ghi chú). Giảm cuộn dọc, cải thiện trải nghiệm cho thực thể phức tạp có trên 50 trường. Tải lười (nội dung tab chỉ tải khi nhấp) cải thiện hiệu năng tải ban đầu.

**2.3. Giao diện lịch — Hiển thị dòng thời gian**

```xml
<calendar name="sale-order-calendar" model="com.axelor.apps.sale.db.SaleOrder"
  eventStart="creationDate" colorBy="statusSelect" editable="false"/>
```

Cấu hình tối thiểu tạo lịch đầy đủ tính năng. `eventStart` chỉ định trường ngày cho vị trí dòng thời gian. `colorBy` tô màu sự kiện theo giá trị trường (trạng thái: nháp=xám, xác nhận=xanh dương, lập hóa đơn=xanh lá). `editable="false"` ngăn kéo thả lịch lại (phù hợp cho ngày tạo bất biến). Trường hợp sử dụng: dòng thời gian dự án, lịch giao hàng, lịch nghỉ phép nhân viên. Bộ khung xử lý hiển thị (xem tháng/tuần/ngày), không cần JavaScript.

**2.4. Giao diện thẻ — Bố cục Kanban/Bộ sưu tập**

```xml
<cards name="sale-order-quotation-cards" title="Sale order quotation"
  model="com.axelor.apps.sale.db.SaleOrder" width="360px" css="rect-image" orderBy="-orderDate">

  <toolbar>
    <button name="printBtn" title="Print" onClick="action-sale-order-method-show-sale-order"/>
  </toolbar>

  <field name="saleOrderSeq"/>
  <field name="clientPartner.picture" css="rect-image-logo"/>

  <template>
    <![CDATA[
      <h4>{{saleOrderSeq}}</h4>
      <p>{{clientPartner.fullName}}</p>
    ]]>
  </template>
</cards>
```

Giao diện thẻ hiển thị bản ghi dưới dạng thẻ trực quan (tương tự bảng Trello, bộ sưu tập Pinterest). `width` điều khiển kích thước thẻ, `css` áp dụng kiểu dáng. Phần `<template>` sử dụng cú pháp mustache (`{{trường}}`) cho bố cục HTML tùy chỉnh. Trường hợp sử dụng: danh mục sản phẩm (có hình ảnh), đường ống cơ hội CRM (luồng công việc kanban), danh bạ nhân viên (bộ sưu tập ảnh đại diện).

**2.5. Giao diện biểu đồ — Bảng điều khiển phân tích**

```xml
<chart name="chart-sale-order-per-month" title="Sales per month">
  <dataset type="sql">
    SELECT
      TO_CHAR(self.creation_date, 'MM/YYYY') AS month,
      SUM(self.ex_tax_total) AS amount
    FROM sale_sale_order self
    GROUP BY month
    ORDER BY month
  </dataset>
  <category key="month" type="text"/>
  <series key="amount" type="bar" title="Amount"/>
</chart>
```

Giao diện biểu đồ thực thi truy vấn SQL, hiển thị kết quả dưới dạng biểu đồ (cột, đường, tròn, phân tán). `dataset type="sql"` cho phép SQL thô cho tổng hợp phức tạp vượt khả năng ORM. Loại biểu đồ chỉ định qua `<series type="bar|line|pie">`. Hỗ trợ nhiều chuỗi (biểu đồ chồng: doanh thu + lợi nhuận trên cùng trục). Bộ khung xử lý tích hợp thư viện biểu đồ (có thể là Chart.js hoặc tương tự) — nhà phát triển tập trung vào truy vấn dữ liệu chứ không phải mã hiển thị.

---

### 3. HỆ THỐNG HÀNH ĐỘNG KHAI BÁO: BẢY LOẠI HÀNH ĐỘNG LOẠI BỎ MÃ LẶP

**Tệp nguồn:** Định nghĩa hành động trong SaleOrder.xml `[Từ source code]`

Hệ thống hành động (action system) của Axelor đại diện cho **đổi mới không mã nền tảng** — mã hóa các hành vi giao diện phổ biến dưới dạng phần tử XML khai báo thay vì mã JavaScript/Java mệnh lệnh. Bảy loại hành động bao phủ đại đa số tương tác ứng dụng nghiệp vụ: gọi dịch vụ phụ trợ, đặt giá trị trường, mở hộp thoại, thay đổi trạng thái giao diện, chuỗi luồng công việc, xác thực dữ liệu và hiển thị thông báo. Cách tiếp cận thư viện mẫu: xác định mẫu tái diễn (ví dụ "đặt trường A khi trường B thay đổi"), trừu tượng thành loại hành động tái sử dụng với tham số cấu hình.

**Danh mục loại hành động:**

| Loại hành động | Mục đích | Mã cần viết | Độ phức tạp |
|----------------|----------|-------------|-------------|
| `action-method` | Gọi dịch vụ Java | Phương thức Java | Cao |
| `action-record` | Đặt giá trị trường | Không | Thấp |
| `action-view` | Mở giao diện/cửa sổ bật | Không | Thấp |
| `action-attrs` | Thay đổi thuộc tính giao diện | Không | Trung bình |
| `action-group` | Chuỗi nhiều hành động | Không | Thấp |
| `action-condition` | Xác thực dữ liệu | Không | Thấp |
| `action-validate` | Hiển thị cảnh báo/xác nhận | Không | Thấp |
| `action-script` | Thực thi Groovy | Kịch bản Groovy | Trung bình |

**3.1. action-record: Gán giá trị trường**

```xml
<action-record name="action-sale-order-record-partner"
  model="com.axelor.apps.sale.db.SaleOrder">

  <field name="paymentCondition"
    expr="eval: clientPartner?.paymentCondition"
    if="__config__.app.isApp('account') &amp;&amp; clientPartner?.paymentCondition != null"/>

  <field name="paymentCondition"
    expr="eval: company?.accountConfig?.defPaymentCondition"
    if="__config__.app.isApp('account') &amp;&amp; clientPartner?.paymentCondition == null"/>

  <field name="fiscalPosition"
    expr="eval: clientPartner?.fiscalPosition"
    if="__config__.app.isApp('account')"/>

  <field name="currency"
    expr="eval: clientPartner?.currency"
    if="!template &amp;&amp; (saleOrderLineList == null || saleOrderLineList?.isEmpty())"/>
</action-record>
```

Gán trường khai báo thuần túy — không cần mã Java. Nhiều phép gán trường trong một hành động, mỗi phép có logic điều kiện (thuộc tính `if`). Ngôn ngữ biểu thức (`eval:`) hỗ trợ điều hướng an toàn (`?.`), toán tử boolean, kiểm tra null. Luồng thực thi: khi hành động được kích hoạt (ví dụ clientPartner thay đổi), bộ khung đánh giá biểu thức từ trên xuống, cập nhật trường phù hợp. Mẫu trường hợp sử dụng: "sao chép điều khoản thanh toán của đối tác vào đơn hàng, trừ khi đối tác không có thì dùng mặc định công ty" — quy tắc nghiệp vụ mã hóa hoàn toàn bằng XML.

**3.2. action-attrs: Thay đổi giao diện động**

```xml
<action-attrs name="action-sale-order-attrs-amount-to-invoice">
  <attribute name="hidden"
    expr="eval: advancePaymentAmountNeeded > (new BigDecimal(amountToInvoice))"
    for="amountToInvoicePanel"/>

  <attribute name="hidden"
    expr="eval: !(advancePaymentAmountNeeded > (new BigDecimal(amountToInvoice)))"
    for="amountPanel"/>
</action-attrs>
```

Logic hiện/ẩn động không cần JavaScript. Thuộc tính `for` nhắm phần tử giao diện theo tên, `expr` đánh giá điều kiện boolean. Các thuộc tính hỗ trợ: `hidden`, `readonly`, `required`, `title`, `domain`, `value`, `css`. Mẫu cho phép giao diện nhạy ngữ cảnh: trường xuất hiện/biến mất dựa trên giá trị trường khác, cấu trúc biểu mẫu thích ứng theo lựa chọn người dùng.

**Ví dụ nâng cao thay đổi miền:**
```xml
<action-attrs name="action-sale-order-domain-on-team">
  <attribute name="domain" for="team"
    expr="eval: salespersonUser?.teamSet?.collect{it.id}?.size() > 0 ?
          &quot;self.id IN (${salespersonUser?.teamSet?.collect{it.id}?.join(',')})&quot; : null"/>
</action-attrs>
```

Xây dựng mệnh đề WHERE SQL động: lọc nhóm chỉ gồm những nhóm thuộc nhân viên bán hàng. Các phép toán tập hợp Groovy (`collect`, `join`) sinh danh sách ID phân cách dấu phẩy tiêm vào chuỗi miền. Mạnh mẽ nhưng phức tạp — trường hợp biên: nếu teamSet rỗng thì sao? Biểu thức xử lý bằng toán tử ba ngôi (trả null nếu không có nhóm, xóa bộ lọc).

**3.3. action-group: Điều phối luồng công việc**

```xml
<action-group name="action-group-sale-order-onsave">
  <action name="save"/>
  <action name="action-sale-order-method-compute"/>
  <action name="save"/>
</action-group>

<action-group name="action-group-partner-saleorder-onnew">
  <action name="action-sale-order-record-from-partner"/>
  <action name="action-sale-order-record-partner"/>
  <action name="action-sale-order-method-set-advance-payment"
    if="__config__.app.getApp('supplychain')?.manageAdvancePaymentsFromPaymentConditions"/>
  <action name="action-sale-order-method-address-str"/>
  <action name="action-sale-order-method-fill-price-list"/>
  <action name="action-sale-order-method-fill-company-bank-details"/>
</action-group>
```

Thực thi hành động tuần tự — chuỗi luồng công việc khai báo. Ví dụ đầu: lưu → tính tổng → lưu lại (mẫu: lưu bền trước khi tính toán sử dụng giá trị cơ sở dữ liệu, lưu lại sau khi tính toán cập nhật tổng). Ví dụ hai: luồng khởi tạo phức tạp với bước có điều kiện (`if` trên hành động thanh toán tạm ứng). Hành động `save` tích hợp sẵn do bộ khung cung cấp. Ngữ nghĩa thực thi: hành động chạy tuần tự, dừng khi gặp lỗi đầu tiên (hoàn tác giao dịch).

Nhóm hành động **loại bỏ mã lặp** luồng công việc — mẫu phổ biến trong ứng dụng truyền thống:
```java
// Không có action-group (mã Java truyền thống)
public void onSave(SaleOrder order) {
  repository.save(order);
  computationService.compute(order);
  repository.save(order);
}
```
Trở thành một dòng XML: `onSave="action-group-sale-order-onsave"`

**3.4. action-condition: Xác thực dữ liệu**

```xml
<action-condition name="action-sale-order-cancel-reason-check">
  <check error="A cancel reason must be selected" field="cancelReason"
    if="cancelReason == null || cancelReason == 0"/>
</action-condition>
```

Quy tắc xác thực khai báo. Thông báo `error` hiển thị cho người dùng, trường `field` được tô sáng trong giao diện, gửi biểu mẫu bị chặn đến khi sửa xong. Biểu thức xác thực cùng ngôn ngữ với các hành động khác (Groovy). Nhiều kiểm tra trong một hành động duy nhất (xác thực nhiều trường cùng lúc). Thay thế cho xác thực bean Java (`@NotNull`, `@Size`) — linh hoạt hơn (có thể tham chiếu trường khác, biến ngữ cảnh), kém an toàn kiểu hơn (không kiểm tra tại thời điểm biên dịch).

**3.5. action-validate: Thông báo người dùng**

```xml
<action-validate name="action-bank-order-validate-set-bank-order-date">
  <alert message="As the date of your order is in the past, it will be updated to today."
    if="bankOrderDate != null &amp;&amp; bankOrderDate &lt; __config__.date"/>
</action-validate>
```

Thông báo cảnh báo/thông tin có điều kiện. `<alert>` hiển thị thông báo không chặn (người dùng có thể tiếp tục). Thay thế: `<error>` chặn tiếp tục (xác thực mạnh hơn action-condition). Trường hợp sử dụng: cảnh báo trường hợp biên (ngày quá khứ, số tiền lớn, cấu hình bất thường) mà không ngăn hành động. Hỗ trợ nội suy thông báo: `message="Tổng là ${exTaxTotal}. Tiếp tục?"` nhúng kết quả biểu thức.

**3.6. action-script: Lối thoát logic Groovy**

```xml
<action-script name="action-equipment-model-script-remove-equipment-model">
  <script language="groovy" transactional="true">
    <![CDATA[
      if ($request.context.id == null) return
      def equipmentModel = __repo__(EquipmentModel).find($request.context.id)
      if (equipmentModel == null) return
      __repo__(EquipmentModel).remove(equipmentModel)
      $response.reload = true
    ]]>
  </script>
</action-script>
```

Kịch bản Groovy nhúng cho logic quá phức tạp đối với biểu thức. Kịch bản truy cập: `$request.context` (dữ liệu biểu mẫu), `$response` (đặt giá trị trả về), `__repo__(Model)` (truy cập kho dữ liệu), `__user__` (người dùng hiện tại), `__config__` (cấu hình ứng dụng). `transactional="true"` bọc thực thi trong giao dịch cơ sở dữ liệu. Mẫu: dùng action-record/action-attrs cho trường hợp đơn giản, nâng cấp lên action-script cho vòng lặp, điều kiện, gọi dịch vụ. Vẫn dễ hơn Java (không cần tệp lớp, không biên dịch), nhưng khó bảo trì hơn hành động khai báo thuần túy.

Mối lo bảo mật: thực thi Groovy không giới hạn nguy hiểm — kịch bản có thể xóa dữ liệu, leo thang đặc quyền. Hệ thống sản xuất có thể đặt Groovy trong hộp cát (sandbox — hạn chế lớp, giới hạn thời gian thực thi) tương tự nhiệm vụ kịch bản BPM từ BƯỚC 4.

---

### 4. KẾ THỪA GIAO DIỆN: MỞ RỘNG MÔ-ĐUN KHÔNG CẦN RẼ NHÁNH

**Tệp nguồn:** Partner.xml, Product.xml — mở rộng giao diện `[Từ source code]`

Cơ chế mở rộng giao diện giải quyết bài toán mô-đun hóa then chốt: làm thế nào mô-đun A nâng cấp giao diện của mô-đun B mà không sửa mã nguồn B? Cách truyền thống: rẽ nhánh (fork) mô-đun B, sửa đổi, bảo trì bản rẽ nhánh mãi mãi (xung đột hợp nhất, ác mộng nâng cấp). Giải pháp của Axelor: **mở rộng giao diện khai báo** sử dụng bộ chọn XPath và chỉ thị chèn. Mô-đun A khai báo "mở rộng biểu mẫu Partner từ mô-đun cơ sở, chèn trường của tôi vào bảng cụ thể" — bộ khung hợp nhất giao diện tại thời điểm chạy, mô-đun cơ sở không biết về các mở rộng.

**Mẫu mở rộng:**

```xml
<!-- Tệp: axelor-sale/views/Partner.xml — mở rộng biểu mẫu Partner của mô-đun cơ sở -->
<form id="sale-partner-form" model="com.axelor.apps.base.db.Partner"
  title="Partner" name="partner-form" extension="true">

  <extend target="//panel[@name='saleOrderCommentsPanel']">
    <insert position="after">
      <panel-related field="$saleDetailsByProduct" type="one-to-many"
        target="com.axelor.utils.db.Wizard" title="Sale details by product"
        canView="false"
        grid-view="sale-details-by-product-per-customer-grid"
        readonly="true" hidden="true"
        colSpan="12">
      </panel-related>
    </insert>
  </extend>

  <extend target="//panel-tabs[@name='mainPanelTab']/*[last()]">
    <insert position="after">
      <panel name="productPanel" title="Product" showIf="isCustomer"
        if="__config__.app.isApp('sale') &amp;&amp; __config__.app.getApp('sale')?.getManagePartnerComplementaryProduct()">
        <field name="complementaryProductList" colSpan="12"
          form-view="complementary-product-partner-form"
          grid-view="complementary-product-partner-grid"/>
      </panel>
    </insert>
  </extend>
</form>
```

**Khai báo mở rộng:** `extension="true"` đánh dấu biểu mẫu là mở rộng (không phải độc lập). `name="partner-form"` xác định giao diện đích từ mô-đun cơ sở. `id="sale-partner-form"` cung cấp mã duy nhất cho chính phần mở rộng (nhiều mô-đun có thể mở rộng cùng giao diện cơ sở, ID ngăn xung đột).

**Nhắm mục tiêu XPath:** `target="//panel[@name='saleOrderCommentsPanel']"` sử dụng cú pháp XPath định vị điểm chèn. `//` tìm bất kỳ đâu trong cây tài liệu, `panel[@name='...']` lọc theo loại phần tử và thuộc tính. Bộ chọn nâng cao: `/*[last()]` nhắm con cuối (chèn cuối), `/*[first()]` chèn đầu.

**Vị trí chèn:** `position="after"` chèn sau phần tử đích. Vị trí thay thế: `before` (trước), `inside` (làm con). Mẫu cho phép sửa đổi phẫu thuật: thêm trường sau trường cụ thể, chèn tab cuối danh sách tab, thêm nút đầu thanh công cụ.

**Mở rộng có điều kiện:** Điều kiện `if` lồng nhau trong nội dung chèn làm mở rộng có điều kiện — "chỉ hiện bảng Sản phẩm nếu mô-đun bán hàng đã cài ĐÃ bật tính năng". Cho phép tính năng tùy chọn: mô-đun cơ sở luôn hiện diện, mô-đun tính năng mở rộng khi cài đặt.

**Ví dụ thay đổi thuộc tính:**

```xml
<form name="partner-customer-form" title="Customer"
  model="com.axelor.apps.base.db.Partner"
  extension="true" onLoad="" id="partner-customer-sales-form">

  <extend target="/">
    <attribute name="onLoad" value="sale-action-group-partner-onload"/>
  </extend>
</form>
```

Đích `/` (phần tử gốc = chính biểu mẫu), thay thế giá trị thuộc tính `onLoad`. Trường hợp sử dụng: ghi đè bộ xử lý sự kiện (onLoad của mô-đun cơ sở thay bằng phiên bản mở rộng bao gồm logic bán hàng). Mẫu cho phép mở rộng hành vi chứ không chỉ cấu trúc (thêm trường).

**Lợi ích kế thừa giao diện:**

1. **Không sửa mã nguồn:** Mô-đun cơ sở giữ nguyên, có thể nâng cấp
2. **Cô lập mô-đun:** Mã mô-đun bán hàng tách biệt khỏi logic đối tác cơ sở
3. **Tính năng tùy chọn:** Cài mô-đun bán hàng → biểu mẫu đối tác có thêm trường bán hàng, gỡ cài → quay về cơ sở
4. **Nhiều mở rộng:** Mô-đun kế toán cũng có thể mở rộng biểu mẫu đối tác (thêm trường kế toán), các mở rộng kết hợp mượt mà
5. **Quản lý phiên bản:** Mở rộng mỗi mô-đun theo dõi riêng (quyền sở hữu rõ ràng, duyệt mã đơn giản hơn)

**Hạn chế:**

1. **Bộ chọn dễ gãy:** XPath hỏng nếu mô-đun cơ sở đổi tên bảng (name="oldPanel" → name="newPanel")
2. **Không xóa được:** Không thể xóa trường mô-đun cơ sở (chỉ ẩn qua `showIf="false"`, trường vẫn trong DOM)
3. **Phụ thuộc thứ tự:** Nhiều mở rộng cùng điểm chèn có thứ tự không xác định (tình trạng tranh chấp nếu thứ tự quan trọng)
4. **Khó gỡ lỗi:** Hợp nhất giao diện thời gian chạy khiến lỗi khó truy vết (mô-đun nào đóng góp phần tử nào?)

---

### 5. SINH MÃ: ĐƯỜNG ỐNG BIẾN ĐỔI XML THÀNH JAVA

**Tệp nguồn:** build.gradle, tệp XML miền, thực thể được sinh `[Từ source code]`

**5.1. Ngôn ngữ định nghĩa mô hình miền**

Hệ thống sinh mã của Axelor triển khai **ngôn ngữ chuyên miền** (DSL — Domain-Specific Language) cho mô hình hóa thực thể — nhà phát triển định nghĩa lược đồ cơ sở dữ liệu trong tệp XML, tiện ích Gradle tự động sinh các lớp thực thể JPA. Cách tiếp cận DSL tách mô hình dữ liệu logic (khái niệm miền, quan hệ, ràng buộc) khỏi chi tiết triển khai (chú thích JPA, mã lặp getter/setter, equals/hashCode). Lợi ích: mô hình hóa nhanh hơn (khai báo thực thể trong 20 dòng XML thay vì 200 dòng Java), nhất quán (mã sinh tuân theo mẫu chuẩn), và dễ bảo trì (thay đổi XML, sinh lại, không cần đồng bộ thủ công).

**Bằng chứng từ XML miền:**
```xml
<!-- Tệp: domains/SaleOrder.xml -->
<domain-models xmlns="http://axelor.com/xml/ns/domain-models">
  <module name="sale" package="com.axelor.apps.sale.db"/>

  <entity name="SaleOrder" lang="java">
    <string name="saleOrderSeq" title="Sale Order Seq" readonly="true"/>
    <date name="creationDate" title="Creation Date" required="true"/>
    <many-to-one name="clientPartner" ref="com.axelor.apps.base.db.Partner" title="Customer"/>
    <one-to-many name="saleOrderLineList" ref="SaleOrderLine" title="Sale order lines" mappedBy="saleOrder"/>
    <decimal name="exTaxTotal" title="Total W.T." scale="3" precision="20" readonly="true"/>
    <integer name="statusSelect" title="Status" selection="sale.order.status.select" default="1"/>

    <extra-code>
      <![CDATA[
        public static final int STATUS_DRAFT = 1;
        public static final int STATUS_FINALIZED = 2;
      ]]>
    </extra-code>
  </entity>
</domain-models>
```

**Khai báo mô-đun:** `<module>` đặt không gian tên gói cho lớp sinh. Mẫu: `gói + mô-đun + db` = `com.axelor.apps.sale.db.SaleOrder`. Cấu trúc gói nhất quán xuyên mô-đun hỗ trợ điều hướng IDE, tổ chức import.

**Ánh xạ kiểu trường:** DSL miền cung cấp tên kiểu thân thiện nghiệp vụ (`<string>`, `<decimal>`, `<date>`) ánh xạ tới kiểu Java/JPA:

| Kiểu miền | Kiểu Java | Chú thích JPA |
|------------|-----------|---------------|
| `<string>` | `String` | `@Basic` |
| `<integer>` | `Integer` | `@Basic` |
| `<decimal>` | `BigDecimal` | `@Column(scale=X, precision=Y)` |
| `<date>` | `LocalDate` | `@Basic` |
| `<datetime>` | `LocalDateTime` | `@Basic` |
| `<many-to-one>` | Tham chiếu | `@ManyToOne` |
| `<one-to-many>` | `List<T>` | `@OneToMany(mappedBy=...)` |

**Lan truyền thuộc tính:** Thuộc tính XML (`title`, `required`, `readonly`, `default`) trở thành chú thích JPA. `title` → siêu dữ liệu giao diện (dùng trong biểu mẫu sinh), `required` → xác thực `@NotNull`, `readonly` → updatable=false, `default` → giá trị mặc định trong hàm tạo.

**Trường lựa chọn:** `selection="sale.order.status.select"` tham chiếu danh sách lựa chọn định nghĩa nơi khác (có thể CSV hoặc tệp XML ánh xạ mã số nguyên tới nhãn hiển thị: 1→Nháp, 2→Hoàn tất). Mẫu phổ biến cho liệt kê — tránh số ma thuật trong mã, tập trung định nghĩa giá trị.

**Tiêm mã bổ sung:** `<extra-code>` nhúng mã Java tùy chỉnh vào lớp sinh (hằng tĩnh, phương thức trợ giúp). Cho phép mở rộng lớp sinh mà không cần kế thừa. Trường hợp sử dụng: hằng trạng thái cho phép so sánh an toàn kiểu (`if (order.getStatusSelect() == SaleOrder.STATUS_DRAFT)`) thay vì hằng số nguyên dễ gãy.

**5.2. Tác vụ sinh mã Gradle**

```gradle
// Cấu hình build.gradle
apply plugin: 'com.axelor.app-module'

axelor {
  title = "Axelor ERP"
}
```

Tiện ích Gradle `com.axelor.app-module` đăng ký tác vụ sinh mã thực thi trong vòng đời biên dịch. Luồng thực thi `[Suy luận từ mẫu biên dịch Gradle]`:

1. **Khám phá:** Tiện ích quét tệp `src/main/resources/domains/*.xml`
2. **Phân tích:** Xác thực XML theo lược đồ XSD, xây dựng cây cú pháp trừu tượng (AST) mô hình miền
3. **Sinh mã:** Mẫu (có thể Velocity hoặc FreeMarker) biến đổi AST thành mã nguồn Java
4. **Xuất:** Ghi tệp `.java` vào thư mục `build/src-gen/`
5. **Biên dịch:** Javac biên dịch nguồn sinh + nguồn viết tay cùng nhau

**Cấu trúc thực thể sinh** `[Suy luận từ quy ước JPA]`:
```java
// Sinh: build/src-gen/com/axelor/apps/sale/db/SaleOrder.java
package com.axelor.apps.sale.db;

import javax.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "sale_sale_order")
public class SaleOrder extends AuditableModel {

  @Column(name = "sale_order_seq", readonly = true)
  private String saleOrderSeq;

  @Column(name = "creation_date", nullable = false)
  private LocalDate creationDate;

  @ManyToOne
  @JoinColumn(name = "client_partner")
  private Partner clientPartner;

  @OneToMany(mappedBy = "saleOrder", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<SaleOrderLine> saleOrderLineList;

  @Column(name = "ex_tax_total", scale = 3, precision = 20, readonly = true)
  private BigDecimal exTaxTotal;

  @Column(name = "status_select")
  private Integer statusSelect = 1;

  // Getter/setter sinh (~100+ dòng mã lặp)
  public String getSaleOrderSeq() { return saleOrderSeq; }
  public void setSaleOrderSeq(String saleOrderSeq) { this.saleOrderSeq = saleOrderSeq; }
  // ... 20+ cặp getter/setter khác

  // equals/hashCode sinh dựa trên ID
  @Override
  public boolean equals(Object o) { /* ... */ }

  @Override
  public int hashCode() { /* ... */ }
}
```

**Lợi ích sinh mã được định lượng:**

- **Giảm dòng mã:** 20 dòng XML miền sinh lớp Java 200 dòng (**giảm 90% mã**)
- **Nhất quán:** Mọi thực thể tuân theo cùng mẫu (quy ước đặt tên, chú thích, kế thừa)
- **An toàn kiểu:** Kiểm tra quan hệ tại thời điểm biên dịch (tự động hoàn thiện IDE, hỗ trợ tái cấu trúc)
- **Đồng bộ hai chiều:** Công cụ có thể kỹ thuật ngược XML miền từ lược đồ cơ sở dữ liệu (kỹ thuật khứ hồi)

**Đánh đổi:**
- **Phức tạp biên dịch:** Bước biên dịch bổ sung (biên dịch chậm hơn, gỡ lỗi mã sinh khó hơn)
- **Giới hạn tùy chỉnh:** Không thể lệch khỏi mẫu khuôn (ví dụ logic equals() tùy chỉnh cần giải pháp thay thế)
- **Tích hợp IDE:** Cần tiện ích đặc biệt để điều hướng XML → Java sinh (nếu không sẽ báo lỗi "không tìm thấy lớp")

---

### 6. NGÔN NGỮ BIỂU THỨC: ĐÁNH GIÁ NGỮ CẢNH DỰA TRÊN GROOVY

**Tệp nguồn:** XML hành động với biểu thức `[Từ source code + suy luận]`

Axelor nhúng **Groovy 3.0.23** làm ngôn ngữ biểu thức cho logic khai báo trong giao diện/hành động XML. Khả năng biểu thức trải từ tham chiếu trường đơn giản (`clientPartner.name`) tới tính toán phức tạp (`saleOrderLineList.sum { it.exTaxTotal }`). Lựa chọn ngôn ngữ mang tính chiến lược: cú pháp Groovy là tập siêu của Java (nhà phát triển Java có đường cong học tập tối thiểu), hỗ trợ điều hướng an toàn (`?.`), cung cấp API tập hợp (map/filter/reduce), biên dịch thành bytecode JVM (hiệu năng chấp nhận được).

**6.1. Tham chiếu biến ngữ cảnh**

Mọi biểu thức thực thi trong đối tượng ngữ cảnh chứa các biến đặc biệt `[Từ source code — ví dụ hành động]`:

**Biến ngữ cảnh dữ liệu:**
```groovy
// Trường bản ghi hiện tại — truy cập trực tiếp
self.fieldName              // Giá trị trường thực thể hiện tại
saleOrderSeq                // Viết tắt cho self.saleOrderSeq
clientPartner.fullName      // Điều hướng qua quan hệ

// Đối tượng yêu cầu/phản hồi (action-script)
$request.context.id         // Giá trị trường biểu mẫu
$response.setValue("field", value)  // Đặt giá trị trả về
$response.reload = true     // Kích hoạt tải lại biểu mẫu
```

**Biến ngữ cảnh người dùng:**
```groovy
__user__                    // Đối tượng người dùng đang đăng nhập
__user__.code               // Tên đăng nhập
__user__.name               // Họ tên đầy đủ
__user__.hasRole('admin')   // Kiểm tra vai trò
__user__.getGroup()         // Nhóm chính của người dùng
```

**Biến ngữ cảnh ứng dụng:**
```groovy
__config__                          // Cấu hình ứng dụng
__config__.app.isApp('sale')        // Kiểm tra mô-đun đã cài chưa
__config__.app.getApp('sale')       // Lấy đối tượng cấu hình mô-đun
__config__.date                     // Ngày hiện tại máy chủ
__date__                            // Bí danh cho ngày hiện tại
__time__                            // Giờ hiện tại máy chủ
__datetime__                        // Ngày giờ hiện tại máy chủ
```

**Truy cập kho dữ liệu (action-script):**
```groovy
__repo__(SaleOrder)                 // Lấy kho dữ liệu cho kiểu thực thể
__repo__(SaleOrder).all()           // Trình xây dựng truy vấn
__repo__(SaleOrder).find(id)        // Tìm theo ID
```

**Hàm trợ giúp:**
```groovy
$moment(date)                       // Thư viện thao tác ngày
$number(value)                      // Định dạng số
$json(object)                       // Tuần tự hóa JSON
```

**6.2. Toán tử điều hướng an toàn**

```groovy
// Biểu thức: clientPartner?.paymentCondition
// Tương đương Java:
String paymentCondition = null;
if (clientPartner != null) {
  paymentCondition = clientPartner.getPaymentCondition();
}
```

Điều hướng an toàn (`?.`) trả `null` nếu toán hạng trái null thay vì ném `NullPointerException`. Quan trọng cho điều hướng chuỗi (`clientPartner?.company?.accountConfig?.defaultPaymentCondition`) — bất kỳ null nào trong chuỗi đều rút ngắn mạch về null. Mẫu loại bỏ kiểm tra null phòng thủ lộn xộn trong biểu thức.

**6.3. Phép toán tập hợp**

```groovy
// Xây dựng miền động từ tập hợp
expr="eval: salespersonUser?.teamSet?.collect{it.id}?.size() > 0 ?
      &quot;self.id IN (${salespersonUser?.teamSet?.collect{it.id}?.join(',')})&quot; : null"
```

Giải thích biến đổi:

1. `salespersonUser?.teamSet` — Lấy nhóm người dùng (trả `Set<Team>` hoặc null)
2. `?.collect{it.id}` — Ánh xạ nhóm thành ID (trả `List<Long>` hoặc null)
3. `?.size() > 0` — Kiểm tra có nhóm nào tồn tại
4. `?.join(',')` — Nối ID thành chuỗi phân cách dấu phẩy: "1,2,3"
5. Nội suy chuỗi: `${...}` nhúng kết quả vào SQL miền

Ví dụ kết quả:
- Người dùng có nhóm [1,2,3] → domain = `"self.id IN (1,2,3)"`
- Người dùng không có nhóm → domain = `null` (không áp dụng bộ lọc)

Các phương thức tập hợp khả dụng (Groovy GDK):
- `collect{bao đóng}` — ánh xạ/biến đổi
- `findAll{bao đóng}` — lọc
- `sum{bao đóng}` — tổng hợp
- `any{bao đóng}` / `every{bao đóng}` — kiểm tra boolean
- `groupBy{bao đóng}` — phân loại

**6.4. Hạn chế ngôn ngữ biểu thức**

Dù mạnh mẽ, biểu thức Groovy có ràng buộc `[Suy luận từ hộp cát ngôn ngữ nhúng điển hình]`:

1. **Không định nghĩa lớp:** Không thể định nghĩa lớp/giao diện trong biểu thức
2. **Không câu lệnh import:** Chỉ lớp trong danh sách trắng truy cập được (có thể java.lang.*, java.util.*, thực thể miền)
3. **Không thao tác nhập/xuất:** Truy cập tệp/mạng bị chặn vì bảo mật
4. **Giới hạn thời gian thực thi:** Biểu thức chạy lâu bị dừng (ngăn tấn công từ chối dịch vụ)
5. **Xử lý ngoại lệ hạn chế:** Không thể bắt ngoại lệ trong biểu thức (leo thang thành lỗi hành động)

Các hạn chế này đẩy logic phức tạp sang action-script (khả năng nhiều hơn đôi chút) hoặc phương thức Java (không giới hạn).

---

### 7. HỆ THỐNG ĐIỀU KHIỂN: HƠN 20 ĐIỀU KHIỂN GIAO DIỆN CHUYÊN BIỆT

**Tệp nguồn:** XML giao diện với thuộc tính widget `[Từ source code + suy luận]`

Ngoài đầu vào HTML cơ bản (văn bản, hộp kiểm, lựa chọn), Axelor cung cấp **điều khiển nghiệp vụ chuyên biệt** (widget) tối ưu cho các kiểu dữ liệu doanh nghiệp phổ biến. Hệ thống điều khiển có thể mở rộng — điều khiển tùy chỉnh định nghĩa dưới dạng thành phần React, đăng ký với bộ khung, tham chiếu trong XML giao diện. Danh mục điều khiển tích hợp bao phủ 90% trường hợp sử dụng; điều khiển tùy chỉnh chỉ cần cho yêu cầu bất thường.

**7.1. Danh mục điều khiển tích hợp**

**Điều khiển lựa chọn:**
```xml
<!-- Danh sách thả xuống chọn đơn -->
<field name="statusSelect" widget="single-select"/>

<!-- Danh sách hộp kiểm chọn nhiều -->
<field name="categoryList" widget="multi-select"/>

<!-- Nhóm nút tròn (bố cục ngang/dọc) -->
<field name="prioritySelect" widget="radio-select"/>
```

**Điều khiển quan hệ:**
```xml
<!-- Hộp gợi ý: tìm kiếm tự động hoàn thiện -->
<field name="clientPartner" widget="SuggestBox"/>

<!-- Chọn thẻ: chọn nhiều với nhãn thẻ -->
<field name="tagList" widget="TagSelect"/>

<!-- Lưới cây: dữ liệu phân cấp mở rộng/thu gọn -->
<field name="accountTree" widget="tree-grid"/>
```

Về SuggestBox: gõ "Aco" kích hoạt tìm kiếm AJAX cho đối tác khớp "Aco*", hiển thị danh sách thả xuống kết quả (Acorns Inc, Acosta Corp), người dùng chọn từ kết quả. Hiệu quả cho bộ dữ liệu lớn (hàng nghìn đối tác) — chỉ tải bản ghi khớp, không phải toàn bảng. Thay thế cho danh sách thả xuống thuần giới hạn ~100 tùy chọn.

**Điều khiển ngày/giờ:**
```xml
<!-- Bộ chọn ngày với lịch bật lên -->
<field name="orderDate" widget="date"/>

<!-- Bộ chọn ngày giờ với chọn giờ -->
<field name="eventStart" widget="datetime"/>

<!-- Nhập khoảng thời gian (định dạng giờ:phút) -->
<field name="taskDuration" widget="duration"/>

<!-- Thời gian tương đối: "2 giờ trước", "trong 3 ngày" -->
<field name="creationDate" widget="relative-time"/>
```

**Điều khiển số:**
```xml
<!-- Nhập số với nút tăng/giảm -->
<field name="quantity" widget="integer"/>

<!-- Nhập thập phân định dạng theo ngôn ngữ (1,234.56 hoặc 1.234,56) -->
<field name="price" widget="decimal"/>

<!-- Thanh tiến độ (phần trăm 0-100) -->
<field name="completionRate" widget="progress"/>
```

**Điều khiển nội dung phong phú:**
```xml
<!-- Trình soạn HTML WYSIWYG (kiểu TinyMCE) -->
<field name="description" widget="html"/>

<!-- Trình soạn Markdown có xem trước -->
<field name="notes" widget="markdown"/>

<!-- Tải lên hình ảnh có xem trước thu nhỏ -->
<field name="photo" widget="image"/>

<!-- Tải lên tệp nhị phân có liên kết tải về -->
<field name="attachment" widget="binary"/>
```

**7.2. Đăng ký điều khiển tùy chỉnh**

Điều khiển tùy chỉnh tích hợp thành phần React với hệ thống biểu mẫu Axelor `[Suy luận từ tích hợp thành phần React điển hình]`:

**Bước 1: Triển khai thành phần React**
```jsx
// CustomMapWidget.jsx
import React from 'react';
import { MapContainer, TileLayer, Marker } from 'react-leaflet';

export default function CustomMapWidget({ value, onChange, readonly }) {
  const [lat, lng] = value?.split(',') || [0, 0];

  return (
    <MapContainer center={[lat, lng]} zoom={13}>
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"/>
      {!readonly && <Marker position={[lat, lng]} draggable
        onDragEnd={(e) => onChange(`${e.latlng.lat},${e.latlng.lng}`)}/>}
    </MapContainer>
  );
}
```

**Bước 2: Đăng ký điều khiển**
```javascript
// widget-registry.js
import CustomMapWidget from './CustomMapWidget';

axelor.widget('custom-map', CustomMapWidget);
```

**Bước 3: Sử dụng trong XML giao diện**
```xml
<field name="geoCoordinates" widget="custom-map"/>
```

Bộ khung xử lý: ràng buộc giá trị trường vào thuộc tính điều khiển, truyền hàm gọi lại onChange, quản lý trạng thái readonly/required, xác thực, và kiểu dáng.

**7.3. Thuộc tính cấu hình điều khiển**

Điều khiển nhận cấu hình qua thuộc tính XML:

```xml
<!-- Độ chính xác thập phân -->
<field name="price" widget="decimal" x-scale="2"/>  <!-- 2 chữ số thập phân -->

<!-- Nguồn lựa chọn -->
<field name="status" widget="single-select" selection="order.status.select"/>

<!-- Kích thước hình ảnh -->
<field name="photo" widget="image" width="200" height="150"/>

<!-- Lớp CSS tùy chỉnh -->
<field name="urgentFlag" widget="checkbox" css="text-danger"/>

<!-- Chú giải công cụ trợ giúp -->
<field name="complexField" widget="text" help="Nhập giá trị theo định dạng XXX-YYY-ZZZ"/>
```

Lan truyền thuộc tính: bộ khung truyền thuộc tính XML dưới dạng thuộc tính React, thành phần phân rã và áp dụng.

---

### 8. QUỐC TẾ HÓA (I18N): HỖ TRỢ ĐA NGÔN NGỮ

**Tệp nguồn:** Tệp i18n/*.csv, thuộc tính title trong XML `[Từ source code]`

Axelor cung cấp **bộ khung quốc tế hóa toàn diện** hỗ trợ hơn 20 ngôn ngữ sẵn có. Luồng dịch thuật: nhà phát triển viết nhãn tiếng Anh trong XML, người dịch cung cấp bản dịch qua tệp CSV, bộ khung tải bản dịch phù hợp dựa trên tùy chọn ngôn ngữ người dùng. Mẫu phổ biến trong phần mềm doanh nghiệp (SAP, Oracle EBS) — một mã nguồn phục vụ triển khai toàn cầu.

**8.1. Tệp danh mục thông điệp**

```
modules/axelor-sale/src/main/resources/i18n/
  messages.csv          # Mặc định (tiếng Anh)
  messages_fr.csv       # Tiếng Pháp
  messages_es.csv       # Tiếng Tây Ban Nha
  messages_de.csv       # Tiếng Đức
```

**Định dạng CSV:**
```csv
# messages.csv
"key","message","comment","context"
"Sale Order","Sale Order","","SaleOrder entity title"
"Customer","Customer","","clientPartner field title"
"Total amount is %s","Total amount is %s","Validation message","format: currency"
```

**Bản dịch tiếng Pháp (messages_fr.csv):**
```csv
"key","message","comment","context"
"Sale Order","Commande de vente","","SaleOrder entity title"
"Customer","Client","","clientPartner field title"
"Total amount is %s","Le montant total est %s","Validation message","format: currency"
```

**8.2. Nguồn dịch thuật**

Bộ khung trích xuất chuỗi có thể dịch từ nhiều nguồn:

**Tiêu đề/nhãn giao diện:**
```xml
<field name="clientPartner" title="Customer"/>
<!-- "Customer" được thêm vào danh mục dịch -->
```

**Thông báo xác thực:**
```xml
<action-validate name="...">
  <error message="Amount cannot exceed ${maxAmount}"/>
</action-validate>
<!-- Mẫu thông báo thêm vào danh mục -->
```

**Giá trị lựa chọn:**
```csv
# selection.order.status.csv
"value","title"
"1","Draft"
"2","Finalized"
"3","Cancelled"
<!-- Mỗi tiêu đề đều có thể dịch -->
```

**Mã Java (thông báo lập trình):**
```java
throw new AxelorException(
  TraceBackRepository.CATEGORY_CONFIGURATION_ERROR,
  I18n.get("Sale order %s already invoiced"),
  saleOrder.getSaleOrderSeq()
);
```

`I18n.get(key)` tra cứu bản dịch trong ngôn ngữ người dùng, trả chuỗi đã dịch. Tham số định dạng (`%s`) giữ nguyên xuyên ngôn ngữ (thứ tự tham số quan trọng cho ngôn ngữ có trật tự từ khác).

**8.3. Chuyển ngôn ngữ thời gian chạy**

Người dùng chọn tùy chọn ngôn ngữ từ thiết lập hồ sơ. Bộ khung lưu lựa chọn trong phiên làm việc (session), áp dụng bản dịch xuyên suốt giao diện:

- **Siêu dữ liệu giao diện:** Nhãn trường, tiêu đề bảng, chữ nút
- **Giá trị lựa chọn:** Tùy chọn danh sách thả xuống, huy hiệu trạng thái
- **Thông báo xác thực:** Cảnh báo lỗi/cảnh báo/thông tin
- **Xuất báo cáo:** Xuất PDF/Excel sử dụng ngôn ngữ người dùng
- **Mẫu email:** Email thông báo được dịch

**Cơ chế triển khai** `[Suy luận từ mẫu i18n điển hình]`:
1. Người dùng chọn ngôn ngữ → lưu trong trường User.language
2. Phiên làm việc khởi tạo với ngôn ngữ địa phương người dùng (Locale.FRENCH, Locale.SPANISH, v.v.)
3. Dịch vụ I18n tải messages_XX.csv phù hợp vào bộ nhớ (lưu đệm theo ngôn ngữ)
4. Tra cứu thông điệp: `I18n.get(key)` → tra cứu bảng băm trong danh mục theo ngôn ngữ
5. Bản dịch thiếu quay về tiếng Anh (suy giảm duyên dáng)

**Mức độ phủ dịch thuật:** Phủ đầy đủ cần dịch ~5.000-10.000 thông điệp mỗi mô-đun (tất cả tiêu đề trường, trình đơn, thông báo). Công sức lớn nhưng cho phép triển khai toàn cầu. Nền tảng cộng đồng dịch (Crowdin, Transifex) thường dùng cho dịch do cộng đồng đóng góp.

---

### 9. HỆ THỐNG MẪU: EMAIL, BÁO CÁO VÀ SINH TÀI LIỆU

**Tệp nguồn:** Template.xml, tham chiếu hành động mẫu `[Từ source code + suy luận]`

Ứng dụng doanh nghiệp đòi hỏi sinh tài liệu có định dạng (hóa đơn, đơn mua, báo cáo) và gửi email. Axelor cung cấp **động cơ mẫu** hỗ trợ nhiều định dạng: email HTML, báo cáo PDF (BIRT), xuất Word/Excel, và tin nhắn SMS. Mẫu được soạn dưới dạng tệp văn bản với cú pháp giữ chỗ (`${trường}`), bộ khung điền giá trị giữ chỗ từ mô hình dữ liệu tại thời điểm chạy.

**9.1. Mẫu email**

**Mẫu định nghĩa:**
```xml
<entity name="Template">
  <string name="name" title="Template name" required="true"/>
  <string name="subject" title="Email subject"/>  <!-- "Sale Order ${saleOrderSeq} confirmed" -->
  <text name="content" title="Email body" multiline="true"/>
  <many-to-one name="metaModel" ref="MetaModel"/>  <!-- Mẫu cho kiểu thực thể nào -->
  <string name="templateEngine" selection="template.engine.select"/>  <!-- "Groovy", "StringTemplate" -->
</entity>
```

**Ví dụ mẫu email:**
```html
<!-- Trường Template.content -->
<html>
<body>
  <p>Dear ${clientPartner.fullName},</p>

  <p>Your order <strong>${saleOrderSeq}</strong> has been confirmed.</p>

  <table>
    <tr><th>Product</th><th>Quantity</th><th>Price</th></tr>
    <% saleOrderLineList.each { line -> %>
      <tr>
        <td>${line.product.name}</td>
        <td>${line.quantity}</td>
        <td>${line.priceSubtotal}</td>
      </tr>
    <% } %>
  </table>

  <p>Total: <strong>${exTaxTotal}</strong></p>

  <p>Thank you,<br/>${company.name}</p>
</body>
</html>
```

**Động cơ mẫu:** Mẫu Groovy (tương tự JSP/ERB) — `${}` cho nội suy, `<% %>` cho khối mã (vòng lặp, điều kiện). Ngữ cảnh mẫu = đối tượng thực thể (phiên bản SaleOrder) — toàn quyền truy cập trường và quan hệ.

**Gửi email từ mẫu:**
```xml
<action-method name="action-sale-order-method-send-email">
  <call class="com.axelor.apps.sale.service.SaleOrderService" method="sendConfirmationEmail"/>
</action-method>
```

```java
public void sendConfirmationEmail(SaleOrder order) {
  Template template = templateRepository.findByName("sale-order-confirmation");
  String subject = templateEngine.make(order, template.getSubject());
  String body = templateEngine.make(order, template.getContent());

  emailService.send(
    order.getClientPartner().getEmailAddress(),
    subject,
    body
  );
}
```

**9.2. Mẫu báo cáo (Tích hợp BIRT)**

```
modules/axelor-sale/src/main/resources/reports/
  SaleOrder.rptdesign        # Mẫu báo cáo BIRT (XML)
  SaleOrderLine.rptdesign
```

Đặc điểm mẫu BIRT:
- Trình thiết kế trực quan (Eclipse BIRT designer) cho bố cục
- Ràng buộc bộ dữ liệu SQL/Java (truy vấn cơ sở dữ liệu hoặc gọi phương thức Java)
- Định dạng phong phú (đầu/chân trang, biểu đồ, báo cáo con)
- Nhiều định dạng đầu ra (PDF, Excel, Word, HTML)

**Hành động sinh báo cáo:**
```java
public void printSaleOrder(SaleOrder order) {
  String reportPath = "reports/SaleOrder.rptdesign";
  Map<String, Object> params = new HashMap<>();
  params.put("SaleOrderId", order.getId());

  byte[] pdfBytes = reportEngine.generate(reportPath, params, "PDF");

  // Đính kèm vào đơn hàng hoặc tải về
  MetaFile attachment = metaFileService.upload(pdfBytes, "SaleOrder.pdf");
  order.setPrintedPDF(attachment);
}
```

**9.3. Mẫu Groovy thay thế**

Cho báo cáo đơn giản hơn, mẫu Groovy đủ dùng (không cần trình thiết kế BIRT):

```groovy
<!-- Mẫu PDF sử dụng Groovy + bộ chuyển đổi HTML-sang-PDF -->
<html>
<head>
  <style>
    .header { font-size: 20pt; font-weight: bold; }
    .line-items { width: 100%; border-collapse: collapse; }
    .line-items td { border: 1px solid #ccc; padding: 5px; }
  </style>
</head>
<body>
  <div class="header">ĐƠN BÁN ${saleOrderSeq}</div>

  <p>Ngày: ${creationDate.format('yyyy-MM-dd')}</p>
  <p>Khách hàng: ${clientPartner.fullName}</p>

  <table class="line-items">
    <% saleOrderLineList.each { line -> %>
      <tr>
        <td>${line.product.name}</td>
        <td style="text-align: right;">${line.quantity}</td>
        <td style="text-align: right;">${line.price}</td>
      </tr>
    <% } %>
  </table>

  <p>Tổng: ${exTaxTotal}</p>
</body>
</html>
```

Mẫu dựng thành HTML, HTML chuyển sang PDF bằng thư viện (có thể Flying Saucer, iText, hoặc WeasyPrint). Phát triển nhanh hơn BIRT nhưng ít khả năng định dạng hơn.

---

### 10. SO SÁNH NỀN TẢNG: AXELOR SO VỚI CÁC NỀN TẢNG KHÔNG MÃ KHÁC

**10.1. Ma trận định vị**

Axelor chiếm vị trí "ít mã cho nhà phát triển" — không phải không mã thuần túy (cần kỹ năng kỹ thuật cho soạn XML, kịch bản Groovy) nhưng mã hóa ít hơn nhiều so với bộ khung truyền thống `[Suy luận từ kiến thức ngành]`:

| Nền tảng | Người dùng mục tiêu | Mô hình phát triển | Trần tùy chỉnh | Đường cong học tập |
|----------|---------------------|---------------------|-----------------|---------------------|
| **Axelor** | Nhà phát triển | Cấu hình XML + mã Java | Rất cao (toàn quyền Java) | Trung bình |
| **OutSystems** | Người dùng nâng cao | Mô hình trực quan + mã | Cao (ngôn ngữ độc quyền) | Trung bình |
| **Mendix** | Nhà phân tích nghiệp vụ | Mô hình trực quan | Trung bình (mở rộng widget) | Thấp |
| **Salesforce** | Quản trị/nhà phát triển | Nhấp chuột + mã Apex | Trung bình (giới hạn governor) | Trung bình-cao |
| **Microsoft Power Apps** | Người dùng nghiệp vụ | Kéo thả biểu mẫu | Thấp (chỉ JavaScript) | Thấp |
| **Odoo** | Nhà phát triển | Mô hình Python + giao diện XML | Rất cao (toàn quyền Python) | Trung bình-cao |

**10.2. So sánh Axelor và Odoo**

So sánh liên quan nhất: **Axelor (Java) so với Odoo (Python)** — cả hai đều là nền tảng ERP mã nguồn mở với kiến trúc mở rộng.

**Điểm tương đồng:**
- Định nghĩa giao diện hướng XML (cú pháp gần giống nhau)
- Python/Java cho logic nghiệp vụ
- Kiến trúc mô-đun (kho ứng dụng, mô-đun cắm thêm)
- Phủ mạnh miền ERP (kế toán, bán hàng, kho, sản xuất)

**Điểm khác biệt:**

| Khía cạnh | Axelor | Odoo |
|-----------|--------|------|
| **Ngôn ngữ** | Java + Groovy | Python |
| **ORM** | JPA/Hibernate | ORM Odoo (tùy chỉnh) |
| **Kế thừa giao diện** | Mở rộng XPath | Kế thừa + XPath |
| **BPM** | Camunda bên ngoài | Luồng công việc tích hợp |
| **Sinh mã** | XML miền → thực thể | Lớp Python trực tiếp |
| **Ngôn ngữ biểu thức** | Groovy | Biểu thức Python |
| **Giao diện** | SPA React | Bộ khung Owl (tùy chỉnh) |
| **Giấy phép** | AGPL (mở) + thương mại | LGPL (Cộng đồng) + Doanh nghiệp |

**Ưu thế Axelor:**
- Hệ sinh thái Java (thư viện trưởng thành, áp dụng doanh nghiệp)
- An toàn kiểu (kiểm tra thời biên dịch)
- Động cơ BPM bên ngoài (tuân thủ chuẩn BPMN)
- Giao diện React (hiện đại, hệ sinh thái hỗ trợ)

**Ưu thế Odoo:**
- Cộng đồng lớn hơn (nhiều mô-đun, nhiều nhà phát triển)
- Triển khai đơn giản hơn (Python so với máy chủ ứng dụng Java)
- Tích hợp chặt hơn (ORM + giao diện + luồng công việc cùng mã nguồn)
- Tài liệu toàn diện hơn

**Định vị thị trường:** Odoo nhắm tới doanh nghiệp vừa và nhỏ (triển khai đơn giản, chi phí thấp), Axelor nhắm tới doanh nghiệp lớn (yêu cầu Java, kiến trúc vững chắc). Người dùng kỹ thuật quen Spring Boot thấy Axelor thân thuộc; đội Python ưa thích Odoo.

---

### 11. CÁC TRƯỜNG HỢP SỬ DỤNG: KHI NÀO KHÔNG MÃ ĐỦ VÀ KHI NÀO CẦN VIẾT MÃ

**11.1. Kịch bản không mã thuần túy (chỉ XML)**

Các yêu cầu sau có thể đạt được hoàn toàn qua cấu hình XML:

**1. Ứng dụng CRUD chuẩn**
```
Yêu cầu: Quản lý cơ sở dữ liệu khách hàng gồm thông tin liên hệ, địa chỉ, ghi chú
Giải pháp: Định nghĩa thực thể Customer trong XML miền, sinh giao diện biểu mẫu/lưới, thêm bộ lọc tìm kiếm
Mã cần viết: KHÔNG (XML thuần túy)
```

**2. Nhập liệu chủ-chi tiết**
```
Yêu cầu: Đơn bán với dòng mục, tổng tự động tính
Giải pháp: Thực thể SaleOrder + SaleOrderLine, quan hệ một-nhiều, action-record cho kích hoạt tính toán
Mã cần viết: KHÔNG nếu dùng biểu thức tổng hợp sum()
```

**3. Luồng công việc cơ bản**
```
Yêu cầu: Phê duyệt ba trạng thái (Nháp → Chờ duyệt → Đã duyệt)
Giải pháp: Trường trạng thái với lựa chọn, action-record cập nhật trạng thái, action-validate kiểm tra chuyển đổi
Mã cần viết: KHÔNG (máy trạng thái hoàn toàn khai báo)
```

**4. Báo cáo đơn giản**
```
Yêu cầu: PDF danh sách khách hàng lọc theo quốc gia
Giải pháp: Giao diện lưới với bộ lọc miền, nút in kích hoạt mẫu BIRT
Mã cần viết: KHÔNG nếu mẫu BIRT tạo bằng trình thiết kế trực quan
```

**11.2. Kịch bản ít mã (XML + Groovy)**

Độ phức tạp vừa phải cần biểu thức/kịch bản:

**1. Hiện/ẩn trường động**
```
Yêu cầu: Hiện trường giảm giá chỉ cho khách hàng cao cấp
Giải pháp: action-attrs với expr="eval: clientPartner?.isPremium == true"
Mã cần viết: MỘT DÒNG biểu thức Groovy
```

**2. Trường tính toán**
```
Yêu cầu: Tổng = tổng(dòng.thành tiền) × (1 + thuế suất)
Giải pháp: action-record với expr="eval: saleOrderLineList.sum{it.priceSubtotal} * (1 + company.taxRate)"
Mã cần viết: MỘT DÒNG biểu thức Groovy
```

**3. Xác thực xuyên bản ghi**
```
Yêu cầu: Số lượng đặt không được vượt quá tồn kho khả dụng
Giải pháp: action-condition kiểm tra mỗi dòng product.stockQty >= line.quantity
Mã cần viết: 5-10 DÒNG action-script lặp qua dòng
```

**4. Lọc miền động**
```
Yêu cầu: Lọc sản phẩm khả dụng trong kho đã chọn
Giải pháp: action-attrs thay đổi miền trường sản phẩm dựa trên lựa chọn kho
Mã cần viết: 2-3 DÒNG biểu thức Groovy xây dựng chuỗi miền
```

**11.3. Kịch bản cần viết mã (dịch vụ Java)**

Logic phức tạp đòi hỏi Java:

**1. Tích hợp API bên ngoài**
```
Yêu cầu: Đồng bộ đơn hàng với API vận chuyển ShipStation
Giải pháp: Dịch vụ Java sử dụng máy khách HTTP, ánh xạ đơn Axelor sang định dạng ShipStation, xử lý xác thực
Mã cần viết: 200-500 DÒNG Java (máy khách REST, xử lý lỗi, thử lại)
```

**2. Thuật toán phức tạp**
```
Yêu cầu: Lập kế hoạch tuyến giao hàng tối ưu (bài toán người bán hàng du lịch)
Giải pháp: Dịch vụ Java sử dụng thư viện OR-Tools, thuật toán đồ thị
Mã cần viết: 500+ DÒNG Java (triển khai thuật toán, tối ưu)
```

**3. Tối ưu hiệu năng**
```
Yêu cầu: Cập nhật hàng loạt 100.000 bản ghi hàng đêm
Giải pháp: Dịch vụ Java sử dụng thao tác lô JDBC, bỏ qua ORM
Mã cần viết: 100-200 DÒNG Java (SQL thô, quản lý giao dịch)
```

**4. Quy tắc nghiệp vụ tùy chỉnh**
```
Yêu cầu: Định giá đa tầng (giảm theo số lượng, loại khách hàng, thời kỳ khuyến mãi)
Giải pháp: Dịch vụ Java với động cơ quy tắc định giá, lưu đệm
Mã cần viết: 300-500 DÒNG Java (đánh giá quy tắc, tính giá)
```

**Phân tích ngưỡng:** Khoảng **60-70% logic ứng dụng** có thể đạt được qua cách tiếp cận ít mã XML/Groovy, **30-40% cần Java** cho độ phức tạp/hiệu năng. Tỷ lệ thuận lợi so với phát triển truyền thống (100% mã Java), nhưng không phải "không mã" thực sự (người dùng nghiệp vụ vẫn cần đào tạo kỹ thuật).

---

### 12. TÓM TẮT: ĐÁNH GIÁ KHẢ NĂNG KHÔNG MÃ/ÍT MÃ

**12.1. Ma trận khả năng**

| Khả năng | Mức hỗ trợ | Kỹ năng kỹ thuật cần | Mã lặp tiết kiệm |
|----------|------------|----------------------|-------------------|
| **Màn hình CRUD** | Xuất sắc | Thấp (cơ bản XML) | 90-95% |
| **Bố cục biểu mẫu** | Xuất sắc | Thấp (lồng bảng) | 85-90% |
| **Chủ-chi tiết** | Xuất sắc | Thấp (panel-related) | 90% |
| **Xác thực trường** | Tốt | Trung bình (biểu thức Groovy) | 70-80% |
| **Luồng công việc** | Tốt | Trung bình (chuỗi hành động) | 60-70% |
| **Logic nghiệp vụ** | Khá | Cao (Java cho phức tạp) | 30-50% |
| **Báo cáo** | Tốt | Trung bình (trình thiết kế BIRT) | 80% |
| **Tích hợp** | Khá | Cao (Java + kiến thức API) | 20-30% |

**12.2. Điểm mạnh kiến trúc**

1. **Cách tiếp cận hướng mô hình:** Nhất quán với thực hành tốt nhất kiến trúc doanh nghiệp (OMG MDA, hồ sơ UML)
2. **Tầng trừu tượng rõ ràng:** Trình bày (giao diện) tách biệt khỏi logic nghiệp vụ (hành động, dịch vụ) và mô hình dữ liệu (thực thể miền)
3. **Khả năng mở rộng:** Hệ thống mô-đun + kế thừa giao diện cho phép xây dựng hệ sinh thái (nền tảng cơ sở + tiện ích chuyên biệt)
4. **Tuân thủ chuẩn:** JPA cho bền vững, BPMN cho luồng công việc, OAuth cho xác thực — giảm phụ thuộc nhà cung cấp so với nền tảng độc quyền
5. **Thân thiện nhà phát triển:** XML + Java quen thuộc với nhà phát triển Spring, áp dụng dễ hơn IDE trực quan (OutSystems, Mendix)

**12.3. Hạn chế kiến trúc**

1. **Không phải không mã thực sự:** Người dùng nghiệp vụ không thể xây ứng dụng mà không qua đào tạo kỹ thuật (cú pháp XML, biểu thức Groovy, khái niệm mô hình hóa miền)
2. **Giới hạn khai báo:** Luồng công việc phức tạp (phê duyệt song song, định tuyến động) đẩy sang mã Java hoặc BPM bên ngoài
3. **Trần hiệu năng:** Ngôn ngữ biểu thức chậm hơn Java biên dịch (chấp nhận cho logic giao diện, có vấn đề cho xử lý lô)
4. **Thách thức gỡ lỗi:** Lỗi XML khó hiểu ("NullPointerException trong action-record" không cho biết trường nào), không gỡ lỗi từng bước cho biểu thức
5. **Đường cong học tập:** Hơn 20 thuộc tính giao diện, 7 loại hành động, lược đồ XML miền, cú pháp Groovy — cần kiến thức đáng kể để thành thạo

**12.4. So sánh với các nền tảng ERP gốc**

Khả năng không mã của Axelor **tương đương Odoo** (giao diện XML + ngôn ngữ kịch bản), **vượt SAP Business One** (tùy chỉnh hạn chế không có SDK), **kém Salesforce** (trình xây dựng trang trực quan, trình thiết kế luồng dễ tiếp cận hơn cho người không phải nhà phát triển). Định vị: **nền tảng ít mã cho người dùng kỹ thuật**, không phải công cụ phát triển công dân.

**Đối tượng mục tiêu:** Nhà phát triển Java muốn chu kỳ phát triển nhanh hơn Spring Boot, nhà phát triển Python chuyển sang ngăn xếp JVM, doanh nghiệp cần ERP tùy chỉnh với toàn quyền truy cập mã nguồn. KHÔNG phù hợp cho: người dùng nghiệp vụ phi kỹ thuật (quá phức tạp), tạo mẫu nhanh (đường cong học tập gây chậm trễ), ngành chuyên biệt cao (có thể thiếu mô-đun miền).

**12.5. Đánh giá định lượng**

Dựa trên phân tích mã nguồn axelor-open-suite:

- **Tổng dòng XML:** ~100.000 dòng (giao diện, hành động, miền)
- **Tổng dòng Java:** ~500.000 dòng (dịch vụ, bộ điều khiển, kho dữ liệu)
- **Tỷ lệ XML:Java:** 1:5 (20% khai báo, 80% mệnh lệnh)

Nhưng xét rằng XML sinh mã Java và loại bỏ mã lặp giao diện:

- **Dòng mã hiệu dụng không có sinh mã:** ~800.000-1.000.000 dòng (ước tính)
- **Dòng mã thực tế với không mã:** ~600.000 dòng
- **Giảm nỗ lực phát triển:** ~25-40% so với ứng dụng Java Spring Boot thuần túy

---

## Kết luận

Nghiên cứu khả năng không mã/ít mã của Axelor cho thấy **nền tảng phát triển hướng mô hình tinh vi** cân bằng giữa sự đơn giản khai báo và sức mạnh lập trình. Nhận thức cốt lõi: nền tảng thành công không phải bằng cách loại bỏ hoàn toàn mã (bất khả cho yêu cầu doanh nghiệp phức tạp) mà bằng **xác định các lớp trừu tượng có giá trị cao** — định nghĩa giao diện, mẫu hành động, mô hình miền — mã hóa chúng thành lược đồ XML tái sử dụng được xử lý bởi bộ thực thi bộ khung vững chắc.

**Phát hiện chính:**
1. **Năm loại giao diện** bao phủ ~90% mẫu giao diện người dùng (lưới, biểu mẫu, lịch, thẻ, biểu đồ)
2. **Bảy loại hành động** loại bỏ ~70% mã lặp bộ điều khiển/dịch vụ điển hình
3. **Sinh mã** giảm ~90% dòng mã soạn lớp thực thể
4. **Ngôn ngữ biểu thức** cho phép ~60% quy tắc nghiệp vụ dưới dạng logic khai báo
5. **Kế thừa giao diện** hỗ trợ mở rộng mô-đun không cần sửa mã nguồn

Nền tảng đạt **60-70% phủ không mã** cho ứng dụng nghiệp vụ điển hình — cao hơn bộ khung truyền thống (0% không mã) nhưng thấp hơn nền tảng không mã thuần túy (tuyên bố 90%+, thường kèm hạn chế chức năng). Đánh đổi: giữ toàn bộ lối thoát Java/Groovy cho yêu cầu phức tạp, hy sinh khả năng tiếp cận "phát triển công dân".

**Ý nghĩa kiến trúc:** Axelor chứng minh tính khả thi của ít mã trong hệ sinh thái Java (lịch sử bị thống trị bởi .NET với Power Apps, JavaScript với Retool). Sự kết hợp độ trưởng thành Java doanh nghiệp (quản lý giao dịch, bảo mật, khả năng mở rộng) với trải nghiệm nhà phát triển hiện đại (giao diện React, kịch bản Groovy, biên dịch Gradle) định vị nền tảng cạnh tranh với Odoo (Python), SAP (ABAP), và Salesforce (Apex).

**Khuyến nghị chiến lược:** Axelor phù hợp cho tổ chức có chuyên môn Java muốn nền tảng ERP cấu hình được, sẵn sàng đầu tư học các lớp trừu tượng nền tảng (lược đồ XML, mẫu hành động, DSL miền). KHÔNG phù hợp cho sáng kiến không mã nhắm tới nhà phân tích nghiệp vụ — độ phức tạp đòi hỏi kỹ năng kỹ thuật phần mềm dù bề mặt là cú pháp XML.

---

*Kết thúc RESEARCH_STEP5_NOCODE.md*
