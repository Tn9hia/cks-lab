# Lab 2.1 — Linux Capabilities

**Domain:** System Hardening
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** chỉ cần `kubectl`, không cần SSH node.

## Mục tiêu

- Hiểu vì sao container chạy `root` vẫn bị chặn một số hành động (vd. đổi giờ hệ thống) — vì container chỉ có ~14 capability mặc định, không phải full root.
- Biết cách `drop`/`add` capability trong `securityContext`.
- Áp dụng nguyên tắc least-privilege: `drop: ALL` rồi chỉ `add` đúng capability cần thiết.

## Bước 1 — Chứng minh root trong container KHÔNG phải root thật

```bash
kubectl run ubuntu-sleeper --image=ubuntu --restart=Never -- sleep 3600
kubectl wait --for=condition=Ready pod/ubuntu-sleeper --timeout=30s

kubectl exec -it ubuntu-sleeper -- whoami
# -> root

kubectl exec -it ubuntu-sleeper -- date -s '19 APR 2012 22:00:00'
# -> date: cannot set date: Operation not permitted (hoặc "Permission denied" tuỳ bản coreutils)
```

Dù `whoami` báo `root`, lệnh đổi giờ vẫn bị từ chối vì container mặc định
không có `CAP_SYS_TIME` (Docker/containerd chỉ cấp ~14 capability mặc định:
`CAP_CHOWN`, `CAP_NET_RAW`, `CAP_SETUID`... không có `CAP_SYS_TIME`,
`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`...).

> **CẢNH BÁO — đã kiểm chứng thực tế, đọc kỹ trước khi làm Bước 2:**
> `CLOCK_REALTIME` (đồng hồ hệ thống) **không bị cô lập theo namespace** trong
> Linux — kể cả Linux time namespace (kernel 5.6+) cũng chỉ cô lập
> `CLOCK_MONOTONIC`/`CLOCK_BOOTTIME`, không cô lập giờ thực. Nghĩa là khi một
> container có `CAP_SYS_TIME` và chạy `date -s`, nó **đổi luôn đồng hồ của
> node host thật**, ảnh hưởng đến toàn bộ pod khác đang chạy trên node đó
> (TLS certificate validation cho image pull sẽ fail nếu giờ sai quá xa).
> Mình đã tự làm sập image-pull trên 1 worker node của bạn khi test lab này
> và phải tự sửa lại giờ. **Nếu bạn tự làm lab này: chạy Bước 2 xong, verify
> ngay, và luôn có sẵn lệnh set lại giờ đúng (`date -s "$(date -u
> +'%Y-%m-%d %H:%M:%S') UTC"` với giờ lấy từ 1 node/máy khác chưa bị đổi) để
> revert ngay lập tức.**

## Bước 2 — Add capability cụ thể để cho phép đổi giờ (rồi trả lại giờ đúng NGAY)

```bash
kubectl delete pod ubuntu-sleeper
kubectl apply -f manifests/add-cap-sys-time.yaml
kubectl wait --for=condition=Ready pod/ubuntu-sleeper --timeout=30s

# Ghi lại giờ đúng hiện tại TRƯỚC khi đổi, để revert ngay sau đó
date -u

kubectl exec ubuntu-sleeper -- date -s '19 APR 2012 22:00:00'
# -> thành công (không còn bị chặn) — CHỨNG MINH capability hoạt động

# REVERT NGAY LẬP TỨC — thay giá trị bằng giờ UTC hiện tại thật
kubectl exec ubuntu-sleeper -- date -s '<giờ UTC hiện tại, ví dụ: 2026-09-09 18:00:00>'
```

> Vì lệnh trên đổi giờ của cả node, hãy verify lại node đó vẫn hoạt động
> bình thường sau khi revert: `kubectl get nodes`, và thử chạy 1 pod mới
> dùng image chưa cache trên node đó để chắc chắn TLS/image-pull không bị
> ảnh hưởng.

