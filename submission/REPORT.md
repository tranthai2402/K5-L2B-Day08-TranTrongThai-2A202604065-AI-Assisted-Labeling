# Báo cáo Day 08 — 2A202604065 - Trần Trọng Thái

> Một file duy nhất cho cả bài (đã gộp recipe, QC và ghi chú cá nhân). Mọi con số kèm **mẫu số** và nguồn (file nào). Viết bằng lời của bạn.

## 1. Model và recipe (Part 2–3)

| Lớp | Model + cỡ | Ngưỡng | Vì sao (dẫn `run_id` trong `model_log.csv`) |
|---|---|---|---|
| person / bicycle / motorbike / car / bus / truck | YOLO26-m | 0.25 làm nền (r02); ngưỡng riêng theo lớp từ `qc/threshold_sweep.md`: `bicycle=0.2`, `motorbike=0.15`, `three_wheeler=0.15`; `person`, `car`, `bus`, `truck` dùng mặc định 0.3 | r02: 0.25 cho 215 box / 8 ảnh, 0.79 s/ảnh, box xe và người ngồi tách riêng. r03: 0.5 giảm 35% box nhưng mất xe thật (img_011 motorbike score 0.467, img_016 car score 0.258) nên không dùng một ngưỡng chung. r09: 0.05 chỉ để sweep vì sinh nhiều box rác (banana ×104, umbrella ×57). |
| three_wheeler | YOLOE-s, prompt: `three_wheeler` | conf ≥ 0.25 | r06: tìm được tuk-tuk / ba gác ở img_002 (4 xe), img_032 (3 xe) và xe mép trái img_011 mà YOLO26 gọi `truck`. r07 (prompt "three-wheeled vehicle"): nhạy hơn nhưng nhầm ô tô thành xe ba bánh (img_011 score 0.303, img_016 score 0.391). r08 (prompt "tuk-tuk"): hụt ba gác (img_032 chỉ còn 1 / 3 xe). r10: ngưỡng 0.05 sinh box lặp (img_003 có 12 box) nên chỉ dùng conf ≥ 0.25. |

- Lệnh `merge_prelabels.py` đã dùng (dán nguyên): `python scripts/merge_prelabels.py work/prelabel/yoloe_all/pre.coco.json work/prelabel/y26_all/pre.coco.json --iou 0.6 --conf 0.3 --thr "person=0.4,three_wheeler=0.25" --split "batch,gold" --out work/prelabel/merged_review/pre.coco.json` → gộp 2 file ra 547 box / 28 ảnh (YOLOE `three_wheeler` ưu tiên cao nên loại được car ×8, truck ×6, bus ×2, bicycle ×1 của YOLO26 khi chồng). Lưu ý: `--iou 0.6` **không** loại được các box trùng mà QC phát hiện (IoU 0.36–0.46), nên lỗi `thua` phải xử lý ở `qc_fixed.json` chứ không hy vọng tham số này.
- **Phiên bản SAM bạn chọn**: SAM 2.1 Base (`sam2.1_b.pt`). Chất lượng mask: r04 cho 16/16 box có mask, biên mượt; MobileSAM (r05) mask lỏng hơn, box [333,153,112,203] của img_013 chỉ 3.4k px so với 9.5k px của SAM 2.1 (IoU 0.21). Tốc độ: chậm hơn (5.05 vs 2.65 s/ảnh trên CPU; khi thêm click là ~7.3 s/ảnh) nhưng chấp nhận được. RAM / quyền truy cập: chạy offline bằng `scripts/sam2_points.py`, **không cần GPU** (chạy CPU, r04-r05, r11-r13), không cần tải quyền riêng vì checkpoint `sam2.1_b.pt` nằm sẵn trong repo; mỗi lần chạy với 3-5 click mất ~7.3 s/ảnh nên 3 ca Part 5 tốn khoảng 22 s máy.
- Lỗi **không sửa được bằng đổi ngưỡng**: tuk-tuk bị gọi `truck` (img_011, mép trái). YOLO26 không có lớp `three_wheeler` nên ngưỡng nào cũng sai lớp. Cách sửa: chạy YOLOE riêng cho `three_wheeler` (r06), duyệt tay vì score thấp (box img_011 [0,0,138,680] chỉ 0.283, xe sát mép ảnh), và xoá box `truck` / `car` trùng khi duyệt. Prompt rộng hơn không giải quyết được: r07 chuyển lỗi sang nhầm `car`, r08 hụt ba gác.
- Box **điểm cao mà sai**: img_016 box [455,482,113,84] (YOLOE score 0.391, thực chất là ô tô), img_011 box [527,223,260,205] (score 0.303, ô tô). Box **điểm thấp mà đúng**: img_011 motorbike x=971 (0.467), img_016 car x=585 (0.258), img_011 xe ba bánh mép trái (0.283).
- Giới hạn: ngưỡng chọn trên chính 4 ảnh gold nên số đo máy trên gold là **lạc quan** (in-sample). Chỉ có 62 đối tượng gold nên khoảng tin cậy rộng, và các đối tượng cùng ảnh không độc lập nên khoảng thật còn rộng hơn.

