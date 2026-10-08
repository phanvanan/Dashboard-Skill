# Hợp đồng widget

Tra nhóm cần dùng, không coi đây là danh sách mọi field hợp lệ. Preserve field không biết khi chỉnh một widget tồn tại. Các fragment trong [contracts.json](../examples/contracts.json) là dữ liệu giả để học cấu trúc, không phải preset nghiệp vụ.

## Các trường chung

| Vị trí | Công dụng / lưu ý |
| --- | --- |
| `type` | Mã renderer như bar, line, combo, kpi, table, matrix, slicer; lấy mã đúng từ catalog/export của deployment |
| `title` | Tên widget nội bộ; không thay cho text của header |
| `formatConfig.header` | show, text, containerStyle, textStyle; textStyle.fontSize/color/fontFamily... |
| `formatConfig.title` | Header legacy: show, text, fontSize/color...; tránh để hai giá trị text mâu thuẫn |
| `formatConfig.container` | Nền/viền/padding/shape của card; khác grid nội dung chart |
| `formatConfig.xAxis/yAxis` | Các field được consumer hỗ trợ như show, showLabel, showTitle, title, fontSize, labelRotate; không tự tạo field yAxis2 chỉ vì chart có trục phụ |
| `formatConfig.legend` | show, position, fontSize, alias...; tên legend không thay đổi field binding |
| `formatConfig.dataLabels` | Nhãn giá trị series, không phải title/markPoint |
| `formatConfig.tooltip` | Show/style/format tooltip; tooltipFields chứa binding bổ sung ở dataConfig |
| `formatConfig.numberFormat` | Chuỗi enum auto/none/thousands/millions/billions/vi-* theo consumer; không phải object numberFormat của KPI |
| `formatConfig.sorting` | sortBy (`category`, `value`, physical field, `none`), order asc/desc; xem lỗi sort |
| `formatConfig.seriesAction` | Page/modal/navigation/URL; khả năng tùy component, bookmark chưa đủ |

Typography có nhiều đích: header, trục, legend, nhãn, tooltip, giá trị KPI và markPoint. Không tăng `textStyle.fontSize` toàn cục rồi khẳng định mọi font đều đổi. Header shared được render HTML ngoài option ECharts.

## Bar / line / area

Binding phổ biến: `xAxis: [categoryField]`, `yAxis: [measureField]`; `groupBy` nếu cần tách nhóm. Chart thường có `seriesTypes: {measureField: "sum"}` hoặc avg/count/min/max; không dùng giá trị `bar` cho phép tổng hợp chart thường.

Mã bar/line/area và các biến thể phải đi qua dispatcher. Đừng dùng bất kỳ enum có tên tương tự mà chưa có renderer: một số alias chỉ tồn tại trong type. Với donut, catalog/export dùng mã như `pie_donut`; việc PieFactory có nhánh cho `donut` không chứng minh dispatcher route mã `donut` vào đó.

### Tô màu cột theo khoảng cố định

```json
{"bar":{"colorByValue":true,"colorPieces":[{"gt":100,"color":"#dc2626"}]}}
```

Đặt fragment trong formatConfig. Trên nhóm bar hỗ trợ, giá trị >100 đỏ; ngoài khoảng giữ màu series. Biên dưới `gt` là loại trừ, biên trên `lte` là bao gồm. Ví dụ `{"lte":100,...}` nghĩa **≤100**, không phải `<100`. Khoảng chồng nhau lấy khoảng đầu tiên. Không đặt một epsilon tùy tiện để mô phỏng `<` cho dữ liệu liên tục; nếu cần strict `<`, xem consumer conditionalColors/UI đang triển khai hoặc nói rõ giới hạn.

`formatConfig.conditionalColors` là cơ chế khác có rules condition/value/value2/color ở một số factory; không đổi tên nó thành colorPieces. Cơ chế đã xác minh cho bar colorByValue so số cố định, không tra cột QCVN khác cùng row. Ngưỡng động dạng field không được giải quyết chỉ bằng thêm line lên chart.

### Đường ngưỡng và điểm đánh dấu

`formatConfig.thresholds` dùng enabled/value/color/lineType ở các factory hỗ trợ, tạo đường tham chiếu số cố định. `markLine` và `markPoint` là annotation riêng. Nhập `name` của markPoint không đảm bảo label hiển thị tên; nhánh combo không tự thêm label formatter/fontSize cho tên. Đường QCVN từ dữ liệu phải binding một series, không nhầm với thresholds.value.

## Combo

```text
dataConfig.yAxis = [measure, standard]
dataConfig.seriesTypes = { measure: bar, standard: line }
dataConfig.seriesAxes = { measure: 0, standard: 0 }
formatConfig.activeSeriesKey = measure
formatConfig.seriesFormat[measure] = { bar: {...}, dataLabels: {...} }
```

Trong fragment trên, `measure` và `standard` là tên cột, không phải literal nếu nguồn khác. Types được ComboFactory chấp nhận gồm bar, line, area, scatter, pie, donut, step. Type không hợp lệ fallback bar; không xem fallback là cấu hình đúng.

UI chuyển panel theo `activeSeriesKey`: chọn bar để thấy cấu hình cột; chọn line để thấy cấu hình đường. `seriesFormat[field]` override format gốc; một số object như bar/line/dataLabels/tooltip được merge, không phải mọi field đều merge sâu.

Bar-series có colorByValue theo khoảng; line-series không được tô bằng cùng resolver. Bật colorByValue nhưng không có khoảng ở combo không sinh gradient liên tục như bar thường. `seriesColors` giữ màu theo measure/group/series.

Tất cả series dùng chung category order. Sort theo standard sẽ làm cả cột và các line còn lại đổi vị trí theo category, không xáo độc lập từng dãy. Xem nhánh sort lỗi trước khi cam kết order-by.

