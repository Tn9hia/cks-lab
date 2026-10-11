# Lab 3.6 — Container Sandboxing với gVisor (RuntimeClass)

**Domain:** Minimize Microservice Vulnerabilities
**Thời gian gợi ý:** 30-40 phút
**Yêu cầu:** SSH + sudo vào ít nhất 1 worker node. **Mình chỉ viết hướng
dẫn, không tự cài** — cài `runsc` (gVisor) đòi hỏi sửa cấu hình `containerd`
(`/etc/containerd/config.toml`) và restart service `containerd` trên node,
tức là **restart lại container runtime đang chạy mọi pod hiện có trên node
đó**. Đây là thay đổi có blast radius lớn nhất trong toàn bộ bộ lab — nếu
sai cấu hình containerd, node có thể mất khả năng chạy bất kỳ container nào
cho tới khi sửa lại. Đã xác nhận cụm bạn **hiện chưa có RuntimeClass nào**
(`kubectl get runtimeclass` trả về rỗng) — nghĩa là gVisor chưa được cài.

## ĐỌC TRƯỚC KHI LÀM

1. Làm trên **1 worker node duy nhất** trước (không phải control-plane).
2. Backup `/etc/containerd/config.toml` trước khi sửa.
3. Sau khi restart containerd, verify ngay `crictl ps` chạy được và
   `kubectl get nodes` vẫn Ready trước khi làm bước tiếp theo.
4. Cân nhắc dùng `kubectl cordon <node>` trước khi thử nghiệm để tránh
   scheduler đẩy thêm pod mới vào node đang bị thay đổi runtime.

## Mục tiêu

- Hiểu kiến trúc gVisor (Sentry chặn syscall, Gofer proxy file access) và vì
  sao nó cách ly mạnh hơn seccomp/AppArmor (chặn ở tầng syscall interception
  thay vì chỉ whitelist/blacklist syscall cho kernel thật).
- Cài `runsc`, đăng ký với containerd, tạo `RuntimeClass`, chạy pod dùng
  runtime đó.

## Bước 1 — Cài gVisor (`runsc`) trên 1 worker node

```bash
ssh <user>@172.16.5.13   # ví dụ k8s-worker-01
sudo -i

curl -fsSL https://gvisor.dev/archive.key | sudo gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | sudo tee /etc/apt/sources.list.d/gvisor.list
apt-get update
apt-get install -y runsc

runsc --version
```

## Bước 2 — Đăng ký `runsc` với containerd

```bash
cp /etc/containerd/config.toml /etc/containerd/config.toml.bak

sudo tee -a /etc/containerd/config.toml <<'EOF'

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF

sudo systemctl restart containerd
```

## Bước 3 — Verify containerd + kubelet vẫn khoẻ (bắt buộc trước khi tiếp tục)

```bash
sudo crictl ps
sudo systemctl status kubelet --no-pager
```

Từ máy làm việc:

```bash
kubectl get nodes
kubectl get pods -A --field-selector spec.nodeName=k8s-worker-01 | grep -v Running
```

Nếu có bất thường, restore ngay:

```bash
cp /etc/containerd/config.toml.bak /etc/containerd/config.toml
systemctl restart containerd
```

## Bước 4 — Tạo RuntimeClass (làm từ máy làm việc, qua kubectl)

```yaml
# manifests/runtimeclass-gvisor.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

```bash
kubectl apply -f manifests/runtimeclass-gvisor.yaml
```

## Bước 5 — Deploy pod dùng gVisor, pin vào đúng node đã cài

```yaml
# manifests/gvisor-test-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-test
spec:
  nodeName: k8s-worker-01   # đổi đúng tên node bạn vừa cài runsc
  runtimeClassName: gvisor
  containers:
    - name: nginx
      image: nginx:1.27
```

```bash
kubectl apply -f manifests/gvisor-test-pod.yaml
kubectl wait --for=condition=Ready pod/gvisor-test --timeout=30s
```

## Bước 6 — Verify pod thực sự chạy trong gVisor sandbox

```bash
kubectl exec gvisor-test -- dmesg | head -1
# -> nếu thấy dòng nhắc tới "gVisor" thay vì kernel Linux thật, xác nhận đúng sandbox

# Trên node, kiểm tra qua crictl
sudo crictl ps | grep gvisor-test
sudo crictl inspect <container-id> | grep -i runtimeType
# -> "io.containerd.runsc.v1"
```

So sánh với 1 pod thường (không có `runtimeClassName`) chạy trên cùng node
để thấy khác biệt runtime.

## Dọn dẹp / Revert

```bash
kubectl delete pod gvisor-test --ignore-not-found
kubectl delete runtimeclass gvisor --ignore-not-found

# Trên node (nếu muốn gỡ hẳn):
cp /etc/containerd/config.toml.bak /etc/containerd/config.toml
systemctl restart containerd
apt-get remove -y runsc
```

## Mẹo thi

- Đề thi CKS thường **cho sẵn gVisor/Kata đã cài trên node**, không yêu cầu
  bạn tự cài từ đầu (rủi ro quá cao để đưa vào 1 câu hỏi có giới hạn thời
  gian) — trọng tâm câu hỏi thường là: viết đúng `RuntimeClass` +
  `runtimeClassName` trong Pod spec, và biết cách verify pod đang chạy đúng
  sandbox runtime.
- `RuntimeClass` là **cluster-scoped**, nhưng handler `runsc` chỉ hoạt động
  trên node nào đã cài đặt — nếu pod dùng `runtimeClassName: gvisor` bị
  schedule sang node chưa cài, pod sẽ lỗi `CreateContainerError`. Cách xử lý
  chuẩn hơn `nodeName` (chỉ dùng để test): gắn `nodeSelector`/taint riêng
  cho nhóm node đã cài gVisor, rồi RuntimeClass tham chiếu tới đúng
  `scheduling.nodeSelector` đó.
- So sánh nhanh 3 lớp cô lập đã học: **seccomp** (whitelist/blacklist syscall
  cụ thể, chạy trên kernel thật) → **AppArmor** (whitelist theo path/hành vi,
  cũng trên kernel thật) → **gVisor/Kata** (chặn toàn bộ syscall bằng 1 lớp
  kernel giả lập hoặc VM riêng, cô lập mạnh nhất, overhead cao nhất).
