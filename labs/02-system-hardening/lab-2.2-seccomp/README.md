# Lab 2.2 — Seccomp

**Domain:** System Hardening
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** `kubectl` thôi cũng đủ cho Phần A. Phần B (custom profile) cần
file JSON tồn tại trên node — lab này dùng một Job với `hostPath` để tự ghi
file lên node **mà không cần SSH thủ công**.

## Mục tiêu

- Hiểu Kubernetes **không bật seccomp mặc định** trước v1.27 (annotation),
  và có `RuntimeDefault` sẵn để dùng từ v1.19+ qua `securityContext`.
- Biết 3 loại profile: `RuntimeDefault`, `Unconfined`, `Localhost` (custom).
- Tự viết seccomp profile chặn 1 syscall cụ thể và verify bằng cách gọi
  đúng syscall đó.

## Phần A — RuntimeDefault vs Unconfined

Đọc trực tiếp field `Seccomp` trong `/proc/1/status` của chính container —
cách này không phụ thuộc image bên thứ 3 nào (image `amicontained` gợi ý
trong tài liệu gốc dùng registry cá nhân `r.j3ss.co`, mình test thấy registry
này đang down/không ổn định nên đổi sang cách kiểm tra built-in, đáng tin
cậy hơn cho việc luyện tập lâu dài). Giá trị `Seccomp`: `0` = disabled,
`1` = strict, `2` = filtering (đây là quy ước chuẩn của kernel Linux, không
phải do lab này tự đặt ra).

### Baseline — không set gì (mặc định)

```bash
kubectl run seccheck --image=ubuntu --restart=Never --command -- \
  sh -c "grep Seccomp /proc/1/status; sleep 5"
sleep 6 && kubectl logs seccheck
kubectl delete pod seccheck
# -> Seccomp: 0 (disabled) — Kubernetes không tự bật seccomp nếu không khai báo
```

### RuntimeDefault

```bash
kubectl apply -f manifests/seccheck-runtimedefault.yaml
sleep 6 && kubectl logs seccheck-runtimedefault
kubectl delete pod seccheck-runtimedefault
# -> Seccomp: 2 (filtering) — dùng profile mặc định của containerd, chặn ~44 syscall nguy hiểm
```

### Unconfined (tường minh, để so sánh)

```bash
kubectl apply -f manifests/seccheck-unconfined.yaml
sleep 6 && kubectl logs seccheck-unconfined
kubectl delete pod seccheck-unconfined
# -> Seccomp: 0 (disabled) — giống hệt baseline, vì baseline vốn đã "unconfined" theo mặc định
```

## Phần B — Custom profile (Localhost) không cần SSH

### Bước 1 — Ghi profile lên 1 node cụ thể bằng Job (hostPath)

Ta pin cả Job ghi file lẫn pod test vào cùng 1 node (`k8s-worker-02`) để
không phải soạn DaemonSet cho toàn cluster.

```bash
kubectl apply -f manifests/write-profiles-job.yaml
kubectl wait --for=condition=complete job/write-seccomp-profiles --timeout=30s
kubectl logs job/write-seccomp-profiles
```

Job này ghi 3 file vào `/var/lib/kubelet/seccomp/profiles/` trên
`k8s-worker-02`:
- `audit.json`: `{"defaultAction": "SCMP_ACT_LOG"}` — log mọi syscall, không chặn gì.
- `violation.json`: `{"defaultAction": "SCMP_ACT_ERRNO"}` — chặn tất cả, minh hoạ lỗi.
- `block-chmod.json`: chặn riêng syscall `chmod`/`fchmod`/`fchmodat`, cho phép mọi syscall khác.

### Bước 2 — Test profile chặn toàn bộ (`violation.json`)

```bash
kubectl apply -f manifests/pod-violation.yaml
kubectl get pod test-violation
# -> STATUS: StartError (chính xác thông báo tuỳ containerd/runc version,
#    có thể là "ContainerCannotRun" trên bản khác — bản chất giống nhau:
#    container không start được vì seccomp chặn cả syscall cần thiết)
kubectl describe pod test-violation | tail -5
# Message: "failed to start containerd task ...: cannot start a stopped process"
```

### Bước 3 — Test profile chặn đúng 1 syscall (`block-chmod.json`)

```bash
kubectl apply -f manifests/pod-block-chmod.yaml
kubectl wait --for=condition=Ready pod/test-block-chmod --timeout=30s

# Syscall không bị chặn vẫn hoạt động bình thường
kubectl exec test-block-chmod -- touch /tmp/f
kubectl exec test-block-chmod -- ls -la /tmp/f

# chmod bị chặn -> "Operation not permitted"
kubectl exec test-block-chmod -- chmod 777 /tmp/f
```

### Bước 4 — Profile `SCMP_ACT_LOG` để "học" syscall trước khi viết rule chặt

```bash
kubectl apply -f manifests/pod-audit.yaml
kubectl wait --for=condition=Ready pod/test-audit --timeout=30s
kubectl exec test-audit -- sh -c "echo hello && sleep 2"
```

Trên node `k8s-worker-02` (cần SSH lần này vì log ghi vào syslog của host,
không đọc được qua kubectl):

```bash
grep syscall /var/log/syslog | tail -20
# hoặc trên hệ thống dùng journald:
journalctl -k | grep -i audit | tail -20
```

Đây là kỹ thuật thật sự được dùng để xây custom profile: chạy ứng dụng với
`SCMP_ACT_LOG`, gom toàn bộ syscall nó thực sự gọi, rồi viết profile
`SCMP_ACT_ERRNO` mặc định + `SCMP_ACT_ALLOW` cho đúng danh sách syscall đó.

## Dọn dẹp

```bash
kubectl delete pod amicontained test-violation test-block-chmod test-audit --ignore-not-found
kubectl delete job write-seccomp-profiles --ignore-not-found
```

(File JSON để lại trên `k8s-worker-02` không gây hại — có thể xoá thủ công
qua SSH nếu muốn: `rm -rf /var/lib/kubelet/seccomp/profiles/*.json`)

## Mẹo thi

- `localhostProfile` là đường dẫn **tương đối** so với thư mục gốc seccomp
  của kubelet (`/var/lib/kubelet/seccomp/` theo mặc định, có thể khác nếu
  `--root-dir` của kubelet bị đổi) — chỉ ghi `profiles/xxx.json`, không ghi
  full path.
- Nếu pod stuck ở `CreateContainerError` sau khi gán seccomp profile, khả
  năng cao là: (1) sai đường dẫn file, (2) file không tồn tại trên đúng node
  pod được schedule tới (seccomp profile phải có mặt trên **từng node**, nên
  production dùng DaemonSet để đồng bộ), hoặc (3) profile chặn cả syscall
  cần thiết để container runtime khởi động process.
- Có thể set `seccompProfile` ở cấp Pod (`spec.securityContext`, áp dụng cho
  mọi container) hoặc cấp Container (`spec.containers[].securityContext`,
  override riêng) — container-level luôn thắng nếu set cả hai.
