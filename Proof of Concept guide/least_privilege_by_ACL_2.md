# Node chỉ truy cập đúng dịch vụ được cấp quyền

## 1. Tạo chính sách

Tab `Network Policy` -> ACLs -> Create ACL

1. Cho phép các thiết bị được chỉ định truy cập hệ thống portal 10.110.240.48 port 80, 443
   
    **Sources/Destinations hỗ trợ các kiểu:** `IP/CIDR`, `Group`, `Host`, `User `
    ```commandline
   Name: Allow Data Entry
   Description: Cho phép các thiết bị được chỉ định truy cập hệ thống portal 10.110.240.48 port 80, 443
   Protocol: ALL (TCP, UDP)
   ACL Effective Time Range: None (Do không có yêu cầu về lịch truy cập)
   Source: 100.64.17.76
   Destinations: 10.110.240.48:80, 10.110.240.48:443
   ```
         
    Người dùng `dongtv28` có hai thiết bị với IP Ztrust Network là `100.64.17.76` và `100.64.78.145`. Chỉ thiết bị có IP `100.64.17.76` được phép truy cập, thiết bị còn lại thì bị cấm.
   ![img.png](images/device_acl.png)

3. Click `Save Changes` để lưu lại chính sách
4. Click `Comit Changes`, giao diện hiển thị các dòng chính sách thay đổi, quản trị viên cần điền lý do comit và thực hiện click `Confirm & Submit`
![img.png](images/devices_acl.png)



Cần click `Enabled` để đảm bảo chính sách được áp dụng (quá trình đợi diễn ra trong khoảng 1 phút)
![img_1.png](images/device_Acl4.png)

**Kết quả**
- Thiết bị 100.64.17.76 của dongtv28 có thể truy cập được dịch vụ
![img_1.png](images/devices_acl33.png)
- Thiết bị 100.64.78.145 của dongtv28 không thể truy cập được dịch vụ

