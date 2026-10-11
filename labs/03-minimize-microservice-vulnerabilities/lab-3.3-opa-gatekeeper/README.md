# Lab 3.3 — OPA Gatekeeper (+ ValidatingAdmissionPolicy đã kiểm chứng)

**Domain:** Minimize Microservice Vulnerabilities
**Thời gian gợi ý:** 30 phút

## Quan trọng — đọc trước

Notes KodeKloud dạy OPA theo hướng **Gatekeeper** (không phải raw OPA +
custom webhook). Mình đã cài thử `Gatekeeper v3.14.0` (bản khuyến nghị trong
tài liệu) lên đúng cụm của bạn (Kubernetes v1.36.0) và phát hiện **2 vấn đề
thực tế nghiêm trọng**:

1. **ConstraintTemplate mẫu trong tài liệu bị sai tên**: ví dụ gốc dùng
   `metadata.name: systemrequiredlabels` (số nhiều) trong khi `kind:
   SystemRequiredLabel` (số ít) — Gatekeeper bắt buộc `metadata.name` phải
   đúng bằng lowercase của `kind`. Dùng đúng tên gốc từ tài liệu sẽ lỗi
   `create_error: ConstraintTemplate's name ... is not equal to the
   lowercase of CRD's Kind`. Đã sửa bằng cách đổi tên thành
   `systemrequiredlabel` (số ít) trong file `requiredlabels-template.yaml`.

2. **Nghiêm trọng hơn — webhook không nhận được request cho Pod dù cấu hình
   đúng**: sau khi sửa lỗi (1), tạo pod vi phạm policy vẫn được **chấp
   nhận** thay vì bị từ chối. Kiểm tra log `gatekeeper-controller-manager`
   cho thấy **không có bất kỳ admission review nào được ghi nhận** cho pod
   đó — nghĩa là `kube-apiserver` không thực sự gọi tới webhook của
   Gatekeeper cho resource Pod, dù rule trong `ValidatingWebhookConfiguration`
   khớp đúng. Ngoài ra, webhook phụ `check-ignore-label` (dùng
   `failurePolicy: Fail`) từng **chặn đứng toàn bộ việc tạo namespace mới
   trên cụm** trong lúc mình test — mình đã phải patch `failurePolicy` của
   nó sang `Ignore` để cứu cụm, rồi cuối cùng **gỡ hẳn Gatekeeper** vì không
   xác định được nguyên nhân gốc trong thời gian hợp lý (khả năng cao là
   version skew — Gatekeeper v3.14.0 phát hành trước khi Kubernetes v1.36
   tồn tại, nhiều khả năng chưa test tương thích với admission API mới nhất).

**Kết luận:** nếu bạn muốn tự cài Gatekeeper để luyện đúng y hệt giáo trình,
**hãy làm trên 1 cụm test riêng, không phải cụm chính**, và biết trước rằng
có rủi ro webhook `failurePolicy: Fail` làm nghẽn cluster nếu Gatekeeper gặp
sự cố tương thích tương tự. Phần hướng dẫn cài đặt bên dưới viết đúng theo
tài liệu (đã sửa lỗi tên ConstraintTemplate) nhưng **chưa xác nhận hoạt động
đúng trên mọi phiên bản Kubernetes** — coi đây là tài liệu tham khảo, không
phải "đã test thành công" như các lab khác trong bộ này.

Để bạn vẫn có phần thực hành **đã kiểm chứng hoạt động đúng 100% trên cụm
của bạn**, mình bổ sung thêm cách làm tương đương bằng
**`ValidatingAdmissionPolicy`** (tính năng GA từ Kubernetes 1.30, built-in,
không cần cài gì thêm, không có webhook nên không có rủi ro fail-open/
fail-closed như trên) ở Phần B. Đây không phải nội dung trong notes CKS gốc,
nhưng là kiến thức đang ngày càng quan trọng và giải quyết đúng bài toán mà
Gatekeeper/OPA được dạy để giải quyết.

