# Lab 1.5 — Bảo vệ Node Metadata

**Domain:** Cluster Hardening — Restrict access to Node Metadata
**Thời gian gợi ý:** 20 phút

## Bối cảnh

"Node metadata" trong CKS có 2 khía cạnh cần phân biệt:

1. **Metadata trong Kubernetes API** (`kubectl get nodes -o yaml`): tên node,
   IP nội bộ, phiên bản kubelet, kernel version, labels, taints... Nếu user
   thường/attacker đọc được, họ có thể dò bản kubelet/kernel để tìm CVE phù
   hợp, hoặc vẽ bản đồ mạng nội bộ từ danh sách IP.
2. **Cloud provider instance metadata API** (`169.254.169.254` trên
   AWS/GCP/Azure): nếu một pod bị chiếm quyền có thể gọi tới endpoint này, nó
   có thể lấy được IAM credentials/service account token của chính node —
   đây là một trong những đường leo thang quyền phổ biến nhất trong môi
   trường cloud-native (SSRF -> cloud credential theft).

Cụm thực hành của bạn không chạy trên cloud nên phần 2 mang tính mô phỏng,
nhưng cách chặn (NetworkPolicy egress) áp dụng y hệt trên EKS/GKE/AKS thật.

## Phần A — Hạn chế đọc Node metadata qua Kubernetes API

### Bước 1 — Mô phỏng "attacker" là 1 SA có quyền hạn chế

```bash
kubectl create namespace lab-metadata
kubectl create serviceaccount limited-sa -n lab-metadata
kubectl apply -f manifests/limited-role.yaml   # chỉ cho get/list pods trong namespace này
```

### Bước 2 — Chứng minh mặc định SA "hàng xóm" không đọc được node info (đúng như mong đợi vì RBAC là deny-by-default)

```bash
kubectl auth can-i list nodes --as=system:serviceaccount:lab-metadata:limited-sa
# -> no (mặc định không có quyền gì ngoài RBAC được cấp)

kubectl auth can-i get nodes --as=system:serviceaccount:lab-metadata:limited-sa
# -> no
```

Đây chính là bài học chính: **không có "opt-out" cho node metadata riêng** —
nó được bảo vệ bằng đúng cơ chế RBAC như mọi resource khác. Rủi ro thực tế
nằm ở việc **cấp dư quyền** kiểu này:

```bash
kubectl apply -f manifests/overprivileged-role.yaml
kubectl auth can-i get nodes --as=system:serviceaccount:lab-metadata:limited-sa
# -> yes (vì overprivileged-role lỡ cho get trên "nodes")
```

Nếu "yes", một pod dùng SA này có thể chạy được:

```bash
kubectl run attacker --image=bitnami/kubectl -n lab-metadata \
  --overrides='{"spec":{"serviceAccountName":"limited-sa"}}' --command -- sleep 3600
kubectl exec -n lab-metadata attacker -- kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'
kubectl exec -n lab-metadata attacker -- kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

-> Attacker giờ biết chính xác kubelet version (để tra CVE) và toàn bộ IP nội
bộ (để tấn công mạng). Đây là lý do luôn phải audit kỹ mọi Role/ClusterRole
có `resources: ["nodes"]`.

### Bước 3 — Sửa lại: gỡ quyền thừa

```bash
kubectl delete -f manifests/overprivileged-role.yaml
kubectl auth can-i get nodes --as=system:serviceaccount:lab-metadata:limited-sa
# -> no
```

## Phần B — Chặn pod truy cập Cloud Metadata API (mô phỏng)

### Bước 1 — Baseline: pod gọi được "metadata endpoint" (mô phỏng bằng 1 IP nội bộ đóng vai trò 169.254.169.254)

Vì cụm không có cloud metadata thật, ta mô phỏng bằng NetworkPolicy chặn hẳn
dải IP `169.254.169.254/32` — áp dụng được ngay cả khi không có server nào ở
đó, mục đích là luyện đúng cú pháp NetworkPolicy dùng trong đề thi/thực tế.

```bash
kubectl apply -f manifests/deny-metadata-egress.yaml
```

Nội dung chính: default-deny egress tới `169.254.169.254/32` cho **mọi pod**
trong namespace, trong khi vẫn cho phép egress bình thường tới nơi khác
(dùng `ipBlock.except`).

### Bước 2 — Verify

```bash
kubectl run test-pod --image=curlimages/curl -n lab-metadata --command -- sleep 3600
kubectl wait --for=condition=Ready pod/test-pod -n lab-metadata --timeout=30s

# Bị chặn (timeout) vì nằm trong dải except
kubectl exec -n lab-metadata test-pod -- curl -m 3 -s -o /dev/null -w "%{http_code}\n" http://169.254.169.254/ || echo "blocked (đúng)"

# DNS (được cho phép riêng qua rule tới kube-system:53) vẫn hoạt động bình thường
kubectl exec -n lab-metadata test-pod -- nslookup kubernetes.default.svc.cluster.local
```

> **Đã kiểm chứng — lưu ý quan trọng về Cilium:** nếu bạn thử `curl` thẳng
> tới IP của một pod khác trong cùng cluster (ví dụ `10.244.0.27`) để "chứng
> minh traffic khác không bị chặn", bạn sẽ thấy nó **cũng bị chặn** dù IP đó
> không nằm trong `except`. Đây không phải do policy sai: Cilium enforce
> `ipBlock` theo cơ chế identity-based cho traffic **nội bộ cluster** (pod
> IP), nên rule CIDR như trên chủ yếu có ý nghĩa với traffic đi **ra ngoài
> cluster** (đúng use-case chặn cloud metadata). Muốn cho phép lại traffic
> tới pod khác, phải khai thêm rule `podSelector`/`namespaceSelector` riêng
> (như đã làm ở Lab 1.4) — `ipBlock` và `podSelector` không tự động cộng dồn
> theo cách bạn nghĩ nếu chỉ dựa vào CIDR. Đây là kiến thức đặc thù Cilium,
> không phải hành vi chuẩn NetworkPolicy nói chung — trên CNI khác (Calico,
> Weave) traffic pod-to-pod theo IP thường không bị ảnh hưởng bởi rule CIDR
> kiểu này.

## Dọn dẹp

```bash
kubectl delete namespace lab-metadata
```

## Mẹo thi

- CKS không kỳ vọng bạn tự dựng cloud metadata server — điều quan trọng là
  **biết viết NetworkPolicy chặn egress tới 1 IP/CIDR cụ thể** bằng
  `ipBlock` + `except`, đây là kỹ năng transferable dùng được cho mọi tình
  huống "chặn truy cập ra ngoài tới địa chỉ X".
- Trên EKS thật, cách chuẩn hơn NetworkPolicy là bật
  **IMDSv2 + hop limit = 1** ở tầng EC2 (ngoài phạm vi Kubernetes) — CKS chỉ
  test phần bạn làm được trong Kubernetes, nhưng nên biết bối cảnh này khi
  được hỏi lý thuyết.
- Luôn tự hỏi "ClusterRole/Role nào đang có `resources: ["nodes"]` hoặc
  `resources: ["nodes/proxy"]`" khi audit RBAC — đây là quyền nhạy cảm hay bị
  cấp dư thừa nhất vì nhiều ClusterRole mẫu có sẵn (như `view`) đã bao gồm
  quyền đọc nodes.