## 2. AI error profile (từ `qc/review_stats.md`)

| Trạng thái | Số | Tỉ lệ | Mẫu số |
|---|---|---|---|
| accepted | 582 | 91.1% | 639 box máy |
| edited | 15 | 2.3% | 639 box máy |
| deleted | 37 | 5.8% | 639 box máy |
| added | 57 | 8.7% | 657 đối tượng sau duyệt |

- Lớp nào máy tệ nhất? Vì sao (dẫn ảnh)?
  - **three_wheeler** tệ nhất về box sai: xoá 9 / 41 box máy = 22.0%, accepted chỉ 31/41 = 75.6% (các lớp lớn đều ≥ 90%). Nguyên nhân: score thấp, box lặp / trùng (img_003, img_002, img_023 ở r10) và prompt nhầm sang ô tô (img_011, img_016).
  - **car** tệ nhất về bỏ sót: vẽ thêm 18, tức 18/61 = 29.5% xe sau duyệt là do người vẽ, chưa phải do máy. Ví dụ ảnh: img_016 (car score 0.258 bị mất khi nâng ngưỡng). Một phần có thể là đổi lớp từ `three_wheeler`.
  - bus (3 box máy) và truck (5 box máy) quá ít mẫu để kết luận, dù truck bị xoá 2/5 (img_011 có truck sai là tuk-tuk).
  - Số vật máy sót thật ≈ 57 − 8 (đổi lớp) = 49.
- So với gold: precision / recall của **máy** 75.7% (KTC 95%: 65–84%) / 85.5% (KTC 95%: 75–92%) (mẫu số: 62 đối tượng gold; TP 53, FP 17, FN 9). Số này thấp hơn nhiều so với 91.1% accepted ở trên vì khác tập ảnh (4 ảnh gold vs 24 ảnh duyệt) và khác tiêu chí, nên không gộp hai con số làm một.

## 3. Thiên kiến tự động hóa và ảnh bẫy

- Bắt bẫy: **11/12 = 91.7%** (KTC 95%: 65-99%, nguồn `qc/review_stats.md`), vượt ngưỡng 80%. Theo loại: `extra` 3/3, `missing` 2/3, `wrong_class` 3/3, `loose` 3/3. **Lọt 1 bẫy**: `img_024.jpg` (loại `missing`) - ảnh ban đêm, đám xe máy chồng nhau nên tôi đã coi cả cụm là không tách được và không vào từng chiếc, nên không thấy vật bị thiếu. Bước bị bỏ: ở ảnh tối / máy chồng, tôi dừng ở mức "quét để gán" mà không quét lại theo lưới để đối chiếu với pre-label.
- Accepted cao mà bắt bẫy thấp (hoặc ngược lại) nói gì về cách bạn duyệt? Ở đây **accepted cao (582/639 = 91.1%) đi kèm bắt bẫy cũng cao (11/12 = 91.7%)**, nên chưa thấy dấu hiệu "duyệt hờ" (accepted cao nhưng bẫy lọt nhiều). Tuy nhiên con số accepted cao **không** chứng minh box đúng: 91.1% chỉ là "không sửa", còn lỗi mà người duyệt **không nhận ra** (đặc biệt box trùng / lồng, loại lỗi `thua` chiếm 6/7 lỗi ở mục 7) không nằm trong mẫu số này. Nói cách khác, bẫy kiểm được **thiên lệch về phía có lỗi**, còn `accepted` đo cả những lỗi mình không thấy - hai con số này không đối chiếu trực tiếp với nhau.

