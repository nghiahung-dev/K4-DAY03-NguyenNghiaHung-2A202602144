# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: Nguyễn Nghĩa Hùng / 2A

Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Bổ sung của nhóm (nếu có): `Chỉ gán vehicle thật trong cảnh. Không gán hình xe trên biển quảng cáo, phản chiếu hoặc vật thể tĩnh chỉ có hình dạng giống xe. Xe thật đang dừng/đỗ vẫn được gán.`

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật của nhóm | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che **dưới 25 frame** (mặc định của lab: 25 frame = 2 giây @ 12.5 fps) và vẫn xác định được continuity từ vị trí, hướng di chuyển và appearance | tránh tách một vehicle thành nhiều identity chỉ vì occlusion ngắn |
| Xe bị che lâu hơn ngưỡng trên | mặc định tạo **track mới** nếu không còn đủ bằng chứng chắc chắn để nối identity; chỉ giữ ID khi chuỗi trước/sau rất rõ | sau occlusion dài, nối lại ID dễ trở thành suy đoán |
| Xe rời khung hình rồi quay lại | mặc định: **track mới** | object đã rời hoàn toàn khỏi FOV nên continuity quan sát bị mất |
| Hai xe cắt nhau / chồng lên nhau | nhìn chuỗi frame trước và sau overlap; giữ ID theo hướng chuyển động của từng xe, **không đổi ID chỉ vì bbox giao nhau** | tránh ID switch tại vùng overlap |

**Quy tắc bổ sung sau khi chấm gold:**

- Nếu vehicle **hoàn toàn không nhìn thấy** trong một đoạn nhưng vẫn giữ cùng track qua occlusion ngắn, đặt `outside` cho đoạn invisible; không để bbox interpolation treo trên vùng không có object.
- Không đổi ID chỉ vì bbox thay đổi mạnh. ID dựa trên identity/trajectory, bbox chỉ mô tả vị trí object ở frame đó.

## 3. Luật bbox

| Tình huống | Luật của nhóm |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm phần **nhìn thấy được** |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | bắt đầu track từ frame đầu tiên xác định chắc chắn là xe bốn bánh và có thể đặt bbox ổn định; **không tạo bbox sớm hơn chỉ vì đoán trajectory** |
| Xe đang đỗ, không di chuyển | vẫn gán nếu đó là vehicle thật và còn xuất hiện trong cảnh; nếu bbox gần như đứng im nhiều frame thì kiểm tra bằng mắt để phân biệt xe thật đang đứng yên với bbox treo do quên `outside` |
| Keyframe đặt dày ở đâu | đặt dày hơn khi xe đổi hướng/tốc độ, thay đổi kích thước nhanh, đi sát rìa, overlap/occlusion hoặc khi interpolation bắt đầu lệch; đoạn chuyển động đều có thể đặt thưa hơn |

**Quy tắc start/end/outside sau khi chấm gold:**

- Frame đầu của visible segment phải là frame đầu tiên vehicle thực sự nhìn thấy/đủ xác định.
- Frame cuối của visible segment phải là frame cuối cùng vehicle còn nhìn thấy.
- Ngay khi vehicle hoàn toàn biến mất khỏi đoạn track, đặt `outside` để interpolation không tạo bbox thừa.
- Sau khi đặt keyframe đầu/cuối, luôn tua lại vài frame trước và sau để kiểm tra bbox có bị "treo" hay xuất hiện sớm không.

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1

- Clip / frame / ID: `clip_01 / frame 81–100 / ID 6`
- Tình huống: `annotation có bbox trước khi track tham chiếu ID 6 thực sự xuất hiện; tổng cộng 20 frame bbox thừa.`
- Quyết định: `chỉnh lại visible start/outside của ID 6 để không có bbox trong đoạn vehicle chưa xuất hiện.`
- Lý do: `không được suy đoán trajectory và để interpolation tạo bbox ở nơi chưa có object; đây là nguồn FP lớn nhất trong lỗi pre-gold.`

### Ca 2

- Clip / frame / ID: `clip_01 / frame 94–118 / ID 4`
- Tình huống: `bbox bị drift giữa các keyframe; các frame 94, 95, 114, 115, 116, 117, 118 chỉ có IoU khoảng 0.50–0.55 với gold.`
- Quyết định: `thêm/chỉnh keyframe quanh đoạn xe thay đổi vị trí/kích thước để interpolation bám sát object.`
- Lý do: `keyframe quá thưa ở đoạn chuyển động không tuyến tính làm bbox trôi khỏi xe dù identity vẫn đúng.`

### Ca 3

- Clip / frame / ID: `clip_01 / track ID 4 / cuối track frame 149–151`
- Tình huống: `bbox vẫn còn 3 frame sau khi track tham chiếu đã rời khung; đồng thời track 4 chỉ phủ 75/95 frame của gold.`
- Quyết định: `rà lại toàn bộ vòng đời ID 4: bổ sung đoạn còn thiếu, nhưng kết thúc visible segment đúng frame và đặt outside ngay khi vehicle thực sự biến mất.`
- Lý do: `coverage phải đủ nhưng không được kéo dài bbox quá frame object tồn tại; hai lỗi này cần được kiểm tra cùng nhau bằng cách tua từ đầu đến cuối track.`

### Ca 4

- Clip / frame / ID: `clip_01 / frame 1–15 / ID 3`
- Tình huống: `check_mot_labels.py cảnh báo bbox gần như đứng im trong 15 frame.`
- Quyết định: `không xoá chỉ vì warning; mở visualization để xác nhận đây là vehicle thật đang đứng yên hay bbox treo do quên outside.`
- Lý do: `warning chỉ là tín hiệu bất thường, không phải ground truth. Xe thật có thể dừng/đỗ nên cần quyết định bằng quan sát.`

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:

- `Bổ sung luật start/end/outside: không được để track có bbox trước frame vehicle thực sự xuất hiện hoặc sau frame vehicle đã biến mất. Nếu object hoàn toàn invisible thì dùng outside thay vì để interpolation tạo bbox.`
- `Bổ sung luật keyframe: khi bbox nội suy bắt đầu lệch, đặc biệt ở đoạn xe thay đổi hướng/kích thước hoặc overlap, phải thêm keyframe; không cố dùng ít keyframe nếu làm giảm độ khít.`
- `Bổ sung bước kiểm tra coverage theo từng track: tua từ frame đầu đến frame cuối để phát hiện cả hai loại lỗi đối nghịch — thiếu đoạn và bbox treo/thừa.`
- `Bổ sung quy tắc cho cảnh báo bbox đứng im: vehicle thật đang dừng vẫn được giữ; chỉ sửa khi visualization cho thấy bbox tồn tại ở nơi không còn object.`
- `Phần kiểm chéo với partner chưa có dữ liệu; sau khi review_partner.md hoàn tất cần bổ sung thêm các luật mà hai người quyết khác nhau.`
