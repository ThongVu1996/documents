# <center>**Tạo EKS**</center>

---

## **I. Tạo IAM và Access Key**
  ### **1. Tạo IAM Account:**
  - Tạo IAM để truy cập cli lẫn web console.
  <image src="./1.png">
  - Tạo User
  <image src="./2-1.png">
  - Tại các options của user
    - User name: chỉ có ký tự thường không có ký tự đặc biêt
    - Tích chọn **Provide user access to AWS Management Console**
    - Tích chonk **Console password** rồi gõ mật khẩu
    - Next
    <image src="./2.png">
  - Chọn Attach policy dirrectly và chọn AdministratorAccess
  <image src="./3.png">
  - Review and create bấm Create user
  <image src="./2-2.png">
  - Return to users list để xem user
  <image src="./4.png">
  - Hiển thị thông báo thành công
  <image src="./5.png">
  ### **2. Tạo token:**
  - Click chuột trái vào user bạn muốn tạo access key.
  <image src="./6.png">
  - Tại màn detail user -> Create access key.
  - Tích chọn Command Line Interface (CLI). -> Next
  <image src="./7.png">
  - Viết mô tả cho access key ở phần desciption và ấn Create
  <image src="./8.png">
  - Lưu lại Access key và Secret access (tải file csv xuống)
  - Ra màn hình sẽ nhìn thấy Access key đã được Active
  <image src="./9.png">

---

## **II. Cài đặt AWS CLI, eksctl, kubectl**
 - Update tìm các phần mềm [tại đây](https://search.nixos.org/packages)
   - awscli2 
   <image src="./11.png">
   - eksctl
   <image src="./12.png">
   - kubectl
   <image src="./13.png">
   - Thêm vào file home.nix
   - Build lại là xong
 - Kiểm tra
   - aws sts get-caller-identity
   - kubectl version --client
   - eksctl version

---

## **III. Cài đặt EKS**
- Bước 1: Thiết lập role EKS Cluster cho user thongdev
  - IAM -> Access management -> Role -> Click Create role
  <image src="./14.png">
  - Lựa chọn các options:
    - Trust entity type: chọn AWS service , 
    - Use case chọn EKS-Cluster 
    <image src="./15.png">
  - Next
    <image src="./16.png">
  - Add Permission để mặc định -> Next
    <image src="./17.png">
  - Name, review, and create
    - Role name: Tên role
    - Ấn Create role
    <image src="./18-1.png">
  - Role tạo thành công
    <image src="./18.png">
- Bước 2: Tạo role cho các worker nodes
  - Cấp quyền cho phép các máy EC2 có thể nghe lệnh từ Cluster
    - Create role 
    - Trusted entity type: Chọn AWS service.
  - Lựa chọn các options:
    - Trust entity type: chọn AWS service , 
    - Use case chọn EC2-EC2 
    <image src="./19.png">
  - Add permissions: Nhấn vào ô tìm kiếm với 3 options
    - AmazonEKSWorkerNodePolicy (Quyền làm công nhân EKS).
    - AmazonEC2ContainerRegistryReadOnly (Quyền tải Docker Image về).
    - AmazonEKS_CNI_Policy (Quyền kết nối mạng IP).
    <image src="./20.png">
    <image src="./21.png">
  - Bấm Next
  - Role name: thongdev_EKS_Worker
    <image src="./22.png">
  - Create
  - Tạo role thành công
    <image src="./23.png">
- Bước 3: Tạo token cho network
  - EC2 -> Network & Security -> Key Pairs -> Create key pair
    <image src="./24.png">
  - Điền vào các options:
    - Name : tên của key nhớ tên này sẽ dùng trong phần script tạo sau này
    - Key pair tyle chọn RSA
    - Private key file format : .pem
    - Ấn create
    <image src="./25-1.png">
  - Key tạo thành công
    <image src="./25.png">
- Bước 4: Tạo cụm EKS bằng eksclt
  - Kiểm tra kết nối bằng `aws configure`
  <image src="./26.png">
  - Tạo file thongdev-cluster-v1.yaml tại `$HOME/Devops/aws-thongdev`
    ```
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: thongdev-lab-cluster
      region: ap-southeast-1
      version: "1.32"

    # =======================================================
    # 1. NETWORKING
    # =======================================================
    vpc:
      # Disable NAT Gateway to save costs (~$30/month).
      # Nodes will use Public IPs for internet access.
      nat:
        gateway: Disable

      clusterEndpoints:
        publicAccess: true
        privateAccess: false

    # =======================================================
    # 2. IAM (CLUSTER ROLE)
    # =======================================================
    iam:
      # ACTION REQUIRED: Replace '241688915712' with YOUR AWS Account ID
      # Click vào role thongdev_EKS_Cluster để lấy 
      serviceRoleARN: "arn:aws:iam::605695176329:role/thongdev_EKS_Cluster"
      withOIDC: true

    # =======================================================
    # 3. ADD-ONS
    # =======================================================
    addons:
      - name: vpc-cni
      - name: coredns
      - name: kube-proxy
      - name: metrics-server

    # =======================================================
    # 4. NODE GROUP (WORKER)
    # =======================================================
    managedNodeGroups:
      - name: student-workers
        instanceType: c7i-flex.large
        amiFamily: Ubuntu2404

        # Scaling configuration
        minSize: 1
        maxSize: 2
        desiredCapacity: 2
        volumeSize: 30

        # SSH Access
        # ACTION REQUIRED: Ensure key pair 'tony-key' exists in EC2
        ssh:
          allow: true
          publicKeyName: thong-key

        # Node IAM Role
        # ACTION REQUIRED: Replace '241688915712' with YOUR AWS Account ID
        iam:
          # Click vào role thongdev_EKS_Worker để lấy 
          instanceRoleARN: "arn:aws:iam::605695176329:role/thongdev_EKS_Worker" 
    ```
  - Đứng tại thư mục `$HOME/Devops/aws-thongdev` chạy lệnh (ở trên máy macOS)
    ```
      eksctl create cluster -f thongdev-cluster-v1.yaml
    ```
  - Tạo thành công
    <image src="./27.png">
  - Kiểm tra
    ```
    kubectl get node
    ```
    <image src="./28.png">