# Domain 4: Supply Chain Security

Nguồn: module `Supply-Chain-Security` trong notes CKS (Trivy, KubeLinter,
Minimal Base Images, SBOM, Image Policy Webhook).

| Lab | Chủ đề | Test trạng thái |
|---|---|---|
| [4.1](lab-4.1-trivy-scanning/README.md) | Trivy image scanning | ✅ Đã test trên máy làm việc |
| [4.2](lab-4.2-kubelinter/README.md) | KubeLinter | ✅ Đã test trên máy làm việc |
| [4.3](lab-4.3-minimal-base-images/README.md) | Minimal Base Images | ✅ Đã build & test trên máy làm việc |
| [4.4](lab-4.4-sbom/README.md) | SBOM (Syft + Grype) | ✅ Đã test trên máy làm việc |
| [4.5](lab-4.5-image-policy-webhook/README.md) | Whitelist Registry (ImagePolicyWebhook) | 📖 Phần A: hướng dẫn (kube-apiserver edit). Phần B (ValidatingAdmissionPolicy): viết theo pattern đã kiểm chứng ở Lab 3.3, **chưa kịp test lại** do mất kết nối mạng tới cụm giữa buổi làm việc |

Điểm khác biệt của domain này so với Domain 1-3: phần lớn lab (4.1-4.4)
chạy hoàn toàn trên máy làm việc (Trivy/KubeLinter/Syft/Grype/Docker cục bộ),
không cần chạm tới cụm Kubernetes — nên rủi ro thấp và test được đầy đủ dễ
dàng hơn hẳn.

## Lưu ý về kết nối mạng

Giữa lúc làm Lab 4.5, kết nối từ máy làm việc tới cụm (172.16.5.x) bị mất
(VPN/tunnel phía máy làm việc, không phải sự cố trên cụm) khiến phần
ValidatingAdmissionPolicy chưa kịp verify trực tiếp như các lab khác. Bạn
nên tự chạy lại phần verify trong Lab 4.5 khi có kết nối, hoặc nhờ chạy lại.
