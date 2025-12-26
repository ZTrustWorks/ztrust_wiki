# Cấp quyền truy cập tài nguyên trong thời gian giới hạn
# Thay đổi ACL có hiệu lực tức thì

## 1. Tạo chính sách

Tab `Network Policy` -> ACLs -> Chọn ACL muốn sửa

1. Nội dung ACL muốn sửa như sau:
![img.png](images/edit_acl.png)
1. Sửa quyền nhân sự dongtv28 không còn được truy cập 10.110.84.110 port 3000, 8000, 8005
   ```commandline
   Name: Allow Data Entry
   Description: Cho phép nhóm nhân viên nhập liệu truy cập hệ thống 10.110.84.110 port 3000, 8000, 8005
   Protocol: ALL (TCP, UDP)
   ACL Effective Time Range: None (Do không có yêu cầu về lịch truy cập)
   Source: Group:data_entry (loại bỏ User:dongtv28)
   Destinations: 10.110.84.110:3000, 10.110.84.110:8000, 10.110.84.110:8005
   ```
   **Sources/Destinations hỗ trợ các kiểu:** `IP/CIDR`, `Group`, `Host`, `User `
    ![img.png](images/edit_acl2.png)
2. Click `Save Changes` để lưu lại chính sách. Thông tin chính sách sau khi loại bỏ người dùng `dongtv28`
    ![img.png](images/edit_acl3.png)

3. Click `Comit Changes`, giao diện hiển thị các dòng chính sách thay đổi, quản trị viên cần điền lý do comit và thực hiện click `Confirm & Submit`
![img.png](images/edit_acl4.png)

**Kết quả**
- Nhân sự dongtv28 **trước khi** bị quản trị viên xóa quyền truy cập
![img.png](images/output_acl.png)
- Nhân sự dongtv28 **sau khi** bị quản trị viên xóa quyền truy cập. Ngay lập tức không thể truy cập dịch vụ được nữa:
![img.png](images/edit_acl5.png)
