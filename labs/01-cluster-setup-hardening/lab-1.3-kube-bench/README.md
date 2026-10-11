# Lab 1.3 — kube-bench / CIS Benchmark

**Domain:** Cluster Setup — CIS benchmark
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** SSH vào từng node (control-plane + ít nhất 1 worker) với quyền
root/sudo — chạy trực tiếp trên node, không qua Claude Code.

## Mục tiêu

- Chạy kube-bench để tự động chấm điểm cụm theo CIS Kubernetes Benchmark.
- Đọc report, phân biệt `[PASS]` / `[FAIL]` / `[WARN]`, tự sửa ít nhất 5 FAIL.
- Hiểu vì sao kube-bench chạy khác nhau trên control-plane vs worker (khác bộ
  rule vì khác thành phần: apiserver/etcd/scheduler chỉ có trên control-plane).

## Bước 1 — Cài kube-bench trên control-plane node

Cách nhanh nhất: chạy dưới dạng Job trong cluster (không cần cài binary):

```bash
kubectl apply -f manifests/kube-bench-job.yaml
kubectl wait --for=condition=complete job/kube-bench --timeout=60s
kubectl logs job/kube-bench
```

Hoặc cài binary trực tiếp trên node (bám sát cách thi thật hay làm hơn, vì
trong đề thi bạn SSH thẳng vào node):

```bash
# Trên node
curl -L https://github.com/aquasecurity/kube-bench/releases/latest/download/kube-bench_linux_amd64.tar.gz -o kube-bench.tar.gz
tar -xzf kube-bench.tar.gz
sudo ./kube-bench run --targets master,etcd,node,policies > report-controlplane.txt
less report-controlplane.txt
```

## Bước 2 — Chạy trên worker node

```bash
# SSH sang k8s-worker-01
sudo ./kube-bench run --targets node > report-worker.txt
```

## Bước 3 — Đọc report và ưu tiên xử lý

```bash
grep -E "\[FAIL\]" report-controlplane.txt
```

Chọn ít nhất 5 FAIL để sửa. Các FAIL phổ biến hay gặp và cách sửa nhanh:

| Finding thường gặp | Sửa ở đâu | Cách sửa |
|---|---|---|
| Kubelet `--anonymous-auth` không phải false | `/var/lib/kubelet/config.yaml` | set `authentication.anonymous.enabled: false`, restart kubelet |
| Permissions của file manifest quá lỏng (>600) | `/etc/kubernetes/manifests/*.yaml` | `chmod 600 /etc/kubernetes/manifests/*.yaml` |
| Permissions của `admin.conf`/`kubelet.conf` quá lỏng | `/etc/kubernetes/*.conf` | `chmod 600 /etc/kubernetes/admin.conf` |
| etcd data dir không giới hạn quyền | `/var/lib/etcd` | `chmod 700 /var/lib/etcd` |
| `--profiling` chưa tắt trên scheduler/controller-manager | static pod manifest tương ứng | thêm `--profiling=false` vào command |
| Owner của file pki không phải root:root | `/etc/kubernetes/pki` | `chown -R root:root /etc/kubernetes/pki` |

Sau mỗi lần sửa, chạy lại kube-bench để xác nhận FAIL đã chuyển thành PASS:

```bash
sudo ./kube-bench run --targets master,etcd,node,policies | grep -A2 "<mã-check-vừa-sửa>"
```

## Bước 4 — Verify cluster vẫn khoẻ sau khi sửa

Luôn verify từ máy làm việc sau mỗi thay đổi trên control-plane, vì sửa
kubelet config hay static pod sai có thể làm node `NotReady`:

```bash
kubectl get nodes
kubectl get pods -A | grep -v Running
```

## Dọn dẹp

```bash
kubectl delete job kube-bench --ignore-not-found
```

## Mẹo thi

- Đề thi CKS thật **không cài kube-bench cho bạn** và cũng hiếm khi hỏi trực
  tiếp "chạy kube-bench" — nhưng các câu hỏi hardening (file permissions,
  disable anonymous-auth, tắt profiling...) chính là các item trong CIS
  Benchmark. Luyện lab này để quen mặt các finding, không phải để nhớ cú
  pháp CLI của kube-bench.
- Luôn sửa từng thay đổi một rồi verify ngay (`kubectl get nodes`), đừng sửa
  hàng loạt rồi mới kiểm tra — nếu apiserver/kubelet crash sẽ khó biết dòng
  nào gây lỗi.
- Phân biệt rõ 2 nhóm check: **control plane** (API server, scheduler,
  controller-manager, etcd — chỉ chạy trên control-plane node) và **kubelet**
  (chạy trên mọi node, kể cả worker).
