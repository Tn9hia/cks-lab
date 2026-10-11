# Lab 5.2 — Viết Falco Rule tuỳ chỉnh

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** đã hoàn thành Lab 5.1 (Falco đang chạy).

> **✅ Đã test trên cụm thật** (2026-10-11), Falco `0.45.0`. Quá trình test
> phát hiện **3 lỗi/caveat quan trọng** so với bản gốc — đã sửa trực tiếp
> trong `manifests/custom-rules.yaml`. Đọc kỹ phần "Phát hiện quan trọng từ
> test thực tế" ở cuối file trước khi làm lab — đây là kiến thức thi rất
> đáng giá (hiểu rule engine, không chỉ chép cú pháp).

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
cat /etc/passwd
```

Kỳ vọng: log Falco xuất hiện dòng `CRITICAL` "Sensitive file opened for
reading inside container".

> **⚠️ Dùng `/etc/passwd`, KHÔNG dùng `/etc/shadow` để demo rule này** —
> xem phần "Phát hiện quan trọng" ở cuối file, mục #3. Falco built-in đã có
> rule riêng cho `/etc/shadow` và với `rule_matching: first` (default của
> chart này), rule built-in đó luôn thắng, custom rule của bạn sẽ không bao
> giờ chạy khi test bằng `/etc/shadow`. `/etc/passwd` không nằm trong danh
> sách built-in nên custom rule mới chắc chắn là rule duy nhất match.

## Bước 5 — Trigger rule "Unexpected write outside app directory"

```bash
kubectl run webapp --image=kodekloud/simple-webapp
kubectl wait --for=condition=Ready pod/webapp --timeout=30s
kubectl exec -ti webapp -- bash -c "echo hack > /tmp/malicious.txt"
```

> **⚠️ Image `kodekloud/simple-webapp` chỉ build cho amd64** — nếu cụm bạn
> dùng node arm64 (vd Apple Silicon qua multipass), pod sẽ
> `CrashLoopBackOff` với lỗi `exec format error`. Cách né tạm để vẫn test
> được logic rule: SSH vào 1 node, tag thủ công 1 image arm64 có sẵn thành
> đúng tên repo cần test —
> ```bash
> multipass exec <node> -- sudo ctr -n k8s.io images pull docker.io/library/busybox:latest
> multipass exec <node> -- sudo ctr -n k8s.io images tag docker.io/library/busybox:latest docker.io/kodekloud/simple-webapp:latest
> ```
> rồi deploy pod với `image: kodekloud/simple-webapp:latest`,
> `imagePullPolicy: Never`, và `command: ["sleep", "3600"]` (busybox không
> có binary `bash`/web server mặc định, dùng `sh` thay cho `bash` ở các
> lệnh test). Trên cụm amd64 thật (đa số đề thi CKS) thì image gốc chạy
> bình thường, không cần workaround này.

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

## Phát hiện quan trọng từ test thực tế (2026-10-11)

Rule gốc trong `manifests/custom-rules.yaml` **có 2 lỗi** đã được sửa trực
tiếp trong file — nếu bạn so với bản backup/git history cũ, đây là diff:

**1. Rule "Detect Shell inside a container" thiếu giới hạn event-type.**

Bản gốc: `condition: container and proc.name in (linux_shells)`. Falco báo
warning khi load: `LOAD_NO_EVTTYPE... Rule matches too many evt.type
values`. Hậu quả không chỉ là "chậm" — khi test, rule này bắn **hàng chục
dòng alert trùng lặp** cho đúng 1 lệnh `bash -c "..."` (vì nó match MỌI
syscall của process bash, không riêng lúc spawn). Nghiêm trọng hơn: trong
Falco bản này, 1 rule không giới hạn event-type sẽ "nuốt" luôn các syscall
event lẽ ra thuộc về rule khác (xem mục #2) — làm hỏng hẳn rule "Unexpected
write" khi 2 rule cùng load. Fix: thêm macro `spawned_process` (built-in,
tương đương `evt.type in (execve, execveat)`):

```yaml
condition: spawned_process and container and proc.name in (linux_shells)
```

Sau khi fix: 1 shell → đúng 1 alert, không còn warning khi load.

**2. `rule_matching: first` — chỉ 1 rule được alert mỗi event, không phải
"mọi rule khớp đều alert".**

Đây là config mặc định của Falco Helm chart bản này
(`/etc/falco/falco.yaml: rule_matching: first`) — khác với giả định phổ
biến (và hành vi mặc định cũ của Falco) là mọi rule match 1 event đều sinh
alert riêng. Hệ quả quan sát được:

- Khi rule "Detect Shell" (lỗi #1, chưa fix) match MỌI syscall của process
  `sh`/`bash`, nó "chiếm" luôn event `openat` của chính lệnh ghi file —
  khiến rule "Unexpected write outside app directory" **không bao giờ
  chạy** dù logic đúng 100%. Fix lỗi #1 cũng giải quyết luôn lỗi này.
- `/etc/shadow` nằm trong list built-in `sensitive_file_names` (dùng bởi
  rule có sẵn "Read sensitive file untrusted") — rule built-in này load
  **trước** file custom nên luôn "thắng". Custom rule "Read sensitive file
  in container" của bạn **sẽ không bao giờ fire khi test bằng
  `/etc/shadow`**, dù rule hoàn toàn đúng cú pháp. `/etc/passwd` không có
  trong list built-in nên không bị tranh — dùng để demo (xem Bước 4).

**3. `container.image.repository` trả về có prefix registry, không phải
chỉ `<org>/<repo>`.**

Giá trị thật quan sát được: `docker.io/kodekloud/simple-webapp` (có tiền
tố `docker.io/`), không phải `kodekloud/simple-webapp` như rule gốc so
sánh bằng `=`. So sánh exact-match `=` **không bao giờ khớp**, rule im
lặng không báo lỗi parse (hợp lệ cú pháp) nhưng cũng không bao giờ fire.
Fix: dùng `contains` thay vì `=`:

```yaml
condition: >
  open_write and container
  and container.image.repository contains "kodekloud/simple-webapp"
  and fd.directory != "/opt/app"
```

**Bài học thi:** khi viết/sửa rule so khớp tên image, ưu tiên `contains`
hoặc `endswith`-style thay vì `=` tuyệt đối, trừ khi bạn chắc chắn giá trị
field đầy đủ (có thể kiểm tra bằng cách thêm field đó vào `output` của 1
rule debug tạm thời rồi đọc log thật).

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
