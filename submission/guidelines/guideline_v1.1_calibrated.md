# Label contract — Phố châu Á — BỔ SUNG v1.1 — 2A202604065 - Trần Trọng Thái

> Chỉ viết phần thay đổi so với v1.0 (không copy cả file). Chốt các câu hỏi mở ở mục 6 của v1.0 (hoặc nói rõ câu nào chưa chốt, vì sao)
> và thêm quy tắc mới rút ra từ lỗi của máy / lỗi của bạn. Mỗi ca khó có ảnh ví dụ (`img_0xx`) và khớp một dòng trong `qc/calibration_decisions.csv`.

## 1. Câu hỏi mở đã chốt (từ mục 6 của v1.0)

| # | Câu hỏi | Quyết định | Ảnh ví dụ | `case_id` |
|---|---|---|---|---|
| 1 | Xe đạp điện / xe máy điện nhỏ: `bicycle` hay `motorbike`? | Phân biệt theo hình dáng: thấy bàn đạp → `bicycle`; có yên + sàn để chân và không có bàn đạp → `motorbike`. Không phân biệt được thì tick `can_review`. | `img_025` | 434 |
| 2 | Khối xe máy đỗ dày đặc gần camera: gán từng xe hay `ignore_region`? Ngưỡng cụ thể? | Gán từng xe khi nhìn thấy ≥ 30–40% thân xe; phần còn lại (quá nhỏ / quá mờ / không tách được) gộp vào một `ignore_region` bao cả cụm. | `img_026` | 470 |
| 3 | Hàng nhô ra (xe đạp / ba gác chở tre, rổ): box xe có gồm hàng không? Mask thì sao? | Box và mask chỉ ôm xe + người; hàng nhô xa không tính. Hàng sát thân xe thì tính vào box/mask. | `img_010` | — |
| 4 | Người ngồi trong xe ba bánh có mui che: ngưỡng "≥ 20%" có hợp lý không? Độ lệch giữa 2 người đánh giá? | Bỏ mốc "20%". Chỉ gán `person` khi thấy rõ đầu hoặc vai; không thấy đầu/vai thì không gán người (xe vẫn gán). Độ lệch giữa hai người sẽ đo bằng pilot 50 ca, 2 người gán độc lập (chưa chạy). | `img_006` | — |
| 5 | Kính chắn gió / cửa sổ xe: thuộc mask xe hay không? | Thuộc mask xe (vùng kính nằm trong polygon). Người nhìn thấy qua kính vẫn gán `person` riêng theo câu 4 ở trên. | `img_006` | — |
| 6 | Box máy chồng nhau quá nhiều trong ảnh đêm: gộp hay giữ? | Giữ riêng nếu phân biệt được từng cá thể (mỗi xe một box); không phân biệt được thì gộp thành một `ignore_region`. | `img_024` | — |
| 7 | Mức tin cậy tối thiểu để chấp nhận box máy không cần sửa? | Có đặt ngưỡng `conf`, nhưng dùng để phân luồng duyệt chứ không phải để đổi nhãn: `conf` cao → kiểm tra mẫu 10%; `conf` thấp → sửa toàn bộ. Số cụ thể chốt sau khi đo precision trên pilot (chưa chốt). | `img_010`, `img_026` | — |

> Chưa chốt được: câu 4 (độ lệch) và câu 7 (con số `conf`) đang phụ thuộc pilot, nên v1.1 chỉ chốt nguyên tắc, chưa chốt con số.

## 2. Ca khó mới (từ lỗi của máy, bẫy lọt, box `can_review`)

