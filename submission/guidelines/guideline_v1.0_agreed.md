# Guideline gán nhãn — v0.1 — 064

- Phiên bản: v0.1 (bản cá nhân, viết độc lập ở Pha 1, chưa họp nhóm)
- Ngày: 2026-11-16
- Người viết / duyệt: 064
- Phạm vi: 40 ảnh `data/images/img_001.jpg` … `img_040.jpg` (cạnh dài tối đa 1024 px), gán bounding box trên CVAT, xuất COCO 1.0

Quy ước đọc: `R-xx` là mã quy tắc ở mục 3; mọi ví dụ ghi tên ảnh `img_0xx` để người đọc mở lại được. Ca ghi **Hỏi reviewer** ở mục 4 là ca tôi đã có quyết định tạm nhưng cần nhóm chốt ở Pha 2.

---

## 1. Mục tiêu

- **Mô hình:** phát hiện vật thể 2D (bounding box + lớp) trên ảnh camera đường phố, dùng cho camera giao thông và xe tự lái.
- **Cần phát hiện:** mọi người thật và mọi phương tiện thật nhìn thấy trong ảnh, dù đang chạy, đang đỗ hay đang trưng bày, ở lòng đường, vỉa hè, trong quán hay trên ban công.
- **Sai nguy hiểm nhất — bỏ sót vật thật:** xe tự lái không thấy người hoặc xe nên không phanh. Vì vậy vật thật còn nhận ra được thì phải vẽ, kể cả khi bị che, bị cắt mép hay bị nhòe (R-10, R-11, R-12).
- **Sai thứ hai — báo nhầm vật không thật:** tượng, ảnh in, hình phản chiếu làm xe phanh gấp vô cớ và dạy mô hình sai khái niệm "người", "xe". Loại bằng các dấu hiệu cụ thể ở R-04.
- **Khi phân vân vật thật hay không thật:** xét dấu hiệu ở R-04; nếu dấu hiệu mâu thuẫn thì vẽ box và tick `can_review`. Ưu tiên không bỏ sót.
- **Ngoài phạm vi (không gán box, không vẽ `ignore_region`):** vật không thật (R-04), xe chủ (R-07), động vật (con chó ở img_009, chim bồ câu ở img_017), cọc tiêu (img_004, img_032), biển báo, xe đẩy, quầy hàng.

## 2. Lớp và thuộc tính

### 2.1 Lớp

Mỗi vật thuộc đúng một lớp. Phân lớp xe làm theo phép thử R-13 (xe từ 4 bánh trở lên) và R-14 (xe 2–3 bánh).

| Lớp | Định nghĩa (một câu) | Ví dụ | Phản ví dụ / dễ nhầm với |
|---|---|---|---|
| `car` | Ô tô thật 4 bánh chở người, có nắp capo và khoang khách nóc thấp (sedan, hatchback, SUV, taxi, xe mui trần), không thuộc `truck`, `bus`, `van`. | img_015 (xe đỏ, xe bạc), img_023 (sedan đen), img_031 (SUV trắng), img_019 (xe diễu hành có biển số) | Bán tải → `truck` (img_012); van → `van` (img_016); ô tô đồ chơi → không gán (img_038) |
| `van` | Xe 4 bánh thân hộp liền khối, nóc cao phẳng tới đuôi, có nhiều nhất 3 ô kính hông phía sau ghế lái. | img_005 (van bạc 51LD-70.65), img_016 (van trắng) | Xe tải thùng kín có cabin tách rời → `truck` (Isuzu ở img_007); xe có từ 4 ô kính khách → `bus` (img_009) |
| `truck` | Xe chở hàng có cabin tách rời khỏi thùng hàng (thùng lửng, thùng kín, thùng bạt, giàn chở xe), kể cả bán tải có hoặc không có nắp thùng. | img_001, img_002, img_007, img_012, img_028, img_036, img_037 | Tượng bán tải bằng đá (img_027) → không gán; van (img_016) → `van` |
| `bus` | Xe chở khách thân dài liền khối, có từ 4 ô kính khách mỗi bên trở lên (xe buýt, minibus, xe khách). | img_009 (minibus 6 ô kính), img_031 (xe buýt số 82), img_037 (2 xe buýt xanh) | Van có nhiều nhất 3 ô kính → `van` (img_016) |
| `motorcycle` | Xe 2 bánh có động cơ xăng hoặc điện (xe máy, xe tay ga, mô tô), kể cả khi gắn thùng bên hay kéo thùng, đang chạy hay đang đỗ. | img_010, img_026, img_029 (xe máy điện), img_022 và img_025 (gắn thùng bên) | Mô hình xe máy bằng phế liệu (img_011) → không gán; xe đạp → `bicycle` |
| `tuk_tuk` | Xe 3 bánh có động cơ, khung 3 bánh liền một khối, có mui hoặc cabin chung (tuk-tuk, xe lam, auto-rickshaw). | img_004, img_016, img_035, img_040 | Xe máy gắn thùng bên → `motorcycle` (img_025); xích lô đạp chân → `bicycle` (img_023) |
| `bicycle` | Xe không động cơ do người đạp: xe đạp 2 bánh, xích lô. | img_003, img_006, img_023 (xích lô), img_029 | Xe máy điện → `motorcycle` (img_029) |
| `person` | Người thật nhìn thấy đủ để nhận ra là người (đầu, thân hoặc chân), ở bất kỳ vị trí nào, trừ người ngồi trong xe có cửa kín (R-03). | img_014 và img_034 (người đi bộ), img_026 (người trên xe máy), img_009 (người trên ban công) | Tượng (img_017), ma-nơ-canh (img_033), người trên biển quảng cáo (img_018), ảnh người dán trên kính taxi (img_040) → không gán |
| `ignore_region` | Vùng chữ nhật bao một cụm vật thuộc các lớp trên mà R-08 hoặc R-09 cho phép không vẽ từng box; bỏ sót trong vùng này không tính lỗi; không có thuộc tính. | img_019 (đám đông sau rào), img_021 (dòng xe máy phía sau), img_030 (bãi xe máy) | Không dùng để che tượng, poster, phản chiếu: những vật đó chỉ cần không gán |

