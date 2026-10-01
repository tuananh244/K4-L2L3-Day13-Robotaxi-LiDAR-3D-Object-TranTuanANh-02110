# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng:
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group` / `executed-on-room-LC-machine` / `provided-results`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Tuấn Anh; 2026-10-01; Windows amd64
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lab`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: KITTI Pretrained
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: RGB=0, z_ground=0.075m

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A` | Chỉ bắt được 1 hộp (vehicles=1) |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B` | Tăng đột biến (vehicles=10, ped=2, two-wheels=1) |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C` | Chỉ bắt được 6 người đi bộ, mất toàn bộ ô tô |

- A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao? **Khác biệt hoàn toàn.** Phép dịch trục Z trước inference thay đổi hẳn cấu trúc không gian mà mạng nhìn thấy, khớp với bộ dữ liệu huấn luyện, từ đó thay đổi số lượng nhận diện (1 lên 13). Việc dịch hộp sau inference chỉ làm dời tọa độ của 1 hộp đã được nhận diện chứ không sinh ra được 12 hộp bị bỏ sót.
- B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không? **Cấu hình B tốt hơn.** Đổi pillar từ 0.16 lên 0.32 làm nén cấu trúc vật thể lớn (ô tô) quá mức, dẫn đến mạng không còn nhận ra chiếc xe nào nữa.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Góc nhìn Side chỉ là hình chiếu x-z, các xe có thể bị xếp chồng lấp lên nhau theo trục y. Hơn nữa do giới hạn chỉ tính ROI phía trước, ta không lấy dữ liệu hộp phía sau để đánh giá độ chính xác tổng thể.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? JSON của lượt A và C sai lệch quá nặng về số lượng/vật thể, không đủ cơ sở dùng. Lượt B cũng chỉ là gợi ý, cần QC bằng tay.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không | Tiếp tục import bình thường | Giữ nguyên từ lượt B |
| case-batch-z | 13 / 13 | -1.805 m | Không | **Dừng batch**, báo LC lỗi pipeline | Cả 13 hộp đều bị chìm xuống cùng 1 lượng z |
| case-one-box-z | 1 / 13 | -1.805 m | Không | **Kiểm từng hộp** | Chỉ có đúng 1 hộp bị trượt cao độ, các hộp khác bình thường |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
