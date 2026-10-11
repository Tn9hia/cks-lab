# Lab 3.5 — Resource Quotas & LimitRange

**Domain:** Minimize Microservice Vulnerabilities (multi-tenancy)
**Thời gian gợi ý:** 20 phút
**Yêu cầu:** chỉ cần `kubectl`.

## Mục tiêu

- Giới hạn tổng tài nguyên 1 namespace được dùng bằng `ResourceQuota`.
- Hiểu hệ quả quan trọng: khi namespace có `ResourceQuota` khai
  `requests.cpu`/`requests.memory`, **mọi pod tạo trong namespace đó bắt
  buộc phải khai `resources.requests`**, nếu không sẽ bị từ chối thẳng.
- Dùng `LimitRange` để tự động gán default requests/limits, tránh phải sửa
  từng pod.

## Bước 1 — Tạo ResourceQuota

```bash
kubectl create namespace team-a
kubectl apply -f manifests/resourcequota.yaml
kubectl describe resourcequota cpu-memory-quota -n team-a
```

## Bước 2 — Pod KHÔNG khai resources -> bị từ chối

```bash
kubectl run no-resources --image=nginx -n team-a
```

Kết quả mong đợi: bị từ chối ngay, message dạng "must specify
limits.cpu,limits.memory,requests.cpu,requests.memory" — đây là hệ quả trực
tiếp của việc namespace có Quota nhưng pod không khai resources.

## Bước 3 — Pod khai đúng resources -> được tạo

```bash
kubectl apply -f manifests/pod-with-resources.yaml
kubectl get pod resource-limited-pod -n team-a
kubectl describe resourcequota cpu-memory-quota -n team-a
# -> cột "Used" tăng lên đúng bằng request của pod
```

## Bước 4 — Vượt quota -> bị từ chối

```bash
kubectl apply -f manifests/pod-exceeds-quota.yaml
```

Pod này request `cpu: "5"` trong khi quota namespace chỉ cho phép tổng
`requests.cpu: "4"` (đã dùng 250m ở Bước 3) -> bị từ chối vì vượt hạn mức
còn lại.

## Bước 5 — Dùng LimitRange để không phải khai resources thủ công mỗi pod

Xoá pod thất bại trước, áp LimitRange đặt default:

```bash
kubectl delete pod no-resources -n team-a --ignore-not-found
kubectl apply -f manifests/limitrange.yaml

kubectl run auto-resources --image=nginx -n team-a
kubectl get pod auto-resources -n team-a -o jsonpath='{.spec.containers[0].resources}'
```

Giờ pod không khai resources vẫn được tạo thành công — LimitRange tự động
gán `default`/`defaultRequest` từ cấu hình namespace, thoả điều kiện của
ResourceQuota mà không cần sửa YAML của từng pod.

## Bước 6 — Quan sát Quality of Service (QoS) class

```bash
kubectl get pod resource-limited-pod -n team-a -o jsonpath='{.status.qosClass}'
# -> Burstable (vì requests != limits)

kubectl apply -f manifests/pod-guaranteed.yaml
kubectl get pod guaranteed-pod -n team-a -o jsonpath='{.status.qosClass}'
# -> Guaranteed (vì requests == limits cho MỌI container)
```

QoS class quyết định thứ tự pod bị **evict trước** khi node thiếu tài
nguyên: `BestEffort` (không khai gì) bị evict trước tiên, rồi đến
`Burstable`, cuối cùng mới tới `Guaranteed`.

## Dọn dẹp

```bash
kubectl delete namespace team-a
```

## Mẹo thi

- Đây là kiến thức chuyển tiếp từ CKA — nếu đã quen ResourceQuota/LimitRange
  ở CKA thì phần này chỉ là ôn lại, KHÔNG có gì đặc thù riêng cho bảo mật
  ngoài việc dùng nó như 1 lớp phòng vệ **multi-tenancy** (1 namespace/tenant
  bị compromise hoặc bug cũng không thể ăn hết tài nguyên node, ảnh hưởng
  tenant khác).
- Luôn nhớ: `ResourceQuota` cho `requests.cpu/requests.memory` bắt buộc kéo
  theo yêu cầu **mọi pod** trong namespace đó phải khai `resources` — dùng
  `LimitRange` để tránh phải sửa YAML ứng dụng có sẵn.
- 3 QoS class (`Guaranteed` > `Burstable` > `BestEffort`) không phải field
  bạn tự set — Kubernetes tự tính dựa trên requests/limits của pod.
