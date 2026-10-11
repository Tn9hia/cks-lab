# Lab 1.1 — RBAC từ số 0

**Domain:** Cluster Hardening — RBAC
**Thời gian gợi ý:** 25-30 phút
**Yêu cầu:** `kubectl` đã trỏ tới cụm (đã xác nhận OK).

## Mục tiêu

- Tạo `ServiceAccount`, `Role`, `RoleBinding` giới hạn quyền chỉ đọc Pods trong 1 namespace.
- Hiểu khác biệt giữa `Role` (namespace-scoped) và `ClusterRole` khi bind bằng `RoleBinding` (namespace-scoped) vs `ClusterRoleBinding` (cluster-wide).
- Dùng `kubectl auth can-i` và impersonation để verify quyền — đây là kỹ năng bắt buộc trong đề thi thật vì bạn không có nhiều thời gian để test bằng cách tạo pod thật.

## Phần A — Role + RoleBinding cơ bản

1. Tạo namespace và ServiceAccount:

```bash
kubectl create namespace lab-rbac
kubectl create serviceaccount pod-reader-sa -n lab-rbac
```

2. Áp dụng Role + RoleBinding (file có sẵn ở `manifests/part-a-role.yaml`):

```bash
kubectl apply -f manifests/part-a-role.yaml
```

3. Tạo vài pod mẫu để test:

```bash
kubectl run nginx-1 --image=nginx -n lab-rbac
kubectl run nginx-2 --image=nginx -n lab-rbac
```

4. **Verify bằng impersonation** (không cần deploy pod thật để lấy token):

```bash
# Được phép: list pods trong lab-rbac
kubectl auth can-i list pods -n lab-rbac \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
# -> yes

# Không được phép: xoá pod
kubectl auth can-i delete pods -n lab-rbac \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
# -> no

# Không được phép: đọc secrets
kubectl auth can-i get secrets -n lab-rbac \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
# -> no

# Không được phép: list pods ở namespace khác (default)
kubectl auth can-i list pods -n default \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
# -> no
```

5. **Verify thật với pod chạy SA đó** (mô phỏng đúng cách app trong cluster gọi API server):

```bash
kubectl apply -f manifests/part-a-test-pod.yaml
kubectl exec -n lab-rbac -it rbac-test-pod -- sh
```

Trong shell của pod, gọi thẳng API server bằng token của SA:

```sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
APISERVER=https://kubernetes.default.svc

# Phải trả về 200 + danh sách pod
curl -sS --cacert $CACERT -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/lab-rbac/pods | head -c 300

# Phải trả về 403 Forbidden
curl -sS -o /dev/null -w "%{http_code}\n" --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  $APISERVER/api/v1/namespaces/lab-rbac/secrets
```

## Phần B — Bẫy thường gặp: ClusterRole + RoleBinding

Đây là điểm hay bị nhầm trong đề thi: bind một `ClusterRole` bằng `RoleBinding`
sẽ **chỉ** cấp quyền trong namespace của RoleBinding đó, kể cả khi ClusterRole
liệt kê resource cluster-scoped.

```bash
kubectl apply -f manifests/part-b-clusterrole.yaml
```

File này tạo:
- `ClusterRole node-and-pod-reader`: cho phép `get/list` trên `nodes` (cluster-scoped) và `pods` (namespaced).
- `RoleBinding` trong namespace `lab-rbac` gán ClusterRole đó cho SA `pod-reader-sa`.

Verify:

```bash
# Pods trong lab-rbac: yes (đúng như mong đợi)
kubectl auth can-i list pods -n lab-rbac \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa

# Nodes: PHẢI là "no" dù ClusterRole cho phép — vì RoleBinding giới hạn theo namespace
# và "nodes" không thuộc namespace nào nên request bị chặn ở tầng authorization
kubectl auth can-i list nodes \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
```

Giờ đổi sang `ClusterRoleBinding` để cấp quyền thật sự cluster-wide:

```bash
kubectl apply -f manifests/part-b-clusterrolebinding.yaml

kubectl auth can-i list nodes \
  --as=system:serviceaccount:lab-rbac:pod-reader-sa
# -> yes
```

## Phần C — Tự làm thêm (không có đáp án sẵn)

Để luyện phản xạ, tự viết RBAC cho các tình huống sau (bài thi hay ra dạng này):

1. Một user `jane` (dùng cert, xem Lab 1.2) chỉ được `create` và `get` Deployments
   trong namespace `blue`, không được `delete`.
2. Một ServiceAccount `ci-bot` cần quyền `get/list/watch` trên `pods`, `deployments`,
   `replicasets` ở **mọi namespace** nhưng tuyệt đối không được đụng vào `secrets`
   hay `configmaps`.
3. Kiểm tra quyền hiện tại của chính bạn (user đang dùng kubeconfig) bằng
   `kubectl auth can-i --list`.

## Dọn dẹp

```bash
kubectl delete namespace lab-rbac
kubectl delete -f manifests/part-b-clusterrole.yaml --ignore-not-found
kubectl delete -f manifests/part-b-clusterrolebinding.yaml --ignore-not-found
```

## Mẹo thi

- Luôn dùng `kubectl auth can-i ... --as=<user/SA>` để verify nhanh thay vì tạo pod thật — tiết kiệm thời gian đáng kể.
- `kubectl create role/clusterrole/rolebinding` bằng CLI (`--verb`, `--resource`) nhanh hơn viết YAML tay khi vào phòng thi.
- Nhớ phân biệt rõ: `Role`+`RoleBinding` = trong 1 namespace; `ClusterRole`+`ClusterRoleBinding` = toàn cluster; `ClusterRole`+`RoleBinding` = quyền của ClusterRole đó nhưng giới hạn trong 1 namespace (dùng để tái sử dụng ClusterRole có sẵn như `view`, `edit`, `admin`).
