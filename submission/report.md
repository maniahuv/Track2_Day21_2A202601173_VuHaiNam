# Báo Cáo Lab MLOps — Day 21 CI/CD cho AI Systems

Họ tên: Vu Hai Nam — Lớp: 2A202601173
Repo: https://github.com/maniahuv/Track2_Day21_2A202601173_VuHaiNam

## 1. Bộ Siêu Tham Số Đã Chọn Và Lý Do

Sau 5 lần chạy thử nghiệm ở Bước 1 (so sánh qua MLflow UI), bộ tốt nhất ban đầu là
`n_estimators=150, max_depth=None, min_samples_split=2` (accuracy 0.678). Các lần dùng
`max_depth` nông (3, 5) đều cho kết quả thấp hơn hẳn (0.56–0.64) do underfit với bài toán
phân loại 3 lớp có ranh giới phức tạp giữa các mức chất lượng rượu vang.

Ở Bước 2, để tối ưu thêm cho ngưỡng eval gate, đã dò thêm 75 tổ hợp
(`n_estimators` ∈ {150,200,300,400,500}, `max_depth` ∈ {None,15,20,25,30},
`min_samples_split` ∈ {2,3,4}). Kết quả tốt nhất:

**`n_estimators=150, max_depth=20, min_samples_split=2` → accuracy = 0.688, f1 = 0.687**
(trên `train_phase1.csv`, 2998 mẫu). Giới hạn `max_depth=20` (thay vì không giới hạn) giúp
mô hình tổng quát hoá tốt hơn một chút trên tập eval, giảm overfit nhẹ so với cây không giới
hạn độ sâu.

Ở Bước 3, cùng bộ tham số này khi huấn luyện trên dữ liệu gộp (`train_phase1` + `train_phase2`,
5996 mẫu) đạt **accuracy = 0.758, f1 = 0.757** — tăng đáng kể so với Bước 2, cho thấy việc bổ
sung dữ liệu cải thiện rõ rệt chất lượng mô hình.

| Chỉ số | Bước 2 (2998 mẫu) | Bước 3 (5996 mẫu) |
|---|---|---|
| accuracy | 0.6880 | 0.7580 |
| f1_score | 0.6868 | 0.7569 |

*Lưu ý:* ngưỡng eval gate trong `mlops.yml` được điều chỉnh từ 0.70 xuống **0.68** (đã xin phép
giảng viên), vì RandomForest chỉ với 3 siêu tham số điều chỉnh được (`n_estimators`, `max_depth`,
`min_samples_split`) đạt trần khoảng 0.688 trên riêng `train_phase1`, không đủ 0.70. Cơ chế
eval gate vẫn hoạt động đúng vai trò chặn deploy khi chưa đạt ngưỡng (đã có ảnh chụp lần chạy
bị chặn ở 0.70 làm bằng chứng).

## 2. Khó Khăn Gặp Phải Và Cách Giải Quyết

- **Thư mục `mlruns/` bị commit nhầm vào git và hỏng** (thiếu `meta.yaml`), khiến
  `mlflow.start_run()` crash khi chạy `pytest` không set `MLFLOW_TRACKING_URI` — làm job Unit
  Test trên CI đỏ ngay từ đầu. Khắc phục: `git rm --cached mlruns` và thêm `mlruns/` vào
  `.gitignore`.

- **`gcloud compute scp ... instance:~/file` lỗi trên Windows** (`pscp: unable to open`) vì
  gcloud SDK trên Windows dùng PuTTY's `pscp`, không hỗ trợ ký hiệu `~/`. Khắc phục: dùng đường
  dẫn tương đối không có `~/` (scp mặc định "đứng" ở thư mục home).

- **`dvc pull` trên CI báo lỗi `Invalid Credentials, 401`** dù secret `CLOUD_CREDENTIALS` đúng.
  Nguyên nhân: `.dvc/config` (file có commit vào git) chứa `credentialpath = ../sa-key.json` —
  đường dẫn chỉ đúng trên máy cá nhân, không tồn tại trên CI runner, khiến DVC rơi về xác thực
  rỗng. Khắc phục: chuyển `credentialpath` sang `.dvc/config.local` (DVC tự động gitignore),
  để CI dùng đúng biến môi trường `GOOGLE_APPLICATION_CREDENTIALS`.

- **Repo là fork nên GitHub Actions bị tắt mặc định**, khiến mọi lần push không có workflow run
  nào được ghi nhận. Khắc phục: bật `Actions permissions` trong Settings, sau đó trigger lại
  bằng `workflow_dispatch`.

- **Secret `VM_SSH_KEY` dán thiếu nội dung** gây lỗi `ssh: no key found` ở job Deploy. Khắc
  phục: copy lại toàn bộ private key bằng `Get-Content -Raw | Set-Clipboard` để tránh mất dòng
  khi copy tay.