## Phần A — OPA Gatekeeper (theo tài liệu, đã sửa lỗi tên, CHƯA verify trên K8s 1.36)

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml
kubectl wait --for=condition=Ready pod -n gatekeeper-system --all --timeout=90s
kubectl get all -n gatekeeper-system
```

```bash
kubectl apply -f manifests/requiredlabels-template.yaml   # đã sửa: name = systemrequiredlabel (số ít)
kubectl apply -f manifests/require-label-billing.yaml
kubectl create namespace expensive
kubectl run test-pod -n expensive --image=nginx   # kỳ vọng: bị từ chối vì thiếu label "billing"
```

Nếu bạn tự làm và gặp lỗi tương tự mình đã gặp:

```bash
# Nếu ufw/namespace bị nghẽn vì check-ignore-label webhook (failurePolicy: Fail):
kubectl patch validatingwebhookconfiguration gatekeeper-validating-webhook-configuration \
  --type='json' -p='[{"op": "replace", "path": "/webhooks/1/failurePolicy", "value": "Ignore"}]'

# Gỡ hoàn toàn nếu cần:
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml
```

### Constraint Template gốc (Rego)

`manifests/requiredlabels-template.yaml` — chú ý field đúng cho Gatekeeper
là `input.review.object` (không phải `input.request.object` như ví dụ raw-
webhook trong bài "OPA in Kubernetes" của tài liệu — 2 field input khác
nhau tuỳ bạn dùng Gatekeeper hay tự viết webhook, đây là điểm dễ nhầm khi
đọc tài liệu vì 2 lesson liền kề dùng 2 quy ước input khác nhau).

## Phần B — ValidatingAdmissionPolicy (ĐÃ TEST THÀNH CÔNG trên cụm bạn)

Cùng bài toán "namespace X phải có label billing trên mọi Pod", giải quyết
bằng CEL (Common Expression Language) — built-in Kubernetes, không cần cài
gì, không có webhook nên không có rủi ro fail-open/fail-closed:

```bash
kubectl create namespace expensive
kubectl apply -f manifests/vap-require-billing-label.yaml
```

Nội dung `vap-require-billing-label.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-billing-label
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: "'billing' in object.metadata.labels"
      message: "you must provide label: billing"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-billing-label-binding
spec:
  policyName: require-billing-label
  validationActions: ["Deny"]
  matchResources:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: expensive
```

Verify:

```bash
echo "--- pod thiếu label billing (expect DENY) ---"
kubectl run test-pod -n expensive --image=nginx
# -> "ValidatingAdmissionPolicy 'require-billing-label' ... denied request:
#     you must provide label: billing"

echo "--- pod có label billing (expect ALLOW) ---"
kubectl run test-pod2 -n expensive --image=nginx --labels="billing=yes"
# -> pod/test-pod2 created
```

## Dọn dẹp

```bash
kubectl delete namespace expensive
# Nếu có làm Phần A:
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml --ignore-not-found
```

## Mẹo thi

- **Đề thi CKS hiện tại vẫn hỏi OPA/Gatekeeper theo đúng khái niệm Rego +
  ConstraintTemplate/Constraint** — bạn vẫn cần hiểu cú pháp Rego cơ bản
  (`violation[{"msg": msg}] { ... }`, `input.review.object`) dù bản thân
  Gatekeeper có thể không được cài sẵn trong môi trường thi (thường đề bài
  cho sẵn cụm đã cài Gatekeeper nếu câu hỏi liên quan tới nó).
- `ValidatingAdmissionPolicy`/CEL **chưa chắc có trong đề thi** (tuỳ phiên
  bản curriculum tại thời điểm bạn thi) nhưng là xu hướng thay thế
  Gatekeeper cho các rule đơn giản — biết thêm không thừa, và an toàn hơn
  nhiều để tự luyện tập vì không có thành phần ngoài (pod, webhook, cert) có
  thể gây sự cố như Gatekeeper.
- Bài học lớn nhất từ sự cố mình gặp: **bất kỳ ValidatingWebhookConfiguration
  nào có `failurePolicy: Fail` đều có khả năng "brick" một phần cluster**
  nếu webhook server đó down hoặc không tương thích — luôn kiểm tra
  `failurePolicy` khi audit 1 cụm lạ, và cân nhắc kỹ trước khi tự thêm
  webhook `Fail` vào cụm production.
