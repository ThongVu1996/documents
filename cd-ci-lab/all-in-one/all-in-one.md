# <center> CI/CD all in one với docker host </center>

## 🎯 Mục tiêu bài Lab

- Bài lab này hướng dẫn chi tiết cách xây dựng luồng CI/CD "Cách 1: All-in-One" hoàn chỉnh, deploy ứng dụng (frontend + backend) lên Docker host. Lab này bao gồm cấu hình mạng MACVLAN bền vững, DNS tập trung, Reverse Proxy (NPM) với SSL, và quy trình build/deploy tự động có bước phê duyệt thủ công.

## Sơ đồ Luồng hoạt động

```
                👨‍💻 Developer push code mới lên GitLab (nhánh `nodejs`).

                                    ⬇️

                📡 GitLab tự động gửi Webhook cho **Jenkins**.

                                    ⬇️

                👷‍♂️ Jenkins nhận code, build **Docker images** (frontend & backend).

                                    ⬇️

                📦 Jenkins push images lên GitLab Registry.

                                    ⬇️

                ⏸️Jenkins dừng lại, chờ phê duyệt từ CTO.

                                    ⬇️

                🧑‍💼 CTO đăng nhập Jenkins và nhấn "Approve".

                                    ⬇️

                🚀 Jenkins chạy `kubectl apply` để deploy lên Kubernetes.
```

## 📂 Cấu trúc Thư mục Chuẩn bị

```
/opt/devcom/
├── docker-compose.yml     # (File 1: Xem Bước 4 bên dưới)
│
├── jenkins/
│   └── Dockerfile         # (File 2: Xem Bước 3 bên dưới)
│
├── gitlab/                # (Docker sẽ tự tạo khi chạy lần đầu)
│   ├── config/            # Chứa file gitlab.rb quan trọng!
│   ├── data/
│   └── logs/
│
├── npm/                   # (Docker sẽ tự tạo)
├── runner-config/         # (Docker sẽ tự tạo)
└── technitium-config/     # (Docker sẽ tự tạo)
```

### 🤔 Tại sao lại chọn `/opt/devcom/` ?

- **Quy ước phổ biến (Common Convention)**: Thư mục `/opt` (optional) trong Linux thường dùng cho các ứng dụng "bên thứ ba" hoặc đóng gói. Đặt toàn bộ dự án Docker Compose vào đây giúp **tách biệt (isolate)** nó khỏi hệ điều hành chính.

- **Tập trung & Dễ quản lý (Centralized & Manageable)**: Giữ tất cả các file cấu hình (IaC - Infrastructure as Code) và thư mục dữ liệu (volumes) tại cùng một nơi giúp bạn dễ dàng tìm kiếm, sao lưu, hoặc di chuyển khi cần.

- **Các lựa chọn "chuẩn" FHS khác (Advanced FHS Alternatives)**: `/srv/devcom/` (dùng cho "service data") hoặc tách bạch (config ở `/etc`, data ở `/var/lib`).

#### ⚠️ Tránh dùng (Avoid)

- Không nên tạo thư mục tùy tiện ở cấp gốc như `/projects/` vì nó không theo chuẩn FHS và làm lộn xộn cấu trúc thư mục gốc của hệ điều hành.

### Kết luận (Conclusion)

- Sử dụng `/opt/devcom/` là một lựa chọn tốt, cân bằng giữa tính đơn giản, quy ước phổ biến và dễ quản lý cho bài lab này.

## Giai đoạn 0: Cài đặt Hạ tầng (The Foundation) 🛠️

