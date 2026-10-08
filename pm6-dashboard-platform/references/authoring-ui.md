# Thao tác khi chỉ có nền tảng đang chạy

Không cần mã nguồn. Dùng browser/app tool được host AI cho phép; quan sát UI hiện tại và label/accessibility thay vì tọa độ hoặc selector cứng. Không gọi API/công cụ giả định không tồn tại.

## Tìm đúng bề mặt

- Quản lý layout: chọn layout đích hoặc tạo mới trong phạm vi yêu cầu. Editor thường là `/cau-hinh-layout` hoặc `/cau-hinh-layout/{id}` dưới context của deployment.
- Các trang/tab nằm trong layout; mỗi trang chứa widgets và source mặc định. Đừng nhầm thao tác chọn portal slot với thêm chart lên canvas.
- Properties có hai tab **Giá trị** và **Định dạng**. Tab Giá trị phục vụ binding và nội dung; tab Định dạng phục vụ trình bày, nhưng một số điều kiện KPI cũng xuất hiện trong panel Giá trị. Không suy tính năng vắng mặt chỉ vì đang ở sai tab/series.
- Một chart trong danh mục không nhất thiết đã có binding của chart đang thả vào trang. Kiểm lại nguồn và cột sau khi thêm.

## Từ tên output đến widget

1. Trong bộ chọn nguồn, xác định database/output; lấy field list và mẫu bằng công cụ được cấp quyền nếu UI có, hoặc API discovery đã xác minh.
2. Tự phân tích dữ liệu/nghiệp vụ theo yêu cầu người dùng rồi chọn trang và visual phù hợp. Không ép layout ba trang, sáu KPI hay bộ màu nhất định.
3. Thêm widget, chọn nó, vào Giá trị. Với chart trục, binding dimension vào X, measure vào Y, group vào Chú thích/Nhóm khi cần; bubble/depth/tooltip có vùng riêng.
4. Đặt phép aggregate theo dữ liệu và implementation đúng loại. Với table dùng cấu hình dimension/metric/cột, không chỉ kéo theo thói quen bar. Với KPI dùng cấu hình main/trend/target riêng.
5. Vào Định dạng để chỉnh giao diện theo thiết kế tự chọn. Nếu combo, chọn series cần chỉnh trước: bar mở cấu hình cột, line mở cấu hình đường.

Đây là bản đồ thao tác, không phải trình tự bắt buộc cho mọi yêu cầu. Với chỉnh nhỏ, chỉ tác động đúng widget/field được giao.

## Các điểm UI dễ gây hiểu lầm

| Quan sát | Kiểm gì trước |
| --- | --- |
| Đổi tên nhưng header không hiện | Header text/show, title legacy, KPI title; tên nội bộ khác text render |
| Màu `<100` không đúng | Loại rule, biên `gt/lte`, giá trị sau aggregate, active combo series, màu element KPI |
| Cột ngưỡng nằm cùng chart | Có phải fixed thresholds hay data series? Bar không tự so với line |
| Đã chọn sort nhưng không đổi | Loại metadata, aggregate, seriesTypes, category ties; xem giới hạn sort |
| Bật scroll nhưng không cuộn | Native direction, eligibility combo, CSS viewport/Y-axis, lượng dữ liệu và range |
| Slicer hiển thị tháng nhưng chart khác | Scope, targetWidgetIds, mapping, filterMode submit, dataset/API và default/TopN |
| Action không chạy trong editor | Thử View; Edit có thể chỉ chọn widget thay vì điều hướng |
| Có menu bookmark | Không đồng nghĩa có bookmark snapshot và handler hoạt động |
| Save xong mobile mất bố cục | Override thiết bị và sidecar, đúng ID sau remap, đọc lại page |

## Lưu và kiểm tra

Commit field đang gõ bằng hành vi UI bình thường (blur/Enter theo control), rồi dùng nút lưu phù hợp. Không ép event JavaScript nội bộ thay cho thao tác nếu host tool không cho phép hoặc chưa hiểu runtime.

Với layout có nhiều trang, source hiện có đường chỉ lưu nội dung trang đang active, sau đó mới lưu metadata layout. Xác nhận đã lưu từng trang thay đổi; đừng dựa vào một toast duy nhất để nói cả dashboard đã lưu.

Mở View bằng link thật sau lưu, kiểm những chart/filter/action đã tác động. Reload hoặc phiên View mới giúp phân biệt state editor/cache với persistence. Không tự đổi quyền/public để mở được View; lỗi quyền phải xử lý bằng quyền được cấp.

Khi không thể lưu hoặc View không khớp, giữ bản thiết kế/config và bằng chứng phần đã thực hiện; báo chính xác “đã cấu hình nhưng chưa xác minh View”, thay vì nói hoàn tất hoặc đổ lỗi backend không có bằng chứng.
