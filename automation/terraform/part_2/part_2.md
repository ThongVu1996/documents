# [Terraform 102] Làm chủ ngôn ngữ HCL - Giải mã bí ẩn đằng sau những dòng code

## Lời mở đầu: HCL - "Bản hợp đồng" giữa bạn và Cloud

Ở bài trước, chúng ta đã biết muốn "sai bảo" Terraform thì phải thạo HCL. Đừng coi nó là code khô khan, hãy coi HCL là **một bản hợp đồng quyền lực**: Bạn chỉ cần ghi rõ "điều khoản" mình muốn, Terraform sẽ tự đi "đòi" kết quả từ Cloud Provider cho bạn.

HCL sinh ra để kết thúc kỷ nguyên của những file YAML dài dằng dặc hay những script Bash "chạy bằng niềm tin" đầy rủi ro. Với HCL, việc quản lý hạ tầng sẽ trở nên mượt mà, logic và an tâm hơn bao giờ hết. Hãy cùng tôi giải mã nó ngay sau đây!

---

## 1. HCL là gì? Đừng gọi nó là ngôn ngữ lập trình, tội nó!

**HCL (HashiCorp Configuration Language)** không dùng để viết app như PHP hay JS, nó dùng để **định nghĩa hạ tầng**.

*   **Với Dev:** Nó dễ đọc như JSON nhưng không có đống ngoặc nhọn `{}` gây rối mắt.
*   **Với SysAdmin/DevOps:** Nó ổn định hơn Bash. Bạn không phải viết hàng tá câu lệnh `if-else` chỉ để kiểm tra xem một con server đã tồn tại hay chưa.

---

## 2. Tư duy "Khai báo" (Declarative) – Đặt hàng thay vì tự đi nấu

Đây là triết lý giúp anh em mình "nhàn thân":

### Cách cũ (Mệnh lệnh - Imperative)
Giống như bạn dặn: *"Đi ra chợ mua thịt, về bật bếp, chờ dầu nóng, rán lên, cho ra đĩa"*.
> ⚠️ **Rủi ro:** Sai một bước là hỏng cả bữa ăn. Nếu chạy lại lần 2 mà không kiểm tra, bạn có thể mua thừa thịt.

### Cách HCL (Khai báo - Declarative)
Bạn chỉ cần nói: **"Tôi muốn có một đĩa thịt rán trên bàn"**.
> ✅ **An tâm:** Việc đi chợ hay nấu nướng là việc của Terraform. Nếu đĩa thịt đã có sẵn trên bàn, nó sẽ không làm gì thêm.

![Imperative vs Declarative](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/part_2/thumbnail.svg)

---

## 3. Tại sao không xài "cơm nguội" JSON hay YAML?

Để thấy rõ vì sao HCL là "chân ái", hãy nhìn vào sự khác biệt khi chúng ta muốn tạo một máy ảo (Instance) đơn giản:

### Cách cũ: Dùng Bash Script (Mệnh lệnh)
Bạn phải tự ra lệnh từng bước. Chạy lại lần 2 dễ lỗi hoặc bị trùng lặp tài nguyên.

```bash
# Phải check xem instance đã tồn tại chưa, cấu hình network...
aws ec2 run-instances --image-id ami-xxxxxx --count 1 --instance-type t2.micro
```

### Cách mới: Dùng HCL (Khai báo)
Bạn chỉ mô tả "đích đến". Terraform tự hiểu: chưa có thì tạo, có rồi thì giữ nguyên.

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-xxxxxx" # ID của Image hệ điều hành
  instance_type = "t2.micro"   # Cấu hình phần cứng
}
```

### Bảng so sánh nhanh

| Tiêu chí | JSON | YAML | HCL |
| :--- | :--- | :--- | :--- |
| **Dễ đọc** | Kém (nhiều `{}`) | Tốt (nhưng dễ sai thụt lề) | **Rất tốt** (mạch lạc) |
| **Logic** | Không có | Rất hạn chế | **Mạnh mẽ** (Biến, Hàm, Loop) |
| **Comment** | Không hỗ trợ | Có | **Có** (`#` hoặc `//`) |

---

## 4. Hệ sinh thái HashiCorp – Học một lần, "vẩy tay" mọi mặt trận

Thông thạo HCL là bạn đã nắm trong tay "chìa khóa" của bộ công cụ DevOps xịn nhất hiện nay:

*   **Terraform:** Xây nhà (Provisioning hạ tầng AWS, Azure, GCP).
*   **Vault:** Giữ két sắt (Quản lý password, key bí mật).
*   **Packer:** Đúc khuôn (Tạo các bản image server chuẩn).

---

## 5. Setup nhanh để "vọc" (Cho anh em dùng Mac/Nix/Neovim)

Vì chúng ta là dân chuyên nghiệp, hãy setup môi trường thật chuẩn:

*   **Nix-shell:** Cài Terraform CLI chỉ với một câu lệnh.
*   **Neovim:** Cài LSP cho HCL để có tính năng tự động gợi ý và format code (`terraform fmt`). Code hạ tầng mà mượt như code Web!

> **💡 Mẹo cho người mới:** Nếu bạn dùng **VS Code**, hãy cài ngay extension chính chủ **HashiCorp Terraform** để được hỗ trợ highlight và nhắc lệnh tận răng nhé.

---

## Lời kết & Gợi mở bài sau

Hiểu được "tại sao lại là HCL" là bạn đã đi được nửa chặng đường làm chủ Terraform. Nhưng một bản hợp đồng thực tế sẽ không chỉ đơn giản như ví dụ trên. Nó cần những tham số linh hoạt, những giá trị đầu ra và khả năng tái sử dụng.

*   Làm sao để biến những dòng code cứng nhắc thành một hệ thống linh động?
*   Làm sao để truyền dữ liệu qua lại giữa các thành phần hạ tầng?

Hẹn gặp lại các bạn ở bài viết tiếp theo: **[Terraform 103] Variables & Outputs – Biến hóa hạ tầng theo cách của bạn.**

