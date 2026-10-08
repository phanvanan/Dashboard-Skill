# Dashboard Skill

Giúp AI hiểu nền tảng **PM6/IOC** để tạo chart và dashboard từ yêu cầu của bạn — **không cần mã nguồn**. AI tự đề xuất thiết kế; bạn quyết định kết quả.

## Cách dùng

**1. Cài skill** bằng cách gửi cho AI có hỗ trợ cài skill:

```text
Cài skill từ https://github.com/phanvanan/Dashboard-Skill/tree/main/pm6-dashboard-platform
```

**2. Mở nền tảng, đăng nhập và cho AI truy cập bằng công cụ của bạn.**

**3. Gõ yêu cầu**, ví dụ:

```text
Dùng pm6-dashboard-platform, vẽ dashboard Đầu tư công từ output IOC_DAUTUCONG.
Hãy tìm hiểu dữ liệu và đề xuất các trang phù hợp.
```

Bạn không cần đọc tài liệu kỹ thuật — AI sẽ tự tra cứu.

## AI sẽ làm việc thế nào (từ bản 1.1.0)

1. **Khám phá dữ liệu** bằng API chỉ đọc. AI báo lại cho bạn:
   - các bảng tìm được;
   - kỳ mới nhất;
   - bảng cũ hoặc rỗng;
   - số liệu bất thường.
2. **Đề xuất các trang.** Bạn chọn trang và layout đích.
3. **Dựng từng trang.** Mỗi trang xong, AI mở View cho bạn xem ngay và báo các con số đã đối chiếu.
4. **Bàn giao:** link View, các giả định, những điểm cần người có nghiệp vụ kiểm tra.

Lưu ý: nếu bạn đang mở editor trong lúc AI ghi, hãy **tải lại (F5) editor** trước khi tự chỉnh hoặc bấm Lưu. Skill không chứa dữ liệu hay tài khoản; AI cần quyền và công cụ truy cập hệ thống để tạo dashboard thật.

Nếu cài thủ công: [tải ZIP](https://github.com/phanvanan/Dashboard-Skill/archive/refs/heads/main.zip), giải nén và thêm thư mục `pm6-dashboard-platform` vào nơi quản lý skill của công cụ AI bạn dùng.
