# Báo cáo Bonus — Phân tích Thí nghiệm 4C & Tập Validation: "Tập Val Có Nói Thật Không?"

**Học viên:** Nguyễn Thành Luân  
**Mã sinh viên:** 2A202602769  
**Bài Lab:** Lab 18 — 2D Perception: Detection · Segmentation · Keypoints  

---

## 1. Đặt vấn đề và Mục tiêu thí nghiệm

Trong bài toán phát hiện tư thế (Pose Estimation) cho đối tượng không đối xứng về mặt giải phẫu (như động vật bốn chân hoặc người), kỹ thuật tăng cường dữ liệu lật ngang (Horizontal Flip Augmentation - `fliplr=0.5`) là kỹ thuật phổ biến nhằm nhân đôi kích thước và sự đa dạng của tập huấn luyện.

Tuy nhiên, nếu cấu hình `flip_idx` không đúng theo quy ước **giải phẫu học** (anatomical convention) mà giữ nguyên cấu hình **đồng nhất** (identity mapping: `[0, 1, ..., 11]`), model sẽ bị gán nhãn sai khi ảnh bị lật gương.

Câu hỏi cốt lõi được đặt ra trong thí nghiệm 4C là:
> **Tại sao một model mang lỗi nghiêm trọng về `flip_idx` vẫn có thể đạt điểm số mAP rất cao trên tập validation chuẩn? Metric nào đã che giấu lỗi này, và làm thế nào để thiết kế một tập validation trung thực phản ánh đúng năng lực triển khai?**

---

## 2. Kết quả Thí nghiệm 4C (Val Gốc vs. Val Lật Gương)

Thí nghiệm được thiết lập so sánh hai mô hình YOLO26n-pose (huấn luyện 40 epochs trên `tiger-pose`):
1. **Model A (Quy ước Giải phẫu):** Sử dụng `FLIP_IDX = [0, 1, 2, 3, 7, 6, 5, 4, 10, 11, 8, 9]`, trong đó các cặp chi trái/phải (`left_*` $\leftrightarrow$ `right_*`) được hoán đổi đối xứng khi lật gương.
2. **Model B (Quy ước Đồng nhất):** Sử dụng `FLIP_IDX = [0, 1, ..., 11]` (mặc định trong file cấu hình gốc), không hoán đổi vị trí của các nhãn trái/phải khi lật gương.

Cả hai model được đánh giá chéo trên hai tập kiểm thử:
- **Tập Val Gốc (`tiger-pose-anat.yaml`):** 53 ảnh, 100% con hổ quay đầu sang phải (hướng nhìn tự nhiên trong dataset).
- **Tập Val Lật Gương (`tiger-pose-mirror.yaml`):** 53 ảnh được tạo bằng cách lật ngang hình ảnh ($x \to 1 - x$) và đổi nhãn keypoint tương ứng theo quy ước giải phẫu, mô phỏng trường hợp hổ quay đầu sang trái khi triển khai thực tế.

### Bảng Kết quả Tổng hợp (Pose mAP50-95)

| Mô hình | Pose mAP50-95 (Val Gốc) | Pose mAP50-95 (Val Lật Gương) | Biến thiên mAP ($\Delta$) |
|---|:---:|:---:|:---:|
| **Model A (`flip_idx` Giải phẫu)** | **0.865** | **0.862** | **-0.003** (Ổn định tuyệt đối) |
| **Model B (`flip_idx` Đồng nhất)** | **0.868** | **0.312** | **-0.556** (Sụp đổ hoàn toàn) |

---

## 3. Phân tích: Metric nào đã che giấu lỗi `flip_idx`?

### 3.1. Sự "nói dối" của tập Validation Gốc
- Trong tập dữ liệu gốc, toàn bộ **210 ảnh train** và **53 ảnh val** đều có hổ quay đầu sang phải (tỷ lệ 100%).
- Khi Model B được train với `flip_idx` đồng nhất:
  - Khi gặp ảnh gốc (quay phải): model học đúng chân phía camera là `right_*`.
  - Khi gặp ảnh augment lật ngang (quay trái): do `flip_idx` không hoán đổi, chân phía camera (thực chất là chân trái `left_*`) lại bị gán nhãn là `right_*`. Model bị "nhiễm độc" và học một quy tắc ngầm theo góc nhìn camera: *"cứ chân nào ở gần mắt người xem thì gọi là `right_*`"*.
- Khi đem Model B chấm điểm trên **tập Val Gốc**: vì 100% hổ ở tập val đều quay phải, chân ở gần camera trùng khớp ngẫu nhiên với chân `right_*` thật sự! Kết quả là Model B đạt điểm mAP cao ngất ngưởng (**0.868**), thậm chí ngang ngửa hoặc nhỉnh hơn Model A do không phải giải quyết sự phức tạp của việc học phân biệt tư thế xoay.
- **Metric mAP chuẩn trên tập Val Gốc đã hoàn toàn che giấu lỗi sai logic nghiêm trọng này!**

