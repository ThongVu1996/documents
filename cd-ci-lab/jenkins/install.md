# <center>**Hướng dẫn Cài đặt và Cấu hình Jenkins với Docker Compose** </center>

---

## **1\. Giới thiệu về Jenkins**

---

- **Jenkins** là một công cụ tự động hóa mã nguồn mở, đóng vai trò quan trọng trong quy trình CI/CD (Continuous Integration/Continuous Deployment).

- **Chức năng chính:** Build, Test, Deploy và thông báo kết quả.  
- **Kiến trúc:** Hoạt động theo mô hình **Master \- Agent** (phân tán).  
  - **Master:** 
    - Là trung tâm điều khiển toàn bộ hệ thống Jenkins
    - Chịu trách nhiệm:
    - Quản lý cấu hình và giao diện web
    - Lên lịch và phân phối job tới các agent
    - Lưu trữ kết quả build, logs
    - Cài plugin và quản lý các credentials 
  - **Agent:**
    - Là node (vật lý, VM, container) được kết nối với node Master để thực thi các job build/test/deploy.
    - Mỗi agent có thể cài đặt môi trường riêng: Java, Node.js, Maven, Docker,…
    - Khi job được trigger, Master sẽ phân công cho 1 Agent khả dụng chạy job.
  
---

## **2. Cài đặt Jenkins với Docker Compose**

---

### **1. Cấu trúc thư mục dự án**

```
    ├── gitlab
    │   ├── config
    │   ├── data
    │   └── logs
    ├── gitlab-runner
    │   └── config
    ├── jenkins
    │   ├── agent
    │   └── jenkins_home
    ├── lost+found
    └── setup
        ├── docker-compose.yaml
        └── jenkins
            ├── agent
            │     └── Dockerfile
                └── Dockerfile
```

### **2. Nội dung file docker-compose.yaml**

   ```
        version: '3.8'

        services:
        gitlab:
            image: 'gitlab/gitlab-ce:latest'
            container_name: gitlab
            restart: always
            hostname: 'gitlab.local'
            environment:
            GITLAB_OMNIBUS_CONFIG: |
                gitlab_rails['gitlab_shell_ssh_port'] = 2222
                external_url 'http://gitlab.local'

                ### SMTP config for Gmail
                gitlab_rails['smtp_enable'] = true
                gitlab_rails['smtp_address'] = "smtp.gmail.com"
                gitlab_rails['smtp_port'] = 587
                gitlab_rails['smtp_user_name'] = "diendt@gmail.com"
                gitlab_rails['smtp_password'] = "your_app_password"
                gitlab_rails['smtp_domain'] = "smtp.gmail.com"
                gitlab_rails['smtp_authentication'] = "login"
                gitlab_rails['smtp_enable_starttls_auto'] = true
                gitlab_rails['smtp_tls'] = false

                gitlab_rails['gitlab_email_from'] = 'diendt@gmail.com'
                gitlab_rails['gitlab_email_reply_to'] = 'diendt@gmail.com'
                gitlab_rails['gitlab_email_display_name'] = 'GitLab'
            ports:
            - '80:80'
            - '443:443'
            - '2222:22'
            volumes:
            - '/opt/gitlab/config:/etc/gitlab'
            - '/opt/gitlab/logs:/var/log/gitlab'
            - '/opt/gitlab/data:/var/opt/gitlab'
            networks:
            - gitlab-network

        gitlab-runner:
            image: 'gitlab/gitlab-runner:latest'
            container_name: gitlab-runner
            restart: always
            depends_on:
            - gitlab
            volumes:
            - '/opt/gitlab-runner/config:/etc/gitlab-runner'
            - '/var/run/docker.sock:/var/run/docker.sock'
            networks:
            - gitlab-network

        jenkins:
            build:
            context: ./jenkins
            container_name: jenkins
            restart: unless-stopped
            privileged: true
            user: root
            ports:
            - "8080:8080"   # Jenkins UI
            - "50000:50000" # Agent communication
            volumes:
            - /home/iadmin/.kube:/root/.kube           # mount local kube config
            - /home/iadmin/.minikube:/root/.minikube   # mount minikube certs and keys
            - /opt/jenkins/jenkins_home:/var/jenkins_home
            - /var/run/docker.sock:/var/run/docker.sock # Allows Jenkins to run docker commands
            networks:
            - gitlab-network
            extra_hosts:
            - "gitlab.local:host-gateway"

        jenkins-agent:
            build:
            context: ./jenkins/agent
            container_name: jenkins-agent
            restart: unless-stopped
            privileged: true
            extra_hosts:
            - "jenkins.local:host-gateway"
            environment:
            - JENKINS_URL=http://jenkins:8080
            - JENKINS_AGENT_NAME=docker-agent
            - JENKINS_SECRET=<SECRET>
            - JENKINS_WEB_SOCKET=true
            volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - /opt/jenkins/agent:/home/jenkins/agent
            networks:
            - gitlab-network

        volumes:
        jenkins_home:
            name: jenkins_home

        networks:
        gitlab-network:
            external: true
            name: gitlab-network

   ```

