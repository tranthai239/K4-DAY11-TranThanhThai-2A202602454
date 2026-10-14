MSSV: 202602454

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): **Town C**

---

## 1. Phát lại log có trung thực không? (bước A)

### Bảng số liệu từ `evidence/replay.json`:
* `replay.json: n = 400`
* `replay.json: dropped_human_crash_logs = 4` (số kịch bản bị loại vì chính người lái gây va chạm trong dữ liệu log ghi nhận).
* `replay.json: n_used = 396` (số kịch bản hợp lệ được đưa vào thử nghiệm so sánh).
* `replay.json: replay.crashes = 111` (planner chạy trên log phát lại).
  * Tiếp xúc: `rear = 97`, `side = 14`, `front = 0`.
* `replay.json: reactive.crashes = 0` (planner chạy trong thế giới phản ứng tương tác khép kín).
* `replay.json: crash_in_both = 0`.

### Giải thích bản chất:
* **Vì sao phần lớn va chạm của "phát lại" là `rear` (97/111 vụ = 87.4%):** Khi phát lại log tĩnh (`others = "replay"`), các xe xung quanh di chuyển cứng nhắc theo đúng quỹ đạo thời gian mà người lái gốc đã ghi nhận. Tuy nhiên, xe tự lái (planner) có xu hướng quan sát thận trọng và đạp phanh sớm hơn so với người lái người thật. Vì xe phía sau trong log không phản ứng lại sự giảm tốc này, nó tiếp tục lao về phía trước với vận tốc đã ghi trong log và tông thẳng vào đuôi xe ta (`rear`).
* **Vì sao đây là va chạm giả (phantom collision):** Trong thực tế (và trong môi trường mô phỏng phản ứng `reactive`), người lái xe phía sau có mắt nhìn và sẽ nhường đường/giảm tốc theo mô hình IDM khi thấy xe phía trước phanh. Bằng chứng là trong thế giới `reactive`, số va chạm là **0**. Va chạm trong log phát lại là hệ quả của việc phá vỡ vòng lặp phản hồi kín (open-loop playback vs closed-loop reality).
* **Minh hoạ một kịch bản trong `replay_only`:** Kịch bản `scenario: 3` (`replay.json: replay_only[0]`).
  * Trong log phát lại: xe ta bị đâm từ phía sau (`contact = "rear"`).
  * Trong thế giới phản ứng: hoàn toàn không có va chạm, khoảng thời gian chạm đuôi nhỏ nhất lên tới `min_ttc_reactive = 7.46 s` (rất an toàn). Xe bám đuôi TG trong thế giới phản ứng đã tự giảm tốc nhường xe ta.

---

## 2. Kịch bản bất khả thi (bước B)

### Bốn luật vật lý trong `is_feasible` và căn cứ:
1. **Giới hạn bám ngang khi chuyển làn:** Luật kiểm tra gia tốc ngang đỉnh $a_{lat} = (\pi / \text{cutDur})^2 \times 3.5 / 2 \le \mu g$. Căn cứ: xe chuyển làn theo quỹ đạo cosine rộng 3.5 m. Theo định luật ma sát Coulomb, lực bám ngang cực đại của lốp bị chặn bởi $F_{lat} \le \mu m g$. Nếu $a_{lat} > \mu g$, xe chắc chắn bị trượt văng lật (spin-out/drift), không thể bám đường để hoàn thành chuyển làn.
2. **Gia tốc hãm sau nhập làn:** Luật kiểm tra `cutBrake` $\le \mu g$. Căn cứ: gia tốc phanh dọc tối đa của một xe cơ giới trên mặt đường có hệ số ma sát $\mu$ không thể vượt quá giới hạn ma sát trượt của lốp $\mu g$.
3. **Giới hạn vận tốc người đi bộ:** Luật kiểm tra `pedV` $\le 2.5\text{ m/s}$ ($9\text{ km/h}$). Căn cứ sinh trắc học: $2.5\text{ m/s}$ là ngưỡng giới hạn chạy nước rút đô thị của người bình thường; người đi bộ qua đường không bao giờ duy trì vận tốc siêu nhân vượt quá con số này.
4. **Vận tốc xe chuyển làn:** Luật kiểm tra `0 < cutV <= 22.0 m/s` ($0 < v \le 79.2\text{ km/h}$). Căn cứ: trong phố đô thị, một xe không thể chuyển làn khi đứng yên/đi lùi ($\le 0$), và tốc độ đường phố không vượt quá $22\text{ m/s}$.

