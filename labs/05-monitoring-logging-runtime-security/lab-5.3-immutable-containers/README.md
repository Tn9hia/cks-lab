# Lab 5.3 — Immutable Containers + Falco (kết hợp Domain 3 và Domain 5)

**Domain:** Monitoring, Logging & Runtime Security
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** đã làm Lab 3.1 (Security Context) và Lab 5.1 (Falco).

> **✅ Đã test trên cụm thật** (2026-10-11). Kết quả khớp 100% với kỳ vọng
> trong tài liệu: `apt update` trong container báo đúng lỗi
> `Read-only file system` dù `privileged: true`; Falco (sau khi áp rule đã
> fix ở Lab 5.2) bắt đúng 1 dòng "Shell opened inside container" sạch,
> không noise.

## Mục tiêu

- Chứng minh `readOnlyRootFilesystem` vẫn có hiệu lực **ngay cả khi container
  chạy privileged** — 2 cơ chế bảo vệ độc lập nhau (privileged liên quan
  syscall/capability, readOnlyRootFilesystem liên quan filesystem).
- Dùng Falco để quan sát và xác nhận hành vi bị chặn thay vì chỉ tin vào
  thông báo lỗi của ứng dụng.

## Bước 1 — Deploy pod immutable + privileged

```bash
kubectl apply -f manifests/nginx-immutable-privileged.yaml
kubectl wait --for=condition=Ready pod/nginx-immutable --timeout=30s
```

## Bước 2 — Thử sửa đổi filesystem dù đang privileged

```bash
kubectl exec -ti nginx-immutable -- apt update
```

Kỳ vọng (theo tài liệu, dựa trên Lab 3.1 đã verify hành vi tương tự):

```
E: List directory /var/lib/apt/lists/partial is missing. - Acquire (30: Read-only file system)
command terminated with exit code 100
```

`privileged: true` cho container gần như toàn quyền syscall lên kernel,
nhưng **không** ghi đè được việc mount root filesystem là read-only ở tầng
container runtime — đây là 2 lớp kiểm soát khác nhau.

## Bước 3 — Quan sát qua Falco

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

Trong terminal khác, thử một số hành vi bên trong container:

```bash
kubectl exec -ti nginx-immutable -- bash
```

Falco sẽ log "Shell opened inside container" (từ rule đã viết ở Lab 5.2)
— minh hoạ rằng dù filesystem bị khoá, các hành vi khác (spawn shell, đọc
file...) vẫn cần Falco để phát hiện, `readOnlyRootFilesystem` không giải
quyết được lớp bảo vệ này.

## Bước 4 — Cảnh báo quan trọng về `/proc` khi dùng privileged (KHÔNG tự làm trên cụm chính)

Tài liệu gốc lưu ý: container privileged vẫn có thể ghi vào `/proc` (pseudo
filesystem, không bị `readOnlyRootFilesystem` chặn) và **ảnh hưởng trực
tiếp tới host**, ví dụ đổi `vm.swappiness`:

```bash
# CHỈ đọc để hiểu, KHÔNG tự ý chạy lệnh ghi trên cụm dùng chung:
kubectl exec -ti nginx-immutable -- sh -c "echo 10 > /proc/sys/vm/swappiness"
```

Đây chính là lý do cốt lõi tài liệu nhấn mạnh: **tránh dùng `privileged:
true`** trừ khi thực sự bắt buộc — `readOnlyRootFilesystem` không đủ để
bảo vệ host khỏi 1 container privileged.

## Dọn dẹp

```bash
kubectl delete pod nginx-immutable --ignore-not-found
```

## Mẹo thi

- Đừng nhầm "immutable container" với "immutable infrastructure" (khái niệm
  rộng hơn, áp dụng cho cả VM/node) — CKS tập trung vào immutable ở cấp
  container/pod thông qua `securityContext`.
- Nếu đề bài có pod `privileged: true` VÀ yêu cầu "đảm bảo không ai sửa được
  file trong container", câu trả lời đúng vẫn là thêm
  `readOnlyRootFilesystem: true` — privileged không loại trừ khả năng dùng
  cùng lúc với readOnlyRootFilesystem, và bài học chính là 2 field này độc
  lập, không mâu thuẫn.
