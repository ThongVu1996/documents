# PHẦN 1: CÀI ĐẶT DOCKER & BUILDX (LÀM TỪ ĐẦ PHẦN 1: CÀI ĐẶT DOCKER & BUILDX (LÀM TỪ ĐẦU)
## Bước 1: Cập nhật hệ thống và thêm Repository của Docker
```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg
    # Thêm khóa GPG
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # Thêm Repository chính thức (Tự động nhận diện kiến trúc ARM/AMD)
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Bước 2: Cài đặt Docker Engine và Buildx Plugin
```bash
    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Bước 3: Cấu hình quyền và khởi động dịch vụ
```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    sudo usermod -aG docker $USER
    # Chạy lệnh dưới để áp dụng quyền ngay lập tức mà không cần Log out
    newgrp docker
```

---

# PHẦN 2: CẤU HÌNH GIẢ LẬP ĐA NỀN TẢNG (QEMU)
Vì bạn dùng máy ARM để build cho Intel (AMD64), bạn cần "thông dịch viên" QEMU.

## Bước 4: Cài đặt QEMU binfmt
```bash
    sudo apt-get install -y qemu-user-static binfmt-support
    # Đăng ký trình giả lập với Docker (Sử dụng bản mới nhất của tonistiigi)
    docker run --privileged --rm tonistiigi/binfmt --install all
```

## Bước 5: Thiết lập Builder cố định (Sử dụng vĩnh viễn) Tạo một "cỗ máy build" riêng biệt hỗ trợ đa nền tảng:
```bash
    docker buildx create --name mybuilder --driver docker-container --use
    docker buildx inspect --bootstrap
```

--- 

# PHẦN 3: CÁC BƯỚC TEST (ĐỂ KIỂM TRA ĐÃ CHUẨN CHƯA)
## Bước 6: Kiểm tra Builder đã nhận AMD64 chưa
```bash
   docker buildx ls
```
Kết quả phải thấy cột Platforms có: linux/amd64, linux/arm64...

## Bước 7: Tạo file test và Build thử
```bash
    # Tạo thư mục test
    mkdir test-arch && cd test-arch

    # Tạo Dockerfile mẫu
    echo "FROM alpine:latest" > Dockerfile
    echo 'CMD ["uname", "-m"]' >> Dockerfile

    # Thực hiện build cho cấu trúc AMD64 (Lệnh quan trọng)
    docker buildx build --platform linux/amd64 -t test-image:amd64 --load .

```

## Bước 8: Kiểm tra Image vừa tạo Có 2 cách để xác nhận:
### Kiểm tra thông số kỹ thuật:
```bash
    docker image inspect test-image:amd64 | grep Architecture
```
Phải hiện: "Architecture": "amd64"
### Chạy thử container:
```bash
    docker run --rm test-image:amd64
```
Kết quả phải trả về: x86_64 (Đây là kiến trúc của Intel/AMD).
