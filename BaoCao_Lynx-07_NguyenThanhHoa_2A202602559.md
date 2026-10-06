# Báo cáo nhiệm vụ Lynx-07 — Chẩn đoán cảm biến xe tự hành

**Kỹ sư:** 2A202602559 (Nguyễn Thanh Hòa)
**Notebook:** `kalman_fusion_lab_2A202602559.ipynb`
**Kết luận ngắn:** GPS bị lệch hằng số (bias). Đã sửa bằng cách trừ residual trung bình khỏi phép đo GPS. Pooled mean NIS sau sửa = 2.86 (ngưỡng < 8).

---

## 1. Bằng chứng

Kết quả `diagnose(mission)` (bộ lọc tạm tin vào SPEC của cả hai cảm biến):

| Cảm biến | n | mean(NIS) | median(NIS) | Residual trung bình `[x, y]` |
|:--|--:|--:|--:|:--|
| GPS | 450 | 3.68 | 2.44 | `[-0.59, -0.28]` m |
| UWB | 900 | 3.73 | 2.75 | `[1.14, 0.59]` m |

Giá trị kỳ vọng của NIS 2 chiều khi bộ lọc nhất quán: mean ≈ 2, median ≈ 1.39.

**Loại trừ hai kiểu lỗi còn lại**
- **Không phải `outlier_burst`:** median cũng cao theo mean (2.4–2.75, không về ≈ 1.4). Cả phân bố bị dịch lên, không phải vài điểm đột biến kéo mean.
- **Không phải `underrated_noise`:** mean chỉ cao khoảng 1.8 lần mức kỳ vọng. Nhiễu bị đánh giá thấp 2.5–4.5 lần sẽ cho NIS cao gấp 6–20 lần.

**Nhận dạng `bias`**
- Residual trung bình của cả hai cảm biến lệch khỏi `[0, 0]` theo hướng cố định và ngược dấu nhau (GPS ≈ −0.65 m, UWB ≈ +1.28 m).
- Bộ lọc kéo ước lượng về giữa hai cảm biến, nên sai lệch tương đối giữa chúng (~1.9 m) bị chia ra hai phía.

**Xác định cảm biến nào bị lệch**
- Riêng residual không trả lời được câu hỏi này. Bias trên GPS hoặc bias trên UWB cho ra số liệu có cùng cấu trúc.
- Cần một mốc tham chiếu bên ngoài. Tôi nêu rõ đây là một **giả định**, không phải dữ kiện đề cho: xe xuất phát tại gốc `(0, 0)` ở t = 0.
- Căn cứ của giả định:
  1. Bộ lọc Phần 9 khởi tạo với `x0 = [0, 0, 0, 0]`.
  2. Hàm sinh quỹ đạo `_true_mission_path` luôn bắt đầu từ waypoint `[0, 0]`.
- Hồi quy tuyến tính 8 giây đầu rồi ngoại suy về t = 0:

  | Cảm biến | Vị trí ngoại suy tại t = 0 | Lệch so với gốc |
  |:--|:--|:--|
  | GPS | `(-1.56, -0.82)` m | ~1.8 m |
  | UWB | `(0.16, -0.07)` m | ≈ 0 (trong nhiễu) |

- Kết luận: **GPS bị bias**, UWB bình thường. Phép kiểm tra này nằm ở ô code "9.1b".
- Nếu giả định xuất phát tại gốc sai, kết luận về loại lỗi (`bias`) vẫn đúng nhưng cảm biến nào lệch thì không còn chắc chắn.

**Chẩn đoán đã khai báo:** `MY_DIAGNOSIS_SENSOR = "GPS"`, `MY_DIAGNOSIS_TYPE = "bias"`.

---

## 2. Cách sửa

Chọn **`bias` trên GPS** (`FIX_SENSOR = "GPS"`, `FIX_METHOD = "bias"`): trừ residual trung bình khỏi các phép đo GPS trước mỗi bước update.

Kết quả: pooled mean NIS = **2.86** (median 1.87) trên 1350 phép đo được chấp nhận. Mức này giảm từ khoảng 3.7, thấp hơn hẳn ngưỡng 8 và gần giá trị kỳ vọng ≈ 2.

**Vì sao hai cách còn lại không khớp**
- **`inflate_R`:** cách này dành cho trường hợp nhiễu lớn hơn spec, khi mean và median của NIS đều ≫ 2. Ở đây NIS chỉ cao vừa phải. Phình R chỉ che dấu độ lệch bằng cách làm bộ lọc bớt tin GPS, không loại bỏ được sai số hệ thống.
- **`gate`:** cách này dành cho vài phép đo đột biến (mean NIS rất cao, median bình thường). Ở đây độ lệch xảy ra ở mọi phép đo GPS, nên cổng χ² không có gì để chặn. Hạ ngưỡng gate chỉ vứt bỏ dữ liệu tốt.

---

## 3. Độ tin cậy cuối (1σ)

Từ ma trận `P` ở dòng cuối bảng điều khiển:

- Pxx = Pyy = 0.0788, tức σ ≈ 0.28 m mỗi trục.
- **1σ vị trí ≈ 0.40 m** (√(Pxx + Pyy) = √0.158).
- **Vùng tin cậy 95%:** bán kính khoảng **0.69 m** (√(χ²₀.₉₅(2) · Pxx), tức ≈ 2.45σ mỗi trục).

---

## 4. Hạn chế

Cách sửa `bias` ước lượng độ lệch từ residual trung bình. Residual chỉ đo được sai lệch tương đối giữa hai cảm biến và bị chia theo trọng số. Cụ thể, residual GPS là −0.59 m trong khi bias thực khoảng −1.8 m, nên cách sửa chỉ bù được khoảng 1/3. Vị trí tuyệt đối vì thế còn lệch dư vài chục cm, và NIS (2.86) chưa về hẳn 2.

Cách sửa này sẽ thất bại hoặc không đủ trong các tình huống:
- Cả hai cảm biến cùng lệch theo một hướng (sai số chung).
- Bias thay đổi theo thời gian (drift).
- Không có mốc tham chiếu độc lập để biết cảm biến nào lệch. Ở bài này mốc đó chỉ là giả định xuất phát tại gốc.

Hướng khắc phục: đưa bias vào vector trạng thái để bộ lọc tự theo dõi, hoặc hiệu chuẩn lại với một tham chiếu độc lập.
