
# [Terraform 101] Tư duy Hệ thống & Mô hình 'Nhà máy' trong Infrastructure as Code

---

## Mục lục
- [1. Mở đầu: Từ "ClickOps" đến Tự động hóa](#1-mở-đầu-từ-clickops-đến-tự-động-hóa)
- [2. Tư duy cốt lõi: Imperative vs. Declarative](#2-tư-duy-cốt-lõi-imperative-vs-declarative)
- [3. Mô hình "Công xưởng chế tác" (The Factory Model)](#3-mô-hình-công-xưởng-chế-tác-the-factory-model)
  - [A. Kho bãi & Công cụ (The Setup)](#a-kho-bãi--công-cụ-the-setup)
  - [B. Nguyên liệu đầu vào (The Inputs)](#b-nguyên-liệu-đầu-vào-the-inputs)
  - [C. Dây chuyền sản xuất (The Process - Resource)](#c-dây-chuyền-sản-xuất-the-process---resource)
  - [D. Đầu ra (The Outputs)](#d-đầu-ra-the-outputs)
- [4. "Trái tim" của Nhà máy: State File (Sổ Kho)](#4-trái-tim-của-nhà-máy-state-file-sổ-kho)
  - [Tại sao cần "Sổ kho"?](#tại-sao-cần-sổ-kho)
  - [State File lưu cái gì?](#state-file-lưu-cái-gì)
- [5. Minh họa luồng dữ liệu](#5-minh-họa-luồng-dữ-liệu)
- [6. Thực hành: Code hoàn chỉnh (main.tf)](#6-thực-hành-code-hoàn-chỉnh-maintf)
- [7. Vòng đời hoạt động (The Workflow)](#7-vòng-đời-hoạt-động-the-workflow)
- [8. Khi nào nên (và không nên) dùng Terraform?](#8-khi-nào-nên-và-không-nên-dùng-terraform)
  - [✅ NÊN dùng khi:](#-nên-dùng-khi)
  - [❌ KHÔNG NÊN dùng khi:](#-không-nên-dùng-khi)
- [9. Lời kết: Đừng vội mừng...](#9-lời-kết-đừng-vội-mừng)

---

## 1. Mở đầu: Từ "ClickOps" đến Tự động hóa

Nếu bạn từng phải tạo thủ công 10 máy ảo (VM) trên AWS Console, bạn sẽ hiểu "nỗi đau" của việc click chuột mỏi tay và dễ sai sót. Tệ hơn nữa, nếu muốn xóa chúng đi, bạn lại phải đi tìm từng cái để xóa. Một ngày đẹp trời, sếp hỏi: *"Cấu hình con server này ai sửa, sửa khi nào?"*, bạn hoàn toàn không có câu trả lời.

**Vấn đề:** Làm sao để tạo, sửa đổi và quản lý hạ tầng một cách tự động, chính xác và có thể lưu lại lịch sử thay đổi (version control) giống như Code?

**Giải pháp: Terraform.**

- Terraform là công cụ **Infrastructure as Code (IaC)** phổ biến nhất hiện nay. Bạn viết cấu hình, Terraform sẽ "xây" hạ tầng. Nhưng Terraform không chỉ là một script chạy lệnh, nó là một hệ thống quản lý trạng thái thông minh.

- Terraform được tạo ra bởi **Mitchell Hashimoto** (đồng sáng lập HashiCorp) và phát hành phiên bản đầu tiên vào tháng 7 năm 2014. Trong bối cảnh mỗi nhà cung cấp Cloud đều có công cụ riêng (AWS có CloudFormation, Azure có ARM Templates), Terraform ra đời với sứ mệnh **"Cloud Agnostic"** - một ngôn ngữ chung nhất quán để quản trị hạ tầng trên bất kỳ nền tảng nào (AWS, Azure, GCP, hay thậm chí VMWare on-premise).

---

## 2. Tư duy cốt lõi: Imperative vs. Declarative

Để làm chủ Terraform, bạn cần thay đổi tư duy từ **Mệnh lệnh (Imperative)** sang **Khai báo (Declarative)**.

*   **Imperative (Ví dụ: Bash script):** Bạn chỉ định **cách làm từng bước** (Bước 1: Tạo VM -> Bước 2: Cài OS -> Bước 3: Mở Port).
*   **Declarative (Ví dụ: Terraform, NixOS):** Bạn chỉ định **kết quả mong muốn (Desired State)**. Bạn bảo: *"Tôi muốn một VM chạy Ubuntu mở port 80"*. Terraform tự tính toán các bước cần thiết để đạt được điều đó.

> **Liên hệ với NixOS:** Nếu bạn đã dùng NixOS, tư duy này hoàn toàn tương đồng. File config mô tả trạng thái cuối cùng của hệ điều hành. Khi chạy lệnh apply, hệ thống tự biến đổi để khớp với config đó.

Hãy hình dung Terraform hoạt động như một cỗ máy xử lý dữ liệu với luồng đi rõ ràng:
**Input $\rightarrow$ Process $\rightarrow$ Output**

---

## 3. Mô hình "Công xưởng chế tác" (The Factory Model)

Để dễ hình dung và không bị lạc trong code, hãy coi file code Terraform (`main.tf`) như quy trình vận hành của một nhà máy sản xuất:

### A. Kho bãi & Công cụ (The Setup)
Trước khi làm việc, thợ cần xác định nguồn cung ứng và chuẩn bị đồ nghề.

**Khối `terraform` (Đơn đặt hàng nhà cung cấp):**
Khai báo rõ ràng là mình sẽ dùng công cụ của hãng nào, phiên bản bao nhiêu.
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```
*Lệnh `terraform init` chính là hành động mở cửa kho và tải các tool này về.*

**`provider` (Cấu hình công cụ):**
Lôi đồ nghề ra cấu hình (VD: chọn region AWS là `us-east-1`).

### B. Nguyên liệu đầu vào (The Inputs)
Để sản xuất, ta cần nguyên liệu từ 3 nguồn:

1.  **`variable` (Yêu cầu từ khách hàng):** Nguyên liệu thô người dùng đưa vào. (VD: "Server Size L").
2.  **`data` (Hàng nhập khẩu):** Nguyên liệu có sẵn trên thị trường (Cloud) lấy về dùng. (VD: ID bản Ubuntu mới nhất từ AWS).
3.  **`local` (Sơ chế tại chỗ):** Nguyên liệu tự pha trộn nội bộ. (VD: Ghép tên dự án làm Tag).

### C. Dây chuyền sản xuất (The Process - Resource)
Đây là phần quan trọng nhất.
**`resource` (Bản vẽ kỹ thuật):** Tổng hợp nguyên liệu (`var`, `data`, `local`) và dùng công cụ (`provider`) để tạo ra tài nguyên thực tế.

### D. Đầu ra (The Outputs)
1.  **Hạ tầng thực tế (Real Infrastructure):** VM, DB đã chạy trên Cloud.
2.  **`output` (Tem nhãn):** Thông số quan trọng (IP, DNS) in ra màn hình.

---

## 4. "Trái tim" của Nhà máy: State File (Sổ Kho)

Nếu `main.tf` là **Bản vẽ thiết kế**, thì `terraform.tfstate` chính là cuốn **Sổ kho (Ledger/Inventory)** của người quản lý nhà máy.

### Tại sao cần "Sổ kho"?
Hãy tưởng tượng bạn ra lệnh: *"Xây thêm 1 cái server"*.
*   Nếu không có sổ kho: Nhà máy sẽ xây mới hoàn toàn (thành 2 cái server).
*   Nếu có sổ kho: Quản lý sẽ tra sổ, thấy đã có 1 cái rồi, và kiểm tra xem cái đó có đúng như bản vẽ không. Nếu đúng, ông ấy báo "Không cần làm gì cả". Nếu sai, ông ấy sửa lại.

### State File lưu cái gì?
Nó là một file JSON (thường nằm ẩn trong máy bạn hoặc trên Cloud), ánh xạ giữa **Tài nguyên trong Code** và **Tài nguyên thực tế**.

| Code (Bản vẽ) | State File (Sổ kho) | Thực tế (Cloud) |
| :--- | :--- | :--- |
| `resource "aws_instace" "web"` | Ánh xạ: `aws_instance.web` = `i-01234abc` | Máy ảo có ID `i-01234abc` đang chạy |

> **Lưu ý sống còn:** KHÔNG BAO GIỜ được sửa tay file `terraform.tfstate`. Hãy để Terraform tự quản lý nó. Mất file này giống như nhà máy bị cháy sổ sách, việc khôi phục rất đau đầu.

---

## 5. Minh họa luồng dữ liệu

Biểu đồ dưới đây tóm tắt cách luồng dữ liệu di chuyển từ đầu vào, qua xử lý, được ghi chép vào State (Sổ kho) và tạo ra kết quả.

![Terraform Architecture](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/part_1/terraform-code-structure.svg)

---

## 6. Thực hành: Code hoàn chỉnh (main.tf)

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "us-east-1" }

variable "instance_size" { default = "t2.micro" }

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web_server" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_size
  tags = {
    Name = "Terraform-Lab-WebServer"
  }
}

output "server_public_ip" { value = aws_instance.web_server.public_ip }
```


![terraform code structure](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/part_1/terraform-code-structure.svg)

---

## 7. Vòng đời hoạt động (The Workflow)

Quy trình 4 bước thần thánh vận hành nhà máy:

1.  **`terraform init` (Nhập kho):** Tải plugin, chuẩn bị môi trường.
2.  **`terraform plan` (Kiểm tra & So sánh):**
    *   Giám đốc cầm bản vẽ (`main.tf`) so với Sổ kho (`tfstate`) và Thực tế.
    *   In ra kế hoạch: *"Sẽ xây mới 1 cái, sửa 1 cái"*. Chưa làm thật.
3.  **`terraform apply` (Sản xuất):**
    *   Duyệt kế hoạch -> Thi công thật trên Cloud.
    *   **Quan trọng:** Cập nhật ngay vào Sổ kho (`tfstate`) sau khi làm xong.
4.  **`terraform destroy` (Thanh lý):**
    *   Dựa vào Sổ kho để phá dỡ đúng những gì mình đã xây.

---

## 8. Khi nào nên (và không nên) dùng Terraform?

Không phải lúc nào cũng vác "dao mổ trâu" Terraform ra dùng.

### ✅ NÊN dùng khi:
*   **Quản lý hạ tầng vòng đời dài (Long-lived Infra):** VPC, Load Balancer, Database, cụm Kubernetes (EKS/GKE).
*   **Làm việc nhóm (Team Collaboration):** Cần Code Review, cần lịch sử thay đổi để biết ai đã làm gì.
*   **Multi-Cloud:** Cần quản lý tài nguyên trên cả AWS, Azure và Cloudflare cùng một lúc.
*   **Môi trường phức tạp:** Cần dựng nhiều môi trường giống hệt nhau (Dev, Staging, Prod) chỉ bằng cách thay đổi biến số.

### ❌ KHÔNG NÊN dùng khi:
*   **Cấu hình bên trong Server (Config Management):** Cài cắm phần mềm, sửa file config trong OS... Terraform làm được (dùng remote-exec) nhưng rất tệ. Hãy dùng **Ansible** cho việc này.
*   **Hạ tầng "ăn liền" (Ad-hoc):** Cần test nhanh 1 cái máy ảo dùng 5 phút rồi bỏ -> Click chuột cho lẹ.
*   **Ứng dụng trên Kubernetes:** Deploy Pod, Service, Deployment... Terraform làm được nhưng **Helm** hoặc **ArgoCD** chuyên dụng và linh hoạt hơn nhiều.

---

## 9. Lời kết: Đừng vội mừng...

Chúc mừng bạn! Nếu nắm vững mô hình "Nhà máy" và "Sổ kho", bạn đã hiểu được *cách Terraform hoạt động*. Bạn chạy code mẫu ở trên và thấy nó tạo ra server vèo vèo. Thật kỳ diệu!

**Nhưng... hãy thử nhìn vào một file cấu hình thực tế của các dự án lớn.**

Bạn sẽ thấy những ký tự lạ lẫm như `${var...}`, `for_each`, `dynamic`, `module "..."` hay những hàm tính toán `cidrsubnet(...)` loằng ngoằng.
Lúc này, cảm giác hào hứng ban đầu sẽ thay thế bằng sự hoang mang:
*   *"Tại sao chỗ này lại dùng ngoặc nhọn, chỗ kia lại dùng ngoặc vuông?"*
*   *"Làm sao tôi có thể tự viết ra những dòng code này theo ý mình thay vì chỉ copy-paste?"*
*   *"Làm sao để biến những ý tưởng trong đầu tôi thành những dòng cấu hình chính xác mà Terraform hiểu được?*"

Nếu bạn đang cảm thấy như vậy, đừng lo. Đó là biểu hiện của việc bạn đang thiếu mảnh ghép quan trọng nhất: **Ngôn ngữ**.

Để giao tiếp trôi chảy với "người thợ" Terraform và bắt nó làm theo mọi ý muốn của mình, bạn cần thông thạo ngôn ngữ mẹ đẻ của nó: **HCL (HashiCorp Configuration Language)**.

Làm sao để giải mã những syntax khó hiểu kia? Làm sao để viết code mượt mà như viết văn?

Tất cả sẽ được giải mã trong bài viết tiếp theo: **[Terraform 102] Làm chủ ngôn ngữ HCL - Giải mã bí ẩn đằng sau những dòng code**.

