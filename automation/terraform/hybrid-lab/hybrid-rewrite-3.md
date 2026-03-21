# Triển Khai Hạ Tầng Hybrid (AWS & Proxmox) với HCP Terraform

---

## 1. Yêu Cầu

### 1.1 Kịch Bản Hệ Thống

| Thành phần | Nền tảng | Mục tiêu |
| :--- | :--- | :--- |
| **Database (MySQL/PostgreSQL)** | On-Premise (Proxmox) | Tối ưu chi phí, kiểm soát và bảo mật dữ liệu ở mức cao nhất |
| **Web Server + ALB** | Public Cloud (AWS) | Đón lưu lượng người dùng, đảm bảo tính sẵn sàng cao |
| **State Management** | HCP Terraform (Terraform Cloud) | Quản lý State file, bảo mật biến số, phê duyệt trước khi cấp phát hạ tầng |

### 1.2 Nguyên Tắc Triển Khai

- **VCS Integration:** Tích hợp GitHub/GitLab vào HCP Terraform để tự động hóa quy trình — Engineer chỉ cần commit code, HCP tự động trigger plan.
- **Execution Mode:** Bắt buộc cài đặt `Execution Mode` ở trạng thái **Agent** để agent nội bộ có thể gọi API tới Proxmox mà không cần mở port inbound.
- **Bảo mật biến số:** Tuyệt đối không hardcode mật khẩu trong file code. Tất cả khóa API (AWS Access Key, Proxmox Token) phải được mã hóa dạng **Sensitive Variables** hoặc **Environment Variables** trên giao diện HCP Terraform Cloud.
- **Modularization:** Chia cấu trúc thư mục rõ ràng thành `modules/aws` và `modules/proxmox`.
- **Output:** Hệ thống phải xuất ra địa chỉ DNS của AWS ALB và địa chỉ IP nội bộ của VM chứa Database.

### 1.3 Phân Công Nhiệm Vụ (Team Roles)

Đây là bài lab **team-based** — mỗi role có trách nhiệm và quyền hạn riêng biệt, không được tự ý vượt qua ranh giới của mình.

```
┌─────────────────────────────────────────────────────────────────┐
│                      LUỒNG PHỐI HỢP ĐẦY ĐỦ                      │
│                                                                 │
│  👤 Workspace Admin (Team Leader)                               │
│     └─► Chuẩn bị toàn bộ hạ tầng nền (Proxmox, HCP, Agent)      │
│     └─► Tạo Org / Project / Workspace trên HCP Terraform        │
│     └─► Setup Sensitive Variables (Proxmox token, AWS key...)   │
│     └─► Kết nối HCP với VCS (GitHub/GitLab)                     │
│     └─► Tạo skeleton code, push lên GitHub                      │
│                          │                                      │
│              ┌───────────┴───────────┐                          │
│              ▼                       ▼                          │
│  👤 On-Premise Engineer      👤 AWS Engineer                    │
│     └─► Viết modules/proxmox    └─► Viết modules/aws            │
│     └─► git commit & push        └─► git commit & push          │
│              │                       │                          │
│              └───────────┬───────────┘                          │
│                          ▼                                      │
│              HCP tự động trigger Plan                           │
│                          │                                      │
│                          ▼                                      │
│  👤 DevOps Reviewer & Approver                                  │
│     └─► Đọc terraform plan trên HCP UI                          │
│     └─► Kiểm tra luồng kết nối AWS ↔ Proxmox                    │
│     └─► Bấm "Confirm & Apply" — người DUY NHẤT được apply       │
│                          │                                      │
│                          ▼                                      │
│        Hạ tầng được tạo trên cả AWS lẫn Proxmox                 │
└─────────────────────────────────────────────────────────────────┘
```

Chi tiết trách nhiệm và quyền hạn từng role:

| Role | Trách nhiệm chính | Quyền `apply`? |
| :--- | :--- | :--- |
| **👤 Workspace Admin** | Chuẩn bị hạ tầng nền (Proxmox, HCP, Agent, VCS integration), setup Sensitive Variables, tạo skeleton code | ✅ Có (setup ban đầu) |
| **👤 On-Premise Engineer** | Viết `modules/proxmox`, cấu hình VM, Cloud-Init, DB | ❌ Chỉ được commit code lên Git |
| **👤 AWS Engineer** | Viết `modules/aws`, cấu hình VPC, EC2, ALB | ❌ Chỉ được commit code lên Git |
| **👤 DevOps Reviewer** | Đọc plan, kiểm tra luồng mạng AWS ↔ Proxmox, bấm "Confirm & Apply" trên HCP UI | ✅ Người **duy nhất** được `apply` trên HCP |

> **Tại sao phải tách Reviewer riêng?** Đây là nguyên tắc **"Four-Eyes Principle"** — không ai được tự viết code lẫn tự phê duyệt code đó. Reviewer độc lập giúp phát hiện lỗi cấu hình (sai CIDR, thiếu Security Group, VM tạo nhầm network...) trước khi hạ tầng thực sự bị thay đổi.

---

## 2. Chuẩn Bị Hạ Tầng `👤 Workspace Admin`

> Toàn bộ mục này do **Workspace Admin** thực hiện — đây là công việc khởi tạo nền tảng **một lần duy nhất**. Sau khi hoàn thành, Engineer chỉ cần commit code lên Git, không cần tương tác với HCP hay Proxmox theo bất kỳ hình thức nào.