**Vật không thuộc lớp nào:** không gán, không vẽ `ignore_region` (xem "Ngoài phạm vi" ở mục 1). Phương tiện cơ giới lạ (xe cẩu, xe công nông…) gán vào lớp cho ra ở phép thử R-13 hoặc R-14 và tick `can_review`. Không tạo thêm lớp khi đang gán; muốn thêm lớp thì đưa ra họp nhóm.

### 2.2 Thuộc tính

Chỉ dùng thuộc tính trong panel **Objects** của CVAT. Không dùng cờ occluded có sẵn của CVAT (phím `Q`); script bỏ qua cờ này.

| Thuộc tính | Mức | Nghĩa và ngưỡng | Cách đo nhanh | Ví dụ |
|---|---|---|---|---|
| `occluded_level` | `0` | Bị vật khác che không quá 5% | Hình dung khung chữ nhật của **toàn bộ** vật trong ảnh (kể cả phần bị che), chia thành lưới 3×3 = 9 ô; ô nào bị che quá nửa thì đếm. **0 ô** | img_014 (2 người đi bộ), img_028 (bán tải) |
| | `1` | Che 6–30% | **1–2 ô** | img_001 (xe tải bị người đứng che khoảng 12%), img_018 (xe sau song rào) |
| | `2` | Che 31–70% | **3–6 ô** | img_005 (van bị người đi xe máy che khoảng một nửa), img_012 (xe tải sau bán tải) |
| | `3` | Che 71–90%, vẫn nhận ra lớp chắc chắn | **7–8 ô** | img_037 (xe buýt sau bán tải), img_022 (người chỉ thấy đầu) |
| | (không vẽ box) | Che trên 90%, hoặc phần nhìn thấy không đủ nhận ra lớp | **9 ô** | img_005 (thân xe máy bị người che gần hết), img_040 (chỉ thấy 2 gương xe máy) |
| `truncated` | `yes` | Một phần vật nằm ngoài khung ảnh: cạnh box chạm hoặc cách mép ảnh không quá 2 px **và** vật còn kéo dài ra ngoài ảnh. | Nhìn 4 cạnh box | img_015 (2 ô tô ở hai mép), img_026 (bánh xe máy chạm mép dưới), img_040 (người tiền cảnh) |
| | `no` | Toàn bộ vật nằm trong ảnh. | | img_009 (minibus), img_028 (bán tải) |
| `can_review` | tick | Chỉ tick khi có ít nhất một trong 3 điều kiện: (a) phân vân giữa hai lớp; (b) phân vân vật thật hay không thật (R-04); (c) biên box có thể lệch hơn khoảng 10% chiều cao vật vì nhòe, tối hoặc bị che. | Ghi điều kiện a, b hoặc c vào nhật ký cá nhân | (a) img_011 xe trắng sau tượng; (b) img_019 nhân vật hóa trang; (c) img_014 xe máy nhòe |
| | không tick | Mọi trường hợp còn lại. | | |

Ba lưu ý khi tính `occluded_level` (chi tiết ở R-11):

- Không tính phần nằm ngoài ảnh (phần đó thuộc `truncated`).
- Không tính phần bị chính người lái hoặc người ngồi trên xe đó che; hàng chở trên xe thì vẫn tính.
- Vật thưa nhìn xuyên được (song rào, lưới, kính) chỉ tính phần bị song hoặc khung che.

Mỗi box tick `can_review` tạo một ca `co_can_review` trong `calibration_candidates.csv`, nên chỉ tick khi đúng 3 điều kiện trên, không tick "cho chắc".

## 3. Quy tắc vẽ box

### 3.1 Thứ tự quyết định cho mỗi vật

1. **Vật thật?** Không (tượng, mô hình, ảnh in, hình phản chiếu…) → bỏ qua (R-04).
2. **Là xe chủ, hoặc phần nhìn thấy không đủ nhận ra lớp?** → bỏ qua (R-07, mục 2.2).
3. **Box cao dưới 20 px?** → bỏ qua, hoặc gộp vào `ignore_region` nếu nằm trong nhóm từ 3 vật nhỏ trở lên (R-08).
4. **Nằm trong cụm từ 10 vật cùng lớp trở lên?** → xử lý theo R-09.
5. **Chọn lớp** theo bảng 2.1 và phép thử R-13, R-14.
6. **Vẽ box** theo R-01 (thêm R-02, R-05, R-06 nếu có người trên xe, xe chở xe hoặc hàng hóa), rồi gán `occluded_level`, `truncated`, `can_review` theo mục 2.2.

Mỗi vật đúng một box, mỗi box đúng một vật; chỉ `ignore_region` được bao nhiều vật. Ảnh không có vật nào cần gán vẫn giữ trong Task (R-16).

### 3.2 Quy tắc

