# Lab 4.5 — Whitelist Allowed Registries (ImagePolicyWebhook)

**Domain:** Supply Chain Security
**Thời gian gợi ý:** 30 phút

> **Lưu ý về trạng thái test:** lab này được viết trong lúc kết nối mạng từ
> máy làm việc tới cụm bị gián đoạn (VPN/tunnel rớt giữa chừng, không liên
> quan tới thao tác trong lab). Phần A (ImagePolicyWebhook) viết theo đúng
> tài liệu, mức rủi ro tương đương Lab 3.4/4.5 (sửa `kube-apiserver` +
> triển khai webhook ngoài) nên vốn dĩ đã được lên kế hoạch là "hướng dẫn,
> không tự test". Phần B (`ValidatingAdmissionPolicy`) đã viết file manifest
> theo đúng pattern đã kiểm chứng ở Lab 3.3/4.5 nhưng **chưa kịp chạy thử
> trên cụm thật** do mất kết nối — bạn nên tự `kubectl apply` và verify lại
> trước khi tin tưởng hoàn toàn, hoặc nhờ mình test lại khi có mạng.

## Mục tiêu

- Hiểu vì sao cần whitelist registry: bất kỳ ai có quyền tạo pod đều có thể
  kéo image từ registry công khai bất kỳ, kể cả registry chứa mã độc.
- Biết 2 cách chặn: `ImagePolicyWebhook` (admission controller built-in gọi
  ra webhook ngoài) và `ValidatingAdmissionPolicy` (CEL, built-in, không cần
  webhook ngoài).

## Phần A — ImagePolicyWebhook (theo đúng tài liệu, cần SSH vào control-plane)

**Cảnh báo rủi ro:** giống Lab 3.4, đây là thay đổi `kube-apiserver` static
pod — sai cú pháp có thể làm apiserver không start được. Luôn backup trước:
`cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak`.

### Bước 1 — Viết webhook server (ví dụ tối giản bằng Flask, theo đúng tài liệu)

```python
# webhook.py
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.route("/validate", methods=["POST"])
def validate():
    review = request.json
    image = review["spec"]["containers"][0]["image"]
    allowed = image.startswith("internal-registry.io/")
    return jsonify({
        "apiVersion": "imagepolicy.k8s.io/v1alpha1",
        "kind": "ImageReview",
        "status": {
            "allowed": allowed,
            "reason": "" if allowed else f"Image '{image}' is not from a trusted registry"
        }
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=443, ssl_context=("server.crt", "server.key"))
```

> Lưu ý: `ImagePolicyWebhook` dùng schema `ImageReview`
> (`imagepolicy.k8s.io/v1alpha1`) — **khác** với `AdmissionReview` dùng bởi
> `ValidatingWebhookConfiguration`/Gatekeeper. Đừng nhầm 2 API này khi viết
> webhook server, request/response body có cấu trúc khác nhau hoàn toàn.

Webhook cần chạy HTTPS (apiserver bắt buộc TLS khi gọi webhook), nên cần tự
ký cert hoặc dùng cert do cluster CA cấp (xem lại Lab 1.2 — flow CSR).

### Bước 2 — Deploy webhook (trong cluster, dùng Service để apiserver gọi tới được)

```bash
kubectl create namespace image-policy
# build + deploy webhook.py thành 1 Deployment + Service trong namespace này
```

### Bước 3 — Tạo AdmissionConfiguration + kubeconfig cho webhook

```yaml
# /etc/kubernetes/imagepolicy/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/imagepolicy/kubeconfig.yaml
        allowTTL: 50
        denyTTL: 50
        retryBackoff: 500
        defaultAllow: false
```

```yaml
# /etc/kubernetes/imagepolicy/kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
  - name: image-policy-webhook
    cluster:
      server: https://image-policy-webhook.image-policy.svc:443/validate
      certificate-authority: /etc/kubernetes/imagepolicy/ca.crt
contexts:
  - name: image-policy-webhook
    context:
      cluster: image-policy-webhook
current-context: image-policy-webhook
```

