# Triển Khai Hạ Tầng Hybrid (AWS & Proxmox) với HCP Terraform

--- 

## 1.Yêu cầu 
### Kịch bản hệ thống 
- On-Premise (Proxmox): Cấu hình và triển khai Database (MySQL/PostgreSQL) nhằm tối ưu chi phí, đảm bảo kiểm soát và bảo mật dữ liệu ở mức cao nhất.
- Public Cloud (AWS): Triển khai cụm Web Server đằng sau Application Load Balancer (ALB) để đón lưu lượng người dùng và đảm bảo tính sẵn sàng cao.
- Quản lý Trạng thái (State Management): Sử dụng HCP Terraform (Terraform Cloud) làm trung tâm điều phối chung. Quản lý State file, bảo mật biến số (Variables) và thực hiện quy trình phê duyệt (Approval) trước khi cấp phát hạ tầng.
### Cách triển khai
- Thiết lập Organization và Workspace chung trên HCP Terraform. Tích hợp VCS (GitHub/GitLab) để tự động hóa quy trình.
- Cấu hình block `terraform { cloud { ... } }`. Bắt buộc cài đặt `Execution Mode` ở trạng thái Local để máy cá nhân có thể gọi API tới Proxmox nội bộ.
- Tuyệt đối không hardcode mật khẩu trong file code. Khóa API (AWS Access Key, Proxmox Token) phải được mã hóa dạng Sensitive Variables trên giao diện Terraform Cloud.
- Áp dụng tư duy Modularization: Chia cấu trúc thư mục rành mạch thành `modules/aws`, `modules/proxmox`.
- Hệ thống phải xuất ra (Output) địa chỉ DNS của AWS ALB và địa chỉ IP nội bộ của VM chứa Database.

--- 

## 2.Chuẩn bị hạ tầng
### 2.1.Proxmox
#### 2.1.1 Yêu cầu chung
- Yêu cầu cần có 1 promox.
- Cài đặt sẵn 1 templete là `ubuntu orginal` như hình.
    ![ubuntu orginal templete](./ubuntu-original.png) 
