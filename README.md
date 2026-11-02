# Bài 1 : Sự cố không truy cập được hệ thống quản trị
## Phần 1: Phân tích
### 1. DNS Lookup là gì?
DNS (Domain Name System) là hệ thống phân giải tên miền, hoạt động giống như một cuốn danh bạ Internet.

DNS giúp chuyển đổi tên miền mà con người dễ nhớ thành địa chỉ IP mà máy tính sử dụng để xác định máy chủ cần kết nối.
- Input: Tên miền, ví dụ: phong-dieu-phoi.greenlog.vn.
- Output: Địa chỉ IP của máy chủ, ví dụ: 203.0.113.10 (IP minh họa, không phải IP thực tế).
### 2. Tiến trình DNS Lookup
Khi Nam nhập tên miền vào trình duyệt, quá trình phân giải DNS diễn ra như sau:
```text
Người dùng (Client)
        |
        v
Nhập tên miền vào trình duyệt
        |
        v
Kiểm tra DNS Cache
        |
        v
DNS Resolver
        |
        v
Root DNS Server
        |
        v
TLD DNS Server (.vn)
        |
        v
Authoritative DNS Server
        |
        v
Trả về địa chỉ IP
        |
        v
Trình duyệt kết nối đến máy chủ
```
**Giải thích:**
1. Trình duyệt kiểm tra bộ nhớ DNS Cache để xem tên miền đã được phân giải trước đó chưa.
2. Nếu chưa có kết quả phù hợp, máy tính gửi truy vấn đến DNS Resolver.
3. Resolver tìm kiếm thông tin qua hệ thống DNS phân cấp (Root, TLD và Authoritative DNS).
4. Authoritative DNS trả về địa chỉ IP tương ứng nếu có bản ghi hợp lệ.
5. Trình duyệt sử dụng địa chỉ IP để kết nối đến máy chủ website.
### 3. Hậu quả của việc nhập sai tên miền
Nam nhập:
```text
phong-dieu-phoi.grenlog.vn
```
Trong khi tên miền đúng là:
```text
phong-dieu-phoi.greenlog.vn
```
Việc thiếu chữ `e` khiến tên miền bị thay đổi hoàn toàn. DNS xem đây là hai tên miền khác nhau và không tự động hiểu ý định của người dùng.

Nếu tên miền viết sai không tồn tại hoặc không có bản ghi DNS phù hợp, DNS sẽ trả về lỗi phân giải như `NXDOMAIN`.

**Kết luận:** Chưa có cơ sở để khẳng định máy chủ công ty bị sập nguồn. Nguyên nhân có thể đơn giản là Nam nhập sai tên miền.
## Phần 2: Triển khai
### 1. Phân biệt lỗi DNS và lỗi HTTP 500
|Tiêu chí	|DNS_PROBE_FINISHED_NXDOMAIN|	500 Internal Server Error|
|---|---|---|
|Bản chất|	Không phân giải được tên miền|	Máy chủ gặp lỗi khi xử lý yêu cầu|
|Nguyên nhân|	Sai tên miền, tên miền không tồn tại hoặc DNS không có bản ghi|	Lỗi code Backend, database hoặc cấu hình Server|
|Vị trí lỗi|	Trước khi kết nối HTTP đến máy chủ	|Phía Server|
|HTTP Status Code|	Không có phản hồi HTTP từ website|	500|
|Cách xử lý|	Kiểm tra tên miền và DNS|	Kiểm tra log và xử lý lỗi máy chủ|

**Kết luận:**
- Lỗi DNS: Trình duyệt chưa tìm được địa chỉ IP để kết nối đến máy chủ.
- Lỗi 500: Yêu cầu đã đến được máy chủ nhưng máy chủ không thể xử lý thành công.
### 2. Quy trình kiểm tra HTTP Status Code bằng F12
Nam thực hiện các bước sau:

**Bước 1: Mở Developer Tools**

Nhấn `F12` hoặc `Ctrl + Shift + I` để mở công cụ dành cho nhà phát triển.

**Bước 2: Chọn Network**

Chọn tab Network, sau đó tải lại trang bằng `F5` để ghi nhận các yêu cầu mạng.

**Bước 3: Kiểm tra Status Code**

Quan sát cột Status để xem máy chủ trả về mã trạng thái HTTP nào.

**Bước 4: Xem chi tiết phản hồi**

Nhấn vào request, chọn mục Headers, kiểm tra phần General và dòng Status Code.
### 3. Một số mã HTTP Status Code cần biết
|Mã trạng thái|	Ý nghĩa|	Đặc điểm|
|---|---|---|
|200 OK|	Thành công|	Máy chủ xử lý yêu cầu thành công|
|301 Moved Permanently	|Chuyển hướng vĩnh viễn|	Tài nguyên đã chuyển sang địa chỉ mới|
|403 Forbidden	|Bị từ chối truy cập	|Không có quyền truy cập tài nguyên|
|404 Not Found	|Không tìm thấy|	Không tìm thấy tài nguyên được yêu cầu|
|500 Internal Server Error|	Lỗi máy chủ	|Máy chủ gặp lỗi nội bộ|
|502 Bad Gateway	|Cổng kết nối lỗi	|Máy chủ trung gian nhận phản hồi không hợp lệ|
|503 Service Unavailable|	Dịch vụ không khả dụng|	Máy chủ tạm thời không thể phục vụ|

**Nguyên tắc quan trọng:** Nếu đã nhận được HTTP Status Code từ website thì yêu cầu đã đi qua bước phân giải DNS và thiết lập kết nối HTTP/HTTPS đến một máy chủ phản hồi. Tuy nhiên, chưa chắc đó chính là máy chủ ứng dụng mong muốn vì có thể là máy chủ Proxy hoặc CDN.
## Phần 3: Kiểm thử
### Tình huống: Nam sửa đúng tên miền nhưng nhận mã 404 Not Found.
**Phân tích nguyên nhân:**

Lỗi `404 Not Found` cho biết máy chủ đã nhận và xử lý yêu cầu HTTP nhưng không tìm thấy tài nguyên tương ứng với URL mà Nam yêu cầu.

Một số nguyên nhân có thể xảy ra:
1. Đường dẫn URL bị sai hoặc không tồn tại.
2. Trang quản trị đã bị đổi đường dẫn.
3. Route trên Backend chưa được khai báo.
4. Tài nguyên hoặc trang quản trị đã bị xóa.

**Hướng xử lý:**
- Kiểm tra lại đường dẫn URL.
- Kiểm tra cấu hình Routing của Backend.
- Kiểm tra tài nguyên và quyền truy cập.
- Tra cứu log Server để xác định nguyên nhân.
