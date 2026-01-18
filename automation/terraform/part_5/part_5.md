# [Terraform 105] Logic & Expressions: Biến HCL thành "ngôn ngữ" thực thụ

## Lời mở đầu: Thổi bùng sức sống cho đống dữ liệu

Ở bài trước, chúng ta đã nắm trong tay các "nguyên liệu" (Kiểu dữ liệu) và biết cách tạo ra các "đơn đặt hàng" linh hoạt (Variables). Tuy nhiên, nếu chỉ dừng lại ở đó, Terraform vẫn chỉ là một file cấu hình tĩnh.

Hạ tầng thực tế luôn đầy rẫy những bài toán "Nếu... thì...". Bài học hôm nay về **Logic & Expressions** chính là "bộ não" giúp bạn xử lý những logic đó, biến HCL từ một file khai báo khô khan thành một ngôn ngữ có khả năng tư duy và biến hóa theo từng điều kiện môi trường.

---

## 1. Toán tử (Operators): Những phép tính cơ bản

HCL cung cấp bộ toán tử rất giống với các ngôn ngữ lập trình hiện đại để bạn tính toán và so sánh.

*   **Toán tử số học:** `+`, `-`, `*`, `/`. (Ví dụ: `disk_size = var.base_size * 2`).
*   **Toán tử so sánh:** `==`, `!=`, `>`, `<`, `>=`, `<=`. (Dùng để kiểm tra điều kiện môi trường).
*   **Toán tử logic:** `&&` (AND), `||` (OR), `!` (NOT).

---

## 2. Conditional Expressions: Cú pháp "Hỏi - Đáp" (Ternary)

Đây là cách phổ biến nhất để Terraform đưa ra quyết định. Nó hoạt động y hệt toán tử 3 ngôi trong JavaScript.

**Cú pháp:** `condition ? true_val : false_val`

**Góc nhìn Dev (JS):**
```javascript
const storage = isProduction ? 100 : 20;
```

**Góc nhìn System:** "Nếu máy này là Database, hãy gắn thêm ổ cứng SSD, nếu không thì dùng HDD".

**Ví dụ HCL:**
```hcl
# Nếu env là prod thì tạo 3 instance, ngược lại chỉ tạo 1
count = var.env == "prod" ? 3 : 1
```

---

## 3. Hàm tích hợp (Built-in Functions): Bộ "đồ nghề" có sẵn

Terraform không cho bạn tự viết hàm, nhưng nó cung cấp sẵn một "siêu thị" các hàm để bạn gọt giũa dữ liệu.

*   **Xử lý chuỗi:** `upper()`, `lower()`, `join()`.
    *   *Góc nhìn System:* Giống như dùng `tr`, `sed` để chuyển đổi chữ hoa/thường.
*   **Xử lý Collection:** `length()`, `lookup()`.
    *   *Góc nhìn Dev (JS):* Giống như `array.length` hoặc `Map.get()`.

**Ví dụ thực tế:**
```hcl
# Gộp các tag và viết hoa tên dự án
project_tag = upper(join("-", ["project", var.project_name]))
# Kết quả: "PROJECT-BLOGAI"
```

---

## 4. Sức mạnh của For Expressions: "Phù thủy" biến đổi dữ liệu

Đây là tính năng giúp bạn xử lý hàng loạt dữ liệu (List, Map) cực kỳ mạnh mẽ mà không cần viết lặp lại code.

### Ví dụ 1: Biến đổi dữ liệu (Mapping)
Giả sử bạn có danh sách tên dịch vụ và muốn viết hoa toàn bộ chúng để làm Tag.

*   **Góc nhìn Dev (JS):** `apps.map(app => app.toUpperCase())`
*   **Góc nhìn System (Bash):** `for app in "${apps[@]}"; do echo "${app^^}"; done`

**Cú pháp HCL:**
```hcl
# Input: ["nginx", "web"] -> Output: ["NGINX", "WEB"]
upper_services = [for s in var.services : upper(s)]
```

### Ví dụ 2: Lọc dữ liệu (Filtering)
Bạn chỉ muốn lấy những IP nào thuộc dải mạng nội bộ (10.x.x.x).

*   **Góc nhìn Dev (JS):** `ips.filter(ip => ip.startsWith("10."))`
*   **Góc nhìn System (Bash):** `for ip in "${ips[@]}"; do [[ $ip == 10.* ]] && echo $ip; done`

**Cú pháp HCL:**
```hcl
# Chỉ lấy các IP bắt đầu bằng "10."
private_ips = [for ip in var.all_ips : ip if startswith(ip, "10.")]
```

### Ví dụ 3: Xử lý Map (Key-Value)
Biến một bảng tra cứu IP thành danh sách các câu lệnh SSH.

```hcl
# Input: { "web" = "1.1.1.1", "db" = "2.2.2.2" }
ssh_list = [for name, ip in var.server_map : "ssh admin@${ip} # ${name}"]
```

### Minh họa quy trình xử lý (Logic Funnel)
![HCL Logic Funnel](./hcl_logic_funnel.svg)

---

## 5. Bảng đối chiếu nhanh "Não trái - Não phải"

| Tính năng | Góc nhìn Dev (JS) | Góc nhìn System Admin | Vai trò trong HCL |
| :--- | :--- | :--- | :--- |
| **Conditional** | `a ? b : c` | `if [ $env == "prod" ]; ...` | Ra quyết định tại chỗ. |
| **Functions** | `array.length` | `wc -l`, `sed`, `awk` | Gọt giũa dữ liệu thô. |
| **For Expression** | `.map()` / `.filter()` | `for i in $list; do...done` | Tái cấu trúc dữ liệu hàng loạt. |

---

## Lời kết & Gợi mở

Khi đã nắm được Logic, bạn không còn là người "khai báo" đơn thuần nữa, mà đã trở thành một Programmer thực thụ trong giới hạ tầng. Code của bạn giờ đây đã thông minh hơn, biết tự ứng biến theo môi trường.

Nhưng có một vấn đề: Làm sao để lấy dữ liệu từ những thứ đã tồn tại sẵn trên Cloud (như một cái KeyPair bạn tạo bằng tay, hay một con VPC của dự án khác)? Làm sao để "soi" thông tin đó về file HCL của mình?

Hẹn gặp lại các bạn ở bài viết tiếp theo: **[Terraform 106] Data Sources - Cửa sổ nhìn ra thế giới Cloud.**

