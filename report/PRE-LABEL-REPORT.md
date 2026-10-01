# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Yêu Anh Vương Lắm
- Thành viên: xem `TEAMMATES.md` (họ tên và MSSV đã chuyển sang file này; vai trò từng lượt đã xoay).
- Trạng thái: `executed-by-group`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: nhóm chạy trên máy Mac Apple Silicon, Docker Linux/arm64, 2026-10-01 ~14:54–14:55 ICT. Lượt A theo dõi lệnh; lượt B và lượt C theo dõi từng lượt; vai trò chi tiết và MSSV xem trong `TEAMMATES.md`.
- Image tag và image ID; phiên bản repo: tag `day13-pointpillars:lc-20261001-arm64`; image ID `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1`; `repo_revision` `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd`, `frame_id=demo`, 17238 điểm, SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60` (gói Student KITTI, không phải Robotaxi)
- Checkpoint: `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance nguồn bị bỏ, adapter dùng kênh hằng, RGB=0; `z_ground` ước lượng từ PCD = **0.075 m** (không lấy đường z=0 trên ảnh Side làm mặt đường)

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | ---: | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ 1 hộp `vehicles` tại (13.15, −0.45, 0.33), score 0.322. `rear=0`. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 `vehicles` + 1 `two-wheels` + 2 `pedestrian`. z hộp từ 0.70 đến 1.43, không phải cùng một lượng. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | 6 hộp, **toàn `pedestrian`**, không còn `vehicles`. Score cao nhất 0.808. |

- A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao? **Có khác.** A không hạ PCD (`delta=0`) nên input không giống KITTI (mặt đường model quen ~−1.73 m); chỉ còn 1 hộp. B hạ `delta=1.73` trước inference rồi cộng ngược `z_source = z_model + z_ground + delta` nên ra 13 hộp, class đa dạng. Đây là model chạy lại trên input đã dịch, **không** phải dịch mọi hộp A thêm 1.73 m (1 hộp không thể thành 13 hộp bằng phép tịnh tiến).
- B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không? Giữ `delta=1.73`, chỉ đổi pillar 0.16 → 0.32. Số hộp 13 → 6, class đổi hẳn (mất hết xe, chỉ còn pedestrian). Ô lưới to gấp đôi làm model xem dữ liệu kiểu checkpoint 0.16 m chưa quen. **Chưa đủ bằng chứng** để nói C tốt hơn hay kém hơn: không có nhãn đúng, không import CVAT, nhiều hộp hơn (B) cũng không tự là đúng hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Cả ba lượt `rear=0` (chỉ cửa sổ phía trước, score 0.3). Ảnh Side là chiếu toàn scene x–z, xe khác y có thể chồng. Đường `z=0` trên ảnh là tham chiếu plot, không phải mặt đường cục bộ. Không dùng Side một mình để chốt miss/yaw; cần Top/Front/camera khi sang Phần 2.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không import A/B/C hay `case-*.json` vào CVAT/Robotaxi. PCD này là KITTI demo khác frame. Cần LC/operator import prediction đúng job Robotaxi riêng.

## Ca QC có kiểm soát — không import CVAT

Helper tạo biến đổi có chủ đích từ prediction lượt B (`boxes-demo-delta-1.73-voxel-0.16.json`, SHA256 `51f49ac6ba8c97f8a2458e289fee48a33e1d07dd94de99ba935c142fd7e26016`). Không chạy lại detector. Lượng lệch = `delta + z_ground` = **1.805 m**. Class/x/y/yaw/kích thước **không đổi**.

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không; z trùng B | Bản đối chiếu, không phải hộp đúng | `qc-cases/case-correct.json`, `side-correct.png` |
| case-batch-z | 13 / 13 | mọi hộp z −1.805 m | Không | **Dừng sửa tay**, báo LC kiểm transform/pipeline | `case-batch-z.json`; manifest `height_offset_m=1.805` |
| case-one-box-z | 1 / 13 | chỉ hộp đầu z −1.805 m | Không | **Kiểm từng hộp** qua nhiều view, không kết luận lỗi pipeline | `case-one-box-z.json`; các hộp 1–12 giữ z của B |

Quy tắc: cả loạt hộp cùng nổi/chìm một lượng → lỗi pipeline → dừng, gọi LC. Một hộp lệch → sửa riêng hộp đó.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc.

