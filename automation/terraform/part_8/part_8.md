# [Terraform 108] Module hóa & Cấu trúc dự án thực tế

---

## Mục lục
- [Lời mở đầu: Từ "Spaghetti Code" đến kiến trúc module](#lời-mở-đầu-từ-spaghetti-code-đến-kiến-trúc-module)
- [1. Module là gì?](#1-module-là-gì)
- [2. Cấu trúc thư mục chuẩn của một dự án](#2-cấu-trúc-thư-mục-chuẩn-của-một-dự-án)
- [3. Cách đóng gói và gọi Module](#3-cách-đóng-gói-và-gọi-module)
- [4. Best Practices: Quy tắc vàng cho code "sạch"](#4-best-practices-quy-tắc-vàng-cho-code-sạch)
- [Lời kết: Chuẩn bị cho Lab thực chiến](#lời-kết-chuẩn-bị-cho-lab-thực-chiến)

---

## Lời mở đầu: Từ "Spaghetti Code" đến kiến trúc module

Khi mới bắt đầu, chúng ta thường viết tất cả code vào một file `main.tf` duy nhất. Nhưng khi dự án lớn dần với hàng trăm tài nguyên, file đó sẽ trở thành một "nồi lẩu thập cẩm" cực kỳ khó quản lý.

Trong lập trình, chúng ta có hàm (functions) hay lớp (classes). Trong Terraform, chúng ta có **Module**. Module giúp bạn đóng gói các thành phần hạ tầng thành những khối lắp ghép (như Lego), giúp code dễ đọc, dễ bảo trì và cực kỳ chuyên nghiệp.

---

## 1. Module là gì?

Nói đơn giản, một Module là một tập hợp các file `.tf` nằm chung trong một thư mục.

*   **Root Module:** Là thư mục chính nơi bạn đứng để gõ lệnh `terraform apply`.
*   **Child Module:** Là các thư mục con được "đóng gói" để Root Module gọi vào sử dụng.

| Góc nhìn | Liên tưởng |
| :--- | :--- |
| **Dev (JS)** | Giống như việc bạn tách code thành các **Component** trong React hoặc Module trong Node.js để `import` vào ứng dụng chính. |
| **System** | Giống như việc viết một Shell Script tổng để gọi các Sub-scripts khác nhau cho từng nhiệm vụ (Cài Web, Cài DB, Firewall). |

### Minh họa: Cây Component (Modular Tree)
![Terraform Modular Tree](./terraform_modular_tree.svg)

---

## 2. Cấu trúc thư mục chuẩn của một dự án

Một dự án Terraform chuyên nghiệp thường tuân thủ cấu trúc "tam trụ" để tách biệt trách nhiệm:

| File | Vai trò | Liên tưởng (Dev JS) |
| :--- | :--- | :--- |
| **main.tf** | Khai báo các Resource chính. | Logic xử lý chính. |
| **variables.tf** | Định nghĩa các tham số đầu vào. | Props hoặc tham số hàm. |
| **outputs.tf** | Trích xuất dữ liệu trả về. | Giá trị `return` của hàm. |

**Cấu trúc thư mục thực tế:**
```plaintext
my-project/
├── main.tf           # Root module: Gọi các linh kiện bên dưới
├── variables.tf      # Biến đầu vào cho toàn dự án
├── outputs.tf        # Kết quả đầu ra tổng thể
└── modules/          # "Kho linh kiện" tái sử dụng
    └── vpc/          # Module quản lý mạng
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 3. Cách đóng gói và gọi Module

Giả sử bạn đã viết xong bộ code tạo VPC cực chuẩn trong thư mục `modules/vpc`. Để sử dụng lại nó, bạn chỉ cần gọi nó trong dự án chính.

**Ví dụ tại Root Module (`main.tf`):**
```hcl
module "network_prod" {
  source = "./modules/vpc" # Đường dẫn tới linh kiện

  # Truyền tham số (Props) vào
  vpc_cidr = "10.0.0.0/16"
  env      = "production"
}
```

---

## 4. Best Practices: Quy tắc vàng cho code "sạch"

Để bản "hợp đồng hạ tầng" của bạn chuyên nghiệp, hãy nhớ 3 lệnh "thần thánh":

1.  **`terraform fmt` (Làm đẹp):** Tự động căn chỉnh dấu `=`, thụt đầu dòng theo chuẩn. Đừng để code lộn xộn khi commit!
2.  **`terraform validate` (Kiểm tra):** Check lỗi cú pháp và logic (thiếu biến, sai kiểu dữ liệu) mà không cần tạo tài nguyên thật.
3.  **Naming Convention:** Sử dụng `_` (snake_case). Đặt tên tài nguyên ngắn gọn nhưng mang tính gợi nhớ (ví dụ: `aws_instance.web_app`).

---

## Lời kết: Tạm gác lý thuyết - Bước vào Lab thực chiến!
Chúng ta đã cùng nhau đi qua 8 bài học với một lượng kiến thức nền tảng vững chắc. Nhưng Terraform không phải là môn học để "đọc", nó là môn học để "làm". Lý thuyết dù hay đến đâu cũng sẽ trôi tuột nếu không được chuyển hóa thành những dòng lệnh thực tế.

Bạn đã sẵn sàng để thấy sức mạnh của Terraform khi chỉ cần một lệnh duy nhất, toàn bộ hệ thống từ Key truy cập, Firewall cho đến Web Server tự động mọc lên hoàn chỉnh chưa?

Đã đến lúc tạm dừng những trang lý thuyết khô khan. Hãy mở sẵn Terminal và chuẩn bị tài khoản Cloud của bạn. Ở bài viết ngay tiếp theo, chúng ta sẽ bước vào **[Lab #1]: Triển khai Web Server hoàn chỉnh với SSH Key tự động.**

