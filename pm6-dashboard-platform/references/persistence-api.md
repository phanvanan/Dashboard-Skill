# Kết nối, API và lưu/khôi phục

## Điều kiện dùng

Các đường dẫn dưới đây được trích từ frontend, **không phải cam kết mọi server hiện tại giữ nguyên contract**. Ghép với base URL/context đúng của môi trường đã được người dùng cấp quyền. Dùng connector/API đã có hoặc phiên trình duyệt hợp lệ trên tab người dùng đang mở; không yêu cầu copy token vào chat, không dò filesystem/profile để lấy thông tin đăng nhập. Axios client của ứng dụng gắn Bearer web/embed, BaseService riêng chỉ cung cấp header JSON.

Nếu môi trường chỉ có browser UI, dùng [authoring-ui.md](authoring-ui.md). Không cần phát minh MCP server hoặc yêu cầu người dùng cài backend để dùng skill. Chỉ dùng API trực tiếp khi công cụ hiện có cho phép và contract đích đã được xác nhận bằng read/response, tài liệu hoặc request UI hợp lệ.

## Endpoint đọc và khám phá

| Method / path | Mục đích |
| --- | --- |
| GET `/data-hub-app/api/v1/database/authorized-list` | Database được cấp quyền; id, databaseName, displayName, databaseType |
| GET `/data-hub-app/api/v1/database/{databaseId}/tables/list` | Tables/outputs; frontend dùng tableName |
| GET `/data-hub-app/api/v1/database/{databaseId}/tables/{tableId}/metadata/detail` | Metadata cột; tham số service tên tableId, lấy giá trị mà UI/response môi trường dùng, không tự bịa UUID |
| GET `/iam-service/api/v1/category-domain/list` | Danh mục lĩnh vực nếu cần cateDomainId |
| POST `/visualization-app/api/v1/visualizations/raw-data/list` | Đọc row vật lý, request có querySpec |
| POST `/visualization-app/api/v1/visualizations/query` | Đọc query tổng hợp; không phải API lưu chart |
| GET `/visualization-app/api/v1/layouts/tree?id={layoutId}` | Cây layout đầy đủ theo V1 |
| GET `/visualization-app/api/v2/layouts/tree/metadata?id={layoutId}` | Metadata, pages nhưng không nhất thiết pageItems |
| GET `/visualization-app/api/v2/layouts/tree/pages/{pageId}` | Chi tiết một page |

Metadata response có thể là array hoặc object `columns`, có thể bọc `data`; giữ metadata như `lastDataUpdatedAt`. Data response được frontend nhận dạng array, data array, content array, hoặc data.content. Không coi một envelope chưa hiểu là dữ liệu rỗng; đọc response thực.

Request raw mẫu có giới hạn, thay placeholder bằng tên/ID đã khám phá:

```json
{
  "keywords":"",
  "pageable":{"pageSize":20,"pageNumber":0},
  "sorts":[],
  "databaseId":"DATABASE_ID",
  "tableName":"OUTPUT_NAME",
  "querySpec":{
    "dimensions":["PHYSICAL_FIELD_A","PHYSICAL_FIELD_B"],
    "aggregateFields":[],
    "whereConditions":[]
  }
}
```

Raw `querySpec.dimensions` là danh sách cột vật lý cần lấy, kể cả cột measure. Query aggregate dùng dimensions cho group và aggregateFields dạng `{fieldName,aggFunc:"SUM"}`; query builder chấp nhận SUM/AVG/COUNT/MIN/MAX/DISTINCT_COUNT nhưng backend support vẫn phải xác minh. Không tự giả định tên output của cột aggregate; kiểm row trả về rồi binding đúng.

Filter request: `{fieldName,operator,filterValue}`; sorts request: `{sortBy,direction:"ASC"|"DESC"}`. `format.sorting` vẫn có thể sort lại frontend. Xem [data-and-filters.md](data-and-filters.md) cho sự khác nhau giữa request và widget filter.