| Ảnh | Tình huống | Máy làm gì (model nào) | Quyết định của bạn | Quy tắc mới (vào mục nào của v1.0) |
|---|---|---|---|---|
| `img_025` | Xe đạp điện / xe máy điện nhỏ, dễ nhầm lẫn lớp | YOLO26 (chưa chốt ngưỡng `conf`) | Theo hình dáng: có bàn đạp → `bicycle`; có yên, sàn để chân, không bàn đạp → `motorbike` | Mục 2: bổ sung vào định nghĩa `bicycle` ("xe đạp điện có bàn đạp vẫn là `bicycle`") và `motorbike` ("xe máy điện không bàn đạp là `motorbike`") |
| `img_026` | Khối xe máy đỗ dày đặc gần camera, từng cá thể không tách được | YOLO26 (chưa chốt ngưỡng `conf`) | Từng xe nhìn thấy ≥ 30–40% thân xe → gán từng xe; phần còn lại gộp 1 `ignore_region` | Mục 2 (`ignore_region`) + mục 4.1.6: đổi tiêu chí "cao < 20 px" thành "thấy ≥ 30–40% thân xe thì gán từng xe, phần còn lại gộp một `ignore_region`" |
| `img_010` | Xe đạp / ba gác chở hàng, hàng nhô ra xa khỏi thân xe | YOLO26 (chưa chốt ngưỡng `conf`) | Không tính hàng nhô xa vào box/mask; hàng sát thân thì tính | Mục 4.1.5 + mục 4.2.3: box và mask chỉ ôm xe + người; hàng nhô xa bỏ ra, hàng sát thân giữ lại |
| `img_006` | Người ngồi trong xe ba bánh có mui, chỉ thấy thoáng; máy gán box `person` chồng/chia vụn | YOLO26 (chưa chốt ngưỡng `conf`) | Bỏ mốc "20%": chỉ gán `person` khi thấy rõ đầu hoặc vai; đo độ lệch bằng pilot 50 ca, 2 người gán độc lập | Mục 4.1.2 + mục 5: đổi dòng "Tuk-tuk có hành khách ngồi nhìn thấy qua cạnh hở — ≥ 20%" thành "thấy rõ đầu hoặc vai mới gán `person`" |
| `img_006` | Kính chắn gió che giữa người và camera | YOLO26 (chưa chốt ngưỡng `conf`) | Kính thuộc mask xe; người nhìn thấy qua kính vẫn gán `person` riêng theo câu 4 | Mục 4.2.3: chốt "kính xe thuộc vật (mask)", đồng thời "người thấy qua kính vẫn gán riêng" |
| `img_024` | Ảnh ban đêm, box máy chồng nhau nhiều, khó tách từng xe | YOLO26 (chưa chốt ngưỡng `conf`) | Giữ riêng nếu phân biệt được từng cá thể; không phân biệt được → gộp một `ignore_region` | Mục 4.1.6 + mục 2: box chồng nhau không tự động là lỗi — tách được thì giữ, không tách được thì `ignore_region` |
| `img_010`, `img_026` | Không biết box máy nào đáng tin để duyệt nhanh | YOLO26 (chưa chốt ngưỡng `conf`) | Có ngưỡng `conf`, dùng để phân luồng duyệt: `conf` cao → kiểm tra mẫu 10%; `conf` thấp → sửa toàn bộ; số chốt sau khi đo precision trên pilot | Mục 3 (`conf`) + mục 7: `conf` là công cụ phân luồng duyệt, không phải quy tắc chất lượng và không phải ngưỡng người dùng |

## 3. Lịch sử thay đổi

| Phiên bản | Thay đổi | Lý do (ca nào, `case_id`) |
|---|---|---|
| v1.0 | Bản giảng viên | — |
| v1.1 | Mục 1: chốt 7 câu hỏi mở của v1.0 — (1) xe đạp/xe máy điện theo hình dáng; (2) xe đỗ dày gần camera: gán từng xe nếu thấy ≥ 30–40% thân xe, còn lại gộp `ignore_region`; (3) hàng nhô xa không tính vào box/mask, hàng sát thân thì tính; (4) bỏ mốc "20%" cho người trong xe ba bánh có mui, chỉ gán khi thấy rõ đầu/vai (độ lệch đo bằng pilot 50 ca, 2 người); (5) kính chắn gió thuộc mask xe, người thấy qua kính vẫn gán riêng; (6) box chồng nhau ban đêm: giữ riêng nếu tách được, không thì `ignore_region`; (7) có ngưỡng `conf` để phân luồng duyệt (cao → mẫu 10%, thấp → sửa toàn bộ), số chốt sau pilot. Mục 2: thêm 7 ca khó mới với ảnh ví dụ và quy tắc nhắm vào mục 2 / 3 / 4.1.2 / 4.1.5 / 4.1.6 / 4.2.3 / 5 / 7 của v1.0. Mục 4.1.5 và 4.1.6 của v1.0 cần sửa theo câu 3 và câu 6; dòng "Tuk-tuk có hành khách … ≥ 20%" ở mục 5 của v1.0 cần sửa theo câu 4. | `img_025` (case_id 434), `img_026` (case_id 470), `img_010`, `img_006`, `img_024`; lỗi QC ở `img_003` (tuk-tuk bị tách thành nhiều box trùng), `img_006` (box `person` trùng lặp), `img_031` (box `bicycle` lồng nhau), `img_035` (tuk-tuk tách mui/thân) |
