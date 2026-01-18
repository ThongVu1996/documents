# [Lab #2] Terraform Meta-Arguments Masterclass - Kịch bản Multi-Region & Scaling hạ tầng

---

## 1. Giới thiệu mục tiêu bài Lab

Chào mừng anh em quay trở lại! Nếu ở Lab #1 chúng ta mới chỉ học cách "xây một cái nhà", thì ở Lab #2 này, chúng ta sẽ học cách "quy hoạch cả một khu đô thị".

**Thành quả đạt được sau bài Lab:**

1.  **Multi-Region:** Điều khiển cùng lúc 2 vùng địa lý (Singapore & Virginia) trong một cấu hình duy nhất.
2.  **Scaling:** Nhân bản hàng loạt EC2 Instance chỉ bằng một biến số.
3.  **Identity Management:** Tự động tạo hàng loạt tài khoản IAM User từ một danh sách (Map).
4.  **Zero Downtime:** Cấu hình vòng đời tài nguyên để cập nhật hạ tầng không gây gián đoạn.
5.  **Fix lỗi Duplicate:** Kỹ thuật sử dụng `random_id` để code có thể chạy lại nhiều lần mà không bị trùng tên tài nguyên trên Cloud.

### Minh họa: Kiến trúc Đa vùng & Scaling
![Terraform Lab 2 Architecture](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/terraform_lab_2_arch.svg)

---

## 2. Chuẩn bị (Prerequisites)

Ngoài các bước chuẩn bị ở bài Lab #1, anh em cần lưu ý thêm:

*   **Tăng quyền IAM:** Vì bài này có tạo IAM User, tài khoản bạn dùng để chạy Terraform cần quyền `IAMFullAccess`.
*   **Kiểm tra Quota:** Đảm bảo tài khoản AWS của bạn có thể tạo tài nguyên ở cả vùng `ap-southeast-1` (Singapore) và `us-east-1` (Virginia - Mỹ).

---

## 3. Nội dung File Lab (`main.tf`)

Anh em tạo một thư mục mới (ví dụ `terraform-lab-2`), tạo file `main.tf` và dán đoạn code "vũ khí hạng nặng" này vào. Hãy chú ý các phần chú thích `[META-ARGUMENT]` để hiểu cách vận hành:

