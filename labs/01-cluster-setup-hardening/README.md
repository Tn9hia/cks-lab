# Domain 1: Cluster Setup & Cluster Hardening

Nguồn: module `Cluster-Setup-and-Hardening` trong notes CKS (RBAC, TLS/Certificates,
KubeConfig, Network Policy, kube-bench/CIS, Node Metadata, Auditing).

| Lab | Chủ đề | Ánh xạ đề thi |
|---|---|---|
| [1.1](lab-1.1-rbac/README.md) | RBAC từ số 0 | Cluster Hardening: RBAC |
| [1.2](lab-1.2-certificates-api/README.md) | Certificates API (CSR) | Cluster Hardening: TLS/Certificates |
| [1.3](lab-1.3-kube-bench/README.md) | kube-bench / CIS Benchmark | Cluster Setup: CIS benchmark |
| [1.4](lab-1.4-network-policy/README.md) | NetworkPolicy với Cilium | Cluster Hardening: restrict network access |
| [1.5](lab-1.5-node-metadata/README.md) | Chặn truy cập Node Metadata | Cluster Hardening: securing metadata |
| [1.6](lab-1.6-audit-logging/README.md) | Audit Logging | Cluster Hardening: audit logs |

Thứ tự làm gợi ý: 1.1 → 1.2 → 1.4 → 1.6 → 1.3 → 1.5 (1.3 và 1.5 cần thao tác
trên node nên để sau khi đã quen thao tác API-level).
