# <center>**Triển Khai Helm Chart Thông qua ArgoCD Kết hợp với Nginx Load Balangcer trên EKS**</center>

---

## **I. Tạo chứng chỉ SS**
  - Nếu chưa tạo chứng chỉ thì làm như sau (ở đây là trường hợp tên miền được quản lý bằng CloudFlare)
    - Tìm kiếm và truy cập dịch vụ Certificate Manager
    <image src="./35.png">
    - Chọn đúng khu vực
    - Ấn Request
    <image src="./36.png">
    - Chọn Request a public certificate
    <image src="./37.png">
    - Next
    - Request public certificate
      - Domain names: Nhập *.ten_domain
      <image src="./43.png">
    - Ấn Request
    - Sau đó chứng chỉ được tạo ra ở trạng thái pending
    <image src="./38.png">
    - Vào cloudFlare -> DNS -> Records -> Add Record -> Chọn type CNAME
    <image src="./39.png">
    - Name và Targer điền điển CNAME name và CNAME value của chứng chỉ AWS
    <image src="./40.png">
    - Tắt Proxy Status
    - Save
    - Được thành quả như hình
    <image src="./41.png">
    - Kiểm tra lại chứng chỉ trên AWS, thấy trạng thái Success
    <image src="./42.png">
  - Lấy chứng chỉ bằng câu lệnh (Nếu tạo rồi)
    ```
      aws acm list-certificates --region ap-southeast-1
    ```
    <image src="./44.png">
  - Cài đặt helm
    ```
      helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      helm repo update
    ```
    ```
      helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-type"="nlb" \
  --set controller.service.annotations."service.beta.kubernetes.io/aws-load-balancer-scheme"="internet-facing" \
  --set controller.service.annotations."service.beta.kubernetes.ioaws-load-balancer-ssl-cert"="arn:aws:acm:ap-southeast-1:605695176329:certificate/e96fa410-a0d4-4cc2-875e-7b20dbff4786" \
  --set controller.service.annotations."service.beta.kubernetes.ioaws-load-balancer-ssl-ports"="443" \
  --set controller.service.targetPorts.https=http
    ```
    <image src="./45.png">
    <image src="./46.png">

---

## **II. Đưa ARGOCD public ra ngoài bằng ALB INGRESS**
 -  Kích hoạt OIDC
   ```
      eksctl utils associate-iam-oidc-provider --cluster thongdev-lab-cluster --region ap-southeast-1 --approve
   ```
   <image src="./47.png">
- Gắn Tag cho Subnet Public (BẮT BUỘC)
  - Vào AWS Console -> VPC -> Subnets
  - Chọn các subnet public
  <image src="./48.png">
  - Click vào Subnet ID -> Màn đetail -> Action -> Manage tags
    <image src="./49.png">
    - Kiểm tra xem đã có tag kubernetes.io/role/elb với Value: 1, nếu chưa thì thêm vào
    <image src="./50.png">
  - Đảm bảo rằng Security group trong EC2 đã được set cho phép port 8080 truy cập từ 0.0.0.0
    - Chọn cái nào có tên dạng **eks-cluster-sg-**
    <image src="./54.png">
    -  Nhấn vào ID của nó
    -  Vào Edit Inbound Rules
    <image src="./51.png">
    -  Và thêm như hình
     <image src="./52.png">

- Thiết lập quyền hạn (IAM & SERVICE ACCOUNT)
  - Tải Policy Chuẩn Mới Nhất
    ```
      curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
    ```
  - Tạo IAM Policy
    ```
      aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy_Final --policy-document file://iam_policy.json
    ```
  - Tạo Service Account (Kết nối vào K8s)
    ```
      eksctl create iamserviceaccount \
      --cluster=thongdev-lab-cluster \
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --role-name AmazonEKSLoadBalancerControllerRole_Final2 \
      --attach-policy-arn=arn:aws:iam::605695176329:policy/AWSLoadBalancerControllerIAMPolicy_Final \
      --region=ap-southeast-1 \
      --override-existing-serviceaccounts \
      --approve
    ```
    <image src="./55.png">
---

## **III. Cài đặt CONTROLLER (HELM)**
- Cài Helm vào 
  - Thêm `kubernetes-helm` vào home.nix
  - Build lại
- Thêm kho chứa (Repo) và cài đặt Controller bằng HELM
  ```
      helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
      -n kube-system \
      --set clusterName=thongdev-lab-cluster \
      --set serviceAccount.create=false \
      --set serviceAccount.name=aws-load-balancer-controller \
  ```
- Kiểm tra xem nó sống chưa
  ```
    kubectl get deployment -n kube-system aws-load-balancer-controller
  ```

    ![check-alive](/Users/thongvu/DevOps/documents/cd-ci-lab/aws/check-alive.png)

## **IV. Kích hoạt INGRESS**
- Tạo argocd-ingress.yaml
  ```
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-server-ingress
      namespace: argocd
      annotations:
        # 1. Controller Designation
        # Specifies that this Ingress is managed by the AWS Load Balancer Controller.
        kubernetes.io/ingress.class: alb

        # 2. Network Scheme
        # 'internet-facing': Creates a public Load Balancer accessible from the internet.
        # 'internal': Creates a private Load Balancer (internal VPC only).
        alb.ingress.kubernetes.io/scheme: internet-facing

        # 3. Target Type
        # 'ip': Routes traffic directly to the Pod IP (High Performance & Faster).
        # 'instance': Routes traffic to NodePort (Legacy mode).
        alb.ingress.kubernetes.io/target-type: ip

        # --- SSL/HTTPS CONFIGURATION ---
        
        # 4. ACM Certificate
        # ACTION REQUIRED: Replace the ARN below with your actual Certificate ARN from AWS ACM.
        alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-southeast-1:241688915712:certificate/70c58476-9a59-4bdc-b1df-cca71c88963a

        # 5. Listen Ports
        # Configures the Load Balancer to listen on both HTTP (80) and HTTPS (443).
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'

        # 6. SSL Redirection
        # Automatically redirects all HTTP traffic to HTTPS for security.
        alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'

        # --- BACKEND PROTOCOL (CRITICAL FOR ARGOCD) ---
        
        # 7. Backend Protocol
        # We use 'HTTP' here because we have disabled internal HTTPS on the ArgoCD Pod 
        # (using the --insecure flag) to avoid "Login Loop" issues and simplify SSL termination.
        alb.ingress.kubernetes.io/backend-protocol: HTTPS

        # 8. Health Check Settings
        # Since the backend speaks HTTP, the health check must also use HTTP.
        alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
        # ArgoCD exposes a specific health check endpoint at /healthz.
        alb.ingress.kubernetes.io/healthcheck-path: /healthz
        # Codes considered "Healthy" by the Load Balancer.
        alb.ingress.kubernetes.io/success-codes: '200'

    spec:
      # Standard Ingress Class definition
      ingressClassName: alb
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: argocd-server
                    port:
                      # This must match the HTTP port exposed by the argocd-server Service.
                      number: 80
  ```
- Apply
    ```
    kubectl apply -f argocd-ingress.yaml
    ```
- Reset Controller
    ```
      kubectl delete pod -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
    ```
    
    ![reset-controller](/Users/thongvu/DevOps/documents/cd-ci-lab/aws/reset-controller.png)
## **V. Nghiệm thu**
  ```
    kubectl get ingress -n argocd
  ```
  
   ![nghiem-thu](/Users/thongvu/DevOps/documents/cd-ci-lab/aws/58.png)