```hcl
# [5] COMMENT:
# Bài tập: Terraform Meta-Arguments Masterclass (Fixed Duplicate Error)
# Kịch bản: Triển khai Multi-Region, Scaling VM, IAM User và Network an toàn.

# [1] BLOCK: Terraform configuration
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    # Fix Lỗi Duplicate: Cần provider random để tạo tên unique
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
    tls = {
      source  = "hashicorp/tls"
      version = "~> 4.0"
    }
  }
}

# --- PROVIDERS (META-ARGUMENT: ALIAS) ---

# Provider chính (Singapore)
provider "aws" {
  region = "ap-southeast-1"
}

# [META-ARGUMENT 1: Provider Alias]
# Provider phụ cho Disaster Recovery (Virginia)
provider "aws" {
  alias  = "dr_region"
  region = "us-east-1"
}

# --- VARIABLES ---

variable "project_config" {
  type = object({
    env            = string
    instance_size  = map(string)
    whitelist_ips  = list(string)
    instance_count = number
  })

  default = {
    env = "dev"
    instance_size = {
      dev  = "t3.micro"
      prod = "t3.medium"
    }
    whitelist_ips  = ["0.0.0.0/0"]
    instance_count = 2 # Scaling: Tạo 2 servers
  }
}

variable "sysadmin_users" {
  description = "Danh sách user quản trị (Map)"
  type        = map(string)
  default = {
    "alice" = "DevOps-Lead"
    "bob"   = "SysAdmin-Intern"
  }
}

# --- FIX: RANDOM ID GENERATOR ---

# Tạo chuỗi ngẫu nhiên 4 ký tự (ví dụ: a1b2) để gắn vào tên resource
resource "random_id" "lab_id" {
  byte_length = 4
  keepers = {
    # Thay đổi env sẽ tạo ra ID mới
    env = var.project_config.env
  }
}

# --- RESOURCES ---

# 1. SSH KEY & KEY PAIR
resource "tls_private_key" "ssh_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

# Key Pair Chính (Singapore)
resource "aws_key_pair" "generated_key" {
  # [FIX DUPLICATE] Tên Key = tony-lab-key-dev-a1b2
  key_name   = format("tony-lab-key-%s-%s", var.project_config.env, random_id.lab_id.hex)
  public_key = tls_private_key.ssh_key.public_key_openssh

  # [META-ARGUMENT 2: Lifecycle]
  lifecycle {
    ignore_changes = [tags] 
  }
}

# Key Pair Phụ (Virginia) - Dùng Provider Alias
resource "aws_key_pair" "generated_key_dr" {
  # [META-ARGUMENT 1: Provider]
  provider = aws.dr_region

  key_name   = format("tony-lab-key-dr-%s-%s", var.project_config.env, random_id.lab_id.hex)
  public_key = tls_private_key.ssh_key.public_key_openssh
}

resource "local_file" "private_key_pem" {
  content         = tls_private_key.ssh_key.private_key_pem
  filename        = "${path.module}/generated-key.pem"
  file_permission = "0400"
}

# 2. IAM USERS (Identity)
resource "aws_iam_user" "sysadmins" {
  # [META-ARGUMENT 3: For_each]
  for_each = var.sysadmin_users

  name = "user-${each.key}-${random_id.lab_id.hex}"
  tags = {
    Department = each.value
    Role       = "SystemOperator"
  }
}

# 3. SECURITY GROUP (Network)
resource "aws_security_group" "tony_web_sg" {
  name         = "tony-web-sg-${var.project_config.env}-${random_id.lab_id.hex}"
  description  = "Allow HTTP and SSH"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.project_config.whitelist_ips
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # [META-ARGUMENT 2: Lifecycle]
  lifecycle {
    create_before_destroy = true
  }
}

# 4. COMPUTE (EC2)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web_server" {
  # [META-ARGUMENT 4: Count]
  count = var.project_config.instance_count

  # [META-ARGUMENT 5: Depends_on]
  depends_on = [
    aws_iam_user.sysadmins,
    aws_key_pair.generated_key
  ]

  ami           = data.aws_ami.ubuntu.id
  instance_type = var.project_config.env == "prod" ? var.project_config.instance_size["prod"] : var.project_config.instance_size["dev"]

  vpc_security_group_ids = [aws_security_group.tony_web_sg.id]
  key_name               = aws_key_pair.generated_key.key_name

  user_data = <<-EOF
              #!/bin/bash
              sudo apt-get update
              sudo apt-get install -y nginx
              echo "<h1>Server #${count.index + 1} (${var.project_config.env})</h1>" > /var/www/html/index.html
              sudo systemctl enable nginx
              sudo systemctl start nginx
              EOF

  tags = {
    Name = "WebServer-${upper(var.project_config.env)}-${count.index}-${random_id.lab_id.hex}"
  }
}

# --- OUTPUTS ---

output "server_ips" {
  description = "Danh sách IP của các server (Splat Operator)"
  value       = aws_instance.web_server[*].public_ip
}

output "generated_key_name" {
  description = "Tên Key Pair đã được tạo (có random id)"
  value       = aws_key_pair.generated_key.key_name
}
```

---

## 4. Giải mã các Meta-Arguments "Hạng nặng"

Trong bài Lab này, anh em cần tập trung soi kỹ 5 "món bảo bối" sau:

1.  **Provider Alias:** Quan sát cách chúng ta dùng `provider = aws.dr_region` để đẩy Key Pair sang tận Virginia (Mỹ) mà không cần viết lại code.
2.  **Lifecycle:** `create_before_destroy = true` giúp cập nhật Security Group theo kiểu: "Xây cái mới xong mới phá cái cũ", đảm bảo không bị ngắt kết nối giữa chừng.
3.  **For_each:** Cách chúng ta duyệt qua Map `sysadmin_users` để tạo ra User Alice và Bob chỉ bằng 1 khối code duy nhất.
4.  **Count:** Biến số `instance_count = 2` ra lệnh cho Terraform nhân bản đúng 2 máy ảo song song.
5.  **Depends_on:** "Người điều phối" giao thông. EC2 phải đợi IAM User và Key tạo xong thì mới được khởi chạy.

