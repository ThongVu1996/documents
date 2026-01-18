# [Lab #4] Refactoring - Làm chủ Locals & Tổ chức biến chuyên nghiệp

---
## Mục lục
- [1. Giới thiệu: Tại sao code lại "bốc mùi"?](#1-giới-thiệu-tại-sao-code-lại-bốc-mùi)
  - [Minh họa: The Variable Refinery (Nhà máy xử lý biến)](#minh-họa-the-variable-refinery-nhà-máy-xử-lý-biến)
- [2. Mục tiêu bài Lab](#2-mục-tiêu-bài-lab)
- [3. Chuẩn bị (Prerequisites)](#3-chuẩn-bị-prerequisites)
- [4. Cấu trúc File Lab (Best Practice)](#4-cấu-trúc-file-lab-best-practice)
  - [File 1: `variables.tf` (Định nghĩa đầu vào)](#file-1-variablestf-định-nghĩa-đầu-vào)
  - [File 2: `main.tf` (Xử lý logic & Tạo tài nguyên)](#file-2-maintf-xử-lý-logic--tạo-tài-nguyên)
  - [File 3: `outputs.tf` (Kết quả báo cáo)](#file-3-outputstf-kết-quả-báo-cáo)
- [5. Phân tích chuyên sâu: Variable vs Local](#5-phân-tích-chuyên-sâu-variable-vs-local)
- [6. Thao tác thực hành](#6-thao-tác-thực-hành)
  - [Bước 1: Khởi tạo và Kiểm tra](#bước-1-khởi-tạo-và-kiểm-tra)
  - [Bước 2: Thử thách thay đổi biến số](#bước-2-thử-thách-thay-đổi-biến-số)
  - [Bước 3: Kiểm chứng Output](#bước-3-kiểm-chứng-output)
  - [Bước 4: Dọn dẹp](#bước-4-dọn-dẹp)
- [7. Tổng kết](#7-tổng-kết)

---
## 1. Giới thiệu: Tại sao code lại "bốc mùi"?

Chào mừng bạn đến với Lab #4. Nếu 3 bài trước chúng ta tập trung vào "làm cho chạy" (Make it work), thì hôm nay chúng ta sẽ học cách "làm cho sạch" (Make it clean).

Hãy tưởng tượng công ty yêu cầu bạn đặt tên Server theo quy tắc khắt khe: `[TênDựÁn]-[MôiTrường]-[LoạiServer]`. Nếu không có kỹ thuật, file code của bạn sẽ tràn ngập những chuỗi lặp đi lặp lại như thế này:

```hcl
# Code "xấu" (Hard to maintain)
resource "aws_instance" "web" {
  tags = { Name = "${var.project}-${var.env}-web" }
}
resource "aws_instance" "db" {
  tags = { Name = "${var.project}-${var.env}-db" }
}
```

Nếu sếp đổi ý muốn thêm Region vào tên, bạn sẽ phải đi sửa 100 chỗ. Rất dễ sai sót!

**Giải pháp:** Sử dụng **Locals (Local Values)**. Hãy coi `variable` là nguyên liệu thô (thịt, rau), còn `local` là món đã sơ chế (thịt xay, rau thái sẵn) để dùng ngay vào việc nấu nướng (`resource`).

### Minh họa: The Variable Refinery (Nhà máy xử lý biến)
![Terraform Lab 4 Refinery](./terraform_lab_4_refinery.svg)

---

## 2. Mục tiêu bài Lab

1.  **Hiểu bản chất Locals:** Phân biệt rõ ràng với Variables.
2.  **Refactoring (Tái cấu trúc):** Chia nhỏ file, biến code "rối" thành code "sạch".
3.  **Outputs thực dụng:** Xuất ra các thông tin cần thiết nhất để debug và kết nối.

---

## 3. Chuẩn bị (Prerequisites)

*   **Terraform:** Version >= 1.0.0.
*   **AWS CLI:** Đã cấu hình credentials.
*   **Thư mục:** Tạo thư mục mới `terraform-lab-4`.

---

## 4. Cấu trúc File Lab (Best Practice)

Thay vì viết tất cả vào 1 file, từ bài này chúng ta sẽ tập chia file để quản lý chuyên nghiệp hơn.

### File 1: `variables.tf` (Định nghĩa đầu vào)
Nơi khai báo các "nguyên liệu thô" cho dự án.

```hcl
variable "project_name" {
  description = "Ten du an"
  type        = string
  default     = "SuperApp"
}

variable "environment" {
  description = "Moi truong trien khai (dev, prod, staging)"
  type        = string
  default     = "dev"
}

variable "instance_type" {
  description = "Loai EC2 Instance"
  type        = string
  default     = "t3.micro"
}
```

### File 2: `main.tf` (Xử lý logic & Tạo tài nguyên)
Đây là "nhà bếp" chính. Hãy chú ý khối `locals` - ngôi sao của bài hôm nay.

```hcl
provider "aws" {
  region = "ap-southeast-1"
}

# -----------------------------------------------------------
# 1. DATA SOURCE (Ôn lại bài cũ)
# Tự động tìm AMI Ubuntu mới nhất
# -----------------------------------------------------------
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# -----------------------------------------------------------
# 2. LOCALS (Trái tim của bài Lab)
# Nơi "chế biến" các biến thô thành giá trị chuẩn hóa
# -----------------------------------------------------------
locals {
  # Logic: Ghép tên dự án và môi trường thành một chuỗi duy nhất
  # Ví dụ: SuperApp-dev
  full_server_name = "${var.project_name}-${var.environment}"
  
  # Bạn có thể định nghĩa thêm nhiều local khác
  common_tags = {
    ManagedBy = "Terraform"
    Project   = var.project_name
  }
}

# -----------------------------------------------------------
# 3. RESOURCES
# Sử dụng giá trị từ Locals thay vì ghép chuỗi thủ công
# -----------------------------------------------------------
resource "aws_instance" "my_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    # Gọi local bằng từ khóa "local." (không có chữ s)
    Name = local.full_server_name
    
    # Kết hợp thêm các tag khác nếu muốn
    Code = "Lab-4-Refactoring"
  }
}
```

### File 3: `outputs.tf` (Kết quả báo cáo)
Phần này giúp bạn kiểm tra kết quả ngay lập tức mà không cần mò mẫm trên AWS Console.

```hcl
# 1. In ra tên Server để kiểm tra biến nào thắng (Debug xem tên ghép đúng chưa)
output "final_server_name" {
  description = "Ten server cuoi cung duoc tao ra"
  value       = "Server Name: ${local.full_server_name}"
}

# 2. In ra IP Public (Thông tin quan trọng để connect)
output "public_ip" {
  description = "IP Public cua Server"
  value       = aws_instance.my_server.public_ip
}

# 3. In ra câu lệnh SSH (Tiện ích cho người dùng copy-paste ngay)
output "ssh_command" {
  value = "ssh ubuntu@${aws_instance.my_server.public_ip}"
}
```

---

## 5. Phân tích chuyên sâu: Variable vs Local

| Đặc điểm | Variable (`var`) | Local (`local`) |
| :--- | :--- | :--- |
| **Vai trò** | Tham số đầu vào (Input) | Biến nội bộ (Internal) |
| **Thay đổi** | Người dùng có thể thay đổi (qua CLI, `.tfvars`) | User KHÔNG thể thay đổi trực tiếp |
| **Mục đích** | Nhận "nguyên liệu thô" | Tính toán, sơ chế, chuẩn hóa logic |

**Quy tắc vàng:** Nếu giá trị đó cần người dùng nhập vào -> Dùng **Variable**. Nếu giá trị đó được tính toán từ các giá trị khác -> Dùng **Local**.

---

## 6. Thao tác thực hành

### Bước 1: Khởi tạo và Kiểm tra
```bash
terraform init
terraform plan
```
Hãy nhìn vào Plan: Bạn sẽ thấy tag `Name` dự kiến là `SuperApp-dev`. Terraform đã tự tính toán trong Locals trước khi chạy.

### Bước 2: Thử thách thay đổi biến số
Hãy thử chạy lệnh apply nhưng thay đổi môi trường sang `prod` xem locals xử lý thế nào nhé:
```bash
terraform apply -var="environment=prod" -auto-approve
```

### Bước 3: Kiểm chứng Output
Sau khi chạy xong, hãy nhìn vào phần Outputs ở cuối màn hình:
*   `final_server_name`: Phải là `Server Name: SuperApp-prod`. Điều này chứng tỏ biến `prod` bạn truyền vào đã "thắng" default và được Locals xử lý chính xác.
*   `ssh_command`: Copy nguyên dòng lệnh `ssh ubuntu@...` dán vào terminal để kết nối thử. Tiện hơn hẳn việc phải tự gõ lại IP đúng không?

### Bước 4: Dọn dẹp
```bash
terraform destroy -auto-approve
```

---

## 7. Tổng kết

Qua bài Lab này, bạn đã học được:
1.  **Locals:** Cách "chế biến" dữ liệu thô thành dữ liệu tinh.
2.  **Refactoring:** Cách tách file `variables.tf`, `outputs.tf` để dự án gọn gàng, dễ đọc.
3.  **User Experience:** Cách viết Outputs sao cho hữu ích nhất cho người dùng cuối.
