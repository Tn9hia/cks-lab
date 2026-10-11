# Lab 5.2 — Viết Falco Rule tuỳ chỉnh

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** đã hoàn thành Lab 5.1 (Falco đang chạy).

> **⚠️ CHƯA TEST TRÊN CỤM** — cùng lý do với Lab 5.1 (mất kết nối mạng giữa
> buổi). Cú pháp rule dựa đúng theo tài liệu KodeKloud + Falco official
> docs, nhưng bạn nên tự verify trước khi tin chắc 100%.

## Mục tiêu

- Hiểu 5 key bắt buộc của 1 Falco rule: `rule`, `desc`, `condition`,
  `output`, `priority`.
- Viết rule phát hiện: mở shell trong container, đọc file nhạy cảm
  (`/etc/shadow`), ghi file ngoài thư mục ứng dụng cho phép.
- Áp dụng custom rule vào Falco đang chạy (qua Helm values, không cần SSH).

## Bước 1 — Áp dụng custom rules vào Falco (qua Helm upgrade)

```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set driver.kind=modern_ebpf \
  --set tty=true \
  -f manifests/custom-rules.yaml
```

File `manifests/custom-rules.yaml` chứa 3 rule (xem nội dung trong file):
1. **Detect Shell inside a container** — cảnh báo WARNING khi có shell được
   mở trong container (bash, sh, zsh...).
2. **Read sensitive file in container** — cảnh báo CRITICAL khi đọc
   `/etc/shadow` hoặc `/etc/passwd`.
3. **Unexpected write outside app directory** — cảnh báo CRITICAL khi ghi
   file ngoài `/opt/app` trong container dùng image
   `kodekloud/simple-webapp` (đổi tên image cho khớp ứng dụng thật của bạn).

## Bước 2 — Verify rule được load

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=50 | grep -i "rule\|loaded"
```

## Bước 3 — Trigger rule "Detect Shell inside a container"

```bash
kubectl run nginx --image=nginx
kubectl wait --for=condition=Ready pod/nginx --timeout=30s

# Terminal khác: theo dõi log Falco real-time
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

```bash
kubectl exec -ti nginx -- bash
```

Kỳ vọng: log Falco xuất hiện ngay dòng cảnh báo `WARNING` với nội dung
"Shell opened inside container", kèm `user`, `container`, `image`.

## Bước 4 — Trigger rule "Read sensitive file"

Trong shell vừa mở ở Bước 3:

```bash
cat /etc/shadow
```

Kỳ vọng: log Falco xuất hiện dòng `CRITICAL` "Sensitive file opened for
reading inside container".

## Bước 5 — Trigger rule "Unexpected write outside app directory"

```bash
kubectl run webapp --image=kodekloud/simple-webapp
kubectl wait --for=condition=Ready pod/webapp --timeout=30s
kubectl exec -ti webapp -- bash -c "echo hack > /tmp/malicious.txt"
```

Kỳ vọng: log Falco `CRITICAL` "Unexpected write outside /opt/app" — vì
`/tmp` không nằm trong `/opt/app`.

Verify chiều ngược lại — ghi file ĐÚNG trong `/opt/app` thì KHÔNG bị cảnh
báo:

```bash
kubectl exec -ti webapp -- bash -c "echo ok > /opt/app/normal.txt"
```

## Bước 6 — Đổi priority của rule có sẵn (built-in)

Falco built-in đã có sẵn rule "Terminal shell in container" tương tự rule 1
mình viết tay — trong thực tế nên **override priority của rule built-in**
thay vì viết trùng rule mới, tránh bị double-alert:

```yaml
# thêm vào custom-rules.yaml, dùng đúng tên rule built-in để override
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
    and not user_expected_terminal_shell_in_container_conditions
  output: >
    A shell was spawned in a container with an attached terminal (user=%user.name
    %container.info shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

## Dọn dẹp

```bash
kubectl delete pod nginx webapp --ignore-not-found
```

## Mẹo thi

- Nhớ đúng 5 key bắt buộc: `rule`, `desc`, `condition`, `output`,
  `priority` — thiếu 1 trong 5 rule sẽ không load được.
- Thứ tự file trong `rules_file` (xem Lab 5.1/Falco config) quyết định rule
  nào override rule nào nếu trùng tên — **file load sau đè file load
  trước**. Custom rule luôn đặt trong file riêng, load sau file built-in.
- Cú pháp `condition` dùng ngôn ngữ riêng của Falco (không phải Rego, không
  phải CEL) — các field hay dùng: `container` (macro có sẵn kiểm tra đang
  trong container), `proc.name`, `fd.name`, `fd.directory`, `open_read`,
  `open_write`, `spawned_process`. Không cần nhớ hết, nhưng nên quen mặt các
  field phổ biến này để đọc/sửa rule có sẵn trong đề thi.
- Falco phát hiện **sau khi hành vi đã xảy ra** (runtime detection), khác
  với admission controller (OPA/PSA — Domain 3) **chặn trước khi xảy ra**.
  Đây là lớp phòng thủ bổ sung (defense in depth), không thay thế nhau.
