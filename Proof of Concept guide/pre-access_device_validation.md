# Thiết bị không đạt chuẩn → không được join Ztrust

Agent Ztrust kết hợp với các phần mềm tại endpoint để thực hiện kiểm tra điều kiện bảo mật của thiết bị. Hiện tại Ztrust đang cung cấp 9 mẫu `posture rule` miễn phí.


Ví dụ: Yêu các thiết bị thuộc nhóm SEC khi tham gia vào mạng Ztrust bắt buộc phải cài đặt phần mềm `Velociraptor` và `wazuh-agent`, nếu không cài đặt sẽ thực hiện `Block` khỏi mạng Ztrust. Chi tiết cấu hình như sau:

![img.png](images/posture_checks.png)

Khi người dùng thuộc nhóm SEC truy cập mà không cài đặt đủ phần mềm bắt buộc thì sẽ không truy cập được, thông báo sẽ như sau:
![img.png](posture_check2.png)

Đồng thời một cảnh báo cũng được chuyển đến cho quản trị viên
![img.png](alert_porture.png)