## Endpoint ghi layout/page

| Method / path | Hành vi frontend sử dụng |
| --- | --- |
| POST `/visualization-app/api/v1/layouts/tree` | Tạo layout mới với cây trang |
| PUT `/visualization-app/api/v1/layouts/tree/{layoutId}` | Cập nhật metadata/config layout; đường editor hiện bỏ layoutPages khi update |
| PUT `/visualization-app/api/v1/layouts/tree/pages` | Tạo page trong layout hiện có; **PUT**, không suy là POST |
| PUT `/visualization-app/api/v1/layouts/tree/pages/{pageId}` | Cập nhật nội dung page |
| PUT `/visualization-app/api/v1/layouts/tree/pages/order` | `{pages:[{id,pageIndex},...]}` |

Không có endpoint `POST /charts/generate-from-prompt` được xác minh. Một yêu cầu ngôn ngữ tự nhiên phải được AI chuyển thành cấu hình và thao tác thật, không gửi prompt vào URL tự nghĩ ra.

## Cấu trúc lưu: ba cấp cần phân biệt

### Layout mới

```text
{
  id: null hoặc bỏ khi serialize,
  name, page, layoutConfigs, isPublic, active,
  layoutPages: [{
    id: null, fakeId, title, pageIndex,
    databaseId, tableName, cateDomainId, active,
    pageFilterConfigs, pageFilterValues, layoutId,
    pageItems: [...]
  }]
}
```

`page` trong mapper là số lượng trang để điều khiển page bar (hoặc 0 khi tắt), **không phải ID trang đang mở**. `pageIndex` là thứ tự; page ID do backend cấp. Fake IDs dùng để liên kết đối tượng mới; sau lưu lấy lại mapping ID thực. Không dùng fakeId làm URL View.

Source mapper đặt mặc định isPublic=true và active=1, nhưng đây **không phải sự cho phép publish của người dùng**. Giữ visibility/quyền của đích hoặc chọn chế độ private/draft được deployment hỗ trợ; nếu UI tạo mới có tác động công khai chưa được ủy quyền, hỏi đúng điểm đó. Không mặc định sao chép cờ public từ ví dụ.

### PageItem CHART

```text
{
  id, fakeId?, key?, type: "CHART", url: "",
  x, y, w, h,
  pageItemConfigs: {...},
  chartInfo: {
    id, key?, name, chartTypeCode, cateDomainId,
    databaseId, tableName,
    chartConfigs: {
      xAxis, yAxis, seriesTypes, ...các field dataConfig,
      formatConfig: {...}
    },
    metadata: {...}
  }
}
```

Đây là mô tả shape, không phải payload đủ validation mọi backend. ID pageItem khác chartInfo.id; key string không được tự đổi thành số. Với update page còn có `chartId` ở top-level item và chartInfo được làm sạch. Source databaseId/tableName được giữ ở chartInfo, bản duplicate trong chartConfigs có thể bị loại.

Metadata legacy được mapper tạo từ binding:

- `xAxis`: cột đầu tiên; `yAxisList[]`: fieldName, aggFunc.
- `filterDataList[]`: fieldName/operator/filterValue từ queryFilters và visualFilterValues.
- `sortByYAxis`, `isXAxisAscending` là projection hạn chế, không đủ biểu diễn mọi format.sorting.
- `noteDataList`, `attributes`, metadata id cần giữ từ đối tượng có sẵn; coverageInformation có cấu trúc riêng.

Đặc biệt, mapper có thể ghi seriesTypes bar/line của combo vào metadata aggFunc, trong khi query builder có bước whitelist SQL aggregate. Vì thế không coi metadata.aggFunc luôn hợp lệ để gửi thẳng endpoint query. Với API authoring mới, lấy payload hợp lệ của một widget cùng loại trên deployment, thay đúng binding/format/IDs thay vì giả định fragment skill là schema backend đầy đủ.

### PageItem GROUP / IFRAME