**R-01 — Box ôm sát phần nhìn thấy.**
- Box là hình chữ nhật nhỏ nhất chứa mọi pixel nhìn thấy của vật; mỗi cạnh lệch không quá 2 px so với biên vật. Không ước phần bị che.
- Không bao bóng đổ và bóng phản chiếu trên mặt đường ướt (img_013, img_015, img_031).
- Xe: gồm gương, tay lái, cản, mui; không gồm anten, dây buộc, hàng nhô ra (R-06).
- Người: gồm tóc, nón, mũ bảo hiểm, quần áo, túi đeo sát người; không gồm vật cầm hoặc gánh nhô ra ngoài thân (R-06).
- Box nằm trong ảnh: không âm, không vượt chiều rộng hay chiều cao ảnh (chú ý ảnh dọc 784×1024 ở img_026).
- Lý do: box cao từ 50 px phải đạt IoU ≥ 0.85 giữa hai người gán; mỗi cạnh lệch vài pixel là đã có thể rớt ngưỡng.

**R-02 — Người trên xe hở: vẽ box riêng cho xe và cho từng người.**
- Áp dụng cho người trên xe máy, xe đạp, xích lô, tuk-tuk, dù xe đang chạy hay đang đỗ.
- Box `person` bao từ đỉnh đầu (hoặc mũ) tới bàn chân, kể cả chân đặt trên xe.
- Box xe bao từ điểm thấp nhất của bánh tới điểm cao nhất của **chính chiếc xe** (tay lái, gương, mui); không kéo lên theo đầu người.
- Phần xe bị chính người lái che không tính vào `occluded_level` (R-11).
- Áp thử trên nhiều ảnh:
  - img_035: người áo trắng đội mũ bảo hiểm trên xe máy → 1 `person` + 1 `motorcycle` = **2 box**.
  - img_026: 2 người trên 1 xe máy đỏ → 2 `person` + 1 `motorcycle` = **3 box**.
  - img_029: người đạp xe đạp → 1 `person` + 1 `bicycle` = **2 box**; ống nhựa không thuộc box nào.
  - img_010: người phụ nữ ngồi trên xe máy đang đỗ → 1 `person` + 1 `motorcycle` = **2 box**.

**R-03 — Người trong xe.**
- Người ngồi trong xe có cửa kín (ô tô, van, buýt, cabin xe tải): không gán, kể cả khi thấy qua kính. Ví dụ: hành khách trong minibus ở img_009.
- Người trên xe không có cửa hông (xe máy, xe đạp, xích lô, tuk-tuk) hoặc xe mui trần: gán `person` nếu nhìn thấy, kể cả qua kính chắn gió. Ví dụ: tài xế tuk-tuk ở img_004, img_016, img_035; tài xế xe mui trần ở img_019.
- Lý do: người trong xe kín chỉ thấy lờ mờ, mỗi người gán sẽ thấy khác nhau, và box của xe đã bao vùng đó.

**R-04 — Vật không thật thì không gán.**
- Không gán: tượng, phù điêu, mô hình, đồ chơi, ma-nơ-canh; người hoặc xe trên poster, biển quảng cáo, tranh tường, ảnh dán; hình phản chiếu trong gương, kính, mặt nước.
- Dấu hiệu vật không thật: chất liệu đá, đồng, nhựa, phế liệu, giấy in; đặt trên bệ, thảm trưng bày hoặc trong tủ kính; kích thước khác vật thật; xe không có kính, không có biển số.
- Dấu hiệu vật thật: có biển số, có người điều khiển, đang ở trên đường, kích thước bình thường.
- Dấu hiệu mâu thuẫn → vẽ box + tick `can_review` (điều kiện b).
- Người mặc đồ hóa trang, có kích thước và tư thế của người → `person` + `can_review` (img_019).
- Vật không thật đặt trên vật thật: chỉ gán vật thật, box không bao vật không thật (tượng ngựa trên bán tải ở img_028).
- Ảnh đen trắng hoặc ảnh cũ chụp cảnh thật vẫn gán bình thường (img_022); chỉ ảnh in nằm **bên trong** cảnh mới không gán.

**R-05 — Xe chở xe, xe trên xe.**
- Ô tô thật nguyên chiếc trên xe chuyên chở ô tô: mỗi chiếc một `car`. Xe chở là **một** `truck` bao đầu xe, giàn chở và rơ-moóc nối liền (img_002: 1 `truck` + 8 `car`).
- Vật hình xe đặt trên giá nóc của xe thường là đồ trưng bày, không gán (vỏ xe vàng không kính ở img_024).
- Nếu nhóm đổi quy tắc này phải rà lại cả img_002, img_024, img_028.

**R-06 — Hàng hóa và đồ mang theo.**
- Box xe bao thân xe và hàng nằm trong thùng hoặc trên giá của xe, với điều kiện hàng không cao hơn nóc cabin/mui và không vượt ra ngoài mép thân xe (img_001 ống sắt nằm trên thùng lửng, img_012 hàng trong thùng bán tải → nằm trong box).
- Phần hàng cao hơn nóc hoặc nhô ra ngoài thân xe: không bao (bao tải trên tuk-tuk ở img_035, ống nhựa dài ở img_029, tượng ngựa ở img_028).
- Box người không bao vật cầm hoặc gánh nhô ra ngoài thân (đòn gánh và thúng hàng ở img_003).

**R-07 — Xe chủ.** Không gán phần xe đang chở người chụp (gương, cửa, khung kính, nắp capo, tay lái) và vệt phản chiếu trên kính xe chủ (img_020, img_018).

**R-08 — Vật nhỏ.**
- Vật có box cao dưới 20 px (tính trên ảnh gốc) không vẽ box.
- Từ 3 vật nhỏ trở lên nằm sát nhau (khoảng trống giữa chúng nhỏ hơn chiều cao của chúng) → một `ignore_region` bao cả nhóm. Ví dụ: người ở cuối đường dạo img_025, cao khoảng 16 px.
- Vật nhỏ đứng lẻ thì bỏ qua, không tính là bỏ sót. Ví dụ: xe trắng ở xa trong img_002, cao khoảng 15 px.
- Sau khi export, `validate_labels.py` in `[WARN] ... rà lại quy tắc vật nhỏ` cho box cao dưới 20 px → xóa box đó hoặc gộp vào `ignore_region`.

