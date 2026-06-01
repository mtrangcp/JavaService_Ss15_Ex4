
Phần 1 - Phân tích logic
1. Vai trò của UserDetailsService và PasswordEncoder trong Spring Security
   UserDetailsService (Bộ nạp thông tin người dùng): Đảm nhận nhiệm vụ giao tiếp với tầng dữ liệu (Database). 
Nó chỉ chứa duy nhất một phương thức loadUserByUsername(String username). Khi người dùng nhập tên đăng nhập,
thành phần này sẽ truy vấn vào database, lấy ra các thông tin như: mật khẩu (đã băm), trạng thái tài khoản 
(bị khóa, hết hạn hay không) và danh sách các quyền/vai trò (GrantedAuthority). Sau đó, nó đóng gói toàn bộ 
thông tin này vào một đối tượng UserDetails để trả về cho Spring Security xử lý tiếp.

   PasswordEncoder (Bộ mã hóa và so khớp mật khẩu): Đảm nhận nhiệm vụ bảo mật mật khẩu. Nó có hai chức năng chính: 
encode(CharSequence rawPassword) (băm mật khẩu thô trước khi lưu vào DB) và matches(CharSequence rawPassword, 
String encodedPassword) (so sánh mật khẩu thô người dùng vừa nhập ở form đăng nhập với mật khẩu đã băm lấy ra từ DB).

2. Nguy cơ của 'Plain Text' và lý do BCrypt được khuyến nghị

Tấn công nội bộ (Insider Threats): Quản trị viên hệ thống, lập trình viên hoặc bất kỳ ai có quyền truy cập 
trực tiếp vào DB đều có thể nhìn thấy toàn bộ mật khẩu của khách hàng.

Lỗ hổng SQL Injection hoặc Backup dữ liệu: Nếu hacker khai thác được lỗi phần mềm để tải file backup DB hoặc 
dùng SQL Injection để đọc dữ liệu, toàn bộ tài khoản người dùng sẽ bị chiếm đoạt ngay lập tức.

Tái sử dụng mật khẩu (Credential Stuffing): Người dùng thường có thói quen dùng một mật khẩu cho nhiều trang web 
(Gmail, Ngân hàng, Thư viện). Nếu mật khẩu thư viện lộ ở dạng plain text, hacker có thể dùng nó để tấn công vào 
các tài khoản quan trọng khác của họ.

   Tại sao BCryptPasswordEncoder được khuyến nghị rộng rãi?
BCrypt là một giải pháp băm mật khẩu cực kỳ mạnh mẽ dựa trên thuật toán mã hóa Blowfish nhờ tích hợp các đặc tính:

Cơ chế Salt tự động (Muối ngẫu nhiên): Với mỗi mật khẩu (ví dụ: password123), BCrypt sẽ trộn thêm một chuỗi ký 
ự ngẫu nhiên (Salt) trước khi băm. Do đó, hai người dùng có cùng mật khẩu password123 khi lưu vào DB sẽ tạo ra 
hai chuỗi băm hoàn toàn khác nhau. Điều này vô hiệu hóa hoàn toàn kiểu tấn công bằng bảng tra cứu trước 
(Rainbow Table Attack).

Cơ chế Adaptive Hashing (Chậm có chủ đích): BCrypt cho phép cấu hình một tham số gọi là "độ giá trị" 
(Strength/Work Factor - mặc định là 10). Tham số này làm tăng số vòng lặp tính toán để băm mật khẩu. 
Nó khiến máy tính mất khoảng vài trăm mili-giây để kiểm tra một mật khẩu. Đối với người dùng thông thường, 
độ trễ này không đáng kể; nhưng đối với hacker muốn dùng máy tính cấu hình cao để thử hàng tỷ mật khẩu/giây 
(Brute-force Attack), việc này trở nên bất khả thi vì tốn quá nhiều thời gian và tài nguyên phần cứng.


Phần 2 - Thực thi (Kiến trúc giải pháp và Sơ đồ luồng)
Sơ đồ định dạng Mermaid (Graph TD)

graph TD
A[Client / Login Form] -- 1. Gửi Username & Password thô --> B(UsernamePasswordAuthenticationFilter)
B -- 2. Đóng gói thành Authentication Object --> C(AuthenticationManager)
C -- 3. Ủy nhiệm xác thực --> D(DaoAuthenticationProvider)
D -- 4. Gọi loadUserByUsername(username) --> E(CustomUserDetailsService)
E -- 5. Truy vấn SQL/JPA --> F[(Database)]
F -- 6. Trả về thông tin User (Username, Hashed Password, Roles) --> E
E -- 7. Đóng gói & Trả về UserDetails --> D

D -- 8. Truyền (Mật khẩu thô, Mật khẩu đã băm từ DB) --> G(BCryptPasswordEncoder)
G -- 9. Thực hiện so khớp matches() --> D

D -- 10. Nếu khớp, trả về Authentication thành công --> C
C -- 11. Lưu thông tin vào SecurityContextHolder --> B
B -- 12. Phản hồi thành công (Redirect / Trả Token) --> A

style E fill:#f9f,stroke:#333,stroke-width:2px
style G fill:#bbf,stroke:#333,stroke-width:2px
style F fill:#f96,stroke:#333,stroke-width:2px

Giải thích các thành phần chính trong sơ đồ:
UsernamePasswordAuthenticationFilter: Bộ lọc chặn HTTP Request POST chứa thông tin đăng nhập từ client, 
trích xuất thông tin gửi đến và chuyển giao cho Manager.

AuthenticationManager: Bộ điều phối trung tâm, nhận yêu cầu xác thực và tìm kiếm AuthenticationProvider 
phù hợp để xử lý.

DaoAuthenticationProvider: Trái tim của luồng xác thực dữ liệu từ DB. Nó chính là thành phần trực tiếp gọi 
CustomUserDetailsService để lấy dữ liệu lên, sau đó lấy BCryptPasswordEncoder để thực hiện so sánh mật khẩu.

SecurityContextHolder: Nơi lưu trữ thông tin của người dùng đã xác thực thành công (chứa Principal và Authorities). 
Các request tiếp theo sẽ nhìn vào đây để biết người dùng hiện tại là ai và có quyền gì (ADMIN, LIBRARIAN, USER) 
nhằm thực hiện phân quyền truy cập.
