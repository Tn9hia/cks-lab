# Lab 2.3 — AppArmor

**Domain:** System Hardening
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** `kubectl` thôi — lab này dùng kỹ thuật "privileged pod +
`nsenter`" để load AppArmor profile vào kernel của node **mà không cần SSH
thủ công**. Đã kiểm chứng trên node Ubuntu 24.04 của cụm bạn (AppArmor kernel
module có sẵn, `apparmor_parser` có sẵn ở `/usr/sbin/apparmor_parser`).

## Mục tiêu

- Hiểu AppArmor cần: (1) kernel module bật, (2) profile được load trên từng
  node, (3) container runtime hỗ trợ — thiếu 1 trong 3 là pod không enforce
  được.
- Viết 1 AppArmor profile đơn giản (deny toàn bộ file write).
- Gán profile vào pod bằng `securityContext.appArmorProfile` (cú pháp GA từ
  Kubernetes 1.30+, thay thế annotation `container.apparmor.security.beta...`
  của bản cũ).

## Cách load profile lên node không cần SSH

Bình thường tài liệu yêu cầu bạn SSH vào từng node, tạo file profile trong
`/etc/apparmor.d/`, rồi chạy `apparmor_parser -q -r <file>`. Ở đây ta làm
tương đương bằng 2 Job chạy trong cluster:

1. **`write-profile-job.yaml`**: dùng `hostPath` mount `/etc/apparmor.d` để
   ghi file profile trực tiếp lên node (giống kỹ thuật ở Lab 2.2).
2. **`load-profile-job.yaml`**: chạy pod `privileged: true` với `hostPID:
   true`, dùng lệnh `nsenter -t 1 -m -u -n -i -- apparmor_parser -q -r
   <file>` để "nhảy" vào namespace của tiến trình PID 1 trên host (init/
   systemd) và chạy đúng `apparmor_parser` **của host** — kết quả giống hệt
   như bạn tự SSH vào chạy lệnh đó.

Cả 2 job đều `nodeName: k8s-worker-02` (pin cứng 1 node) — vì AppArmor
profile chỉ tồn tại trên node nó được load, pod dùng profile này cũng phải
pin vào đúng node đó (`nodeName` trong `ubuntu-sleeper-apparmor.yaml`).
Muốn dùng ở toàn cluster thì đây chính là lý do production luôn deploy các
job/DaemonSet loại này trên **mọi node**, không phải 1 node.

## Bước 1 — Ghi profile lên node

```bash
kubectl apply -f manifests/write-profile-job.yaml
kubectl wait --for=condition=complete job/write-apparmor-profile --timeout=30s
kubectl logs job/write-apparmor-profile
```

Nội dung profile (`apparmor-deny-write`):

```
profile apparmor-deny-write flags=(attach_disconnected) {
  file,
  # Deny all file writes.
  deny /** w,
}
```

## Bước 2 — Load profile vào kernel

```bash
kubectl apply -f manifests/load-profile-job.yaml
kubectl wait --for=condition=complete job/load-apparmor-profile --timeout=30s
```

Verify profile đã ở chế độ enforce (đọc trực tiếp
`/sys/kernel/security/apparmor/profiles` trên host qua cùng kỹ thuật nsenter):

```bash
kubectl run aa-status --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"nodeName":"k8s-worker-02","hostPID":true,"containers":[{"name":"check","image":"busybox:1.36","command":["nsenter","-t","1","-m","-u","-n","-i","--","sh","-c","cat /sys/kernel/security/apparmor/profiles | grep deny-write"],"securityContext":{"privileged":true}}]}}'
kubectl logs aa-status
# -> apparmor-deny-write (enforce)
kubectl delete pod aa-status
```

## Bước 3 — Deploy pod dùng profile và verify enforcement

```bash
kubectl apply -f manifests/ubuntu-sleeper-apparmor.yaml
kubectl wait --for=condition=Ready pod/ubuntu-sleeper --timeout=30s
kubectl logs ubuntu-sleeper
# -> "Sleeping for an hour!"

kubectl exec ubuntu-sleeper -- touch /tmp/test
# -> touch: cannot touch '/tmp/test': Permission denied
```

