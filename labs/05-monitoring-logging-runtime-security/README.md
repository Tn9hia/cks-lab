# Domain 5: Monitoring, Logging & Runtime Security

Nguồn: module `Monitoring-Logging-and-Runtime-Security` trong notes CKS
(Falco, Immutable Containers, Behavioral Analytics).

| Lab | Chủ đề | Test trạng thái |
|---|---|---|
| [5.1](lab-5.1-falco-install/README.md) | Cài Falco (Helm DaemonSet) | ⚠️ Chưa test — mất kết nối mạng tới cụm giữa buổi |
| [5.2](lab-5.2-falco-custom-rules/README.md) | Falco custom rules | ⚠️ Chưa test |
| [5.3](lab-5.3-immutable-containers/README.md) | Immutable Containers + Falco | ⚠️ Phần readOnlyRootFilesystem đã test ở Lab 3.1; phần kết hợp privileged+Falco chưa test |
| [5.4](lab-5.4-privilege-escalation-detection/README.md) | Phát hiện Privilege Escalation (bài tự luyện, không dịch trực tiếp từ notes) | ⚠️ Chưa test |

## ⚠️ Toàn bộ Domain 5 chưa được test trực tiếp trên cụm

Khác với Domain 1-4 (mọi lab đều được tự chạy và verify trên cụm/máy làm
việc trước khi giao), **toàn bộ Domain 5 được viết trong lúc mất kết nối
mạng từ máy làm việc tới cụm** (VPN/tunnel rớt giữa buổi làm việc, xác nhận
không phải do bất kỳ thao tác nào trong các lab trước gây ra — `ping` và
`kubectl` đều timeout, không liên quan tới sự cố Gatekeeper đã xử lý ở
Domain 3).

Nội dung dựa trên:
- Notes KodeKloud CKS (Falco install, config, custom rules — đã đọc kỹ và
  bám sát).
- Falco Helm chart chính thức (cách dùng phổ biến, ổn định).
- Lab 5.4 là bài tự thiết kế thêm (ghi rõ trong file), không có "đáp án gốc"
  để đối chiếu.

**Khuyến nghị:** khi có kết nối lại, tự chạy qua từng lab theo đúng thứ tự
5.1 → 5.2 → 5.3 → 5.4, đối chiếu kết quả thực tế với phần "kỳ vọng" trong
mỗi README. Nếu có sai lệch (đặc biệt cú pháp Falco rule ở 5.2/5.4 — Falco
có thể khác version với tài liệu gốc), báo lại để cập nhật chính xác.
