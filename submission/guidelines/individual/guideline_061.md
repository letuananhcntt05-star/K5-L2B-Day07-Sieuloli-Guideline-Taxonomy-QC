# Guideline gán nhãn — v0 — 061

- Phiên bản: v0 (đề xuất cá nhân, trước calibration)
- Ngày: 2026-11-16
- Người viết: mã 061
- Phạm vi: bounding box 2D trên 40 ảnh đường phố; taxonomy hiện có gồm `car`, `truck`, `van`, `motorcycle`, `person`, `ignore_region`.

> Đây là bản guideline cá nhân để làm rõ các điểm v0 còn mơ hồ. Các quyết định dưới đây cần được đối chiếu với ảnh gốc khi thao tác gán nhãn; nếu vùng ảnh không đủ rõ để áp dụng quy tắc, dùng `can_review` hoặc ghi vấn đề cấp ảnh thay vì đoán.

## 1. Mục tiêu

Tạo nhãn nhất quán cho mô hình phát hiện người và một số loại phương tiện giao thông trong ảnh 2D. Mô hình cần biết vị trí và lớp của từng đối tượng thuộc phạm vi. Bỏ sót người/xe thật, gán nhầm lớp, hoặc coi tượng/ảnh phản chiếu như đối tượng thật đều làm sai kết quả và có thể gây rủi ro nếu dùng cho phân tích giao thông. Đây là bộ dữ liệu thực hành, không đại diện cho mọi loại phương tiện hay mọi điều kiện đường phố.

## 2. Lớp và thuộc tính

### Lớp

| Lớp | Định nghĩa | Ví dụ trong bộ ảnh / quy tắc nhận diện | Phản ví dụ / dễ nhầm với |
|---|---|---|---|
| `car` | Ô tô chở người có thân xe kín hoặc kiểu wagon/SUV; gồm từng ô tô con riêng biệt đang được chở trên xe chuyên dụng. | Ô tô con nhìn thấy trong cảnh giao thông; từng xe con riêng trên sàn xe chở ô tô ở `img_002.jpg`. | Xe bán tải có thùng hàng riêng xếp `truck`; xe mô hình, xe đồ chơi hoặc xe chỉ xuất hiện trong ảnh in không phải `car` cần gán. |
| `truck` | Xe cơ giới có thùng/chở hàng riêng, gồm xe tải và xe bán tải có thùng hàng. | Xe tải giao hàng; xe bán tải ở `img_001.jpg`, `img_012.jpg`, `img_028.jpg`, `img_036.jpg`, `img_037.jpg`; xe chuyên chở ô tô ở `img_002.jpg`. | Van thùng kín cỡ nhỏ xếp `van`; ô tô con/wagon xếp `car`; xe buýt, tuk-tuk, cyclo, xe kéo không có lớp riêng trong taxonomy này. |
| `van` | Xe chở người/hàng cỡ nhỏ có khoang thân kín liên tục, không có thùng hàng tách biệt như xe tải. | Chỉ gán khi hình dáng khoang kín và kích thước/kiểu thân xe phân biệt được với xe tải. | Pickup có thùng riêng là `truck`; xe con wagon vẫn là `car`. Nếu ảnh không đủ để phân biệt, bật `can_review`. |
| `motorcycle` | Xe hai bánh có động cơ, gồm xe máy/mô tô; một xe là một đối tượng dù chở nhiều người hoặc hàng. | Xe máy chạy/đỗ trong các cảnh giao thông như `img_005.jpg`, `img_008.jpg`, `img_026.jpg`, `img_029.jpg`, `img_030.jpg`, `img_040.jpg`. | Xe đạp, cyclo, xe ba bánh chở khách/tuk-tuk và xe máy đồ chơi không thuộc lớp này. Sidecar gắn liền không tạo thêm box `motorcycle` thứ hai (`img_022.jpg`). |
| `person` | Một người thật, bao gồm người đi bộ, người lái và hành khách trên phương tiện; mỗi người nhìn thấy được là một đối tượng riêng. | Người đi bộ/đứng/bán hàng; người lái và hành khách xe máy. Người hóa trang vẫn là người thật nếu cơ thể là người thật (`img_011.jpg`). | Tượng (`img_017.jpg`, `img_027.jpg`), mannequin (`img_033.jpg`), người in trên billboard (`img_018.jpg`), hình trong gương (`img_020.jpg`) và búp bê/đồ chơi (`img_038.jpg`) không phải `person`. |
| `ignore_region` | Vùng ảnh không gán từng đối tượng thuộc phạm vi, chỉ dùng cho cụm quá nhỏ/dày đặc hoặc vùng không thể phân định đáng tin cậy; đây là nhãn vùng, không phải vật thể. | Một cụm người/xe quá nhỏ để tách box riêng; phần cảnh ngoài taxonomy khi cần ghi rõ vùng không đánh giá. | Không dùng thay cho box của một người/xe rõ ràng và đủ lớn; không phủ lên box hợp lệ chỉ để né quyết định. Vùng ignore bất đồng giữa người gán phải được xem xét ở QC. |

