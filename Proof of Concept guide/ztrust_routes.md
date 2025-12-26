# Thực hiện phê duyệt routes
Tab `Router Gateway` -> Chọn Action với từng node gateway cần phê duyệt

1. Chọn node gateway muốn đang chờ approve, sau đó nhấn nút `Appover`
![img.png](images/select_gateway.png)
2. Chọn subnet route muốn phê duyệt, sau đó nhấn nút `Approve`
![img.png](images/approve_routes.png)
3. Đợi ít phút để cập nhật thông tin cho toàn mạng. Tại thời điểm hiện tại, các lưu lượng khi đến các subnet routes được khai báo sẽ được đi qua node gateway phụ trách subnet route đó

**Chú ý:**
- Trong trường hợp khai báo trùng subnet route, ztrust sẽ ưu tiên dùng node gateway khai báo đầu tiên
- Ztrust ưu tiên sử dụng node gateway có khai báo subnet nhỏ hơn. Ví dụ: Node gateway A khai báo route cho subnet 10.10.10.0/16, node gateway B khai báo route cho subnet 10.10.10.0/24 => Ưu tiên B