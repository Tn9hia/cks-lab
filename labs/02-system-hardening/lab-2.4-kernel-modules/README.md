# Lab 2.4 — Restrict Kernel Modules

**Domain:** System Hardening
**Thời gian gợi ý:** 15 phút
**Yêu cầu:** SSH + sudo vào 1 worker node. **Lab này mình chỉ viết hướng dẫn,
không tự chạy trên cụm của bạn** — blacklist kernel module cần reboot để có
hiệu lực đầy đủ, rủi ro cao nếu tự động hoá sai trên node đang chạy workload
thật (khác với các lab trước, đây là thay đổi cấp kernel-wide, không cô lập
theo container).

## Mục tiêu

- Hiểu vì sao cần blacklist kernel module ngay cả khi không ai "chủ động"
  load nó — chỉ cần một process trong pod tạo network socket loại lạ (vd
  SCTP) là kernel tự động load module tương ứng.
- Biết cách blacklist qua `/etc/modprobe.d/`.

## Bước 1 — Kiểm tra module đang load

```bash
ssh <user>@172.16.5.13   # ví dụ k8s-worker-01
lsmod | grep -E "sctp|dccp"
```

Nếu trống, nghĩa là chưa có module nào trong 2 module này được load — đây là
baseline tốt để test.

## Bước 2 — Thử trigger load module SCTP (để thấy rủi ro thực tế)

```bash
sudo modprobe sctp
lsmod | grep sctp
# -> thấy module đã load
```

Đây chính là kịch bản đáng lo: một ứng dụng trong container (không cần
privileged) mở 1 socket kiểu `SOCK_SEQPACKET`/`IPPROTO_SCTP` cũng có thể
khiến kernel tự load module này trên toàn bộ node — ảnh hưởng đến mọi pod
khác trên cùng node đó, không chỉ pod gây ra.

## Bước 3 — Gỡ module và blacklist

```bash
sudo modprobe -r sctp
sudo tee /etc/modprobe.d/blacklist-cks.conf <<'EOF'
blacklist sctp
blacklist dccp
install sctp /bin/false
install dccp /bin/false
EOF
```

> Dòng `install sctp /bin/false` quan trọng hơn `blacklist sctp`: `blacklist`
> chỉ chặn **auto-load qua alias** (vd khi hệ thống tự dò module theo tên
> thiết bị), nhưng **không chặn** việc module bị load trực tiếp qua tên (vd
> kernel tự `modprobe sctp` khi có socket SCTP được tạo) — đây là lỗ hổng
> hay bị hiểu nhầm. `install <module> /bin/false` mới thực sự chặn triệt để
> vì nó thay thế hành động "load module" bằng việc chạy `/bin/false` (luôn
> exit fail) thay vì gọi `insmod` thật.

## Bước 4 — Verify (cần reboot để chắc chắn)

```bash
sudo modprobe sctp
echo $?
# -> khác 0 (thất bại) nếu blacklist đúng, ngay cả không cần reboot vì
#    modprobe đọc /etc/modprobe.d/ mỗi lần chạy

lsmod | grep sctp
# -> trống
```

Để chắc chắn 100% (một số cấu hình chỉ áp dụng đầy đủ sau khi initramfs
được rebuild), làm thêm:

```bash
sudo update-initramfs -u   # trên Ubuntu/Debian
sudo reboot
```

Sau khi node reboot xong, verify lại từ máy làm việc:

```bash
kubectl get nodes   # đảm bảo node Ready trở lại trước khi làm gì tiếp
```

rồi SSH lại kiểm tra `lsmod | grep sctp` một lần nữa.

## Dọn dẹp (nếu muốn revert)

```bash
sudo rm /etc/modprobe.d/blacklist-cks.conf
sudo update-initramfs -u
```

## Mẹo thi

- CKS hay hỏi đúng 2 module ví dụ: `sctp` và `dccp` (hai giao thức transport
  hiếm khi dùng trong workload Kubernetes thông thường, dễ bị lợi dụng để
  bypass NetworkPolicy vì nhiều tool chỉ filter TCP/UDP).
- Luôn nhớ **cả 2 dòng**: `blacklist <module>` (chặn auto-detect theo alias)
  **và** `install <module> /bin/false` (chặn load trực tiếp theo tên) — chỉ
  viết `blacklist` là câu trả lời không đầy đủ trong bài thi.
- Đây là thay đổi ở **node**, không phải object Kubernetes nào — nếu đề bài
  ghi rõ "trên node X", đừng tìm cách làm qua kubectl.
