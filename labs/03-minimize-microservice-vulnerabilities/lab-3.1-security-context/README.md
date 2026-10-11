# Lab 3.1 — Security Context

**Domain:** Minimize Microservice Vulnerabilities
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** chỉ cần `kubectl`.

## Mục tiêu

- Áp dụng `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`.
- Hiểu thứ tự ưu tiên khi set securityContext ở cả cấp Pod và Container.
- Xử lý đúng lỗi phát sinh khi bật `readOnlyRootFilesystem` (ứng dụng cần ghi tạm).

## Bước 1 — `runAsNonRoot` chặn image chạy root

```bash
kubectl apply -f manifests/01-run-as-nonroot-fail.yaml
kubectl get pod nonroot-fail
```

`nginx` image mặc định chạy `USER root` trong Dockerfile — khi ép
`runAsNonRoot: true` mà không set `runAsUser`, Kubernetes biết image sẽ chạy
root và **từ chối start container** trước cả khi thử:

```bash
kubectl describe pod nonroot-fail | grep -A3 "Reason\|Message"
# -> Error: container has runAsNonRoot and image will run as root
```

## Bước 2 — Sửa đúng: chỉ định `runAsUser` cụ thể

Dùng `busybox` thay vì `nginx` ở bước này để tách riêng khái niệm "chạy đúng
UID" khỏi vấn đề quyền ghi thư mục cache của nginx (sẽ xử lý riêng ở Bước 3-4
— nếu bạn thử đổi image này thành `nginx` ngay tại đây, nó sẽ crash vì
`/var/cache/nginx` do image build sẵn với owner root, không liên quan gì tới
đúng/sai của `runAsNonRoot`).

```bash
kubectl apply -f manifests/02-run-as-nonroot-fixed.yaml
kubectl wait --for=condition=Ready pod/nonroot-fixed --timeout=30s
kubectl logs nonroot-fixed
# -> uid=101 gid=0(root)... không phải uid=0
```

## Bước 3 — `readOnlyRootFilesystem` và cách xử lý khi app cần ghi

```bash
kubectl apply -f manifests/03-readonly-root-fail.yaml
kubectl wait --for=condition=Ready pod/readonly-fail --timeout=30s 2>&1 || true
kubectl logs readonly-fail --tail=20
```

nginx sẽ lỗi vì cần ghi PID file / cache dir mà root filesystem đã readonly.
Cách chuẩn để sửa: mount `emptyDir` đúng vào các thư mục nginx cần ghi, giữ
nguyên phần còn lại của filesystem readonly (đúng nguyên tắc least-privilege
— chỉ mở đúng chỗ cần, không tắt hẳn readonly).

```bash
kubectl apply -f manifests/04-readonly-root-fixed.yaml
kubectl wait --for=condition=Ready pod/readonly-fixed --timeout=30s
kubectl exec readonly-fixed -- curl -s -o /dev/null -w "%{http_code}\n" http://localhost
# -> 200

kubectl exec readonly-fixed -- touch /usr/share/nginx/html/test
# -> Read-only file system (đúng, vì thư mục này KHÔNG nằm trong danh sách emptyDir mount)
```

## Bước 4 — Kết hợp đầy đủ + `allowPrivilegeEscalation: false`

```bash
kubectl apply -f manifests/05-full-hardened.yaml
kubectl wait --for=condition=Ready pod/full-hardened --timeout=30s
kubectl exec full-hardened -- curl -s -o /dev/null -w "%{http_code}\n" http://localhost
```

Xem `manifests/05-full-hardened.yaml`: kết hợp toàn bộ — `runAsNonRoot`,
`runAsUser`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`,
`capabilities.drop: [ALL]` + add đúng `NET_BIND_SERVICE` (từ Lab 2.1). Đây
là pattern "hardened pod" chuẩn hay xuất hiện trong đề thi CKS.

## Bước 5 — Container-level override Pod-level

```bash
kubectl apply -f manifests/06-override-precedence.yaml
kubectl exec override-demo -- id
```

Pod-level set `runAsUser: 1000`, nhưng container `app` override riêng
`runAsUser: 2000` — container-level luôn thắng. Verify `id` trả về uid 2000,
không phải 1000.

## Dọn dẹp

```bash
kubectl delete pod nonroot-fail nonroot-fixed readonly-fail readonly-fixed full-hardened override-demo --ignore-not-found
```

## Mẹo thi

- `runAsNonRoot: true` chỉ **kiểm tra** UID khác 0, không tự đổi UID — luôn
  phải đi kèm `runAsUser: <uid khác 0>` nếu image mặc định chạy root.
- `readOnlyRootFilesystem: true` gần như luôn cần thêm `emptyDir` cho các
  thư mục ứng dụng cần ghi (log, cache, pid file, socket) — đọc kỹ log lỗi
  của container để biết chính xác thư mục nào cần mount thêm thay vì đoán.
- Thứ tự ưu tiên: `securityContext` ở **container** override
  `securityContext` ở **Pod** cho field trùng tên; field không set ở
  container sẽ kế thừa từ Pod.
