# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm TFT — Ca học Day 13
- Thành viên: xem `TEAMMATES.md` (Vũ Tuấn Minh - MSSV: 2A202602240).
- Trạng thái: `executed-on-room-LC-machine` (hoặc `provided-results` khi phân tích dữ liệu chuẩn bị sẵn của phòng LC).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nhóm TFT (Vũ Tuấn Minh cùng nhóm); 03/10/2026; x86_64 (Intel Core / AMD64).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lab` (Image ID: `c4a3b890f...`); Repo version: `v1.0-student`.
- PCD được cấp / frame_id: `pointcloud/1772259105-699840307.pcd` (Frame ID: `1772259105-699840307`).
- Checkpoint: PointPillars pretrained KITTI / Robotaxi adapter.
- Phạm vi: `front-window` (cửa sổ phía trước); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Bỏ reflectance thật nguồn KITTI, sử dụng giá trị hằng số kênh thứ 4 theo adapter; `z_ground` được ước lượng từ thuật toán phân đoạn mặt đất cục bộ.


## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 m | 0.16 m | 8 | -0.42 m | `run-A/summary.csv`, `run-A/boxes-*.json`, `run-A/side-*.png` | Mốc đối chiếu (delta=0). Số lượng hộp ít hơn do mặt đất chưa nâng đúng độ cao sensor KITTI. |
| B | 1.73 m | 0.16 m | 12 | 1.28 m | `run-B/summary.csv`, `run-B/boxes-*.json`, `run-B/side-*.png` | Baseline chuẩn (delta=1.73m). Phát hiện thêm các xe ở khoảng xa $x \approx 20-30\text{m}$. |
| C | 1.73 m | 0.32 m | 9 | 1.21 m | `run-C/summary.csv`, `run-C/boxes-*.json`, `run-C/side-*.png` | Pillar lớn hơn (0.32m) làm giảm độ phân giải x-y, gộp một số điểm thưa dẫn tới mất 3 hộp xa. |

- **A/B — chỉ đổi delta:** A có 8 hộp; B có 12 hộp. Ảnh chiếu Side và file JSON tại vùng $x \approx 15-30\text{m}$ khác nhau rõ rệt. Việc đổi delta trước inference là đưa input điểm đã dịch chiều cao vào model để trích xuất đặc trưng mới, nên model dự đoán ra số hộp và vị trí khác hẳn chứ không chỉ dịch cứng các hộp của A. Điều em chưa chắc là liệu mặt đất ước lượng cục bộ có bị sai lệch ở những vùng điểm rất thưa hay không.
- **B/C — chỉ đổi pillar:** B có 12 hộp; C có 9 hộp. Tại vùng $x > 25\text{m}$, kích thước ô pillar tăng gấp đôi (0.32m) khiến các cột điểm bị thô hóa, giảm độ nhạy với xe cỡ nhỏ/xa. Chưa đủ bằng chứng để kết luận C tốt hơn B; B phản ánh chi tiết hình học tốt hơn ở pillar 0.16m.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Góc chiếu Side là hình chiếu phẳng $x-z$ toàn bộ điểm. Do nén chiều $y$, các vật thể ở các tọa độ $y$ khác nhau bị chồng lên nhau trên ảnh Side. Phạm vi `front-window` bỏ qua vùng đằng sau và hai bên vehicle, nên các vật thể ngoài ROI không thể coi là model bỏ sót.
- **JSON nào còn chưa đủ cơ sở để import?** Cả 3 file JSON A/B/C đều là output của thí nghiệm pretrained KITTI demo, **không được phép import** trực tiếp vào job Robotaxi trong CVAT. Cần kiểm tra đúng schema Robotaxi và nạp pre-label từ Portal do LC chuẩn bị sẵn.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 12 | 0 m | Không đổi | Giữ nguyên prediction B | `qc-cases/case-correct.json` trùng khớp với `run-B` |
| case-batch-z | 12 / 12 | 1.73 m | Class, x, y, yaw giữ nguyên | **Dừng batch, kiểm pipeline** | Tất cả 12 hộp đều nảy chìm/nổi cùng một khoảng z exact (1.73m) do thiếu phép cộng z ngược |
| case-one-box-z | 1 / 12 | 1.73 m | 11 hộp đúng, 1 hộp bị chìm z | **Kiểm từng hộp (đối tượng)** | Chỉ có đúng 1 hộp (ID 4) bị lệch, 11 hộp khác bám sát cụm điểm. Đây là lỗi đối tượng cụ thể |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Thành viên: Vũ Tuấn Minh (MSSV: 2A202602240)
- **Vai trò đảm nhận:** Đã đóng góp phân tích cấu hình JSON, kiểm tra bảng đối chiếu A/B/C, xem ảnh Side chiếu ngang và tổng hợp báo cáo. (Trạng thái thực hiện: `executed-on-room-LC-machine` / `provided-results`).
- **Quan sát có bằng chứng:** Tại file `run-B/side-demo.png` và `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, ở khoảng cách $x \approx 20\text{m}$, lượt B phát hiện thêm 4 hộp xe so với lượt A (`run-A/side-demo.png`). Việc điều chỉnh `delta=1.73m` giúp nâng các cột điểm vào vùng cao độ hoạt động tối ưu của mạng PointPillars.
- **Diễn giải phép z:** Phép biến đổi z tuân theo $z_{model} = z_{source} - z_{ground} - \delta$. Việc đổi `delta` trước inference sẽ thay đổi cấu trúc dữ liệu đầu vào trích xuất bởi mạng PointPillars, khác hoàn toàn với việc dịch tọa độ $z$ thủ công sau khi model đã xuất bounding box.
- **Quyết định lỗi batch & Hành động:** Khi gặp trường hợp `case-batch-z` (tất cả các hộp trong batch đều lệch cùng một lượng z = 1.73m), em quyết định **dừng ngay việc sửa tay** và yêu cầu kiểm tra lại ma trận chuyển đổi tọa độ của pipeline, vì đây là lỗi hệ thống pipeline chứ không phải lỗi của từng hình học vật thể đơn lẻ.
- **Điểm còn chưa chắc chắn:** Tại vùng điểm LiDAR bị thưa do khoảng cách xa ($x > 30\text{m}$), mật độ điểm không đủ để xác định chắc chắn phần viền đáy (bottom-z) của cuboid nếu không có hình ảnh camera hỗ trợ.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
