# Monitoring

## Tại sao cần Monitoring?

- Làm sao để giám sát được tài nguyên, cơ sở hạ tầng trong doanh nghiệp?
- Khi có hàng trăm, hay hàng ngàn `server` và trên mỗi `server` có thể chạy nhiều `service` khác nhau
sử dụng các `ports`, `processes` khác nhau. Vậy thì làm sao để phát hiện được ngay khi `server` có
vấn đề để nhanh chóng xử lý để tránh gây ra những hậu quả lớn hơn?
- Đối với các hệ thống lớn có hàng trăm nghìn hay hàng triệu người dùng, thì khi có vấn đề mà không
nhanh chóng khắc phục sẽ gây ra thất thoát lớn. Ví dụ như hệ thông e-commerce, trong 1 phút có biết
bao nhiêu người mua hàng. Nên dù hệ thống chậm hoặc treo vài phút cũng ảnh hưởng rất lớn đến trải
nghiệm người dùng cũng như doanh sô, và uy tín của doanh nghiệp. Vì vậy, không thể đợi đến lúc hệ
thống gặp vấn đề hoặc đợi khách hàng phản ánh thì chúng ta mới tiến hành đi sửa được.

**Monitoring** giúp nhanh chóng phát hiện lỗi và giảm thiểu tài nguyên con người dành cho việc giám
sát hệ thống.

## Monitoring là gì?

Monitoring là hệ thống đảm bảo rằng:

- Tự động giám sát các `server` và cảnh báo khi `server` bị lỗi như: đầy ram, cao CPU, mạng chậm, etc.
Giúp chúng ta nắm được tình hình và đưa ra các phương án khắc phục trước khi tài nguyên gặp những vấn đề nặng hơn.

## Zabbix

Zabbix cung cấp:

- Nhiều loại `template` có sẵn giúp quản trị hệ thống mà không cần biết quá chuyên sâu.
- Tích hợp nhiều nền tảng giao tiếp: Telegram, slack, email, ...

Tương tự như `Jenkins`, `Zabbix` cũng có `zabbix server` và `zabbix agent`.

### Zabbix server

1. Tạo và cài đặt `server` để `host zabbix`.
2. Cài đặt `zabbix server`
    - Truy cập [Zabbix Packages Download](https://www.zabbix.com/download) và làm theo hướng dẫn.
    - `Add host` cho các `server` cần được `monitoring`.
3. Cấu hình `Apache` để thay đổi đường dẫn zabbix từ `http://zabbix-server.remote/zabbix` thành
`http://zabbix-server.remote`:

    ```bash
    vi /etc/apache2/sites-available/000-default.conf
    ```

    ```ini
    DocumentRoot /usr/share/zabbix/
    ```

    ```bash
    systemctl restart apache2
    ```

4. Cấu hình `zabbix` bằng `UI`.
5. Đăng nhập với `username: Admin` và `password: zabbix`.
6. Cài đặt `zabbix agent` trên các `server` cần được `monitoring` và thêm vào trong `zabbix`.

- Cài đặt `zabbix agent` trên các `slave server`:
  - Làm theo hướng dẫn trên [Zabbix Packages Download](https://www.zabbix.com/download). Chỉ cần cài `zabbix-agent` package thôi.
  - Cấu hình `zebbit agent` ở file `/etc/zabbix/zabbix-agentd.conf`:

    ```ini
    Server=<zabbix-url>
    # Server=zabbix-server.remote
    ServerActive=<zabbix-url>
    # ServerActive=zabbix-server.remote
    Hostname=<server_name-IP_address>
    # Hostname=dev-server_172.16.25.99
    ```

    ```bash
    systemctl restart zabbix-agent
    ```

- `Create host` trên `zabbix server`:
  - `Templates`: Linux by Zabbix agent.
  - `Host groups`: Linux servers.
  - `Interfaces`: Agent.

### Item

`Item` là thông tin được giám sát và thu thập dựa trên điều kiện từ các `agent`.

`Data collection > Hosts > dev-server > Items > Create item`:

- `Name`: shoeshop service on server 172.16.25.99 running on port 8080 is unavailable.
- `Key`: net.tcp.listen[8080].
- `Type of information`: Numeric. Bởi vì `key: net.tcp.listen` trả về giá trị 0 hoặc 1.
- `Update interval`: 10s. Sẽ có nhiều tư duy để thiết lập được thông số này.

### Trigger

`Trigger` là sẽ kích hoạt các cảnh báo khi giá trị `Item` thoã mãn những điều kiện được đặt ra.

`Data collection > Hosts > dev-server > Triggers > Create trigger`:

- `Name`: shoeshop service on server 172.16.25.99 running on port 8080 is unavailable.
- `Severity`: tuỳ thuộc vào mức độ quan trọng.
- `Expression`: Đặt ra điều kiện để `trigger` dựa trên giấ trị của `item`.