**R-09 — Đám đông và cụm dày.**
- Cụm là nhóm từ **10 vật cùng lớp** trở lên, mà box của mỗi vật chạm hoặc chồng lên box của ít nhất một vật khác trong nhóm.
- Trong cụm: vật có `occluded_level` ≤ 1 **và** cao từ 50 px → vẽ box riêng. Mọi vật còn lại của cụm → gộp vào một `ignore_region`.
- Người lái đi theo xe của mình: xe có box thì người lái có box; xe nằm trong vùng ignore thì người lái cũng nằm trong vùng đó.
- Vùng ignore không được phủ từ 50% diện tích một box đã vẽ, vì script coi box đó là nằm trong vùng ignore. Nếu một hình chữ nhật không đáp ứng được thì tách thành 2 vùng.
- Nhóm dưới 10 vật: vẽ từng vật theo R-01 (img_036: 2 cụm xe máy, mỗi cụm dưới 10 xe).
- Ví dụ áp dụng: img_008, img_019, img_021, img_030, img_032.
- Lý do: vật trong cụm dày không tách được biên nên IAA thấp; mốc 50 px trùng mốc `large_px` của ngưỡng IoU trong `taxonomy.json`.

**R-10 — Cắt mép.** Vật bị khung ảnh cắt vẫn vẽ nếu phần trong ảnh đủ nhận ra lớp; box dừng ở mép ảnh; `truncated = yes` (img_015, img_016, img_040).

**R-11 — Cách tính che khuất.**
- Chỉ tính phần bị vật khác che, kể cả hàng chở trên chính xe đó (mẹt rau che xe đạp ở img_010).
- Không tính phần bị chính người lái hoặc người ngồi trên xe đó che, và không tính phần nằm ngoài ảnh.
- Vật thưa nhìn xuyên được (song rào, lưới, kính) chỉ tính phần bị song hoặc khung che (xe sau song rào ở img_018 → mức `1`).

**R-12 — Nhòe và tối.**
- Vật nhòe do chuyển động hoặc nằm trong vùng tối vẫn vẽ nếu còn nhận ra lớp. Box bao phần thân đậm nhất, không bao vệt nhòe nhạt; tick `can_review` (điều kiện c).
- Không còn nhận ra hình dạng → không vẽ.
- Ví dụ: img_014, img_015 (nhòe), img_007 (tối).

**R-13 — Phân lớp xe từ 4 bánh trở lên** (xét lần lượt, dừng ở bước đầu tiên đúng):
1. Cabin tách rời khỏi thùng hàng (nhìn thấy khe hoặc vách giữa cabin và thùng) → `truck`. Gồm cả bán tải gắn nắp thùng cao (img_037) và xe chở ô tô (img_002).
2. Thân liền khối chở khách, có từ 4 ô kính khách mỗi bên → `bus` (img_009).
3. Thân hộp liền khối, nóc cao phẳng tới đuôi, nhiều nhất 3 ô kính hông sau ghế lái → `van` (img_005, img_016).
4. Còn lại → `car`.

Nếu phân vân giữa hai lớp thì chọn lớp ở bước sớm hơn và tick `can_review` (điều kiện a).

**R-14 — Phân lớp xe 2–3 bánh:**
1. Không có động cơ, người đạp → `bicycle` (xe đạp, xích lô img_023).
2. Có động cơ, và nếu tháo thùng bên hoặc thùng kéo ra vẫn còn nguyên một chiếc xe máy 2 bánh → `motorcycle`; box bao cả thùng gắn liền (img_022, img_025). Xe máy điện cũng là `motorcycle` (img_029).
3. Có động cơ, khung 3 bánh liền một khối có mui hoặc cabin chung → `tuk_tuk` (img_004, img_035).

**R-15 — Xe thật đang đỗ hoặc đang trưng bày vẫn gán** như xe đang chạy (bảo tàng xe img_024, bãi đỗ img_018, bãi xe máy img_030).

**R-16 — Ảnh không có vật cần gán vẫn giữ trong Task CVAT** và có mặt trong file COCO với 0 annotation: img_017, img_020, img_027, img_033, img_038.

## 4. Ca khó

Bảng ghi 37/40 ảnh. Ba ảnh còn lại (img_006, img_034, img_039) là cảnh phố thông thường, chỉ cần R-01, R-02, R-08, R-14.