### Phân tích cuộc săn trong `hunt.json`:
* Cuộc săn sinh nhiều kịch bản bất khả thi nhất: **`crosswalk_block`** với **12** kịch bản bất khả thi bị loại bỏ (`hunt.json: crosswalk_block.infeasible = 12`; tiếp theo là `rear_ended` với 9).
* **Vì sao CEM bị kéo về phía đó:** Thuật toán CEM cập nhật phân phối đề xuất dựa trên điểm nguy hiểm `danger score`. Điểm nguy hiểm đạt mức cao nhất (tiến sát 0) khi khoảng cách phanh còn lại trước người đi bộ bị âm sâu (tai nạn không thể tránh khỏi). Những kịch bản bất khả thi (như mặt đường trơn trượt $\mu = 0.5$ nhưng người đi bộ chạy cực nhanh hoặc phanh cực gắt) tạo ra điểm nguy hiểm cao bất thường, khiến CEM lầm tưởng đó là vùng cần khai thác và tập trung phân phối $q$ vào vùng phi vật lý đó.
* **Hậu quả nếu đếm chúng là va chạm:** Xác suất tai nạn của planner sẽ bị thổi phồng nghiêm trọng (false positives). Kỹ sư sẽ mất công sức đi tối ưu một hệ thống để giải quyết các tình huống "ảo ảnh" vi phạm định luật vật lý cơ bản.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

