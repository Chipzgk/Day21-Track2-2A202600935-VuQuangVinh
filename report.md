# Báo Cáo Lab MLOps - Day 21
**Sinh viên:** Vũ Quang Vinh  
**MSSV:** 2A202600935  
**Course:** AIInAction - VinUni  

---

## 1. Bộ Siêu Tham Số Đã Chọn

Sau khi chạy nhiều thí nghiệm với MLflow, bộ siêu tham số tốt nhất được chọn là:

| Tham số | Giá trị | Lý do |
|---|---|---|
| `n_estimators` | 1000 | Nhiều cây hơn giúp mô hình ổn định, giảm variance |
| `max_depth` | 20 | Đủ sâu để học các pattern phức tạp mà không overfit |
| `min_samples_split` | 2 | Giá trị mặc định, phù hợp với dataset có ~3000 mẫu |
| `min_samples_leaf` | 1 | Cho phép cây phân chia đến mức chi tiết nhất |

**Kết quả:** Accuracy = 0.6800, F1 = 0.6786 trên tập eval (500 mẫu).

So sánh các lần thử nghiệm:

| Lần | n_estimators | max_depth | Accuracy | F1 |
|---|---|---|---|---|
| 1 | 100 | 5 | 0.5640 | 0.5534 |
| 2 | 50 | 3 | 0.5580 | 0.5185 |
| 3 | 200 | 10 | 0.6440 | 0.6417 |
| 4 (tốt nhất) | 1000 | 20 | 0.6800 | 0.6786 |

---

## 2. Kiến Trúc Hệ Thống

```
[Máy cá nhân]
      |  git push (data/*.dvc, src/*.py, params.yaml)
      v
[GitHub repository]
      |  GitHub Actions tự động kích hoạt
      v
[Runner: Unit Test → Train → Eval (>= 0.65) → Deploy]
      |                                    |
      |  dvc pull                          |  upload model.pkl
      v                                    v
[AWS S3 Bucket]                      [AWS EC2 VM]
  data/                                FastAPI serve
  models/latest/model.pkl              POST /predict
```

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

### 3.1 MLflow không tìm thấy experiment
**Vấn đề:** Lỗi `Could not find experiment with ID 0` khi chạy `train.py`.  
**Nguyên nhân:** Biến môi trường `MLFLOW_TRACKING_URI` chưa được set.  
**Giải pháp:** Thêm `mlflow.set_tracking_uri("sqlite:///mlflow.db")` trực tiếp vào đầu hàm `train()`.

### 3.2 DVC không nhận diện S3
**Vấn đề:** Lỗi `s3 is supported, but requires 'dvc-s3' to be installed`.  
**Nguyên nhân:** Package `dvc-s3` chưa được cài.  
**Giải pháp:** Cài thêm `pip install dvc-s3`.

### 3.3 File model.pkl quá lớn trên GitHub
**Vấn đề:** GitHub từ chối push vì `mlruns/` chứa file `.pkl` hơn 100MB.  
**Nguyên nhân:** Thư mục `mlruns/` chưa được thêm vào `.gitignore` trước khi commit.  
**Giải pháp:** Dùng `git-filter-repo` để xóa `mlruns/` khỏi toàn bộ git history, sau đó force push.

### 3.4 GitHub Actions branch không khớp
**Vấn đề:** Pipeline không được kích hoạt sau khi push.  
**Nguyên nhân:** Workflow cấu hình `branches: [main]` nhưng repo dùng branch `master`.  
**Giải pháp:** Sửa workflow thành `branches: [master]`.

### 3.5 Service trên VM chạy file serve.py cũ
**Vấn đề:** VM báo lỗi `No module named 'google'` dù đã copy file mới.  
**Nguyên nhân:** File `serve.py` cũ vẫn còn dòng `from google.cloud import storage`.  
**Giải pháp:** Ghi đè trực tiếp file trên VM bằng `cat > ~/src/serve.py << 'EOF'`.

### 3.6 Health check timeout
**Vấn đề:** Deploy job fail dù service đang chạy bình thường.  
**Nguyên nhân:** `sleep 5` không đủ thời gian chờ model load (~7 giây).  
**Giải pháp:** Tăng lên `sleep 15`.

---

## 4. Kết Quả Đạt Được

| Hạng mục | Kết quả |
|---|---|
| MLflow tracking | ✅ 4 runs, ghi nhận accuracy và f1_score |
| DVC + S3 | ✅ 3 file CSV được version hóa và lưu trên AWS S3 |
| GitHub Actions | ✅ 4 jobs (Unit Test, Train, Eval, Deploy) đều pass |
| Eval gate | ✅ Chặn deploy khi accuracy < 0.65 |
| FastAPI trên EC2 | ✅ `POST /predict` trả về kết quả đúng |
| Continuous training | ✅ Commit data mới → pipeline tự động chạy lại |

**Endpoint:** `http://18.140.57.206:8000`  
**Health check:** `{"status": "ok"}`  
**Predict:** `{"prediction": 0, "label": "thấp"}`