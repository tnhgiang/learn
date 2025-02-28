# Git

## Git là gì?

Git là một Version Control System giúp:

- Nhiều người cùng phát triển.
- Theo dõi các thay đổi
- Kiểm soát truy cập
- Quản lý phiên bản
- Sao lưu và phục hồi

## Cài đặt Gitlab server

Mỗi doanh nghiệp thường cài đặt `git server` riêng để đảm bảo tính bảo mật, tốc độ và khả năng kiểm soát.

Nhiều doanh nghiệp dùng `gitlab` làm `git server`, vì `gitlab` được tạo ra để hướng đến doanh nghiệp.

1. Thiết lập server.
2. Googling `gitlab ee packages` và chọn phiên bản phù hợp.
3. Chạy các lệnh được hướng dẫn trong document: lệnh `curl` và `apt install`
4. Tạo `domain` cho gitlab-server:

    ```bash
    vi /etc/hosts
    ```

    ```ini
    ; Append new line
    <ip> <domain name>
    ```

5. Cấu hình `gitlab`:

    ```bash
    vi /etc/gitlab/gitlab.rb
    ```

    ```gitlab
    external_url <url>
    ```

6. Reconfigure `gitlab`:

    ```bash
    gitlab-ctl reconfigure
    ```

7. Thử đăng nhập với account mặc định:
    - user: root
    - password: chạy lệnh `cat /etc/gitlab/initial_root_password`

**Chú ý**: Do domain tự tạo (không tồn tại trên google) nên cần phải `add host` ở các máy client.

**Chú ý**: Nên `disable` các mục `Sign-up enable` và `Require admin approval for new sign-ups` ở
trong `Settings > General > Sign-up restrictions`  vì khi có user mới `root` hay admin sẽ là người
tạo, chứ không để ai khác tự tạo.

**Chú ý**: `Disable` mục `Default to Auto DevOps pipeline for all projects` ở trong
`Settings > CI/CD > Continuous Integration and Deployment`.

## Triển khai Git workflow

![git workflow](./assets/02-git-workflow.png)
**Chú ý**: Cần phải thiết lập `Protected branches` để ngăn việc tự ý `merge` hay `commit` vào những
`branches` quan trọng. `Allow to merge`: Maintainer, còn `Allow to push`: No one. Tuy nhiên những
thiết lập này cũng nên linh hoạt tuỳ vào văn hoá công ty.
