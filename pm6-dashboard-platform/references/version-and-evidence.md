# Phiên bản và bằng chứng

## Phạm vi kiến thức

- Skill: `pm6-dashboard-platform`, phiên bản gói 1.1.1. Bản 1.0.0 được rút ra từ source; bản 1.1.0 bổ sung các quan sát trên deployment live và quy trình vận hành; bản 1.1.1 thay quy tắc “tab làm việc riêng” bằng “mọi thao tác trên tab người dùng đang mở, không mở tab riêng”.
- Source frontend: commit `834c199985a3dfbdad2c673438f7bf5ef8479006`, khảo sát 2026-10-08.
- “PM6/IOC” là tên nhận diện nền tảng trong nhiệm vụ này, không phải tuyên bố một API chuẩn chung cho mọi sản phẩm IOC.
- Không kèm source ứng dụng, source backend, thư viện riêng, credentials, URL server riêng hoặc dữ liệu khách hàng.
- Tài liệu mô tả capability/contract đã đọc; các UI/endpoint deployment mới có thể khác. Source reference dưới đây dành cho người bảo trì gói, **không phải prerequisite người dùng skill**.

## Kiểm chứng nền tảng trước khi đóng gói

Kiểm kê 737 file dưới src; đọc sâu các đường dashboard cốt lõi, không tuyên bố đã nghiệm thu mọi chức năng. Baseline trong bản sao cô lập: 188 test pass thuộc 33 file, 3 file không load được do import/dependency ứng dụng. Thêm 12 characterization test pass ghi nhận sort, scroll, query zero, pivot và aggregation—pass nghĩa tái hiện hành vi hiện tại, không nghĩa đã sửa lỗi.

Môi trường cô lập dùng Vitest 2.1.9, Vite 5.4.21, React 18.3.1, ECharts 6.0.0, Zustand 4.5.7 cùng helper dependency; không phải dependency tree production. Chưa build toàn ứng dụng, chưa E2E server/browser; không dùng số test làm chứng nhận full stack.

Các phát hiện bổ sung khi xây skill: header text khác widget title; KPI column comparison có cả UI lẫn runtime; main KPI nhiều yAxis cộng các aggregate; update page có contract riêng và chỉ active page; nguồn và sidecar khác giữa tạo mới/update. Chúng được đọc từ code, chưa nghiệm thu backend.

Khi đóng gói, thêm 9 test với fixture của chính skill gọi code đã khảo sát: bar/combination values và màu, biên >100, raw request, biểu diễn filter, aggregate mẫu KPI, field mẫu table, serialize widget thành chartConfigs và khôi phục sidecar. Cả 9 pass trong môi trường cô lập. Test KPI/table ở đây chỉ xác minh hợp đồng dữ liệu/helper, không render React component. Gói còn được kiểm frontmatter, liên kết reference, JSON và khả năng chuyển thư mục; các kiểm này không thay thế một thử nghiệm AI khác tự thao tác hệ thống live.

## Kiểm chứng live (1.1.0)

Ngày 2026-10-08, skill được dùng để dựng một dashboard 4 trang, 34 widget (KPI, slicer, line, bar, bar_horizontal, pie, table) từ một họ output đầu tư công, trên một deployment PM6/IOC thật. Việc này dùng phiên trình duyệt đã đăng nhập và gọi API ngay trong page context. Bằng chứng gồm ba loại:

- **Response API:** mã 201 hoặc 200, và ID thật trong response.
- **Readback cây layout** sau mỗi lần ghi.
- **Ảnh chụp View** của từng trang. Có đối chiếu số liệu: tổng các nguồn vốn bằng dòng tổng cộng; đổi kỳ ở slicer thì KPI trên các bảng được ánh xạ cũng thay đổi.