## 4. Máy có đáng dùng không?

- Thời gian (từ `time_log.csv`): **chưa đo** - `submission/time_log.csv` hiện chỉ có dòng header, không có số phút nào, nên tôi không dẫn được thời gian gán tay vs duyệt. (Số duy nhất đo được là thời gian chạy máy: YOLO26m 0.56 s/ảnh (r09), YOLOE-s 0.29 s/ảnh (r10), SAM 2.1-B 5.05 s/ảnh (r04), MobileSAM 2.65 s/ảnh (r05), SAM + click 7.3 s/ảnh (r11-r13) - đây là thời gian máy, không phải thời gian tôi ngồi sửa.) Thiên lệch: tôi gán gold trước nên đã quen cảnh 4 ảnh đó, và các ảnh này cũng là ảnh dùng để chọn ngưỡng nên số đo trên gold lạc quan.
- Kết luận: **có, nhưng chỉ làm pre-label để sửa, chưa dùng để xuất thẳng** - dựa trên accepted 582/639 = 91.1% (ít phải sửa), bắt bẫy 11/12 = 91.7% (lỗi máy tạo ra phần lớn được phát hiện), nhưng bù lại máy còn bỏ sót 57/657 = 8.7% đối tượng (vật máy sót thật ≈ 49 vì 8 trường hợp là đổi lớp) và precision trên gold chỉ 75.7% (KTC 65-84%). Nói cách khác: máy tiết kiệm thời gian ở phần **hình học đúng** nhưng vẫn cần người cho phần **lớp** và phần **vật bị bỏ sót**. Riêng `three_wheeler` tệ nhất: xoá 9/41 = 22.0% box máy, accepted chỉ 31/41 = 75.6%.
- **Ba quy tắc giảm thiên lệch của bạn** (mỗi quy tắc kiểm được bằng số hoặc việc làm):
  1. **Quét ảnh trước khi mở lớp nhãn máy**: tôi xác định vị trí vật từ ảnh gốc rồi mới mở pre-label để so. Nếu không, tỉ lệ accepted lên 582/639 = 91.1% nhưng sẽ là "nhìn cùng một thứ", và bẫy `missing` đã lọt 1/3 (lọt `img_024.jpg`) là bằng chứng bước này chưa đủ ở ảnh tối.
  2. **Mỗi ca `can_review` phải có dòng trong `calibration_decisions.csv`**: `can_review` là tín hiệu "chưa chắc quy tắc", nên tôi ghi lại thay vì tự quyết. Hiện có 3 dòng (CAL-01..CAL-03) cho các ca đã chốt.
  3. **Không dùng `conf` để quyết định giữ / sửa, chỉ dùng để xếp thứ tự duyệt**: vì ở `img_016` có box score 0.391 mà thực chất là ô tô, còn ở `img_016` có xe score 0.258 mà thật - `conf` không phân biệt được đúng / sai. Quy tắc này khớp với câu 7 đã chốt trong v1.1 (conf cao → kiểm mẫu 10%, conf thấp → sửa toàn bộ).

## 5. Mask SAM cho 3 vật (Part 5)