### Thành viên 1
- Vai trò: A theo dõi lệnh · B đọc cấu hình/số liệu · C xem ảnh Side.
- Quan sát A/B/C: A chỉ 1 hộp `vehicles` (`run-A/summary.csv`, mean_z 0.330). B cùng pillar 0.16 nhưng `delta=1.73` ra 13 hộp (`run-B/boxes-demo-delta-1.73-voxel-0.16.json`). Số hộp đổi nên đây không phải dịch mọi hộp sau inference.
- Phép z: trước model `z_model = z_source − z_ground − delta`; sau khi có hộp `z_source = z_model + z_ground + delta`. `z_ground` ước lượng = 0.075 m.
- Quyết định batch: `case-batch-z` mọi hộp z −1.805 m → dừng sửa tay, báo LC kiểm pipeline.
- Chưa chắc: ảnh Side chồng xe khác y nên chưa chốt miss/yaw từng hộp chỉ từ Side.

### Thành viên 2
- Vai trò: A đọc cấu hình/số liệu · B xem ảnh Side · C ghi chép.
- Quan sát A/B/C: C giữ `delta=1.73` nhưng pillar 0.32 → 6 hộp, toàn `pedestrian` (`run-C/summary.csv`). Đổi biểu diễn input, không phải quên cộng z ngược.
- Phép z: hạ PCD cho giống KITTI rồi cộng ngược để hộp về hệ nguồn. Đổi delta là đổi input trước model.
- Quyết định batch: cả loạt cùng lệch một lượng → lỗi pipeline, không kéo tay từng hộp.
- Chưa chắc: chưa đủ nhãn để nói B (13 hộp) tốt hơn C (6 hộp).

### Thành viên 3
- Vai trò: A xem ảnh Side · B ghi chép · C theo dõi lệnh.
- Quan sát A/B/C: ảnh `run-A/side-demo-delta-0-voxel-0.16.png` chỉ còn 1 hộp xe trong cửa sổ trước; ảnh `run-B/side-demo-delta-1.73-voxel-0.16.png` có 13 hộp trải dọc cửa sổ trước (`rear=0`), với các hộp ở vùng gần người quan sát và z khoảng 0.70–1.43 m. Số lượng tăng rõ rệt khi đổi `delta` mà không đổi pillar, nên xu hướng là về mặt hình học bên trái/đúng không phải một bản dịch đơn giản của cùng một dự đoán.
- Phép z: `delta=1.73` là giả định cao độ sensor KITTI trong bài, không áp cho mọi sensor. Cộng ngược khác với dịch mọi hộp một lượng cố định.
- Quyết định batch: `case-one-box-z` chỉ 1/13 hộp lệch → kiểm hộp đó nhiều góc, không kết luận lỗi pipeline.
- Chưa chắc: đường z=0 trên Side chỉ là tham chiếu plot, chưa phải mặt đường cục bộ.

### Thành viên 4
- Vai trò: A ghi chép · B theo dõi lệnh · C đọc cấu hình/số liệu.
- Quan sát A/B/C: checkpoint, score 0.3 và ROI giữ nguyên; chỉ đổi một biến giữa các lượt. Image ID `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1`.
- Phép z: quên phép ngược thì mọi hộp cùng chìm `delta + z_ground` = 1.805 m (`qc-cases/manifest.json`).
- Quyết định batch: thấy cả loạt nổi/chìm cùng lượng → dừng, gọi LC. Một hộp lệch → sửa riêng.
- Chưa chắc: JSON KITTI demo không đủ cơ sở import CVAT/Robotaxi; cần LC import đúng frame.

## LC ghi nhận riêng

> LC ghi nhận ngày 01/10/2026. **Kết luận: ĐẠT.**

- **Quyền dùng PCD/image và đúng ca:** Gói Student KITTI 000008 (giấy phép CC BY-NC-SA 3.0), không dùng dữ liệu Robotaxi. Input SHA-256 `3b5ea3da…` và image bản arm64 `sha256:dd6999ad…` khớp `smoke.json`.
- **Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:** Đã nhận `smoke.json`: passed, Mac arm64, chạy 14:54–14:55 (khớp giờ trong báo cáo); nạp image 23,8 s, A/B/C 25,0 / 16,1 / 7,7 s, kết quả 1/13/6.
- **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** Đủ `run-A/B/C` và `qc-cases`. Không sửa JSON. Không đưa ca lỗi vào CVAT.
- **Nhận xét từng thành viên và quyết định dừng pipeline:** Đủ 4 người, đủ 5 ý, hiểu đúng phép z và quyết định batch/one-box. Quan sát Side A/B đã được bổ sung với số hộp và vị trí cụ thể.
- **Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:** **Đồng ý chuyển sang chỉnh/QC.** Đã bổ sung đủ `smoke.json` và output.