Container **có** quyền Linux/RBAC để ghi `/tmp` bình thường — chỉ AppArmor
mới chặn. Đây là điểm khác biệt cần nhớ: AppArmor hoạt động ở **tầng kernel
LSM** (Linux Security Module), độc lập hoàn toàn với RBAC/SecurityContext
capabilities.

## Bước 4 (tự làm thêm) — Tạo profile bằng `aa-genprof`

Tài liệu gốc còn giới thiệu cách sinh profile tự động bằng `aa-genprof`
(chạy thử ứng dụng ở chế độ "complain", AppArmor ghi lại mọi hành vi rồi hỏi
bạn allow/deny từng cái) — đây là cách làm thực tế hơn nhiều so với viết tay
`deny /** w,`. Vì `aa-genprof` cần chạy tương tác (interactive) nên hợp làm
qua SSH thật hơn là qua Job. Nếu muốn luyện thêm: SSH vào 1 node, cài
`apparmor-utils` (`apt-get install -y apparmor-utils`), viết 1 script bash
nhỏ, chạy `aa-genprof ./script.sh`, quan sát flow interactive.

## Dọn dẹp

> **Đã kiểm chứng — gotcha quan trọng:** khi mình test lab này,
> `kubectl delete pod ubuntu-sleeper` bị **kẹt ở trạng thái `Terminating`
> nhiều phút**, không tự hết. `kubectl describe pod` cho thấy lý do:
> `FailedKillPod ... unable to signal init: permission denied`. Nguyên nhân:
> profile `apparmor-deny-write` chỉ khai `file,` + `deny /** w,` mà không có
> rule `signal`, nên trên kernel AppArmor mới (Ubuntu 24.04) — nơi signal
> mediation được bật mặc định — **chính runc cũng bị AppArmor chặn không
> gửi được SIGKILL vào container để dừng nó**. Đây là một pod tự làm mình
> "không thể bị kill" chỉ vì viết thiếu 1 dòng trong profile.
>
> **Cách gỡ khi gặp tình huống này:** gỡ profile khỏi kernel host TRƯỚC
> (bước dưới), việc gỡ sẽ giải phóng signal ngay cho process đang chạy dở,
> sau đó `kubectl delete pod` sẽ hoàn tất bình thường trong vài giây.

Gỡ profile khỏi kernel của node:

```bash
kubectl run aa-unload --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"nodeName":"k8s-worker-02","hostPID":true,"containers":[{"name":"unload","image":"busybox:1.36","command":["nsenter","-t","1","-m","-u","-n","-i","--","apparmor_parser","-R","/etc/apparmor.d/apparmor-deny-write"],"securityContext":{"privileged":true}}]}}'
sleep 5
kubectl delete pod aa-unload --ignore-not-found
```

Rồi mới xoá pod + job:

```bash
kubectl delete pod ubuntu-sleeper --ignore-not-found
kubectl delete job write-apparmor-profile load-apparmor-profile --ignore-not-found
```

## Mẹo thi

- Trong đề thi thật, profile **luôn được cho sẵn và đã load trên node** —
  bạn chỉ cần biết cách **gán** nó vào pod đúng cú pháp
  (`securityContext.appArmorProfile.localhostProfile: <tên-profile>`, tên
  này lấy từ dòng `profile <tên> {` trong file, không phải tên file).
- `type: Localhost` cần đi kèm `localhostProfile`; nếu chỉ cần dùng profile
  mặc định của container runtime thì dùng `type: RuntimeDefault` (giống hệt
  logic của seccomp ở Lab 2.2).
- Nếu pod bị `Blocked`/`CreateContainerError` sau khi gán appArmorProfile,
  nguyên nhân gần như luôn là: profile chưa được load trên đúng node pod bị
  schedule tới. Luôn kiểm tra bằng `aa-status` trên node đó trước khi nghi
  ngờ cú pháp YAML sai.
- Profile viết thiếu rule có thể khiến pod **không xoá được** (xem phần Dọn
  dẹp bên dưới — đã tự gặp phải khi làm lab này). Trong đề thi thực tế bạn
  hiếm khi tự viết profile từ đầu, nhưng nếu được yêu cầu viết, đừng chỉ copy
  đúng 3 dòng `deny /** w,` — hiểu rằng profile quá tối giản có thể gây tác
  dụng phụ ngoài ý muốn.