### **3. Cấu hình Dockerfile cho Jenkins Master**

File jenkins/Dockerfile:

```
FROM jenkins/jenkins:lts  
USER root  
   
\# Cài đặt curl, kubectl và docker CLI  
RUN apt-get update \\  
  && apt-get install \-y ca-certificates curl apt-transport-https gnupg2 lsb-release docker.io \\  
  && curl \-fsSL "\[https://dl.k8s.io/release/$(curl\](https://dl.k8s.io/release/$(curl) \-L \-s \[https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl\](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)" \-o /usr/local/bin/kubectl \\  
  && chmod \+x /usr/local/bin/kubectl /usr/bin/docker || true \\  
  && apt-get clean && rm \-rf /var/lib/apt/lists/\*

USER jenkins

### **2.4. Cấu hình Dockerfile cho Jenkins Agent**

File jenkins/agent/Dockerfile:

FROM jenkins/inbound-agent:latest  
USER root

\# Cài đặt docker CLI và kubectl cho Agent  
RUN apt-get update \\  
  && apt-get install \-y ca-certificates curl apt-transport-https gnupg2 lsb-release docker.io \\  
  && curl \-fsSL "\[https://dl.k8s.io/release/$(curl\](https://dl.k8s.io/release/$(curl) \-L \-s \[https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl\](https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl)" \-o /usr/local/bin/kubectl \\  
  && chmod \+x /usr/local/bin/kubectl /usr/bin/docker \\  
  && usermod \-aG docker jenkins || true \\  
  && apt-get clean && rm \-rf /var/lib/apt/lists/\*

USER jenkins
```

---

## **3. Khởi chạy và Thiết lập ban đầu**

### **1. Khởi chạy container**

```
docker compose up \-d
```

### **2. Mở khóa Jenkins (Unlock Jenkins)**

- Truy cập trình duyệt: http://localhost:8080 (hoặc http://jenkins.local:8080 nếu đã cấu hình host).  
- Lấy mật khẩu khởi tạo bằng lệnh:  
   ```
     docker exec \-it jenkins cat /var/jenkins\_home/secrets/initialAdminPassword
   ```

- Dán mật khẩu vào ô **Administrator password** trên màn hình Unlock Jenkins. 
  <image src="./2.png">   
- Chọn **Install suggested plugins** để cài đặt các plugin cơ bản.

- Màn hình đăng nhập
  <image src="./3.png"> 

## **4. Cấu hình sau cài đặt**



### **1. Cài đặt Plugin cần thiết**

Vào **Manage Jenkins** \-\> **Plugins** \-\> **Available Plugins**, tìm và cài đặt các plugin sau:

* Blue Ocean  
* Docker Pipeline  
* Kubernetes CLI  
* Gitlab Plugin  
* Role-based Authorization Strategy
<image src="./1.png">  

### **3. Phân quyền người dùng (RBAC)**

Để quản lý người dùng và quyền hạn (ví dụ: tạo user CTO, Dev):

1. **Bật chế độ bảo mật:**  
   * Vào **Manage Jenkins** \-\> **Security**.  
   * Trong mục **Authorization**, chọn Role-Based Strategy nếu đã cài plugin.  
   * Đảm bảo tích chọn **Enable security**.  
   <image src="./4.png">  
2. **Tạo User mới:**  
   * Vào **Manage Jenkins** \-\> **Users** \-\> **Create User**.  
   * Tạo các user ví dụ: dev, cto.  
   <image src="./5.png">  
   <image src="./6.png">  
   <image src="./7.png">  
3. **Cấp quyền (Role-Based Strategy ):**  
   - Vào **Manage Jenkins** \-\> **Manage and Assign Roles**.  
   <image src="./8.png">
     - admin: tất cả quyền
     - cto: chỉ có quyền (Read, Build, Job -> Discovery, Read, Build, Overall -> Read)
     - dev: chỉ có quyền xem
   - Manage Roles:
     - Global roles
     - Item roles
     - Agent roles
     <image src="./9.png">
     <image src="./10.png">
  - Assign Roles:
     - Global roles
     <image src="./11.png">
     - Item roles
     <image src="./12.png">
     - Agent roles
     <image src="./13.png">