---

## 5. Thao tác thực hành

Mở Terminal tại thư mục chứa file và thực hiện các lệnh sau:

### Bước 1: Khởi tạo
```bash
terraform init
```

![Terraform Init](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/init.png)

### Bước 2: Kiểm tra bản phác thảo
Hãy quan sát kỹ số lượng tài nguyên sẽ được tạo. Nhờ `count` và `for_each`, bạn sẽ thấy con số tài nguyên lên tới 7-8 objects chỉ với vài dòng code.
```bash
terraform plan
```

![Terraform Plan](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/plan.png)

### Bước 3: Triển khai đa vùng
Triển khai hạ tầng lên cả Singapore và Mỹ cùng lúc:
```bash
terraform apply -auto-approve
```

![Terraform Apply](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/apply.png)

### Bước 4: Kiểm chứng (Checklist)
*   **Singapore:** Vào EC2 Dashboard để thấy 2 Server đang chạy.
![Server in AWS](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/server.png)
*   **Virginia:** Đổi vùng trên Console sang N. Virginia để thấy Key Pair dự phòng đã được đẩy sang.
![Key pair in AWS](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/key_pair.png)
*   **IAM Dashboard:** Kiểm tra danh sách User để thấy `user-alice` và `user-bob` với các Tag tương ứng.
![User in AWS](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/user.png)

### Bước 5: Dọn dẹp
Đừng quên xóa bỏ mọi thứ sau khi kết thúc Lab để tránh phát sinh chi phí cho cả 2 vùng:
```bash
terraform destroy -auto-approve
```

![Terraform Destroy](https://github.com/ThongVu1996/documents/raw/main/automation/terraform/lab_2_meta_argument/destroy.png)

---

## 6. Tổng kết Lab #2

Chúc mừng anh em đã vượt qua bài Lab "khó nhằn" này! Chúng ta đã thực sự biến Terraform thành một cỗ máy vận hành hạ tầng chuyên nghiệp:

*   **Multi-Region:** Phá bỏ giới hạn địa lý, điều khiển cả Singapore và Mỹ cùng lúc.
*   **Scaling & Identity:** Quản lý hàng loạt Server và User chỉ bằng vài dòng Variable.
*   **Random ID:** Kỹ thuật "sinh tồn" giúp code của bạn không bao giờ bị đụng hàng trên Cloud.

---

## Lời kết: Nhiệm vụ mới - Kẻ truy vết hạ tầng (The Tracker)

Sau 2 bài Lab, chúng ta đã rất thuần thục trong việc "khai sơn phá thạch" — tức là tạo mới mọi thứ từ con số 0. Nhưng thực tế đi làm lại khắc nghiệt hơn nhiều. Sẽ có lúc sếp đưa cho bạn một hệ thống "cũ kỹ" được tạo bằng tay từ 5 năm trước, hoặc một team khác đã tạo sẵn Network và yêu cầu bạn: *"Hãy gắn Server của cậu vào đúng cái Security Group mà tôi đã tạo, tôi chỉ cho cậu cái Tag Name thôi, tự đi mà tìm ID!"*

Lúc này, bạn không thể dùng `resource` để tạo mới (vì nó đã có rồi), cũng không thể copy-paste ID bằng tay (vì nó quá thủ công và thiếu chuyên nghiệp). Bạn cần một "chiếc kính lúp" để soi vào đống hạ tầng khổng lồ của AWS và trích xuất đúng thông tin mình cần. Đó chính là lúc sức mạnh của **Data Sources** thực sự tỏa sáng.

Hãy chuẩn bị tinh thần cho một kịch bản "nhập vai" đầy kịch tính. Ở bài tiếp theo, chúng ta sẽ vào vai một Agent thực hiện nhiệm vụ: **Truy tìm ID của một "Căn cứ bí mật" (Security Group) thông qua các manh mối (Tags) để triển khai đặc vụ Agent-007 (EC2 Instance).**

Nhiệm vụ tiếp theo: **[Lab #3] Data Source Challenge - Truy vết và Kết nối hạ tầng có sẵn.**

*Manh mối đã có sẵn trên Cloud. Bạn đã sẵn sàng để "soi" chưa? Hẹn gặp lại anh em ở bài Lab tới!*

