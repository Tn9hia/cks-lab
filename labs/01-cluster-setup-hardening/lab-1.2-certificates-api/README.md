# Lab 1.2 — Certificates API (CSR) để cấp quyền cho user mới

**Domain:** Cluster Hardening — TLS/Certificates, Authentication
**Thời gian gợi ý:** 20-25 phút
**Yêu cầu:** `kubectl` đã trỏ tới cụm, `openssl` trên máy làm việc.

## Bối cảnh

Kubernetes không có khái niệm "User" lưu trong etcd như ServiceAccount. Một
cách chuẩn để cấp quyền cho một người dùng thật (hoặc hệ thống ngoài) là dùng
client certificate ký bởi CA của cluster, thông qua flow
`CertificateSigningRequest` (CSR).

## Bước 1 — Tạo key + CSR cho user `jane`

```bash
mkdir -p /tmp/jane-cert && cd /tmp/jane-cert

openssl genrsa -out jane.key 2048

# CN = username, O = group (dùng cho RBAC group binding nếu cần)
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane/O=developers"
```

## Bước 2 — Tạo đối tượng CertificateSigningRequest trên cluster

```bash
cat manifests/csr.yaml.tmpl | \
  sed "s/BASE64_CSR/$(cat /tmp/jane-cert/jane.csr | base64 | tr -d '\n')/" \
  > /tmp/jane-cert/csr.yaml

kubectl apply -f /tmp/jane-cert/csr.yaml
kubectl get csr
```

Trạng thái sẽ là `Pending`.

## Bước 3 — Approve CSR (người có quyền admin làm bước này)

```bash
kubectl certificate approve jane-csr
kubectl get csr jane-csr -o jsonpath='{.status.conditions[0].type}'
# -> Approved
```

## Bước 4 — Lấy certificate đã ký

```bash
kubectl get csr jane-csr -o jsonpath='{.status.certificate}' | base64 -d > /tmp/jane-cert/jane.crt
openssl x509 -in /tmp/jane-cert/jane.crt -text -noout | grep -A1 "Subject:"
```

## Bước 5 — Gán RBAC cho user `jane`

CSR chỉ xác thực (authentication) danh tính `jane`, chưa cấp quyền
(authorization). Áp dụng RBAC:

```bash
kubectl create namespace blue
kubectl apply -f manifests/jane-rbac.yaml
```

`jane-rbac.yaml` cấp cho user `jane` quyền `create/get/list` (không có
`delete`) trên `deployments` trong namespace `blue` — đúng tình huống Phần C
của Lab 1.1.

## Bước 6 — Tạo kubeconfig riêng cho jane và verify

```bash
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath='{.clusters[0].name}')
SERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
kubectl config view --minify --flatten \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > /tmp/jane-cert/ca.crt

kubectl config set-cluster $CLUSTER_NAME --server=$SERVER \
  --certificate-authority=/tmp/jane-cert/ca.crt --embed-certs=true \
  --kubeconfig=/tmp/jane-cert/jane.kubeconfig

kubectl config set-credentials jane \
  --client-certificate=/tmp/jane-cert/jane.crt \
  --client-key=/tmp/jane-cert/jane.key \
  --embed-certs=true \
  --kubeconfig=/tmp/jane-cert/jane.kubeconfig

kubectl config set-context jane-context \
  --cluster=$CLUSTER_NAME --namespace=blue --user=jane \
  --kubeconfig=/tmp/jane-cert/jane.kubeconfig

kubectl config use-context jane-context --kubeconfig=/tmp/jane-cert/jane.kubeconfig

# Test bằng chính kubeconfig của jane
kubectl --kubeconfig=/tmp/jane-cert/jane.kubeconfig create deployment nginx --image=nginx -n blue
# -> phải thành công

kubectl --kubeconfig=/tmp/jane-cert/jane.kubeconfig delete deployment nginx -n blue
# -> phải bị Forbidden
```

## Bước 7 — Quản lý vòng đời CSR

```bash
# Xem tất cả CSR đang chờ duyệt
kubectl get csr

# Từ chối một CSR đáng ngờ
kubectl certificate deny <tên-csr>

# Xoá CSR đã xử lý xong (không tự dọn)
kubectl delete csr jane-csr
```

## Dọn dẹp

```bash
kubectl delete namespace blue
kubectl delete csr jane-csr --ignore-not-found
rm -rf /tmp/jane-cert
```

## Mẹo thi

- CSR mặc định dùng `signerName: kubernetes.io/kube-apiserver-client` cho
  client cert xác thực user — nhớ chính xác tên signer này, sai là CSR sẽ
  không được ký.
- Certificate hết hạn mặc định sau khi approve dùng `expirationSeconds`
  (nếu không set, dùng giá trị mặc định của signer).
- Trong đề thi, phần này hay được hỏi kèm RBAC: cấp cert xong luôn phải nhớ
  tạo Role/RoleBinding, nếu không user mới chỉ "xác thực được" nhưng làm gì
  cũng bị 403.
