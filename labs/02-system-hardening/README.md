# Domain 2: System Hardening

Nguồn: module `System-Hardening` trong notes CKS (Linux Capabilities,
Seccomp, AppArmor, Kernel Modules, UFW/Firewall, SSH Hardening).

| Lab | Chủ đề | Test trên cụm thật? |
|---|---|---|
| [2.1](lab-2.1-linux-capabilities/README.md) | Linux Capabilities | ✅ Đã test |
| [2.2](lab-2.2-seccomp/README.md) | Seccomp (RuntimeDefault + custom Localhost) | ✅ Đã test |
| [2.3](lab-2.3-apparmor/README.md) | AppArmor | ✅ Đã test |
| [2.4](lab-2.4-kernel-modules/README.md) | Restrict Kernel Modules | ⚠️ Chỉ hướng dẫn (cần bạn tự SSH) |
| [2.5](lab-2.5-minimize-external-access/README.md) | UFW Firewall | ⚠️ Chỉ hướng dẫn (cần bạn tự SSH) |

Lab 2.1-2.3 dùng kỹ thuật "Job/Pod privileged + hostPath + nsenter" để thao
tác trên node **mà không cần SSH thủ công** — cách này chỉ an toàn với các
thay đổi có thể revert dễ dàng trong container/kernel namespace tạm thời.
Lab 2.4-2.5 đụng tới cấu hình kernel module và firewall ở mức ảnh hưởng lâu
dài/khó revert từ xa, nên được viết dưới dạng hướng dẫn để bạn tự thực hiện
có kiểm soát (mở sẵn session SSH dự phòng trước khi đổi firewall).

## Sự cố đã gặp khi viết & test lab (để bạn tham khảo, không cần lo lắng)

1. **Lab 2.1**: cấp `CAP_SYS_TIME` cho 1 container rồi chạy `date -s` đã đổi
   luôn đồng hồ hệ thống của `k8s-worker-01` (không chỉ container) — vì
   `CLOCK_REALTIME` không bị cô lập theo namespace trong Linux. Đã tự phát
   hiện và sửa lại giờ ngay. Bài học này đã được đưa thành cảnh báo chính
   trong lab.
2. **Lab 2.3**: 1 pod dùng AppArmor profile chỉ có `deny /** w,` (không có
   rule `signal`) đã bị **kẹt ở trạng thái `Terminating` không xoá được**
   vì kernel chặn cả việc `runc` gửi SIGKILL vào container. Phải gỡ profile
   khỏi kernel host để giải phóng rồi xoá pod mới thành công. Đã ghi thành
   gotcha trong lab.

Cả 2 sự cố đều đã được khắc phục hoàn toàn, cụm hiện khoẻ mạnh (4/4 node
Ready).