| Ảnh | Tình huống | Rule ID | Quyết định | Lý do | Loại |
|---|---|---|---|---|---|
| img_001 | Xe tải nhỏ thùng lửng chở ống sắt, bị cắt ở mép phải và mép dưới; cô gái đứng che đuôi xe; nhiều người ngồi trong quán phía sau; poster xúc xích | R-13, R-06, R-10, R-11, R-04 | 1 `truck` (`truncated=yes`, `occluded_level=1`), ống sắt nằm trong box; mỗi người nhận ra được 1 `person`, mức che theo mục 2.2; poster không gán | Cabin tách khỏi thùng; người đứng che khoảng 12% thân xe; ống nằm gọn trên thùng | Quy tắc |
| img_002 | Xe chở ô tô có rơ-moóc, 8 ô tô (5 chiếc tầng trên, 3 chiếc tầng dưới); 1 công nhân đứng cạnh rơ-moóc; xe trắng rất xa cao khoảng 15 px | R-05, R-13, R-08 | 1 `truck` bao đầu xe, giàn và rơ-moóc; 8 `car`; 1 `person`; xe ở xa không vẽ | Xe được chở là ô tô thật nguyên chiếc; đổi quy tắc này phải rà lại img_024, img_028 | Quy tắc |
| img_003 | Người gánh hàng (nón chạm mép trên); 2 xe đạp có giỏ (1 xe bị cắt ở mép trái); người đứng cạnh xe đạp; người lái xe máy | R-06, R-14, R-02, R-10 | `person` cho người gánh hàng, không bao đòn gánh và thúng, `truncated=yes`; 2 `bicycle`; người đứng cạnh xe đạp là một `person` riêng; người lái xe máy: `person` + `motorcycle` | Thúng hàng nhô ra ngoài thân người | Quy tắc |
| img_004 | 2 tuk-tuk đỗ, mỗi xe có tài xế ngồi bên trong; cọc tiêu | R-14, R-03 | 2 `tuk_tuk` + 2 `person`; cọc tiêu không gán | Tuk-tuk không có cửa hông nên gán người bên trong | Quy tắc |
| img_005 | Cận cảnh nhiều người trên các xe máy chen sát nhau, thân xe gần như bị người che hết; van bạc phía sau | R-02, R-11, R-10, mục 2.2 | Mỗi người thấy mặt hoặc mũ bảo hiểm: 1 `person` (người ở mép ảnh `truncated=yes`); xe máy chỉ vẽ khi còn nhận ra thân xe, bị che trên 90% thì không vẽ; van: `van`, `occluded_level=2` | Van bị người che khoảng một nửa (3–6 ô lưới) | Quy tắc |
| img_007 | Nhà xe tối: bán tải Toyota; xe tải thùng có đèn phía sau bán tải; xe tải Isuzu ở mép phải | R-13, R-11, R-10, R-12 | 3 `truck`: bán tải; xe thùng phía sau `occluded_level=2`; Isuzu `truncated=yes` | Bán tải có cabin tách thùng; xe phía sau bị bán tải che khoảng nửa thân | Quy tắc |
| img_008 | Ban đêm, hơn 10 xe máy đỗ sát nhau; người áo cam số 4 ngồi trên xe, người áo cam số 7 đứng cạnh xe; người đi bộ phía sau | R-09, R-02 | Xe có `occluded_level` ≤ 1 và cao từ 50 px → `motorcycle`; các xe còn lại → 1 `ignore_region`; người số 4: `person` + `motorcycle`; người số 7: `person` | Cụm từ 10 xe chồng nhau | Quy tắc |
| img_009 | Minibus 6 ô kính hông, hành khách thấy qua kính; người lái xe máy; người trên ban công cao khoảng 30 px; con chó | R-13, R-03, R-02 | `bus`; hành khách không gán; người lái xe máy: `person` + `motorcycle`; người trên ban công: `person`; con chó không gán | Có từ 4 ô kính khách → `bus`; người ở ban công vẫn là người thật, cao từ 20 px | Quy tắc |
| img_010 | 2 người trên 1 xe máy; người phụ nữ ngồi trên xe máy đang đỗ; xe đạp chở mẹt rau bị hàng che phần lớn; người bán hàng đứng | R-02, R-06, R-11 | 2 `person` + 1 `motorcycle`; xe đang đỗ: `person` + `motorcycle`; `bicycle` không bao mẹt rau nhô cao, phần hàng che tính vào `occluded_level`; người đứng: `person` | Người ngồi trên xe đỗ vẫn theo R-02 | Quy tắc |
| img_011 | Tượng thú khổng lồ và mô hình xe máy làm từ xích, phế liệu đặt trên thảm trưng bày; 2 người đứng gần; xe trắng bị tượng che; đám đông ở xa | R-04, R-09, R-11 | Không gán tượng và mô hình xe; 2 `person`; xe trắng gán theo R-13, `occluded_level=2`, `can_review` (phân vân `car`/`van`); đám đông xa theo R-09 | Chất liệu phế liệu, đặt trên thảm, không biển số | Hỏi reviewer |
| img_012 | Bán tải chở hàng trong thùng (hàng thấp hơn nóc cabin), bị cắt ở mép dưới và mép phải; SUV Mazda; xe tải phía sau bị bán tải che; ô tô ở xa cao khoảng 26 px | R-13, R-06, R-10, R-11 | `truck` bao cả hàng trong thùng, `truncated=yes`; Mazda: `car`; xe tải phía sau: `truck`, `occluded_level=2`; ô tô ở xa: `car` | Hàng không cao hơn nóc cabin; xe phía sau bị che khoảng 60% | Quy tắc |
| img_013 | Ban đêm, đường ướt; tượng vàng và tượng đá cạnh 2 người đi bộ; xe máy đỗ sau cọc tiêu; ô tô trắng ở xa cao khoảng 28 px; bóng phản chiếu trên đường | R-04, R-01, R-08 | 2 `person`; không gán tượng; `motorcycle`; ô tô: `car`; bóng phản chiếu không gán và không bao vào box | Ô tô cao từ 20 px nên vẫn vẽ | Quy tắc |
| img_014 | 2 người đi bộ rõ nét; xe máy và người lái chạy nhòe ở hai bên và phía sau | R-12, R-02 | 2 `person`, `occluded_level=0`; xe và người nhòe còn nhận ra hình dạng → box quanh phần đậm nhất + `can_review`; vệt nhòe không còn nhận ra thì không vẽ | Không bỏ sót vật thật dù nhòe | Hỏi reviewer |
| img_015 | Ô tô đỏ ở mép trái, ô tô bạc ở mép phải; xe phía sau nhòe; mặt đường ướt phản chiếu | R-10, R-12, R-01 | 2 `car`, `truncated=yes`; xe nhòe còn nhận ra → `car` + `can_review`; bóng trên mặt đường không bao | Vật bị cắt mép vẫn phải phát hiện | Quy tắc |
| img_016 | 2 tuk-tuk ở hai mép có người bên trong; bán tải Toyota sau tuk-tuk; xe Vespa có người lái; van trắng; sedan tối màu bị che; người phụ nữ áo đỏ ở góc phải dưới | R-14, R-13, R-02, R-03, R-10 | 2 `tuk_tuk`, `truncated=yes`, cộng `person` cho người thấy được bên trong; bán tải: `truck`, `occluded_level=2`; Vespa: `motorcycle` + `person`; `van`; sedan: `car`; người áo đỏ: `person`, `truncated=yes` | Bán tải bị tuk-tuk che khoảng nửa thân | Quy tắc |
| img_017 | Tượng đồng người cưỡi ngựa đội nón giao thông | R-04, R-16 | Không gán; 0 annotation | Hai người cùng vẽ `person` ở đây vẫn sai dù IAA cao | Quy tắc |
| img_018 | Ảnh chụp qua kính xe; biển quảng cáo in chân dung; 5 xe đỗ sau hàng rào song sắt, 2 xe ở hai mép | R-04, R-07, R-11, R-15, R-10 | Không gán người trên biển và vệt phản chiếu trên kính; mỗi xe 1 box theo R-13, `occluded_level=1` vì song rào (xe bị tấm biển "Ben Car" che thêm thì tính thêm); xe ở mép `truncated=yes` | Song rào chỉ che khoảng 10–20% | Quy tắc |
| img_019 | Xe diễu hành dáng giả tưởng nhưng có biển số YR04 NUM và tài xế thật; nhân vật chuột hóa trang ngồi ghế phụ; đám đông sau rào chắn; đầu khán giả ở góc dưới phải | R-04, R-03, R-09, R-10 | `car`; tài xế: `person`; nhân vật chuột: `person` + `can_review`; đám đông theo R-09; đầu khán giả: `person`, `truncated=yes` | Xe có biển số và người lái là xe thật; nhân vật hóa trang có tư thế và kích thước người | Hỏi reviewer |
| img_020 | Gương chiếu hậu phản chiếu xe phía sau; vỏ gương, cửa, kính của xe chủ | R-04, R-07, R-16 | Không gán; 0 annotation | Hình phản chiếu và xe chủ | Quy tắc |
| img_021 | Dòng xe máy dày đặc hàng chục xe; ô tô trắng bên trái | R-09, R-02, R-11 | Xe có `occluded_level` ≤ 1 và cao từ 50 px → `motorcycle` + `person` cho người lái; phần còn lại → `ignore_region`, không phủ từ 50% box đã vẽ; ô tô: `car`, mức che theo mục 2.2 | Phần xa không tách được biên từng xe | Quy tắc |
| img_022 | Ảnh đen trắng chụp cảnh thật: xe máy gắn sidecar trên đường rừng; một người ngồi sau kính chắn của sidecar, chỉ thấy đầu | R-04, R-14, R-11 | 1 `motorcycle` bao cả sidecar; người: `person`, `occluded_level=3`, `can_review` | Ảnh đen trắng của cảnh thật vẫn gán; tháo thùng vẫn còn xe máy 2 bánh | Hỏi reviewer |
| img_023 | Xích lô có người đạp; sedan đen; xe máy và người đi bộ ở xa | R-14, R-02, R-08 | `bicycle` + `person`; `car`; vật ở xa cao từ 20 px thì vẽ, dưới 20 px thì bỏ | Xích lô không có động cơ | Quy tắc |
| img_024 | Bảo tàng ô tô; vỏ xe vàng không kính, nhỏ hơn xe thật, đặt trên giá nóc xe đen; nhiều xe trưng bày; xe cam ở góc dưới trái | R-15, R-05, R-06, R-09, R-10 | Mỗi xe trưng bày: `car`; vỏ xe vàng không gán, box xe đen không bao nó; xe cam `truncated=yes`; dãy xe xa chồng nhau theo R-09 | Xe trưng bày là vật thật; vỏ xe không kính là đồ trưng bày | Hỏi reviewer |
| img_025 | Đường dạo biển; người đi bộ; người ở cuối đường cao khoảng 16 px; xe máy gắn thùng hàng bên hông có người lái; taxi; xe bạc ở mép phải | R-08, R-14, R-13, R-10 | `person` cho người từ 20 px; người khoảng 16 px không vẽ (từ 3 người sát nhau thì 1 `ignore_region`); 1 `motorcycle` bao cả thùng + `person`; taxi: `car`; xe bạc gán theo R-13, `truncated=yes` | Tháo thùng vẫn còn xe máy 2 bánh → không phải `tuk_tuk` | Quy tắc |
| img_026 | 2 người trên 1 xe máy đỏ, bánh trước chạm mép dưới; tranh tường in cảnh cảng biển; ảnh dọc 784×1024 | R-02, R-04, R-10 | 2 `person` + 1 `motorcycle` (`truncated=yes`); tranh không gán; box không vượt 784×1024 | Mỗi người một box | Quy tắc |
| img_027 | Tượng bán tải bằng đá đen trên mộ, nghĩa trang không có người hay xe thật | R-04, R-16 | Không gán; 0 annotation | Phản ví dụ của `truck` | Quy tắc |
| img_028 | Bán tải cổ thật chở tượng ngựa cao hơn nóc cabin; SUV đỏ phía sau | R-04, R-06, R-13, R-11 | `truck`, cạnh trên box ở nóc cabin, không bao tượng; tượng không gán; SUV: `car`, mức che theo phần bị bán tải che | Vật không thật trên vật thật: chỉ gán vật thật | Quy tắc |
| img_029 | Người đi xe máy điện và người đi xe đạp cùng khiêng các ống nhựa dài; người đi bộ, người đạp xe ở xa | R-06, R-14, R-02 | `motorcycle` + `person`; `bicycle` + `person`; ống không thuộc box nào; vật ở xa theo R-08 | Ống nhô ra ngoài thân cả hai xe | Quy tắc |
| img_030 | Bãi xe máy đỗ dày hàng chục xe; bảo vệ áo xanh; poster in người | R-09, R-04 | Xe có `occluded_level` ≤ 1 và cao từ 50 px → `motorcycle`; phần còn lại → `ignore_region`; bảo vệ: `person`; poster không gán | Cụm từ 10 xe chồng nhau | Quy tắc |
| img_031 | Ban đêm, đường ướt: xe buýt số 82; SUV trắng và các ô tô; người đi xe máy; người đi bộ; tranh hoạt hình trên tường | R-13, R-12, R-04, R-01 | `bus`; `car`; `motorcycle` + `person`; người đi bộ: `person`; tranh không gán; bóng trên đường không bao | Vùng tối vẫn vẽ nếu nhận ra lớp | Quy tắc |
| img_032 | Hơn 30 người đứng sát nhau chờ xe buýt; 2 người đứng tách ở bên phải; ô tô xanh ở mép trái; cọc tiêu | R-09, R-10 | Người có `occluded_level` ≤ 1 và cao từ 50 px → `person`; người bị che nhiều → `ignore_region`; 2 người đứng tách: `person`; ô tô: `car`, `truncated=yes`; cọc tiêu không gán | Cụm từ 10 người | Hỏi reviewer |
| img_033 | Ma-nơ-canh; ảnh in người mẫu trên kính; phản chiếu cửa và gương xe | R-04, R-16 | Không gán; 0 annotation | Toàn là vật không thật | Quy tắc |
| img_035 | Tuk-tuk chất bao tải cao gần gấp đôi thân xe; tuk-tuk thứ hai bị người đi xe máy che; người áo trắng đi xe máy; người đi xe máy bên trái; 2 ô tô trắng | R-06, R-14, R-02, R-03, R-11 | Tuk-tuk thứ nhất: box từ bánh tới nóc mui, không bao bao tải; tài xế: `person`; tuk-tuk thứ hai: `tuk_tuk`, mức che theo mục 2.2; mỗi xe máy: `motorcycle` + `person`; 2 `car` | Bao tải cao hơn nóc mui | Hỏi reviewer |
| img_036 | Xe tải nhỏ phủ bạt; 2 cụm xe máy đỗ, mỗi cụm dưới 10 xe; người đứng sau dãy xe | R-13, R-09, R-11 | `truck`; mỗi xe máy nhận ra được: 1 `motorcycle` theo mức che; người: `person` | Cụm dưới 10 xe → vẽ từng xe | Quy tắc |
| img_037 | Bán tải gắn nắp thùng cao; xe buýt bên trái bị cắt mép, xe buýt phía sau bị bán tải che gần hết; người đi bộ | R-13, R-10, R-11 | `truck`; 2 `bus` (`truncated=yes`; `occluded_level=3`); `person` | Cabin vẫn tách khỏi thùng, nắp thùng không biến xe thành `van` | Hỏi reviewer |
| img_038 | Tủ kính đồ chơi và xe mô hình | R-04, R-16 | Không gán; 0 annotation | Đồ chơi là vật không thật | Quy tắc |
| img_040 | Người đội mũ bảo hiểm ở tiền cảnh; taxi có ảnh người dán trên kính; tuk-tuk vàng có tài xế; SUV đen; xe buýt phía sau; 2 gương xe máy ở đáy ảnh | R-10, R-11, R-04, R-03 | `person`, `truncated=yes`; taxi: `car`, `truncated=yes`, mức che theo mục 2.2; ảnh dán không gán; `tuk_tuk` + tài xế `person`; SUV: `car`; `bus` theo mức che; 2 gương không vẽ | Chỉ thấy gương thì không nhận ra được xe | Quy tắc |

