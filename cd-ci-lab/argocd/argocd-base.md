# <center>**Triển khai CI/CD K8S với Jenkins, ArgoCD**</center>

---

## **1. Luồng hoạt động**

```
                👨‍💻 Developer push code mới lên GitLab (nhánh `argocd-base`).

                                    ⬇️

                📡 GitLab tự động gửi Webhook cho **Jenkins**.

                                    ⬇️

                👷‍♂️ Jenkins nhận code, build **Docker images** (frontend & backend).

                                    ⬇️

                📦 Jenkins push images lên GitLab Registry.

                                    ⬇️

                ⏸️ Jenkins dừng lại, chờ phê duyệt từ CTO.

                                    ⬇️

                🧑‍💼 CTO đăng nhập Jenkins và nhấn "Approve".

                                    ⬇️

                🚀 Jenkins push registry để lên gitlab.

                                    ⬇️

                🔍 ArgoCD Phát Hiện Thay Đổi
                (ArgoCD polling Git repository (mỗi 3 phút), phát hiện commit mới từ Jenkins)
                
                                    ⬇️

                🚀 ArgoCD Auto-Sync to Kubernetes
                (
                  ✅ Apply manifests: kubectl apply -f deployment.yaml
                  🔄 Rolling Update: Pods được update với image mới
                  💚 Health Check: ArgoCD kiểm tra pods healthy
                )

```

---

