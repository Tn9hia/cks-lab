# Lab 5.4 — Phát hiện Privilege Escalation bằng Falco (bài tự luyện nâng cao)

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** đã làm Lab 5.1-5.3.

> **Nguồn:** lab này KHÔNG dịch trực tiếp từ 1 bài học cụ thể trong notes
> KodeKloud (notes chỉ dừng ở mức khái niệm "Falco phát hiện hành vi bất
> thường" qua ví dụ shell/shadow file ở bài "Perform behavioral analytics
> of syscall process"). Đây là bài tự thiết kế để luyện tập **kết hợp**
> kiến thức Domain 3 (Security Context, privilege escalation) với Domain 5
> (Falco) — đúng tinh thần "ứng dụng kiến thức đã học vào tình huống mới"
> mà đề thi CKS thường yêu cầu, nhưng không có gì để bạn tra "đáp án gốc"
> trong tài liệu.
>
> **✅ Đã test trên cụm thật** (2026-10-11), đã sửa **1 lỗi nghiêm trọng**:
> rule "Privileged container started" dùng macro `container_started`
> **không tồn tại** trong Falco 0.45.0 → không phải rule đó âm thầm bị bỏ
> qua, mà **toàn bộ Falco pod crash** (`CrashLoopBackOff`, mọi detection
> trên node đó ngừng hoạt động) vì Falco coi rules file "Invalid" và từ
> chối khởi động. Đã sửa, xem phần "Phát hiện quan trọng" ở cuối file.

## Mục tiêu

- Viết rule Falco phát hiện 3 dấu hiệu privilege escalation phổ biến: process
  lạ xuất hiện trong container (drift), gọi syscall `setuid`/`setgid` bất
  thường, và container privileged khởi động.
- Tự tay tái tạo 1 kịch bản tấn công đơn giản để kiểm chứng rule.

## Bước 1 — Áp dụng rule

```bash
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set driver.kind=modern_ebpf \
  --set tty=true \
  -f manifests/detect-privesc-rules.yaml
```

## Bước 2 — Kịch bản 1: Container Drift (process lạ xuất hiện)

```bash
kubectl run nginx --image=nginx
kubectl wait --for=condition=Ready pod/nginx --timeout=30s
kubectl exec -ti nginx -- bash -c "apt-get update && apt-get install -y netcat"
kubectl exec -ti nginx -- nc -lvp 4444 &
```

Kỳ vọng: Falco log "Unexpected process spawned in container" khi `apt-get`,
`nc` chạy (không nằm trong whitelist `nginx, sh, bash, cat, ls, curl,
wget`) — đây chính là cách phát hiện "attacker cài thêm tool sau khi có
shell access".

> **⚠️ Thực tế quan sát được: rule này rất noisy với whitelist đơn giản.**
> Ngay khi pod `nginx` khởi động — **chưa cần `exec` gì cả** — rule đã bắn
> alert cho chính script entrypoint của image (`docker-entrypoint.sh` gọi
> `awk`, `basename`, `30-tune-worker-processes.sh`...), vì các binary đó
> không nằm trong whitelist. Trên cụm thật, rule tương tự cũng bắn liên
> tục cho các pod hệ thống như `kube-proxy` (`iptables`/`ip6tables` là
> hành vi bình thường, không phải attacker). Đây là bài học quan trọng cho
> đề thi: **whitelist tĩnh kiểu "liệt kê vài binary" không đủ cho môi
> trường thật** — cần whitelist theo entrypoint/base-image cụ thể của từng
> workload (hoặc dùng tool build whitelist tự động như Falco's "drift"
> plugin chính thức), không phải liệt kê thủ công vài tên quen mắt.

## Bước 3 — Kịch bản 2: Privileged container khởi động

```bash
kubectl apply -f ../lab-5.3-immutable-containers/manifests/nginx-immutable-privileged.yaml
```

Kỳ vọng: Falco log ngay "Privileged container started" khi pod
`nginx-immutable` khởi động — cảnh báo này nên trigger **runbook**: security
team review lý do vì sao cần chạy privileged, có đúng theo policy tổ chức
không.

## Bước 4 — Kịch bản 3: setuid/setgid (khó tái tạo an toàn, đọc để hiểu)

Hầu hết ứng dụng bình thường không tự gọi `setuid`/`setgid` lúc runtime —
rule này chủ yếu bắt các trường hợp bất thường như: 1 binary có SUID bit bị
lợi dụng để leo quyền. Để tái tạo an toàn (không cần binary thật có lỗ hổng):

```bash
kubectl run test-setuid --image=ubuntu --command -- sleep 3600
kubectl wait --for=condition=Ready pod/test-setuid --timeout=30s
kubectl exec -ti test-setuid -- bash -c "apt-get update && apt-get install -y gosu && gosu nobody id"
```

`gosu` nội bộ gọi `setuid`/`setgid` để đổi từ root sang user `nobody` — một
cách an toàn để trigger đúng syscall mà không cần khai thác lỗ hổng thật.

## Bước 5 — Đối chiếu với các lớp phòng thủ đã học trước đó

| Lớp phòng thủ | Ngăn chặn TRƯỚC hay phát hiện SAU? | Lab liên quan |
|---|---|---|
| RBAC | Trước (chặn API call không đủ quyền) | 1.1 |
| PSA / ValidatingAdmissionPolicy | Trước (chặn pod spec vi phạm) | 3.2, 3.3 |
| SecurityContext (drop capabilities, non-root) | Trước (giới hạn khả năng ngay từ lúc chạy) | 2.1, 3.1 |
| Seccomp / AppArmor | Trước (chặn syscall/hành vi cụ thể) | 2.2, 2.3 |
| **Falco** | **Sau** (phát hiện hành vi đã xảy ra, alert để phản ứng) | 5.1-5.4 |

Bài học tổng kết: không lớp nào thay thế được lớp nào — CKS kỳ vọng bạn
hiểu **defense in depth**, kết hợp nhiều lớp thay vì chỉ dựa vào 1 cơ chế.

## Dọn dẹp

```bash
kubectl delete pod nginx nginx-immutable test-setuid --ignore-not-found
```

## Phát hiện quan trọng từ test thực tế (2026-10-11)

Rule gốc trong `manifests/detect-privesc-rules.yaml` có **1 lỗi nghiêm
trọng** (Falco crash) và 1 warning deprecated — cả 2 đã sửa trực tiếp
trong file.

**1. [Nghiêm trọng] Macro `container_started` không tồn tại trong Falco
0.45.0.**

Bản gốc:

```yaml
condition: container_started and container.privileged=true
```

Khi `helm upgrade` với rule này, `kubectl get pods -n falco` cho thấy pod
vào `CrashLoopBackOff`. Log container `falco`:

```
Error: /etc/falco/rules.d/rules-privesc.yaml: Invalid
LOAD_ERR_VALIDATE (Error validating rule/macro/list/exception objects):
Undefined macro 'container_started' used in filter.
```

Khác với lỗi rule thông thường (Falco bỏ qua rule lỗi, các rule khác vẫn
chạy), lỗi **macro không tồn tại** khiến Falco coi toàn bộ rules file là
invalid và **từ chối khởi động** — daemonset mất hẳn khả năng detect trên
node đó, không chỉ mất 1 rule. Đây là lý do nên luôn `kubectl logs` kiểm
tra ngay sau mỗi `helm upgrade` rule, không chỉ nhìn `STATUS Running`
(pod cũ vẫn Running cho tới khi pod mới thay thế xong, dễ tạo cảm giác
"vẫn ổn" trong vài giây dù rule mới bị lỗi).

Falco không có khái niệm "container vừa start" như 1 macro riêng — cách
built-in ruleset xử lý tương tự (vd rule "Debugfs Launched in Privileged
Container") là kết hợp `spawned_process` (process được spawn — gần đúng
thời điểm container start nếu đó là tiến trình đầu tiên) với
`container.privileged=true`. Fix dùng thêm macro built-in `proc_is_new`
(tuổi process ≤ 5 giây) để giới hạn alert chỉ trong khoảng ngắn sau khi
container khởi động, tránh bắn lại mỗi lần có process mới:

```yaml
condition: >
  spawned_process and container
  and container.privileged=true and proc_is_new
```

**2. [Warning, không fatal] `evt.dir = <` đã deprecated.**

```
LOAD_DEPRECATED_ITEM (Used deprecated item: field 'evt.dir'): due to the
drop of enter events, 'evt.dir = <' always evaluates to true...
```

Falco bản mới không còn phân biệt enter/exit event theo cách cũ nên
`evt.dir = <` luôn đúng — vô nghĩa, có thể xoá an toàn. Đã xoá khỏi
condition của rule "Setuid or setgid syscall in container".

**Kết quả sau khi sửa:** cả 3 kịch bản (drift, privileged container
started, setuid/setgid qua `gosu`) đều hoạt động đúng, không còn warning
khi load rule.

## Mẹo thi

- Nếu đề thi cho 1 kịch bản "mô tả hành vi tấn công" và hỏi "Falco rule nào
  sẽ bắt được", hãy nghĩ theo syscall/hành vi cụ thể (spawn process, open
  file, syscall setuid...) thay vì nghĩ theo "loại tấn công" trừu tượng —
  Falco rule luôn viết ở mức syscall/hành vi, không ở mức khái niệm.
- Ưu tiên **sửa/mở rộng rule built-in** hơn là viết mới hoàn toàn khi có
  thể — rule built-in đã được cộng đồng Falco kiểm chứng kỹ, ít false
  positive hơn rule tự viết vội trong lúc thi.
