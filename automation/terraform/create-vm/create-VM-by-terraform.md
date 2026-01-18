## Mục lục
- [1.Mục tiêu bài](#1mục-tiêu-bài)
- [2.Chuẩn bị hạ tầng](#2chuẩn-bị-hạ-tầng)
  - [2.1 Các lệnh terrform cơ bản](#21-các-lệnh-terrform-cơ-bản)
- [3.Tiến hành lab](#3tiến-hành-lab)
  - [3.1 Tạo VM trên AWS EC2](#31-tạo-vm-trên-aws-ec2)
  - [3.2 Tạo VM trên Proxmox](#32-tạo-vm-trên-proxmox)
    - [3.2.3 Triển khai](#323-triển-khai)
    - [3.2.4 Kết quả](#324-kết-quả)
- [4. Tổng kết](#4-tổng-kết)

---
## 1.Mục tiêu bài
- Tạo và xóa VM trên EC2 của AWS tự động.
- Kết nối đến proxmox, tạo templete và dùng nó để tạo ra VM trên proxmox.

---

## 2.Chuẩn bị hạ tầng
- Cài đặt Terraform
 - Ở đây hướng dẫn cài đặt qua home-manager của nixdeamon
  - Vào trang web `https://search.nixos.org/packages` rồi search từ khóa `terrform` sẽ thấy kết qur như trong ảnh 
    
    ![terrform-in-nix-search](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-in-nix.png)

  - Vào trong home.nix để thêm nó vào trong file như ảnh
        
    ![add-package-to-nix](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/add-package-to-nix.png)

  - Build lại và kiểm tra bằng lệnh `terrform -v`
        
    ![check-version-terraform](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/check-version-terraform.png)
    
- Cài đặt Vscode

    | Extension | Nhà phát triển | Chức năng |
    | :--- | :--- | :--- |
    | **HashiCorp Terraform** | HashiCorp | Syntax highlight, IntelliSense, format, validate, module explorer. |
    | **HashiCorp HCL** | HashiCorp | Syntax highlighting cho ngôn ngữ HCL (HashiCorp Configuration Language). |
    | **Terraform (tflint)** | terraform-linters | Lint code (tflint), phát hiện lỗi best practice. |
           
    ![hashicorp-terraform-ext](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/hashicorp-terraform-ext.png)
    
    ![hashiCorp-HCL-ext](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/hashiCorp-HCL-ext.png)

---

### 2.1 Các lệnh terrform cơ bản
- Các lệnh chính:
  - **terraform init:** Khởi tạo thư mục làm việc. Lệnh này sẽ tải xuống các Provider (như AWS, Azure, Google Cloud) và các module cần thiết. Đây là lệnh đầu tiên bạn phải chạy.
  - **terraform plan:** Tạo bản xem trước các thay đổi. Terraform sẽ so sánh cấu hình hiện tại của bạn với trạng thái thực tế và cho bạn biết những gì sẽ được thêm mới, sửa đổi hoặc xóa bỏ.
  - **terraform apply:** Thực thi các thay đổi. Sau khi kiểm tra plan, bạn chạy lệnh này để Terraform bắt đầu tạo hoặc cập nhật hạ tầng thực tế.
  - **terraform destroy:** Xóa toàn bộ hạ tầng đã được quản lý bởi cấu hình Terraform đó. Hãy cẩn thận khi dùng lệnh này!

- Các lệnh định dạng:
  - **terraform fmt**: Tự động định dạng lại mã nguồn theo chuẩn của HashiCorp (căn lề, khoảng cách...). Giúp code dễ đọc và đồng nhất.
  - **terraform validate**: Kiểm tra cú pháp của các file cấu hình xem có hợp lệ hay không mà không cần kết nối với Cloud.
  - **terraform show**: Hiển thị trạng thái hiện tại của hạ tầng (lấy từ file state) dưới dạng con người có thể đọc được.

- Các lệnh quản lý State và Output:
  - **terraform output**: Hiển thị các giá trị đầu ra mà bạn đã định nghĩa (ví dụ: địa chỉ IP của Server sau khi tạo xong).
  - **terraform state**: Dùng để quản lý file trạng thái (state file). Ví dụ: terraform state list để liệt kê các tài nguyên đang được Terraform quản lý.

---

## 3.Tiến hành lab
 - Để bắt đầu làm việc với `terrform` chúng ta cần chạy lênh sau:
    ```bash
      terrform init
    ```

    ![terraform-init](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-init.png)
    
- Lệnh này sẽ tải xuống các Provider (như AWS, Azure, Google Cloud) và các module cần thiết.

### 3.1 Tạo VM trên AWS EC2
 #### 3.1.1 Chuẩn bị hạ tầng
   - **Access Key & Secret Key:**: Giúp cho terraform có thể tương tác được với tài khoản AWS của bạn (cách tạo có thể xem
[tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/aws/install.md)).
   - **AMI ID:**: Đây là ID giúp terraform xác định kernal của máy ảo sẽ tạo ra. Chú ý là phải lấy đúng khu vực mà bạn
   đã khai báo trong file `.tf`. Bạn vào mục search gõ `AMI Catalog` rồi lựa chọn `AMI ID` như ảnh.

      ![search-ami-catalog](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/search-ami-catalog.png)

      ![ami-id](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/ami-id.png)

   - Tạo VPC và tạo Subnet để có thể tạo máy ảo trên đó. 
   
      ![create-VPC](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/create-VPC.png)

      ![create-subnet-in-vpc](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/create-subnet-in-vpc.png)

      ![subnet-id](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/subnet-id.png)

  #### 3.1.2 Tiến hành tạo VM
   - Tạo file `main.tf` tại thư mục vừa nãy chạy lệnh `terraform init` với nội dung như sau:
  
      ```bash
            terraform {
              required_providers {
                aws = {
                  source  = "hashicorp/aws"
                  version = "6.27.0"
                }
              }
            }

            provider "aws" {
              region     = "ap-southeast-1"
              access_key = "DIEN_ACCESS_KEY_CUA_BAN"
              secret_key = "DIEN_SECRET_KEY_CUA_BAN"
            }

            resource "aws_instance" "webserver" {
             # ID AMI đã lấy ở bước trên
              instance_type = "t3.micro"
              ami           = "ami-05f071c65e32875a8"
              
              # --- ĐIỀN Subnet ID CỦA BẠN VÀO ĐÂY ---
              subnet_id     = "subnet-0a0896cfe9aeae176" 
              
              tags = {
                Name = "HelloWorld-Terraform"
              }
            }
      ```

   - Chạy lệnh sau ở thử mục chứa file `main.tf`:

     ```bash
        terraform apply --auto-approve
     ```

        ![terrform-apply-1](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terrform-apply-1.png)
        
        ![terraform-apply-2](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-apply-2.png)

   - Kết tạo ra 1 instance trong EC2:

      ![result-terraform](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/result-terraform.png)

   - Để hủy VM đã tạo ra chúng ta chạy lệnh sau:

      ```bash
          terraform destroy
      ```
        
      ![terrform-apply-1](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-destroy-1.png)
                  
      ![terraform-destroy-2](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-destroy-2.png)

      ![terraform-destroy-3](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-destroy-3.png)

  - Như vậy là ta đã thấy instance của EC2 đã được hủy, nhưng ở đây vẫn còn VPC và Subnet chúng ta tự tạo ra bằng tay.
  
    ![vpc-instance](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/vpc-instance.png)

    ![subnet-vpc](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/subnet-vpc.png)

   - Chúng ta vẫn phải xóa chúng đi bằng tay, thế có cách nào tự động hết bằng terraform và cũng có thể ẩn đi 
   được các dữ liệu nhạy cảm là access_key và secret_key không ?
   - Câu trả lời là có, để làm được điều đó ta cập nhật file `main.tf` với nội dung sau:

      ```bash
        provider "aws" {
          region = "ap-southeast-1"
          # Lưu ý: Hãy dùng lệnh 'aws configure' để nhập key, đừng dán trực tiếp vào đây nữa nhé!
        }

        # 1. Tạo VPC
        resource "aws_vpc" "main_vpc" {
          cidr_block           = "10.0.0.0/16"
          enable_dns_hostnames = true
          tags = { Name = "my-custom-vpc" }
        }

        # 2. Tạo Subnet (Mạng con)
        resource "aws_subnet" "public_subnet" {
          vpc_id                  = aws_vpc.main_vpc.id
          cidr_block              = "10.0.1.0/24"
          map_public_ip_on_launch = true # Tự động cấp IP công cộng cho máy ảo
          availability_zone       = "ap-southeast-1a"
          tags = { Name = "my-public-subnet" }
        }

        # 3. Tạo Internet Gateway (Cổng ra internet)
        resource "aws_internet_gateway" "igw" {
          vpc_id = aws_vpc.main_vpc.id
          tags   = { Name = "my-igw" }
        }

        # 4. Tạo Route Table (Bảng định tuyến)
        resource "aws_route_table" "public_rt" {
          vpc_id = aws_vpc.main_vpc.id

          route {
            cidr_block = "0.0.0.0/0"
            gateway_id = aws_internet_gateway.igw.id
          }
        }

        # 5. Liên kết Route Table với Subnet
        resource "aws_route_table_association" "public_assoc" {
          subnet_id      = aws_subnet.public_subnet.id
          route_table_id = aws_route_table.public_rt.id
        }

        # 6. Tìm AMI Amazon Linux 2023 mới nhất (Tự động theo vùng)
        data "aws_ami" "amazon_linux" {
          most_recent = true
          owners      = ["amazon"]
          filter {
            name   = "name"
            values = ["al2023-ami-2023*-x86_64"]
          }
        }

        # 7. Tạo máy ảo EC2 trong Subnet vừa tạo
        resource "aws_instance" "webserver" {
          ami           = data.aws_ami.amazon_linux.id
          instance_type = "t3.micro"
          
          # Chỉ định đúng Subnet đã tạo ở trên
          subnet_id     = aws_subnet.public_subnet.id

          tags = {
            Name = "Webserver-in-Custom-VPC"
          }
        } 
      ```

   - Và sử dụng AWS CLI để cấu hình access_key và secret_key bằng lệnh `aws configure`
    
- Ta thấy tự động tạo VPC, Subnet, rồi tiếp tục tạo instance. Lên AWS console để kiểm chứng
  
  ![terrform-auto-apply](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terrform-auto-apply.png)

  ![terrform-create-vpc](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terrform-create-vpc.png)


  ![terraform-creat-subnet](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-creat-subnet.png)

- Tương tự để hủy thì sẽ chạy lệnh
   
   ```bash
      terraform destroy
   ```

   ![terraform-auto-destroy](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/terraform-auto-destroy.png)

### 3.2 Tạo VM trên Proxmox
   #### 3.2.1. Chuẩn bị
   - Tạo templete từ máy ảo 
   - Tạo API token (API phải có quyền `PVEDatastoreAdmin` trên `/storage/local-lvm` giúp ghi vào ổ nhớ)
   - Lấy các thông số sau:
    - Endpoint: URL của Proxmox (ví dụ: `https://172.199.10.115:8006`).
    - Node name: Tên node vật lý (ví dụ: `node01`).
    - Pool: Resource Pool (ví dụ: `vuthong-pool`).
    - Source VM ID: ID của Template bạn vừa tạo (ví dụ: `101`).
    
     ![info](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/info.png)

   - Nội dung file cấu hình `main.tf`:
      ```bash
          terraform {
            required_providers {
              proxmox = {
                source = "bpg/proxmox"
                version = ">=0.80.0"
              }
            }
          }

          provider "proxmox" {
            # Thay IP Proxmox của bạn
            endpoint = "https://192.168.39.229:8006/" 
            
            # Token format: USER@REALM!TOKEN_ID=UUID
            api_token = "root@pam!create-vm=9d6a470d-0f22-42c1-8591-987cb9c57a65"
            
            insecure = true # Bỏ qua check SSL
          }

          resource "proxmox_virtual_environment_vm" "vm" {
            name      = "thongvu-ubuntu-lab"
            node_name = "node01"
            pool_id   = "vuthong-pool"
            
            # Tự động bật máy ngay sau khi tạo xong
            started = true

            clone {
              vm_id = 101 # ID của Template
              full  = true
            }

            cpu {
              cores = 2
              type  = "host" # Tối ưu hiệu năng CPU
            }

            memory {
              dedicated = 2048
            }

            network_device {
              bridge = "vmbr0"
            }

            # Tắt Agent check để tránh timeout lúc khởi tạo
            agent {
              enabled = false
            }

            initialization {
              ip_config {
                ipv4 {
                  address = "dhcp"
                }
              }
            }
          }
      ```
#### 3.2.3 Triển khai
  - Khởi tạo terrform bằng lệnh
  
    ```bash
    terrform init
    ```

  - Tạo VM bằng terrform chạy lệnh sau

    ```bash
    terraform apply --auto-approve
    ```

#### 3.2.4 Kết quả
   - Kiểm tra trên promox 

     ![create-vm-promox](https://github.com/ThongVu1996/documents/tree/main/automation/create-vm/create-vm-promox.png)

---

## 4. Tổng kết
- Như vậy ở bài này chúng ta có thể tạo được VM trên AWS và promox hoàn toàn tự động bằng code. Đây là một bài tập cơ bản giúp chúng ta hình dung được cách terrform hoạt động và làm quen với các lệnh cơ bản trong terraform.