Vật ngoài các lớp trên không được ép vào lớp gần giống nhất. Nếu vật ngoài phạm vi gây vùng không thể đánh giá hoặc nhiều đối tượng quá nhỏ/dày để tách, dùng `ignore_region` cho vùng đó; nếu không, không tạo box. Xe buýt, tuk-tuk/auto-rickshaw, cyclo, xe đạp, xe kéo, động vật, hàng hóa, cone và đồ vật không có lớp riêng. Người thật trên các phương tiện ấy vẫn được gán `person` nếu đủ nhìn thấy và đủ lớn. Với phương tiện lạ chưa chắc phân lớp, đánh dấu `can_review` thay vì tự tạo lớp mới.

### Thuộc tính

| Thuộc tính | Mức cố định | Cách quyết định |
|---|---|---|
| `occluded_level` | `0`, `1`, `2`, `3` | Ước lượng phần diện tích/hình dáng đối tượng bị vật khác che khuất so với toàn bộ đối tượng có thể suy ra: `0` = không hoặc ≤10%; `1` = >10% đến 35%; `2` = >35% đến 70%; `3` = >70% nhưng vẫn nhận ra đối tượng và còn phần nhìn thấy. Không dùng mức 3 cho vật hoàn toàn không nhìn thấy. |
| `truncated` | `no`, `yes` | `yes` chỉ khi đối tượng bị biên ảnh cắt mất một phần; nếu toàn bộ phần nhìn thấy nằm trong ảnh và phần còn lại bị vật khác che thì là `no`. |
| `can_review` | `false`, `true` | Để `false` nếu áp dụng được quy tắc. Chọn `true` khi còn lưỡng lự thực sự về lớp, ranh giới box, số đối tượng hoặc mức thuộc tính; ghi thêm vấn đề vào phiếu QC/ghi chú cấp ảnh nếu không thể tạo box. |

## 3. Quy tắc vẽ box

1. **Một box cho một đối tượng riêng biệt thuộc phạm vi.** Dùng hình chữ nhật thẳng trục ảnh, ôm sát phần đối tượng nhìn thấy; không ước đoán/kéo box ra ngoài khung ảnh để bao phần bị che hoặc bị cắt. Không gộp nhiều người hay nhiều xe thành một box.
2. **Người và xe máy là hai đối tượng khác nhau.** Với người lái/hành khách nhìn thấy được, vẽ một box `person` cho từng người và một box `motorcycle` cho toàn chiếc xe. Ranh giới người không bao gồm xe; ranh giới xe không bao gồm người/hàng hóa. Một người bị che vẫn được gán nếu phần nhìn thấy đủ để nhận biết; gán `occluded_level` theo mức che.
3. **Xe chở xe:** vẽ box `truck` cho xe chuyên chở và một box `car` riêng cho từng ô tô con nhìn thấy, tách biệt trên sàn/giá chở. Không gộp cả đoàn xe thành một box. Chỉ gán chiếc xe con nếu có thể nhận ra đó là một xe riêng; phần khuất không được suy diễn thành box độc lập khi không xác định được đối tượng.
4. **Cắt mép ảnh và che khuất:** vẽ phần còn trong ảnh. Chọn `truncated=yes` khi biên ảnh cắt đối tượng; chọn mức `occluded_level` khi vật khác che. Hai hiện tượng có thể cùng xảy ra: ghi `truncated=yes` và vẫn đánh giá mức che do vật khác.
5. **Ngưỡng kích thước:** chiều cao box đối tượng phải đạt ít nhất 20 px để gán box đối tượng. Nếu thấp hơn 20 px, không vẽ box riêng; trong vùng nhiều vật nhỏ/dày đặc cần loại khỏi đánh giá, vẽ `ignore_region` bao vùng nhỏ gọn và không che các đối tượng lớn có thể gán. Ghi ảnh/vùng nếu có thể làm thay đổi kết quả QC.
6. **Đám đông:** gán từng người phân biệt được, có box cao ≥20 px. Không dùng một box `person` cho cả đám đông. Nếu các cá thể hòa lẫn, không thể tách nhất quán, dùng một `ignore_region` quanh phần cụm không thể gán, tránh phủ lên người/xe rõ ràng.
7. **Vật không phải đối tượng thật trong cảnh:** không gán tượng, mannequin, hình in/ảnh trên biển quảng cáo, phản chiếu trong gương/kính, đồ chơi, mô hình hoặc vật trang trí thành người/xe thật. Không tạo box cho cầu vồng/ảnh phản chiếu trong gương (`img_020.jpg`). Người mặc trang phục hóa trang vẫn là `person` nếu đó là người thật; không gán trang phục/đạo cụ thành lớp khác.
8. **Phương tiện ngoài taxonomy:** không đổi tuk-tuk, cyclo, xe buýt hoặc xe đạp thành `car`, `truck`, `van` hay `motorcycle` chỉ vì hình dáng gần giống. Không có lớp phù hợp thì bỏ box phương tiện đó; chỉ dùng `ignore_region` khi cần xác định vùng không đánh giá được. Gán riêng người thật trên phương tiện nếu đủ điều kiện.
9. **Hàng hóa và vật gắn trên xe:** không gán hàng, bao tải, thùng, sidecar hay phụ kiện thành phương tiện/người. Box xe chỉ bao phần cấu thành xe nhìn thấy. Nếu hàng che xe, áp dụng quy tắc che khuất; không lấy cả khối hàng làm box xe.
10. **Ảnh không có đối tượng thuộc lớp:** vẫn phải có ảnh trong COCO với danh sách annotation rỗng; không tạo box giả để tránh ảnh rỗng.
11. Không chồng `ignore_region` lên một box đối tượng hợp lệ. Nếu vùng không chắc chắn làm thay đổi lớp/số lượng/ranh giới box, đặt `can_review=true` trên box liên quan hoặc ghi rõ ảnh, vị trí và lý do trong `image_issues.csv`.

