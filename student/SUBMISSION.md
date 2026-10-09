# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Báo cáo CP6. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Nguyễn Văn Diện
- MSSV: 02615
- Email: diendls321@gmail.com
- Link repo (fork): https://github.com/diendls321-commits/K4-L2L3-DAY23-NguyenVanDien-02615-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): lấy SHA 40 ký tự của commit CP6 sau khi commit và push; đối chiếu tại [commit mới nhất trên main](https://github.com/diendls321-commits/K4-L2L3-DAY23-NguyenVanDien-02615-SensorFusion/commits/main). SHA được gửi cùng link repo trong hội thoại để học viên nộp LMS. File nằm trong chính commit CP6 nên không ghi SHA của chính nó trước khi commit.

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: compare; [0, 198] (199 frame mỗi mode); training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord; seed = 0.
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: 0.9700934579439252; 0.7004048582995951; tp = 519, fp = 16, fn = 222.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.15032268781360134 m; 502; 11.343649056695735 m²; 0; 239; 2.522613065326633.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.1358667883353908 m; 502; 9.26681165463209 m²; 0; 239; 2.522613065326633.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: Fused giảm RMSE 0.014455899478210549 m (khoảng 9.62%) so với LiDAR, đồng thời số cặp ghép đều là 502, ghost đều bằng 0 và miss đều là 239. Trong lần chạy này camera cải thiện sai số vị trí của các cặp được ghép, không tăng số lượng cặp ghép hoặc giảm ghost/miss. Cả hai mode có precision_track = 1.0 và coverage = 502/519 = 0.9672447013487476 theo rubric; RMSE đều dưới 0.45 m và chênh lệch fused − lidar nhỏ hơn 0.05 m. Log có tổng 741 lượt GT hợp lệ; detector có 222 FN, còn tracker có 239 miss. Đây là hai phép đánh giá khác nhau, nên các tổng này không đủ để xác định từng miss thuộc nguyên nhân nào. Camera lấy tâm hộp ground-truth FRONT thêm nhiễu theo seed; kết quả chỉ phản ánh thử nghiệm fusion trong lab.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?

   LiDAR đo tâm hộp 3D: `z` có kích thước 3×1, đơn vị mét; mặc định `R = diag(0.1², 0.1², 0.1²)` m². Mô hình đo tuyến tính lấy vị trí từ trạng thái 6 chiều `[x, y, z, vx, vy, vz]`, với `H` kích thước 3×6 sau đổi hệ tọa độ. Camera đo tâm hộp ảnh: `z = [u, v]ᵀ` có kích thước 2×1, đơn vị pixel; `R = diag(5², 5²)` pixel². Hàm chiếu pinhole `h(x)` phi tuyến nên EKF dùng Jacobian `H` kích thước 2×6. Cả hai cập nhật cùng trạng thái, nhưng innovation và covariance phải được tính theo mô hình và đơn vị của từng sensor; không ghép trực tiếp tọa độ pixel với tọa độ mét.

2. Vì sao cần gating Mahalanobis trước khi gán?

   `d² = γᵀ S⁻¹ γ`, với `γ = z − h(x)` và `S = HPHᵀ + R`, đo độ bất thường của residual theo độ bất định và tương quan giữa các chiều. Khoảng cách Euclidean không xét covariance: cùng một residual có thể hợp lý với track chưa chắc chắn nhưng rất bất thường với track đã ổn định. Loại cặp ngoài FOV trước khi chiếu, rồi chỉ giữ cặp có `d² ≤ chi2.ppf(0.995, dim_meas)` (3 chiều cho LiDAR, 2 cho camera). Các cặp bị loại có cost vô cùng nên bước gán greedy không chọn chúng. Gating giảm ghép sai nhưng không bảo đảm mọi cặp trong gate đều đúng.

3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.

   Pipeline là **track-then-fuse**: mỗi frame predict một lần cho các track hiện có, association/update LiDAR và quản lý vòng đời, rồi association/update camera trong mode fused. Thứ tự này thể hiện trực tiếp trong [run_lab.py](../platform/fusion_lab/scripts/run_lab.py), các lệnh `KF.predict`, `associate_and_update(..., lidar_sensor)` và `associate_and_update(..., camera_sensor)`; association không predict thêm.

   Trong [grade_run.log](artifacts/grade_run.log), cả hai mode có `confirmed = 0` ở frame 0–3, đến frame 4 có `confirmed = 2`, `matches = 2`, `ghosts = 0`, `misses = 0`. Ở frame 4, `sum_sq_err` là 0.011281520740472877 m² cho LiDAR và 0.031987316784416 m² cho fused: camera có thể làm sai số một frame tăng, dù RMSE toàn segment giảm từ 0.15032268781360134 xuống 0.1358667883353908 m. Log ghi thống kê đánh giá theo frame, không ghi thứ tự lời gọi predict/update; vì vậy dùng mã runner để xác nhận thứ tự, và log để đối chiếu kết quả hai mode.