### 2.0 Git, AWS
- Chuẩn bị 1 tài khoản git (github, gitlab, ...)
- Chuẩn bị 1 tài khoản AWS
- Source code cho toàn bộ dự án [tại đây](https://github.com/ThongVu1996/terraform-hybrid-lab) vả  [tại đây](https://github.com/ThongVu1996/terraform-hybrid-lab-laravel-code)

### 2.1 Proxmox

#### 2.1.1 Yêu Cầu Chung

Trước khi bắt đầu, cần chuẩn bị các thành phần sau trên Proxmox:

- **Một node Proxmox** đang hoạt động.
- **Template `ubuntu-original`** đã được cài sẵn (xem hình).

  ![ubuntu orginal templete](./ubuntu-original.png)

- **Template có sẵn `cloud-init`** để có thể cấp IP tĩnh cho VM, thay vì dùng DHCP cấp động.
- **Một VM riêng biệt** dùng làm Agent của HCP Terraform Cloud.
- **Cấu hình Storage cho Cloud-Init:** Cho phép `cloud-init` lưu file cấu hình vào Proxmox qua đường dẫn: `Datacenter → Storage → chọn nơi lưu trữ (ví dụ: local) → Edit`.

  ![setting local storage](./setting-snipset-storage.png)

#### 2.1.2 Cài Đặt Ubuntu Template (Base)

**Bước 1: Tải Ubuntu ISO về Proxmox**

Có thể tải qua giao diện Web UI (nhập URL ISO vào ô URL):

![download ubuntu iso](./download-iso-ubuntu.png)

Hoặc dùng CLI trực tiếp trên Proxmox host:

```bash
cd /var/lib/vz/template/iso
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
```

**Bước 2: Tạo VM và cài Ubuntu**

Tạo máy ảo mới trên Proxmox:

![create vm](./create-vm.png)

Tiến hành cài Ubuntu lên máy ảo theo [hướng dẫn tại đây](https://www.youtube.com/watch?v=DjlGte968ko&list=PLsvroIvFNP1KU8foUeCC-hbJbqnAggWL2&index=4).

**Bước 3: Chuyển VM thành Template**

Sau khi cài đặt xong, right-click vào VM và chọn "Convert to Template":

![create template](./create-template.png)

#### 2.1.3 Tạo Ubuntu Template Tích Hợp Cloud-Init

Đây là bước quan trọng nhất, tạo ra một "golden template" chuẩn có sẵn `cloud-init` để Terraform có thể cấp IP tĩnh khi clone VM.

**Bước 1: Khởi chạy VM từ template base và kiểm tra Cloud-Init**

```bash
cat /etc/machine-id
ls -la /var/lib/cloud/
```

![check cloud-init](./check-clould-init.png)

- **Chuỗi machine-id (Mũi tên đỏ thứ nhất):** Mỗi máy ảo Linux (khi dùng DHCP hoặc quản lý mạng) cần một machine-id duy nhất. Việc mũi tên chỉ vào chuỗi ký tự này cho thấy máy ảo đang có một ID cụ thể. Nếu bạn giữ nguyên và clone (nhân bản) ra, các máy con sẽ bị trùng ID, dẫn đến xung đột mạng và không nhận được IP đúng.

- **Thư mục instances (Mũi tên đỏ thứ hai):** Thư mục này lưu trữ trạng thái (state) của những lần cloud-init đã cấu hình trước đó. Khi có dữ liệu ở đây, cloud-init sẽ hiểu là máy này "đã được khởi tạo mạng rồi". Khi bạn clone máy này ra, cloud-init trên máy mới sẽ bỏ qua bước cấu hình mạng ban đầu, khiến cho cấu hình IP tĩnh từ Terraform truyền vào bị vô tác dụng.


**Bước 2: Kích hoạt QEMU Guest Agent**

Đảm bảo agent đang chạy để Proxmox có thể giao tiếp với VM:

![enable qemu-guest-agent](./enable-qemu-guest-agent.png)

**Bước 3: Cài đặt các gói cần thiết**

```bash
sudo apt update
sudo apt install qemu-guest-agent cloud-init -y
sudo systemctl enable --now qemu-guest-agent
```

**Bước 4: Cấu hình hệ thống cho Template**

```bash
# Dừng và tắt dịch vụ tự cập nhật (tránh xung đột khi boot)
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades

# Cấu hình Cloud-Init KHÔNG tự update khi boot
sudo sed -i 's/package_upgrade: true/package_upgrade: false/' /etc/cloud/cloud.cfg

# Xóa file cấu hình mạng cũ (để Cloud-Init tự tạo file mới khi clone)
sudo rm -f /etc/netplan/*.yaml

# [Tùy chọn] Cho phép SSH bằng mật khẩu nếu không dùng SSH Key
# Cảnh báo: Không nên bật tùy chọn này trên môi trường production
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```
**Bước 5: Dọn dẹp và "niêm phong" VM**

> ⚠️ **Cực kỳ quan trọng:** Sau bước này, **KHÔNG được bật lại VM**. Mọi thay đổi sau khi bật lại sẽ làm hỏng template.

```bash
# 1. Dọn dẹp cache APT để giảm dung lượng template
sudo apt clean
sudo rm -rf /var/log/*

# 2. Xóa định danh máy (machine-id) để mỗi VM clone có ID mạng riêng, tránh xung đột
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id

# 3. Xóa dấu vết của Cloud-Init cũ để nó chạy lại hoàn toàn khi boot lần đầu
sudo cloud-init clean --logs

# 4. Tắt máy ngay lập tức
sudo poweroff
```

**Bước 6: Chuyển VM thành Template và thêm Cloud-Init Drive**

Sau khi VM đã tắt, chuyển nó thành template, sau đó thêm Cloud-Init Drive vào (có thể xóa CD/DVD Drive cũ để nhường cổng `ide2` cho Cloud-Init):

![add cloud-init 1](./add-clould-init-1.png)
![add-clould-init-2](./add-clould-init-2.png)
![add-clould-init-3](./add-clould-init-3.png)
![add new cloud drive to ide2](./add-new-clould-drive-to-ide2.png)

Sau khi hoàn thành, Admin đã có thể chỉ định IP tĩnh cho từng VM clone từ template này.

#### 2.1.4 Tạo `Api Tokens` và thêm permission

Vào giao diện web của Proxmox, chọn DataCenter, chọn **Permissions** -> **API Tokens** -> **Add**.

![create api token](./create-api-token.png)
![create api token result](./create-api-token-result.png)

Vào giao diện web của Proxmox, chọn DataCenter, **Permissions** -> **Add** -> **API Token** ở trên vào.

![use api token in permission](./use-api-token-in-permission.png)

#### 2.1.5 Thiết Lập `terraform-user` Cho Agent SSH

Bước này tạo một user giới hạn quyền để Agent SSH vào Proxmox host đẩy file Snippet — thay vì dùng `root` nguy hiểm. Đây là điều kiện tiên quyết để provider `bpg/proxmox` hoạt động.

**Bước 1: Tạo user hệ thống**

```bash
sudo useradd -m -s /bin/bash terraform-user
id terraform-user
```

**Bước 2: Cấu hình SSH Key cho Agent**

Admin ssh vào trong máy promox dán **Public Key** của Agent vào `authorized_keys` để Agent SSH vào mà không cần mật khẩu :

```bash
sudo mkdir -p /home/terraform-user/.ssh
sudo chmod 700 /home/terraform-user/.ssh
sudo chown -R terraform-user:terraform-user /home/terraform-user/
sudo nano /home/terraform-user/.ssh/authorized_keys
```

**Bước 3: Cấp quyền "Hộp cát" (Sandbox) cho thư mục Snippets**

Đây là thư mục sẽ lưu lại file giúp chạy script cài đặt ở lần đầu tiên máy VM được tại ra

```bash
sudo chown root:root /var/lib/vz/snippets
sudo chmod 1777 /var/lib/vz/snippets
```

> **Giải thích `chmod 1777`:**
> - `777`: Mọi user đều có quyền Đọc, Ghi trong thư mục.
> - Số `1` ở đầu **(Sticky Bit):** *"Dù ai cũng có quyền ghi vào đây, nhưng CHỈ CÓ NGƯỜI TẠO RA FILE mới có quyền Xóa hoặc Sửa file đó."*
> - **Tại sao vẫn an toàn?** Thư mục này chỉ accessible từ nội bộ Proxmox host, không exposed ra internet. Sticky Bit đảm bảo Agent chỉ quản lý được file do chính nó tạo ra.

> Luôn luôn cấp quyền cho thư mục .ssh là 700, còn câc private_key và public_key trong là 600 (áp dụng cho cả máy proxmox và máy cài agent HCP)
---

### 2.2 HCP Terraform Cloud

HCP Terraform Cloud đóng vai trò là "bộ não trung tâm" của toàn bộ hệ thống, chịu trách nhiệm lưu trữ State file, quản lý biến nhạy cảm và điều phối lệnh thực thi đến các Agent.

#### 2.2.1 Nguyên Lý Hoạt Động Của Terraform Agent

Terraform Agent hoạt động theo mô hình **"Outbound-Only"** (chỉ kết nối chiều ra) — đây là điểm mấu chốt về kiến trúc bảo mật:

```
┌─────────────────────────────────────────────────────────┐
│                   MẠNG NỘI BỘ (On-Premise)              │
│                                                         │
│   ┌──────────────┐   Pull lệnh   ┌──────────────────┐   │
│   │  VM Agent    │ ─────────────►│  HCP Terraform   │   │
│   │  (Proxmox)   │◄───────────── │  Cloud           │   │
│   └──────┬───────┘   Nhận lệnh   └──────────────────┘   │
│          │                         (Internet)           │
│          │ Gọi API                                      │
│          ▼                                              │
│   ┌──────────────┐                                      │
│   │  Proxmox API │                                      │
│   │  (Tạo VM)    │                                      │
│   └──────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

**Tại sao Agent an toàn hơn chạy trực tiếp từ máy local?**

| Tiêu chí | Chạy Local (Không Agent) | Dùng Terraform Agent |
| :--- | :--- | :--- |
| **Mở Port Inbound** | Phải mở port để HCP gọi vào | **Không cần mở bất kỳ port nào** |
| **Khởi tạo kết nối** | HCP Cloud → Máy local | Agent → HCP Cloud (chiều ra) |
| **Bảo mật Firewall** | Rủi ro cao, phải whitelist IP | An toàn, Agent chủ động poll lệnh |
| **Tính sẵn sàng** | Phụ thuộc vào máy cá nhân | Agent chạy 24/7 trên VM nội bộ |
| **Truy cập API nội bộ** | Chỉ khi người dùng đang online | Luôn sẵn sàng, độc lập với người dùng |
| **Credential** | Lưu trên máy cá nhân | Lưu an toàn trên HCP Cloud |

**Luồng hoạt động sau khi tích hợp VCS:**

1. On-Premise Engineer **commit code** lên Git repo.
2. HCP Terraform Cloud nhận webhook từ VCS, tự động tạo Run mới.
3. **Agent** (đang chạy trên VM Proxmox) liên tục **poll** HCP Cloud: *"Có lệnh nào cho tôi không?"*
4. Agent nhận lệnh, tải xuống code Terraform từ HCP Cloud, chạy `plan`.
5. DevOps Reviewer xem xét plan trên HCP UI và bấm **"Confirm & Apply"**.
6. Agent thực thi **ngay trên mạng nội bộ**, gọi Proxmox API để tạo VM.
7. Kết quả được gửi lên HCP Cloud để lưu State file và hiển thị log.

> **Kết luận:** Mạng nội bộ không bao giờ nhận kết nối từ bên ngoài. Toàn bộ giao tiếp là **Agent gọi ra ngoài** qua HTTPS port 443, loại bỏ hoàn toàn rủi ro từ các cuộc tấn công inbound.

#### 2.2.2 Cấu Trúc Tổ Chức Trên HCP Terraform

Luồng tạo tài nguyên theo thứ tự: `Organization → Project → Agent Pool → Workspace`

| Thành phần | Cấp độ | Mục đích |
| :--- | :--- | :--- |
| **Organization** | Gốc (Root) | Quản lý thành viên, cài đặt chung, hóa đơn |
| **Project** | Con của Org | Phân nhóm các Workspace theo phòng ban hoặc dự án |
| **Workspace** | Con của Project | Quản lý State, thực thi code Terraform cụ thể |
| **Agent Pool** | Thuộc Org | Cung cấp hạ tầng thực thi lệnh trong mạng nội bộ |

#### 2.2.3 Cách 1: Tạo Bằng Giao Diện Web UI

**Tạo Organization:**

Truy cập [https://app.terraform.io/app/organizations/new](https://app.terraform.io/app/organizations/new). Chọn plan **Business** nếu cần thêm thành viên vào làm cùng.

![create-orgranization](./creat-orgranization.png)
![create HCP group](./create-HCP-group.png)

**Tạo Workspace:**

![creat-HCP-workspace](./creat-HCP-workspace.png)

**Tạo Variables cho Workspace:**

Tại giao diện Workspace, chọn tab **Variables**. Đây là nơi quản lý toàn bộ cấu hình hạ tầng và bảo mật. Bạn cần phân biệt rõ hai loại biến dựa trên mục đích sử dụng:

| Loại biến (Category) | Mục đích chính | Yêu cầu trong Code | Ví dụ điển hình |
| :--- | :--- | :--- | :--- |
| **Terraform variable** | Định nghĩa **thuộc tính** hạ tầng | **Bắt buộc** khai báo trong `variables.tf` | `vm_name`, `cpu_count`, `ram_size` |
| **Environment variable**| Xác thực (Credentials) & Runtime | **Không cần** khai báo (Provider tự nhận) | `AWS_ACCESS_KEY_ID`, `PM_API_TOKEN_ID` |

**Quy tắc lựa chọn nhanh:**
- **Chọn `Terraform variable`:** Cho các giá trị cấu hình (thường thay đổi tùy môi trường Dev/Prod). Truy cập trong code qua `var.name`.
- **Chọn `Environment variable`:** Cho các mã khóa bí mật (Secrets). 
    - *Lưu ý:* Hầu hết các Provider (AWS, Proxmox) đều tự động tìm kiếm các biến môi trường chuẩn. Nếu muốn gán giá trị bí mật cho một biến trong Terraform mà không muốn lưu xuống file, hãy thêm tiền tố `TF_VAR_` (ví dụ: `TF_VAR_db_pass`).

**Tính năng `Sensitive` (Cực kỳ quan trọng):**
Tích chọn **`Sensitive`** cho tất cả các mật khẩu và mã khóa. Khi đó:
- Giá trị sẽ được lưu dưới dạng chuỗi ẩn — không ai (kể cả Admin) có thể xem lại giá trị đã nhập.
- Giá trị bị ẩn hoàn toàn trong Logs của Terraform, đảm bảo an toàn tuyệt đối khi Review.

![creat variables workspace hcp](./create-variable-workspace-HCP.png)
![create-variable-workspace-HCP 2](./create-variable-workspace-HCP-2.png)
![create-variable-workspace-HCP 3](./create-variable-workspace-HCP-3.png)
![create-variable-workspace-HCP 4](./create-variable-workspace-HCP-4.png)

Kết quả sau khi tạo — các biến được đánh dấu `Sensitive` sẽ hiển thị ở trạng thái "Write-only":

![create-variable-workspace-HCP 5](./create-variable-workspace-HCP-5.png)
![create-variable-workspace-HCP 6](./create-variable-workspace-HCP-6.png)

**Tạo Agent Pool:**

Vào `Organizations → Settings → Agents` để tạo Agent Pool mới:

![create agent](./create-agent.png)
![create agent-pools](./creat-agent-pool.png)

#### 2.2.4 Cách 2: Tạo Bằng API (Tự Động Hóa)

Trước tiên, lấy API Token tại `Account Settings`:

![api token](./api-token.png)

Khai báo biến môi trường:

```bash
export ORG_NAME="your-org-name"
export ORG_EMAIL="your-email@example.com"
export TFC_TOKEN="your-api-token"
```

**Tạo Organization:**

```bash
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data "{
    \"data\": {
      \"type\": \"organizations\",
      \"attributes\": {
        \"name\": \"$ORG_NAME\",
        \"email\": \"$ORG_EMAIL\"
      }
    }
  }" \
  "https://app.terraform.io/api/v2/organizations"