## Bước 3 — Drop capability để chứng minh chiều ngược lại

```bash
kubectl delete pod ubuntu-sleeper
kubectl apply -f manifests/drop-cap-chown.yaml
kubectl wait --for=condition=Ready pod/ubuntu-sleeper --timeout=30s

kubectl exec ubuntu-sleeper -- touch /tmp/testfile
kubectl exec ubuntu-sleeper -- chown nobody /tmp/testfile
# -> chown: changing ownership of '/tmp/testfile': Operation not permitted
# (CAP_CHOWN vốn có mặc định, nhưng đã bị drop tường minh)
```

## Bước 4 — Least privilege thật sự: drop ALL, chỉ add đúng capability cần thiết

Tình huống thực tế: nginx cần bind port 80 (thường yêu cầu root vì port
<1024), cần `chown`/`setuid`/`setgid` lúc khởi động để tự hạ quyền worker
process xuống user `nginx`. Thay vì giữ nguyên 14 capability mặc định (nhiều
cái không dùng tới, ví dụ `NET_RAW`, `AUDIT_WRITE`), ta drop hết rồi chỉ add
đúng 4 cái nginx thực sự cần:

```bash
kubectl apply -f manifests/nginx-minimal-caps.yaml
kubectl wait --for=condition=Ready pod/nginx-minimal-caps --timeout=30s

kubectl exec nginx-minimal-caps -- curl -s -o /dev/null -w "%{http_code}\n" http://localhost:80
# -> 200, nginx chạy tốt dù chỉ còn 4/14 capability
```

Xem `manifests/nginx-minimal-caps.yaml`: `capabilities.drop: [ALL]`, chỉ
`add: [NET_BIND_SERVICE, CHOWN, SETUID, SETGID]`. Container vẫn chạy
`runAsUser: 0` (root) vì bản thân master process của nginx cần quyền root để
tự `setuid`/`setgid` xuống user `nginx` cho worker process — đây là lý do
`CHOWN`/`SETUID`/`SETGID` vẫn cần giữ lại dù không chạy full 14 capability.
Nếu bạn thử đổi sang `runAsUser: 101` (non-root) mà không sửa thêm gì khác,
nginx sẽ crash vì không `mkdir` được `/var/cache/nginx/client_temp` (thư mục
này do image gốc tạo sẵn với owner root) — đây là ví dụ cho thấy
**capabilities và filesystem permissions là hai lớp bảo mật độc lập**, drop
capability không thay thế được việc chỉnh quyền file (`securityContext.fsGroup`,
custom base image, hoặc init container `chown`).

## Dọn dẹp

```bash
kubectl delete pod ubuntu-sleeper nginx-minimal-caps --ignore-not-found
```

## Mẹo thi

- `getcap <binary>` cho biết binary yêu cầu capability gì (vd `getcap
  /usr/bin/ping` -> `cap_net_raw+ep`); `getpcaps <pid>` cho biết 1 process
  đang có capability nào — cả hai đều hay dùng để debug "vì sao container
  X không làm được Y".
- Luôn viết tên capability trong YAML **không có tiền tố `CAP_`** (vd
  `NET_BIND_SERVICE` chứ không phải `CAP_NET_BIND_SERVICE`) — đây là lỗi cú
  pháp rất hay gặp khi làm bài thi vội.
- Pattern chuẩn cho production: luôn `drop: ["ALL"]` trước, sau đó chỉ `add`
  đúng những gì ứng dụng thật sự cần — không bao giờ giữ nguyên default set
  nếu không rõ ứng dụng cần capability nào.
- **`CAP_SYS_TIME`, `CAP_SYS_ADMIN`, `CAP_SYS_MODULE`, `CAP_NET_ADMIN`** là
  nhóm capability nguy hiểm nhất vì chúng có thể ảnh hưởng tới **cả node
  host**, không chỉ container — đây chính xác là loại câu hỏi CKS hay hỏi
  "vì sao không nên cấp capability X cho container".
