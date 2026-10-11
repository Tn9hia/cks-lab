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
> trong tài liệu. **CHƯA TEST TRÊN CỤM** — làm cẩn thận, có thể rule 2
> (setuid/setgid) cần điều chỉnh cú pháp `evt.dir` tuỳ phiên bản Falco.

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

## Mẹo thi

- Nếu đề thi cho 1 kịch bản "mô tả hành vi tấn công" và hỏi "Falco rule nào
  sẽ bắt được", hãy nghĩ theo syscall/hành vi cụ thể (spawn process, open
  file, syscall setuid...) thay vì nghĩ theo "loại tấn công" trừu tượng —
  Falco rule luôn viết ở mức syscall/hành vi, không ở mức khái niệm.
- Ưu tiên **sửa/mở rộng rule built-in** hơn là viết mới hoàn toàn khi có
  thể — rule built-in đã được cộng đồng Falco kiểm chứng kỹ, ít false
  positive hơn rule tự viết vội trong lúc thi.