```

![organization api](./organization-api.png)

**Tạo Project:**

```bash
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{
    "data": {
      "type": "projects",
      "attributes": {
        "name": "Internal-Infrastructure"
      }
    }
  }' \
  "https://app.terraform.io/api/v2/organizations/$ORG_NAME/projects"
```

> 📌 Lưu lại giá trị `"id": "prj-xxxxx"` từ phản hồi JSON để dùng cho bước tạo Workspace.

![project api](./project-api.png)

**Tạo Agent Pool:**

```bash
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{
    "data": {
      "type": "agent-pools",
      "attributes": {
        "name": "proxmox-home-pool",
        "organization-scoped": true
      }
    }
  }' \
  "https://app.terraform.io/api/v2/organizations/$ORG_NAME/agent-pools"
```

> 📌 Lưu lại giá trị `"id": "apool-xxxxx"` từ phản hồi JSON để dùng cho bước tạo Workspace.

![agent api](./agent-api.png)

**Tạo Workspace:**

```bash
curl \
  --header "Authorization: Bearer $TFC_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "name": "mysql-prod-01",
        "execution-mode": "agent",
        "agent-pool-id": "apool-xxxx-id-cua-pool",
        "description": "Workspace cho DB Team"
      },
      "relationships": {
        "project": {
          "data": {
            "type": "projects",
            "id": "prj-xxxx-id-cua-project"
          }
        }
      }
    }
  }' \
  "https://app.terraform.io/api/v2/organizations/$ORG_NAME/workspaces"
```

![workspace api](./workspace-api.png)

Bảng tóm tắt các endpoint API:

| Tài nguyên | Method | Endpoint URL |
| :--- | :--- | :--- |
| **Organization** | `POST` | `/api/v2/organizations` |
| **Project** | `POST` | `/api/v2/organizations/{org_name}/projects` |
| **Agent Pool** | `POST` | `/api/v2/organizations/{org_name}/agent-pools` |
| **Workspace** | `POST` | `/api/v2/organizations/{org_name}/workspaces` |

#### 2.2.5 Cách 3: Tạo Bằng Terraform IaC (Khuyến Nghị)

Đây là cách được khuyến nghị nhất vì toàn bộ cấu hình HCP được quản lý dưới dạng code, dễ tái sử dụng và version control.

> 📌 **Lưu ý:** Đây là trường hợp **duy nhất hợp lệ** để dùng `terraform login` và `terraform apply` trực tiếp từ terminal — vì đây là Admin đang setup hạ tầng HCP lần đầu, không phải Engineer triển khai code.

Khai báo token vào terminal:

```bash
export TF_VAR_tfc_token="your-tfc-token"
```

**File `variables.tf`:**

```hcl
variable "tfc_token" {
  description = "User API Token lấy từ biến môi trường terminal"
  type        = string
  sensitive   = true
}

variable "org_name" {
  type    = string
  default = "your-org-name"
}

variable "org_email" {
  type    = string
  default = "admin@example.com"
}

variable "project_name" {
  type    = string
  default = "Internal-Services"
}

variable "agent_pool_name" {
  type    = string
  default = "proxmox-home-pool"
}

variable "workspace_name" {
  type    = string
  default = "mysql-automation-project"
}
```

**File `providers.tf`:**

```hcl
terraform {
  required_providers {
    tfe = {
      source  = "hashicorp/tfe"
      version = "~> 0.50.0"
    }
  }
}

provider "tfe" {
  token = var.tfc_token
}
```

**File `main.tf`:**

```hcl
resource "tfe_organization" "org" {
  name  = var.org_name
  email = var.org_email
}

resource "tfe_project" "project" {
  name         = var.project_name
  organization = tfe_organization.org.name
}

resource "tfe_agent_pool" "pool" {
  name                = var.agent_pool_name
  organization        = tfe_organization.org.name
  organization_scoped = true
}

resource "tfe_workspace" "workspace" {
  name           = var.workspace_name
  organization   = tfe_organization.org.name
  project_id     = tfe_project.project.id
  execution_mode = "agent"
  agent_pool_id  = tfe_agent_pool.pool.id
}

resource "tfe_agent_token" "token" {
  agent_pool_id = tfe_agent_pool.pool.id
  description   = "Token cho VM Agent trên Proxmox"
}

output "hcp_agent_token" {
  value     = tfe_agent_token.token.token
  sensitive = true
}
```

> 📎 Source code đầy đủ: [terraform-hybrid-lab/create-hcl-cloud](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-hcl-cloud)

Thực thi — đây là trường hợp Admin dùng `terraform apply` trực tiếp, hợp lệ vì đang setup hạ tầng HCP ban đầu:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

![terraform cloud](./terraform-cloud.png)

Lấy Agent Token sau khi apply:

```bash
terraform output hcp_agent_token
```

![hcp agent token](./hcp-agent-token.png)

Kiểm tra trên UI để xác nhận Execution Mode đã được cấu hình đúng:

![Excution Mode](./excution-mode.png)

---

### 2.3 Khởi Chạy Terraform Agent

> Agent phải được khởi chạy và kết nối HCP thành công **trước** khi tích hợp VCS. Lý do: khi Admin test commit sau bước 2.4, HCP sẽ lập tức trigger Run — nếu Agent chưa sẵn sàng thì Run sẽ bị treo ở trạng thái "Waiting for agent".

Trên VM đóng vai trò Agent (đang chạy trên Proxmox), Admin khởi chạy Agent bằng Docker:

```bash
export TFC_AGENT_TOKEN="your-agent-token"
export TFC_AGENT_NAME="proxmox-agent-01"

docker run \
  --platform=linux/amd64 \
  -e TFC_AGENT_TOKEN \
  -e TFC_AGENT_NAME \
  hashicorp/tfc-agent:latest