## **2. Yêu cầu chuẩn bị (Prerequisites)**

  - Chúng ta đã có cụm k8s, gitlab, dns, npm, jenkins từ bài [all-in-one](https://github.com/ThongVu1996/cd-ci-lab/blob/master/all-in-one/all-in-one.md)

- Chúng ta đã cài thành công Argocd theo hướng dẫn [tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/argocd/install.md)

---

## **3. Triển khai**

- Như đã đề cập ở [Luồng hoạt đông](#1-luồng-hoạt-động) thì argocd sẽ lấy code từ git, nhưng đây là bài lab nội bộ gitlab được self-host trên máy local, đươc phân giải bằng DNS nội bộ, forward bằng npm nội bộ.
  
- Nên chúng ta sẽ gặp lỗi đó **trên máy chủ (Host) thì truy cập được, nhưng trong Pod (Container) lại không?**
  
  ```
    ArgoCD -> CoreDNS -> Upstream DNS
  ```
  
- Nhìn vào luồng hoạt động ở trên ta thấy, nó hỏi CoreDNS nhưng không biết tên miền của git, nó tiếp tục hỏi đến bên ngoài. Nhưng vì đó là link nội bộ thì bên ngoài internet cũng không biết
  
- Chính vì vậy chúng ta sẽ dùng Zero Trust của CloudFlare như [hướng dẫn tại dây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/all-in-one/all-in-one.md#giai-%C4%91o%E1%BA%A1n-6-public-h%E1%BB%87-th%E1%BB%91ng-qua-zero-trust-c%E1%BB%A7a-cloudflare) để public link git lên internet (với ip sẽ là `192.168.1.32:80`)

  ### **3.1 Update Jenkins file**
     - Tạo 1 nhánh tên là `argocd-base`
     - Sửa Jenkins file thành như sau
  
     ```
        pipeline {
            agent any 

            environment {
                // --- Application & Image Naming ---
                APP_NAME            = 'corejs'
                REGISTRY_HOST       = 'register.thongdev.site'
                GITLAB_PROJECT_PATH = 'tonylab/corejs' 

                // 1. Định nghĩa Tag động (Quan trọng cho GitOps)
                IMAGE_TAG           = "v${env.BUILD_NUMBER}"

                // 2. Định nghĩa tên Image (Bỏ chữ :latest đi để ghép chuỗi cho dễ)
                FRONTEND_IMAGE      = "${env.REGISTRY_HOST}/${env.GITLAB_PROJECT_PATH}/frontend"
                BACKEND_IMAGE       = "${env.REGISTRY_HOST}/${env.GITLAB_PROJECT_PATH}/backend"
                
                // 3. Tách biệt Credential theo đúng ý bạn
                // Nhớ tạo 2 ID này trong Jenkins (dù bên trong có thể chứa cùng 1 PAT hoặc 2 PAT khác nhau tùy bạn)
                REGISTRY_CREDENTIAL_ID = 'gitlab-registry-creds' 
                REPO_CREDENTIAL_ID     = 'gitlab-repository-creds' 

                // 4 Tên nhánh để code up lên
                BRANCH_NAME= 'argocd-base'
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
                                    // Build kèm Tag version
                                    echo "INFO: Building Frontend: ${env.FRONTEND_IMAGE}:${env.IMAGE_TAG}"
                                    sh "docker build -t ${env.FRONTEND_IMAGE}:${env.IMAGE_TAG} ." 
                                }
                            }
                        }
                        stage('Build Backend') {
                            steps {
                                dir('CoreAPI') {
                                    // Build kèm Tag version
                                    echo "INFO: Building Backend: ${env.BACKEND_IMAGE}:${env.IMAGE_TAG}"
                                    sh "docker build -t ${env.BACKEND_IMAGE}:${env.IMAGE_TAG} ."
                                }
                            }
                        }
                    } 
                } 
                
                // --- Stage 2.5: Push Images (Dùng quyền Registry) ---
                stage('2.5. Push Images to Registry') {
                    steps {
                        script {
                            // CHỈ DÙNG ID CỦA REGISTRY TẠI ĐÂY
                            withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIAL_ID, passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                                
                                echo "INFO: Logging in to Registry..."
                                // User/Pass ở đây được lấy từ 'gitlab-registry-creds'
                                sh "docker login -u ${REG_USER} -p ${REG_PASS} ${env.REGISTRY_HOST}"
                                
                                echo "INFO: Pushing Images version ${env.IMAGE_TAG}..."
                                sh "docker push ${env.FRONTEND_IMAGE}:${env.IMAGE_TAG}"
                                sh "docker push ${env.BACKEND_IMAGE}:${env.IMAGE_TAG}"
                                
                                sh "docker logout ${env.REGISTRY_HOST}"
                            }
                        }
                    }
                } 
                
                // --- Stage 3: Manual Approval ---
                stage('3. CTO Approval') {
                    steps {
                        timeout(time: 1, unit: 'HOURS') { 
                            input message: "Approve deployment of version ${env.IMAGE_TAG}?",
                                  ok: 'Proceed to Update Git',
                                  submitter: 'cto'
                        }
                    }
                }
                
                // --- Stage 4: Update Manifest (Dùng quyền Repository) ---
                stage('4. Update Manifest in Git') {
                    steps {
                        script {
                            // CHỈ DÙNG ID CỦA REPO TẠI ĐÂY
                            withCredentials([usernamePassword(credentialsId: env.REPO_CREDENTIAL_ID, usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                                
                                // Cấu hình danh tính cho Bot commit
                                sh "git config user.email 'jenkins-bot@thongdev.site'"
                                sh "git config user.name 'Jenkins Bot'"
                                
                                dir('k8s') {
                                    echo "INFO: Updating manifests to version ${env.IMAGE_TAG}"
                                    
                                    // --- CẬP NHẬT VERSION TRONG FILE YAML ---
                                    
                                    // Logic: Tìm dòng chứa tên image, thay thế phần sau dấu : bằng tag mới
                                    // Frontend
                                    sh """
                                        sed -i 's|${env.FRONTEND_IMAGE}:.*|${env.FRONTEND_IMAGE}:${env.IMAGE_TAG}|g' frontend-deployment.yaml
                                    """
                                    
                                    // Backend
                                    sh """
                                        sed -i 's|${env.BACKEND_IMAGE}:.*|${env.BACKEND_IMAGE}:${env.IMAGE_TAG}|g' backend-deployment.yaml
                                    """
                                    
                                    // Kiểm tra log (tùy chọn)
                                    sh "grep 'image:' *.yaml"
                                    
                                    // --- COMMIT VÀ PUSH ---
                                    sh "git add ."
                                    sh "git commit -m 'Jenkins update image to version ${env.IMAGE_TAG}'"
                                    
                                    // Dùng GIT_USER/GIT_PASS từ 'gitlab-repository-creds' để xác thực lệnh Push
                                    sh "git push https://${GIT_USER}:${GIT_PASS}@gitlab.thongdev.site/${env.GITLAB_PROJECT_PATH}.git HEAD:refs/heads/${BRANCH_NAME}"
                                }
                            }
                        }
                    }
                } 
            } // End of stage
            
          // Notification
            post {
                success {
                    script {
                        echo "✅✅✅ PIPELINE SUCCESS ✅✅✅"
                        echo "Git Manifest đã được cập nhật lên version: ${env.IMAGE_TAG}"
                        echo "ArgoCD sẽ tự động đồng bộ sau vài phút."
                        
                        // VÍ DỤ GỬI EMAIL (Nếu cài Plugin Email Extension)
                        // mail to: 'admin@thongdev.site',
                        //      subject: "Deployment Success: Build #${env.BUILD_NUMBER}",
                        //      body: "Code mới (${env.IMAGE_TAG}) đã được đẩy lên Git thành công. ArgoCD đang deploy."

                        // VÍ DỤ GỬI SLACK (Nếu cài Plugin Slack Notification)
                        // slackSend (color: '#36a64f', message: "SUCCESS: Pipeline ${env.JOB_NAME} [${env.BUILD_NUMBER}] finished successfully.")
                    }
                }
                failure {
                    script {
                        echo "❌❌❌ PIPELINE FAILED ❌❌❌"
                        echo "Có lỗi xảy ra trong quá trình Build hoặc Push."
                        echo "Vui lòng kiểm tra Console Log."
                        
                        // mail to: 'admin@thongdev.site',
                        //      subject: "Deployment FAILED: Build #${env.BUILD_NUMBER}",
                        //      body: "Vui lòng kiểm tra Jenkins gấp!"
                        
                        // slackSend (color: '#FF0000', message: "FAILED: Pipeline ${env.JOB_NAME} [${env.BUILD_NUMBER}] encountered an error.")
                    }
                }
                aborted {
                    echo "⚠️ Pipeline đã bị hủy (Aborted) bởi người dùng hoặc quá thời gian chờ (Timeout)."
                }
            } // End of post
        }

     ```

    - Ở đây ta cần chú ý **REGISTRY_CREDENTIAL_ID = 'gitlab-registry-creds',
      REPO_CREDENTIAL_ID     = 'gitlab-repository-creds'**, đây là 2 token. được tạo ra (xem cách tạo [tại đây](https://github.com/ThongVu1996/cd-ci-lab/blob/master/all-in-one/all-in-one.md)) lần lượt với các quyền tương tác với repo và registry
    - Ngoài ra cần chỉ định đúng nhánh bằng biến **BRANCH_NAME**  
  
  ### 3.2 Tạo kết nối tới repo gitlab
  - Vì repo trên gitlab là private, nên cần tạo kết nối để giúp ArgoCD pull code từ gitlab về bằng access-toke
  - Vào `Settings` -> `+CONNECT REPO`

    ![Tạo connect repo](/Users/thongvu/DevOps/documents/cd-ci-lab/argocd/26.png)
- Điền cá thống tin như hình

    ![Thông tin kết nối Repo](/Users/thongvu/DevOps/documents/cd-ci-lab/argocd/27.png)
  - Điền password là access token có quyền truy cập đến repo (ở trên đã nói cách tạo)
  - Vào như hình để tạo app
  - Nếu mà repo là public thì bỏ qua bước 3.2 này
  ### **3.3 Tạo app trên Argocd**
    - Vào Applications -> NEW APP
      
    - Điền các thông tin sau:
      - Application name: chỉ bao gồm chữ thường
      - Project Name: default
      - Sync Policy: Automatic và tích chọn **Enable Auto-Sync**, **Self Heal**
      - Repository URL: chọn options GIT và điền link repo nhớ có .git (eg: https://gitlab.thongdev.site/tonylab/corejs.git)
      - Revision: chọn options Branches, điền tên branh, chỉ đúng tên branch thôi nha (eg:argocd-base)
      - PATH: đường dãn tới file manifest (eg: ./k8s)
      - DESTINATION: chọn Cluster URL -> https://kubernetes.default.svc
      <image src="./21.png">
      <image src="./22.png">
      <image src="./23.png">
    - Ấn Create
    - Thành công sẽ có như hình
      <image src="./24.png">
      <image src="./25.png">


