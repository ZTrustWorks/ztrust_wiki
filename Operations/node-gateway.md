# Node Gateway & Subnet Routers

![img.png](images/node-gateway-topology.png)

## Nhu cầu
- Kiểm soát quyền truy cập của từng thiết bị nhân viên vào các vùng mạng nội bộ.
- Một thiết bị có thể truy cập vào một hoặc nhiều subnet.
- Mỗi subnet chỉ cần một thiết bị chạy Ztrust Agent (Tailscale client) đóng vai trò gateway, không yêu cầu cài đặt trên toàn bộ server.
- Triển khai gateway để đăng ký subnet vào Ztrust Network

**Ví dụ triển khai**

| Thiết bị           | Vai trò          | IP Ztrust Network | Internal Subnet                  |
| ------------------ | ---------------- |-------------------|----------------------------------|
| **Ztrust Agent 1** | Client nhân viên | 100.x.y.55        | N/A                              |
| **Ztrust Agent 2** | Client nhân viên | 100.x.y.66        | N/A                              |
| **Ztrust Agent 3** | Subnet Router    | 100.x.y.111       | 10.10.x.0/24 (Marketing Subnet)  |
| **Ztrust Agent 4** | Subnet Router    | 100.x.y.222       | 10.11.x.0/24 (Accountant Subnet) |


- Cài đặt Ztrust Agent trên 4 thiết bị, trong đó: 
  - Ztrust Agent 1, Ztrust Agent 2 là thiết bị của nhân viên. 
  - Ztrust Agent 3 là thiết bị (có thể là máy tính hoặc máy chủ) thuộc Marketing Subnet
  - Ztrust Agent 4 là thiết bị (có thể là máy tính hoặc máy chủ) thuộc Accountant Subnet
- Ztrust Agent 1 có quyền truy cập vào các tài nguyên của Accountant Subnet và Marketing Subnet, Ztrust Agent 3 là thiết bị (có thể là máy tính hoặc máy chủ) thuộc Marketing Subnet
- Ztrust Agent 2 chỉ có quyền truy cập vào các tài nguyên của Marketing Subnet

## Các bước cấu hình
### 1. Quản trị viên tạo tài khoản riêng phụ trách từng node gateway
- Chú ý khi tạo tài khoản node gateway cần chọn `Type = Server` và số lượng thiết bị tối đa là 1 hoặc 2 (trong trường hợp sử dụng LB) để đảm bảo việc `Chống chối bỏ`.
- Tên của tài khoản nên có một cấu trúc cố định để dễ quản lý và rà soát khi có lỗi. VD: `{subnetname}_{hostname}_gateway_01` => marketing_haproxy_gateway_01, accountant_haproxy_gateway_01
![img.png](images/add-user-node-gateway.png)
### 2. Quản trị viên tạo Auth Key cho từng tài khoản phụ trách node gateway
- Khi tạo Auth Key cho tài khoản cần xác định:
  - **Expire**: Thời gian hết hạn của key
  - **Reusable**: Cho phép key có thể được dùng lại trong phiên đăng nhập khác của node (trong trường hợp cần đăng nhập lại hoặc đăng nhập cho LB)
  - **Ephemeral** (Không khuyến nghị bật cho node gateway): Xác định tạo khóa cho node tạm thời, ở trường hợp này key sẽ tự hết hạn hoặc bị xóa khi node ngắt kết nối

![img.png](images/create-auth_key.png)
### 3. Tại từng Note Gateway, thực hiện đăng nhập các tài khoản vừa tạo và đăng ký subnet router cho Ztrust Network
**Đảm bảo các node gateway đều đang cho phép ip forward (Bắt buộc).**
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

```
**Trên Ztrust Agent 3 (Marketing Subnet)**
```bash
sudo tailscale up \
--login-server https://ztcontroller.{org_domain} \
--auth-key {Auth Key của tài khoản marketing_haproxy_gateway_01 vừa tạo ở bước 2} \
--advertise-routes=10.10.x.0/24 \
--accept-dns=false \
--force-reauth
```
**Trên Ztrust Agent 4 (Accountant Subnet)**
```bash
sudo tailscale up \
--login-server https://ztcontroller.{org_domain} \
--auth-key {Auth Key của tài khoản accountant_haproxy_gateway_01 vừa tạo ở bước 2} \
--advertise-routes=10.11.x.0/24 \
--accept-dns=false \
--force-reauth
```

### 4. Tại Ztrust Controller, thực hiện phê duyệt cho phép các subnet router vận hành cho mạng
![img.png](images/accept_node.png)
![img.png](images/accept_node_2.png)

**Đợi ít phút để thông tin được cập nhật cho toàn mạng, lúc này trạng thái subnet router sẽ được chuyển sang là `Serving`**
![img.png](images/accept_node3.png)

### 5. Tại Ztrust Controller, thực hiện cấu hình chính sách cho phép truy cập
