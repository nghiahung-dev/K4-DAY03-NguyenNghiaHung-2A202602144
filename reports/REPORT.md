# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: Nguyễn Nghĩa Hùng / 2A
Ngày: 15/09/2026

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) | 5 phút |
| Thời gian gán `clip_01` | 10 phút |
| Số track đã vẽ trong `clip_01` | 8 |
| Số keyframe trung bình mỗi track | 10 |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. **Xe xuất hiện/rời khung hình:** cần đặt đúng frame bắt đầu/kết thúc và dùng trạng thái `outside` đúng chỗ. Gold cho thấy tôi để bbox thừa ở ID 6 (frame 81–100), ID 5 (73–78) và ID 4 (149–151).
2. **BBox bị trôi giữa hai keyframe:** interpolation có thể làm bbox lệch khi xe đổi vị trí/kích thước nhanh. Gold phát hiện nhiều frame quanh 94–118, đặc biệt track 4, có IoU chỉ khoảng 0.50–0.55.
3. **Giữ đủ coverage của một track:** không kết thúc track quá sớm hoặc bỏ mất một đoạn khi xe vẫn còn thuộc cùng identity. Track gold 4 dài 95 frame nhưng annotation của tôi mới phủ 75 frame (79%).

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

- Lượt 1: Kiểm tra continuity của ID; file hiện có 8 track với ID `[1,2,3,4,5,6,7,8]`. So với gold, pre-gold có `IDSW = 0`, nên phần giữ identity nhìn chung tốt.
- Lượt 2: Cần chú ý hơn frame đầu/cuối. Gold phát hiện bbox thừa ở ID 6 frame 81–100, ID 5 frame 73–78 và ID 4 frame 149–151.
- Lượt 3: Cần thêm keyframe ở các đoạn bbox nội suy bị trôi, nhất là track 4 quanh frame 94–118 và track 6 quanh frame 106.

Ngoài ra, `check_mot_labels.py` cảnh báo **track 3 gần như đứng im từ frame 1 đến 15**. Cảnh báo này cần kiểm tra bằng mắt để xác định đó là xe thật đang đứng yên hay track chưa được đặt `outside` đúng.

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | `5d9922b04616506520ae499ce314b5837afe1dd8c7c518e89cea398f1eadff67` |
| Thời điểm khóa | 11h |
| Số row / frame / track trước khi mở reference | `597 row / 190 frame / 8 track` |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | 0.733 | 0.714 | 0.753 | 0.834 | 0.923 | 0.843 | 0.816 | 57 | 33 | 0 |
| Sau rework |  |  |  |  |  |  |  |  |  |  |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có**  
- IDF1 = **0.923** >= 0.80
- MOTA = **0.843** >= 0.75
- MOTP = **0.816** >= 0.70

Sau khi đọc danh sách lỗi, bạn đã sửa cụ thể những gì? Ghi theo frame và ID:

> Chưa có output đánh giá **sau rework**, nên bảng dưới ghi các sửa cần thực hiện từ danh sách lỗi của pre-gold. Sau khi sửa xong cần chạy lại evaluation để xác nhận.

| Loại lỗi | Frame | ID | Đã sửa thế nào |
| --- | --- | --- | --- |
| Bbox thừa / sai thời điểm xuất hiện | 81–100 | 6 | Cần chỉnh lại frame bắt đầu/`outside` để không tạo bbox khi vehicle chưa xuất hiện trong reference. |
| Bbox thừa / sai thời điểm xuất hiện | 73–78 | 5 | Cần chỉnh lại frame bắt đầu/`outside`, tránh cho track tồn tại sớm hơn object thật. |
| Bbox treo sau khi xe đã rời | 149–151 | 4 | Cần kết thúc visible segment đúng frame và đặt `outside` ngay khi xe không còn xuất hiện. |
| Bbox trôi | 114–118, 94–95 | 4 | Cần thêm/chỉnh keyframe quanh đoạn này để interpolation bám sát xe. |
| Bbox trôi | 106 | 6 | Cần thêm keyframe gần frame 106 và chỉnh bbox cho khít. |
| Thiếu đoạn | track gold 4 chỉ phủ 75/95 frame | 4 | Cần rà lại toàn bộ vòng đời track 4, bổ sung đoạn bị thiếu và giữ cùng ID nếu identity vẫn liên tục. |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | `3.13.15 / 8.4.145 / 2.11.0+cpu / 0.5.13` |
| weights / hai tracker | `yolo26n.pt / bytetrack.yaml / configs/trackers/botsort-reid.yaml` |
| conf / IoU / imgsz / classes | `0.25 / 0.70 / 960 / [2, 5, 7]` |
| device | `cpu` |

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.733 | 0.714 | 0.753 | 0.834 | 0.923 | 0.843 | 0.816 | 57 | 33 | 0 |
| ByteTrack control vs gold |  | | | | | | | | | |
| BoT-SORT + ReID vs gold |  | | | | | | | | | |
| ReID vs bạn | 0.662 | 0.608 | 0.722 | 0.821 | 0.849 | 0.688 | 0.798 | 113 | 72 | 1 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

