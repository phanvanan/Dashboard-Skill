# Quy trình vận hành chuẩn

Quy trình này được rút ra từ một phiên dựng dashboard thật trên deployment PM6/IOC (2026-10-08): dựng 4 trang, 34 widget bằng API, mất một lần toàn bộ trang và dựng lại. Mục tiêu là **người dùng thấy được tiến độ và kết quả**, không phải ngồi chờ mà không biết AI đang làm gì. Kết quả cuối, tức dashboard đã lưu và hiển thị đúng trong View, là thứ quan trọng nhất. Thao tác chuột trên editor chỉ dùng khi người dùng muốn xem hoặc khi API không làm được.

## 0. Nguyên tắc minh bạch

- Trước khi làm, nói kế hoạch trong 3–5 dòng. Sau mỗi mốc (khám phá xong, đề xuất xong, ghi xong mỗi trang, xác minh xong), gửi một cập nhật ngắn: đã làm gì, thấy gì, bước tiếp theo.
- Không im lặng quá vài phút. Nếu một bước dò cấu trúc kéo dài, báo đang dò gì và vì sao.
- Mỗi trang ghi xong thì **mở ngay trong View trên tab người dùng đang xem**. Không gom đến cuối mới cho xem.
- Chỉ báo nguyên nhân sự cố khi có bằng chứng. Nếu chưa có thì ghi rõ là chưa biết, hoặc gắn nhãn suy luận.

## 1. Khám phá dữ liệu (chỉ đọc API)