Trong phiên, toàn bộ trang mới bị xóa bằng `DELETE /layouts/tree/pages/{id}`, ngay sau khi editor được mở lại và có thao tác trên thanh tab trang. Trang không khôi phục được (404), nên đã phải dựng lại bằng script. Bài học được ghi trong [operating-workflow.md](operating-workflow.md): snapshot trước khi ghi, tải lại editor, làm trên một tab duy nhất và nạp lại helper sau mỗi lần điều hướng.

Các quan sát chi tiết nằm trong ba file:

- [persistence-api.md](persistence-api.md#quan-sát-trên-deployment-live)
- [widget-contracts.md](widget-contracts.md)
- [data-and-filters.md](data-and-filters.md#nối-slicer-với-widget-quan-sát-live)

Gói không chứa host, ID, token hay dữ liệu của deployment đó.

**Chưa kiểm:**

- Mobile và tablet layout.
- Portal.
- Chia sẻ.
- Combo chart.
- Quy tắc tô màu có điều kiện của bảng.
- Việc queryFilters có được đẩy xuống backend thành whereConditions ở luồng tương tác hay không.

## Bản đồ source cho người bảo trì

Các đường sau tương đối với repo gốc, không liên kết file bắt buộc trong gói. Prefix `B` chỉ thư mục `src/modules/layout-config/ui/screens/builder`.

| Nội dung | Source chứng cứ |
| --- | --- |
| Route web/V1/mobile/portal | src/routes/index.tsx, src/routes/path.ts, src/modules/layout-view/router.tsx |
| Quyền frontend / token | src/hook/useRoute.tsx, src/apis/axiosClient.tsx, layout-view-mobile/infra/embedAuth.ts |
| Types, binding, format | B/types.ts, B/WidgetProperties.tsx, B/properties/component/ChartValuesPanel.tsx |
| Source discovery và request | B/infra/index.ts, B/domain/index.ts, B/utils/queryBuilder.ts, B/hooks/usePageDataLoad.ts |
| Filter và TopN | B/store/useFilterStore.ts, filterResolvers.ts, B/charts/SlicerWidget.tsx, B/charts/components/SlicerFilterItem.tsx |
| Renderer/dispatch | B/ChartRenderer.tsx, B/utils/chartOptionsFactory.ts, B/utils/chartRegistry.ts |
| Màu/sort/annotation | B/utils/factories/BarFactory.ts, ComboFactory.ts, factoryHelpers.ts, B/utils/widgetCalculations.ts |
| Header và KPI | B/charts/WidgetHeader.tsx, KPIWidget.tsx, B/properties/value/KPIValuePanel.tsx |
| Table/matrix | B/charts/TableWidget.tsx, MatrixWidget.tsx, B/utils/pivotUtils.ts |
| Save/readback | B/DashboardBuilder.tsx, B/infra/layoutService.ts, B/utils/layoutMapper.ts, layoutSidecars.ts |
| Responsive/visibility | B/utils/responsive/*, B/utils/pageActiveMap.ts, pageItemVisibilityMap.ts |
| Portal | src/modules/dashboard-config/infra/index.ts, src/modules/dashboard-view/ui/screens/DashboardViewScreen.tsx |

## Cập nhật khi platform thay đổi

Đối chiếu UI/response hợp lệ ở môi trường đích với schema/behavior tương ứng. Một endpoint trả 404 không có nghĩa tự thay v1 bằng v2 cho mọi ghi; một lỗi đã được sửa ở deployment không còn là lệnh cấm sử dụng tính năng. Giữ tên phiên bản/bằng chứng rõ ràng.

Người bảo trì có source nên cập nhật đúng reference và examples, chạy kiểm cấu trúc gói và fixture trước khi phát hành phiên bản mới. Không cần nhúng toàn repo vào skill. Người nhận chỉ cần gói này cùng quyền/công cụ truy cập nền tảng; nếu gặp điểm chưa được mô tả, dùng introspection UI/export/API được cấp quyền, không bịa nội hàm.