| Ảnh | SAM (phiên bản) | Số click | Mask gồm người lái / hàng hoá? | Lỗi quan sát | File trong `masks/` |
|---|---|---|---|---|---|
| `img_013.jpg` (cụm tuk-tuk giữa chợ, nhiều người chung xe) | `sam2.1_b.pt` (SAM 2.1 Base) | 3 (2 `--pos` + 1 `--neg`) | **CÓ, nặng**: mask nuốt cả người lái lẫn hành khách — `person` id=148 bị 42% diện tích nằm trong mask, `person` id=156 bị 45% | 1 điểm âm (720,300) chỉ khoét được lưng áo người lái, chưa tách được đầu/tay/chân; mask tràn **45.4% ảnh** (317 255 px), 1 polygon duy nhất ôm trọn cả cụm xe + đám người đứng sau, không tách được từng chiếc xe dù guideline 4.2.4 đòi mỗi vật một polygon | `masks/tuktuk_img013.jpg` |
| `img_004.jpg` (tuk-tuk lớn bên phải, có người lái ở mép trái) | `sam2.1_b.pt` (SAM 2.1 Base) | 5 (3 `--pos` + 2 `--neg`) | **CÓ, một phần**: `person` id=34 (người lái) còn ~21% nằm trong mask; người xa id=33 **không** dính (0%) | Mask chỉ phủ ~38% diện tích box xe (bbox mask 478–832 × 112–544 so với box 474–835 × 229–597): bám đúng thân xe nhưng còn dính nền phía trên và bỏ sót thân dưới; 2 điểm âm chưa khoét hết tay/chân người lái. Tỷ lệ 1 polygon/1 mask là tốt nhất trong 3 ca | `masks/tuktuk_img004.jpg` |
| `img_010.jpg` (xích lô/xe ba bánh nhỏ, góc phải) | `sam2.1_b.pt` (SAM 2.1 Base) | 5 (3 `--pos` + 2 `--neg`) | **KHÔNG**: các box `person` (id=134/136/139/140) chỉ bị 0–11% lọt vào mask → tách người lái thành công | Mask rò sang **mảnh mái xe khác** (8 680 px, x 74–372, y 198–258) ngoài vật đích; lỗ thủng ở khung xích lô rỗng bị đóng lại thay vì giữ lỗ theo 4.2.2; mask chỉ 2.5% ảnh nên phủ ~35% box xe | `masks/tuktuk_img010.jpg` |

- **Phiên bản SAM đã dùng**: SAM 2.1 Base (`sam2.1_b.pt`) cho cả 3 ca. Lý do (đối chiếu `model_log.csv` r04/r05 ở Part 2B): SAM 2.1-B bám viền nan hoa/bánh xe/mái vòm tốt hơn, còn MobileSAM (~2× nhanh hơn: 2.65 vs 5.05 s/ảnh trên CPU) cho mask lệch (IoU 0.21 ở box `[333,153,112,203]` của `img_013`) — với bài toán cần tách người lái khỏi xe thì độ trôi đó là không chấp nhận được, nên chọn SAM 2.1-B.
- **Số click thực tế**: 3 click cho 1 vật (với 1 điểm âm), nhưng **vật càng sáng/thì càng cần nhiều click**: `img_013` chỉ 3 click (2 pos + 1 neg) → mask tràn 45% ảnh; hai ca còn lại phải dùng 5 click (3 pos + 2 neg) mới tách được người lái ra. 1 điểm âm không đủ — phải đặt thêm lên đầu/tay/chân người lái.
- **Đối chiếu guideline mục 4.2**:
  - *4.2.1 (polygon bám biên ≤ ~3 px)*: **giúp** — SAM cho biên mịn, sạch, không thô như vẽ tay; nhưng *độ ổn định* thì không: cùng 1 vật, chỉ khác số điểm âm là mask đổi từ 45% → 2.5% ảnh.
  - *4.2.2 (lỗ thủng: lỗ < 5% diện tích thì không vẽ, lỗ lớn thì bỏ ra khỏi polygon)*: **thiếu** — guideline không nói rõ với vật rỗng như xích lô/khung xe ba bánh thì lỗ bao nhiêu là "lỗ", và SAM có xu hướng **đóng lỗ** (không tự để lỗ). Cần quy tắc bổ sung cho `three_wheeler` rỗng.
  - *4.2.3 (bóng đổ không thuộc vật)*: **thiếu quy tắc vận hành** — SAM không có khái niệm "bóng", phải tự đặt `--neg` lên vùng bóng; ở `img_004` nền phía trên xe vẫn còn dính vì không có điểm âm nào ở đó.
  - *4.2.4 (hai vật chạm nhau: mỗi vật một polygon, không chồng nhau)*: **thiếu** — khi các tuk-tuk chạm nhau, 1 click của SAM trả về **một mask gộp cả cụm** (`img_013`), phải dùng nhiều `--neg` để tách từng chiếc; offline tốn nhiều click hơn hẳn so với Interactor trong CVAT (nơi có thể chỉnh từng đường viền bằng tay).
  - *4.2.2/4.2.4 chưa đủ cho `three_wheeler` có người lái*: guideline mục 5 nói "xích lô: 1 `three_wheeler` + mỗi người 1 `person`" nhưng mục 4.2 không nói **mask của xe có được chứa người lái không**. Thực tế Part 5 của tôi: mask nên **không** chứa người lái (tách bằng `--neg`), vì người là vật riêng → đề xuất bổ sung vào `guideline_v1.1_calibrated.md`.
