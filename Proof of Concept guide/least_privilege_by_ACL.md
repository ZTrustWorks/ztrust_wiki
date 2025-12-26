# Node chỉ truy cập đúng dịch vụ được cấp quyền

## 1. Tạo chính sách

Tab `Network Policy` -> ACLs -> Create ACL

1. Cho phép các thiết bị của nhân sự thuộc nhóm data_entry và thiết bị của nhân sự dongtv28 truy cập hệ thống nhập liệu 10.110.84.110 port 3000, 8000, 8005
   ```commandline
   Name: Allow Data Entry
   Description: Cho phép nhóm nhân viên nhập liệu truy cập hệ thống 10.110.84.110 port 3000, 8000, 8005
   Protocol: ALL (TCP, UDP)
   ACL Effective Time Range: None (Do không có yêu cầu về lịch truy cập)
   Source: User:dongtv28, Group:data_entry
   Destinations: 10.110.84.110:3000, 10.110.84.110:8000, 10.110.84.110:8005
   ```
   **Sources/Destinations hỗ trợ các kiểu:** `IP/CIDR`, `Group`, `Host`, `User `

   ![img.png](images/create_acl.png)



2. Click `Save Changes` để lưu lại chính sách
3. Click `Comit Changes`, giao diện hiển thị các dòng chính sách thay đổi, quản trị viên cần điền lý do comit và thực hiện click `Confirm & Submit`
![img.png](images/comit_acl.png)

**Chú ý mặc định các chính sách mới tạo sẽ không tự động enabled**

Cần click `Enabled` để đảm bảo chính sách được áp dụng (quá trình đợi diễn ra trong khoảng 1 phút)
![img_1.png](images/enabled_acl.png)

