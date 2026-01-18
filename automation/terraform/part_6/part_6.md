# [Terraform 106] Meta-Arguments & Dynamic Blocks: Nghệ thuật nhân bản hạ tầng

--- 

## Mục lục
- [Lời mở đầu: Đừng làm thợ xây, hãy làm kiến trúc sư](#lời-mở-đầu-đừng-làm-thợ-xây-hãy-làm-kiến-trúc-sư)
- [1. Meta-Arguments là gì?](#1-meta-arguments-là-gì)
- [2. Meta-arguments "Nhân bản": Count vs For_each](#2-meta-arguments-nhân-bản-count-vs-for_each)
  - [A. count: Nhân bản theo số lượng](#a-count-nhân-bản-theo-số-lượng)
  - [B. for_each: Nhân bản theo danh sách (Khuyên dùng)](#b-for_each-nhân-bản-theo-danh-sách-khuyên-dùng)
- [3. Kiểm soát "Sinh - Tử": depends_on & lifecycle](#3-kiểm-soát-sinh---tử-depends_on--lifecycle)
- [4. Dynamic Blocks: "Co giãn" cấu hình con](#4-dynamic-blocks-co-giãn-cấu-hình-con)
  - [Minh họa quy trình nhân bản (The Replication Factory)](#minh-họa-quy-trình-nhân-bản-the-replication-factory)
- [Bảng đối chiếu nhanh cho anh em](#bảng-đối-chiếu-nhanh-cho-anh-em)
- [Lời kết](#lời-kết)

--- 

## Lời mở đầu: Đừng làm thợ xây, hãy làm kiến trúc sư

Bạn đã bao giờ rơi vào cảnh phải Copy-Paste một đoạn code tạo Server ra 10 lần chỉ vì khách hàng muốn có 10 con Server giống hệt nhau chưa? Nếu có, thì bài học hôm nay là dành cho bạn.

Chúng ta sẽ học cách sử dụng **Meta-Arguments** để "sai bảo" Terraform tự nhân bản tài nguyên và **Dynamic Blocks** để xử lý những cấu hình con "lắt léo". Mục tiêu cuối cùng: Viết ít nhất, nhưng tạo ra nhiều nhất.

---

## 1. Meta-Arguments là gì?

Meta-Arguments là các cấu hình đặc biệt được Terraform cung cấp để thay đổi hành vi của Resource hoặc Module.

*   **Điểm khác biệt:** Nếu như Argument thông thường (như `ami`, `instance_type`) chỉ dùng cho một loại tài nguyên nhất định, thì Meta-Arguments có thể áp dụng cho hầu hết mọi tài nguyên.
*   **Vai trò:** Chúng không định nghĩa "thông số" của tài nguyên, mà định nghĩa "cách thức" Terraform quản lý, nhân bản hoặc sắp xếp thứ tự triển khai tài nguyên đó.

---

## 2. Meta-arguments "Nhân bản": Count vs For_each

### A. count: Nhân bản theo số lượng
Dùng khi bạn muốn tạo ra nhiều bản sao y hệt nhau và chỉ khác nhau ở con số thứ tự.
*   **Góc nhìn Dev (JS):** Giống như vòng lặp `for (let i = 0; i < 3; i++)`.

**Cú pháp HCL:**
```hcl
resource "aws_instance" "server" {
  count         = 3 
  ami           = "ami-xxxxxx"
  instance_type = "t2.micro"
  tags = {
    Name = "Server-${count.index}" # index chạy từ 0, 1, 2
  }
}
```

### B. for_each: Nhân bản theo danh sách (Khuyên dùng)
Xịn hơn `count`, nó dùng một Map hoặc Set để tạo tài nguyên. Mỗi tài nguyên sẽ có một "định danh" riêng biệt ngay từ đầu.
*   **Góc nhìn Dev (JS):** Giống như `Array.forEach()` hoặc `Array.map()`.
*   **Góc nhìn System:** "Tạo server riêng cho từng phòng ban dựa trên danh sách".

**Cú pháp HCL:**
```hcl
resource "aws_instance" "office_server" {
  for_each = toset(["ke-toan", "nhan-su", "ky-thuat"])
  
  ami           = "ami-xxxxxx"
  instance_type = "t2.micro"
  tags = {
    Name = "Server-${each.key}" # 'each.key' sẽ là tên phòng ban
  }
}
```

---

## 3. Kiểm soát "Sinh - Tử": depends_on & lifecycle

Hạ tầng không chỉ là tạo ra, mà còn là quản lý thứ tự và cách nó biến đổi.

*   **depends_on (Thứ tự ưu tiên):** Dùng khi bạn muốn chỉ định rõ: *"Phải tạo xong Database rồi mới được tạo App Server"*.
*   **lifecycle (Vòng đời):** Đây là "tấm khiên" bảo vệ tài nguyên khỏi các tác động ngoài ý muốn:
    *   `prevent_destroy = true`: Cấm xóa tài nguyên (Dùng cho Database quan trọng).
    *   `create_before_destroy = true`: Tạo bản mới thay thế thành công rồi mới xóa bản cũ (**Zero Downtime**).
    *   `ignore_changes`: Bỏ qua các thay đổi trên attributes cụ thể.
        *   *Góc nhìn System:* Giống như việc bạn cấu hình file nhưng không muốn script tự động cập nhật lại các dòng cụ thể khi chạy lại.
        *   *Ví dụ:* Bạn gán Tag thủ công trên Console và không muốn Terraform xóa mất nó mỗi khi apply.

---

## 4. Dynamic Blocks: "Co giãn" cấu hình con

Đây là kỹ thuật nâng cao để xử lý các block lặp lại bên trong một Resource (như các rule của Firewall).

*   **Góc nhìn Dev (JS):** Giống như render một list các sub-components bằng vòng lặp trong React/Vue.
*   **Góc nhìn System:** "Mở danh sách các Port từ một file cấu hình duy nhất".

**Ví dụ thực tế:**
```hcl
variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

resource "aws_security_group" "web_sg" {
  name = "web-firewall"

  # Tự động tạo nhiều block ingress dựa trên mảng allowed_ports
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Minh họa quy trình nhân bản (The Replication Factory)
![Terraform Replication Factory](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/part_6/terraform_replication_factory.svg)

---

## Bảng đối chiếu nhanh cho anh em

| Tính năng | Khi nào dùng? | Liên tưởng nhanh |
| :--- | :--- | :--- |
| **count** | Tạo n đối tượng giống hệt nhau. | `for (i=0; i<n; i++)` |
| **for_each** | Tạo đối tượng dựa trên danh sách tên. | `list.forEach()` |
| **ignore_changes** | Giữ nguyên các config đã sửa bằng tay. | `exclude` trong rsync/git. |
| **dynamic** | Số lượng block con bên trong thay đổi. | Nested Loop / `.map()` UI. |

---

## Lời kết

Đến đây, bạn đã nắm trong tay những kỹ thuật "thượng thừa" để nhân bản và bảo vệ hạ tầng. Bạn không còn phải Copy-Paste thủ công, code của bạn đã cực kỳ tinh gọn và chuyên nghiệp.

Nhưng nãy giờ chúng ta toàn nói về việc Tạo mới. Vậy nếu hạ tầng đã có sẵn từ trước (ví dụ do đồng nghiệp tạo bằng tay) thì sao? Làm sao để Terraform "soi" thấy chúng?

Hẹn gặp lại các bạn ở bài viết cuối cùng của series: **[Terraform 107] Data Sources - Cửa sổ nhìn ra thế giới Cloud thực tế.**

