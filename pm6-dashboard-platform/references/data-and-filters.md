# Dữ liệu, binding và filter

## Khám phá output trên máy không có source

Từ tên như `IOC_DAUTUCONG`, tìm database người dùng được cấp quyền, rồi table đúng tên. Không đồng nhất database displayName với ID, table label với tableName, hoặc metadata column id với tên cột trong row. Nếu trùng output ở nhiều database, xác định nguồn từ ngữ cảnh; chỉ hỏi khi chưa thể phân biệt.

API discovery được mô tả trong [persistence-api.md](persistence-api.md); trên UI dùng bộ chọn nguồn của editor. Metadata có `name`, `label?`, `dataType`, `id`; chart binding dùng **khóa thực trong row**, alias là phần hiển thị. Metadata date/number cần đối chiếu mẫu, vì metadata `number` và các loại decimal/integer có thể đi qua nhánh sort khác nhau.

Đọc mẫu có giới hạn và metadata đầy đủ, không mặc định tải toàn bộ output. Xác định một row đại diện cho gì, cột nào là khóa, thời gian/kỳ, measure, số lũy kế hoặc tỷ lệ đã tính. Nếu dữ liệu chưa đủ để hiểu công thức, hỏi hoặc nêu giả định thay vì suy từ tên. Đây là suy luận của AI/người dùng, không có bộ KPI đầu tư công cố định trong skill.

## Binding của chart

| Key | Ý nghĩa |
| --- | --- |
| `databaseId`, `tableName` | Nguồn widget; không chỉ nguồn mặc định của page |
| `xAxis` | Tên cột hoặc mảng tên cột category; nhiều cột thường được nối thành nhãn |
| `yAxis` | Tên cột hoặc mảng tên cột measure |
| `groupBy` | Nhóm series, không phải phép join nhiều bảng |
| `tooltipFields` | Cột bổ sung vào tooltip ở các consumer hỗ trợ |
| `xAxisAlias`, `yAxisAlias`, `tooltipFieldsAlias` | Map physical field → tên hiển thị |
| `seriesVisible` | Map field → boolean; khác cơ chế hiddenSeries ở một số nhánh |
| `seriesColors` | Map field/group/series/category → màu, tùy factory |
| `seriesTypes` | **Bị dùng hai nghĩa**: phép aggregate ở chart thường, loại bar/line/... ở combo |
| `seriesAxes` | Map measure → 0 hoặc 1 cho trục chính/phụ |
| `aggregation` | Fallback ở một số nhánh, không bảo đảm override mọi phép tính |
| `dimensions`, `metrics` | Schema phục vụ table/matrix/query, không tự thay thế đầy đủ xAxis/yAxis ở mọi chart |

`dimensions[]` có `id,name,field,type`, optional `role: row|column`, alias, granularity. `metrics[]` có `id,name,field,aggregation,alias`. `formula` được khai báo nhưng chưa có evaluator metric tổng quát được xác minh. Không cấu hình công thức bất kỳ rồi hứa platform tự tính.

`sum,avg,count,min,max` có helper frontend. Count đếm giá trị không null/undefined/rỗng, không nhất thiết COUNT(*). Helper có fallback count khi yêu cầu toán học mà toàn dữ liệu không số. `distinct_count` có ở type/API query nhưng không được hỗ trợ đồng nhất bởi helper frontend. Đừng dùng fallback này để che metadata sai.

Nếu measure là tỷ lệ/lũy kế/ngưỡng, việc chọn aggregate do AI/người dùng quyết định theo grain; platform không có semantic model tự đảm bảo đúng nghiệp vụ. Với combo, `seriesTypes[field]='line'` không đồng thời biểu diễn `avg/max`; nguồn đã tổng hợp sẵn theo grain có thể cần thiết. Không tự tạo SQL view hay sửa output khi người dùng mới yêu cầu vẽ chart.

## Ba biểu diễn filter khác nhau

### Query trong widget

```json
{"queryFilters":[{"field":"period","operator":"in","value":["2026-09"]}]}
```

Key/operator ở đây là lower-case; được lưu trong dataConfig. Các loại được khai báo: eq, neq, gt, gte, lt, lte, contains, in, between, top_largest, top_smallest. Việc khai báo không có nghĩa mọi local filter consumer xử lý hết như nhau.

### Runtime filter và dimension mapping