## 4. Ca khó

| Ảnh | Tình huống cần xử lý | Rule ID | Quyết định ban đầu | Lý do |
|---|---|---|---|---|
| `img_002.jpg` | Xe tải chở nhiều ô tô | `R-03` | Một box `truck` cho xe chở và một box `car` cho từng ô tô riêng nhìn thấy được trên giá; không gộp các xe con vào box truck. | Mỗi phương tiện phân biệt được là một đối tượng. Đây là ca cần đối chiếu kỹ box xe con bị che/khuất. |
| `img_005.jpg` | Người đi xe máy trong cảnh giao thông, có khả năng che khuất/cắt mép | `R-02`, `R-04` | Mỗi người nhìn thấy được một box `person`, mỗi xe máy một box `motorcycle`; chỉ phần nhìn thấy, gắn thuộc tính theo ngưỡng. | Tách người khỏi phương tiện để nhất quán số box. |
| `img_008.jpg`, `img_030.jpg`, `img_040.jpg` | Nhiều xe máy/người trong cảnh phố | `R-01`, `R-06` | Gán riêng từng người và xe máy đủ rõ, cao ≥20 px; cụm quá nhỏ/dày không tách được thì `ignore_region`. | Tránh gộp nhiều cá thể hoặc bỏ qua có chọn lọc. |
| `img_011.jpg` | Người hóa trang cạnh xe máy trang trí | `R-07` | Gán `person` nếu quan sát được đó là người thật; xe máy thật gán `motorcycle`. Không gán tượng/đạo cụ/trang trí. | Trang phục không làm thay đổi lớp người thật. |
| `img_017.jpg` | Tượng người cưỡi ngựa | `R-07` | Không gán `person` cho tượng; không có lớp ngựa. Không tạo box cho tượng. | Phạm vi là người thật và phương tiện thuộc taxonomy. Hai người cùng gán tượng vẫn có thể cùng sai. |
| `img_018.jpg` | Billboard có hình người | `R-07` | Không gán người in trên billboard. | Hình in không phải người thật trong cảnh. |
| `img_019.jpg` | Xe diễu hành/xe mang hình thức đồ chơi hoặc trang trí | `R-07`, `R-08` | Chỉ gán `car` nếu là ô tô thật thuộc phạm vi; nếu là mô hình/đạo cụ diễu hành thì không gán xe. Người thật đi cùng vẫn gán riêng. Đánh dấu `can_review` nếu không phân biệt chắc chắn. | Không dùng hình dáng giống xe để suy ra đối tượng thật. |
| `img_020.jpg` | Hình phản chiếu/cầu vồng trong gương xe | `R-07` | Không gán box cho hình phản chiếu; chỉ gán xe thật nếu nó xuất hiện rõ trong ảnh và thuộc lớp. | Phản chiếu không phải một đối tượng độc lập trong cảnh. |
| `img_022.jpg` | Xe máy có sidecar | `R-08` | Một box `motorcycle` cho xe máy và phần sidecar gắn liền; người thật gán riêng nếu thấy được. | Sidecar không phải xe máy thứ hai. |
| `img_026.jpg` | Hai người trên một xe máy | `R-02` | Một box `motorcycle` cho xe; một box `person` cho từng người nhìn thấy được. | Số người và số xe là hai phép đếm riêng. |
| `img_027.jpg` | Tượng/monument có hình xe bán tải | `R-07` | Không gán tượng/xe tượng thành `truck`; chỉ gán phương tiện thật khác trong cảnh nếu có. | Vật trưng bày không phải phương tiện đang tham gia giao thông. |
| `img_029.jpg` | Xe máy chở hàng cồng kềnh | `R-02`, `R-09` | Một box `motorcycle` ôm phần xe nhìn thấy; gán người thật riêng. Không gán hàng hóa; nếu hàng che xe thì gán thuộc tính che khuất. | Hàng hóa không phải bộ phận của box xe. |
| `img_033.jpg` | Mannequin trong không gian phố/cửa hàng | `R-07` | Không gán mannequin thành `person`; người thật ở ngoài mannequin vẫn gán nếu đủ điều kiện. | Hình người giả không phải người thật. |
| `img_035.jpg` | Xe ba bánh/auto-rickshaw chở tải | `R-08`, `R-09` | Phương tiện không có lớp riêng thì không ép vào `truck`/`motorcycle`; gán người thật nếu có. Không gán bao tải/hàng hóa. | Taxonomy hiện tại không có lớp tuk-tuk/auto-rickshaw; cần ghi lại nếu nhóm quyết định mở taxonomy ở phiên sau. |
| `img_038.jpg` | Cửa sổ trưng bày có đồ chơi/búp bê/mô hình xe | `R-07` | Không gán các món đồ chơi, búp bê hoặc xe mô hình thành đối tượng thật. | Vật trưng bày không phải đối tượng giao thông thật. |
| `img_001.jpg`, `img_012.jpg`, `img_028.jpg`, `img_036.jpg`, `img_037.jpg` | Bán tải/xe có thùng hàng riêng | `R-01` | Gán `truck` cho xe bán tải có thùng hàng riêng; người thật bên cạnh gán `person` riêng. | Khớp định nghĩa truck trong taxonomy v0. |