```

Kiểm tra Agent đã kết nối thành công bằng cách vào HCP UI → `Organizations → Settings → Agents` — Agent phải hiển thị trạng thái **"Idle"** (sẵn sàng nhận lệnh).

> 📖 Tham khảo thêm: [HashiCorp Terraform Cloud Agents](https://developer.hashicorp.com/terraform/tutorials/cloud/cloud-agents)

---

### 2.4 Kết Nối HCP Terraform Với VCS (GitHub/GitLab)

Đây là bước **then chốt** của toàn bộ kiến trúc. Sau khi hoàn thành, Engineer chỉ cần `git push` — HCP tự động trigger plan mà không cần Engineer có token HCP hay tương tác với HCP theo bất kỳ hình thức nào.

```
Trước khi kết nối VCS:         Sau khi kết nối VCS:
Engineer cần:                  Engineer chỉ cần:
  - Token HCP                    - Quyền push lên Git repo
  - terraform login
  - terraform plan
```

#### 2.4.1 Cấu Hình OAuth Trên HCP UI

Vào `Organization -> Workspace -> Settings → Verison Control -> Edit → Chon VCS Provider`, chọn GitHub hoặc GitLab và làm theo hướng dẫn tạo OAuth App:

- Đăng nhập vào GitHub/GitLab, tạo OAuth Application
- Điền `Client ID` và `Client Secret` vào HCP UI
- Authorize kết nối
- Các bước lần lượt thì xem ảnh

![Add VCS step 1](./connect-vcs-control-1.png)
![Add VCS step 2](./connect-vcs-control-2.png)
![Add VCS step 3](./connect-vcs-control-3.png)
![Add VCS step 4](./connect-vcs-control-4.png)
![Add VCS step 5](./connect-vcs-control-5.png)
![Add VCS step 6](./connect-vcs-control-6.png)
![Add VCS step 7](./connect-vcs-control-7.png)
![Add VCS step 8](./connect-vcs-control-8.png)

#### 2.4.2 Trỏ Workspace Vào Repo

Vào từng Workspace cần tích hợp → `Settings → Version Control`:

- Chọn VCS Provider vừa kết nối
- Chọn đúng **Repository** chứa code Terraform
- Chỉ định **Branch** (ví dụ: `main`) — HCP sẽ trigger plan khi có commit mới lên branch này
- Chỉ định **Working Directory** (ví dụ: `modules/proxmox`) — chỉ trigger khi có thay đổi trong thư mục này
- Bỏ tích hết tất cả  **Auto-apply** để đảm bảo plan không tự apply mà cần Reviewer duyệt

- Các bước lần lượt thì xem ảnh
![Connect WorkSpace with Repo step 1](./connect-to-repo-in-github-1.png)
![Connect WorkSpace with Repo step 2](./connect-to-repo-in-github-2.png) 
![Connect WorkSpace with Repo step 3](./connect-to-repo-in-github-3.png)
![Connect WorkSpace with Repo step 4](./connect-to-repo-in-github-4.png)
![Connect WorkSpace with Repo step 5](./connect-to-repo-in-github-5.png)
![Connect WorkSpace with Repo step 6](./connect-to-repo-in-github-6.png)
![Connect WorkSpace with Repo step 7](./connect-to-repo-in-github-7.png)
![Connect WorkSpace with Repo step 8](./connect-to-repo-in-github-8.png)

> Hướng dẫn cách đặt giá trị cho mục workspace direction cho việc trigger xem [tại đây](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/vcs#automatic-run-triggering) 

Sau khi cài đặt xong thì phải chạy manual 1 lần để HCP tạo ra file state đầu tiên, sau đó thì không cần chạy manual nữa mà chỉ cần push code lên Git là được.
![Run first time trigger](./connect-to-repo-in-github-9.png)

> Trong docs có viết 1 đoạn như sau Note: A workspace with no runs will not accept new runs via VCS webhook. At least one run must be manually queued to confirm that the workspace is ready for further runs. Tham khảo thêm [tại đây](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/run/ui)

- Cần setting để workspace sử dụng agent pool đã tạo ở bước 2.3 (Vào `workspace` -> `setting` -> `general` -> `Execution Mode`)
![Set agent-pool for workspace](./set-agent-pool-for-workspace.png)

#### 2.4.3 Kiểm Tra Kết Quả

Admin thực hiện một commit nhỏ (ví dụ: sửa comment trong code) lên branch đã cấu hình và theo dõi:

```
Admin git push (commit test)
        │
        ▼
GitHub/GitLab gửi webhook đến HCP
        │
        ▼
HCP tự động tạo Run mới trong Workspace
        │
        ▼
Agent nhận lệnh → Chạy terraform plan
        │
        ▼
Plan hiển thị trên HCP UI tab "Runs"
Trạng thái: "Planned - Needs Confirmation"
```

Nếu thấy Run xuất hiện và plan chạy thành công → pipeline thông suốt, sẵn sàng bàn giao cho Engineer.

- Cách test tham khảo trong ảnh
  - Push code
  ![Push code 1](./auto-run-hcp-when-commit-code-1.png)
  ![Push code 2](./auto-run-hcp-when-commit-code-1.png)
  - Plan, Confirm and apply
  ![Test HCP plan 1](./auto-run-hcp-when-commit-code-plan-1.png)
  ![Test HCP plan 2](./auto-run-hcp-when-commit-code-plan-2.png)
  - Apply
  ![Test HCP Apply](./auto-run-hcp-when-commit-code-apply-1.png)
  - Creeat VM in promox successful
  ![Create VM in promox](./auto-run-hcp-when-commit-code-promox-1.png)
  - Destroy (set count = 0 cho resources)
  ![Test HCP destroy](./auto-destroy-hcp.png)
  ![Test HCP destroy 1](./auto-destroy-hcp-1.png)
  ![Test HCP destroy 2](./auto-destroy-hcp-2.png)
  ![Test HCP destroy 3](./auto-destroy-hcp-3.png)

> ✅ **Từ thời điểm này:** Admin không cần can thiệp thêm vào luồng code. Mọi thay đổi hạ tầng đều đi qua: Engineer commit → HCP trigger → Reviewer duyệt → Agent thực thi.

---

### 2.5 Cấu Hình Tailscale ACL

Bước này Admin cấu hình policy kiểm soát traffic giữa các máy trong mạng Tailscale — xác định máy nào được nói chuyện với máy nào và qua port nào.

Vào **Tailscale Admin Console** → `Access Controls` -> chọn kiểu hiển thị là `JSON editor`, cấu hình ACL:

  ![tailscale access control setting 1](./tailscale-access-control-setting-1.png)

```json
{
	// Define the tags which can be applied to devices and by which users.
	"tagOwners": {
		"tag:terraform-tag-owner": ["anhthongvu1996@gmail.com"],
		"tag:database":            ["tag:terraform-tag-owner"],
		"tag:webserver":           ["tag:terraform-tag-owner"],
	},

	// Define grants that govern access for users, groups, autogroups, tags,
	// Tailscale IP addresses, and subnet ranges.
	"grants": [
		// Allow all connections.
		// Comment this section out if you want to define specific restrictions.
		// {"src": ["*"], "dst": ["*"], "ip": ["*"]},

		// Allow users in "group:example" to access "tag:example", but only from
		// devices that are running macOS and have enabled Tailscale client auto-updating.
		// {"src": ["group:example"], "dst": ["tag:example"], "ip": ["*"], "srcPosture":["posture:autoUpdateMac"]},
		{
			"src": ["tag:webserver"],
			"dst": ["tag:database"],
			"ip":  ["tcp:3306"],
		},
	],
  "ssh": [
		// Allow all users to SSH into their own devices in check mode.
		// Comment this section out if you want to define specific restrictions.
		{
			"action": "accept",
			"src":    ["tag:terraform-tag-owner"], // Thay bằng email của bạn
			"dst":    ["tag:database", "tag:webserver", "tag:terraform-tag-owner"],
			"users":  ["ubuntu", "root", "thong"],
		},
	],
}
```

Kết quả setting:
  ![tailscale access control setting 2](./tailscale-access-control-setting-check-1.png)
  ![tailscale access control setting 3](./tailscale-access-control-setting-check-2.png)

- Cần cấu hình như này để có thể chỉ định được máy nào được nói chuyện với máy nào và qua port nào.
- Ở mục `ssh` ta setting 1 cái cài đựa như trên, thì chính là máy admin có thể ssh vào các máy khác, giúp kiểm tra check kết quả, hoặc lỗi khi cần thiết.
- Ta có thể nhìn ảnh trên để thấy:
  - Là khi ta để `{"src": ["*"], "dst": ["*"], "ip": ["*"]}`, thì tắt cả các máy trong tailnet cùng 1 mạng có thể nhìn tháy nhau (phần số 1). 
  - Nhưng khi ta config cụ thể hơn `{"src": ["tag:webserver"], "dst": ["tag:database"], "ip": ["tcp:3306"]}`, thì webserver chỉ có thể nói chuyện với database qua port 3306 (phần số 2). 
  - Tương tự như vậy ta có thể cấu hình cho các máy khác. Cái `ssh` chỉ ra rằng ta có thể dùng ssh để truy cập vào máy đó.
  - Ngoài ra việc dùng tag còn để đảm bảo `This machine has key expiry disabled and never needs to reauthenticate.`. Key Expiry Disabled: Tailscale tự động vô hiệu hóa việc hết hạn. Máy ảo Proxmox của bạn sẽ Online 1 năm, 2 năm hoặc mãi mãi mà không bao giờ bị bắt "đăng nhập lại". Never needs to reauthenticate: Bạn không bao giờ phải gõ lệnh tailscale up bằng tay lần thứ hai hay dán lại Key mới.
  - Nếu không thì Key Expiry (Hết hạn khóa): Mặc định sau 180 ngày, Tailscale sẽ "đá" bạn ra. Bạn phải mở trình duyệt, đăng nhập lại để chứng minh mình vẫn là chủ sở hữu.

**Kết quả:**

| Nguồn | Đích | Port | Kết quả |
| :--- | :--- | :--- | :--- |
| Web Server (AWS) | VM DB (Proxmox) | 3306 (MySQL) | ✅ Được phép |
| Máy Admin | VM DB (Proxmox) | 22 (SSH) | ✅ Được phép |
| VM DB | Bất kỳ đâu khác | Bất kỳ | ❌ Bị chặn |

Khi ta chạy tailscale trên các máy ta sẽ dùng đến câu lệnh sau

  ```bash
    tailscale up --authkey=${TailScale_Key} --hostname=db-server --ssh --accept-dns=true
  ```

- Chúng ta có thể lấy TailScale_Key trực tiếp từ `Personal Settings` như hình, nhưng mà nó sẽ bị lộ ra ở phần state dù chúng ta đã setting nó là `sensitive = true`.
  ![create person token](./create-person-token.png)
  ![person token leak](./person-token-leak.png)

- Để khắc phục điều đó chúng ta sử dụng đến `OAuth clients` nó có tác dụng sinh ra một key ở mỗi lần tạo và key đó sẽ có 1 thời gian sống cố định (thường để là 60 phút). Như vậy nếu nó bị lộ thì sẽ hết hạn rồi, không ảnh hưởng nữa. Vào đây `https://login.tailscale.com/admin/settings/general`, hoặc nhìn trong ảnh
  ![create oauth client](./create-oauth-client.png)
  ![create oauth client 1](./create-oauth-client-1.png)
  ![setting oauth client](./setting-oauth-client-token.png)

