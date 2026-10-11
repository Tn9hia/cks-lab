# Lab 4.1 — Scan Images với Trivy

**Domain:** Supply Chain Security
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** Trivy CLI. Cài trên máy làm việc bằng `brew install trivy`
(macOS/Linux) — không cần chạy trong cluster hay trên node, Trivy chỉ đọc
image từ registry hoặc file tar cục bộ.

## Mục tiêu

- Hiểu CVE là gì, thang điểm severity (CVSS 0-10).
- Scan image, lọc theo severity, so sánh base image nặng vs nhẹ.
- Áp dụng nguyên tắc: base image càng ít package, attack surface càng nhỏ.

## Bước 1 — Scan 1 image cũ, xem toàn bộ report

```bash
trivy image nginx:1.18.0
```

Report liệt kê từng CVE theo package (`apt`, `bash`, `curl`, `zlib1g`...),
kèm severity, version đang cài, version đã fix.

## Bước 2 — Lọc theo severity (dùng trong CI/CD để set ngưỡng pass/fail)

```bash
trivy image --severity CRITICAL,HIGH nginx:1.18.0
trivy image --severity CRITICAL nginx:1.18.0
trivy image --ignore-unfixed nginx:1.18.0   # chỉ hiện CVE ĐÃ có bản vá
```

`--ignore-unfixed` rất hữu ích trong pipeline thực tế: CVE chưa có fix thì
dù biết cũng không xử lý được ngay, báo cáo chỉ nên tập trung vào cái có thể
hành động được.

## Bước 3 — So sánh base image nặng vs nhẹ (đã test thực tế trên máy)

```bash
trivy image --severity CRITICAL,HIGH --scanners vuln nginx:1.18.0 | grep "Total:"
trivy image --severity CRITICAL,HIGH --scanners vuln nginx:1.27-alpine | grep "Total:"
```

Kết quả thực tế (có thể thay đổi theo thời gian vì CVE database cập nhật
liên tục — chạy lại để lấy số mới nhất):

| Image | Total CRITICAL+HIGH |
|---|---|
| `nginx:1.18.0` (Debian, cũ) | 189 (HIGH: 147, CRITICAL: 42) |
| `nginx:1.27-alpine` (Alpine, mới) | 36 (HIGH: 34, CRITICAL: 2) |

Chênh lệch đến từ 2 yếu tố cộng dồn: (1) Alpine base nhỏ hơn nhiều Debian
→ ít package hệ thống hơn, (2) `1.27` mới hơn `1.18.0` nhiều năm → ít CVE
tồn đọng chưa vá hơn. Đừng kết luận vội "Alpine luôn an toàn hơn Debian" chỉ
từ 1 phép so sánh — luôn kiểm tra cả biến số version khi so sánh.

## Bước 4 — Scan image dạng tar (khi không có sẵn registry, hay dùng trong air-gapped CI)

```bash
docker save nginx:1.27-alpine -o /tmp/nginx.tar 2>/dev/null || echo "cần Docker/OrbStack chạy để thử bước này"
trivy image --input /tmp/nginx.tar
```

## Bước 5 — Output dạng JSON để tích hợp CI/CD

```bash
trivy image --format json --severity CRITICAL -o /tmp/scan-result.json nginx:1.27-alpine
cat /tmp/scan-result.json | python3 -m json.tool | grep -E "VulnerabilityID|Severity" | head -20
```

## Dọn dẹp

```bash
rm -f /tmp/nginx.tar /tmp/scan-result.json
```

## Mẹo thi

- CKS không hỏi bạn cài Trivy (thường có sẵn trong môi trường thi nếu câu
  hỏi cần) — trọng tâm là đọc hiểu report và biết dùng đúng flag
  (`--severity`, `--ignore-unfixed`) để trả lời đúng câu hỏi kiểu "tìm image
  nào có CVE CRITICAL chưa fix".
- Nhớ khái niệm cốt lõi: **giảm số package trong image = giảm attack
  surface** — đây là lý do Lab 4.3 (minimal base image) là bước tiếp theo
  logic sau khi biết cách scan.
- Trivy còn scan được nhiều loại target khác ngoài container image:
  filesystem (`trivy fs .`), Git repo (`trivy repo <url>`), Kubernetes
  cluster trực tiếp (`trivy k8s cluster`) — CKS chủ yếu hỏi về image
  scanning nhưng biết thêm phạm vi công cụ không thừa.
