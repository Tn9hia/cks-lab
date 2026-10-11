# Lab 3.2 — Pod Security Admission (PSA)

**Domain:** Minimize Microservice Vulnerabilities
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** chỉ cần `kubectl` — PSA là admission controller **built-in**,
bật sẵn từ Kubernetes 1.25+, không cần cài thêm gì (đã verify trên cụm bạn,
v1.36).

## Mục tiêu

- Áp dụng 3 profile chuẩn: `privileged`, `baseline`, `restricted`.
- Áp dụng 3 mode: `enforce`, `audit`, `warn` — và hiểu khác biệt.
- Tái tạo đúng ví dụ 3 namespace trong tài liệu KodeKloud: `payroll`
  (enforce restricted), `hr` (enforce baseline), `dev` (warn restricted).

## Bước 0 — Xác nhận PSA đang bật

```bash
kubectl exec -n kube-system kube-apiserver-k8s-controlplane-01 -- \
  kube-apiserver -h 2>&1 | grep -A1 "enable-admission-plugins"
```

(PSA hoạt động như 1 built-in admission controller từ 1.25+, không xuất
hiện trong danh sách bật/tắt như plugin cũ — lệnh trên chỉ để tham khảo cú
pháp; cách xác nhận chắc chắn nhất là **thử luôn** ở bước dưới.)

## Bước 1 — Tạo 3 namespace theo đúng ví dụ

```bash
kubectl create namespace payroll
kubectl create namespace hr
kubectl create namespace dev

kubectl label ns payroll pod-security.kubernetes.io/enforce=restricted
kubectl label ns hr pod-security.kubernetes.io/enforce=baseline
kubectl label ns dev pod-security.kubernetes.io/warn=restricted
```

## Bước 2 — Test namespace `payroll` (enforce restricted)

```bash
# Pod privileged -> bị REJECT hoàn toàn (enforce chặn, không tạo được)
kubectl run bad-pod -n payroll --image=nginx --overrides='{"spec":{"containers":[{"name":"bad-pod","image":"nginx","securityContext":{"privileged":true}}]}}'
```

Kết quả mong đợi: lỗi ngay tại `kubectl run`, pod **không được tạo**, message
liệt kê chi tiết field nào vi phạm (privileged, allowPrivilegeEscalation,
runAsNonRoot, seccompProfile...).

```bash
# Pod tuân thủ restricted -> được tạo
kubectl apply -f manifests/payroll-compliant-pod.yaml
kubectl get pod -n payroll
```

## Bước 3 — Test namespace `hr` (enforce baseline — lỏng hơn restricted)

```bash
# Pod privileged vẫn bị chặn (baseline cũng cấm privileged)
kubectl run bad-pod -n hr --image=nginx --overrides='{"spec":{"containers":[{"name":"bad-pod","image":"nginx","securityContext":{"privileged":true}}]}}'
# -> reject

# Nhưng pod chạy root (không set runAsNonRoot) vẫn được phép — baseline không cấm root user
kubectl run root-ok -n hr --image=nginx
kubectl get pod root-ok -n hr
# -> Running, không bị chặn (khác với payroll ở Bước 2)
```

Đây chính là điểm khác biệt cốt lõi `baseline` vs `restricted`: baseline chỉ
chặn các hành vi rõ ràng nguy hiểm (privileged, hostNetwork, hostPID...),
không ép non-root/readOnlyRootFilesystem/drop-capabilities như restricted.

## Bước 4 — Test namespace `dev` (warn restricted — không chặn, chỉ cảnh báo)

```bash
kubectl run bad-pod -n dev --image=nginx --overrides='{"spec":{"containers":[{"name":"bad-pod","image":"nginx","securityContext":{"privileged":true}}]}}'
```

Kết quả mong đợi: pod **vẫn được tạo** (không bị chặn), nhưng `kubectl` in
ra `Warning:` liệt kê các field vi phạm restricted — đây là cách để "dò"
compliance trước khi chuyển hẳn sang `enforce`.

## Bước 5 — Kết hợp nhiều mode cùng lúc trên 1 namespace

Một namespace có thể vừa `enforce` baseline vừa `warn`/`audit` restricted
cùng lúc — dùng để "chuẩn bị nâng cấp" security level dần dần mà không phá
vỡ workload hiện tại ngay lập tức:

```bash
kubectl label ns dev pod-security.kubernetes.io/enforce=baseline --overwrite
kubectl label ns dev pod-security.kubernetes.io/audit=restricted --overwrite
kubectl label ns dev pod-security.kubernetes.io/warn=restricted --overwrite

kubectl get ns dev --show-labels
```

Giờ namespace `dev` sẽ: chặn cứng mọi thứ vi phạm baseline (enforce), đồng
thời log vào audit log + cảnh báo qua CLI cho bất kỳ thứ gì vi phạm
restricted (dù vẫn cho pass) — chuẩn bị dữ liệu để biết cần sửa gì trước khi
enforce restricted thật sự.

> **Đã kiểm chứng — bonus insight:** nếu namespace đã có sẵn pod (từ Bước 4,
> `bad-pod` chạy privileged), việc đổi label `enforce` sang `baseline` khiến
> `kubectl label` in luôn cảnh báo retroactive: `Warning: existing pods in
> namespace "dev" violate the new PodSecurity enforce level "baseline:latest"`.
> PSA chỉ cảnh báo, **không tự xoá** pod đã chạy vi phạm — enforce chỉ chặn
> request **mới** (create/update), pod cũ tiếp tục chạy bình thường cho đến
> khi bị xoá/tạo lại theo cách khác.

## Dọn dẹp

```bash
kubectl delete namespace payroll hr dev
```

## Mẹo thi

- Nhớ chính xác 3 profile (`privileged`, `baseline`, `restricted`) và 3 mode
  (`enforce`, `audit`, `warn`) — đề thi thường cho sẵn yêu cầu kiểu "namespace
  X phải enforce baseline, namespace Y chỉ cảnh báo restricted" và bạn chỉ
  cần gõ đúng `kubectl label ns ...`, không cần YAML.
- PSA áp dụng ở **namespace level qua label**, khác hẳn với Pod Security
  Policies (PSP) cũ vốn là 1 API object riêng — đừng nhầm 2 khái niệm này,
  PSP đã bị **remove hoàn toàn** khỏi Kubernetes từ 1.25.
- `restricted` yêu cầu khá nhiều field cùng lúc (`runAsNonRoot: true`,
  `seccompProfile.type` được set, `allowPrivilegeEscalation: false`,
  `capabilities.drop: [ALL]`) — nếu pod của bạn bị reject ở namespace
  restricted, dùng `warn` mode trước để xem chính xác thiếu field nào thay
  vì đoán.
