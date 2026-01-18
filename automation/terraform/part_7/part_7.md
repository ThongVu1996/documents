# [Terraform 107] Data Sources - Cầu nối với thế giới Cloud thực tế

---

## Mục lục
- [Lời mở đầu: Đừng xây lại những gì đã có](#lời-mở-đầu-đừng-xây-lại-những-gì-đã-có)
- [1. Data Source là gì?](#1-data-source-là-gì)
- [2. Cú pháp và cách hoạt động](#2-cú-pháp-và-cách-hoạt-động)
- [3. Ví dụ thực chiến: Tìm kiếm và Sử dụng](#3-ví-dụ-thực-chiến-tìm-kiếm-và-sử-dụng)
  - [Tìm ID của một VPC có sẵn](#tìm-id-của-một-vpc-có-sẵn)
  - [Lấy danh sách các Availability Zones](#lấy-danh-sách-các-availability-zones)
- [4. Phân biệt Resource và Data Source](#4-phân-biệt-resource-và-data-source)
- [5. Tổng kết: Bảng đối chiếu nhanh](#5-tổng-kết-bảng-đối-chiếu-nhanh)
- [Lời kết](#lời-kết)

---

## Lời mở đầu: Đừng xây lại những gì đã có

Trong các bài trước, chúng ta luôn bắt đầu bằng việc tạo mới mọi thứ. Nhưng thực tế đi làm không phải lúc nào cũng là "đất trống". Bạn thường xuyên phải:
*   Gắn một con Server mới vào một cái VPC đã có sẵn do team Network tạo.
*   Tìm chính xác ID của bản cài đặt (AMI) mới nhất của Ubuntu để triển khai.
*   Lấy thông tin về KeyPair mà sếp đã tạo thủ công trên Console.

Thay vì ngồi copy-paste các ID khô khan vào code (vừa dễ sai, vừa khó bảo trì), chúng ta sẽ dùng **Data Sources** để "soi" thông tin trực tiếp từ Cloud.

---

## 1. Data Source là gì?

Data Source cho phép Terraform sử dụng các thông tin được định nghĩa bên ngoài Terraform, hoặc được tạo bởi một cấu hình Terraform khác.

*   **Góc nhìn Dev (JS):** Giống như một lệnh `GET` request tới API để lấy dữ liệu về mà không làm thay đổi dữ liệu đó.
*   **Góc nhìn System:** Giống như lệnh `grep` hoặc `find` để tìm kiếm thông tin hệ thống hiện tại (như `ip addr` hay `lsblk`) trước khi thực hiện script.

### Minh họa: Cloud Sonar (Máy quét đám mây)
![Terraform Cloud Sonar](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/part_7/terraform_cloud_sonar.svg)

---

## 2. Cú pháp và cách hoạt động

Cú pháp của Data Source bắt đầu bằng từ khóa `data`.

```hcl
data "<PROVIDER_TYPE>" "<LOCAL_NAME>" {
  # Các bộ lọc (Filters) để tìm kiếm
}
```

Khi chạy `terraform plan`, Terraform sẽ gửi một yêu cầu truy vấn tới Cloud Provider để tìm kiếm đối tượng khớp với các tiêu chí bạn đặt ra. Nếu tìm thấy, nó sẽ mang toàn bộ thông số của đối tượng đó về để bạn sử dụng.

---

## 3. Ví dụ thực chiến: Tìm kiếm và Sử dụng

### Tìm ID của một VPC có sẵn
Thay vì gõ chết `vpc-0123456789`, bạn hãy tìm nó theo Tag Name.

**Cú pháp HCL:**
```hcl
data "aws_vpc" "existing_vpc" {
  filter {
    name   = "tag:Name"
    values = ["Production-VPC"]
  }
}

# Sử dụng nó để tạo Subnet mới
resource "aws_subnet" "new_subnet" {
  vpc_id     = data.aws_vpc.existing_vpc.id # Trích xuất ID từ dữ liệu tìm được
  cidr_block = "10.0.1.0/24"
}
```

### Lấy danh sách các Availability Zones
Bạn muốn biết vùng `ap-southeast-1` có những Zone nào đang hoạt động để phân bổ server.

**Cú pháp HCL:**
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# Sử dụng Output để xem danh sách tìm được
output "zones" {
  value = data.aws_availability_zones.available.names
}
```

---

## 4. Phân biệt Resource và Data Source

Anh em rất hay nhầm lẫn hai cái này khi mới bắt đầu. Hãy nhớ quy tắc đơn giản:

1.  **Resource (`resource`):** Bạn muốn **Quản lý** (Tạo, Sửa, Xóa). Terraform có "quyền sinh sát".
2.  **Data Source (`data`):** Bạn muốn **Đọc** thông tin. Terraform chỉ là "người quan sát", không có quyền thay đổi đối tượng đó.

---

## 5. Tổng kết: Bảng đối chiếu nhanh

| Đặc điểm | Resource | Data Source |
| :--- | :--- | :--- |
| **Từ khóa** | `resource` | `data` |
| **Mục đích** | Tạo/Cập nhật hạ tầng | Truy vấn/Lấy thông tin |
| **Trạng thái** | Có lưu trong tfstate | Thường được fetch mới khi chạy |
| **Góc nhìn Dev** | `POST` / `PUT` / `DELETE` | `GET` |
| **Góc nhìn System** | `mkdir`, `useradd` | `ls`, `cat /etc/passwd` |

---

## Lời kết

Vậy là chúng ta đã đi qua trọn bộ các khái niệm cốt lõi nhất của Terraform:
1.  **Core:** HCL Syntax, Declarative Mindset.
2.  **Variables:** Biến hóa linh hoạt.
3.  **Logic:** Xử lý điều kiện thông minh.
4.  **Meta-Arguments:** Nhân bản tự động.
5.  **Data Sources:** Kết nối với thế giới có sẵn.

Nắm vững 5 trụ cột này, bạn đã đủ tự tin để viết những module Terraform chuyên nghiệp, dễ đọc và dễ bảo trì. Chúc các bạn thành công trên con đường Infrastructure as Code!

