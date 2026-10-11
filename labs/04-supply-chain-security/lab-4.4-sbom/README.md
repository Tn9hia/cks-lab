# Lab 4.4 — SBOM (Software Bill of Materials)

**Domain:** Supply Chain Security
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** `syft` + `grype` (`brew install syft grype` — đã cài, không
cần cluster).

## Mục tiêu

- Hiểu SBOM là "danh sách nguyên liệu" đầy đủ của 1 image: mọi package, mọi
  version, dùng để biết chính xác cái gì đang chạy bên trong.
- Sinh SBOM bằng Syft, scan SBOM đó bằng Grype (tách biệt 2 bước: sinh và
  quét — cho phép lưu trữ/audit SBOM độc lập với việc scan).
- Phân biệt 2 format phổ biến: SPDX (thiên về license/compliance) và
  CycloneDX (thiên về security/vulnerability).

## Bước 1 — Sinh SBOM cho 1 image

```bash
syft nginx:1.27-alpine -o spdx-json > /tmp/nginx-sbom.spdx.json
python3 -c "import json; d=json.load(open('/tmp/nginx-sbom.spdx.json')); print('packages:', len(d.get('packages',[])))"
# -> packages: 69 (con số thực tế khi test — image tưởng "nhẹ" vẫn có gần 70 package)
```

Sinh dạng CycloneDX (format khác, cùng nội dung):

```bash
syft nginx:1.27-alpine -o cyclonedx-json > /tmp/nginx-sbom.cyclonedx.json
```

Sinh cho local source code (không chỉ image):

```bash
syft /tmp -o table 2>/dev/null | head -20
```

## Bước 2 — Đọc thử 1 phần SBOM

```bash
python3 -m json.tool /tmp/nginx-sbom.spdx.json | grep -A5 '"name": "nginx"' | head -10
```

Mỗi package trong SBOM có: tên, version, checksum, download location, và
`externalRefs` dạng `cpe23Type`/`purl` — đây chính là "khoá" để đối chiếu
với CVE database sau này.

## Bước 3 — Scan SBOM bằng Grype (tách biệt bước generate và bước scan)

```bash
grype sbom:/tmp/nginx-sbom.spdx.json
```

Output liệt kê package, version hiện tại, version đã fix (nếu có), CVE ID,
severity — tương tự Trivy nhưng input là file SBOM tĩnh thay vì kéo lại
image từ registry. Đây là điểm khác biệt quan trọng: **SBOM cho phép scan
lại nhiều lần, nhiều công cụ, mà không cần quyền truy cập registry/image gốc
mỗi lần** — hữu ích khi audit compliance định kỳ hoặc chia sẻ cho bên thứ 3
kiểm tra mà không cần cấp quyền pull image.

## Bước 4 — Workflow đầy đủ (theo đúng 6 bước trong tài liệu)

1. **Generate** — `syft <image> -o <format>` (đã làm ở Bước 1).
2. **Store** — lưu file SBOM vào registry riêng (JFrog, Nexus, GitHub
   Packages) hoặc đơn giản là attach vào release artifact trong CI/CD.
3. **Scan** — `grype sbom:<file>` (đã làm ở Bước 3).
4. **Analyze** — đọc kỹ từng CVE, ưu tiên theo severity + exploitability.
5. **Remediate** — update package lên version đã fix, hoặc thay thế
   package/base image.
6. **Monitor** — chạy lại scan định kỳ (SBOM là ảnh chụp tại 1 thời điểm,
   CVE database luôn cập nhật nên hôm nay "sạch" không có nghĩa tuần sau
   vẫn sạch).

## Dọn dẹp

```bash
rm -f /tmp/nginx-sbom.spdx.json /tmp/nginx-sbom.cyclonedx.json
```

## Mẹo thi

- SBOM **không phải** kết quả scan vulnerability — nó là **danh sách thành
  phần** (inventory). Grype/Trivy mới là bước đối chiếu danh sách đó với CVE
  database. Nhầm 2 khái niệm này là lỗi hay gặp khi trả lời câu hỏi lý
  thuyết.
- SPDX thiên về **license compliance** (biết chính xác đang dùng license gì,
  có vi phạm điều khoản open-source nào không), CycloneDX thiên về
  **security** (tối ưu cho việc đối chiếu CVE) — nếu đề hỏi "chọn format nào
  cho mục đích X", trả lời theo đúng thế mạnh từng format.
- SBOM hữu ích nhất khi kết hợp với image đã build sẵn từ lâu mà không còn
  quyền truy cập Dockerfile gốc — SBOM cho biết chính xác bên trong có gì mà
  không cần rebuild hay pull lại.