Đây là giai đoạn **quan trọng nhất (để cấu hình Network adapter ở dạng bridged(Autodetect)**, bao gồm cài đặt Docker, cấu hình mạng vật lý ảo (MACVLAN hoặc IPVLAN), và cấu hình mạng vĩnh viễn cho máy chủ host.

### Bước 1: Cài đặt Docker Engine (Điều kiện tiên quyết)

1. Cài đặt các gói phụ thuộc:

   ```bash
   sudo apt-get update
   sudo apt-get install -y ca-certificates curl gnupg lsb-release
   ```

2. Thêm GPG Key chính thức của Docker:

   ```bash
   sudo mkdir -m 0755 -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   ```

3. Thêm kho lưu trữ (Repository) của Docker:

   ```bash
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list> /dev/null
   ```

4. Cài đặt Docker Engine (Bản đầy đủ):

   Lệnh này sẽ cài đặt Docker daemon (`docker.service`), client (`docker`), và các thành phần cần thiết khác.

   ```bash
   sudo apt-get update
   sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose
   ```

5. Kiểm tra dịch vụ Docker:

   ```bash
   sudo systemctl status docker
   ```

   Bạn sẽ thấy trạng thái `active (running)`. Giờ đây máy chủ Host đã sẵn sàng.

6. (Tùy chọn) Thêm User vào nhóm `docker`:

   Theo mặc định, chỉ user `root` mới có quyền chạy lệnh `docker`. Để cho phép user hiện tại của bạn (ví dụ: `thong`) chạy `docker` mà không cần `sudo`, hãy thêm user đó vào nhóm `docker`.

   ```bash
   sudo usermod -aG docker ${USER}
   ```

   **QUAN TRỌNG:** Sau khi chạy lệnh này, bạn phải **đăng xuất (log out)** khỏi máy chủ và **đăng nhập (log in)** trở lại để thay đổi có hiệu lực.

### Bước 2: Chuẩn bị Mạng Nền (IPVLAN hoặc MACVLAN)

- Để khắc phục lỗi xung đột giữa `systemd-networkd` và `cloud-init` (nguyên nhân gây mất IP tĩnh), chúng ta cần vô hiệu hóa `cloud-init` trước khi cấu hình mạng.

  ```bash
  # 1. Tạo thư mục cấu hình cloud-init
  sudo mkdir -p /etc/cloud/cloud.cfg.d

  # 2. Tạo file cấu hình.
  echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-networking.cfg
  ```

- **Giải thích:**
  - `network: {config: disabled}` báo cho `cloud-init` biết rằng nó **không được phép** quản lý hoặc tạo ra bất kỳ cấu hình mạng nào (`/etc/netplan/*.yaml`). Đây là bước bắt buộc để đảm bảo cấu hình `systemd-networkd` của bạn không bao giờ bị ghi đè sau khi reboot.
  - Trong linux thì có 2 cách quản lý network là NetworkManager và systemd-networkd. Cả 2 đều hoạt động ở phía background (dưới dạng deamon)
  - Còn với netplan, nó dùng file `yaml` để cấu hình, nó dùng ở phía trên, khi chạy nó sẽ apply cái cấu hình đó xuống bên dưới. Ở dưới NetworkManager hay systemd-networkd sẽ tự động sinh file tương ứng để phù hợp với cấu hình đó.

     ```bash
     # Xóa các file Netplan cũ (50-cloud-init.yaml)
     sudo rm -f /etc/netplan/*.yaml

     # Đảm bảo NetworkManager đã disabled nếu systemd-networkd được sử dụng
     sudo systemctl stop NetworkManager || true
     sudo systemctl disable NetworkManager || true

      # Kích hoạt và khởi động lại systemd-networkd để bắt đầu quản lý mạng
      sudo systemctl enable systemd-networkd
      sudo systemctl restart systemd-networkd
      ```

---

- #### Bước 2.1: Đối với dùng IPVLAN (Sử dụng Systemd-networkd)

  Sử dụng `systemd-networkd` để cấu hình Host IPVLAN.(Thay `ens160` bằng tên card mạng vật lý của bạn)

  1. Tạo file `05-ens160.network` (Biến card vật lý thành "cha", dùng `ip a` để kiểm tra):


      ```bash
      sudo nano /etc/systemd/network/05-ens160.network
      ```
      
      ```bash
      [Match]
      Name=ens160
      
      [Network]
      IPVLAN=ipvhost0
      ```

      **[Giải thích File 05-ens160.network]**
      
      - **[Match]**: Định danh card mạng vật lý (parent).
      - **Name=ens160**: Tên card mạng vật lý đang được sử dụng (ví dụ: \`ens160\` hoặc \`eth0\`).
      - **[Network]**: Khối định nghĩa cách thức hoạt động của mạng.
      - **IPVLAN=ipvhost0**: Chỉ định rằng interface ảo có tên \`ipvhost0\` (được định nghĩa trong file \`.netdev\` tiếp theo) sẽ được gắn (attach) vào card mạng vật lý này. Card vật lý giờ đây chỉ là cầu nối (trunk).
  
  2. Tạo file `10-ipvhost0.netdev` (Định nghĩa interface ảo cho Host):

      ```bash
      sudo nano /etc/systemd/network/10-ipvhost0.netdev
      ```
      
      ```ini
      [NetDev]
      Name=ipvhost0
      Kind=ipvlan
      
      [IPVLAN]
      Mode=L2
      ```
  
      **[Giải thích File 10-ipvhost0.netdev]**
      
      - **[NetDev]**: Khối định nghĩa một interface mạng mới.
      - **Name=ipvhost0**: Tên logic của interface ảo.
      - **Kind=ipvlan**: Loại interface là IPVLAN.
      - **[IPVLAN]**: Khối cấu hình riêng cho IPVLAN.
      - **Mode=L2**: Chế độ IPVLAN Layer 2. Đây là chế độ khuyến nghị cho VM, cho phép Host và Container giao tiếp trực tiếp trên cùng một IP subnet.

  3.  Tạo file `10-ipvhost0.network` (Gán IP tĩnh Host và Fix ổn định):
    
      ```bash
      sudo nano /etc/systemd/network/10-ipvhost0.network
      ```
      
      ```bash
      [Match]
      Name=ipvhost0
      
      [Network]
      Address=192.168.1.61/24
      Gateway=192.168.1.1
      DNS=192.168.1.30
      DNS=8.8.8.8
      ```
  
      **[Giải thích File 10-ipvhost0.network]**
      
      - **[Match]**: Áp dụng cấu hình này cho interface ảo \`ipvhost0\`.
      - **[Network]**: Cấu hình mạng.
      - **Address=192.168.1.61/24**: Gán IP tĩnh cho Host. Đây là IP bạn sẽ dùng để SSH.
      - **Gateway=192.168.1.1** Định nghĩa Gateway mặc định.
      - **DNS=192.168.1.30**: Trỏ DNS Host về Technitium DNS Server Container (được tạo ở Bước 4).

  4.  Tạo mạng Docker IPVLAN (Trước khi reboot):

      ```bash
      docker network create -d ipvlan \
        --subnet=192.168.1.0/24 \
        --gateway=192.168.1.1 \
        -o parent=ens160 \
        -o ipvlan_mode=l2 \
        VLAN110
      ```

  5.  Khởi động lại (Reboot) để áp dụng toàn bộ cấu hình mạng Systemd:
  
      ```bash
      sudo reboot
      ```

---

- #### Bước 2.2: Đối với dùng MACVLAN (Cập nhật Systemd-networkd)

  Cấu hình tương tự MACVLAN để Host có thể giao tiếp với Container ( MACVLAN Host cần một IP riêng biệt (eg: `.254`) để giao tiếp với các Container )

  1.  Tạo mạng `macvlan` (Bắt buộc):

      ```bash
      docker network create -d macvlan \
        --subnet=192.168.1.0/24 \
        --gateway=192.168.1.1 \
        -o parent=ens160 \
        VLAN110
      ```

  1.  Tạo file \`.netdev\` (Tạo interface ảo `macvlan-host`):

      ```bash
      sudo nano /etc/systemd/network/20-macvlan-host.netdev
      ```

      ```bash
      [NetDev]
      Name=macvlan-host
      Kind=macvlan

      [MACVLAN]
      Device=ens160
      Mode=bridge
      ```

      **[Giải thích File 20-macvlan-host.netdev]**

      - **Kind=macvlan**: Chỉ định loại interface là MACVLAN.
      - **Device=ens160**: Gắn MACVLAN vào card mạng vật lý.
      - **Mode=bridge**: Cho phép interface ảo này hoạt động như một cầu nối, giúp Host giao tiếp với các Container MACVLAN.

  3.  Tạo file `.network` (Gán IP & Route cho `macvlan-host` và Fix ổn định):

      ```bash
      sudo nano /etc/systemd/network/25-macvlan-host.network
      ```

      ```ini
      [Match]
      Name=macvlan-host

      [Network]
      Address=192.168.1.254/24

      [Link]
      ActivationPolicy=up

      [Route]
      Destination=192.168.1.31/32
      Gateway=192.168.1.254

      [Route]
      Destination=192.168.1.30/32
      Gateway=192.168.1.254

      [Route]
      Destination=192.168.1.32/32
      Gateway=192.168.1.254

      [Route]
      Destination=192.168.1.34/32
      Gateway=192.168.1.254

      [Route]
      Destination=192.168.1.33/32
      Gateway=192.168.1.254

      [Install]
      WantedBy=network-online.target
      ```

      **[Giải thích File 25-macvlan-host.network]**

      - **Address=192.168.1.254/24**: Gán IP tĩnh riêng cho interface MACVLAN Host để giao tiếp (vì Host không thể giao tiếp trực tiếp với MACVLAN Container).
      - **[Route]**: Khối này định nghĩa các tuyến đường (routes).
      - **Destination=192.168.1.31/32**: Tạo tuyến đường cục bộ chỉ ra rằng để tới IP của NPM (\`.31\`), Host phải dùng Gateway chính là IP của chính nó trên MACVLAN (\`.254\`). Đây là cách Host "nhìn thấy" các Container.
      - **WantedBy=network-online.target**: Đảm bảo khởi tạo ổn định sau khi reboot.

  4.  Kích hoạt và Áp dụng `systemd-networkd`:

      ```bash
      # Khởi động lại service mạng (Sau khi reboot)
      sudo systemctl restart systemd-networkd
      ```
---

  - #### Bước 2.3: Cấu hình Docker Daemon (Trust Registry)

    Cấu hình Docker Daemon trên Host OS để tin tưởng Registry nội bộ và biết sử dụng Technitium DNS (`192.168.1.30`) để phân giải tên miền.(**Áp dụng cho cả IPVLAN và MACVLAN.**)

    1.  Sửa file `/etc/docker/daemon.json`:

        ```bash
        sudo nano /etc/docker/daemon.json
        ```

    2.  Thêm nội dung sau (Đã cập nhật domain):

        ```json
        {
          "insecure-registries": ["register.thongdev.site"],
          "dns": ["192.168.1.30", "8.8.8.8"]
        }
        ```

      **Giải thích các tham số (Why we need this)**

      - **"insecure-registries": \["register.thongdev.site"]**: Cho Docker biết rằng nó nên bỏ qua (skip) việc kiểm tra SSL/TLS tiêu chuẩn khi kết nối với Registry nội bộ này. Điều này giúp các container có thể push/pull image mà không bị lỗi xác thực SSL/TLS.
      - **"dns": \["192.168.1.30", "8.8.8.8"]**: Đảm bảo Docker daemon (và các container nếu không chỉ định DNS riêng) sử dụng Technitium DNS (`.30`) để phân giải các tên miền nội bộ như `gitlab.thongdev.site`.

    3.  Khởi động lại Docker daemon:

        ```bash
        sudo systemctl restart docker
        ```

---

- #### Bước 2.4: Cấu hình DNS Server (Host OS)
  **⚠️ XÓA HẲN** file `resolv.conf` cũ đi rồi mới tạo lại. Việc sửa trực tiếp (overwrite) đôi khi không hoạt động do file này là symlink. Nhằm đảm bảo Host OS dùng Technitium DNS (`192.168.1.30`) để phân giải các tên miền cục bộ.

  1.  Xóa và Tạo mới `resolv.conf`:

      ```bash
      # Xóa file cũ (hoặc symlink cũ)
      sudo rm -f /etc/resolv.conf

      # Tạo file mới
      sudo nano /etc/resolv.conf
      ```

  2.  Dán nội dung sau vào file mới:

      ```text
      nameserver 192.168.1.30
      search thongdev.site
      options edns0 trust-ad
      ```

  3.  🔒 Chốt cấu hình (Ngăn ghi đè):

      ```bash
      sudo chattr +i /etc/resolv.conf
      ```

      _(Lệnh này khóa file lại, đảm bảo không chương trình nào sửa đổi nó được nữa)._ 

  4.  Kiểm tra DNS Resolution trên Host (Sau khi service DNS chạy):

      ```bash
      nslookup gitlab.thongdev.site
      ```

      ```bash
      Server:         192.168.1.30
      Address:        192.168.1.30#53
      Name:   gitlab.thongdev.site
      Address: 192.168.1.31
      ```
---

- #### Bước 2.5: Gỡ lỗi và Kiểm tra trạng thái Networkd

  Nếu bạn đã thực hiện các bước trên nhưng vẫn gặp vấn đề (ví dụ: mất IP, DNS không hoạt động), hãy sử dụng lệnh \`journalctl\` để kiểm tra log chi tiết của `systemd-networkd`:

  - Sử dụng các lệnh sau để xem log của dịch vụ mạng, giúp bạn xác định lỗi cấu hình hoặc lỗi thời gian khởi động (race condition):

      ```bash
      # Xem log chi tiết của systemd-networkd (Từ đầu đến cuối)
      journalctl -u systemd-networkd --no-pager
      ```

  - **Giải thích:** Lệnh này hiển thị toàn bộ log của `systemd-networkd`. Bạn nên tìm các từ khóa như `fail`, `error`, `conflict`, hoặc `not found` để biết chính xác lỗi nằm ở file cấu hình nào.

      ```bash
      # Xem log chi tiết của systemd-networkd (Từ cuối log, mở rộng)
      journalctl -xeu systemd-networkd
      ```

  - **Giải thích:** Cờ `-x` (mở rộng) và `-e` (từ cuối log) giúp bạn tập trung vào các sự kiện gần nhất và xem các chi tiết hỗ trợ gỡ lỗi (như mã lỗi và giải thích). Rất hữu ích khi kiểm tra sau khi vừa reboot.

---

### Bước 3: Tạo file `jenkins/Dockerfile`

- Tạo file tại `/opt/devcom/jenkins/Dockerfile` ( có bao gồm cả kubectl )

  ```dockerfile
  # Start from the official Jenkins LTS image using JDK 17
  FROM jenkins/jenkins:lts-jdk17
  # Switch to root user to install packages
  USER root
  # Install prerequisites and Docker GPG key
  RUN apt-get update && apt-get install -y ca-certificates curl gnupg
  RUN install -m 0755 -d /etc/apt/keyrings
  RUN curl -fsSL https://download.docker.com/linux/debian/gpg -o   /etc/apt/keyrings/docker.asc
  RUN chmod a+r /etc/apt/keyrings/docker.asc
  # Add Docker repository
  RUN echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

  # Install Docker CLI
  RUN apt-get update && apt-get install -y docker-ce-cli

  # --- (CẬP NHẬT) CÀI ĐẶT KUBECTL ---
  RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  RUN install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
  # --- HẾT CẬP NHẬT ---

  # Switch back to jenkins user
  USER jenkins
  ```
---
### Bước 4: Tạo file `docker-compose.yml`

- Tạo file tại `/opt/devcom/docker-compose.yml`.

  ```yaml
  # ===================================================================================
  # ==            DOCKER-COMPOSE FOR DEVOPS LAB ENVIRONMENT (THONGDEV.SITE)          ==
  # ... (Header comments) ...
  # Version: 3.8 (Updated: Fixed GitLab Hostname Hairpin Issue)
  # ...
  # ===================================================================================

  version: "3.8"

  services:
    # --- SERVICE 1: TECHNITIUM DNS SERVER (IP: .30) ---
    dns-server:
      image: "technitium/dns-server:latest"
      container_name: dns-server
      hostname: dns.thongdev.site
      restart: always
      ports: ["5380:5380/tcp"] # Web UI
      volumes: ["./technitium-config:/etc/dns"]
      environment: ["TZ=Asia/Ho_Chi_Minh"]
      networks:
        VLAN110: { ipv4_address: 192.168.1.30 }

    # --- SERVICE 2: NGINX PROXY MANAGER (NPM) (IP: .31) ---
    npm:
      image: "jc21/nginx-proxy-manager:latest"
      container_name: nginx-proxy-manager
      restart: always
      ports: ["80:80", "443:443", "81:81"] # HTTP, HTTPS, Admin UI
      volumes: ["./npm/data:/data", "./npm/letsencrypt:/etc/letsencrypt"]
      networks:
        VLAN110: { ipv4_address: 192.168.1.31 }

    # --- SERVICE 3: GITLAB-EE (IP: .32) ---
    gitlab:
      image: "gitlab/gitlab-ee:latest"
      container_name: gitlab
      restart: always
      # [QUAN TRỌNG] SỬA ĐỔI HOSTNAME ĐỂ TRÁNH LỖI HAIRPINNING
      # KHÔNG dùng: hostname: 'gitlab.thongdev.site' (Trùng với domain public)
      hostname: "gitlab-internal"
      environment:
        GITLAB_OMNIBUS_CONFIG: |
          external_url 'http://gitlab.thongdev.site' # Initial, overridden by gitlab.rb
          gitlab_rails['initial_root_password'] = 'Devcom@2025'
      volumes:
        - "./gitlab/config:/etc/gitlab" # Source of Truth for config
        - "./gitlab/logs:/var/log/gitlab"
        - "./gitlab/data:/var/opt/gitlab"
      shm_size: "256m"
      dns: ["192.168.1.30", "1.1.1.1"] # Use internal DNS
      ports: ["22:22", "5050:5050"] # SSH, Registry internal port
      networks:
        VLAN110: { ipv4_address: 192.168.1.32 }

    # --- SERVICE 4: GITLAB RUNNER (IP: .33) - Optional ---
    gitlab-runner:
      image: "gitlab/gitlab-runner:latest"
      container_name: gitlab-runner
      privileged: true
      depends_on: [gitlab]
      restart: always
      volumes: [
          "./runner-config:/etc/gitlab-runner",
          "/var/run/docker.sock:/var/run/docker.sock",
        ] # SECURITY RISK
      dns: ["192.168.1.30", "1.1.1.1"]
      networks:
        VLAN110: { ipv4_address: 192.168.1.33 }

    # --- SERVICE 5: JENKINS (IP: .34) - All-in-One ---
    jenkins:
      build: { context: ./jenkins } # Use custom Dockerfile
      # container_name removed to fix DNS webhook issue
      restart: unless-stopped
      privileged: true # Insecure: Needed for docker.sock
      user: root # Insecure: Needed for docker.sock permissions
      volumes:
        - /opt/jenkins/kube/.kube:/root/.kube # Optional K8s config
        - /opt/jenkins/kube/.minikube:/root/.minikube # Optional K8s config
        - /opt/jenkins/jenkins_home:/var/jenkins_home # Persist Jenkins data
        - "/var/run/docker.sock:/var/run/docker.sock" # Insecure: Access host Docker
      dns: ["192.168.1.30", "1.1.1.1"] # Use internal DNS
      networks:
        VLAN110: { ipv4_address: 192.168.1.34 }

  # --- GLOBAL NETWORK CONFIGURATION ---
  networks:
    VLAN110:
      external: true # Use the pre-created MACVLAN or IPVLAN network
  ```

#### 🧠 KIẾN THỨC CHUYÊN SÂU: Cơ chế Docker DNS & Lỗi "Hairpinning"

- Phần này giải thích tại sao việc đặt `hostname: gitlab.thongdev.site` lại gây lỗi 443 (Connection Refused) và cách Docker DNS hoạt động bên dưới.

- ##### Phân biệt 3 khái niệm dễ nhầm lẫn
  - **Service Name (\`gitlab\`):** Tên bạn khai báo đầu dòng trong `docker-compose.yml`. Đây là tên dùng để các container trong cùng mạng gọi nhau (VD: `ping gitlab`). Docker tự động phân giải tên này ra IP của container.
  - **Container Name (\`gitlab\`):** Tên quản lý của container (hiển thị khi chạy `docker ps`). Giúp con người dễ quản lý (stop/start/logs), ít ảnh hưởng đến mạng nội bộ.
  - **Hostname (\`gitlab-internal\`):** Tên máy chủ \*bên trong\* hệ điều hành của container (hiển thị khi bạn gõ lệnh `hostname` trong terminal của container). **Đây là yếu tố then chốt gây ra lỗi.**

- ##### Cơ chế Docker DNS (Magic 127.0.0.11)
  - Mỗi container Docker đều có một DNS resolver riêng tại địa chỉ `127.0.0.11`. Khi một ứng dụng (như Jenkins) hỏi "IP của domain X là gì?", resolver này sẽ xử lý theo thứ tự ưu tiên:

  - **Local Context (Ưu tiên 1 - Cao nhất):** Kiểm tra xem domain X có trùng với `hostname` của chính nó hoặc của bất kỳ container nào khác trong cùng mạng không? Nếu có, trả về IP nội bộ của container đó ngay lập tức.
  - **Service Discovery (Ưu tiên 2):** Kiểm tra xem domain X có trùng với `service name` nào không?
  - **External DNS (Ưu tiên 3 - Thấp nhất):** Nếu không tìm thấy trong nội bộ, mới chuyển tiếp câu hỏi ra DNS Server bên ngoài (ở bài lab này là Technitium `192.168.1.30`).

- ##### Tại sao trùng tên = Lỗi (Hairpinning/Loopback)?
  - **Cấu hình Sai:** Bạn đặt `hostname: gitlab.thongdev.site` cho container GitLab.
  - **Hành vi:** Docker DNS tự động đăng ký tên miền này vào danh sách "Local Context".
  - **Luồng đi (Jenkins -&gt; GitLab):**
    - Jenkins gọi `https://gitlab.thongdev.site` (Port 443).
    - Docker DNS (127.0.0.11) thấy tên này trùng với hostname của container GitLab -&gt; Trả về IP nội bộ container (ví dụ: `172.20.0.5`) **thay vì** IP của NPM.
    - Jenkins cố kết nối tới `172.20.0.5:443`.
    - **LỖI:** Container GitLab chỉ cấu hình mở Port 80 (HTTP). Nó từ chối kết nối vào Port 443 -&gt; **Connection Refused**.

- ##### Tại sao đổi tên lại Fix được?
  - **Cấu hình Đúng:** `hostname: gitlab-internal`.
  - **Luồng đi:** Jenkins gọi `https://gitlab.thongdev.site`.
  - **Docker DNS:** Kiểm tra nội bộ -&gt; Không có container nào tên là `gitlab.thongdev.site` -&gt; Bỏ qua (Miss).
  - **Forward:** Chuyển tiếp câu hỏi cho Technitium DNS (`192.168.1.30`).
  - **Technitium:** Trả lời "Đó là IP của Nginx Proxy Manager (`192.168.1.31`)".
  - **Kết quả:** Jenkins kết nối `192.168.1.31:443` (NPM) -&gt; NPM hứng traffic, giải mã SSL, rồi chuyển tiếp vào GitLab:80 -&gt; **Thành công!**

---

### Bước 5: Khởi chạy Lần đầu & Cấu hình `gitlab.rb`

  Thực hiện bước này sau khi đã hoàn thành các cấu hình Docker Daemon và DNS Host ở Bước 2.3 và 2.

  1. Chạy Lần đầu (Run First Time):

     ```bash
     docker-compose up -d --build --force-recreate
     ```

  2. Sửa file `gitlab.rb` trên Host:

     Mở file `./gitlab/config/gitlab.rb`. Tìm và sửa các khối sau cho chính xác (bỏ comment `#` và sửa giá trị).

     ```ruby
     # --- Block 1: Web & SSH Config ---
     #dòng 32
     external_url 'https://gitlab.thongdev.site'
     #dòng 81
     gitlab_rails['gitlab_ssh_host'] = 'ssh.gitlab.thongdev.site'

     # --- Block 2: Registry Service Config ---
     #dòng 1027
     registry_external_url 'https://register.thongdev.site'
     #dòng 2411
     registry_nginx['enable'] = false
     #dòng 1047
     registry['enable'] = true
     # phải tự thêm
     registry['port'] = 5050
     #dòng 1055
     registry['registry_http_addr'] = "0.0.0.0:5050"

     # --- Block 3: App-to-Registry Connection ---
     #dòng 1030->1033
     gitlab_rails['registry_enabled'] = true
     gitlab_rails['registry_host'] = "register.thongdev.site"
     gitlab_rails['registry_port'] = "443"
     gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"

     # --- Block 4: Force Internal Nginx to HTTP ---
     #dòng 1858
     nginx['listen_https'] = false
     #dòng 1854
     nginx['listen_port'] = 80
     ```

  3. Áp dụng Cấu hình (Apply Configuration - Reconfigure):

     ```bash
     docker exec -it gitlab gitlab-ctl reconfigure
     ```

  4. #### 🔔 LƯU Ý VỀ THỨ TỰ CHẠY LỆNH DNS
     - Bạn cần khởi động lại service DNS của Host để áp dụng thay đổi Bước 2.4. Lệnh dưới đây **PHẢI ĐƯỢC CHẠY (HOẶC CHẠY LẠI) SAU BƯỚC 5 (Khi DNS Server đã khởi chạy)**.

       ```bash
       # CHẠY LỆNH NÀY SAU KHI LÀM XONG BƯỚC 5:
       sudo systemctl restart systemd-resolved
       sudo resolvectl flush-caches
       ```

     - **Tại sao?** Vì ở Bước 5, DNS Server (container 192.168.1.30) mới thực sự hoạt động. Nếu chạy lệnh này quá sớm khi container chưa lên, Host sẽ thấy DNS chết và chuyển sang dùng 8.8.8.8, dẫn đến lỗi.

---

### Bước 6: Cấu hình Mạng (DNS và NPM)

- Thiết lập hệ thống phân giải tên miền (DNS) và Reverse Proxy (NPM) để các dịch vụ có thể truy cập qua tên miền đẹp và bảo mật (HTTPS).

  1. Technitium DNS (`http://192.168.1.30:5380`):

     **Yêu cầu:** Tạo **7 bản ghi `A`** :
     - `gitlab.thongdev.site` ➡️ `192.168.1.31` (Trỏ về NPM)
     - `register.thongdev.site` ➡️ `192.168.1.31` (Trỏ về NPM)
     - `ssh.thongdev.site` ➡️ `192.168.1.32` (Trỏ **thẳng** vào GitLab)
     - `jenkins.thongdev.site` ➡️ `192.168.1.31` (Trỏ về NPM)
     - `dns.thongdev.site` ➡️ `192.168.1.31` (Trỏ về NPM - cho Web UI)
     - `npm.thongdev.site` ➡️ `192.168.1.31` (Trỏ về NPM - cho Web UI)
     - `ssh.gitlab.thongdev.site` ➡️ `192.168.1.32` (Trỏ **thẳng** vào GitLab)
     <image src="./3.png">

     **Chi tiết: Cách thêm bản ghi A (URL) trong Technitium**
     - Đăng nhập vào Technitium DNS: `http://192.168.1.30:5380`.
     - Nếu chưa có Zone: Nhấn "Add Zone" -&gt; "Primary Zone". Nhập "Zone Name" là `thongdev.site` -&gt; "Add".
     <image src="./1.png">
     - Nhấn vào tên zone `thongdev.site`.
     -  Phía trên, nhấn nút "Add Record".
     - **Name:** Nhập tiền tố (ví dụ: `gitlab`, `register`, `ssh`, `jenkins`, `dns`, `npm`).
     - **Type:** Chọn `A`.
     - **IP Address:** Nhập địa chỉ IP tương ứng (ví dụ: `192.168.1.31` hoặc `192.168.1.32` theo danh sách trên).
     - **TTL:** Để mặc định (hoặc 300).
     - Nhấn "Add Record".
     <image src="./2.png">
     - Lặp lại cho cả 6 bản ghi.

  2. Nginx Proxy Manager (`http://192.168.1.31:81`):
     **Yêu cầu:** Tạo **5 Proxy Hosts**:
     - **Host 1 (GitLab Web):** Domain `gitlab.thongdev.site` ➡️ Forward `http://192.168.1.32:80`
     - **Host 2 (GitLab Registry):** Domain `register.thongdev.site` ➡️ Forward `http://192.168.1.32:5050`
     - **Host 3 (Jenkins):** Domain `jenkins.thongdev.site` ➡️ Forward `http://192.168.1.34:8080` (Tích **Websockets Support**).
     - **Host 4 (DNS UI):** Domain `dns.thongdev.site` ➡️ Forward `http://192.168.1.30:5380`
     - **Host 5 (NPM UI):** Domain `npm.thongdev.site` ➡️ Forward `http://192.168.1.31:81`

     **Chi tiết: Cách thêm Proxy Host (URL) trong NPM**
      -  Đăng nhập vào NPM: `http://192.168.1.31:81` (Email mặc định: `admin@example.com`, Pass: `changeme`).
      - Vào "Hosts" -&gt; "Proxy Hosts" -&gt; "Add Proxy Host".
      -  **Tab Details:**
          <image src ="./4.png">
           - **Domain Names:** Nhập tên miền (ví dụ: `gitlab.thongdev.site`).
           - **Scheme:** `http`
           - **Forward Hostname / IP:** Nhập IP của dịch vụ (ví dụ: `192.168.1.32`).
           - **Forward Port:** Nhập port của dịch vụ (ví dụ: `80`).
           - Tích **"Block Common Exploits"**.
           - Tích **"Websockets Support"** (Đặc biệt quan trọng cho Jenkins và GitLab).

     - **Tab SSL:**
        <image src ="./5.png">
        <image src ="./9.png">
     -  Nhấn "Save".
     - Lặp lại cho các host còn lại.

     **Chi tiết: Cách thêm SSL (Self-Signed hoặc Let's Encrypt)**
     - Sau khi tạo Proxy Host, click vào 3 chấm bên phải của host đó -&gt; "Edit".
     - Chuyển qua tab **SSL**.
     - **Cách 1: Dùng Let's Encrypt (Khuyến nghị nếu có domain public và đã NAT port):**
      [API-token-dns](https://dash.cloudflare.com/profile/api-tokens)
      <image src ="./6.png">
      <image src ="./7.png">
      <image src ="./8.png">

       - Thay token lấy được trên cloudflare vào chỗ dns_cloudflare_api_token
       - Chỗ domain Names: \*.ten_domain (eg: \*.thongdev.site)
     - **Cách 2: Dùng `mkcert` (Tạo Self-Signed được Tin cậy - Khuyến nghị cho Lab):**

       Cách này tạo ra một "Certificate Authority" (CA) giả lập trên máy ảo của bạn, sau đó tạo chứng chỉ từ CA đó. Cuối cùng, bạn chỉ cần "tin tưởng" CA này trên máy Mac/Windows của mình là mọi trình duyệt sẽ hiển thị ổ khóa màu xanh.

       **1. ⚙️ Kiểm tra Kiến trúc & Cài đặt `mkcert` (Trên máy ảo VM)**
       - **Kiểm tra kiến trúc CPU (Quan trọng):**

         ```bash
         uname -m
         ```

         - Nếu kết quả là `x86_64` (hoặc `amd64`), bạn dùng bản **amd64**.
         - Nếu kết quả là `aarch64` (hoặc `arm64`), bạn đang dùng ARM (ví dụ: Mac M1/M2/M3). Bạn phải dùng bản **arm64**.

       - **Cài đặt các gói phụ trợ:**

         ```bash
         sudo apt-get update
         sudo apt-get install -y libnss3-tools
         ```

       - **Tải đúng phiên bản `mkcert`:**  
          (Chỉ chạy **MỘT** trong hai lệnh `wget` sau, tùy theo kết quả bước 1)

         ```bash
         # DÀNH CHO x86_64 / amd64 (Hầu hết máy chủ Intel/AMD)
         wget -O mkcert https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64

         # DÀNH CHO aarch64 / arm64 (VD: Mac M1/M2/M3 VM)
         wget -O mkcert https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-arm64
         ```

       - **Cài đặt `mkcert` vào hệ thống:**

         ```bash
         # Cấp quyền thực thi
         chmod +x mkcert
         # Di chuyển vào thư mục hệ thống
         sudo mv mkcert /usr/local/bin/
         ```

       #### 2. 👑 Tạo Certificate Authority (CA) trên VM

       Đây là bước bạn "phong" cho máy ảo của mình làm một "cơ quan cấp chứng chỉ". **Chỉ chạy một lần duy nhất.**

       ```bash
       mkcert -install
       ```

       **Gặp lỗi `Exec format error`?**  
       Nếu bạn gặp lỗi này khi chạy `mkcert -install`, điều đó 100% có nghĩa là bạn đã tải sai phiên bản (amd64/arm64) ở bước 1. Hãy quay lại, xóa file `/usr/local/bin/mkcert` (`sudo rm /usr/local/bin/mkcert`) và tải lại đúng phiên bản `arm64`.

       #### 3. 📜 Tạo Chứng chỉ (Certificate) cho các Dịch vụ

       Tạo chứng chỉ cho tất cả các tên miền lab của bạn (và cả IP của NPM) trong một lần.

       ```bash
       # Tạo một thư mục để chứa chứng chỉ
       mkdir -p ~/ssl-certs
       cd ~/ssl-certs

       # Lệnh quan trọng: Tạo chứng chỉ cho TẤT CẢ các tên miền lab (CẬP NHẬT DOMAIN)
       mkcert dns.thongdev.site npm.thongdev.site gitlab.thongdev.site register.thongdev.site jenkins.thongdev.site 192.168.1.31
       ```

       Lệnh này sẽ tạo ra 2 file trong thư mục `~/ssl-certs` (tên file có thể dài):
       - `dns.thongdev.site+...pem` (File Certificate)
       - `dns.thongdev.site+...-key.pem` (File Certificate Key)

       #### 4. 📤 Tải file lên Nginx Proxy Manager (NPM)
       - Trong NPM, vào "SSL Certificates" -&gt; "Add SSL Certificate" -&gt; "Custom".
       - **Name:** `DevCom Certs` (hoặc tên bất kỳ bạn muốn).
       - **Certificate Key:** Nhấn "Browse", tìm đến file `dns.thongdev.site+...-key.pem` (file có chữ `-key`).
       - **Certificate:** Nhấn "Browse", tìm đến file `dns.thongdev.site+...pem` (file không có chữ `key`).
       - **Intermediate Certificate:** Để trống.
       - Nhấn "Save".
       - **Gán Chứng chỉ:** Quay lại "Proxy Hosts", sửa từng host. Vào tab **SSL**, chọn `DevCom Certs` từ menu dropdown, và tích "Force SSL". Nhấn "Save".

       #### 5. 💻 (QUAN TRỌNG) "Tin tưởng" (Trust) CA trên máy thật của bạn

       Bước cuối cùng là bảo cho máy thật (Host OS - vd: máy Mac hoặc Windows) của bạn "tin tưởng" cái CA mà máy ảo đã tạo ra.
       - **Trên máy ảo (VM):** Tìm và đọc file Root CA:

         ```
         # (Chạy trên máy Host)
         mkcert -CAROOT

         # Đọc nội dung file rootCA.pem (thay đường dẫn nếu khác)
         cat $(mkcert -CAROOT)/rootCA.pem
         ```

       - **Sao chép** toàn bộ khối văn bản (bắt đầu bằng `-----BEGIN CERTIFICATE-----`).
       - **Trên máy thật (Host - Mac/Windows):**
         - Mở trình soạn thảo văn bản (TextEdit, Notepad), dán nội dung vào.
         - Lưu file trên Desktop với tên `rootCA.pem`.

       - **Cài đặt CA trên máy thật:**
         - **Trên macOS:**
           1. Mở ứng dụng **Keychain Access**.
           2. Kéo file `rootCA.pem` từ Desktop vào cửa sổ Keychain Access (trong mục "Certificates").
           3. Tìm chứng chỉ mới (tên kiểu như "mkcert ...").
           4. Nháy đúp (Double-click) vào chứng chỉ đó.
           5. Mở rộng phần "Trust" (Tin cậy).
           6. Trong mục "When using this certificate:", chọn "Always Trust" (Luôn tin cậy).
           7. Đóng cửa sổ (sẽ yêu cầu mật khẩu máy Mac).

         - **Trên Windows:**
           1. Nháy đúp vào file `rootCA.pem`.
           2. Nhấn "Install Certificate...".
           3. Chọn "Local Machine" -&gt; "Next".
           4. Chọn "Place all certificates in the following store".
           5. Nhấn "Browse..." -&gt; Chọn "Trusted Root Certification Authorities" -&gt; "OK".
           6. Nhấn "Next" -&gt; "Finish".
       1. **Hoàn tất:** Khởi động lại trình duyệt của bạn (Chrome, Firefox...). Bây giờ khi truy cập `https://gitlab.thongdev.site`, bạn sẽ thấy ổ khóa màu xanh!

       2. **💡 Mẹo Fix Bug: Trình duyệt vẫn báo "Not Secure" sau khi đã Trust?**
       - Trình duyệt (đặc biệt là Chrome và Edge) có cache DNS và socket riêng. Bạn cần xóa cache này để nó nhận CA mới:
       - **Trên Chrome/Edge (Windows/Mac/Linux):**
         1. Mở tab mới.
         2. Gõ `chrome://net-internals/#sockets` (cho Chrome) hoặc `edge://net-internals/#sockets` (cho Edge) và nhấn Enter.
         3. Nhấn vào nút **"Flush socket pools"**.

       - **Trên macOS (Chạy thêm trên Terminal):**

       ```bash
       # Chạy lệnh này để xóa cache DNS của hệ điều hành:
       sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
       ```

       - Sau đó, khởi động lại trình duyệt một lần nữa.
   3. ✅ Kiểm tra
   - Truy cập `https://gitlab.thongdev.site` phải thấy GitLab.
   - Truy cập `https://jenkins.thongdev.site` phải thấy Jenkins.
   - Chạy `docker login register.thongdev.site` phải thành công (dùng username GitLab & PAT).
   - Chạy `ssh git@ssh.gitlab.thongdev.site` phải kết nối được. (chạy trên máy Mac)

---

- #### 🕵️‍♂️ BUG: "Hmmm… can't reach this page" (Trên MacOS)

  Nếu bạn đã làm hết các bước trên mà vào Chrome trên Mac vẫn lỗi, hãy làm theo quy trình "bắt bệnh" này:

  **Bước 1**: Kiểm tra Hệ thống DNS MacOS (scutil)

    - Mở Terminal trên Mac và chạy lệnh sau để xem Mac đang dùng DNS nào cho domain `thongdev.site`:

      ```
      scutil --dns
      ```

      **Kết quả mong đợi:** Bạn phải tìm thấy một khối (thường là \`resolver #8\` hoặc tương tự) có nội dung sau:

      ```
      resolver #8
        domain   : thongdev.site
        nameserver[0] : 192.168.1.30
        flags    : Request A records
        reach    : 0x00020002 (Reachable)
      ```

  **Nếu không thấy khối này?**

  - Nghĩa là bạn chưa cấu hình Resolver đúng.
  - Chạy lệnh: `sudo mkdir -p /etc/resolver`
  - Tạo file: `sudo nano /etc/resolver/thongdev.site`
  - Nhập: `nameserver 192.168.1.30`
  - Lưu lại và kiểm tra lại \`scutil --dns\`.
  - **Quan trọng:** Kiểm tra xem có file rác nào khác (như \`dev.com\` cũ) trong thư mục này không bằng lệnh `ls /etc/resolver/`. Nếu có, xóa ngay bằng `sudo rm /etc/resolver/dev.com`.

  **Bước 2**: Kiểm tra "thông đường" (Ping)

  - Vẫn trên Terminal Mac, thử ping tên miền:

    ```
    ping -c 2 gitlab.thongdev.site
    ```

  - **Nếu trả về \`192.168.1.31\`:** DNS Mac đã ngon. Vấn đề nằm ở Chrome. (Sang Bước 3)
  - **Nếu báo \`Unknown host\`:** Quay lại Bước 1, Mac chưa nhận DNS.
  - **Nếu ping được nhưng Browser lỗi:** Kiểm tra Proxy trong NPM xem đã trỏ đúng IP/Port chưa.

  **Bước 3**: Xóa bộ đệm (Cache) cứng đầu của Chrome

  - Chrome có bộ đệm DNS và Socket rất "dai", ngay cả khi Mac đã sửa đúng, Chrome vẫn nhớ cái sai cũ.

    - Mở Chrome, gõ vào thanh địa chỉ: `chrome://net-internals/#dns` -&gt; Nhấn **Clear host cache**.
    - Gõ tiếp: `chrome://net-internals/#sockets` -&gt; Nhấn **Flush socket pools** (Nút này quan trọng nhất).
    - Khởi động lại Chrome và thử truy cập lại.

---

## Giai đoạn 1: GitLab (Nơi chứa Code) 📝

Tạo project và file pipeline `Jenkinsfile`.

1. Tạo Project (Create Project):
   - Truy cập `https://gitlab.thongdev.site` (User: `root`, Pass: `Devcom@2025`).
   - Tạo project mới `corejs`.

2. Tạo `Jenkinsfile` (Create Jenkinsfile):

   Nội dung file `Jenkinsfile` này chỉ dùng cho \*\*Giai đoạn 4 (Deploy Docker Host)\*\*. Chúng ta sẽ tạo file khác cho Giai đoạn 5 (K8s).

```groovy
// Jenkinsfile - Simple Deploy to Docker Host (Cách 1: All-in-One)
pipeline {
   agent any
   environment {
       APP_NAME            = 'corejs'
       FRONTEND_IMAGE      = "devcom/${APP_NAME}-frontend:latest"
       BACKEND_IMAGE       = "devcom/${APP_NAME}-backend:latest"
       FRONTEND_CONTAINER  = "${APP_NAME}-frontend-app"
       BACKEND_CONTAINER   = "${APP_NAME}-backend-app"
       FRONTEND_HOST_PORT = 8081
       BACKEND_HOST_PORT  = 5001
       DOCKER_NETWORK      = 'VLAN110'
   }
   stages {
       stage('1. Checkout Code') { steps { checkout scm } }
       stage('2. Build Docker Images') {
           parallel {
               stage('Build Frontend') {
                   steps {
                       dir('frontend') {
                           sh "docker build -t ${env.FRONTEND_IMAGE} ."
                       }
                   }
               }
               stage('Build Backend') {
                   steps {
                       dir('CoreAPI') {
                           sh "docker build -t ${env.BACKEND_IMAGE} ."
                       }
                   }
               }
           }
       }
       stage('3. CTO Approval') {
           steps {
               timeout(time: 1, unit: 'HOURS') {
                   input message: 'Approve deployment to Production (Docker Host)?',
                         ok: 'Proceed to Deploy',
                         submitter: 'cto'
               }
           }
       }
       stage('4. Deploy to Production (Docker Host)') {
           steps {
               echo "INFO: Stopping and removing old containers (if they exist)..."
               sh "docker stop ${env.FRONTEND_CONTAINER} || true"
               sh "docker rm ${env.FRONTEND_CONTAINER} || true"
               sh "docker stop ${env.BACKEND_CONTAINER} || true"
               sh "docker rm ${env.BACKEND_CONTAINER} || true"

               echo "INFO: Starting new Backend container..."
               sh "docker run -d --name ${env.BACKEND_CONTAINER} -p ${env.BACKEND_HOST_PORT}:80 --network ${env.DOCKER_NETWORK} --hostname ${env.BACKEND_CONTAINER} --restart always ${env.BACKEND_IMAGE}"

                echo "INFO: Starting new Frontend container..."
               sh "docker run -d --name ${env.FRONTEND_CONTAINER} -p ${env.FRONTEND_HOST_PORT}:80 --network ${env.DOCKER_NETWORK} --hostname ${env.FRONTEND_CONTAINER} --restart always ${env.FRONTEND_IMAGE}"

               echo "✅ DEPLOYMENT COMPLETE! Access at: http://:${env.FRONTEND_HOST_PORT}"
           }
       }
   }
}
```

3. Push `Jenkinsfile` lên GitLab:

   ```
   git add Jenkinsfile
   git commit -m "Add Jenkinsfile for Docker host deployment"
   git push -u origin
   ```

## Giai đoạn 2: Jenkins (Nơi Build)

Cấu hình Jenkins để nó "biết" về project và cách thực thi pipeline.

1. Đăng nhập lần đầu & Cài đặt cơ bản:
   - Truy cập `https://jenkins.thongdev.site`.
   - Lấy mật khẩu admin ban đầu, rồi điền vào chỗ ảnh duới

     ```
     docker exec $(docker ps -qf "name=devcom_jenkins") cat /var/jenkins_home/secrets/initialAdminPassword
     ```
     <image src ="./10.png">

   - Hoàn thành cài đặt ("Install suggested plugins"), tạo user admin (ví dụ: `tony`).
   - **Tạo user "cto":** Vào `Manage Jenkins` -&gt; `Security` -&gt; `Manage Users` -&gt; `Create User`. Tạo user tên `cto`.

2. Cài Plugin GitLab & Kubernetes:
   - Vào `Manage Jenkins` -&gt; `Plugins` -&gt; `Available plugins`.
   - Blue Ocean – giao diện pipeline trực quan.
   - Docker Pipeline
   - Kubernetes CLI
   - GitLab Plugin
   - Role-based Authorization Strategy
     <image src ="./11.png">
  2.1 Cài đặt system, security Jenkins (Authorization, Gitlab) như hình.
  <image src ="./14.png">
  <image src ="./13.png">
  <image src ="./12.png">

3. Tạo Credentials cho GitLab: Để Jenkins đọc code VÀ push image

   Chúng ta cần tạo 2 credentials:
   <image src ="./15.png">
   1. **Credential 1: Đọc/Viết Code (Dùng PAT)**
      - Vào GitLab (user `root`), tạo **Personal Access Token (PAT)**. Đặt tên `jenkins-api-token`, chọn scopes **`api`** và **`read_repository`** . Copy token.
      - Vào Jenkins `Manage Jenkins` -&gt; `Credentials` -&gt; `(global)`.
      - `Add Credentials`:
        - **Kind:** `GitLab Personal Access Token`
        - **Token:** Dán token của bạn.
        - **ID:** `gitlab-token` (Ghi nhớ ID này).

   2. **Credential 2: Đăng nhập Registry (Dùng PAT)**

      Jenkins cần credential này để chạy `docker login register.thongdev.site` (ở Giai đoạn 5) và cần nó để có thể push registry lên gitlab ở bước 2.5 như ảnh
      <image src ="./44.png">
      - Vào GitLab (user `root`), tạo PAT mới (hoặc dùng PAT cũ). Đặt tên `jenkins-registry-token`.
      - **Scopes (BẮT BUỘC):** Tick chọn **`read_registry`** VÀ **`write_registry`** . Copy token.
      - Vào Jenkins `Manage Jenkins` -&gt; `Credentials` -&gt; `(global)`.
      - `Add Credentials`:
        - **Kind:** `Username with password`
        - **Username:** `root` (hoặc username GitLab của bạn)
        - **Password:** Dán chuỗi PAT `read/write_registry` vào đây.
        - **ID:** `gitlab-registry-creds` (PHẢI KHỚP với \`Jenkinsfile\` K8s)

4. Tạo Pipeline Job: (Đây là "Công việc" Jenkins sẽ thực thi)
   <image src ="./17.png">
   <image src ="./21.png">
    (Cái này dùng PAT có quyền với Repo vì nó có tác dụng kéo code từ gitlab về)
   <image src ="./18.png">
   - Trang chủ Jenkins -&gt; `New Item`.
   - **Enter an item name:** `corejs-build-deploy`.
   - Chọn **`Pipeline`** -&gt; `OK`.
   - **Tab General:** Tích `GitLab Connection`.
   - **Tab Pipeline:**
     - **Definition:** `Pipeline script from SCM`.
     - **SCM:** `Git`.
     - **Repository URL:** `https://gitlab.thongdev.site/root/corejs.git` (URL HTTPS của project).
     - **Credentials:** Chọn `gitlab-token` (Cũng dùng PAT có quyền tương tác với Repo).
     - **Branches to build** -&gt; **Branch Specifier:** `*/nodejs` (Hoặc nhánh bạn push `Jenkinsfile` lên).
     - **Script Path:** `Jenkinsfile`.
     - \*\*(Quan trọng)\*\* Nhấn `Add` bên cạnh **Additional Behaviours** -&gt; Chọn **`Wipe out repository & force clone`** .

   - Nhấn `Save`.
   - Nhớ tạo credentials cho jenkins ở link [Credential](https://jenkins.thongdev.site/manage/credentials/store/system/domain/_/)
   <image src ="./20.png">
    
   

## Giai đoạn 3: Kết nối Webhook (Trigger) 🔗

1. Lấy Secret Token từ Jenkins:
   - Mở job `corejs-build-deploy` -&gt; `Configure` -&gt; `Build Triggers`.
   - Tích vào: `Build when a change is pushed to GitLab`.
   - Nhấn `Advanced...` -&gt; `Generate` (trong mục Secret token).
   - **Copy** token bí mật đó.
   - Nhấn `Save`.
   <image src ="./22.png">
   <image src ="./23.png">
2. Tạo Webhook trên GitLab:
   - Mở project `corejs` -&gt; `Settings` -&gt; `Webhooks`.
   - **URL:** `https://jenkins.thongdev.site/project/corejs-build-deploy` (Thay tên job nếu khác).
   - **Secret Token:** **Dán** token bí mật từ Jenkins (lấy ở ngay ảnh bên trên).
   - **Trigger:** Chỉ tích `Push events`.
   - **SSL verification:** **BỎ TÍCH** (Untick) ô **"Enable SSL verification"**.
   - Nhấn `Add webhook`.
   <image src ="./24.png">

#### ⚠️ Khắc phục lỗi: "Invalid url given"

Nếu bạn gặp lỗi này khi nhấn "Add webhook" hoặc khi Test, hãy thực hiện cấu hình sau:
<image src ="./25.png">
1. Truy cập **Admin Area** (biểu tượng cờ lê hoặc Menu &gt; Admin).
2. Vào **Settings** &gt; **Network** &gt; **Outbound Requests**.
3. Tích vào ô: **"Allow requests to the local network from webhooks and integrations"**.
4. Tích vào ô: **"Allow requests to the local network from system hooks"**.
5. Nhấn **Save changes**.


**Kiểm tra lại:**

- Quay lại trang Webhooks của Project.
- Sau khi Add Webhook xong, kéo xuống dưới phần "Webhook Hooks".
- Nhấn nút **Test** &gt; Chọn **Push events**.
- Nếu kết quả trả về **HTTP 200** là thành công.

## Giai đoạn 4: Chạy Thử & Kiểm Tra (Deploy Docker Host) 🚀

1. Quan sát Jenkins:
   - Mở `https://jenkins.thongdev.site`.
   - Job `corejs-build-deploy` sẽ tự động chạy (hoặc nhấn `Build Now`).
   - Nó sẽ chạy qua Stage 1, 2 và **DỪNG LẠI** ở Stage 3 "CTO Approval".

2. CTO Phê duyệt (CTO Approval):
   - Đăng nhập vào Jenkins bằng user **`cto`** .
   - Mở job đang tạm dừng.
   - Nhấn nút **`Proceed`** .
<image src ="./27.png">

3. Hoàn tất Deploy & Kiểm tra Ứng dụng:
   - Pipeline sẽ tiếp tục chạy Stage 4 (Deploy Docker Host) và báo `SUCCESS`.
   - Truy cập Ứng dụng: `http://192.168.1.161:8081` và `http://192.168.1.161:5001` (Thay IP host và port nếu bạn đặt khác).
  <image src ="./26.png">
  <image src ="./28.png">
  <image src ="./29.png">
---

## Giai đoạn 5: Triển khai lên Kubernetes 
Sau khi đã thành thạo “Cách 1”, bạn có thể nâng cấp pipeline để deploy ứng dụng lên cụm Kubernetes thay vì Docker host. Đây là hướng dẫn sơ bộ, bạn cần điều chỉnh chi tiết cho phù hợp.

- **Bước 1: Chuẩn bị file Manifest Kubernetes**
  - Bạn cần tạo các file YAML định nghĩa cách ứng dụng chạy trên K8s (Deployment, Service). Tạo một thư mục k8s trong project corejs.
  - File: k8s/namespace.yaml
    ```bash
        apiVersion: v1
        kind: Namespace
        metadata:
        name: corejs-prod # Tên namespace cho ứng dụng
    ```
  - File: k8s/registry-secret.yaml (Bắt buộc nếu Registry không public)
    ```
      kubectl create secret docker-registry gitlab-registry-creds \
      --docker-server=register.dev.com \
      --docker-username=YOUR_GITLAB_USERNAME \
      --docker-password=YOUR_GITLAB_PAT \ # Cái này là PAT có quyền với Registry tạo ở trên đó
      --namespace=corejs-prod \
      --dry-run=client -o yaml > k8s/registry-secret.yaml
    ```
    File registry-secret.yaml sẽ được tạo ra.
  - File: k8s/backend-deployment.yaml
    ```bash
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: corejs-backend
          namespace: corejs-prod
        spec:
          replicas: 1 # Số lượng pod muốn chạy
          selector:
            matchLabels:
              app: corejs-backend
          template:
            metadata:
              labels:
                app: corejs-backend
            spec:
              # K8s sẽ dùng secret này để kéo image
              imagePullSecrets:
              - name: gitlab-registry-creds
              containers:
              - name: backend
                # Image được build bởi Jenkins
                image: devcom/corejs-backend:latest 
                ports:
                - containerPort: 80 # Port mà backend lắng nghe bên trong
    ```

  - File: k8s/backend-service.yaml
      ```
        apiVersion: v1
        kind: Service
        metadata:
          name: corejs-backend-svc # Tên service nội bộ
          namespace: corejs-prod
        spec:
          selector:
            app: corejs-backend
          ports:
            - protocol: TCP
              port: 80 # Port mà các service khác trong K8s gọi đến
              targetPort: 80 # Trỏ đến containerPort của Deployment
          # Type: ClusterIP là mặc định, chỉ truy cập được bên trong K8s
      ```

  - File: k8s/frontend-deployment.yaml
    ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: corejs-frontend
        namespace: corejs-prod
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: corejs-frontend
        template:
          metadata:
            labels:
              app: corejs-frontend
          spec:
            imagePullSecrets:
            - name: gitlab-registry-creds
            containers:
            - name: frontend
              image: devcom/corejs-frontend:latest # Image Nginx đã build
              ports:
              - containerPort: 80 # Port Nginx lắng nghe bên trong
    ```
  - File: k8s/frontend-service.yaml (Dùng NodePort)
    ```
      apiVersion: v1
      kind: Service
      metadata:
        name: corejs-frontend-svc
        namespace: corejs-prod
      spec:
        selector:
          app: corejs-frontend
        # --- SỬ DỤNG NODEPORT ĐỂ TRUY CẬP TỪ BÊN NGOÀI ---
        type: NodePort 
        ports:
          - protocol: TCP
            port: 80       # Port bên trong cluster
            targetPort: 80   # Port của container
            # nodePort: 30080 # Tùy chọn: Chỉ định port cụ thể (30000-32767)
            # Nếu bỏ trống, K8s sẽ tự chọn 1 port NodePort

    ```  
    Đẩy thư mục k8s chứa các file này lên GitLab.

- **Bước 2: Cài đặt kubectl trong Jenkins (Đã làm ở Dockerfile)**
  - Kiểm tra, đi vào container jenkins gõ lệnh kubectl
    ```
      iadmin@srv025-aio:~$ docker exec -it devops-jenkins-1 /bin/bash
      root@f172c54d2f72:/# kubectl 
      kubectl controls the Kubernetes cluster manager.

      Find more information at: https://kubernetes.io/docs/reference/kubectl/

      Basic Commands (Beginner):
        create          Create a resource from a file or from stdin
        expose          Take a replication controller, service, deployment or pod and expose it as a new Kubernetes service
        run             Run a particular image on the cluster
        set             Set specific features on objects
    ```

- **Bước 3: Tạo K8s Credentials trong Jenkins**
  Jenkins cần quyền để kết nối và deploy lên cụm K8s.
  - Cách 1 (Username/Password - Đơn giản nhưng kém an toàn):
    - Vào Jenkins -> Credentials -> (global) -> Add Credentials.
    - Kind: Username with password.
    - Username: devops
    - Password: Devcom@2025
    - ID: k8s-user-creds
    <image src="./30.png">

  - Cách 2 (Kubeconfig - Khuyến nghị):
    - SSH vào k8s-master-1.
    - Copy nội dung file ~/.kube/config.
    - Vào Jenkins -> Credentials -> (global) -> Add Credentials.
    - Kind: Kubernetes configuration (kubeconfig).
    - ID: k8s-cluster-config
    - Kubeconfig: Chọn Enter directly và dán nội dung file config vào.
    <image src="./31.png">
  - Cách 3:
    - SSH vào k8s-master-1.
    - Copy nội dung file ~/.kube/config.
    - Cần phải cài thêm Kubernetes, Kubernetes CLI Plugin sau đó add thêm Credentials
    - Vào Jenkins -> Credentials -> (global) -> Add Credentials.
    - Kind: Secret file.
    - Uploadfile config của kụm k8s lên Jenkins
    - ID: k8s-config-file
    <image src="./32.png">
  - **Lưu ý**: Cách dùng Kubeconfig an toàn và linh hoạt hơn. Jenkins Controller cần mount volume /opt/devops/kube/.kube:/root/.kube (như trong docker-compose.yml) để kubectl hoạt động.
  
**Bước 4: Cập nhật Jenkinsfile (Thêm Stage Deploy K8s)**
Sửa lại Jenkinsfile trong project corejs.
  ```
          // Jenkinsfile - Deploy to Kubernetes
      pipeline {
          agent any 

          environment {
              // --- Application & Image Naming ---
              APP_NAME            = 'corejs'
              REGISTRY_HOST       = 'register.thongdev.site'
              // (QUAN TRỌNG) Đường dẫn project trên GitLab (ví dụ: tonylab/corejs)
              GITLAB_PROJECT_PATH = 'tonylab/corejs' 

              // (QUAN TRỌNG) Tên image đầy đủ. Phải khớp với file deployment.yaml
              FRONTEND_IMAGE      = "${env.REGISTRY_HOST}/${env.GITLAB_PROJECT_PATH}/frontend:latest"
              BACKEND_IMAGE       = "${env.REGISTRY_HOST}/${env.GITLAB_PROJECT_PATH}/backend:latest"

              // --- K8s Variables ---
              
              // (CẬP NHẬT) K8S_NAMESPACE
              // Tác dụng: Chỉ định "không gian làm việc" (namespace) riêng cho ứng dụng trong K8s.
              // Dùng ở đâu: Được dùng trong Stage 4 (Deploy) với cờ '-n' 
              //             (ví dụ: `kubectl apply -f ... -n ${env.K8S_NAMESPACE}`).
              // Tạo ở đâu: Được định nghĩa trong file `k8s/namespace.yaml`.
              K8S_NAMESPACE       = 'corejs-prod'
              
              // (CẬP NHẬT) K8S_CREDENTIAL_ID
              // Tác dụng: Đây là ID của "chìa khóa" (kubeconfig) mà Jenkins cần để có
              //             quyền đăng nhập và điều khiển cụm K8s của bạn.
              // Dùng ở đâu: Được dùng trong Stage 4 (Deploy) bởi hàm `withKubeConfig(...)`.
              // Tạo ở đâu: Bạn phải tạo credential này thủ công trong Jenkins 
              //             (Giai đoạn 5, Bước 2).
              K8S_CREDENTIAL_ID   = 'k8s-cluster-config'
              
              // ID của PAT (Giai đoạn 2, Credential 2)
              REGISTRY_CREDENTIAL_ID = 'gitlab-registry-creds' 
          }

          stages {
              // --- Stage 1: Get latest code ---
              stage('1. Checkout Code') {
                  steps {
                      checkout scm 
                      echo "SUCCESS: Code checked out from GitLab."
                  }
              }
              
              // --- Stage 2: Build Production Docker Images ---
              stage('2. Build Docker Images') {
                  parallel {
                      stage('Build Frontend') {
                          steps {
                              dir('frontend') {
                                  echo "INFO: Building Frontend production image: ${env.FRONTEND_IMAGE}"
                                  sh "docker build -t ${env.FRONTEND_IMAGE} ." 
                              }
                          }
                      }
                      stage('Build Backend') {
                          steps {
                              dir('CoreAPI') {
                                  echo "INFO: Building Backend image: ${env.BACKEND_IMAGE}"
                                  sh "docker build -t ${env.BACKEND_IMAGE} ."
                              }
                          }
                      }
                  } // End parallel build
              } // End Stage 2
              
              // --- (MỚI) Stage 2.5: Push Images to GitLab Registry ---
              stage('2.5. Push Images to Registry') {
                  steps {
                      script {
                          // Sử dụng Credential 2 (Username/Password) đã tạo
                          withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIAL_ID, passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                              
                              echo "INFO: Logging in to ${env.REGISTRY_HOST}..."
                              sh "docker login -u ${REG_USER} -p ${REG_PASS} ${env.REGISTRY_HOST}"
                              
                              echo "INFO: Pushing Frontend image: ${env.FRONTEND_IMAGE}"
                              sh "docker push ${env.FRONTEND_IMAGE}"
                              
                              echo "INFO: Pushing Backend image: ${env.BACKEND_IMAGE}"
                              sh "docker push ${env.BACKEND_IMAGE}"
                              
                              echo "INFO: Logging out..."
                              sh "docker logout ${env.REGISTRY_HOST}"
                          }
                      }
                  }
              } // End Stage 2.5
              
              // --- Stage 3: Manual Approval Gate ---
              stage('3. CTO Approval') {
                  steps {
                      timeout(time: 1, unit: 'HOURS') { 
                          input message: 'ACTION REQUIRED: Approve deployment to Production (Kubernetes)?',
                                ok: 'Proceed to Deploy',
                                submitter: 'cto'
                      }
                  }
              } // End Stage 3
              
              // --- (THAY THẾ) Stage 4: Deploy to Kubernetes ---
              stage('4. Deploy to Production (Kubernetes)') {
                  steps {
                      echo "INFO: Approval received. Deploying application to Kubernetes cluster..."
                      script {
                          // (CẬP NHẬT) Dùng Kubeconfig (Secret file) đã upload 
                          withKubeconfig(credentialsId: env.K8S_CREDENTIAL_ID, variable: 'KUBECONFIG_FILE') {
                              // Jenkins sẽ tự động trỏ biến KUBECONFIG đến file bí mật đã upload
                              
                              echo "INFO: Applying K8s manifests..."
                              // Chạy kubectl apply cho các file YAML (trong thư mục k8s của repo)
                              sh """
                              kubectl apply -f k8s/namespace.yaml || true 
                              kubectl apply -f k8s/registry-secret.yaml -n ${env.K8S_NAMESPACE}
                              kubectl apply -f k8s/backend-deployment.yaml -n ${env.K8S_NAMESPACE}
                              kubectl apply -f k8s/backend-service.yaml -n ${env.K8S_NAMESPACE}
                              kubectl apply -f k8s/frontend-deployment.yaml -n ${env.K8S_NAMESPACE}
                              kubectl apply -f k8s/frontend-service.yaml -n ${env.K8S_NAMESPACE}
                              """

                              echo "INFO: Waiting for deployments to roll out..."
                              sh "kubectl rollout status deployment/corejs-frontend -n ${env.K8S_NAMESPACE}"
                              sh "kubectl rollout status deployment/corejs-backend -n ${env.K8S_NAMESPACE}"

                              def nodePort = sh(
                                  script: "kubectl get service corejs-frontend-svc -n ${env.K8S_NAMESPACE} -o=jsonpath='{.spec.ports[0].nodePort}'",
                                  returnStdout: true
                              ).trim()
                              
                              echo "----------------------------------------------------"
                              echo "✅ KUBERNETES DEPLOYMENT COMPLETE!"
                              echo "   Access Frontend at: http://:${nodePort}"
                              echo "----------------------------------------------------"
                          } // end withKubeconfig
                      } // end script
                  }
              } // End Stage 4 K8s
          } // End of stages
          
      } // End of pipeline
  ```

**Bước 5: Chạy Pipeline và Truy cập Ứng dụng**
  - Trigger pipeline (push code hoặc `Build Now`).
  - Phê duyệt ở Stage 3.
  - Stage 4 sẽ chạy `kubectl apply`.
  - Sau khi thành công, kiểm tra Console Output để lấy NodePort.
  Truy cập ứng dụng qua trình duyệt bằng địa chỉ: http://: (Ví dụ: http://192.168.1.151:30080).
  <image src ="./33.png">

---

## Giai đoạn 6: Public hệ thống qua Zero trust của Cloudflare
- Truy cập https://one.dashboard.cloudflare.com
- Tìm kiếm zero trust
  <image src="./34.png">
- Vào tạo Tunels
  <image src="./36.png">
- Chọn Cloudflared
  <image src="./37.png">
- Nhập tên tunnel -> Save tunnel
  <image src="./38.png">
- Chọn môi trường để connect
  <image src="./39.png">
- Kiểm tra thấy tunnel HEALTHY là tunnel đã được kích hoạt
   <image src="./40.png">
- Chọn Configure
  <image src="./43.png">
- Chọn Published applcation routes
  <image src="./41.png">
- Điền các thông tin cần thiết
  <image src="./42.png">
  
