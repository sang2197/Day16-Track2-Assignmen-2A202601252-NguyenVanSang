# Báo cáo ngắn — Benchmark LightGBM trên CPU (t3.micro)

**Họ tên:** Nguyễn Văn Sáng
**MSSV:** 2A202601252
**Ngày:** 15/08/2026
**Repo:** https://github.com/sang2197/Day16-Track2-Assignmen-2A202601252-NguyenVanSang.git

---

**Thời gian hạ tầng:** `terraform apply` (bao gồm cả lần retry do lỗi instance type không thuộc Free Tier) mất khoảng 19 phút, phần lớn là NAT Gateway (~10-15 phút theo tài liệu AWS) — khớp với ước tính trong README. Sau khi instance chạy, script bootstrap qua `user_data` (cài Python, LightGBM, scikit-learn, Kaggle CLI) hoàn tất trong 1-2 phút, xác nhận bằng việc chạy ngay được lệnh kiểm tra môi trường mà không lỗi thiếu module.

**Training & AUC-ROC:** Huấn luyện 227,845 dòng train (30 feature) mất **4.54 giây** cho 100 iteration, đạt **AUC-ROC 0.8203** — cho thấy CPU giá rẻ xử lý tốt bài toán tabular ở quy mô này.

**Precision/Recall:** Precision **0.4959** và Recall **0.6224** cho thấy model vừa **bỏ sót** gần 38% giao dịch gian lận thật (recall thấp), vừa **báo nhầm** khoảng 50% các cảnh báo đưa ra (precision thấp) — hệ quả của dataset mất cân bằng nặng (0.17% là fraud) khi dùng tham số mặc định, chưa xử lý class weight hay tối ưu ngưỡng phân loại.

**Latency vs Throughput:** Dự đoán 1 dòng đơn lẻ mất **1.52ms**, nhưng khi dự đoán theo batch 1000 dòng, throughput đạt **165,639 dòng/giây** — tương đương chỉ ~0.006ms/dòng trong batch, tức rẻ hơn ~250 lần so với gọi đơn lẻ. Chênh lệch này đến từ chi phí cố định của mỗi lần gọi hàm `predict()` (overhead Python/LightGBM) bị "pha loãng" khi xử lý nhiều dòng cùng lúc.

**CPU/RAM có phải bottleneck?** Không rõ ràng ở quy mô dataset này: `free -h` cho thấy chỉ dùng 185MB/914MB RAM, không có swap, và `top` ghi nhận load average 0.00 (t3.micro dư sức xử lý). Tuy nhiên hai chỉ số này được đo *sau khi* training đã hoàn tất (vì training chỉ mất ~4.5 giây, quá ngắn để bắt được đỉnh tải), nên chưa phản ánh đúng mức tải thực tại thời điểm training — RAM/CPU burst credit của t3.micro có thể là giới hạn thật sự nếu dataset lớn hơn nhiều.

**Chi phí cloud:** Theo bảng ước tính của README, **NAT Gateway** (~$0.045/giờ + phí data processing) là thành phần đóng góp chi phí lớn nhất và liên tục, vì nó tính phí theo giờ 24/7 bất kể có dùng hay không — trong khi EC2 (đã hạ xuống `t3.micro` để hợp lệ Free Tier) gần như $0. ALB cũng thu phí cố định theo giờ tương tự dù ít traffic. Đây là lý do README nhấn mạnh phải `terraform destroy` ngay sau khi hoàn thành để tránh chi phí cộng dồn theo thời gian.

**Ghi chú về ảnh Billing (`billing.png`):** tài khoản AWS dùng cho bài lab thuộc loại **"AWS Free Plan"** — banner trên trang Bills nêu rõ *"Your free plan account does not get charged. Credits cover your free plan costs."* Do toàn bộ chi phí được credit bao trọn tự động, số tiền hiển thị luôn là `USD 0.00` cho mọi dịch vụ. Ban đầu trang Bills không hiện dữ liệu gì (do độ trễ xử lý của AWS), nhưng sau hơn 24 giờ, hệ thống đã trả ra bảng chi phí itemize theo từng dịch vụ (10 active services), trong đó thể hiện rõ đúng các dịch vụ hạ tầng lab đã tạo: **Elastic Compute Cloud** (EC2 — Bastion + Compute Node), **Virtual Private Cloud** (nhóm billing bao gồm NAT Gateway), và **Elastic Load Balancing** (ALB) — khớp với bảng ước tính chi phí trong README, chỉ khác là số tiền thực tế bằng $0 nhờ credit Free Plan bù toàn bộ.