## KPI

Loại `kpi`, binding chính ở dataConfig.yAxis; cấu hình tính/hiển thị phần lớn trong **formatConfig.kpi**, không đặt tất cả vào dataConfig.kpi. Nhánh main value hiện cộng kết quả aggregate của từng field trong yAxis; thả nhiều measure không có nghĩa tự tạo nhiều KPI độc lập.

- `valueAggregation`, `valueAggregations[field]`: aggregate chỉ tiêu chính.
- `mainValueSize`, `mainValueColor`, `mainValueWeight`, title/subtitle/footer, layout.
- `layout`: standard, compact, grid, status-bar, comparison, detailed, custom theo implementation; element layout không giống card chuẩn.
- `numberFormat`: object prefix/suffix/decimals/abbreviate/locale, khác enum numberFormat của chart chung.
- `conditionalFormatting.enabled/rules[]`: id, operator eq/neq/gt/gte/lt/lte, value, color, icon optional. Rule đầu khớp thắng.
- Runtime và KPIValuePanel còn hỗ trợ `valueType: "column"`, `compareField`, `compareAggregation`; so với giá trị **đã aggregate** của cột trên filtered main dataset. Đây không phải row-wise color của bar, và không nên sum một cột ngưỡng lặp nhiều dòng nếu nghiệp vụ không cho phép.
- `elements[]` trong custom layout: id, type, show, style, config. Màu cố định `element.style.color` trên mainValue có thể thắng mainValueColor tính từ điều kiện.
- target/trend/comparison có field, aggregation và filter riêng; timeIntelligence có dateField, anchor, kỳ so sánh. Phải kiểm cửa sổ thời gian và mẫu 0/null, không coi mọi badge là cùng công thức tăng trưởng.

Rule tĩnh tối thiểu trong KPI:

```json
{"conditionalFormatting":{"enabled":true,"rules":[{"id":"above-limit","operator":"gt","value":100,"color":"#dc2626"}]}}
```

Fragment này đặt trong `formatConfig.kpi`. Nếu không đỏ, kiểm val thực sau aggregate/filter và style override trước khi kết luận tính năng không hoạt động.

## Table và matrix

Table dựng cột từ `dataConfig.dimensions[].field` và `metrics[].field`, alias/name làm nhãn. Chỉ đặt xAxis/yAxis kiểu bar có thể không tạo cột table như mong đợi. `formatConfig.table` chứa columnConfigs/defaultColumnConfig, defaultSort, sticky/paging và nhiều tùy chọn; dùng UI/readback để giữ đúng field từng phiên bản. Table sort cục bộ không đồng nhất với format.sorting của chart.

Matrix: dimensions role row/column chọn trục. Metric key ở consumer hiện lấy `alias || name`, fallback yAxis; phải khớp property thực trong dataset, không chỉ một nhãn đẹp tùy ý. Hai bản ghi trùng cùng row/column/metric bị ghi đè bằng dòng cuối, không tự sum. Chuẩn bị dataset có một giá trị cho mỗi ô hoặc xác minh API query đã aggregate, không chỉ chọn SUM trong UI và mặc định đúng.

## Các nhóm khác

| Nhóm | Dữ liệu/cấu hình quan trọng |
| --- | --- |
| Pie/funnel | Category + measure; màu thường key theo category, không áp quy tắc màu measure của bar một cách máy móc |
| Radar | Category là các trục/ray; group/measure tạo series; max tự tính có thể làm khác cách đọc KPI |
| Heatmap | Category + groupBy và measure, hoặc category + nhiều measure; không tự nhận mọi ma trận 2D tùy ý |
| Scatter/bubble | Tọa độ x/y số, bubbleSize riêng; dùng đúng trường không ép category định danh làm tọa độ |
| Map | mapId/geoJsonUrl, mapJoinKey/geoJsonJoinKey; bubble/heatmap cần geoCenterMap phù hợp; kiểm join vùng và asset thực |
| Treemap/sunburst | Dữ liệu phân cấp/group theo factory, không coi table phẳng bất kỳ là cây hoàn chỉnh |
| Candlestick/boxplot | Có nhánh đòi array giá trị vị trí; phải kiểm data shape trước |
| Sankey/tree | Có nhánh nodes/links hoặc cây; chưa có bằng chứng generic binding mọi output tự chuyển đúng |
| Globe/map3d/polygons3d | Các factory đã khảo sát có scene nhưng series rỗng; không hứa visualization measure hoàn chỉnh |
| Calendar | Range hardcode 2025 tại bản nguồn, cần kiểm phiên bản đích |
| Text/media/group/sidebar | Component riêng; không có cùng trục, filter và tooltip như ECharts |

`pie3D`, `line3D`, area3d và các mã Zing cụ thể dùng ZingChart; `plotly_contour` dùng Plotly. Advanced ECharts options không tự áp được cho các engine này hoặc KPI/table.

## Click action

`seriesAction: {enabled:true,type:"page",pageId,openType:"modal"|"navigate"}` tạo tương tác trang ở component hỗ trợ. `pageId` là ID trang, không phải pageIndex. `filterSourceColumn`/`filterTargetColumn` có thể truyền giá trị lọc sang trang đích; cần field có trong data click và mapping đúng.

URL action dùng `urlPattern` và có các biến thay thế tùy component (table có `{{column}}`); quan sát UI hoặc export đúng loại, không dùng một cú pháp chung cho tất cả. Action thường chỉ thực thi ở View, không ở Edit. Bookmark selector/event chưa có consumer hoàn chỉnh xác minh trong repo—không dùng làm điều kiện bắt buộc để dashboard hoạt động.
