# CKS Practice Labs

Bộ lab thực hành tự thiết kế, bám theo giáo trình CKS của KodeKloud, để luyện tập
trên cụm Kubernetes tự dựng (không phải sandbox của KodeKloud).

## Cụm thực hành hiện tại

| Node | Role | OS | Runtime | K8s |
|---|---|---|---|---|
| k8s-controlplane-01 | control-plane | Ubuntu 24.04 | containerd 2.3.3 | v1.36.0 |
| k8s-worker-01/02/03 | worker | Ubuntu 24.04 | containerd 2.3.3 | v1.36.0 |

CNI: **Cilium** (kèm Hubble Relay/UI) — hỗ trợ cả `NetworkPolicy` chuẩn và
`CiliumNetworkPolicy` nâng cao (L3-L7).

`kubectl` trên máy làm việc trỏ sẵn tới cụm này, và có SSH + sudo tới từng
node (`ssh ubuntu@172.16.5.12/13/14/15 -i ~/.ssh/id_public`, `.12` là
control-plane, `.13-.15` là worker). Một số lab (kernel module, UFW,
containerd/gVisor) vẫn được viết ở dạng **hướng dẫn để bạn tự SSH thực
hiện** thay vì tự động test, vì blast radius quá lớn nếu sai sót (restart
containerd, đổi firewall) — xem ghi chú rủi ro trong từng lab đó.

## Cấu trúc

Mỗi domain thi CKS là một thư mục trong `labs/`, mỗi lab là một thư mục con có:
- `README.md`: mục tiêu, bước làm, cách verify, dọn dẹp, mẹo thi
- `manifests/`: YAML áp dụng trực tiếp bằng `kubectl apply -f`

## Domain map (trọng số đề thi hiện tại)

| # | Domain | Trọng số | Thư mục | Trạng thái |
|---|---|---|---|---|
| 1 | Cluster Setup & Hardening | 10% + 15% | `labs/01-cluster-setup-hardening` | ✅ 6/6 lab, phần lớn đã test trên cụm thật |
| 2 | System Hardening | 15% | `labs/02-system-hardening` | ✅ 5/5 lab, 3 đã test trên cụm, 2 dạng hướng dẫn |
| 3 | Minimize Microservice Vulnerabilities | 20% | `labs/03-minimize-microservice-vulnerabilities` | ✅ 6/6 lab, 4 đã test trên cụm |
| 4 | Supply Chain Security | 20% | `labs/04-supply-chain-security` | ✅ 5/5 lab, 4 đã test trên máy làm việc |
| 5 | Monitoring, Logging & Runtime Security | 20% | `labs/05-monitoring-logging-runtime-security` | ✅ 4/4 lab, test trên cụm thật 2026-10-11 — phát hiện & sửa 4 lỗi thực tế (1 làm Falco crash hoàn toàn) |

Nguồn ánh xạ nội dung: notes KodeKloud CKS (mục lục đầy đủ trong MCP
`notes-kodekloud`, khoá "Certified-Kubernetes-Security-Specialist-CKS").

## Sự cố đáng chú ý trong lúc xây dựng bộ lab (để tham khảo)

Toàn bộ đã được xử lý, cụm hiện khoẻ mạnh — liệt kê lại để bạn hiểu rõ những
gì đã xảy ra trên chính cụm của mình:

1. **Lab 2.1**: cấp `CAP_SYS_TIME` cho container rồi chạy `date -s` đã đổi
   đồng hồ hệ thống của `k8s-worker-01` (ảnh hưởng cả node, không chỉ
   container) — đã tự phát hiện và sửa lại giờ ngay.
2. **Lab 2.3**: pod dùng AppArmor profile thiếu rule `signal` bị kẹt
   `Terminating` không xoá được — đã gỡ profile khỏi kernel host để giải
   phóng.
3. **Lab 3.3**: cài thử OPA Gatekeeper v3.14.0 phát hiện lỗi tương thích với
   Kubernetes v1.36 — 1 webhook từng chặn đứng việc tạo namespace toàn cụm.
   Đã patch cứu cụm rồi gỡ hẳn Gatekeeper, thay bằng
   `ValidatingAdmissionPolicy` (built-in, không rủi ro tương tự).

Xem chi tiết từng sự cố trong README của lab tương ứng.
