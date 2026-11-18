# Thông tin học viên

- Họ và tên: Trần Trọng Thái
- MSHV: 2A202604065
- Lớp (L2 / L3 / khác): L2B
- Tài khoản GitHub: tranthai2402
- Repo bài nộp (URL): https://github.com/tranthai2402/K5-L2B-Day08-TranTrongThai-2A202604065-AI-Assisted-Labeling
- Commit chốt bài (hash): 80a158bdbf271b0ebfa4afdefa75772f0b8fd6a5
- Máy: Windows 10 (build 22631), CPU (khuyến nghị chạy script trên CPU), GPU NVIDIA GeForce GTX 1650 (không dùng cho các script của bài)
- CVAT (Docker): phần mềm bản v2.81.0

## Kết quả tóm tắt (điền cuối bài; số lấy từ `submission/qc/*`, kèm mẫu số)

- Máy: accepted / edited / deleted: 582 / 15 / 37 (mẫu số: 639 box máy); added: 57 (mẫu số: 657 đối tượng sau duyệt)
- So với gold: precision / recall của **máy** 75.7% / 85.5% (mẫu số: 62 đối tượng gold; TP 53, FP 17, FN 9; KTC 95% lần lượt 65–84% và 75–92%)
- Bắt bẫy: 11/12 = 91.7%
- Thời gian: gán tay 25 phút/ảnh; duyệt có pre-label 12 phút/ảnh
- Lỗi QC: 80.0% theo ảnh (4/5), 5.8% theo đối tượng (7/120)
- **Quyết định:** Rework — ảnh có lỗi 4/5 = 80.0% vượt ngưỡng 2%, và 6/7 lỗi cùng loại `thua` nên là lỗi hệ thống cần sửa recipe chứ không sửa lẻ từng ảnh.
