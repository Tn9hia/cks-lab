# Lab 4.2 — Lint Manifest với KubeLinter

**Domain:** Supply Chain Security
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** `kube-linter` CLI (`brew install kube-linter` — đã có sẵn trên
máy, không cần cluster).

## Mục tiêu

- Phát hiện lỗi cấu hình phổ biến trong manifest TRƯỚC khi deploy (dịch
  chuyển việc kiểm tra bảo mật sang sớm hơn trong pipeline — "shift left").
- Sửa từng lỗi đến khi manifest sạch hoàn toàn, hiểu rõ lý do đằng sau từng
  rule thay vì chỉ sửa cho hết lỗi.

## Bước 1 — Lint 1 manifest cố tình có nhiều lỗi

```bash
kube-linter lint manifests/bad-deployment.yaml
```

Kết quả thực tế (7 lỗi):

| Check | Vấn đề |
|---|---|
| `latest-tag` | Dùng tag `:latest` — không kiểm soát được version thực sự đang chạy |
| `no-read-only-root-fs` | Root filesystem không readonly |
| `privilege-escalation-container` | Container privileged nên cho phép leo quyền |
| `privileged-container` | Container chạy `privileged: true` |
| `run-as-non-root` | Không set `runAsNonRoot` |
| `unset-cpu-requirements` | Thiếu `resources.requests.cpu` |
| `unset-memory-requirements` | Thiếu `resources.limits.memory` |

## Bước 2 — Sửa từng lỗi, so sánh với bản đã sửa

Xem `manifests/good-deployment.yaml` — đã sửa toàn bộ 7 lỗi trên, cộng thêm:
- `replicas: 3` (thay vì 1) kèm `podAntiAffinity` để tránh cả 3 replica bị
  scheduler dồn chung 1 node (rule `no-anti-affinity` — chỉ kích hoạt khi
  replicas > 1, nên nếu bạn chỉ tăng replicas mà không thêm anti-affinity sẽ
  phát sinh lỗi mới).
- `livenessProbe`/`readinessProbe` để healthcheck đúng cách.

```bash
kube-linter lint manifests/good-deployment.yaml
# -> No lint errors found!
```

## Bước 3 — Xem toàn bộ checks có sẵn

```bash
kube-linter checks list | head -60
```

KubeLinter có rất nhiều check không chỉ về bảo mật (vd `minimum-three-replicas`,
`no-node-affinity`...) — mặc định chỉ 1 tập con được bật (profile
`default`), không phải TẤT CẢ check đều chạy tự động.

## Bước 4 — Tuỳ chỉnh bộ rule (bật/tắt theo nhu cầu tổ chức)

```bash
cat > /tmp/kubelinter-config.yaml <<'EOF'
checks:
  doNotAutoAddDefaults: true
  include:
    - "privileged-container"
    - "run-as-non-root"
    - "latest-tag"
    - "unset-cpu-requirements"
    - "unset-memory-requirements"
EOF

kube-linter lint --config /tmp/kubelinter-config.yaml manifests/bad-deployment.yaml
```

Giờ chỉ hiện đúng 5 check được `include`, bỏ qua các check khác.

> **Đã kiểm chứng — lưu ý dễ nhầm:** field đúng là `doNotAutoAddDefaults:
> true`, KHÔNG phải `addAllBuiltIn: false`. Mặc định kube-linter **luôn tự
> động thêm profile "default"** trừ khi bạn tắt tường minh bằng
> `doNotAutoAddDefaults` — chỉ set `addAllBuiltIn: false` (giá trị vốn đã là
> false mặc định) không có tác dụng gì, `include` sẽ chỉ CỘNG THÊM vào
> default profile chứ không thay thế nó. Mình đã thử sai cách này trước khi
> tìm ra field đúng.

## Bước 5 — Tích hợp vào pipeline (khái niệm)

```bash
kube-linter lint manifests/good-deployment.yaml; echo "Exit code: $?"
# Exit code: 0 nếu pass, khác 0 nếu có lỗi -> dùng trực tiếp trong CI step
```

Trong Jenkins/GitHub Actions, bước "lint" đặt trước bước "build image" —
lint fail thì dừng pipeline ngay, tiết kiệm thời gian build/push image cho
1 manifest chắc chắn không đạt chuẩn.

## Dọn dẹp

```bash
rm -f /tmp/kubelinter-config.yaml
```

## Mẹo thi

- CKS không hỏi bạn nhớ tên từng check, nhưng cần biết **loại vấn đề nào
  KubeLinter/linter nói chung phát hiện được** (privileged, latest tag,
  thiếu resource limits, thiếu security context) — đây đều là kiến thức đã
  học ở Lab 3.1/3.2, KubeLinter chỉ là công cụ tự động hoá việc kiểm tra đó.
- Phân biệt rõ với Trivy (Lab 4.1): **Trivy scan nội dung bên trong image**
  (package, CVE), **KubeLinter scan manifest YAML** (cấu hình pod/deployment)
  — hai lớp kiểm tra bổ sung nhau, không thay thế nhau.