4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?

   Sai extrinsics hoặc intrinsics làm `h(x)` lệch so với tâm ảnh đo được. Innovation camera có thể có độ lệch có hệ thống theo vị trí hoặc hướng nhìn thay vì dao động quanh 0; Mahalanobis tăng và nhiều cặp bị gate loại. Nếu residual vẫn lọt gate, update có thể kéo trạng thái về vị trí sai và làm RMSE fused tăng. Đây là phân tích từ mô hình đo; tôi không thực hiện thử nghiệm thay calibration vì chỉ làm phần bắt buộc.

5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Giải thích vì sao lidar quyết định score/init/delete còn camera chỉ EKF update.

   Khi danh sách đo rỗng, không thể lấy `meas_list[0].sensor` để biết lượt xử lý thuộc LiDAR hay camera. Truyền `sensor` tường minh giúp association luôn gọi `manager.manage_tracks(...)` với đúng sensor, kể cả không có đo hoặc không ghép được cặp nào. Trong lượt LiDAR, track không ghép và nằm trong FOV bị trừ score; đo LiDAR chưa gán có thể tạo track, và các track được xét xóa theo score/covariance. Camera chỉ hiệu chỉnh EKF cho cặp ghép: manager bỏ qua cập nhật score và quản lý vòng đời ở lượt camera. Vì vậy một frame camera rỗng không làm track mất điểm hoặc bị xóa. Quy tắc này tránh tính hai hit/miss tồn tại cho cùng một frame.

6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.

   Track mới từ LiDAR có score `1/window`, trạng thái `initialized`. Mỗi hit LiDAR cộng `1/window`, chặn trên tại 1; miss trong FOV trừ `1/window`. Hit khi chưa đủ ngưỡng chuyển track sang `tentative`; score **lớn hơn** `confirmed_threshold` thì thành `confirmed`. Mặc định `window = 6`, ngưỡng 0.8: năm frame hit liên tiếp tính cả frame khởi tạo cho score 5/6 và xác nhận ở frame thứ năm, khớp hai track ở frame 4 trong log.

   Miss không hạ một track confirmed xuống tentative: trạng thái được giữ cho đến khi track bị xóa. Ví dụ score từ 1 xuống 5/6 sau một miss vẫn giữ confirmed. Xóa nếu `P[0,0] > max_P` hoặc `P[1,1] > max_P` (mặc định 9 m²), hoặc confirmed có score **nhỏ hơn** `delete_threshold` (0.6), hoặc track chưa confirmed có score **≤ 0**. Việc xét xóa chỉ diễn ra trong lượt LiDAR; camera không tạo/xóa track và không đổi score.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không.

## Khai báo sử dụng AI (bắt buộc)

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): OpenAI Codex.
- Dùng cho phần nào (hàm, câu hỏi, debug): Codex đọc yêu cầu và cấu hình môi trường Python 3.14; viết toàn bộ các hàm bắt buộc Part E–H trong `kalman.py`, `camera_fusion.py`, `association.py`, `track_management.py`; rà soát, giải thích và kiểm tra các hàm. Codex chạy pipeline Waymo, tổng hợp số liệu thực tế, soạn sáu câu trả lời và báo cáo này, thực hiện commit/push theo từng checkpoint. Học viên cung cấp thông tin, dữ liệu, lựa chọn phạm vi bắt buộc và duyệt tiến độ trong hội thoại. Không làm bonus.
- Cách kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Codex thực hiện các kiểm tra trên máy học viên, không khẳng định học viên đã tự kiểm tra độc lập. Bộ `student/tests` đạt 128 passed, không failed/error/xfailed trên Python 3.14.7 với `PYTHONUTF8=1`. Đối chiếu công thức EKF, dùng `solve` cho gain/Mahalanobis, kiểm tra FOV trước projection và kiểm tra vòng đời chỉ do LiDAR quyết định. Kiểm tra thêm tham số động, đầu vào ndarray/matrix, biên FOV/độ sâu, ma trận association rỗng, phép biến đổi tọa độ/covariance và giữ confirmed sau miss.
- Kiểm chứng dữ liệu: Chạy compare thử frame 0–20, rồi chạy đủ frame 0–198 với seed 0 trên segment và weights học viên đã tải, bằng PyTorch CPU (`OMP_NUM_THREADS=4`, `MKL_NUM_THREADS=4`). Sáu artifacts do runner sinh, không chỉnh sửa tay. `validate_metrics_records` xác nhận 398 record gộp, 199 record mỗi mode, đủ frame, không thiếu/trùng, đúng invariant và tổng metrics; log từng mode khớp phần tương ứng trong log gộp. Cả hai mode đạt ngưỡng RMSE, precision_track, coverage và chênh lệch RMSE của rubric. Các file cấu hình local, dữ liệu và weights không commit.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Giữ nguyên Part A–D; chỉ implement các phần bắt buộc E–H
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền báo cáo và khai báo AI; SHA cuối cùng lấy sau commit CP6 như ghi ở phần thông tin
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, `.env`, API key
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP` sau khi commit CP6
- [ ] Học viên tự nộp link repo + commit hash trên LMS sau khi nhận kết quả CP6 ([hướng dẫn nộp](../SUBMISSION.md))