## 5. QC và phiên bản

- Box có chiều cao ≥50 px: ngưỡng IoU lớn `0.85`; box nhỏ hơn 50 px nhưng đạt ngưỡng kích thước tối thiểu: IoU `0.70`. Ghép cặp IAA dùng `match_iou=0.5`; chỉ tính IoU trung bình và tỷ lệ cùng lớp trên cặp ghép 1–1. Không có cặp thì báo N/A; box không ghép và bất đồng `ignore_region` phải xem riêng.
- Mục tiêu IAA theo cấu hình lab: IoU trung bình ≥`0.85`, cùng lớp ≥`95%`; phải đủ 100% lượt gán theo assignment mới đo. Báo tử số/mẫu số, không chỉ ghi phần trăm.
- QC lô: tỷ lệ ảnh có ít nhất một lỗi ≤`2%` mới có thể Release, đồng thời phải duyệt toàn bộ lô, không có lỗi lặp hệ thống và có đủ bằng chứng. Với 40 ảnh, `1/40 = 2.5%` đã vượt ngưỡng. Người review nên khác người sửa; quyết định Hold/Rework/Release theo bằng chứng, không chỉ theo điểm IAA.
- Ca không chắc: annotator quyết định độc lập trước, bật `can_review` hoặc ghi `image_issues.csv`; reviewer ghi ảnh/vùng, rule ID, quyết định và kết quả kiểm lại. Không thay nhãn độc lập trước khi lưu bản gốc và đo IAA.
- Mã quy tắc: `R-01` lớp phương tiện và ranh giới xe; `R-02` người/xe máy; `R-03` xe chở xe; `R-04` che khuất/cắt mép; `R-05` ngưỡng kích thước; `R-06` đám đông; `R-07` vật không thật; `R-08` phương tiện ngoài taxonomy/sidecar; `R-09` hàng hóa.

| Phiên bản | Thay đổi | Lý do (ca nào) |
|---|---|---|
| v0 | Bản cá nhân làm rõ định nghĩa lớp, ngưỡng thuộc tính, cách tách người/xe, xe chở xe, vật không thật, vật nhỏ và vật ngoài taxonomy. | Các ca khó nêu ở Mục 4; v0 chưa quy định số box và cách xử lý các tình huống này. |
