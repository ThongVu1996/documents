# CÀI HARBOR CHO CHIP ARM (apple silicon)

---

## Tải bản cài `Offline Installer`

```bash
    wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz
    ```

```bash
    tar xvzf harbor-offline-installer-v2.10.0.tgz
```

```bash
    cd harbor
```
## Chỉnh sửa file harbor.yml

```bash
    cp harbor.yml.tpl harbor.yml
    vi harbor.yml
```
- Đổi hostname thành domain của bản thân 
- Đổi port thành 8080
- Comment tất cả về https
- Đổi pass của harbor ở `harbor_admin_password`

## Chạy script để cài đặt và khởi động
```bash
    sudo ./prepare
```
```bash
    sudo docker-compose up -d
```