**Câu hỏi mang ra họp Pha 2** (ca ghi "Hỏi reviewer" và quy tắc mới cần nhóm chốt):

1. Có thêm lớp `bicycle` không? Nếu không thêm thì xe đạp bị bỏ qua hay gộp vào lớp khác? (img_003, img_006, img_023, img_029)
2. Hàng cao hơn nóc xe hoặc nhô ra ngoài thân xe có bao vào box xe không? (img_035, img_029, img_003)
3. Nhân vật hóa trang: gán `person` + `can_review` hay không gán? (img_019)
4. Vật hình xe nhưng không chắc là xe thật (mô hình phế liệu img_011, vỏ xe trên nóc img_024): quyết định phải nhất quán với xe được chở ở img_002.
5. Ngưỡng cụm R-09 (10 vật, cao 50 px, `occluded_level` ≤ 1) có hợp lý không? (img_019, img_021, img_030, img_032)
6. Bán tải gắn nắp thùng cao: `truck` hay `van`? (img_037)
7. Nhòe tới mức nào thì bỏ không vẽ? (img_014)
8. Người trong xe có cửa kín có cần gán không? (img_009)

Khi gán ở Pha 3: ca chưa chốt thì tick `can_review` trên box. Vật không vẽ được box thì ghi vào `submission/qc/image_issues.csv` (`image,annotator,region,note`) sau khi khóa nhãn. Phiếu QC dẫn đúng mã `R-xx` ở mục 3.

