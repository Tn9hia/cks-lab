# Lab 1.6 — Audit Logging

**Domain:** Cluster Hardening — Audit logs
**Thời gian gợi ý:** 25-30 phút
**Yêu cầu:** SSH vào node `k8s-controlplane-01` với quyền root/sudo (máy
Claude Code không có quyền này — toàn bộ lab này bạn tự SSH và chạy).

## Mục tiêu

- Viết audit policy với các level `None`, `Metadata`, `Request`,
  `RequestResponse` cho từng loại resource khác nhau.
- Bật audit logging trên `kube-apiserver` bằng cách sửa static pod manifest.
- Đọc và lọc audit log để trả lời câu hỏi kiểu "ai đã xoá pod X lúc nào".

Đây là một trong những bài dễ mất điểm nhất trong đề thi thật vì chỉ cần sai
một flag hoặc quên mount volume là `kube-apiserver` sẽ crash-loop và toàn bộ
cluster "sập" cho tới khi bạn revert lại đúng file gốc — luôn **backup file
manifest gốc trước khi sửa**.

## Bước 1 — SSH vào control-plane node

```bash
ssh <user>@172.16.5.12   # hoặc cách bạn vẫn dùng để truy cập node
sudo -i
```

Từ đây, mọi lệnh chạy **trên node**, không phải máy làm việc.

## Bước 2 — Backup manifest gốc (bắt buộc)

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/kube-apiserver.yaml.bak
```

Nếu có sự cố, restore bằng:

```bash
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

(kubelet tự phát hiện thay đổi trong `/etc/kubernetes/manifests/` và
recreate static pod — không cần restart service thủ công.)

## Bước 3 — Tạo thư mục và audit policy

```bash
mkdir -p /etc/kubernetes/audit
```

Tạo `/etc/kubernetes/audit/policy.yaml` với nội dung ở
`manifests/audit-policy.yaml` (copy nội dung sang node, ví dụ bằng `scp` từ
máy làm việc hoặc gõ tay bằng `vi`). Policy này minh hoạ đủ 4 level:

- `RequestResponse` cho `secrets` — log toàn bộ vì đây là resource nhạy cảm nhất.
- `Request` cho `pods/exec`, `pods/attach` — cần biết ai đã exec vào container nào.
- `Metadata` cho `pods`, `deployments` — đủ để trace who/when mà không phình log.
- `None` cho các request `get`/`watch` tới `events`, `leases` (nhiễu, ít giá trị).

## Bước 4 — Sửa `kube-apiserver.yaml` để bật audit

Thêm vào `spec.containers[0].command` 3 flag:

```yaml
- --audit-policy-file=/etc/kubernetes/audit/policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=7
- --audit-log-maxbackup=3
- --audit-log-maxsize=100
```

Và thêm 2 `volumeMounts` + `volumes` tương ứng để container thấy được cả thư
mục policy lẫn thư mục ghi log (container filesystem tách biệt với host):

```yaml
    volumeMounts:
      - name: audit-policy
        mountPath: /etc/kubernetes/audit
        readOnly: true
      - name: audit-log
        mountPath: /var/log/kubernetes/audit
  volumes:
    - name: audit-policy
      hostPath:
        path: /etc/kubernetes/audit
        type: DirectoryOrCreate
    - name: audit-log
      hostPath:
        path: /var/log/kubernetes/audit
        type: DirectoryOrCreate
```

Tham khảo file đầy đủ đã diff sẵn ở `manifests/kube-apiserver-audit-diff.yaml`
(chỉ để đối chiếu — **không apply trực tiếp** file này vì manifest gốc của
bạn có thể khác, ví dụ IP, version, cert paths).

## Bước 5 — Verify apiserver sống lại đúng cách

```bash
mkdir -p /var/log/kubernetes/audit
watch crictl ps --name kube-apiserver
# Ctrl+C khi thấy container ở trạng thái Running, không restart liên tục
```

Nếu apiserver không lên được:

```bash
crictl logs $(crictl ps -a --name kube-apiserver -q | head -1)
```

Lỗi hay gặp: sai path trong `--audit-policy-file`, thiếu `volumeMounts`, hoặc
YAML sai indent làm apiserver không parse được flag.

## Bước 6 — Verify từ máy làm việc

```bash
kubectl get nodes   # nếu lệnh này chạy được nghĩa là apiserver đã sống lại
```

Tạo và xoá 1 pod để sinh audit event:

```bash
kubectl run audit-demo --image=nginx -n default
kubectl exec -n default audit-demo -- true
kubectl delete pod audit-demo -n default
```

## Bước 7 — Đọc audit log (trên node)

```bash
tail -f /var/log/kubernetes/audit/audit.log | jq .

# Tìm ai đã xoá pod audit-demo
cat /var/log/kubernetes/audit/audit.log | \
  jq -c 'select(.objectRef.name=="audit-demo" and .verb=="delete")'

# Tìm mọi lệnh exec vào pod (mức Request — có cả request body)
cat /var/log/kubernetes/audit/audit.log | \
  jq -c 'select(.objectRef.subresource=="exec")'

# Đếm số request theo user
cat /var/log/kubernetes/audit/audit.log | jq -r '.user.username' | sort | uniq -c | sort -rn
```

## Dọn dẹp / Restore

```bash
cp /root/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
watch crictl ps --name kube-apiserver   # đợi apiserver quay lại bản gốc
```

## Mẹo thi

- Trong đề thi, câu audit logging thường cho sẵn 1 phần policy và yêu cầu bạn
  hoàn thiện + bật flag — luôn nhớ đúng 3 flag bắt buộc:
  `--audit-policy-file`, `--audit-log-path`, và đảm bảo cả 2 đều có
  `hostPath` + `volumeMounts` tương ứng (đây là lỗi thi rớt phổ biến nhất:
  set flag đúng nhưng quên mount volume nên file trong container không tồn tại).
- Luôn xác nhận `kubectl get nodes` (hoặc bất kỳ lệnh kubectl nào) chạy được
  ngay sau khi sửa — nếu treo/lỗi, quay lại xem `crictl logs` trước khi làm
  tiếp, đừng đoán mò sửa thêm.
- Audit policy được đánh giá theo **thứ tự rule từ trên xuống, rule đầu tiên
  khớp sẽ được áp dụng** — đặt rule cụ thể (secrets, exec) lên trên, rule
  chung chung (catch-all) xuống dưới cùng.
