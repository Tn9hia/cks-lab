# Lab 2.5 — Minimize External Access to the Network (UFW)

**Domain:** System Hardening
**Thời gian gợi ý:** 30 phút
**Yêu cầu:** SSH + sudo vào node. **Mình chỉ viết hướng dẫn, không tự chạy
trên cụm của bạn** — đây là thay đổi firewall trên node đang chạy
control-plane/etcd/Cilium thật; cấu hình sai có thể làm **mất kết nối SSH
hoặc sập giao tiếp giữa các node trong cluster** mà không có cách tự sửa từ
xa. Rủi ro này vượt quá mức mình tự ý thực hiện thay bạn.

## ĐỌC TRƯỚC KHI LÀM — quy tắc an toàn bắt buộc

1. **Luôn mở sẵn 1 session SSH thứ hai** tới node trước khi `ufw enable`. Nếu
   session đầu bị ngắt do rule sai, dùng session thứ hai để sửa/tắt UFW
   ngay (`sudo ufw disable`).
2. **Không bao giờ set `ufw default deny incoming` rồi `ufw enable` trong
   cùng 1 lệnh** mà chưa có rule allow SSH trước đó.
3. Làm trên **1 node duy nhất trước** (khuyến nghị: 1 worker ít quan trọng
   nhất), verify cluster vẫn khoẻ, rồi mới cân nhắc áp dụng node khác.
4. Nếu bất kỳ bước nào làm `kubectl get nodes` báo node `NotReady` hoặc mất
   SSH, **dừng lại ngay và revert** (`ufw disable`) trước khi làm tiếp.

## Mục tiêu

- Liệt kê chính xác port nào đang mở trên 1 node Kubernetes và vì sao.
- Cấu hình UFW theo nguyên tắc: default-deny inbound, chỉ allow đúng port
  cluster cần, từ đúng dải IP cần.

## Bước 1 — Khảo sát port đang mở (làm trên từng node)

```bash
ssh <user>@172.16.5.13   # k8s-worker-01, đổi IP tương ứng node bạn muốn làm
sudo ss -tulnp | grep LISTEN
```

Đối chiếu với bảng port chuẩn của Kubernetes (điều chỉnh theo CNI đang dùng —
cụm bạn dùng **Cilium**, cột cuối ghi rõ):

| Port | Thành phần | Node nào | Bind |
|---|---|---|---|
| 22 | SSH | mọi node | 0.0.0.0 |
| 6443 | kube-apiserver | control-plane | 0.0.0.0 |
| 2379-2380 | etcd | control-plane | thường bind IP nội bộ |
| 10250 | kubelet API | mọi node | 0.0.0.0 |
| 10256 | kube-proxy healthz (nếu còn dùng kube-proxy) | mọi node | 0.0.0.0 |
| 10257 | kube-controller-manager | control-plane | thường 127.0.0.1 |
| 10259 | kube-scheduler | control-plane | thường 127.0.0.1 |
| 4240 | Cilium health check | mọi node | 0.0.0.0 |
| 4244 | Hubble (nếu bật) | mọi node | 0.0.0.0 |
| 4245 | Hubble Relay | node chạy hubble-relay | 0.0.0.0 |
| 8472 hoặc 6081 | VXLAN/Geneve overlay (Cilium tunnel mode) | mọi node | UDP |
| 9962-9965 | Cilium/Hubble metrics (Prometheus scrape) | mọi node | 0.0.0.0 |

> Danh sách port Cilium có thể khác tuỳ chế độ (tunnel vs native routing,
> kube-proxy replacement bật/tắt) — luôn đối chiếu bằng `ss -tulnp` thực tế
> thay vì chỉ tin bảng trên, và kiểm tra tài liệu Cilium ứng với version
> đang chạy (`cilium-dbg version`).

## Bước 2 — Cài UFW

```bash
sudo apt-get update
sudo apt-get install -y ufw
sudo ufw status   # -> inactive, an toàn để cấu hình tiếp
```

## Bước 3 — Set default policy (CHƯA enable vội)

