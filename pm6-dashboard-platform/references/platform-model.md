# Mô hình nền tảng

## Phân biệt đối tượng

| Đối tượng | Ý nghĩa | Không nhầm với |
| --- | --- | --- |
| Output/table | Bảng hoặc output trong database được cấp quyền | Schema nghiệp vụ cố định có thể đoán từ tên |
| Chart danh mục | Bản ghi chart gắn loại/nguồn, có chart ID | Một vị trí trên canvas |
| Widget | Thành phần đang render: chart, KPI, slicer, table, text, media, group... | JSON option thuần của thư viện vẽ |
| PageItem | Widget hoặc group/iframe ở một vị trí trên trang, có ID riêng | `chartInfo.id` |
| Page | Trang/tab trong layout, có `pageIndex`, source mặc định và pageItems | Trang web/route hoặc một portal slot |
| Layout | Bộ trang/canvas cùng theme, dimension và filter config | Portal dashboard |
| Dashboard portal | Dashboard quản trị chứa cây monitoring-info liên kết layout, có chia sẻ riêng | Layout đang soạn trong editor |

Một yêu cầu thông thường “vẽ dashboard từ output” thường cần tạo/chỉnh **layout và các trang**. Chỉ thêm vào portal khi có đích/nhu cầu; không tự lập dashboard chia sẻ công khai.

## Các lớp cấu hình

Trong editor, widget có hình dạng:

```text
widget
  id, key, realChartId?, type, title
  dataConfig          ← nguồn và binding/query
  formatConfig        ← trình bày/tương tác, cấu hình KPI/slicer/table
  pageItemConfigs     ← vị trí/visibility/override thiết bị
  children?, layout?  ← group
```

`dataConfig`: databaseId, tableName, xAxis, yAxis, groupBy, seriesTypes, seriesAxes, alias, màu, dimensions/metrics, queryFilters.

`formatConfig`: header/title, container, axes, legend, dataLabels, tooltip, sorting, bar/line, seriesFormat, kpi, slicer, table, scrolling, seriesAction.

Khi lưu CHART, nguồn nằm ở `pageItem.chartInfo.databaseId/tableName`; `chartInfo.chartConfigs` chứa dataConfig đã trải phẳng cộng `formatConfig`. **Không phải** `{dataConfig:{...},formatConfig:{...}}` trong chartConfigs. Trình tạo mới có thể còn lưu bản sao source bên trong chartConfigs; đường update có thể loại bản sao này. Bảo toàn cấu trúc hiện có, không xóa nguồn chartInfo.

`widget.title`/`chartInfo.name` là tên đối tượng, không tự bảo đảm header nhìn thấy. Shared header dùng `formatConfig.header.text`, fallback `formatConfig.title.text`; cờ `header.show` ưu tiên `title.show`. KPI còn có title riêng trong format.kpi. Xem widget reference trước khi chỉnh tên hoặc font.

## Luồng dữ liệu thực

1. Tải layout/page config, khôi phục widget và các map phụ.
2. Tải metadata cột và raw data theo database/table. Source dùng chung có cache.
3. Hợp nhất runtime filters theo scope, map dimension logic tới physical column; có dataset filtered riêng theo nguồn/trang/widget.
4. Renderer chọn dataset, lọc cục bộ, rồi gọi component/factory. Một số chart tự aggregate frontend, không phải mọi số đều được server aggregate.
5. Sự kiện slicer có thể làm request lại dữ liệu. Query endpoint tổng hợp được chọn khi cả View mode và interactive flag bật; lần tải ban đầu thường raw.

Hệ quả: thêm sort ở request không đảm bảo trục chart giữ thứ tự đó; alias không đổi cột vật lý; filter trên một nguồn không tự lọc nguồn khác khi thiếu mapping; lỗi API có thể bị biểu diễn thành mảng rỗng. Luôn phân biệt “không có dữ liệu” với lỗi/lọc/cache.

## View, Edit, mobile

- Web View V2: tải metadata layout trước rồi page detail khi mở trang hoặc drill-down. Thiếu pageItems trong metadata là bình thường, **không được lưu ngược thành trang rỗng**.
- View V1 còn tồn tại, tải cây đầy đủ. Một layout có thể có nhiều đường xem khác nhau.
- Edit có state và buffer form; trước save có bước flush. Đổi giá trị ở input chưa chắc đã phản ánh trong payload nếu công cụ chưa commit field.
- Mobile embed có ticket/token/refresh riêng và cùng view-model nền; không coi route public là quyền dữ liệu vô danh.
- Portal View tổ chức Basic/Complex và các mục navigation liên kết layout. Đó không phải nơi cấu hình chi tiết từng bar.

## Bố cục và map phụ

Desktop canvas hiện dùng tọa độ/kích thước pixel trong hệ tọa độ thiết kế: `x,y,w,h`, rồi scale khi hiển thị. Không áp `w=6,h=4` như lưới 12 cột vào mọi canvas desktop chỉ vì tên type là LayoutItem.

Tablet/mobile dùng stack 12 cột: `order`, `colSpan` (1..12), `height` pixel, có thể `width` pixel; có hỗ trợ dữ liệu rectangle cũ `{x,y,w,h}` được migrate. Mobile có thể kế thừa thứ tự tablet rồi thu gọn. Override đã lưu được ưu tiên, không phải lúc nào đổi desktop cũng tái bố trí mobile.

Các cấu hình cần giữ khi đọc/chỉnh/lưu:

- `pageItemConfigs.responsiveLayouts`, `visibility` theo desktop/tablet/mobile.
- `layoutConfigs.responsiveLayoutsMap[pageKey][itemKey]`, `pageItemVisibilityMap[pageKey][itemKey]`, `pageActiveMap[pageKey]`.
- `layoutConfigs.pageFilterConfigsMap`, dimensions, globalFilterValues và các cấu hình page aspect ratio/zoom hiện có.

Map phụ giúp khôi phục field backend có thể không giữ trực tiếp ở page/item. Responsive và visibility per-item ưu tiên bản map nếu còn; pageActiveMap ưu tiên page.active. Page key khi lưu có thể là fakeId hoặc id; khi đọc thử id và fakeId. Item key thường ưu tiên pageItem.id → chartInfo.id → key. Không tự đổi hệ key trên layout đã tồn tại.

Group có children và layout tương đối; API có `type: GROUP`, groupConfigs và children array, đồng thời còn dạng legacy groupId/groupConfig trong chartConfigs. Việc flatten tọa độ group, ID remap, header slicer và targetWidgetIds liên quan nhau. Với nhóm có sẵn, sửa qua UI hoặc giữ cấu trúc readback thay vì dựng lại group từ tên.

## Ranh giới công nghệ

Frontend dựa React/TypeScript; biểu đồ mặc định ECharts, một số nhóm qua ZingChart, `plotly_contour` qua Plotly; KPI/table/matrix/slicer/media là component riêng. Không lấy khả năng của thư viện gốc làm khả năng chắc chắn của UI PM6. `@fis/shared-ui` và backend không nằm trong gói kiến thức; người dùng skill không cần cài chúng để thao tác hệ thống đã triển khai.