> **Tại sao không cần Security Group AWS hay VPN riêng?** Toàn bộ traffic giữa Web Server AWS và VM DB Proxmox đi qua **Tailscale tunnel (WireGuard, mã hóa end-to-end)**. Hai máy giao tiếp với nhau qua Tailscale IP (`100.x.x.x`), không phải qua public IP — nên Security Group không liên quan, VPN riêng là over-engineering.

---

## 3. Viết Code Terraform cho Proxmox `👤 On-Premise Engineer`

> Engineer không có token HCP, không tương tác với HCP theo bất kỳ hình thức nào. Nhiệm vụ duy nhất là **viết code** trong `modules/proxmox` và **commit lên Git** — HCP tự lo phần còn lại.

### 3.1 VCS-Driven Workflow — Luồng Chuẩn

Đây là luồng chuẩn sau khi Admin đã hoàn thành toàn bộ mục 2:

```
On-Premise Engineer      GitHub Repo         HCP Terraform        DevOps Reviewer
        │                      │                    │                     │
        │  viết code           │                    │                     │
        │  modules/proxmox     │                    │                     │
        │  git commit          │                    │                     │
        │  git push ──────────►│                    │                     │
        │                      │  webhook trigger   │                     │
        │                      │───────────────────►│                     │
        │                      │                    │ Auto chạy Plan      │
        │                      │                    │ (qua Agent)         │
        │                      │                    │────────────────────►│
        │                      │                    │                     │ Đọc plan
        │                      │                    │                     │ Kiểm tra
        │                      │                    │◄─ Confirm & Apply ──│
        │                      │                    │                     │
        │                      │                    │ Agent nhận lệnh     │
        │                      │                    │ Proxmox API tạo VM  │
        │                      │                    │                     │
        │◄─── Output: IP VM ───│◄───────────────────│                     │
```

**Những gì Engineer cần biết:**
- Viết code Terraform trong thư mục được phân công
- Commit và push lên đúng branch đã được Admin cấu hình
- Theo dõi kết quả Run trên HCP UI (chỉ cần quyền xem, không cần token)

**Những gì Engineer không cần biết:**
- Token HCP, IP Proxmox, credential bất kỳ
- Cách HCP kết nối với Agent
- Cách Agent tương tác với Proxmox API

**Cần chuẩn bị các biến trên HCP**
Dự án này yêu cầu hai loại biến cấu hình trên HCP Terraform: **Terraform Variables** (Dành cho thông số hạ tầng) và **Environment Variables** (Dành cho việc bảo mật API Key / Mật khẩu và cấu hình Provider).

* Đây là các biến cấu hình cơ bản không yêu cầu tiền tố `TF_VAR_`. Khi tạo, bạn chọn Category là `Terraform variable`.*

| Tên biến (Key) | Thuộc tính | Mô tả ý nghĩa tác dụng | Cách lấy / Xác định giá trị |
| :--- | :--- | :--- | :--- |
| `db_name` | String | Tên Database sẽ được tạo tự động bên trong MySQL khi cài đặt. | Tự đặt tên theo ý muốn (VD: `appdb`). Không được có khoảng trắng. |
| `db_user` | String | Tên tài khoản Database dùng để kết nối vào `db_name`. | Tự đặt tên theo ý muốn (VD: `appuser`). |
| `proxmox_node` | String | Tên Node của máy chủ Proxmox nơi VM sẽ được tạo ra. | Nhìn ở cột bên trái trong giao diện Web Proxmox (thường mặc định là `pve` hoặc `promox`). ![promox node](./promox-node.png)|
| `proxmox_ssh_private_key` | **Sensitive** | Mã Khóa riêng tư giúp Agent SSH vào Proxmox host. | Đây là private key giúp cho máy agent HCP chạy bằng docker có thể ssh vào proxmox và lưu file vào trong proxmox. (Tạo bằng lệnh `ssh-keygen` và copy nội dung).|
| `user_name` | String | Tên user chính của máy ảo Linux. | Tên user dùng để đăng nhập qua ssh vào máy VM được tạo ra (VD: `thong`, `admin`, `ubuntu`). |
| `vm_gateway` | String | Địa chỉ IP Gateway mạng sinh ra máy ảo. | Xem thông số lớp mạng local mà máy Proxmox của bạn đang chạy (VD: `172.199.10.1`). ![ip gateway](./ip-gateway.png)|
| `vm_ip_cidr` | String | Địa chỉ IP tĩnh sẽ cấp cho máy ảo kèm theo Subnet Mask. | Tự chọn 1 IP rảnh trong dải mạng Proxmox cấp (VD: `172.199.10.150/24`). Vui lòng kèm `/24` hoặc subnet tương ứng. |

<br>

*Đây là các biến môi trường để cấu hình thông số kết nối hệ thống (Provider) và ẩn toàn vẹn mật khẩu thông qua tiền tố `TF_VAR_`. Khi tạo, bạn chọn Category là `Environment variable`.*

