# Lab 5.1 — Cài đặt Falco

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 20 phút

> **✅ Đã test trên cụm thật** (2026-10-11) — Falco `0.45.0` (chart
> `falcosecurity/falco`), driver `modern_ebpf`, trên cụm arm64
> (`k8s-master` + `k8s-worker-1`, kernel 6.8, containerd). Cài qua Helm chạy
> trơn tru, DaemonSet `Running 2/2` trên cả 2 node, bắt alert built-in ngay
> (`cat /etc/shadow` → `Sensitive file opened for reading by non-trusted
> program`). Có vài warning vô hại trong log khởi động (xem bên dưới) —
> không ảnh hưởng detection.

## Mục tiêu

- Hiểu Falco hoạt động bằng cách theo dõi syscall (qua kernel module hoặc
  eBPF), đối chiếu với rule engine để phát hiện hành vi bất thường.
- Cài Falco dưới dạng DaemonSet qua Helm (cách khuyến nghị hiện tại, thay vì
  cài package trực tiếp lên từng node — ít rủi ro hơn vì không cần SSH/sửa
  gì ở tầng OS, mọi thứ quản lý qua Kubernetes).

## Bước 1 — Cài Helm (nếu chưa có)

```bash
brew install helm   # hoặc theo hướng dẫn https://helm.sh/docs/intro/install/
```

## Bước 2 — Thêm Falco Helm repo và cài đặt

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

kubectl create namespace falco

helm install falco falcosecurity/falco \
  --namespace falco \
  --set driver.kind=modern_ebpf \
  --set tty=true
```

`driver.kind=modern_ebpf` dùng eBPF thay vì kernel module — ít xâm lấn hơn,
hoạt động tốt trên kernel Linux mới (cụm bạn dùng kernel 6.8, đủ mới để
dùng modern eBPF driver không cần cài thêm gì).

## Bước 3 — Verify Falco chạy trên mọi node

```bash
kubectl get pods -n falco -o wide
```

Kỳ vọng: 1 pod Falco chạy trên **mỗi node** (DaemonSet), status `Running`.
Nếu có node không chạy được, kiểm tra:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50
```

Lỗi hay gặp: kernel không hỗ trợ eBPF driver đang chọn → thử đổi
`driver.kind=ebpf` (legacy eBPF) hoặc `driver.kind=kmod` (kernel module,
cần kernel headers trên node).

**Thực tế quan sát được:** log khởi động có vài dòng warning dạng
`libbpf: failed to determine tracepoint 'syscalls/sys_enter_creat' perf
event ID` và tương tự cho `sys_enter_open`. Đây **không phải lỗi** —
message tự giải thích rõ "Detection will continue to work, but TOCTOU
mitigation may not properly work". Lý do: kernel/arch hiện tại định nghĩa
lại 1 số syscall cũ (`creat`, `open`) qua `openat`/`openat2`, nên tracepoint
riêng cho bản cũ không tồn tại — core detection vẫn hoạt động đầy đủ qua
`openat`. Có thể bỏ qua các warning này.

## Bước 4 — Xem log Falco trực tiếp

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

Để terminal này mở, dùng cho Lab 5.2 khi tạo hành vi để Falco phát hiện.

## Dọn dẹp (chỉ chạy khi muốn gỡ hẳn, giữ lại cho Lab 5.2-5.4)

```bash
helm uninstall falco -n falco
kubectl delete namespace falco
```

## Mẹo thi

- Đề thi thường **cho sẵn Falco đã cài** (cài Falco từ đầu tốn thời gian,
  không phù hợp bài thi có giới hạn thời gian) — trọng tâm câu hỏi là đọc
  hiểu rule có sẵn hoặc viết thêm rule (xem Lab 5.2), không phải quy trình
  cài đặt.
- 2 cách driver chính: **kernel module** (xâm lấn hơn, một số managed K8s
  cấm dùng) vs **eBPF** (ít xâm lấn hơn, được ưa chuộng) — nếu đề hỏi lý do
  chọn eBPF, trả lời đúng trọng tâm này.
- Cài trực tiếp lên node (package `apt install falco`) cũng là phương án
  hợp lệ và được tài liệu KodeKloud dạy chính — nếu đề bài yêu cầu SSH vào
  node và cài qua `apt`, làm đúng theo hướng dẫn trong tài liệu gốc thay vì
  cố dùng Helm.