1. Xác định tiền tố API của deployment bằng một GET vô hại (`.../database/authorized-list`). Deployment đã kiểm dùng tiền tố `/api` trước tên service, xem [persistence-api.md](persistence-api.md#quan-sát-trên-deployment-live).
2. Dùng phiên đăng nhập sẵn có trong trình duyệt của người dùng. Token nằm trong localStorage của ứng dụng và chỉ dùng trong page context. Không in token ra, không gửi token đi nơi khác.
3. Lấy danh sách bảng, lọc theo từ khóa của output. **Tên output người dùng nói có thể không tồn tại đúng như vậy.** Ví dụ người dùng nói một tên viết hoa, nhưng thực tế là một họ gồm nhiều bảng cùng tiền tố. Liệt kê các ứng viên, không tự chọn bừa.
4. Với mỗi ứng viên: lấy metadata và `lastDataUpdatedAt`, rồi lấy mẫu dòng. Profile **ngay trong trình duyệt** và chỉ trả về bản tóm tắt: số dòng, số giá trị khác rỗng, số giá trị phân biệt, min/max, vài giá trị mẫu. Kết quả trả về của công cụ trình duyệt có thể bị cắt sau khoảng 1.000 ký tự, nên lưu chuỗi dài vào biến rồi đọc theo từng đoạn.
5. Báo người dùng các điểm sau:
   - Grain của từng bảng.
   - Kỳ mới nhất.
   - Bảng cũ hoặc rỗng.
   - Cột toàn null.
   - Giá trị bất thường.
   - Dòng tổng hoặc dòng nhóm bị trộn lẫn với dòng chi tiết.
   - Cột kỳ dạng chữ (sắp xếp sai thứ tự).
   - Tên cột khác nhau giữa các bảng (cần ánh xạ dimension).
6. Đề xuất các trang và hỏi người dùng hai điều: chọn trang nào, và dựng vào layout nào (layout mới hay layout có sẵn).

## 2. Lấy mẫu cấu hình thật

- Liệt kê layout bằng `POST /layouts/page`, rồi đọc cây của một dashboard đang chạy tốt, ưu tiên cùng lĩnh vực.
- Với mỗi loại widget cần dùng (kpi, slicer, bar, bar_horizontal, line, pie, table), **clone payload thật** rồi chỉ thay phần binding, filter, header và vị trí. Không tự dựng payload từ đầu bằng fragment trong skill.
- Nếu mẫu dùng giao diện tối (chữ trắng, nền xanh đậm) mà layout đích dùng nền sáng, đổi màu có kiểm soát: chữ trắng sang chữ tối, nền tối sang nền trắng.
- Canvas desktop đã quan sát là hệ tọa độ thiết kế rộng khoảng 1920 px.

## 3. Ghi theo từng trang

Thực hiện tuần tự cho **mỗi trang**:

1. Chụp snapshot cây layout hiện tại (giữ trong biến của tab làm việc) để có thể khôi phục.
2. `PUT /layouts/tree/pages` để tạo trang mới, kèm toàn bộ pageItems, theo [payload mẫu](../examples/create-page-payload.json).
3. Đọc ID thật trong response. Sau đó `PUT /layouts/tree/pages/{pageId}` để gán `slicer.targetWidgetIds` bằng các pageItem.id (dạng chuỗi). Widget mới chưa có ID cho tới khi tạo xong, nên **luôn cần hai bước**.
4. `PUT /layouts/tree/{layoutId}` để thêm `layoutConfigs.dimensions` cho các ánh xạ cột kỳ giữa các bảng. Đọc layoutConfigs mới nhất trước khi ghi, chỉ thêm key, giữ nguyên mọi key khác.
5. Mở View ngay trên tab của người dùng, chụp màn hình và đối chiếu một vài con số với dữ liệu gốc.
6. Gửi cập nhật cho người dùng, rồi mới sang trang tiếp theo.

## 4. Quy tắc tab và editor (bắt buộc)

- **Tách tab:** gọi API trong một tab làm việc riêng (cùng origin, ví dụ trang danh sách layout). Tab của người dùng chỉ dùng để hiển thị kết quả. Lý do: điều hướng một tab sẽ xóa sạch biến và hàm JavaScript đã nạp trong tab đó. Báo cho người dùng biết có tab làm việc này.
- **Editor cũ là nguy hiểm.** Một editor được mở *trước* khi ghi bằng API không biết về các trang mới. Khi đó, Lưu hoặc bất kỳ thao tác đồng bộ nào có thể ghi đè layoutConfigs hoặc xử lý sai các trang.
  - Sau khi ghi bằng API: yêu cầu người dùng **tải lại (F5) editor** trước khi chỉnh tay.
  - Đang chỉnh trên editor: lưu xong rồi mới quay lại dùng API.
  - Không để API và editor cùng sửa một trang.
- **Xóa trang là xóa cứng.** `DELETE /layouts/tree/pages/{id}` khiến trang trả 404 ngay sau đó, không có thùng rác. Trên editor, nút xóa trang nằm ngay cạnh thanh tab trang, nên khi bấm chuyển tab cần nhắm đúng chữ tên tab. Muốn khôi phục chỉ có cách tạo lại từ snapshot hoặc script.
- Sau khi người dùng (hoặc editor) thao tác, đọc lại cây layout để xác nhận số trang và số item, trước khi báo bất cứ điều gì.

## 5. Xác minh và bàn giao

- **Từng trang:** mở View, kiểm tra các điểm sau:
  - KPI có nhãn.
  - Chart có dữ liệu.
  - Thứ tự tháng đúng.
  - Bảng đủ số dòng.
- **Bộ lọc:** đổi slicer sang kỳ khác, xác nhận cả các widget dùng bảng khác (được ánh xạ qua dimension) cũng thay đổi.
- **Tổng:** cộng các thành phần phải ra bằng dòng tổng (ví dụ cộng các nguồn vốn bằng tổng cộng).
- **Đọc lại toàn layout:**
  - Các trang cũ không bị đổi.
  - Các key cũ trong layoutConfigs vẫn còn.
  - Cờ chia sẻ (isPublic) giữ nguyên.
- **Bàn giao cho người dùng:**
  - Link View.
  - Danh sách trang và những con số đã đối chiếu.
  - Giả định nghiệp vụ: đơn vị tính, kỳ mặc định là giá trị cố định.
  - Những điểm dữ liệu cần người có nghiệp vụ kiểm tra.
  - Lưu ý tải lại editor trước khi chỉnh.

## 6. Khi nào dùng chuột

Thao tác chuột trên editor là tùy chọn. Dùng khi:

- người dùng muốn xem quá trình dựng;
- cần một tùy chọn mà payload chưa rõ cấu trúc, ví dụ quy tắc tô màu có điều kiện mới: tạo một quy tắc qua UI, lưu, rồi đọc lại payload để học cấu trúc.

Luôn tải lại editor sau các lần ghi bằng API, chọn widget bằng cách bấm vào vùng nội dung của nó, lưu bằng nút Lưu của editor, rồi đọc lại qua API.