| Tên biến (Key) | Thuộc tính | Mô tả ý nghĩa tác dụng | Cách lấy / Xác định giá trị |
| :--- | :--- | :--- | :--- |
| `AWS_ACCESS_KEY_ID` | String | Key ID của người dùng AWS IAM, dùng để cấp quyền cho AWS Provider. | Có thể xem [tại đây](https://github.com/ThongVu1996/documents/blob/main/cd-ci-lab/aws/install.md) search `2. Tạo token` |
| `AWS_SECRET_ACCESS_KEY` | **Sensitive** | Khóa bí mật đi kèm Access Key AWS. | Lấy cùng lúc với bước tạo Access Key ID ở trên. |
| `PM_API_URL` | String | URL API Proxmox (áp dụng cho Telmate/Proxmox Provider). | Thường là `https://<IP>:8006/api2/json`. |
| `PM_API_TOKEN_ID` | String | Token ID (áp dụng cho Telmate/Proxmox Provider). | Định dạng `USER@REALM!TOKENID` (VD: `root@pam!terraform-module-thong`). Lấy tương tự với `PROXMOX_VE_API_TOKEN` vì bản chất chúng là 1 chỉ là dùng cho 2 provider khác nhau thôi |
| `PM_API_TOKEN_SECRET` | **Sensitive** | Secret của Token ID trên (áp dụng cho Telmate/Proxmox). | Lấy khi tạo API Token trên Proxmox. (Xem mục 2.1.4 ở trên) |
| `PROXMOX_VE_ENDPOINT` | String | URL API Proxmox (áp dụng cho BPG Provider). | Địa chỉ giúp call các API được xây dựng cho Proxmox.Thường là `https://<IP>:8006/api2/json`. |
| `PROXMOX_VE_API_TOKEN` | **Sensitive** | API Token (áp dụng cho BPG Provider). | Giá trị (Xem mục 2.1.4 ở trên). Format: `USER@REALM!TOKENID=SECRET`. ![promox api token](./promox-api-token.png).|
| `TAILSCALE_OAUTH_CLIENT_ID` | String | ID OAuth Client Tailscale. | Lấy từ trang quản trị Tailscale (Xem mục 2.5). |
| `TAILSCALE_OAUTH_CLIENT_SECRET` | **Sensitive** | Secret OAuth Client Tailscale. | Lấy sau khi tạo OAuth client (Xem mục 2.5). |
| `TF_VAR_db_password` | **Sensitive** | Mật khẩu cho MySQL `db_user`. | Tự đặt một chuỗi mật khẩu mạnh để Terraform tự tiêm vào. |
| `TF_VAR_tailscale_auth_key` | **Sensitive** | Auth Key Tailscale (áp dụng cho Provisioner). | Key dùng để authenticate máy ảo vào mạng Tailscale. |
| `TF_VAR_user_password` | **Sensitive** | Mật khẩu đăng nhập cho máy ảo Linux. | Mật khẩu của user của máy VM được tạo ra bằng Terraform. |

---

> ⚠️ **Lưu ý Chung Quan Trọng:**
> 1. **Cẩn trọng với thông tin Sensitive:** Hãy rà soát lại và check mục **Sensitive** trên UI đối với các biến chứa Private Key hay Password. Nó sẽ giúp ẩn nội dung đi (chỉ có trạng thái *writeonly*) và không in log ra màn hình console.
> 2. **Sử dụng đúng tiền tố `TF_VAR_`:** Với các biến cấu hình bảo mật ở phần Environment Variable, phải gõ đúng chữ `TF_VAR_` thì sau đó HCP Cloud mới đối chiếu và map (gắn) khớp với biến tương ứng có trong HCL code của bạn. Ví dụ `TF_VAR_user_password` sẽ tự truyền giá trị vào `variable "user_password" { ... }`.
> 3. **Loại bỏ khai báo thừa:** Nếu trong thực hành (như ảnh chiếu), bạn thấy Tailscale Secret bị thêm 2 lần ở cả `Environment Variable` và `Terraform Variable` => Hãy thống nhất theo cấu trúc code của bạn. Lời khuyên là nếu truyền vào Script Bash Cloud-Init nên đặt nó ở Terraform Variable, hoặc nếu cấp sẵn cho Provider thì đặt ở Environment Variable. Không nên đặt dư thừa ở cả 2 nơi.
> 4. Với các biến cấu hình máy (như số core, tên máy,...) nên dùng `Terraform Variable`, với các biến nhạy cảm (db_password, user_password,...) nên dùng `Enviroment Variable` đưới dạng `TF_VAR_ten_bien`. Còn với các biến cấu hình Provider (như AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY,...) thì nên dùng `Enviroment Variable` không cần tiền tố `TF_VAR_`, vì nó sẽ được map trực tiếp vào provider tương ứng mà các tác giả đã viết trước. Cuối cùng là `Enviroment Variable` được thêm vào quá trình runtime, nên nó sẽ không được lưu vào trong file state ở agent, nên không thể lộ thông tin được. Hình dưới chứng minh là các biến sẽ được lưu vào 1 file dưới agent, dù nó có thể bị xóa sau khi VM tạo xong nhưng vẫn có nguy cơ bị lộ:
![save data sentisive to agent](./save-data-sentisive-to-agent.png)

### 3.2 ⚠️ Bad Practice — `terraform login` Từ Máy Engineer

Trong quá trình phát triển, có thể thấy trong một số tài liệu hoặc ví dụ hướng dẫn Engineer dùng `terraform login` rồi chạy `terraform plan` từ terminal cá nhân. **Đây là bad practice nghiêm trọng** trong kiến trúc này.

**Bảng so sánh:**

| Tiêu chí | ❌ Bad Practice — `terraform login` từ máy Engineer | ✅ VCS-Driven — git push |
| :--- | :--- | :--- |
| **Engineer cần token HCP** | Có — rủi ro lộ token nếu máy bị xâm phạm | Không cần |
| **Trigger plan** | Thủ công, phụ thuộc máy cá nhân đang online | Tự động khi có commit |
| **Audit trail** | Không rõ ai chạy lúc nào từ máy nào | Git commit history + HCP Runs history đầy đủ |
| **Kiểm soát biến môi trường** | Biến có thể bị ghi đè từ local | Chỉ dùng biến trong HCP Sensitive Variables |
| **Nguy cơ apply nhầm** | Engineer có thể vô tình chạy `apply` | Engineer không có quyền apply |
| **Phù hợp với bài lab** | ❌ Vi phạm Four-Eyes Principle | ✅ Đúng tinh thần |

**Rủi ro cụ thể khi Engineer có token HCP:**

```
Nếu Engineer có token HCP và dùng terraform login:
  ├── Có thể đọc State file → thấy IP nội bộ, credential output
  ├── Có thể tự ý chạy plan/apply bất cứ lúc nào
  ├── Bypass hoàn toàn Four-Eyes Principle
  └── Nếu token bị lộ → attacker có toàn quyền với workspace
```

> **Kết luận:** `terraform login` và `terraform apply/plan` từ terminal chỉ dành cho **Workspace Admin** khi setup hạ tầng HCP ban đầu (mục 2.2.5). Sau khi Admin hoàn thành setup và kết nối VCS, không ai cần dùng `terraform login` nữa.

Các ảnh dưới đây minh hoạ bad practice — Engineer dùng `terraform login` và chạy plan từ terminal — **không nên làm theo**:

![terraform login](./terraform-login.png)
![terraform login success](./terraform-login-success.png)

### 3.3 So Sánh Phương Pháp Cài Đặt Phần Mềm Lên VM

Có 3 cách để cài đặt phần mềm (MySQL, Tailscale...) lên VM sau khi tạo. Bảng dưới đây so sánh ưu nhược điểm:

| Tiêu chí | Provisioner | Cloud-Init (telmate) | Cloud-Init (bpg) |
| :--- | :--- | :--- | :--- |
| **Độ ổn định** | Thấp — dễ lỗi, không idempotent | Trung bình | Cao |
| **Inject biến từ HCP** | Có | Không (phải tạo file thủ công) | **Có** |
| **Tự động hóa Snippet** | N/A | Admin phải tạo file tay trên Proxmox | **Agent tự đẩy file** |
| **Được HashiCorp khuyến nghị** | ❌ Không | ✅ Có | ✅ Có |
| **Quản lý vòng đời** | Không | Hạn chế | **Đầy đủ (Resource ID)** |
| **Engineer độc lập với Admin** | Không | ❌ Không — cần Admin tạo Snippet thủ công | ✅ Có |

> 📖 Tham khảo thêm: [Provisioner vs Cloud-Init: Khi nào nên dùng cái nào?](https://oneuptime.com/blog/post/2026-02-23-terraform-provisioners-last-resort/view)

### 3.4 Phương Pháp 1: Provisioner (Không Khuyến Nghị)

> ⚠️ HashiCorp chính thức không khuyến nghị dùng `provisioner` cho production vì thiếu tính idempotent, khó debug và phụ thuộc vào kết nối SSH tại thời điểm `apply`.

![nhược điểm của provisioner](./disadvance-provisioner.png)

Tham khảo code: [terraform-hybrid-lab/create-vm](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-vm)

**Luồng thực thi (VCS-driven):**

Engineer viết code module, commit và push lên Git. HCP tự động trigger plan, Reviewer xem xét và bấm Confirm & Apply.

Kết quả sau khi Reviewer apply thành công:

![creat VM success](./creat-VM-success.png)
![create VM success hcp cloud](./create-VM-success-hcp.png)
![create VM success proxmox](./create-VM-success-proxmox.png)
![check mysql in VM](./check-mysql-vm.png)

### 3.5 Phương Pháp 2: Cloud-Init với Provider `telmate/proxmox`

Provider `telmate/proxmox` yêu cầu tạo file Snippet **thủ công** trên Proxmox host trước khi chạy Terraform.

> ⚠️ **Hạn chế phân quyền:** File Snippet phải do **Workspace Admin** tạo trực tiếp trên Proxmox host. Engineer không có quyền truy cập host nên mỗi khi cần sửa script cài đặt, phải nhờ Admin can thiệp thủ công — phá vỡ tính độc lập và làm chậm chu kỳ phát triển.

Tham khảo code: [terraform-hybrid-lab/create-vm-with-cloud-init](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-vm-with-cloud-init)

Admin tạo file `/var/lib/vz/snippets/setup.yaml` trực tiếp trên Proxmox host:

```yaml
#cloud-config
write_files:
  - path: /root/setup_nodes.sh
    permissions: '0755'
    content: |
      #!/bin/bash
      set -e

      echo "=== [1/3] Dọn dẹp APT & Cài đặt MySQL ==="
      systemctl stop unattended-upgrades || true
      rm -f /var/lib/apt/lists/lock /var/cache/apt/archives/lock /var/lib/dpkg/lock*
      sed -i "s/[a-z]*.archive.ubuntu.com/vn.archive.ubuntu.com/g" /etc/apt/sources.list
      apt-get update -y
      apt-get install -y mysql-server curl

      echo "=== [2/3] Cài đặt & Kích hoạt Tailscale ==="
      # ... (thêm bước cài Tailscale tại đây)

      echo "=== [3/3] Hoàn thành ==="
```

Engineer commit code module lên Git → HCP trigger → Reviewer duyệt → Agent apply.

### 3.6 Phương Pháp 3: Cloud-Init với Provider `bpg/proxmox` (Khuyến Nghị)

Đây là provider thế hệ mới, giải quyết hoàn toàn hạn chế của `telmate` — Engineer kiểm soát toàn bộ nội dung script ngay trong code, không cần Admin can thiệp vào Proxmox host.

**Ưu điểm vượt trội:**

- **Variable Injection:** Biến (mật khẩu DB, Tailscale Auth Key) từ HCP Sensitive Variables được inject **trực tiếp vào nội dung YAML** khi Agent thực thi.
- **Quản lý tài nguyên:** Snippet là Resource có ID — vòng đời được quản lý đầy đủ.
- **Tự động hóa hoàn toàn:** Agent SSH vào Proxmox host qua `terraform-user` (đã được Admin tạo ở mục 2.1.5), đẩy file Snippet lên trước khi tạo VM.

Tham khảo code: [terraform-hybrid-lab/create-vm-with-cloud-init](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-vm-with-cloud-init-bpg-proxmox)

#### 3.6.1 Cấu Hình Provider

Engineer khai báo kết nối SSH trong `provider.tf`, sử dụng `terraform-user` mà Admin đã tạo sẵn:

```hcl
provider "proxmox" {
  # ... cấu hình endpoint và token (lấy từ HCP Sensitive Variables) ...

  ssh {
    agent       = false
    username    = "terraform-user"
    private_key = var.proxmox_ssh_private_key
  }
}
```

#### 3.6.2 Bảo Mật Network VM — iptables

Ngoài Tailscale ACL mà Admin đã cấu hình ở mục 2.5, Engineer bổ sung thêm lớp bảo vệ thứ hai ngay trong cloud-init script: chỉ cho phép traffic qua interface `tailscale0`, DROP tất cả traffic non-Tailscale.

Điều này đảm bảo VM DB không thể kết nối internet thông thường dù Tailscale interface bị tắt hay bypass:

```bash
# Trong cloud-init script của VM DB
# Lớp 2: iptables chặn tất cả traffic không phải Tailscale

# Cho phép loopback và traffic Tailscale
iptables -A INPUT  -i lo          -j ACCEPT
iptables -A INPUT  -i tailscale0  -j ACCEPT
iptables -A INPUT  -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT  -j DROP

iptables -A OUTPUT -o lo          -j ACCEPT
iptables -A OUTPUT -o tailscale0  -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -j DROP

# Persist sau khi reboot
apt-get install -y iptables-persistent
netfilter-persistent save
```

**Kết quả 2 lớp bảo vệ kết hợp:**

```
Lớp 1 — Tailscale ACL (Admin cấu hình):
  Web Server AWS  → VM DB port 3306  ✅
  Admin SSH   → VM DB port 22    ✅
  VM DB           → bất kỳ đâu khác ❌

Lớp 2 — iptables trong VM (Engineer viết trong cloud-init):
  Chỉ traffic qua tailscale0 mới được xử lý
  Tất cả traffic internet thông thường bị DROP
```

#### 3.6.3 Luồng Thực Thi

Engineer commit toàn bộ code (provider, module, cloud-init script) lên Git:

```
Engineer git push
        │
        ▼
HCP nhận webhook → tạo Run
        │
        ▼
Agent chạy terraform plan
        │
        ▼
DevOps Reviewer xem plan trên HCP UI
Kiểm tra: resource nào được tạo, biến nào được dùng,
network config có đúng không
        │
        ▼
Reviewer bấm "Confirm & Apply"
        │
        ▼
Agent thực thi:
  1. SSH vào Proxmox host bằng terraform-user
  2. Đẩy cloud-init Snippet (có inject biến từ HCP)
  3. Gọi Proxmox API tạo VM từ template
  4. VM boot, cloud-init chạy: cài MySQL, Tailscale, iptables
        │
        ▼
VM sẵn sàng — Output: Tailscale IP
```
- Kết quả khi chạy bằng VCS driven cũng sẽ giống với cách đăng nhập (vì cả 2 đều dùng chung code cấu hình mà)

#### 3.6.4 Kết Quả Sau Khi Apply

Kiểm tra Tailscale sau khi VM được tạo thành công:

![tailscale-provisioner-1](./tailscale-provisioner-1.png)
![tailscale-provisioner-2](./tailscale-provisioner-2.png)

**Truy cập VM qua Tailscale SSH:**

Developer không cần biết mật khẩu hay SSH Key của hạ tầng, chỉ cần:

1. Cài Tailscale trên máy Dev (Admin đã cấp tag `developer`).
2. Chạy lệnh: `tailscale ssh username@ten-goi-trong-tailscale`.
3. Xác thực qua trình duyệt (SSO) → Vào thẳng VM.

---

## 4. Tổng Kết — Zero-Touch, VCS-Driven & Four-Eyes

### 4.1 Toàn Bộ Luồng Từ Đầu Đến Cuối

```
👤 Admin setup (mục 2)
  └─► Proxmox template, terraform-user, HCP, Agent, VCS, Tailscale ACL
                    │
                    ▼ (một lần duy nhất)
👤 Engineer commit (mục 3)
  └─► git push → webhook → HCP tạo Run → Agent plan
                    │
                    ▼ (mỗi lần có commit)
👤 Reviewer duyệt
  └─► Đọc plan trên HCP UI → Confirm & Apply
                    │
                    ▼
Agent thực thi trên mạng nội bộ
  └─► Proxmox API → Tạo VM → cloud-init → MySQL + Tailscale + iptables
                    │
                    ▼
Developer truy cập qua Tailscale SSH
```

### 4.2 Hai Ranh Giới Bảo Mật Rõ Ràng

**Ranh giới 1 — Hạ tầng (tuyệt đối):**

Không ai ngoài Admin và Agent có thể chạm vào lớp này.

| Tài sản | Bảo vệ như thế nào |
| :--- | :--- |
| Proxmox API Token | HCP Sensitive Variable — không ai đọc được |
| IP nội bộ Proxmox host | HCP Sensitive Variable — không ai đọc được |
| SSH Private Key của Agent | HCP Sensitive Variable — không ai đọc được |
| State file (chứa output IP, credential) | HCP mã hóa — Engineer không tải về được |
| Proxmox host SSH | Chỉ Agent dùng terraform-user, không ai khác có key |

**Ranh giới 2 — Bên trong VM sau khi tạo (có kiểm soát):**

Developer có quyền truy cập vào bên trong VM qua Tailscale SSH. Đây là rủi ro được chấp nhận có chủ đích vì VM là môi trường làm việc của họ. Các biện pháp giảm thiểu:

- **Tailscale ACL:** Developer chỉ SSH được vào đúng VM của mình, không sang VM khác
- **iptables:** VM không thể kết nối internet thông thường, chỉ qua Tailscale tunnel
- **Tailscale Audit Log:** Ghi lại ai SSH vào lúc nào, làm gì
- **Ephemeral VM:** VM tồn tại ngắn — xóa đi khi không cần, không để lại dấu vết lâu dài

### 4.3 Những Gì Engineer Không Bao Giờ Biết

| Thông tin nhạy cảm | Nằm ở đâu | Engineer có thấy không? |
| :--- | :--- | :--- |
| Proxmox API Token | HCP Cloud (Sensitive Variable) | ❌ Ẩn hoàn toàn |
| IP nội bộ của Proxmox host | HCP Cloud (Sensitive Variable) | ❌ Ẩn hoàn toàn |
| Mật khẩu MySQL / DB | HCP Cloud (Sensitive Variable) | ❌ Ẩn hoàn toàn |
| Tailscale Auth Key | HCP Cloud (Sensitive Variable) | ❌ Ẩn hoàn toàn |
| SSH Private Key của Agent | HCP Cloud (Sensitive Variable) | ❌ Ẩn hoàn toàn |
| Nội dung State file | HCP Cloud (được mã hóa) | ❌ Không thể tải về |
| Token HCP | Không được cấp | ❌ Engineer không có |
| Kết nối trực tiếp vào Proxmox | Chỉ Agent mới có | ❌ Không có đường vào |

### 4.4 Lưu Ý Về Phân Quyền

| Role | Cần gì để làm việc | KHÔNG cần |
| :--- | :--- | :--- |
| **Admin** | User Token HCP (toàn quyền) | — |
| **On-Premise / AWS Engineer** | Quyền push lên Git repo | Token HCP, truy cập Proxmox |
| **DevOps Reviewer** | Tài khoản HCP (xem + approve) | Quyền push Git, truy cập Proxmox |

### 4.5 Tóm Tắt Giá Trị Của Mô Hình

```
✅ Zero-Touch Infrastructure   — Engineer không chạm vào Proxmox, không SSH vào host
✅ Zero Credential Exposure    — Engineer không có token HCP, không có credential nào
✅ Zero Inbound Port           — Mạng nội bộ không mở bất kỳ port nào ra ngoài
✅ VCS-Driven                  — Mọi thay đổi đều có Git commit làm bằng chứng, tự động trigger
✅ Four-Eyes Principle         — Mọi apply đều cần Reviewer duyệt
✅ Full Audit Trail            — Git history + HCP Runs history + Tailscale audit log
✅ Network Isolation           — Tailscale ACL (tầng policy) + iptables (tầng VM)
```

---

## 5. Use Cases Thực Tế

Mô hình Proxmox + HCP Terraform Agent không chỉ giải quyết bài toán tạo DB server. Dưới đây là các tình huống thực tế mà kiến trúc này phát huy tối đa giá trị.

> **Nguyên tắc xuyên suốt:** Engineer commit code lên Git → HCP trigger plan → Reviewer duyệt → Agent apply. Không có ngoại lệ nào.

---

### 5.1 Dev Environment On-Demand (Môi Trường Dev Theo Yêu Cầu)

**Bối cảnh:**
Team có 8 developer, mỗi người cần môi trường riêng để phát triển và test mà không conflict với nhau. Trước đây tất cả dùng chung một server dev dẫn đến xung đột port, dữ liệu bị ghi đè lẫn nhau.

**Cách áp dụng:**

```
On-Premise Engineer              HCP + Agent            DevOps Reviewer
        │                            │                        │
        │  Sửa code: thêm VM mới     │                        │
        │  owner = "alice"           │                        │
        │  git push ─────────────────►                        │
        │                            │ Auto trigger plan      │
        │                            │────────────────────────►
        │                            │                        │ Xem plan
        │                            │                        │ Xác nhận VM
        │                            │◄────Confirm & Apply ---│
        │                            │                        │
        │                            │ Agent tạo VM:          │
        │                            │   dev-alice-env        │
        │                            │   Cài Node.js, MySQL   │
        │◄── Output: Tailscale IP ───│                        │
        │                            │                        │
        │  [Xong việc: xóa VM]       │                        │
        │  Sửa code: xóa resource    │                        │
        │  git push ─────────────────►                        │
        │                            │ Auto trigger plan      │
        │                            │───────────────────────►|
        │                            │◄───── Confirm & Apply ─│
        │                            │ Agent xóa VM           │
```

**Lợi ích đạt được:**

- Không còn xung đột tài nguyên — mỗi người có môi trường riêng biệt
- Mọi tạo/xóa VM đều có Git commit history và HCP Runs history đầy đủ
- Chi phí điện/phần cứng giảm vì không có VM "zombie" chạy qua đêm

**Cấu hình Terraform tham khảo:**

```hcl
variable "owner" {
  type        = string
  description = "Tên dev sở hữu môi trường này"
}

variable "ttl_hours" {
  type        = number
  default     = 8
  description = "Số giờ VM tồn tại"
}

resource "proxmox_vm_qemu" "dev_env" {
  name = "dev-${var.owner}-env"
  tags = "owner=${var.owner},ttl=${var.ttl_hours}h,team=engineering"
}
```

---

### 5.2 Isolated Database Per Feature Branch (DB Riêng Cho Từng Nhánh)

**Bối cảnh:**
Team backend phát triển tính năng yêu cầu thay đổi schema database. Nếu chạy migration trên DB dùng chung, toàn bộ team bị ảnh hưởng.

**Cách áp dụng:**

```
Git Push → feature/payment-v2
          │
          ▼
HCP tự động trigger plan
(VCS integration, không cần Engineer làm gì thêm)
          │
          ▼
DevOps Reviewer xem plan trên HCP UI
Confirm & Apply
          │
          ▼
Agent tạo VM: db-payment-v2
MySQL: paymentdb_v2 (schema sạch)
          │
          ▼
CI chạy migration + test trên DB riêng biệt
          │
          ▼
PR merge → Engineer xóa resource trong code
         → git push → HCP trigger → Reviewer duyệt → VM bị xóa
```

**Lợi ích đạt được:**

- Loại bỏ tình trạng "migration của tôi break CI của bạn"
- Dev có thể `DROP TABLE` thoải mái mà không lo ảnh hưởng người khác
- Mọi DB được tạo đều có Reviewer kiểm soát

---

### 5.3 CI/CD Self-Hosted Runner On-Demand

**Bối cảnh:**
Team đang dùng GitHub Actions runner cloud, chi phí tăng cao. Một số test cần truy cập tài nguyên nội bộ mà GitHub-hosted runner không thể kết nối tới.

**Cách áp dụng:**

```
Engineer viết code tạo VM runner
git push → HCP trigger → Reviewer duyệt → Agent tạo VM
        │
        ▼
Cloud-Init tự động:
  - Cài GitHub Actions Runner
  - Đăng ký runner với repository
  - Label: "self-hosted, proxmox, internal"
        │
        ▼
Job CI chạy trên runner nội bộ
  → Truy cập được DB nội bộ, API nội bộ
        │
        ▼
Hết nhu cầu → Engineer xóa resource trong code
            → git push → Reviewer duyệt → VM bị xóa
```

**So sánh chi phí:**

| Phương án | Chi phí | Truy cập nội bộ |
| :--- | :--- | :--- |
| GitHub-hosted runner | ~$8 / 1000 build-minutes | ❌ Không thể |
| Self-hosted VM chạy 24/7 | ~$15/tháng điện + phần cứng | ✅ Có |
| **Self-hosted On-Demand (mô hình này)** | **Chỉ tốn điện lúc cần** | **✅ Có** |

---

### 5.4 Staging Environment Clone Trước Mỗi Release

**Bối cảnh:**
Mỗi lần release, QA cần staging giống production nhất có thể. Staging dùng chung liên tục bị "ô nhiễm" dữ liệu từ các lần test trước.

**Cách áp dụng:**

```
Engineer commit code staging config
git push → HCP trigger → Reviewer duyệt → Agent tạo staging
              │
              ▼
Agent thực hiện:
┌─────────────────────────────────────────┐
│  1. Clone VM từ template staging-base   │
│  2. Restore DB snapshot từ production   │
│     (đã được ẩn danh hóa - anonymized) │
│  3. Cập nhật config trỏ về đúng service │
│  4. Deploy version mới lên VM           │
└─────────────────────────────────────────┘
              │
              ▼
Release xong → Engineer xóa resource trong code
             → git push → Reviewer duyệt → Giải phóng tài nguyên
```

**Ẩn danh hóa dữ liệu trong cloud-init script:**

```bash
mysql -u root -p"${db_password}" <<EOF
  UPDATE users SET email = CONCAT('user_', id, '@example.com');
  UPDATE users SET phone = '0000000000';
  UPDATE payments SET card_number = '****-****-****-0000';
EOF
```

---

### 5.5 Security Sandbox — Môi Trường Pentest Isolate

**Bối cảnh:**
Security team cần môi trường test vulnerability hoàn toàn isolate, bị xóa ngay sau khi pentest xong.

**Cách áp dụng:**

```
Engineer viết code VLAN isolate + VM
git push → HCP trigger → Reviewer duyệt → Agent tạo môi trường
              │
              ▼
Trong VLAN isolate:
┌────────────────────────────────────────────────┐
│  VM Target: Cài ứng dụng cần pentest           │
│  VM Attacker: Cài Kali Linux + công cụ         │
│  VM Monitor: Cài IDS/IPS ghi lại traffic       │
└────────────────────────────────────────────────┘
              │
              ▼
Security team truy cập qua Tailscale SSH
              │
              ▼
Pentest xong → Engineer xóa resource trong code
             → git push → Reviewer duyệt
             → Toàn bộ VLAN + VM bị xóa sạch
```

**Lợi ích đạt được:**

- Isolate hoàn toàn — exploit thành công cũng không ảnh hưởng production
- Tự động xóa sau khi xong — không lo để lại VM với lỗ hổng đang mở
- Log đầy đủ: Git history + HCP Runs + Tailscale audit — phục vụ compliance

---

### 5.6 Tóm Tắt Các Use Case

| Use Case | Vòng đời VM | Cách Engineer trigger | Approval Gate |
| :--- | :--- | :--- | :--- |
| Dev Environment On-Demand | 4–12 giờ | Thêm/xóa resource trong code, git push | DevOps Reviewer duyệt trên HCP UI |
| Isolated DB Per Branch | Suốt vòng đời PR | git push lên feature branch | DevOps Reviewer duyệt trên HCP UI |
| CI/CD Runner On-Demand | Theo nhu cầu | Thêm/xóa resource trong code, git push | DevOps Reviewer duyệt trên HCP UI |
| Staging Environment Clone | 3–7 ngày | Thêm/xóa resource trong code, git push | DevOps Reviewer duyệt trên HCP UI |
| Security Pentest Sandbox | 1–3 ngày | Thêm/xóa resource trong code, git push | DevOps Reviewer duyệt trên HCP UI |

**Điểm chung xuyên suốt:** Engineer **không bao giờ** dùng `terraform plan/apply` từ terminal. Mọi thay đổi hạ tầng đều đi qua `git push → HCP trigger → Reviewer apply`. Proxmox được bảo vệ hoàn toàn phía sau lớp Agent — Engineer không biết nó tồn tại.

---

### 6 Triển khai AWS ALB và Web Server
