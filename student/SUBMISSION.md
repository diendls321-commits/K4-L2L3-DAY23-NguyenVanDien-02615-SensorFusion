# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Nguyễn Văn Diện
- MSSV: 02615
- Email: diendls321@gmail.com
- Link repo (fork): https://github.com/diendls321-commits/K4-L2L3-DAY23-NguyenVanDien-02615-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`):

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
2. Vì sao cần gating Mahalanobis trước khi gán?
3. Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log `fusion-run-lab`.
4. Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?
5. Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Giải thích vì sao lidar quyết định score/init/delete còn camera chỉ EKF update.
6. Nêu điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xóa track.

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- Không.

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): OpenAI Codex.
- Dùng cho phần nào (hàm, câu hỏi, debug): CP0 — đọc yêu cầu, chuẩn bị môi trường Python 3.14, cấu hình đường dẫn dữ liệu và điền thông tin học viên. CP1 — hỗ trợ cài đặt và giải thích sáu hàm Part E trong kalman.py: build_F, build_Q, ekf_predict, innovation, innovation_covariance, ekf_update. CP2 — hỗ trợ cài đặt và giải thích ba hàm Part G trong camera_fusion.py: is_in_field_of_view, camera_measurement_prediction, build_camera_measurement. Cập nhật phạm vi hỗ trợ sau mỗi checkpoint.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): CP0 — chạy bộ test gốc trên Python 3.14.7 với PYTHONUTF8=1: 82 passed, 46 xfailed do Part E–H còn TODO, không có failed/error; CLI fusion-run-lab --help chạy được. Đọc thành công frame đầu của segment đã tải và nạp được 186 tensor trong weights bằng torch.load(weights_only=True). CP1 — test_kalman.py: 4 passed, không còn xfailed ở Part E. Kiểm tra thêm tham số mặc định lấy động từ cấu hình, dt/q bằng 0, đầu vào ndarray/matrix, giá trị innovation/S và việc không sửa đầu vào. Đối chiếu F vận tốc không đổi, Q = dt * q * I_6, predict và update với công thức trong đề; dùng solve để tính K thay vì tạo nghịch đảo S. CP2 — chạy đúng nhóm test camera trong CHECKPOINTS.md: 20 passed, không failed/error/xfailed; gồm FOV theo calibration, projection, dữ liệu không hợp lệ và Jacobian so với sai phân hữu hạn. Kiểm tra thêm FOV LiDAR, đầu vào ndarray/matrix, hai biên FOV camera, độ sâu đúng 1e-6, ngữ cảnh lỗi và covariance đọc động từ tham số nhiễu pixel. Chưa chạy pipeline Waymo hay tạo metrics; cập nhật kiểm chứng sau từng checkpoint.

- Phạm vi hỗ trợ CP3: Cài đặt và giải thích năm hàm Part F trong association.py: mahalanobis_distance, chi2_gate, association_cost_matrix, pick_next_pair, associate_and_update. Nạp Kalman bằng load_workspace_module("kalman"), kiểm tra FOV trước khoảng cách, gán greedy rồi gọi quản lý track với sensor tường minh; không predict thêm trong association.
- Kiểm chứng CP3: Nhóm test Mahalanobis/greedy/out_of_fov/association trong CHECKPOINTS.md đạt 9 passed, không failed/error/xfailed. Kiểm tra thêm ngưỡng chi-square theo số chiều đo và xác suất cấu hình, loại khoảng cách không hữu hạn/âm, ma trận rỗng 0x0/0x2/2x0, thứ tự greedy nhiều cặp, bảo toàn danh sách đầu vào và không gọi predict. Part H còn TODO; chưa chạy pipeline Waymo hay tạo metrics.

- Phạm vi hỗ trợ CP4: Cài đặt và giải thích ba hàm Part H trong track_management.py: init_track_state_from_meas, update_track_score, should_delete_track. Khởi tạo vị trí/covariance từ đo LiDAR sau đổi sang hệ xe, vận tốc bằng 0 với phương sai sigma^2; hit/miss đổi score theo window, giữ confirmed sau miss và xoá theo score hoặc phương sai ngang. Từ chối khởi tạo bằng đo camera.
- Kiểm chứng CP4: Chạy toàn bộ student/tests trên Python 3.14.7 với PYTHONUTF8=1: 128 passed, không failed/error/xfailed. Các test xác nhận camera không đổi score/khởi tạo/xoá track, miss ngoài FOV LiDAR không giảm score và confirmed được giữ sau miss. Kiểm tra thêm khởi tạo với phép xoay 90 độ và tịnh tiến, covariance vị trí có nhiễu khác nhau theo trục, bình phương sigma vận tốc, window cấu hình, đầu vào ndarray/matrix và bảo toàn đo gốc. Toàn bộ Part E–H đã hoàn thành; chưa chạy pipeline Waymo hay tạo metrics.

- Phạm vi hỗ trợ CP5: Chạy thử compare trên frame 0–20, sau đó đặt paths.yaml thành frame 0–198 và chạy compare với seed 0 trên segment/weights đã tải. Chạy Python 3.14.7, PyTorch CPU với OMP_NUM_THREADS=4 và MKL_NUM_THREADS=4; không sửa thuật toán hay tham số tracking. Điền kết quả và nhận xét từ metrics/log thực tế.
- Kiểm chứng CP5: Đủ sáu artifacts do runner sinh; validate_metrics_records xác nhận file gộp có 398 record và mỗi mode có 199 record, đúng frame 0–198, không thiếu/trùng, mọi invariant và tổng khớp metrics. Hai log riêng đúng bằng phần mode tương ứng trong log gộp; detection, seed và segment nhất quán giữa hai mode. Cả hai mode đạt ngưỡng RMSE ≤ 0.45 m, precision_track ≥ 0.75, coverage ≥ 0.70 và fused − lidar ≤ 0.05 m. Không sửa tay artifacts; .env, paths.yaml, dữ liệu và weights chỉ lưu local.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [ ] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [ ] Đã điền đủ file này, gồm khai báo AI
- [ ] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [ ] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
