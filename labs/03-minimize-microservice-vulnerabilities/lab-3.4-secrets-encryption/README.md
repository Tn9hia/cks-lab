# Lab 3.4 — Encrypting Secret Data at Rest

**Domain:** Minimize Microservice Vulnerabilities
**Thời gian gợi ý:** 25 phút
**Yêu cầu:** SSH + sudo vào `k8s-controlplane-01`. **Mình chỉ viết hướng dẫn,
không tự chạy** — cùng lý do với Lab 1.6 (Audit Logging): thay đổi này sửa
static pod manifest của `kube-apiserver`, rủi ro làm sập API server nếu sai
cú pháp, và (khác Lab 1.6) **sai sót ở bước xoay vòng key có thể khiến toàn
bộ Secret hiện có không đọc lại được** — hậu quả nặng hơn nhiều nên cần bạn
tự kiểm soát từng bước.

## Mục tiêu

- Chứng minh Secret mặc định chỉ base64-encode, KHÔNG mã hoá — ai đọc được
  etcd là đọc được secret plaintext.
- Bật encryption at rest bằng `EncryptionConfiguration` (provider `aescbc`).
- Xoay vòng (rotate) secret đã tồn tại để áp dụng encryption cho dữ liệu cũ.

## Bước 1 — Chứng minh vấn đề: Secret trong etcd không được mã hoá

```bash
kubectl create secret generic my-secret --from-literal=key1=supersecret

ssh <user>@172.16.5.12   # k8s-controlplane-01
sudo -i

ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C | grep -i supersecret
```

Nếu thấy chuỗi `supersecret` xuất hiện dạng plaintext trong output — xác
nhận đúng vấn đề: base64 không phải mã hoá, chỉ là encoding.

## Bước 2 — Tạo encryption key và EncryptionConfiguration

```bash
# Trên control-plane node
head -c 32 /dev/urandom | base64
# -> lưu lại giá trị này, ví dụ: "abcdef0123456789abcdef0123456789abcd=="

sudo mkdir -p /etc/kubernetes/enc
sudo tee /etc/kubernetes/enc/encryption-config.yaml <<'EOF'
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_KEY_TỪ_BƯỚC_TRÊN>
      - identity: {}
EOF
```

> Thứ tự `providers` quan trọng: provider đầu tiên (`aescbc`) dùng để
> **ghi** (encrypt), toàn bộ danh sách dùng để **đọc** (decrypt) — luôn để
> `identity: {}` (không mã hoá) ở CUỐI để vẫn đọc được các secret cũ đã lỡ
> ghi trước khi bật encryption, tránh mất dữ liệu.

## Bước 3 — Backup manifest gốc rồi sửa `kube-apiserver.yaml`

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
```

Thêm flag:

```yaml
- --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

Thêm `volumeMounts` + `volumes` trỏ tới `/etc/kubernetes/enc` (giống hệt
pattern đã dùng ở Lab 1.6 cho `/etc/kubernetes/audit`):

```yaml
    volumeMounts:
      - name: enc
        mountPath: /etc/kubernetes/enc
        readOnly: true
  volumes:
    - name: enc
      hostPath:
        path: /etc/kubernetes/enc
        type: DirectoryOrCreate
```

## Bước 4 — Verify apiserver sống lại

```bash
watch crictl ps --name kube-apiserver
```

Từ máy làm việc:

```bash
kubectl get nodes
```

Nếu lỗi, restore ngay: `cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml`

## Bước 5 — Verify encryption hoạt động (chỉ ảnh hưởng secret MỚI)

```bash
kubectl create secret generic my-secret-2 --from-literal=key1=supersecret2

# Trên control-plane
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret-2 | hexdump -C | head -3
```

Output giờ bắt đầu bằng `k8s:enc:aescbc:v1:key1` thay vì base64 thuần —
xác nhận đã mã hoá. Secret cũ (`my-secret` tạo ở Bước 1) **vẫn chưa được mã
hoá** — encryption chỉ áp dụng cho request ghi mới, không tự động mã hoá
lại dữ liệu cũ.

## Bước 6 — Rotate: mã hoá lại toàn bộ secret cũ

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Lệnh này đọc rồi ghi lại (no-op về nội dung) toàn bộ secret hiện có, buộc
apiserver phải encrypt lại theo config mới. Verify lại `my-secret` (Bước 1)
bằng etcdctl — giờ phải thấy prefix `k8s:enc:aescbc:v1:` thay vì plaintext.

## Dọn dẹp / Restore

```bash
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
watch crictl ps --name kube-apiserver
```

## Mẹo thi

- Đề thi hay cho sẵn 1 phần `EncryptionConfiguration` và yêu cầu bạn hoàn
  thiện + trỏ đúng flag — luôn nhớ thứ tự `providers` (provider ghi ở đầu),
  và luôn giữ `identity: {}` ở cuối trong lúc chuyển đổi để không khoá mất
  dữ liệu cũ.
- `--encryption-provider-config` chỉ mã hoá được resource khai trong danh
  sách `resources:` — mặc định người ta chỉ khai `secrets`, nhưng đề có thể
  yêu cầu thêm `configmaps` hoặc resource khác.
- Bước rotate (`kubectl get ... | kubectl replace -f -`) rất dễ bị quên
  trong bài thi — nếu đề hỏi "đảm bảo TẤT CẢ secret đã được mã hoá" (không
  chỉ secret mới), đây là bước bắt buộc.