## 5. QC và phiên bản

### 5.1 Ngưỡng QC (lấy từ `taxonomy.json → rules`, không sửa)

- IoU giữa hai box của cùng một vật: ≥ 0.85 với box cao từ 50 px; ≥ 0.7 với box thấp hơn.
- IAA: IoU trung bình ≥ 0.85 và cùng lớp ≥ 95%, tính trên các cặp box ghép 1–1 (IoU ghép ≥ 0.5); không có cặp nào thì ghi N/A.
- Vật nhỏ: box cao dưới 20 px xử lý theo R-08.
- Bàn giao: tỉ lệ ảnh có lỗi ≤ 2% mới được Release. Lô 40 ảnh thì chỉ 1 ảnh lỗi (2,5%) đã vượt ngưỡng.

### 5.2 Tự kiểm trước khi nộp nhãn

1. Task CVAT có đủ ảnh trong `submission/qc/assignments/<mã>.txt`, kể cả ảnh không có vật nào.
2. Không dùng phím `Q`; `occluded_level` và `truncated` chọn trong panel Objects.
3. `python scripts/validate_labels.py submission/labels/<mã>.json` không báo lỗi; xử lý mọi dòng `[WARN]` theo R-08.
4. Mỗi box tick `can_review` khớp đúng một điều kiện a, b hoặc c ở mục 2.2.
5. Script chỉ kiểm cấu trúc; việc box ôm sát vật, đúng lớp và không bỏ sót phải tự soát bằng mắt theo mục 3.1.

