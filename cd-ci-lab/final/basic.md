<center>GIẢI THÍCH CÁC KHÁI NIỆM, LOGIC CƠ BẢN CHO BÀI LAB FINAL</center>

# K8s
## Ingress Nginx / AWS ALB

- Đúng như tên gọi thì nó sẽ có tác dụng như Nginx, đứng trước ứng dụng để nhận dữ liệu từ bên ngoài truyền tới và trả dữ liệu cho từ server tới client. Ngoài ra còn đóng vai trò load blancing.
- Nhưng với Ingress Nginx trong cụm k8s còn có thể quản lý các Pod (một cách tự động, không cần sửa tay như Nginx thường) dù chúng được loại bỏ hoặc thêm mới liên tục -> thay đổi địa chỉ IP liên tục
- Với chức năng load blancing
    - Có thể load theo đường dẫn
        - be-au-svc
        - be-api-svc 
    - Theo phiên bản
        - be-svc-v1
        - be-svc-v2 
    - Theo tên miền
        - api.thongdev.site 
        - admin.thongdev.site 
    - Có thể theo độ ưu tiên (khi có 2 service làm cùng 1 nhiệm vụ, service 1 chết -> chuyển request sang service 2)
## Service
```bash 
    apiVersion: v1
    kind: Service
    metadata:
      name: {{ .Values.backend.name }}-svc
    spec:
      type: ClusterIP
      selector:
        app: {{ .Values.backend.name }}
      ports:
        - name: http
          port: {{ .Values.backend.servicePort }}          # Cổng của Service (ClusterIP)
          targetPort: {{ .Values.backend.containerNginxPort }}    # Khớp với containerPort của Nginx trong Deployment
          protocol: TCP
```
- Tiếp nhận request từ Ingres Nginx chuyển tới, để làm được điều đó thì file config sẽ có 1 cái tên ở mục `metadata`, với các tên được khai báo nó có thể dễ dàng giao tiếp với Nginx qua tên hoặc giao tiếp với các service khác (giống như service trong docker, giao tiếp với nhau qua tên vì nó đã được ghi vào DNS nội bộ).
- Sau khi tiếp nhận được, request sẽ được truyền tới các Pod thuộc quản lý của Service (ở đây đóng vai trò Loading Blancing nội bộ) để xử lý.
   - Có thể load theo đường dẫn
        - be-au-svc
        - be-api-svc 
    - Theo phiên bản
        - be-svc-v1
        - be-svc-v2 
    - Theo tên miền
        - api.thongdev.site 
        - admin.thongdev.site 
    - Có thể theo độ ưu tiên (khi có 2 service làm cùng 1 nhiệm vụ, service 1 chết -> chuyển request sang service 2)

- Service sẽ quản lý bất kì Pod nào có `lable` có tên được với `app` được ghi trong `selector` ở file config
## Deployment
```bash
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: {{ .Values.frontend.name }}
    spec:
      replicas: {{ .Values.frontend.replicaCount }}
      selector:
        matchLabels:
          app: {{ .Values.frontend.name }}
      template:
        metadata:
          labels:
            app: {{ .Values.frontend.name }}
        spec:
          containers:
            - name: frontend
              image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
              imagePullPolicy: Always
              ports:
                - containerPort: {{ .Values.frontend.containerPort }}
              volumeMounts:
                # Ghi đè file config.js giả lập vào container
                - name: config-volume
                  mountPath: /usr/share/nginx/html/config.js
                  subPath: config.js
          volumes:
            - name: config-volume
              configMap:
                name: {{ .Values.frontend.name }}-config

```
- Nếu service quản lý các pod thì đây chính là nơi quy định cấu hình và `lable` cho pod.
- Quản lý các pod có `lable` có tên giống với thuộc tính `spec.selector.matchLabels.app`.
- Quy định sẽ có bao nhiêu pod (bằng `replicas`) sẽ được tạo ra và duy trì (Pod ra lệnh cho K8s tạo lại).
- Dán nhãn cho pod bằng thuộc tính `template.metadata.lables.app`, giúp quản lý số lượng pod và service dựa vào labels này để biết pod nào thuộc quyển quản lý để có chuyển request tới.
- Quy định spec cho các container thuộc pod qua `templete.spec.containers`
