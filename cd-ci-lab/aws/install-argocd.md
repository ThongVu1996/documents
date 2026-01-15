# <center>**Triển Khai ArgoCD Trên AWS**</center>

---

## **I. Cài đặt ArgoCD**
- Bước 1: Cài đặt
  - Mở CMD ở thư mục `$HOME/Devops/aws-thongdev` chạy các lệnh (ở trên máy macOS)
    ```
      kubectl create namespace argocd
      kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
    <image src="./29.png">
    <image src="./30.png">
  - Kiểm tra pod argocd
    ```
       kubectl get pods -n argocd
    ```
    <image src="./31.png">

---

## **II. Đăng nhập vào ArgoCD**
- Bước 1: Lấy mật khẩu Admin
  ```
    # 1. Retrieve the Base64-encoded password from the Kubernetes secret
    # and store it in a file (this step is identical on Mac/Linux)
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" > secret.txt

    # 2. Decode the Base64 content of secret.txt into a new file
    # The 'base64' command with the -d (decode) flag is the Unix/Mac equivalent of 
    base64 -d -i secret.txt > decoded.txt

    # 3. Display the contents of the decoded file
    # The 'cat' command is the Unix/Mac equivalent of 'type'
    cat decoded.txt
  ```
  <image src="./32.png">

- Bước 2: “Đào hầm” truy cập (Port Forwarding)
  ```
     kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
  <image src="./33.png">
  - Giải thích: bước này để ta chạy trực tiếp ArgoCD trên localhost như bên dưới đã login thành công với user admin / password là pass vừa lấy được từ file decoded.txt
  <image src="./34.png">