### 5.3 Ai quyết ca mơ hồ

- Pha 3: người gán không hỏi nhau. Ca chưa chắc thì tick `can_review` hoặc ghi vào `image_issues.csv`.
- Pha 2 và Pha 5: cả nhóm chốt ở buổi họp, theo đa số khi không thống nhất. Người chốt ghi mã của mình và mã quy tắc vào `calibration_decisions.csv`.
- Quyết định làm đổi nghĩa quy tắc thì tăng phiên bản guideline, ghi `pham_vi=lo` và rà lại cả lô 40 ảnh.

### 5.4 Những câu trong v0 làm hai người hiểu khác nhau và cách sửa

| Câu trong v0 | Hai cách hiểu có thể xảy ra | Sửa ở v0.1 | Ảnh |
|---|---|---|---|
| "Gán box cho mọi người và phương tiện trong ảnh." | Gán cả tượng, người trên poster, hình phản chiếu, xe chủ — hay chỉ vật thật? | Chỉ vật thật; có dấu hiệu nhận biết; không gán xe chủ (R-04, R-07) | img_017, img_018, img_020, img_027 |
| "Box ôm sát vật." | Ôm phần nhìn thấy hay ước cả phần bị che? Có bao người lái, hàng hóa không? | Ôm phần nhìn thấy, lệch tối đa 2 px; người và xe tách box; quy tắc riêng cho hàng hóa (R-01, R-02, R-06) | img_026, img_035 |
| "`person` — Người." | Có tính tượng, ma-nơ-canh, người trong xe không? | Chỉ người thật; người trong xe có cửa kín không gán (R-03, R-04) | img_009, img_033 |
| "`truck` — Xe tải", "`van` — Xe thùng kín cỡ nhỏ." | Bán tải, xe tải thùng kín nhỏ, xe chở ô tô thuộc lớp nào? | Phép thử 4 bước R-13 (cabin tách thùng → `truck`) | img_007, img_012, img_037 |
| "Mỗi vật thuộc đúng một lớp." | Không có lớp cho xe buýt, tuk-tuk, xe đạp: ép vào lớp gần nhất hay bỏ qua? | Thêm `bus`, `tuk_tuk`, `bicycle` (mục 2.1, R-14) | img_004, img_009, img_023 |
| "`occluded_level`: 1 che ít, 2 che nhiều, 3 không rõ." | Không có ngưỡng; "3 không rõ" trộn mức che với độ chắc chắn. | Ngưỡng %, mẹo lưới 3×3; sự không chắc chắn chuyển sang `can_review` (mục 2.2, R-11) | img_005, img_037 |
| "`truncated`: Vật bị cắt ở mép ảnh." | Cắt bao nhiêu thì tính `yes`? | Cạnh box chạm mép (≤ 2 px) và vật còn kéo dài ra ngoài (mục 2.2, R-10) | img_015, img_040 |
| "`ignore_region` — Vùng không cần gán từng vật." | Khi nào được dùng, bao lớn? | Chỉ cho vật dưới 20 px (R-08) và cụm từ 10 vật (R-09) | img_019, img_021, img_025 |
| "Ca mơ hồ: tick `can_review`." | Thế nào là mơ hồ? | Ba điều kiện a, b, c (mục 2.2) | img_014, img_019 |

### 5.5 Lịch sử phiên bản

| Phiên bản | Thay đổi | Lý do (ca nào) |
|---|---|---|
| v0 | Bản khởi đầu của giảng viên: 5 lớp, thuộc tính mô tả bằng lời, 3 quy tắc vẽ box | — |
| v0.1 | Thêm lớp `bus`, `tuk_tuk`, `bicycle`; ngưỡng đo được cho `occluded_level`, `truncated`, `can_review`; thứ tự quyết định 6 bước; quy tắc R-01 đến R-16; bảng ca khó cho 37/40 ảnh | Các câu mơ hồ ở mục 5.4; ca khó ở img_002, img_004, img_009, img_017, img_019, img_024, img_032, img_035, img_037 |
