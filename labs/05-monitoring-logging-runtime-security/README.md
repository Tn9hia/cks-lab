# Domain 5: Monitoring, Logging & Runtime Security

Nguồn: module `Monitoring-Logging-and-Runtime-Security` trong notes CKS
(Falco, Immutable Containers, Behavioral Analytics).

| Lab | Chủ đề | Test trạng thái |
|---|---|---|
| [5.1](lab-5.1-falco-install/README.md) | Cài Falco (Helm DaemonSet) | ✅ Test trên cụm thật 2026-10-11 |
| [5.2](lab-5.2-falco-custom-rules/README.md) | Falco custom rules | ✅ Test trên cụm thật — phát hiện & sửa 3 lỗi/caveat quan trọng (xem README lab) |
| [5.3](lab-5.3-immutable-containers/README.md) | Immutable Containers + Falco | ✅ Test trên cụm thật, khớp kỳ vọng 100% |
| [5.4](lab-5.4-privilege-escalation-detection/README.md) | Phát hiện Privilege Escalation (bài tự luyện, không dịch trực tiếp từ notes) | ✅ Test trên cụm thật — phát hiện & sửa 1 lỗi làm Falco crash hoàn toàn |

## ✅ Toàn bộ Domain 5 đã được test trên cụm thật (2026-10-11)

Cụm dùng để test: 2 node arm64 dựng bằng multipass (`k8s-master` +
`k8s-worker-1`, Kubernetes v1.36.5, containerd, Cilium CNI), Falco
`0.45.0` cài qua `falcosecurity/falco` Helm chart, driver `modern_ebpf`.

Quá trình test phát hiện **nhiều lỗi/caveat thực tế** không thể lường được
chỉ bằng đọc tài liệu — lý do chính xác là mục tiêu của bước "tự verify
trên cụm thật" mà README gốc đã yêu cầu. Tóm tắt (chi tiết đầy đủ nằm
trong README từng lab, mục "Phát hiện quan trọng từ test thực tế"):

- **Lab 5.2** (3 phát hiện):
  1. Rule thiếu giới hạn event-type (`spawned_process`) không chỉ gây
     noise, mà còn **làm hỏng rule khác** chạy cùng lúc — vì Falco bản này
     dùng `rule_matching: first` (chỉ rule đầu tiên khớp mỗi event được
     alert, không phải "mọi rule khớp đều alert" như giả định phổ biến).
  2. `container.image.repository` trả về có prefix registry
     (`docker.io/...`) — so sánh bằng `=` exact-match không bao giờ khớp,
     phải dùng `contains`.
  3. `rule_matching: first` cũng khiến custom rule cho `/etc/shadow`
     không bao giờ chạy được (built-in rule có sẵn luôn thắng) — phải
     demo bằng `/etc/passwd`.
- **Lab 5.4** (1 lỗi nghiêm trọng): macro `container_started` không tồn
  tại trong Falco 0.45.0 → không phải rule đó bị bỏ qua, mà **toàn bộ
  Falco pod crash** (`CrashLoopBackOff`) vì rules file bị coi là Invalid.
- **Cả 2 lab 5.2 và 5.4**: image test `kodekloud/simple-webapp` chỉ build
  cho amd64 — `exec format error` trên cụm arm64 (ghi caveat + workaround
  trong README lab 5.2).

**Bài học tổng quát:** Falco rule tưởng chừng đúng cú pháp (load được,
không báo lỗi parse) vẫn có thể **không bao giờ fire** do (a) field trả về
khác giá trị mong đợi, (b) rule khác "cướp" event trước do thiếu
event-type restriction, hoặc (c) built-in rule trùng phạm vi luôn thắng.
Luôn tự trigger và đọc log thật — đừng tin chỉ vì rule load thành công
không báo lỗi.

Nguồn gốc nội dung:
- Notes KodeKloud CKS (Falco install, config, custom rules — đã đọc kỹ và
  bám sát).
- Falco Helm chart chính thức (cách dùng phổ biến, ổn định).
- Lab 5.4 là bài tự thiết kế thêm (ghi rõ trong file), không có "đáp án gốc"
  để đối chiếu.