- Một template có sẵn `cloud-init` để có thể cấp được ip tĩnh, không để câp động bằng dhcp.
- Một VM để làm agent của HCP cloud.
- Cài đặt để `cloud init` có thể lưu file cài đặt vào promox (Datacenter -> Storage -> chọn nơi lưu trữ (ở đây là local) -> Edit)
![setting local storage](./setting-snipset-storage.png)
#### 2.1.2 Cài đăt Ubuntu template
  - Tải bản ubuntu iso về promox, làm như trong hình vẽ, ở phần URL bạn nhập link tải của iso (e.g `https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso`)
    ![download ubuntu iso](./download-iso-ubuntu.png) 
  - Hoặc có thể sử CLI như sau 
    ```bash 
        cd /var/lib/vz/template/iso
        wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
    ```

  - Tạo máy ảo bằng cách như hình
   ![create vm](./create-vm.png) 
  - Tiến hành cài ubuntu lên máy ảo tại theo hướng dẫn [tại đây](https://www.youtube.com/watch?v=DjlGte968ko&list=PLsvroIvFNP1KU8foUeCC-hbJbqnAggWL2&index=4) 
  - Tạo template giống ảnh.
    ![create template](./create-template.png) 
#### 2.1.3 Tiến hành tạo ubuntu template có sẵn cloud-init
  - Chạy 1 máy ảo từ ubuntu template orginal
  - Tiến hành kiểm tra xem nó đã có cloud-init chưa bằng cách lệnh sau
    ```bash
        cat /etc/machine-id
        ls -la /var/lib/cloud/
    ```
    ![check cloud-init](./check-clould-init.png) 
   - Cài đặt các gói cần thiết.
      ```bash
          sudo apt update
          sudo apt install qemu-guest-agent cloud-init -y
          sudo systemctl enable --now emu-guest-agent
      ```
   - Dừng các dịch vụ không cần thiết
      ```bash
          # Dừng và tắt dịch vụ tự cập nhật
            sudo systemctl stop unattended-upgrades
            sudo systemctl disable unattended-upgrades

            # Cấu hình để Cloud-init KHÔNG tự update khi mới boot máy
            sudo sed -i 's/package_upgrade: true/package_upgrade: false/' /etc/cloud/cloud.cfg
            # Xóa toàn bộ file cấu hình mạng cũ (để Cloud-init tự tạo file mới)
            sudo rm -f /etc/netplan/*.yaml

            # Cho phép SSH bằng mật khẩu (nếu bạn không dùng SSH Key)
            sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
            sudo systemctl restart ssh
      ```
   - Enable `qemu-guest-agent`
   ![enable qemu-guest-agent](./enable-qemu-guest-agent.png) 
  - Xóa các thông số cloud-init cũ.
    ```bash
          # 1. Xóa các file nhật ký để giảm dung lượng
          sudo apt clean
          sudo rm -rf /var/log/*

          # 2. Xóa định danh máy (để mỗi máy con sinh ra sẽ có ID mạng riêng)
          sudo truncate -s 0 /etc/machine-id
          sudo rm -f /var/lib/dbus/machine-id

          # 3. Xóa dấu vết của Cloud-init cũ
          sudo cloud-init clean --logs

          # 4. Tắt máy ngay lập tức (KHÔNG ĐƯỢC BẬT LẠI)
          sudo poweroff
    ``` 
  - Tiến hành chuyển chuyển VM thành template.
  - Thực hiện thêm cloud-init cho template như hình (có thể tiến hành xóa CD/DVD driver để cloud init dùng ide2)
  ![add cloud-init 1](./add-clould-init-1.png) 
  ![add-clould-init-2](./add-clould-init-2.png) 
  ![add-clould-init-3](./add-clould-init-3.png) 
  ![add new cloud drive to ide2](./add-new-clould-drive-to-ide2.png)
  - Sau khi làm xong thì ta có thể chỉ định được ip cho VM mà không bị ip ngẫu nhiên do dhcp cấp nữa.
### 2.2 HCP cloud
- Đây là nơi lưu trữ các biến dành cho terraform, quản lý lịch sử triển khai.
- Ta sẽ tạo theo flow sau `Organization -> Project -> Agent Pool -> Workspace`.
- Các thành phần cơ bản như sau:
  
  | Thành phần | Cấp độ | Mục đích |
  | :--- | :--- | :--- |
  | **Organization** | Gốc (Root) | Quản lý thành viên, cài đặt chung, hóa đơn. |
  | **Project** | Con của Org | Phân nhóm các Workspace theo phòng ban hoặc dự án. |
  | **Workspace** | Con của Project | Quản lý State, thực thi code Terraform cụ thể. |
  | **Agent Pool** | Thuộc Org | Cung cấp hạ tầng để thực thi lệnh (Execution Environment). |

#### 2.2.1 Sử dụng UI
 - Tạo mới organizations
 ![create-orgranization](./creat-orgranization.png) 
 - Tiến hành tạo group tại [link này](https://app.terraform.io/app/organizations/new), nhớ chọn `Bussiness` để có thêm đồng nghiệp vào làm cùng.
 ![create HCP group](./create-HCP-group.png) 
 - Tạo mới work space 
 ![creat-HCP-workspace](./creat-HCP-workspace.png)
 - Tạo variables cho workspace ta làm theo trong ảnh
 ![creat variables workspace hcp](./create-variable-workspace-HCP.png) 
 ![create-variable-workspace-HCP 2](./create-variable-workspace-HCP-2.png)
 ![create-variable-workspace-HCP 3](./create-variable-workspace-HCP-3.png) 
 ![create-variable-workspace-HCP 4](./create-variable-workspace-HCP-4.png) 
 - Lưu ý có 2 options là HCP (nó sẽ coi các biến truyền vào của value theo ngôn ngữ của HCP),
  còn với Sensitive (nó sẽ coi giá trị là chuỗi và sẽ ẩn đi ở list variable, đồng thời cũng sẽ ẩn log khi terraform tiến hành tạo khi chạy ở mục `Runs`).
 - Ta được kết quả như hình:
 ![create-variable-workspace-HCP 5](./create-variable-workspace-HCP-5.png) 
 ![create-variable-workspace-HCP 6](./create-variable-workspace-HCP-6.png) 
 - Quan sát ta sẽ thấy các biến dược tích `Sensitive`, sẽ bị ẩn giá trị đi, dù khi ấn edit,
 nên dù có edit ta cũng chỉ có thể là tự thêm giá trị vào, chứ không xem được giá trị cũ, làm tăng cường bảo mật.
 - Tạo mới agent bằng vào đúng `organizations -> setting -> agent` như hình:
 ![create agent](./create-agent.png)
 - Kết quả kiểm tra trên UI như hình:
 ![create agent-pools](./creat-agent-pool.png) 
#### 2.2.2 Sử dụng API
  - `TFC_GROUP`: chính là tên group bạn tạo 
  - `TFC_TOKEN`: Chính là token bạn tạo ra để có thể tương tác với HCP bằng API, tạo ra bằng cách vào
  - `Account setting` như trong ảnh để lấy api token
   ![api token](./api-token.png) 

    | Tài nguyên | Method | Endpoint URL |
    | :--- | :--- | :--- |
    | **Organization** | `POST` | `/api/v2/organizations` |
    | **Agent Pool** | `POST` | `/api/v2/organizations/{org_name}/agent-pools` |
    | **Project** | `POST` | `/api/v2/organizations/{org_name}/projects` |
    | **Workspace** | `POST` | `/api/v2/organizations/{org_name}/workspaces` |
  - Thêm biến môi trường vào terminal  
    ```bash
       export ORG_NAME="tonytechlab-group-1"
       export ORG_EMAIL="your-email@example.com"
       export TFC_TOKEN="token"
    ```
 - Organization

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

 - Project
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
    Kết quả: Hãy tìm dòng "id": "prj-xxxxx" trong phản hồi JSON để dùng cho bước sau.
    ![project api](./project-api.png)
 - Agent
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
    Kết quả: Hãy tìm dòng "id": "apool-xxxxx" trong phản hồi JSON để dùng cho bước sau.
    ![agent api](./agent-api.png)

  - Workspace
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
                "description": "Workspace tao tu dong cho DB Team"
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
#### 2.2.3 Sử dụng terraform cloud
- Sử dụng terraform cloud để tạo các resource
- Thêm token vào terminal
  ```bash
    export TF_VAR_tfc_token="your-tfc-token"
  ```
- Tạo file `variables.tf`
  ```hcl
      variable "tfc_token" {
        description = "User API Token lấy từ môi trường terminal"
        type        = string
        sensitive   = true
      }

      variable "org_name" {
        type    = string
        default = "tonytechlab-enterprise-2026"
      }

      variable "org_email" {
        type    = string
        default = "admin@tonytechlab.com"
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
- Tạo file `providers.tf`
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
- Tạo file `main.tf`
  ```hcl
    resource "tfe_organization" "org" {
      name  = var.org_name
      email = var.org_email
    }

    resource "tfe_agent_pool" "pool" {
      name                = var.agent_pool_name
      organization        = tfe_organization.org.name
      organization_scoped = true
    }

    resource "tfe_project" "project" {
      name         = var.project_name
      organization = tfe_organization.org.name
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
      description   = "Token cho VM-101-Proxmox"
    }

    output "hcp_agent_token" {
      value     = tfe_agent_token.token.token
      sensitive = true
    }
  ```
- Có thể lấy code ở tại [tại đây](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-hcl-cloud)
- Tiến hành chạy
  ```bash
    terraform init
    terraform fmt
    terraform validate
    terraform plan
    terraform apply
  ```
  ![terraform cloud](./terraform-cloud.png)
- Sử dụng lệnh để xem token của agent
  ```bash
      terraform output hcp_agent_token
  ```
    ![hcp agent token](./hcp-agent-token.png)
- Kiểm tra trên UI để thấy rằng ta đã có thể dùng agent để có thể tạo được VM
  ![Excution Mode](./excution-mode.png)
### 2.3 Sử dụng terraform agent
- Trên máy ảo VM dùng làm agent trên proxmox, ta sẽ thêm agent vào bằng cách chạy docker
  ```bash
    export TFC_AGENT_TOKEN=your-token
    export TFC_AGENT_NAME=your-agent-name
    docker run --platform=linux/amd64 -e TFC_AGENT_TOKEN -e TFC_AGENT_NAME hashicorp/tfc-agent:latest
  ```
- 
- Chúng ta có thể đọc [tại đây](https://developer.hashicorp.com/terraform/tutorials/cloud/cloud-agents) để hiểu rõ hơn về cách dùng agent giúp bảo mật dữ liệu trên HCP cloud 

---

## 3.Triển khai
### 3.1 Tạo VM trên Promox bằng terraform
- Cùng nhau phân tích bài toán trước khi tiến hành triển khai nhé
- Chúng ta sẽ đi theo flow sau
 - Ở dưới máy local ta cài đặt terraform, rồi dùng nó để login vào HCP cloud (sử dụng token của HCP cloud)
 - Sau đó ta sẽ chạy lệnh `terraform apply`, terraform ở máy local sẽ gọi lên HCP cloud, HCP cloud nhận lênh,
 rồi tiếp đến nó gọi đến các máy ảo đang chạy agent ở dưới promox, cái nào đang rảnh thì sẽ nhận lệnh từ HCP cloud
 - Agent trên máy ảo VM của promox sẽ nhận lệnh, rồi tiến hành tạo ra 1 VM đúng yêu cầu và cài đặt db lên trên đó.
#### 3.1.1 Sử dụng provisioner
- Lấy code [tại đây](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-vm)
- Kết nối terraform với HCP cloud:
  ```bash
    terraform login
  ```
- Tiến hành chạy các câu lệnh:
  ```bash
    terraform init
    terraform fmt
    terraform validate
    terraform plan
    terraform apply
  ```
#### 3.1.2 Kiểm tra kết quả
- Thông báo thành công trên terminal:
  ![creat VM success](./creat-VM-success.png)
- Kiểm tra trên UI HCP cloud:
  ![create VM success hcp cloud](./create-VM-success-hcp.png)
- Kiểm tra trên UI Proxmox: ta thấy tên và ip được tạo ra đúng
  ![create VM success proxmox](./create-VM-success-proxmox.png)
  - SSH vàg trong máy ảo để kiểm tra, ta thấy mysql đã được cài, db đã được tạo thành công
  ![check mysql in VM](./check-mysql-vm.png)
  - Kiểm tra tailscale: 
  ![tailscale-provisioner-1](./tailscale-provisioner-1.png)
  ![tailscale-provisioner-2](./tailscale-provisioner-2.png)
#### 3.1.3 Phân tích code 
- Trong docs của terraform họ cũng nói rằng chúng ta không nên sử dụng `provisioner`
- 1 số nhược điểm như trong ảnh
![nhược điểm của provisioner](./disadvance-provisioner.png)
- Tham khảo thêm [tại đây](https://oneuptime.com/blog/post/2026-02-23-terraform-provisioners-last-resort/view) khi so sánh `provisioner` với và `cloud init`
#### 3.1.4 Sử dụng cloud init
- Trong khi sử dụng cloud init này, chúng ta sẽ có sử dụng 2 provider là `telmate/proxmox`, hoặc `bpg/proxmox`. Thì ở đây chúng ta sẽ sử dụng cả 2 để thấy ưu nhược điểm của từng cái.
#### 3.1.5 telmate/proxmox
- Lấy code [tại đây](https://github.com/ThongVu1996/terraform-hybrid-lab/tree/main/create-vm-with-cloud-init)
- Với provider này chúng ta phải thao tác bằng tay để tạo file cấu hình nhằm tạo db và tailscale tại thư mục `/var/lib/vz/snippet`
  ```bash
      #cloud-config
      write_files:
        - path: /root/setup_nodes.sh
          permissions: '0755'
          content: |
            #!/bin/bash
            set -e # Dừng nếu có lỗi
            
            echo "=== [1/3] Dọn dẹp APT & Cài đặt MySQL ==="
            systemctl stop unattended-upgrades || true
            killall apt apt-get || true
            rm -f /var/lib/apt/lists/lock /var/cache/apt/archives/lock /var/lib/dpkg/lock*
            
            sed -i "s/[a-z]*.archive.ubuntu.com/vn.archive.ubuntu.com/g" /etc/apt/sources.list
            apt-get update -y
            apt-get install -y mysql-server curl
            
            echo "=== [2/3] Cài đặt & Kích hoạt Tailscale ==="
#### 3.1.6 bpg/proxmox (Giải pháp tối ưu và hiện đại nhất)

Đây là Provider thế hệ mới (hiện đại hơn `telmate`), cho phép chúng ta tự động hóa 100% việc tạo Snippet mà không cần can thiệp thủ công vào Proxmox host.

**Sức mạnh của bpg/proxmox so với Telmate:**
- **Variable Injection:** Bạn có thể truyền biến (như mật khẩu DB, Tailscale Auth Key) từ HCP Cloud TRỰC TIẾP vào nội dung file YAML.
- **Quản lý tài nguyên chuyên nghiệp:** Coi Snippet là một Resource có ID, giúp quản lý vòng đời (Lifecycle) dễ dàng.
- **Tự động hóa hoàn toàn:** Agent tự động "đẩy" file lên Proxmox qua SSH trước khi tạo máy ảo.

Để triển khai, chúng ta thực hiện theo 3 bước chuẩn "Expert":

##### Bước 1: Thiết lập "Người thợ xây" (terraform-user) trên Proxmox Host
Thay vì dùng `root` (vốn rất nguy hiểm nếu lộ Key), chúng ta tạo một User riêng có quyền hạn hạn chế:

1. **Tạo user hệ thống:**
   ```bash
   # Tạo user với thư mục home và shell mặc định là bash
   sudo useradd -m -s /bin/bash terraform-user
   ```

2. **Cấu hình "Cửa hậu" SSH cho Agent:**
   Chúng ta cần dán **Public Key** của Agent (hoặc của bạn) vào đây để Agent có thể SSH vào Proxmox mà không cần mật khẩu.
   ```bash
   sudo mkdir -p /home/terraform-user/.ssh
   sudo chmod 700 /home/terraform-user/.ssh
   # Dán Public Key vào file authorized_keys
   sudo nano /home/terraform-user/.ssh/authorized_keys
   sudo chown -R terraform-user:terraform-user /home/terraform-user/.ssh
   ```

3. **Thiết lập quyền "Hộp cát" (Sandbox) cho thư mục Snippets:**
   Đây là phần tinh tế nhất về bảo mật. Ta dùng **Sticky Bit (`1`)** để đảm bảo: *Mọi người có quyền tạo file, nhưng CHỈ CHỦ SỞ HỮU (Agent) mới có quyền Xóa/Sửa file mình đã tạo.*
   ```bash
   # Trả lại quyền sở hữu cho root và cấp quyền đặc biệt 1777
   sudo chown root:root /var/lib/vz/snippets
   sudo chmod 1777 /var/lib/vz/snippets
   ```


##### 🏁 Kết quả & Cách truy cập (Tailscale SSH)
Sau khi `apply` thành công, máy ảo của bạn đã nằm trong mạng VPN. Developer hoàn toàn "mù tịt" về mật khẩu hay SSH Key hạ tầng, họ chỉ cần:
1. Cài Tailscale trên máy Dev.
2. Gõ lệnh: `tailscale ssh username@db-vm`.
3. Đăng nhập qua trình duyệt (SSO) để xác thực ➔ Vào thẳng máy ảo.

- Với quy trình này, bạn đã biến Proxmox thành một Private Cloud chuyên nghiệp không thua kém gì AWS hay Azure.
đọc các biến môi trường từ HCP và bản thân provider `bpg/promox` ra sau và được update thường xuyên hơn `telmate/proxmox`
- Chúng ta sử dụng agent để nhằm ssh vào promox, nên ta cần tạo 1 use riêng biệt và tạo quyền `1777` để có thể tương tác với promox api và không sử dụng user root.
  ```bash
      # Tạo user với thư mục home và shell mặc định là bash
      sudo useradd -m -s /bin/bash terraform-user
      # Kiểm tra lại một lần nữa bằng lệnh id
      id terraform-user
  ```
  ```bash
      # 1. Tạo thư mục SSH
      sudo mkdir -p /home/terraform-user/.ssh
      sudo chmod 700 /home/terraform-user/.ssh
      # 2. Gán quyền sở hữu cho đúng người
      sudo chown -R terraform-user:terraform-user /home/terraform-user/
      # 3. Mở file để dán Public Key được tạo ở trên
      nano /home/terraform-user/.ssh/authorized_keys
  ```
  ```bash
      # 1. Trả lại quyền sở hữu thư mục cho root (để root làm chủ quản)
      sudo chown root:root /var/lib/vz/snippets
      # 2. Cấp quyền: Chủ (root) toàn quyền, Nhóm (ví dụ group terraform) quyền ghi
      # 777: Cho phép mọi người (bao gồm terraform-user) có quyền Đọc, Ghi, Xóa trong thư mục này. (Nghe có vẻ nguy hiểm nhưng hãy xem tiếp).
      # Số 1 ở đầu (Sticky Bit): Đây là phần cực kỳ quan trọng. Khi có số 1 này, Linux áp dụng quy tắc: "Dù ai cũng có quyền ghi vào đây, nhưng CHỈ CÓ NGƯỜI TẠO RA FILE mới có quyền Xóa hoặc Sửa cái file đó".
      sudo chmod 1777 /var/lib/vz/snippets
  ```
- Ta có thể thấy đoạn code quy định điêu đó trong `provider.tf`
  ```bash
    ssh {
      agent       = false
      username    = "terraform-user" # Thường là root để có quyền ghi vào /var/lib/vz/
      private_key = var.proxmox_ssh_private_key
    }
  ```
- 

