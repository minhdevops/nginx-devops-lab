# Nginx Demo — Docker + Helm + ArgoCD

Project dùng để kiểm thử luồng triển khai trong lab:

```text
Source code → Docker image → Docker Hub → Helm Chart → ArgoCD → Kubernetes
```

Thông số mặc định:

| Thành phần | Giá trị |
|---|---|
| Docker image | `dovanminh98/nginx-demo:0.0.1` |
| Container port | `8080` |
| Service port | `80` |
| Namespace | `lab` |
| Domain lab | `nginx-demo.lab.local` |
| Health check | `/healthz` |

## Cấu trúc project

```text
nginx-demo/
├── src/                         # Trang web demo
├── nginx/default.conf           # Nginx chạy port 8080
├── Dockerfile
├── Jenkinsfile
├── helm/nginx-demo/             # Helm chart chạy độc lập
├── argocd/application.yaml      # ArgoCD Application độc lập
└── integration-existing-chart/  # File chèn vào k8s-lab-webapp
```

## Cách 1 — Chạy độc lập thành Application thứ ba

### Bước 1: Test Docker trên máy Windows hoặc Jenkins

```bash
docker build -t dovanminh98/nginx-demo:0.0.1 .
docker run --rm -p 8080:8080 dovanminh98/nginx-demo:0.0.1
```

Truy cập `http://localhost:8080` và kiểm tra health:

```bash
curl http://localhost:8080/healthz
```

### Bước 2: Push image

```bash
docker login
docker push dovanminh98/nginx-demo:0.0.1
```

### Bước 3: Push project lên GitHub

Tạo repository `nginx-demo` tại tài khoản `minhdevops`, sau đó:

```bash
git init
git branch -M main
git add .
git commit -m "Initial nginx demo"
git remote add origin https://github.com/minhdevops/nginx-demo.git
git push -u origin main
```

Nếu tên repository khác, sửa `repoURL` trong `argocd/application.yaml`.

### Bước 4: Kiểm tra Helm trên k8s-master

```bash
helm lint helm/nginx-demo
helm template nginx-demo helm/nginx-demo
```

### Bước 5: Tạo ArgoCD Application

Chạy trên `k8s-master`:

```bash
kubectl apply -f argocd/application.yaml
kubectl get application nginx-demo -n argocd
```

Nếu chưa bật Auto-Sync, mở ArgoCD, chọn `nginx-demo`, sau đó `SYNC` →
`SYNCHRONIZE`.

### Bước 6: Kiểm tra Kubernetes

```bash
kubectl get deployment,pod,service,ingress -n lab
kubectl logs deployment/nginx-demo -n lab --tail=100
curl http://nginx-demo.lab.local/healthz
```

Trên Windows, mở bằng quyền Administrator và thêm vào file
`C:\Windows\System32\drivers\etc\hosts`:

```text
192.168.0.10 nginx-demo.lab.local
```

Sau đó truy cập `http://nginx-demo.lab.local`.

## Cách 2 — Thêm node vào chart k8s-lab-webapp hiện tại

Cách này không tạo Application thứ ba. Nginx sẽ xuất hiện bên trong cây resource
của Application `k8s-lab-webapp`.

1. Copy nội dung `integration-existing-chart/values-snippet.yaml` vào cuối file
   `helm/k8s-lab-webapp/values.yaml` của repository `k8s-lab-webapp`.
2. Copy ba file trong `integration-existing-chart/templates/` vào
   `helm/k8s-lab-webapp/templates/`.
3. Kiểm tra chart:

```bash
helm lint helm/k8s-lab-webapp
helm template k8s-lab-webapp helm/k8s-lab-webapp
```

4. Commit và push:

```bash
git add helm/k8s-lab-webapp
git commit -m "feat: add nginx-demo to existing chart"
git push origin main
```

5. Mở ArgoCD → `k8s-lab-webapp` → `REFRESH` → `SYNC`.

Kiểm tra:

```bash
kubectl get pod,service,ingress -n lab \
  -l app.kubernetes.io/name=nginx-demo
```

## Jenkins

`Jenkinsfile` thực hiện:

1. Build image `dovanminh98/nginx-demo:0.0.BUILD_NUMBER`.
2. Push image lên Docker Hub.
3. Cập nhật `helm/nginx-demo/values.yaml`.
4. Commit và push tag mới để ArgoCD tự đồng bộ.

Credential Jenkins cần có:

| Credential ID | Loại | Mục đích |
|---|---|---|
| `dockerhub-creds` | Username with password | Push Docker Hub |
| `github-creds` | Username with token | Push thay đổi Helm lên GitHub |

Jenkins agent phải có `docker`, có quyền truy cập Docker daemon và có Git.

## Xử lý lỗi nhanh

```bash
kubectl get pods -n lab
kubectl describe pod <ten-pod> -n lab
kubectl logs <ten-pod> -n lab
kubectl describe ingress nginx-demo -n lab
kubectl get ingressclass
```

- `ImagePullBackOff`: kiểm tra image/tag, Docker Hub và `imagePullSecrets`.
- `CrashLoopBackOff`: xem log Pod và kiểm tra Nginx config.
- Ingress không truy cập được: kiểm tra Ingress Controller, `ingressClassName`,
  DNS/hosts và IP `192.168.0.10`.
- ArgoCD vẫn dùng tag cũ: kiểm tra `values.yaml`, `valueFiles` và Helm parameters
  trên Application.