### Số liệu từ `evidence/mc.json`:
* Số lần thử: `mc.json: n = 3000`
* Số vụ va chạm: `mc.json: crashes = 4`
* Tần suất mẫu: $\hat{p} = 4 / 3000 = 0.001333$ ($1.33 \times 10^{-3}$)
* Khoảng tin cậy Wilson 95%: **$[0.0005186, 0.003424]$** hay $[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$.
* Chi tiết các kiểu: `bus_bypass = 4`, `cut_in = 0`, `crosswalk_block = 0`, `rear_ended = 0`, `other = 0`.
* Kịch bản bất khả thi gặp phải: `mc.json: infeasible = 16`.

### Phân tích:
* **Cần bao nhiêu lần chạy để khoảng tin cậy hẹp còn $\pm 20\%$ của $p$:**
  * Độ nửa rộng khoảng Wilson xấp xỉ: $1.96 \times \sqrt{\frac{p(1-p)}{n}} \le 0.20 \cdot p \implies n \approx \frac{1.96^2}{0.04 \cdot p} \approx \frac{96.04}{p}$.
  * Với $p = 0.001333$:
    $$n \approx \frac{96}{0.001333} \approx \mathbf{72{,}000}\text{ lần chạy.}$$
* **Kiểu thất bại chưa hề xuất hiện trong 3 000 lần chạy:**
  * Cả bốn kiểu: `cut_in`, `crosswalk_block`, `rear_ended`, và `other` đều có $k = 0$.
* **Con số đúng cần báo cho các kiểu đó:**
  * **Tuyệt đối không báo bằng 0** (vì 0 lần gặp trong 3 000 mẫu không chứng minh xác suất bằng 0).
  * Cần báo bằng **cận trên khoảng Wilson 95%** (`mc.json: per_mode.<mode>.hi`):
    $$p \le \mathbf{0.001279}\quad (1.28 \times 10^{-3})$$
    (Hoặc theo quy tắc số ba: $3/n = 3/3000 = 1.0 \times 10^{-3}$).

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

### Bảng kết quả từ `evidence/estimate.json`:

| Kiểu thất bại | Seed | $p$ | Hits | ESS | Top Share |
|---|---|---|---|---|---|
| **bus_bypass** | Seed 0 | $1.39 \times 10^{-3}$ | 26 | 141.0 | 0.167 |
| | Seed 1 | $1.29 \times 10^{-3}$ | 27 | 143.2 | 0.275 |
| | Seed 2 | $1.77 \times 10^{-3}$ | 24 | 127.8 | 0.216 |
| **cut_in** | Seed 0 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 119.2 | 0.0 |
| | Seed 1 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 120.8 | 0.0 |
| | Seed 2 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 138.5 | 0.0 |
| **crosswalk_block** | Seed 0 | $5.45 \times 10^{-8}$ | 45 | 129.5 | 0.341 |
| | Seed 1 | $2.59 \times 10^{-7}$ | 56 | 116.1 | 0.445 |
| | Seed 2 | $2.84 \times 10^{-7}$ | 46 | 116.8 | 0.372 |
| **rear_ended** | Seed 0 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 104.1 | 0.0 |
| | Seed 1 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 122.7 | 0.0 |
| | Seed 2 | 0.0 (cận trên $2.5 \times 10^{-2}$) | 0 | 103.4 | 0.0 |
| **TỔNG (Total)** | Seed 0 | **$1.39 \times 10^{-3}$** [$5.38 \times 10^{-4}, 3.53 \times 10^{-3}$] | — | — | — |
| | Seed 1 | **$1.29 \times 10^{-3}$** [$3.93 \times 10^{-4}, 3.47 \times 10^{-3}$] | — | — | — |
| | Seed 2 | **$1.77 \times 10^{-3}$** [$5.49 \times 10^{-4}, 4.26 \times 10^{-3}$] | — | — | — |

*Phần `other` lấy từ Monte Carlo C1:* `estimate.json: other.p = 0.0`, cận trên `other.hi = 0.001279` ($n = 3000$).

### Phân tích:
* **Độ đồng thuận giữa 3 seed và với Monte Carlo:** Cả 3 seed IS ước lượng tổng xác suất va chạm rất sát nhau ($1.29 \times 10^{-3}$ đến $1.77 \times 10^{-3}$, trung vị $\approx 1.39 \times 10^{-3}$). Con số này khớp một cách xuất sắc với kết quả Monte Carlo $\hat{p} = 1.33 \times 10^{-3}$ nằm trọn trong khoảng tin cậy của cả 3 seed.
* **Phân tích `top_share` và độ suy biến:** `top_share` đo tỉ lệ đóng góp của một mẫu có trọng số $w$ lớn nhất vào tổng ước lượng. Trong kết quả trên, `top_share` cao nhất là ở kiểu `crosswalk_block` (đạt 0.445 ở Seed 1). Khi `top_share` vượt quá 0.5, ước lượng bị suy biến vì giá trị kỳ vọng không còn phản ánh trung bình tập hợp mà thực chất bị chi phối bởi trọng số cực lớn của đúng một mẫu may mắn rơi vào vùng hiếm.
* **So sánh ngân sách (`estimate.json: budget = 16800`):**
  * IS dùng tổng cộng $16{,}800$ kịch bản mô phỏng để tìm ra cả va chạm `bus_bypass` lẫn `crosswalk_block` (xác suất cỡ $10^{-7}$).
  * Để đo được biến cố $p \approx 2 \times 10^{-7}$ của `crosswalk_block` bằng Monte Carlo thông thường, ta cần $n \approx 96 / (2 \times 10^{-7}) \approx \mathbf{4.8 \times 10^8}$ lần chạy. Monte Carlo ở ngân sách $3{,}000$ hoàn toàn bất lực ($k = 0$).
* **Phần `other` lấy từ đâu và vì sao không săn nó:**
  * Lấy từ `mc.json: other` (đếm thô trên mẫu tự nhiên).
  * Không săn `other` bằng CEM vì `other` không phải là một cơ chế vật lý cụ thể để xây dựng hàm điểm nguy hiểm định hướng (`danger score`). Nếu gộp chung vào một cuộc săn, CEM sẽ bị phân mảnh hoặc chỉ co cụm vào một kiểu dễ nhất.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

### Hình ảnh và chuỗi sự kiện:

1. **Kiểu `bus_bypass` (Lách qua xe buýt đỗ):**
   * *Hình ảnh chụp từ chương 9:* `evidence/shots/ch9-lab-bus-bypass.jpg`
   * *Chuỗi sự kiện:* Xe buýt `BUS` đỗ tại làn phải ($x = 106\text{ m}$) trước vạch người đi bộ. Xe `C2` đi sau xe buýt quyết định lách sang làn trái để vượt lên với tốc độ chậm ($< 6\text{ m/s}$ tại $x < 112\text{ m}$). Xe ta (`ego`) đang chạy thẳng ở làn trái với tốc độ $12.5\text{ m/s}$. Do thân xe buýt quá cao che khuất tầm nhìn, xe ta không nhận biết được mũi xe `C2` đang nhô ra cho đến khi cự ly quá gần; planner không kịp phanh dừng êm và đâm vào thân/đầu xe `C2`.

2. **Kiểu `crosswalk_block` (Chặn lối người đi bộ trên vạch):**
   * *Hình ảnh chụp từ chương 9:* `evidence/shots/ch9-lab-crosswalk-block.jpg`
   * *Chuỗi sự kiện:* Người đi bộ `P1` bước qua đường từ mép đường phía sau xe buýt đỗ. Thân xe buýt chắn hoàn toàn đường nhìn giữa cảm biến trên nóc xe ta và người đi bộ. Khi `P1` vừa bước qua mũi xe buýt, xe ta mới phát hiện và đạp phanh khẩn cấp. Mặc dù xe ta kịp dừng lại trước khi chạm người (vận tốc về 0 m/s), nhưng thân xe đã dừng lấn đè lên vạch qua đường, chặn đứng lối đi của `P1` khiến người đi bộ phải đứng khựng lại (sự kiện `kind == "block"`).

### Phân tích:
* **Kiểu cuộc săn không tìm thấy:** Cuộc săn `cut_in` và `rear_ended` không sinh ra va chạm nào thuộc đúng kiểu của mình (trong `hunt.json`, cuộc săn `cut_in` lại tìm ra 31 vụ `bus_bypass`; cuộc săn `rear_ended` tìm ra 47 vụ `bus_bypass` và 4 vụ `crosswalk_block`). Điều này cho thấy trong thị trấn C, các kiểu va chạm `cut_in` thông thường và `rear_ended` là cực kỳ hiếm dưới hành vi mặc định của planner.
* **Nếu chỉ chạy MỘT cuộc săn chung cho mọi kiểu:** Thuật toán CEM sẽ bị **co cụm (mode collapse)** về duy nhất một kiểu dễ va chạm nhất (ở đây là `bus_bypass`). Các kiểu hiểm hóc khác nhưng có không gian tham số hẹp hơn như `crosswalk_block` sẽ hoàn toàn bị bỏ sót. Bằng chứng là trong `hunt.json`, cuộc săn `cut_in` và `rear_ended` đều bị điểm nguy hiểm kéo lệch sang sinh ra `bus_bypass`. Do đó bắt buộc phải săn riêng từng kiểu thất bại.

---

## 6. Sửa planner (bước E + 3D)

### Số liệu từ `evidence/fix.json`:
* **Policy được chọn:** `{"cutinLat": 3.3}` (`fix.json: chosen.policy`)
* **Trên tập TRAIN:**
  * Baseline: `fail = 0.004131` (4.13e-3), `fail_hi = 0.009616` (9.62e-3), `progress = 212.25 m`, `harsh = 8.10`
  * Bản sửa: `fail = 4.68e-06` (4.68e-6), `fail_hi = 1.34e-05` (1.34e-5), `progress = 212.01 m`, `harsh = 8.09`
* **Trên tập TEST (đo lường trung thực với mẫu độc lập và cuộc săn nhắm thẳng vào policy mới):**
  * Bản sửa (`chosen_test`):
    * `fail = 1.01e-06` (1.01e-6)
    * `fail_hi_seen = 1.51e-06` (1.51e-6)
    * `fail_hi (valid) = 0.03878` (3.88e-2)
    * `unseen = ["bus_bypass"]`
    * `progress = 210.89 m` (chỉ giảm 0.09% so với baseline 211.08 m, giữ vững quãng đường)
    * `harsh = 8.00`
  * Baseline (`baseline_test`):
    * `fail = 0.001755` (1.76e-3)
    * `fail_hi_seen = 0.003806` (3.81e-3)
    * `fail_hi (valid) = 0.005085` (5.08e-3)
    * `unseen = []`
    * `progress = 211.08 m`
    * `harsh = 8.98`

### Phân tích:
* **Vì sao `fail_hi` của bản sửa ($0.0388$) lại lớn hơn `fail` của baseline ($0.00176$):**
  * Trên TEST, bản sửa `cutinLat = 3.3` đã xoá sổ hoàn toàn va chạm `bus_bypass` ($hits = 0$).
  * Khi một kiểu không xuất hiện bất kỳ va chạm nào trên mẫu $n = 400$, theo nguyên lý kiểm định thống kê bảo thủ, ta không thể khẳng định xác suất bằng 0 mà phải gán cận trên hợp lệ $upper\_zero\_hits = 3 / (0.2 \cdot n) = 15 / 400 = \mathbf{0.0375}$.
  * Cận trên thống kê này ($3.75\%$) phản ánh giới hạn của ngân sách kiểm thử $n = 400$, chứ không phải bản sửa kém an toàn hơn. Muốn chứng minh cận trên hạ xuống $< 10^{-3}$, cần ngân sách kiểm thử độc lập lên tới 15 000 lần chạy tại đúng vùng đó.
* **Núm chỉnh "nghe an toàn" nhưng phản tác dụng:**
  * Giảm `cautionV = 5.0` (giảm tốc độ bò thận trọng gần xe buýt từ 7.5 xuống 5.0 m/s).
  * Trong bảng TRAIN, ứng viên này làm quãng đường sụt giảm nghiêm trọng xuống còn `progress = 170.5 m` (mất gần 20% tiến trình), đồng thời số lần phanh gấp tăng vọt lên `harsh = 12.51`.
  * *Giải thích bằng phản ứng của thế giới:* Xe ta giảm tốc bò quá chậm giữa đường biến thành chướng ngại vật di động; xe bám đuôi TG phía sau bị dồn ép và phải phanh gắt hơn, làm gia tăng nguy cơ bị đâm va từ phía sau hoặc kích động các xe làn bên tạt đầu liều lĩnh hơn.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định phân phối tự nhiên trong `space.py` ảnh hưởng nhiều nhất đến con số ước lượng:

1. **Khe chấp nhận nhập làn (`cutGap`):**
   * *Dữ liệu thật thay thế:* Trích xuất phân phối time gap / distance gap acceptance từ các tập dữ liệu quỹ đạo giao thông đô thị thực tế như HighD, NGSIM, Waymo Open Dataset.
   * *Hướng sai lệch:* Nếu mô phỏng giả định tài xế chỉ tạt đầu khi khe hở lớn hơn thực tế, nguy cơ va chạm `cut_in` và `bus_bypass` trong mô phỏng sẽ bị ước lượng **thấp hơn sự thật** ngoài đời.
2. **Tỉ lệ người đi bộ ngó nghiêng qua xe buýt (`pedLook`):**
   * *Dữ liệu thật thay thế:* Dữ liệu thị giác máy tính quan sát hành vi người đi bộ ở góc khuất qua các tập dữ liệu JAAD (Joint Attention for Autonomous Driving) và PIE (Pedestrian Intent Estimation).
   * *Hướng sai lệch:* Nếu đời thực người đi bộ thường cẩn thận thò đầu ngó trước khi bước qua xe buýt nhiều hơn trong sim (`pedLook` thực tế cao hơn), con số tai nạn `crosswalk_block` trong mô phỏng đang bị thổi phồng **cao hơn sự thật**.
3. **Phân phối hệ số ma sát mặt đường (`mu`):**
   * *Dữ liệu thật thay thế:* Thống kê khí tượng thủy văn theo mùa (tỉ lệ ngày mưa, độ ẩm mặt đường) kết hợp cảm biến đo lực bám lốp thực tế trên đường nhựa đô thị Việt Nam.
   * *Hướng sai lệch:* Nếu mô phỏng giả định tỉ lệ đường ướt ($\mu = 0.5$) thường xuyên hơn đời thực, quãng đường phanh cần thiết bị kéo dài, làm số vụ thất bại `crosswalk_block` bị ước lượng **cao hơn thực tế**.

---

## 8. Giới hạn

Ba điều mà xác suất báo cáo **KHÔNG** nói lên:
1. **Hình thái đường phố bị đơn giản hoá:** Mô phỏng chỉ có một đoạn đường thẳng với 2 làn mỗi chiều và 1 xe buýt đỗ. Xác suất này không phản ánh được nguy cơ ở các ngã tư có đèn tín hiệu, bùng binh, đường dốc, góc cua khuất hoặc đường có giải phân cách cứng.
2. **Các tác nhân xung quanh chỉ tuân theo mô hình IDM:** Con người ngoài đời có phản xạ tránh né linh hoạt, có thể bấm còi, nháy pha hoặc vẫy tay; người đi bộ có thể đổi hướng hoặc nhảy lùi lại khi thấy nguy hiểm, không chỉ đơn thuần dừng bước cứng nhắc như mô hình.
3. **Mô hình cảm biến lý tưởng hoá:** Sim giả định cảm biến nhận thức hoàn hảo 100% mọi đối tượng không bị xe buýt che, không có độ trễ nhận thức (perception delay), không có nhiễu theo vết (tracking jitter) hay điểm mù gương chiếu hậu.

*Nếu có thêm một tuần:* Sẽ bổ sung mô hình **độ trễ nhận thức ngẫu nhiên (100–300 ms)** và **nhiễu định vị bounding box**, đồng thời mở rộng mô phỏng sang kịch bản ngã tư có xe máy lạng lách từ góc khuất.

---

## 9. Rắc rối của bạn (bước G + chương 10)

* **Rắc rối:** Tình huống **Người đi bộ băng ngang đường tự phát ngoài vạch qua đường (Jaywalking Pedestrian - H0)** ngay sau trạm xe buýt ($x = 115\text{–}145\text{ m}$). Ngoài đời, đây là tình huống kinh điển khi hành khách vừa bước xuống xe buýt liền vội vàng băng cắt ngang đường ngay sau đuôi xe buýt để sang đường đối diện.
* **Căn cứ:** Planner của xe ta chỉ kích hoạt logic thận trọng (`cautionRange`) quanh vạch người đi bộ chính thức ($x = 122\text{ m}$). Khi xe vừa vượt qua vạch qua đường, planner bắt đầu tăng tốc trở lại và chỉ coi một vật thể là chướng ngại vật khi đối tượng đã lọt vào hành lang hẹp của làn đường (`cutinLat`). Do đó, xe không hề phòng ngừa trước, dẫn đến nguy cơ va chạm rất cao.
* **Giả định (`hazard.json`):**
  * `space`: `h0_x: [115.0, 145.0]`, `h0_v: [1.1, 1.8]`, `h0_trig: [12.0, 28.0]`. Vận tốc người đi bộ từ 1.1 đến 1.8 m/s phù hợp với sinh trắc học người trưởng thành; cự ly kích hoạt 12–28 m là khoảng cách quan sát thấy khi người vừa ló ra từ làn bên cạnh.
  * `exposure = 0.03`: Theo báo cáo an toàn người đi bộ đô thị NHTSA, trung bình cứ khoảng 33 chuyến đi trong phố (3%) sẽ gặp 1 tình huống người đi bộ sang đường tự phát bất ngờ.
* **Kết quả theo `evidence/mine.json`:**
  * Tỉ lệ khả thi chấp nhận: `mine.json: accepted = 0.9913` (**99.1%** $\ge 50\%$).
  * Số va chạm ở **Policy gốc (Baseline)**:
    * Tổng số va chạm: 8 (trên 1487 lượt thử).
    * `avoidable` (tránh được bằng phanh hết cỡ khi thấy): **4**
    * `unavoidable`: **0**
    * `unattributed` (va chạm thường của thị trấn, không do H0): **4**
    * `low_speed`: **0**
    * $P(\text{lỗi planner} \mid \text{hazard}) = \mathbf{2.69 \times 10^{-3}}$ (Wilson 95%: $[1.05 \times 10^{-3}, 6.90 \times 10^{-3}]$).
    * Tần suất mỗi chuyến: $8.07 \times 10^{-5}$ lần/chuyến (cận trên $2.07 \times 10^{-4}$).
  * Số va chạm ở **Policy sửa (Chosen `cutinLat = 3.3`)**:
    * Tổng số va chạm: 5
    * `avoidable`: **5**
    * `unavoidable`: **0**
    * `unattributed`: **0**
    * `low_speed`: **0**
    * $P(\text{lỗi planner} \mid \text{hazard}) = \mathbf{3.36 \times 10^{-3}}$ (Wilson 95%: $[1.44 \times 10^{-3}, 7.85 \times 10^{-3}]$).
    * Tần suất mỗi chuyến: $1.01 \times 10^{-4}$ lần/chuyến (cận trên $2.35 \times 10^{-4}$).
  * *Nhận xét quan trọng:* Cả 4–5 vụ va chạm đều thuộc loại `avoidable` và bản sửa ở bước E (`cutinLat = 3.3`) **không hề cứu được** rắc rối này (thậm chí tần suất $P$ còn tăng nhẹ lên $3.36 \times 10^{-3}$ vì mở rộng ngưỡng nhận diện xe cắt làn không giúp ích cho người đi bộ băng ngang).
* **Hình ảnh minh họa:** `evidence/shots/ch10-studio.jpg` (chương 10 xưởng kịch bản mô phỏng đối tượng rắc rối băng cắt).
* **Tự lái thì sao:** Người lái thật tránh được dễ dàng vì con người có nhận thức ngữ cảnh (contextual awareness): khi vừa đi qua một chiếc xe buýt đang đỗ trả khách, người lái sẽ chủ động rà chân phanh và liếc nhìn mép đường để dự phòng có người bất ngờ lao ra, thay vì giữ ga tăng tốc như thuật toán IDM của planner.
* **"Tránh được là chuẩn lạc quan":**
  * *Theo chiều đếm nhiều hơn (lạc quan quá mức):* Thuật toán oracle phanh hết cỡ ngay từ khung hình đầu tiên nhìn thấy mục tiêu với gia tốc cực đại $\mu g$. Một planner thực tế không thể lúc nào cũng phanh giật kịch sàn như vậy vì sẽ gây nguy hiểm bị đâm đuôi và mất tiện nghi cho hành khách. Do đó số vụ `avoidable` đếm nhiều hơn số vụ mà một planner thực tế có thể chấp nhận phanh.
  * *Theo chiều đếm ít đi (bảo thủ):* Oracle chỉ thử nghiệm duy nhất hành vi đạp phanh, hoàn toàn không xem xét phương án đánh lái né tránh (steering/swerving). Trong nhiều tình huống, đánh lái nhẹ sang khoảng trống có thể tránh được tai nạn mà không cần phanh gấp.
* **Cách sửa planner:**
  * Không núm chỉnh IDM nào trong 9 núm hiện tại có thể giải quyết triệt để rắc rối này mà không làm tê liệt vận tốc xe.
  * Planner cần bổ sung **logic nhận diện ý đồ người đi bộ (Pedestrian Crossing Intention Logic)**: Khi phát hiện bất kỳ người đi bộ nào đang di chuyển với vector vận tốc hướng vào lòng đường ($v_z > 0.5\text{ m/s}$) trong phạm vi 30 m phía trước, planner phải lập tức áp dụng giảm tốc phòng ngừa (deceleration profile) ngay từ khi người đó còn ở ngoài lề đường, thay vì chờ đến khi đối tượng đã bước vào tim làn xe chạy mới phanh.