```bash
sudo ufw default allow outgoing
sudo ufw default deny incoming
```

## Bước 4 — Allow SSH TRƯỚC TIÊN (bước quan trọng nhất, đừng bỏ qua)

Nếu SSH của bạn tới từ 1 dải IP admin cố định, giới hạn theo IP đó thay vì
allow toàn bộ:

```bash
# Ví dụ nếu bạn luôn SSH từ 1 máy/jump host có IP cố định 172.16.5.1
sudo ufw allow from 172.16.5.1 to any port 22 proto tcp

# Nếu không chắc chắn IP nguồn (vd làm việc từ nhiều nơi), tạm thời allow rộng
# rồi siết lại sau khi đã xác nhận ổn định:
sudo ufw allow 22/tcp
```

## Bước 5 — Allow các port cluster cần, giới hạn theo dải mạng nội bộ

Giả sử dải mạng nội bộ cluster là `172.16.5.0/24` (đúng dải IP node của bạn):

```bash
sudo ufw allow from 172.16.5.0/24 to any port 6443 proto tcp   # apiserver (chỉ cần trên control-plane)
sudo ufw allow from 172.16.5.0/24 to any port 10250 proto tcp  # kubelet
sudo ufw allow from 172.16.5.0/24 to any port 2379:2380 proto tcp  # etcd (chỉ control-plane)
sudo ufw allow from 172.16.5.0/24 to any port 4240 proto tcp   # cilium health
sudo ufw allow from 172.16.5.0/24 to any port 8472 proto udp   # vxlan overlay (nếu dùng tunnel mode)
```

Không allow các port này từ `0.0.0.0/0` — chỉ cluster nội bộ mới cần gọi tới
chúng, không có lý do gì để mở ra internet/mạng ngoài.

## Bước 6 — Enable (đã mở sẵn session SSH dự phòng theo quy tắc an toàn)

```bash
sudo ufw enable
# "Command may disrupt existing ssh connections. Proceed with operation (y|n)?" -> y
sudo ufw status verbose
```

## Bước 7 — Verify cluster vẫn khoẻ (làm từ máy làm việc)

```bash
kubectl get nodes
kubectl get pods -A | grep -v Running
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --brief
```

Nếu node chuyển `NotReady` hoặc pod Cilium bắt đầu `CrashLoopBackOff`/mất
kết nối chéo node, SSH vào node đó (dùng session dự phòng) và:

```bash
sudo ufw disable
```

rồi tìm port còn thiếu bằng `sudo ufw status` đối chiếu với `ss -tulnp` một
lần nữa.

## Dọn dẹp / Revert hoàn toàn

```bash
sudo ufw disable
sudo apt-get remove -y ufw   # nếu muốn gỡ hẳn
```

## Mẹo thi

- Đề thi CKS thường cho sẵn ngữ cảnh giống hệt ví dụ gốc: "chỉ 1 jump server
  IP X được SSH vào, port Y cần mở cho dải mạng Z" — việc của bạn là dịch
  đúng sang `ufw allow from <IP> to any port <port> proto <tcp/udp>`, không
  cần nhớ toàn bộ port Kubernetes.
- Cú pháp hay nhầm: `ufw allow 8080` (mở cho mọi nguồn) khác hẳn `ufw allow
  from <IP> to any port 8080` (chỉ mở cho 1 nguồn) — đọc kỹ đề để chọn đúng
  dạng.
- Luôn set `ufw default deny incoming` + `ufw default allow outgoing` làm
  nền trước, sau đó mới thêm rule allow cụ thể — đây là thứ tự chuẩn, làm
  ngược lại (allow rồi mới default deny) có thể vô tình chặn nhầm rule đã
  set trước đó tùy thứ tự UFW áp dụng.
- Trên cụm dùng Cilium (như của bạn), UFW hoạt động song song với iptables/
  eBPF rules mà Cilium tự quản lý — về mặt lý thuyết chúng độc lập, nhưng
  luôn test kỹ overlay network sau khi bật UFW vì 2 lớp filtering chồng lên
  nhau dễ gây debug khó hiểu nếu có xung đột.