- *(Tuỳ chọn)* **Interactor SAM trong CVAT**: **không làm** — chạy offline bằng `scripts/sam2_points.py` đã đáp ứng đủ 3 ảnh và tôi dùng được số đo (diện tích mask, tỷ lệ tràn vào box `person`) để chấm. Nếu làm Interactor trong CVAT thì khác ở chỗ: **được sửa từng đường viền bằng tay** (thêm/bớt cung, thêm polygon lỗ theo 4.2.2) và xem luôn lớp `person` đang đè lên để tránh chồng — tức là 1 ca mà SAM cho mask sai sẽ sửa được bằng vài giây, thay vì phải thêm điểm `--neg` như offline; đổi lại Interactor dùng chung server CVAT nên chậm hơn và phụ thuộc GPU của container.

## 6. Calibration (Part 6)

- Số quyết định: **3 dòng** trong `calibration_decisions.csv` (CAL-01..CAL-03), cộng **7 ca khó** trong `guideline_v1.1_calibrated.md` mục 2 (mỗi ca một ảnh ví dụ, 2 ca có `case_id` do bài giao: `img_025` = 434, `img_026` = 470). Bảy câu hỏi mở ở mục 6 của v1.0 đã chốt đủ 7 trong v1.1. Quan trọng nhất:
  - `img_025` (case_id 434): xe đạp điện / xe máy điện phân loại theo hình dáng — có bàn đạp → `bicycle`, có yên + sàn để chân và không bàn đạp → `motorbike`.
  - `img_026` (case_id 470): xe máy đỗ dày — gán từng xe nếu thấy ≥ 30–40% thân xe, phần còn lại gộp một `ignore_region`.
  - `img_006`: người trong xe ba bánh có mui chỉ gán khi thấy rõ đầu hoặc vai, bỏ mốc "20%"; độ lệch giữa hai người gán đo bằng pilot 50 ca, 2 người độc lập (chưa chạy).
- Quy tắc mới đã vào `guideline_v1.1_calibrated.md`:
  1. Xe điện: có bàn đạp → `bicycle`; có yên + sàn để chân, không bàn đạp → `motorbike`; không phân biệt được thì tick `can_review`.
  2. Xe đỗ dày: ≥ 30–40% thân xe thì gán từng xe, còn lại gộp một `ignore_region`.
  3. Box/mask chỉ ôm xe + người; hàng nhô xa không tính, hàng sát thân thì tính.
  4. `person` trong xe có mui: chỉ gán khi thấy rõ đầu hoặc vai.
  5. Kính xe thuộc mask xe; người thấy qua kính vẫn gán `person` riêng.
  6. Box chồng nhau ban đêm: tách được thì giữ riêng, không thì `ignore_region`.
  7. `conf` chỉ để phân luồng duyệt (cao → kiểm mẫu 10%, thấp → sửa toàn bộ), không dùng để đổi nhãn.
  - Đề xuất từ thử nghiệm SAM (**chưa** đưa vào v1.1, vẫn là đề xuất): mask xe không chứa người lái; quy tắc lỗ cho `three_wheeler` rỗng (lỗ bao nhiêu thì giữ); hai xe chạm nhau thì mỗi xe một polygon.