### 3.2. Sự sụp đổ trên Tập Val Lật Gương
- Khi đánh giá trên tập Val Lật Gương (hổ quay trái):
  - Model A duy trì mAP **0.862** (gần như tương đương val gốc) vì model đã thực sự hiểu cấu trúc giải phẫu 3D của con hổ độc lập với hướng quay.
  - Model B tụt dốc thảm hại từ **0.868 xuống 0.312** (mất hơn 64% hiệu năng).
  - Phân tích chi tiết OKS cho thấy: Với Model B, toàn bộ 8 keypoint ở 4 chân đều bị hoán đổi chéo giữa chân trái và chân phải ($OKS \to 0$ cho các keypoint chân). 4 keypoint duy nhất còn giữ được điểm là các điểm trên trục đối xứng giữa cơ thể (`nose`, `head`, `withers`, `tail_base`).

---

## 4. Ước lượng `kpt_oks_sigmas` cho Custom Dataset

Trong bài toán mặc định, Ultralytics sử dụng $\sigma_i = 1/K = 1/12 \approx 0.0833$ cào bằng cho tất cả 12 keypoint của con hổ. Việc này che giấu sai số vì:
- **Mũi (`nose`):** Là mốc giải phẫu có độ bất định cực nhỏ ($\sigma \approx 0.025$). Nếu phạt theo $\sigma = 0.083$, model lệch 10 pixel ở mũi vẫn được OKS cao một cách giả tạo.
- **Gốc đuôi (`tail_base`):** Là vùng mô mềm lông xù, độ bất định của người gán nhãn rất cao ($\sigma \approx 0.120$). Phạt theo $\sigma = 0.083$ khiến metric quá khắt khe ở điểm này.

### Phương pháp ước lượng $\sigma$ thực nghiệm:
Dựa trên phương pháp luận chuẩn của MS-COCO:
$$\sigma_i = \sqrt{\frac{1}{N \cdot M} \sum_{n=1}^N \sum_{m=1}^M \frac{\|\mathbf{p}_{i,n}^{(m)} - \bar{\mathbf{p}}_{i,n}\|^2}{s_n^2}}$$
Trong đó:
- $M$: Số lượng annotator độc lập (khuyến nghị $\ge 3$).
- $\mathbf{p}_{i,n}^{(m)}$: Toạ độ keypoint $i$ của đối tượng $n$ do annotator $m$ gắn nhãn.
- $\bar{\mathbf{p}}_{i,n}$: Toạ độ trung bình (consensus landmark).
- $s_n^2$: Diện tích bounding box ($w \times h$) hoặc mask của đối tượng.

### Bộ trọng số đề xuất cho `tiger-pose`:
Cập nhật vào file YAML thông qua tham số `kpt_oks_sigmas`:
```yaml
# Cấu hình kpt_oks_sigmas chuẩn hoá cho 12 keypoint hổ:
# [nose, head, withers, tail_base, r_h_hock, r_h_paw, l_h_paw, l_h_hock, r_f_wrist, r_f_paw, l_f_wrist, l_f_paw]
kpt_oks_sigmas: [0.026, 0.035, 0.075, 0.110, 0.080, 0.065, 0.065, 0.080, 0.070, 0.060, 0.070, 0.060]
```
Khi áp dụng bộ $\sigma$ này, OKS phản ánh chính xác độ chính xác hình học thực tế, không còn hiện tượng "điểm cao ảo" ở vùng đầu và phạt oan ở vùng hông/đuôi.

---

## 5. Đề xuất: Thiết kế Tập Validation Đáng Tin Cậy Cho Triển Khai Thực Tế

Để ngăn chặn triệt để hiện tượng "tập val nói dối", quy trình thiết kế tập dữ liệu kiểm thử trong công nghiệp cần tuân thủ 3 nguyên tắc sau:

1. **Kiểm tra Phân phối Biến đổi Không gian (Spatial Symmetry & Orientation Audit):**
   - Tập validation không được phép chỉ phản ánh một hướng quay đơn điệu (bias hướng quay). Cần đảm bảo phân phối cân bằng giữa các hướng nhìn (hướng trái, hướng phải, trực diện, từ phía sau).
   - Nếu dữ liệu thu thập bị lệch hướng (như trường hợp camera một chiều), bắt buộc phải tạo tập kiểm thử mở rộng bằng cách lật gương nhân tạo có hoán đổi nhãn giải phẫu (như hàm `make_mirrored_val`).

2. **Chia tập theo Thực thể / Cảnh quay (Split by Identity / Sequence / Camera):**
   - Tuyệt đối không chia train/val ngẫu nhiên theo từng frame ảnh (random frame split) vì các frame liền kề trong video có tương quan thị giác quá cao.
   - Bắt buộc phải chia tập theo **cá thể** (con hổ A ở train, con hổ B ở val) hoặc theo **ngày quay / góc đặt camera khác nhau** để kiểm tra khả năng khái quát hoá ngoại cảnh (Out-of-Distribution Generalization).

3. **Báo cáo Metric Đa Chiều (Fine-grained & Stratified Evaluation):**
   - Không chỉ dựa vào một con số mAP trung bình duy nhất.
   - Bắt buộc kiểm tra chỉ số theo từng phân nhóm (Stratified Metrics):
     - mAP theo hướng quay ($mAP_{\text{left}}$ vs. $mAP_{\text{right}}$).
     - mAP theo kích thước đối tượng ($mAP_{\text{small}}$, $mAP_{\text{medium}}$, $mAP_{\text{large}}$).
     - Tỷ lệ đảo nhãn (Swapped Rate): Đo tỷ lệ các ca mà khi đảo nhãn `left_*` $\leftrightarrow$ `right_*` thì OKS lại tăng lên để phát hiện sớm lỗi `flip_idx`.
