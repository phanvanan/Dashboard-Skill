# Khả năng và giới hạn theo bản nguồn

Phạm vi: commit `834c199985a3dfbdad2c673438f7bf5ef8479006`, ngày khảo sát 2026-10-08. Deployment khác có thể đã sửa. **CODE** = đọc đường thực thi, **TEST** = gọi hàm nguồn trong môi trường cô lập; không tương đương E2E production. Những nhận định chưa có live proof không được dùng để khẳng định nguyên nhân duy nhất của lỗi người dùng.

## Sort

**TEST:** Chart sort lấy `seriesTypes[sortField] || sum` ở nhánh aggregate. Combo dùng giá trị bar/line, helper aggregation không hiểu và trả 0, dẫn đến category không đổi khi metadata không rơi vào nhánh number.

**TEST:** Metadata `type: number` lại chọn nhánh lấy dòng đầu trong category và numeric string comparison. A có 2 + 20, B có 9: sum A=22 nhưng descending sort có thể cho B trước A. Không mô tả đây là sort aggregate đúng.

**TEST:** `sortBy: none` vẫn natural-sort category trong nhiều chart, riêng waterfall có ngoại lệ. Order by ở SQL/API không đảm bảo thứ tự giữ nguyên sau frontend.

**TEST:** Khi sort thành công, tất cả series dùng category order mới. Sort QCVN05 thì QCVN23 và cột đi theo category; ngưỡng bằng nhau không tạo một thứ tự thứ cấp nghiệp vụ được bảo đảm.

Ứng xử: không cấm mọi sort. Kiểm dữ liệu có bao nhiêu dòng/category, metadata và loại chart. Nếu cần thứ tự chưa diễn đạt được, mô tả giới hạn hoặc đề xuất cách khác đã kiểm; không hứa “chỉ thêm ORDER BY” luôn giải quyết.

## Màu và ngưỡng

**CODE + test factory hiện có:** Bar hỗ trợ colorPieces; combo chỉ áp resolver đó cho bar-series. Bật màu mà chưa có khoảng có thể gradient ở bar, nhưng combo không triển khai gradient ấy. Khoảng `>gt, ≤lte`, đầu khớp thắng.

**CODE:** Bar colorByValue không tra cột ngưỡng khác theo từng row. `thresholds.value` là một giá trị số, không phải tên field để dereference. Không nhầm với `KPI conditionalFormatting` có nhánh so aggregate cột.

**CODE:** KPI hỗ trợ rule tĩnh hoặc `valueType: column`/compareField/compareAggregation. Custom mainValue element.style.color có thể che màu rule. Không kết luận KPI không hỗ trợ so cột chỉ vì bar không hỗ trợ.

**CODE:** MarkPoint `name` không luôn thành text label; size/formatter của markPoint không tự lấy từ font header/dataLabels. Muốn đổi đúng phải biết đích text và consumer.

## Cuộn

**TEST:** Native scroll bar thiếu direction → `dataZoom: []`; direction both → bốn control khi không tắt scrollbar.

**TEST:** Combo không thuộc điều kiện sinh native scrolling chung, ngay cả khi direction=both; advanced `format.dataZoom` là đường riêng có thể override, không thuộc test này.

**CODE:** UI section chung có enabled/type/itemSize nhưng không có field direction. CSS scroll có implementation kích thước/overflow/sticky axis, song bị tắt nếu viewport <1024 hoặc Y bị ẩn. Không cam kết scroll một cấu hình chạy giống nhau trên desktop/mobile và mọi engine.

## Filter và Top 1

**TEST:** Query helper serialize numeric 0 thành chuỗi rỗng ở scalar; BETWEEN bỏ biên numeric 0. Chuỗi `"0"` không cùng hành vi. Phân biệt sửa request một lần với fix cấu hình qua toàn bộ vòng tương tác.

**CODE:** TopN resolve chạy trong View và Edit; lưu lựa chọn vào customFields.defaultValue chỉ ở Edit/onUpdate. TopN, defaults khi options ready và parent init có nhiều đường ghi filter, application key không bao phủ mọi phụ thuộc options. Có khả năng ghi đè, nhưng **chưa tái hiện production** tình huống tháng 8/tháng 9.

Ứng xử: đọc config/default/runtime selection, raw rows và nguồn options; kiểm View mới không mở Edit trước. Không nói “phải vào Edit định kỳ” như giải pháp đúng hoặc “View chắc chắn không tính Top1”.

## Matrix và phép tính

**TEST:** Pivot duplicate cell 2 và 9 giữ 9, không sum 11. Chỉ đúng khi upstream đã cho một giá trị/cell hoặc last-row semantics là điều người dùng muốn.

**TEST:** `calculateAggregation(..., distinct_count)` không có implementation và trả 0 ở ca số; API query có mã DISTINCT_COUNT nhưng chưa kiểm backend. Không khẳng định toàn platform không hỗ trợ distinct count, cũng không mặc định mọi chart có hỗ trợ.

**CODE:** Formula khai báo trong MetricConfig chưa đủ để coi có công cụ tính measure chung. Ngưỡng, tỷ lệ, tổng lũy kế và KPI nhiều measure phải đối chiếu với phép tính thật.

## Bookmark, engine và chart nâng cao

**CODE:** Bookmark selector có options rỗng; ECharts/KPI phát event nhưng chưa tìm thấy listener áp snapshot trong frontend. Host ngoài repo có thể bổ sung; cần bằng chứng môi trường đích trước khi dùng.

**CODE:** Globe/map3d/polygons3d factory có series rỗng; calendar hardcode 2025. Catalog/type chỉ là tên không bảo đảm dữ liệu được vẽ đầy đủ.

**CODE:** Plotly chỉ được router chọn cho plotly_contour; nhóm Zing cụ thể gồm pie3D, area3d, line3D_area, stackedArea3d, percentStackedArea3d, bar_horizontal3D và stack/percent, zing_bar3D và stack/percent, line3D. Loại còn lại không tự động dùng engine đúng chỉ vì tên 3D.

## Persistence, data và quyền

**CODE:** Metadata V2 lazy không có tất cả pageItems. Không overwrite page từ metadata shell.

**CODE:** Lưu layout mới gửi cây; update editor lưu page active riêng, rồi metadata layout không kèm layoutPages. Map phụ cần giữ; source từng widget khác source page. Readback mới chứng minh persisted.

**CODE:** Cache/raw load có thể dùng tất cả row và nuốt lỗi thành []; performance/freshness chưa benchmark. Empty chart không đủ bằng chứng output trống.

**CODE:** Route private/permission/mobile embed có cơ chế khác nhau; frontend filter không thay quyền backend. Share/publish là hành động khác tạo chart.

Không buộc mọi yêu cầu làm audit toàn bộ danh sách này. Tra điểm nào ảnh hưởng trực tiếp visual/config đang dùng, xác minh phù hợp rủi ro.
