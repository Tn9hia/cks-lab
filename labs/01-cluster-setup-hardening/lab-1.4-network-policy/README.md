# Lab 1.4 — NetworkPolicy với Cilium

**Domain:** Cluster Hardening — Restrict network access
**Thời gian gợi ý:** 25 phút
**Yêu cầu:** Cilium CNI (đã có sẵn trên cụm của bạn).

## Mục tiêu

- Áp dụng default-deny cho một namespace.
- Mở lần lượt ingress/egress cho từng cặp pod cụ thể bằng `podSelector`.
- Hiểu vì sao thiếu egress rule cho DNS là lỗi phổ biến nhất khi làm bài
  NetworkPolicy trong đề thi (pod bị deny DNS lookup dù rule "đúng").

## Bước 0 — Dựng 3 pod mẫu: frontend, backend, db

```bash
kubectl create namespace lab-netpol
kubectl apply -f manifests/00-apps.yaml
kubectl wait --for=condition=Ready pod -l tier -n lab-netpol --timeout=60s
```

Baseline (chưa có policy nào — mọi pod gọi được nhau):

```bash
kubectl exec -n lab-netpol deploy/frontend -- curl -s -o /dev/null -w "%{http_code}\n" http://backend
kubectl exec -n lab-netpol deploy/backend -- curl -s -o /dev/null -w "%{http_code}\n" http://db
kubectl exec -n lab-netpol deploy/frontend -- curl -s -o /dev/null -w "%{http_code}\n" http://db
# cả 3 đều 200 — frontend đang gọi thẳng được db, đây là điều cần chặn
```

## Bước 1 — Default deny all ingress + egress trong namespace

```bash
kubectl apply -f manifests/01-default-deny.yaml
```

Verify mọi kết nối đều bị chặn kể cả DNS:

```bash
kubectl exec -n lab-netpol deploy/frontend -- curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://backend || echo "blocked (đúng)"
```

## Bước 2 — Cho phép DNS egress (bước hay bị quên)

```bash
kubectl apply -f manifests/02-allow-dns.yaml
```

## Bước 3 — Cho phép frontend -> backend, backend -> db (không cho frontend -> db)

```bash
kubectl apply -f manifests/03-allow-frontend-backend.yaml
kubectl apply -f manifests/04-allow-backend-db.yaml
```

Verify:

```bash
echo "frontend -> backend (expect 200):"
kubectl exec -n lab-netpol deploy/frontend -- curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://backend

echo "backend -> db (expect 200):"
kubectl exec -n lab-netpol deploy/backend -- curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://db

echo "frontend -> db (expect timeout/blocked):"
kubectl exec -n lab-netpol deploy/frontend -- curl -s -m 3 -o /dev/null -w "%{http_code}\n" http://db || echo "blocked (đúng)"
```

## Bước 4 — Quan sát bằng Hubble (điểm cộng vì cụm bạn có sẵn Hubble UI)

```bash
kubectl -n kube-system port-forward svc/hubble-ui 12000:80
```

Mở `http://localhost:12000`, chọn namespace `lab-netpol`, xem trực quan traffic
nào bị "dropped" theo policy — rất hữu ích để debug NetworkPolicy trong lúc
luyện tập (trong phòng thi thật sẽ không có UI này).

Hoặc dùng CLI nếu có `hubble` cài trên máy/pod:

```bash
kubectl -n kube-system exec ds/cilium -- hubble observe --namespace lab-netpol --verdict DROPPED -n 20
```

> **Lưu ý đã kiểm chứng:** `CiliumNetworkPolicy` với `rules.http` (lọc theo
> method/path ở L7) được thử nghiệm trên cụm này (Cilium 1.20) nhưng **không
> enforce như tài liệu mô tả** — mọi request vẫn bị `FORWARDED` dù method/path
> không khớp rule. Đây có thể là hành vi đặc thù của phiên bản/cấu hình cụm
> bạn, không phải lỗi thao tác. Vì L7 CiliumNetworkPolicy **không nằm trong
> phạm vi thi CKS** (đề thi chỉ yêu cầu `networking.k8s.io/v1` NetworkPolicy
> chuẩn), mình bỏ qua phần này để tránh học sai kiến thức — nếu bạn muốn đào
> sâu Cilium L7 thì đó là một nhánh điều tra riêng, không phải nội dung CKS.

## Dọn dẹp

```bash
kubectl delete namespace lab-netpol
```

## Mẹo thi

- Luôn nhớ: NetworkPolicy là **whitelist theo namespace** — một khi có bất kỳ
  policy nào chọn trúng pod cho hướng ingress hoặc egress, hướng đó mặc định
  chuyển sang deny-all trừ khi được khai báo rõ.
- `policyTypes` phải khai đủ `Ingress` và `Egress` nếu bài yêu cầu chặn cả
  hai chiều — thiếu 1 trong 2 là lỗi hay gặp nhất.
- Khi pod dùng DNS để resolve service name, luôn cần rule egress tới
  `kube-dns`/`coredns` (port 53 UDP/TCP), nếu không toàn bộ traffic "tưởng
  đã cho phép" vẫn fail vì không resolve được tên.
- NetworkPolicy chuẩn (`networking.k8s.io/v1`) là bắt buộc phải biết cho đề
  thi; CiliumNetworkPolicy (L7) là kiến thức mở rộng, không phải lúc nào
  cũng có trong đề nhưng rất hay gặp trong công việc thực tế với Cilium.