GROUP có type, id/fakeId/key, x/y/w/h, label, groupConfigs `{formatConfig,layout}`, pageItemConfigs và children array. Nội bộ children là map widget + layout tương đối; có legacy groupId/groupConfig và tọa độ absolute khi serialize. Không tự flatten hay đổi key nếu không cần.

IFRAME/video dùng `type: IFRAME`, url, chartInfo tương ứng và có thể bản sao format trong pageItemConfigs để khôi phục khi response thiếu chartConfigs. Giữ bản sao này khi chỉnh tài nguyên/header, không “dọn trùng” một cách tổng quát.

Một số config từ backend là JSON string, hoặc legacy object có key số chứa ký tự. Cần parse cấu trúc dữ liệu, không execute text. Không coi string config là lý do làm mất toàn bộ unknown keys khi round-trip.

## Chu kỳ chỉnh sửa an toàn và đúng engine

1. Đọc cây/chi tiết **đầy đủ** của phần cần sửa, giữ bản trước chỉnh sửa và map phụ. Metadata V2 không đủ cho việc replacement pageItems.
2. Đổi đúng chart/page được yêu cầu; không thay nguồn các slicer khác, không xóa pageItems vắng mặt do lazy-load. Giữ ID và field không hiểu.
3. Với layout đã tồn tại, cập nhật page bằng page endpoint/UI; sau đó cập nhật metadata/layoutConfigs nếu cần. Không gửi lại toàn bộ layout từ một snapshot chỉ có trang đang mở.
4. Editor hiện chỉ đưa **trang đang active** qua savePage/createPage trong save pipeline; việc nhìn nhiều trang trong editor không chứng minh tất cả nội dung đã lưu. Sau thao tác nhiều trang, xác nhận từng page đã có dữ liệu backend/readback.
5. HTTP 200/201 là tín hiệu lưu mà service dùng, không chứng minh đúng visualization. Lấy lại page/layout, kiểm IDs/binding/format/sidecar, mở View và đối chiếu phần đã đổi.

Không retry mù POST tạo layout sau timeout: đọc/tìm lại kết quả bằng ID/response/danh sách trước để tránh tạo trùng. Nếu gặp 401/403, xung đột ID, contract không khớp hoặc outcome không rõ, dừng ghi, giữ cấu hình dự thảo và giải quyết điểm thiếu; không dùng vòng “self-healing” để xóa ID thật hoặc tạo mới thay bản lỗi.

## Portal và route

Route frontend quan sát được (có thể có context prefix):

- `/cau-hinh-layout` tạo, `/cau-hinh-layout/{id}` chỉnh.
- `/v2/view-layout/{id}`, `/v1/view-layout/{id}` xem layout trong mã nguồn đã khảo sát. Deployment live đã kiểm dùng `/view-layout/{id}` (xem mục quan sát live bên dưới).
- `/view-layout-mobile/{id}` embed; cần cơ chế auth của host.
- `/dashboard`, `/cau-hinh-dashboard`, `/view-dashboard/{id}` quản trị/cấu hình/xem portal.

Portal monitoring-info có name, dashboardId, parentId, layoutId, priority, refreshTime, iconUrl, status. Liên kết layout vào portal là thao tác riêng; ưu tiên UI hoặc contract của deployment khi người dùng yêu cầu. Chia sẻ người/tổ chức, thay status và quyền là thao tác riêng ngoài “vẽ chart”.

## Quan sát trên deployment live

Các điểm dưới đây được kiểm trên một deployment PM6/IOC thật ngày 2026-10-08 (skill 1.1.0), bằng phiên trình duyệt đã đăng nhập. Đây là bằng chứng của **một** deployment, không phải hợp đồng chung: luôn thử nhẹ để xác nhận trước khi ghi.

**Đường dẫn và xác thực**

