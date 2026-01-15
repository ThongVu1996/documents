# <center>**CI/CD với Gitlab** </center>

---

### **1. Tạo PAT từ Gitlab**
  - Tạo PAT có quyền với các Repo trên gitlab
  - Vào biểu tưởng User -> Edit Profile
  <image src="./23.png">
  - Vào **Personal Access Token (PAT)** -> chọn Add new token -> chọn các options (read_repository, write_repository, read_api)
  <image src="./24.png">
  - Lưu lại token vào 1 nơi vì chỉ cấp 1 lần duy nhất
  <image src="./25.png">
  
### **2. Cấu hình Jenkins**
  - Tạo Credentials
    - Manage Jenkins -> Credentials -> Global
    <image src="./36.png">
    - Username + Password Credentials
      <image src="./37.png">
      - Kind: Gitlab API token
      - API Token: <PAT>
      - ID: gitlab-cred
  - Tạo pipeline
    - New Item -> Nhập tên -> Chọn Pipeline -> OK
    <image src="./31.png">
    - Cẩu hình trigger để nhận biết gitlab có commit mới
    <image src="./33.png">
  - Cấu hình pipe line
    - Pipeline script nhập luôn pipeline script tại Jenkins
    - Pipeline script from SCM: Pipeline script được lưu ở git
      - Chọn Git trong SCM -> Nhập Repository URL (http://gitlab.local/thongdev.site/corejs) -> Chọn Credentials -> Nhập Braches (nodejs) to build hoặc để ** để build với mọi branches -> Nhập đường dẫn Jenkinsfile
      <image src="./32.png">

### **3. Thêm webhook**
  - Gitlab Project -> Settings -> Webhooks -> Add new webhook
  <image src="./26.png">
  - Cấu hình thêm trong admin để có thể chạy mà không lỗi **URL is invalid** khi thêm webhook
    -  Admin -> setting -> network -> Outbound requests -> chọn các options như hình
    <image src="./27.png">
      - URL: http://gitlab.thongdev.site/project + tên project
      - Secret sẽ lấy ở trong phần cấu hình jenkins
    - Add new webhook (Chọn push event, bỏ tích enable SSL)
    <image src="./28.png"> 
    <image src="./29.png">
    - Vào nút test -> chọn push event -> nhận được kết quả 200 là thành công
    <image src="./30.png">

### **4. Kiểm tra**
  - Thêm mới 1 commit và push lên git
  - Vào trong item vừa chạy và kiểm tra kết quả
  <image src="./38.png">