- Nguyên nhân gốc: **guideline mơ hồ** là chính (v1.0 để 7 câu hỏi mở và thiếu quy tắc cho vật rỗng / vật chạm nhau), cộng **máy** (tạo box trùng, lồng, tách vụn một xe ba bánh thành nhiều box). Quy tắc chống box trùng / lồng / tách vụn chưa có trong v1.1, xem mục 7.

## 7. QC và quyết định (Part 6)

- Mẫu: 5 ảnh (0 ảnh `can_review` + 5 ngẫu nhiên, seed 42), 120 đối tượng; người kiểm: bạn (**hạn chế**: bạn vừa là người duyệt, không có người kiểm độc lập). Chấm theo taxonomy v1.0.
- Lỗi theo **đối tượng**: 7/120 = 5.8% · theo **ảnh**: 4/5 = 80.0% (ngưỡng Release ≤ 2%)
- Top lỗi:

| Loại lỗi | Số lỗi | Ảnh ví dụ |
|---|---|---|
| Thừa | 6 | `img_003` (tuk-tuk bị tách thành nhiều box trùng); ngoài ra `img_006` (box `person` trùng), `img_031` (box `bicycle` lồng nhau) |
| Sai vị trí/kích thước | 1 | `img_035` (tuk-tuk bị tách mui / thân) |

  Lỗi "thừa" chiếm 6/7 nên đây là **lỗi hệ thống**, không phải lỗi lẻ từng ảnh.
- **Giới hạn mẫu nhỏ**: 4 ảnh lỗi trong 5 ảnh, cận trên 95% (Wilson) của tỉ lệ ảnh lỗi thật là 96%, cận dưới khoảng 38%. Mẫu này không thể **chứng minh** ngưỡng ≤ 2%, nhưng cận dưới 38% vẫn cao hơn 2% rất nhiều nên đủ để kết luận **chưa Release**. Mẫu chỉ có ảnh ngẫu nhiên, không có ca khó, nên lỗi ở ca khó chưa được đo. 120 đối tượng nằm trong 5 ảnh nên không độc lập với nhau.
- Script kiểm được / không kiểm được:
  - Kiểm được: đếm lỗi theo phiếu, tỉ lệ theo đối tượng và theo ảnh, khoảng tin cậy Wilson, accepted/edited/deleted/added của máy (582/639 = 91.1% / 15 = 2.3% / 37 = 5.8%; added 57/657 = 8.7%), bắt bẫy 11/12 = 91.7% (1 bẫy lọt).
  - Không kiểm được: script chỉ đếm theo phiếu bạn điền nên không biết phiếu có thiếu lỗi không; ca bạn **cùng bỏ sót với máy**; box có ôm sát vật không; mask sai; độ nhất quán giữa hai người gán.

**Quyết định: Rework**, ngày 2026-11-18. Lý do (đi theo Hold → Rework → Release):
- Không Hold: guideline không mơ hồ tới mức phải dừng và dữ liệu không hỏng. v1.1 đã chốt 5/7 câu hỏi mở bằng quy tắc cụ thể, 2 câu còn lại (độ lệch giữa hai người, số `conf`) đã có nguyên tắc và chờ pilot.
- Rework: ảnh có lỗi 80.0% > 2%, và lỗi lặp lại ("thừa" 6/7 lỗi).
- Không Release: không đạt ngưỡng ảnh lỗi, và mẫu quá nhỏ để chứng minh ngược lại.