- Gateway thêm tiền tố `/api` trước tên service: `/api/data-hub-app/api/v1/...`, `/api/visualization-app/api/v1/...`. Gọi không có tiền tố thì bị chặn; gọi dưới context của ứng dụng (`/dash/...`) thì nhận về HTML.
- Xác thực bằng `Authorization: Bearer <token>`. Token lấy từ localStorage của phiên đăng nhập (key dạng `ACCESS_TOKEN`) và chỉ dùng trong page context. Có lúc nhận 401 khi token hết hạn; ứng dụng tự làm mới, chỉ cần đọc lại token rồi gọi lại.
- Route View trên deployment này là `/view-layout/{id}` (dưới context ứng dụng, ví dụ `/dash/view-layout/{id}`). `/v2/view-layout/{id}` trả 404. Đọc route thật từ ứng dụng thay vì đoán.

**Đọc dữ liệu**

- `.../tables/{tableId}/metadata/detail` chấp nhận **tableName** ở vị trí tableId. Response là `{lastDataUpdatedAt, columns:[{name,dataType,...}]}`.
- `.../tables/list` chỉ trả `[{tableName}]`. Một database có thể có vài trăm bảng, nên lọc theo từ khóa.
- `POST /visualization-app/api/v1/visualizations/raw-data/list` trả **mảng dòng ở cấp cao nhất**. `pageable.pageSize` **bị bỏ qua** (đã nhận đủ hơn 12.000 dòng dù xin 2.000). Vì vậy chỉ profile ngay trong trình duyệt và trả về tóm tắt, không đổ toàn bộ dòng ra ngoài.

**Đọc layout**

- Danh sách layout: `POST /visualization-app/api/v1/layouts/page` với body `{"keywords":"","pageable":{"pageSize":200,"pageNumber":0}}` → `{content:[{id,name,active,totalPages,totalCharts,public}], totalElements}`. `GET /layouts` trả 500.
- `GET /layouts/tree?id=` trả cây đầy đủ, gồm cả group và children. Dùng để clone payload mẫu.

**Ghi trang và layout**

- **Tạo trang** — `PUT /layouts/tree/pages` → 201, trả trang với `id` thật cho trang, pageItem và chartInfo:

  ```text
  {layoutId, id:null, fakeId:"page-<ts>", title, pageIndex, databaseId, tableName,
   cateDomainId:0, active:1, pageItems:[item...]}
  ```

  Item mới:

  ```text
  {type:"CHART", id:null, fakeId:"w-<ts>", x,y,w,h, chartId:null, key:null, url:"",
   pageItemConfigs:{responsiveLayouts:{desktop:{x,y,w,h}}},
   chartInfo:{cateDomainId, databaseId, tableName, name, chartTypeCode, chartConfigs, metadata, key:null}}
  ```

  `chartConfigs` không chứa databaseId/tableName; nguồn nằm ở chartInfo. Không gửi chartInfo.id cho widget mới.
- **Cập nhật trang** — `PUT /layouts/tree/pages/{pageId}`:

  ```text
  {id, title, pageIndex, databaseId, tableName, cateDomainId, active, fakeId:null, pageItems}
  ```

  Mỗi item: `id` = pageItem.id, `chartId` = chartInfo.id, `key:null`. Gửi lại **toàn bộ** pageItems của trang; ID được giữ nguyên.
- **Cập nhật layout** — `PUT /layouts/tree/{layoutId}` với `{id,name,active,isPublic,layoutConfigs}`, không kèm layoutPages. Các trang **không bị ảnh hưởng**. Đọc layoutConfigs mới nhất, chỉ thêm hoặc sửa key cần thiết, giữ nguyên `isPublic` hiện có.
- **Xóa trang** — `DELETE /layouts/tree/pages/{pageId}` là **xóa cứng**: ngay sau đó trang trả 404 với thông báo không tồn tại. Không có cơ chế khôi phục; phải tạo lại từ snapshot.

**Kiểm chứng**

Hình dạng payload ở trên khớp với hàm `createPage`/`savePage` trong bundle frontend của deployment. Khi phiên bản khác, đọc lại bundle hoặc request do UI phát ra thay vì giả định.