> `defaultAllow: false` nghĩa là nếu webhook không trả lời được (timeout,
> down), request bị **từ chối** — an toàn hơn nhưng rủi ro là webhook down
> sẽ chặn TOÀN BỘ việc tạo pod mới trên cluster (tương tự bài học từ sự cố
> Gatekeeper ở Lab 3.3). Cân nhắc kỹ trade-off này trước khi dùng
> `defaultAllow: false` trong production.

### Bước 4 — Sửa `kube-apiserver.yaml`

Thêm flag:

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
- --admission-control-config-file=/etc/kubernetes/imagepolicy/admission-config.yaml
```

Kèm `volumeMounts`/`volumes` trỏ tới `/etc/kubernetes/imagepolicy` (cùng
pattern đã dùng ở Lab 1.6/3.4).

### Bước 5 — Verify

```bash
kubectl run bad-pod --image=some-registry.io/a-very-vulnerable-image
# -> kỳ vọng bị từ chối vì webhook trả allowed=false

kubectl run good-pod --image=internal-registry.io/my-app:1.0
# -> kỳ vọng được tạo
```

## Phần B — ValidatingAdmissionPolicy (viết theo pattern đã kiểm chứng ở Lab 3.3, CHƯA test lại lần này)

```bash
kubectl create namespace restricted-registry
kubectl apply -f manifests/vap-whitelist-registry.yaml
```

Nội dung (`matchConstraints` nhắm vào Pod trong namespace `restricted-registry`,
CEL kiểm tra MỌI container trong pod đều có image bắt đầu bằng
`internal-registry.io/`):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: whitelist-registry
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    - expression: >-
        object.spec.containers.all(c, c.image.startsWith('internal-registry.io/'))
      message: "Image must be pulled from internal-registry.io"
```

Verify (chạy khi có mạng lại):

```bash
kubectl run bad-image -n restricted-registry --image=nginx
# -> kỳ vọng: denied, message "Image must be pulled from internal-registry.io"

kubectl run good-image -n restricted-registry --image=internal-registry.io/nginx:1.27
# -> kỳ vọng: created (dù image không thật sự tồn tại để pull — admission
#    chỉ kiểm tra TÊN image ở bước tạo object, không kiểm tra image có pull
#    thành công hay không, đó là việc của kubelet ở bước sau)
```

## Dọn dẹp

```bash
kubectl delete namespace restricted-registry image-policy --ignore-not-found
# Nếu làm Phần A, restore kube-apiserver.yaml gốc:
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

## Mẹo thi

- Phân biệt 3 cách tiếp cận đã học qua các lab để chặn image không mong
  muốn: **admission webhook tự viết** (Domain 1, Lab về Image Policy Webhook
  gốc), **ImagePolicyWebhook built-in** (Phần A ở đây, dùng schema
  `ImageReview` riêng), và **OPA/Gatekeeper hoặc ValidatingAdmissionPolicy**
  (Rego hoặc CEL, dùng schema `AdmissionReview` chuẩn) — đề thi có thể hỏi
  về bất kỳ cách nào trong 3, đừng nhầm lẫn API schema giữa chúng.
- `ImagePolicyWebhook` là admission controller **có sẵn trong kube-apiserver**
  (không cần cài thêm binary nào), chỉ cần bật qua
  `--enable-admission-plugins` + cấu hình — đây là điểm khác biệt với
  Gatekeeper (cần cài thêm cả 1 hệ thống riêng).
- Luôn nhớ `defaultAllow`/`defaultDeny` (tuỳ tài liệu) khi cấu hình bất kỳ
  admission webhook nào — đây là quyết định cân bằng giữa "an toàn nhưng dễ
  brick cluster nếu webhook down" và "tiện nhưng có thể fail-open khi cần
  chặn nhất".