Việc cần làm tiếp:
1. Thêm quy tắc chống box trùng / lồng / tách vụn (một xe ba bánh = một box, không tách mui và thân; `person` trùng thì giữ một).
2. Sửa recipe pre-label (lọc box trùng theo IoU) và sửa 4 ảnh lỗi `img_003`, `img_006`, `img_031`, `img_035`.
3. QC lại bằng mẫu lớn hơn, lấy phân tầng (có ca `can_review` và ca khó), nhờ người kiểm thứ hai nếu được.
4. Chạy pilot 50 ca với 2 người gán độc lập để chốt độ lệch ở câu 4, rồi đo precision theo khoảng `conf` để chốt ngưỡng ở câu 7.

## 8. Ghi chú cá nhân

- **Dùng AI để làm bài**: dùng Claude để (a) đề xuất quyết định cho 7 câu hỏi mở ở mục 6 của v1.0, (b) giải thích `conf` và cách lấy mẫu pilot phân tầng, (c) đọc hiểu báo cáo QC tự động và điền template này. Kiểm tra: mỗi đề xuất đều phải khớp với một ảnh thật trong bộ dữ liệu (`img_025`, `img_026`, `img_010`, `img_006`, `img_024`) và tôi tự xác nhận lại bằng mắt trên ảnh gốc; phần AI viết là **ý tưởng quy tắc**, không phải nhãn: mọi box trong `qc_review.csv` và `submission/release/final_det.coco.json` là do tôi kiểm lại bằng mắt trên ảnh, có đối chiếu IoU với `final_det.coco.json` để không bỏ sót vật.
- Gold có bị nhiễm không? **Không**: 4 ảnh gold được gán tay trước khi xem pre-label và trước khi hỏi AI về từng ảnh; AI chỉ được hỏi về **quy tắc** (7 câu hỏi mở) và về cách đọc báo cáo, không hỏi vị trí hay lớp của từng vật trong ảnh gold. Đây là điểm yếu còn lại của bài: số đo trên gold vẫn lạc quan vì ngưỡng cũng chọn trên chính 4 ảnh đó.
- Máy / GPU / CVAT: chạy SAM 2.1 Base offline trên CPU bằng `scripts/sam2_points.py` (~5.05 s/ảnh, MobileSAM ~2.65 s/ảnh nhưng mask lệch). Không làm Interactor SAM trong CVAT. Vướng mắc cài đặt: mọi script chạy được trên CPU nên không cần GPU (máy có GTX 1650 nhưng không dùng); môi trường dùng `.venv` riêng trong repo; CVAT chạy bằng Docker.
- Dùng cache `data/cache/` thay vì tự chạy? **Có, với ảnh đã chạy rồi**: lần đầu phải chạy model để có pre-label, nhưng khi sweep ngưỡng và chạy lại các ca thử thì dùng lại output đã lưu (`work/prelabel/*`) thay vì gọi model lần nữa - detector chỉ 0.29-0.56 s/ảnh nhưng SAM + click thì 7.3 s/ảnh nên lãng phí thời gian máy.
- Lỗi / bug script đã gặp: (1) `merge_prelabels.py` báo `error: the following arguments are required: files, --out` khi gõ lệnh thiếu tham số - lỗi do gọi sai, không phải script hỏng; trên PowerShell còn phải gõ `python merge_prelabels.py` chứ không phải `merge_prelabels.py` vì PowerShell không tự chạy file trong thư mục hiện tại. (2) `qc_sample.py` / `qc_report.py` yêu cầu `qc_review.csv` phải điền đủ `error_type`, dòng để trống sẽ bị báo ảnh chưa kiểm. (3) Khi export COCO từ CVAT phải khớp đúng tên frame (`img_001.jpg`), nếu không sẽ lỗi `Could not match item id: 'img_001' with any task frame`.
- Một điều bạn làm khác lần sau: chốt quy tắc chống box trùng / lồng ngay từ pilot, và lấy mẫu QC có cả ca khó (lần này 0 ca khó trong 5 ảnh nên không phản ánh vùng rủi ro nhất).
