# Lab 4.3 — Minimize Base Image Footprint

**Domain:** Supply Chain Security
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** Docker/OrbStack chạy trên máy làm việc (build image cục bộ,
không cần cluster).

## Mục tiêu

- Build cùng 1 ứng dụng trên 2 base image khác nhau (Debian vs Alpine).
- Đo lường trực tiếp: kích thước image, số CVE — thấy rõ hệ quả thực tế của
  việc chọn base image.

## Bước 1 — Build cùng ứng dụng trên 2 base khác nhau

```bash
mkdir -p /tmp/webapp-lab && cd /tmp/webapp-lab
echo "<html><body><h1>CKS Lab 4.3</h1></body></html>" > index.html

cat > Dockerfile.debian <<'EOF'
FROM httpd
COPY index.html /usr/local/apache2/htdocs/index.html
EOF

cat > Dockerfile.alpine <<'EOF'
FROM httpd:alpine
COPY index.html /usr/local/apache2/htdocs/index.html
EOF

docker build -t custom-webapp:debian -f Dockerfile.debian .
docker build -t custom-webapp:alpine -f Dockerfile.alpine .
```

## Bước 2 — So sánh kích thước (đã test thực tế)

```bash
docker images custom-webapp
```

Kết quả thực tế:

| Image | Disk usage |
|---|---|
| `custom-webapp:debian` | 147 MB |
| `custom-webapp:alpine` | 75.9 MB |

Cùng 1 ứng dụng, cùng 1 file `index.html`, chỉ khác base image — chênh lệch
gần gấp đôi dung lượng.

## Bước 3 — So sánh số lượng CVE (đã test thực tế bằng Trivy — xem Lab 4.1)

```bash
trivy image --severity CRITICAL,HIGH custom-webapp:debian | grep "Total:"
trivy image --severity CRITICAL,HIGH custom-webapp:alpine | grep "Total:"
```

Kết quả thực tế:

| Image | CRITICAL+HIGH |
|---|---|
| `custom-webapp:debian` | 65 (HIGH: 61, CRITICAL: 4) |
| `custom-webapp:alpine` | 22 (HIGH: 22, CRITICAL: 0) |

Alpine giảm cả kích thước lẫn CVE — vì `musl libc` + `apk` package set nhỏ
hơn hẳn `glibc` + hệ sinh thái `apt`/Debian.

## Bước 4 — Xem layer nào chiếm dung lượng (để biết chỗ tối ưu tiếp)

```bash
docker history custom-webapp:debian --format "table {{.Size}}\t{{.CreatedBy}}" | head -10
```

Layer lớn nhất luôn là layer cài base OS + runtime — đây là lý do "chọn
đúng base image" tác động lớn hơn nhiều so với việc tối ưu Dockerfile ở các
bước sau.

## Bước 5 — Nguyên tắc bổ sung (áp dụng thủ công, không có lệnh chạy sẵn)

Từ tài liệu, các best practice khác cần nhớ khi viết Dockerfile thật:

1. **1 image = 1 trách nhiệm** — không nhét web server + database chung 1
   image.
2. **Không lưu state trong container** — dùng volume/service ngoài (Redis,
   PVC) thay vì ghi data vào filesystem của container.
3. **Xoá tool không cần ở production** — nếu image production không cần
   `curl`/`wget`/`apt`, đừng để lại (kẻ tấn công có thể lợi dụng để tải thêm
   payload sau khi khai thác được RCE).
4. **Tách image dev/prod** — image dev có thể có debug tool, image prod thì
   không.

## Dọn dẹp

```bash
docker rmi custom-webapp:debian custom-webapp:alpine
rm -rf /tmp/webapp-lab
```

## Mẹo thi

- CKS không yêu cầu bạn build Docker image trong lúc thi (không có Docker
  daemon truy cập trực tiếp thường thấy) — trọng tâm là **đọc hiểu
  Dockerfile có sẵn** và nhận diện dòng nào làm tăng attack surface không
  cần thiết (base image nặng, cài thừa package, để lại `curl`/`wget`/`ssh`
  trong image production).
- Câu hỏi dạng "tại sao nên dùng distroless/alpine thay vì ubuntu/debian
  đầy đủ" — trả lời đúng trọng tâm: **giảm số package = giảm CVE + giảm
  attack surface**, không chỉ vì "nhẹ hơn, tải nhanh hơn".