`MOTA của tôi thấp hơn IDF1: MOTA = 0.843, IDF1 = 0.923. Điều này phù hợp với kết quả IDSW = 0: phần identity của tôi khá tốt, trong khi phần detection/coverage vẫn còn FP=57 và FN=33. Nếu có trường hợp MOTA cao nhưng IDF1 thấp thì thường tracker vẫn phát hiện được nhiều object nhưng identity bị nối/tách sai. MOTA chỉ trừ ID switch như một lỗi rời rạc trong công thức cùng FP và FN, còn IDF1 đo tính nhất quán identity trên toàn chuỗi nên nhạy hơn với việc một object bị chia thành nhiều ID hoặc đổi ID.`

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

`Chưa thể so sánh định lượng ByteTrack control và BoT-SORT + ReID với gold vì notebook trước đó được chạy khi chưa có gold. Cần chạy lại để lấy IDF1, AssA và IDSW cho hai tracker. Tuy nhiên, ở phép ReID vs annotation của tôi, ReID có IDF1=0.849, AssA=0.722 và IDSW=1. Một sequence đáng chú ý là quanh frame 87: track ID 5 của tôi được model ghép với ID 17 rồi chuyển sang ID 18. Sau khi có gold, annotation pre-gold của tôi có IDSW=0, nên evidence hiện tại ủng hộ việc continuity ID của tôi ở sequence này tốt hơn model. Dù vậy đây vẫn là system comparison; ByteTrack và BoT-SORT khác implementation/association, nên không được quy toàn bộ chênh lệch chỉ cho ReID.`

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

`Chưa thể so trực tiếp DetA/FP/FN giữa ByteTrack và ReID với gold cho tới khi rerun model evaluation. Với annotation của tôi so với gold: DetA=0.714, FP=57, FN=33, trong khi AssA=0.753 và IDSW=0. Vì không có ID switch nhưng vẫn có nhiều FP/FN, lỗi pre-gold còn lại chủ yếu nằm ở detection/coverage và localization hơn là identity association. Danh sách lỗi xác nhận điều này: bbox thừa ở ID 5/6/4, bbox drift ở track 4/6 và thiếu coverage của track 4.`

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

`Quanh frame 87, annotation của tôi giữ vehicle ở ID 5, trong khi ReID treatment chuyển association từ model ID 17 sang ID 18. Evaluation của annotation với gold cho IDSW=0 trên toàn clip, vì vậy evidence hiện có ủng hộ việc tôi giữ identity liên tục đúng hơn model ở đoạn này. Đây là ví dụ cho thấy model disagreement chỉ nên dùng để gợi ý kiểm tra, không phải coi model là ground truth.`

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

`Các frame 104–110 là vùng ReID và annotation bất đồng mạnh; riêng frame 107 model có nhiều bbox chỉ xuất hiện ở phía model. Việc này khiến tôi xem lại annotation thay vì tin ngay model. Sau khi chấm với gold, đúng là annotation có vấn đề quanh vùng này: track 6 bị bbox drift ở frame 106 và track 4 có coverage chưa đủ. Vì vậy disagreement của model là tín hiệu diagnostic hữu ích, nhưng vẫn phải dùng gold/quan sát bằng mắt để quyết định sửa gì.`

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

`Tôi sẽ bổ sung bốn luật rõ ràng: (1) frame bắt đầu/kết thúc visible segment phải khớp frame đầu/cuối object thực sự xuất hiện; (2) dùng outside ngay khi object hoàn toàn không còn nhìn thấy, tránh bbox treo do interpolation; (3) với occlusion ngắn, giữ cùng ID nếu identity vẫn xác định được nhưng không tạo bbox ở frame object hoàn toàn invisible; (4) tăng mật độ keyframe khi xe đổi tốc độ, kích thước, hướng hoặc gần vùng overlap để tránh bbox drift. Về workflow, sau khi annotate tôi sẽ luôn chạy check_mot_labels.py rồi visualize theo ba pass: ID continuity, start/end/outside, và bbox interpolation ở frame giữa. Tôi cũng sẽ ưu tiên review các đoạn mà bbox gần đứng im bất thường hoặc coverage của track thấp.`

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt`
- [ ] `annotations/clip_02/gt.txt`
- [x] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json`
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json` — hiện có `outputs/eval_pre_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt`
- [x] `outputs/model_reid_clip_01.txt`
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json` — `eval_reid_vs_me.json` đã có; hai file vs gold cần rerun
- [x] `reports/review_partner.md`
- [x] `reports/REPORT.md` (file này)
