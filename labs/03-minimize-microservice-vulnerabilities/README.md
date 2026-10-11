# Domain 3: Minimize Microservice Vulnerabilities

Nguồn: module `Minimize-Microservice-Vulnerabilities` trong notes CKS
(Security Context, Pod Security Admission, OPA/Gatekeeper, Secrets, Resource
Quotas, Container Sandboxing).

| Lab | Chủ đề | Test trên cụm thật? |
|---|---|---|
| [3.1](lab-3.1-security-context/README.md) | Security Context | ✅ Đã test |
| [3.2](lab-3.2-pod-security-admission/README.md) | Pod Security Admission | ✅ Đã test |
| [3.3](lab-3.3-opa-gatekeeper/README.md) | OPA Gatekeeper + ValidatingAdmissionPolicy | ⚠️ Gatekeeper: lỗi tương thích với K8s 1.36, đã gỡ. ✅ ValidatingAdmissionPolicy: đã test |
| [3.4](lab-3.4-secrets-encryption/README.md) | Encrypting Secrets at Rest | 📖 Chỉ hướng dẫn (cần bạn tự SSH) |
| [3.5](lab-3.5-resource-quotas/README.md) | Resource Quotas & LimitRange | ✅ Đã test |
| [3.6](lab-3.6-gvisor-sandboxing/README.md) | gVisor Sandboxing | 📖 Chỉ hướng dẫn (cần bạn tự SSH, rủi ro cao nhất — restart containerd) |

## Sự cố đáng chú ý khi viết & test lab (Lab 3.3)

Cài thử **OPA Gatekeeper v3.14.0** (đúng bản notes khuyến nghị) lên cụm thật
để test — phát hiện 2 vấn đề:

1. Ví dụ ConstraintTemplate trong tài liệu gốc có **tên sai quy cách**
   (`systemrequiredlabels` số nhiều thay vì `systemrequiredlabel` số ít theo
   `kind`) — Gatekeeper từ chối tạo với lỗi rõ ràng, đã sửa và ghi chú lại.
2. Nghiêm trọng hơn: **webhook phụ của Gatekeeper (`failurePolicy: Fail`)
   từng chặn đứng việc tạo namespace mới trên toàn cụm** trong lúc test, và
   webhook chính không thực sự chặn được Pod vi phạm dù cấu hình đúng —
   nghi vấn version skew giữa Gatekeeper v3.14.0 và Kubernetes v1.36 (rất
   mới). Đã xử lý: patch tạm `failurePolicy` để cứu cụm, sau đó **gỡ hoàn
   toàn Gatekeeper**, xác nhận cụm khoẻ mạnh trở lại.

Để bạn vẫn có phần thực hành đáng tin cậy cho chủ đề "policy enforcement qua
admission controller", lab 3.3 bổ sung cách làm tương đương bằng
`ValidatingAdmissionPolicy` (built-in Kubernetes 1.30+, không có webhook nên
không có rủi ro tương tự) — đã test thành công 100%.

Cụm hiện khoẻ mạnh, không còn tài nguyên test nào sót lại (đã xác nhận
`kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations`
trả về rỗng).