```json
{
  "dimensions": {
    "reporting_period": {
      "dimensionId": "reporting_period",
      "dimensionName": "Kỳ báo cáo",
      "mappings": [
        {"databaseId":"DATABASE_ID","tableName":"OUTPUT_A","column":"period"},
        {"databaseId":"DATABASE_ID","tableName":"OUTPUT_B","column":"report_period"}
      ]
    }
  },
  "globalFilterValues": {
    "reporting_period": {"operator":"IN","values":["2026-09"]}
  }
}
```

Đây là fragment của `layoutConfigs`, không phải request query. Placeholder phải thay từ môi trường thật. FilterStore dùng map globalFilters, pageFilters[pageId], widgetFilters[widgetKey] với payload `{operator,values}`. Scope merge global → page → widget; scope sau ghi đè cùng dimension, không tự AND ba giá trị của cùng key.

Khi lưu, cấu hình global ở layoutConfigs; page có pageFilterValues; visual values có thể ở `chartInfo.chartConfigs.visualFilterValues`. Đừng persist nguyên store cache/loading/interactive flags vào layout. `dimensions` logic trong layoutConfigs khác `dataConfig.dimensions[]` dùng làm trục/bảng.

### Request backend

```json
{"fieldName":"period","operator":"IN","filterValue":"2026-09"}
```

Đặt trong `querySpec.whereConditions`. Không gửi nguyên `{field,operator,value}` nếu endpoint đang yêu cầu dạng trên. `IN` array thường được serialize bằng dấu phẩy; giá trị chứa dấu phẩy cần kiểm contract triển khai. Query builder ánh xạ `gte` thành `GREATER_THAN_OR_EQUAL`, `lte` thành `LESS_OR_EQUAL`; đừng tự thống nhất tên operator theo suy đoán.

## Slicer

Slicer cũng là widget có source riêng. Phần UI chính nằm trong `formatConfig.slicer`:

- `scope`: runtime đọc mặc định `page`; có thể global theo UI. Không nhét scope vào từng query condition.
- `customFields[]`: id, label, controlType (`combobox`, `textbox`, `date`), defaultValue và cấu hình nguồn/đích.
- Combobox: `comboSource: table|multiTable`, `comboSources[]` có databaseId/tableName/valueColumn/labelColumn, `comboTargetColumn[]`, `comboLabelColumn`, `allowMultiple`.
- Textbox: `textColumns[]`, `textOperator`. Date: dateType/dateViewMode/dateTargetColumn.
- Cascading: parentFilterId, referenceField; cần xác nhận mapping và miền options.
- `targetWidgetIds[]`: không rỗng thì nhắm các widget đã chọn; rỗng không phải “không lọc ai”. Phải dùng identity thực mà runtime dùng, kể cả sau remap group/new ID.
- `filterMode: onChange|onButtonSubmit`: đổi giá trị chưa chắc lập tức áp query nếu đang chế độ nút submit.

Options từ nhiều nguồn và mapping không phải tính năng join measure. Default tĩnh không bằng “mới nhất mỗi lần mở”. Query Top N trong slicer có thể resolve thành lựa chọn IN dựa trên source rows/options và runtime filters.

## Freshness và lỗi đã biết có liên quan

- Top N chạy cả View/Edit; **chỉ Edit** persist lựa chọn vào customFields.defaultValue. Default cũ và effect resolve Top N có thể cạnh tranh. Mở Edit rồi thấy View thay đổi không chứng minh nguyên nhân duy nhất; xem cấu hình/readback và dữ liệu kỳ mới.
- Key cache raw dựa database/table; query khác có thể cần skipCache ở luồng phù hợp. Skill không cho quyền xóa cache hay state của người dùng ngoài phạm vi tác vụ.
- `eq` với numeric 0 bị query builder chuyển thành chuỗi rỗng; `between [0,100]` có thể mất biên 0. Một request tự xây đúng phải giữ `"0"`, nhưng sửa request đơn lẻ không bảo đảm các request UI tương lai không tái lỗi. Kiểm query thực sau tương tác.
- Hai cột khác tên giữa nguồn không tự liên thông. Resolve không được cột có nhánh bỏ điều kiện, khiến một chart trông như không phản ứng.
- Khi kiểm filter, đối chiếu ba thứ: selection UI, request/backend hoặc dataset thực, và giá trị chart. Nhìn label slicer chưa đủ.
