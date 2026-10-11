# Lab 5.1 — Cài đặt Falco

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 20 phút

> **⚠️ CHƯA TEST TRÊN CỤM:** lab này viết trong lúc mất kết nối mạng tới
> cụm (VPN rớt giữa buổi). Nội dung dựa trên cách dùng chuẩn của Falco Helm
> chart (rất ổn định, được dùng rộng rãi) nhưng chưa được tự chạy-verify
> trên cụm của bạn như các lab khác trong bộ này. Hãy tự chạy và đối chiếu,
> hoặc nhờ chạy lại khi có mạng — nếu có sai lệch (vd version chart thay đổi
> API), báo lại để cập nhật.

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
