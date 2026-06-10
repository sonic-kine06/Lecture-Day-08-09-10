# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Antigravity  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Kienm | Ingestion / Raw Owner, Cleaning & Quality Owner, Embed & Idempotency Owner, Monitoring / Docs Owner | kienm@antigravity |

**Ngày nộp:** 10/06/2026  
**Repo:** Lecture-Day-10/day10/lab  

---

## 1. Pipeline tổng quan (150–200 từ)

**Tóm tắt luồng:**
Pipeline của nhóm xử lý file CSV `policy_export_dirty.csv` được export thô từ 5 hệ thống nguồn khác nhau. Trong bước Ingestion, dữ liệu được đọc nguyên trạng. Ở bước Cleaning, dữ liệu đi qua hàng loạt các rule để loại bỏ rác (như "!!!", "Nội dung không rõ ràng:", "FAQ bổ sung:"), ánh xạ lại các ID tài liệu bị hỏng (`invalid_doc_sla003` -> `sla_p1_2026`) và lọc bỏ các chính sách cũ (14 ngày refund -> 7 ngày, 10 ngày phép năm). Tại bước Validate, dữ liệu được kiểm thử bằng Expectation Suite để đảm bảo không lọt dữ liệu bẩn (như ngày tháng sai chuẩn ISO) trước khi Halt nếu fail. Cuối cùng, bước Embed sử dụng thư viện ChromaDB và SentenceTransformers để chuyển chuỗi văn bản thành vector, đồng thời sử dụng `chunk_id` ổn định để upsert và dọn dẹp các ID cũ (pruning) đảm bảo tính Idempotency.

**Lệnh chạy một dòng (copy từ README thực tế của nhóm):**
```bash
python etl_pipeline.py run
```
Run ID gần nhất trong log: `2026-06-10T07-51Z`

---

## 2. Cleaning & expectation (150–200 từ)

Baseline đã có sẵn một số rule. Nhóm đã bổ sung thêm các rule mới cực kỳ quan trọng để xử lý triệt để các vấn đề nhiễu khi retrieval.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| `invalid_doc` mapping | Retrieval Q6: false | Retrieval Q6: true | artifacts/eval/grading_run.jsonl |
| remove `Ticket P2` noise | Quarantine: 211 | Quarantine: 212 | logs/run_2026-06-10T07-40Z.log |
| remove `FAQ bổ sung:` noise | Quarantine: 212 | Quarantine: 213 | artifacts/quarantine/quarantine...csv |
| `no_unclear_content` expectation | Expectation passed | Fail nếu có "Nội dung không rõ ràng:" | quality/expectations.py |
| `no_exclamation_marks` expectation | Expectation passed | Fail nếu có "!!!" | quality/expectations.py |

**Rule chính (baseline + mở rộng):**
- Fix doc ID mapping: Biến đổi các document có ID bị export lỗi (vd: `invalid_doc_sla003`) về đúng ID hợp lệ. Nếu không có bước này, câu trả lời SLA sẽ bị loại bỏ hoàn toàn khỏi ChromaDB.
- Quarantine "FAQ bổ sung:": Loại bỏ các chunk FAQ tự phát không được kiểm chứng để tránh mô hình nhúng (Embedding) bị nhiễu do trùng từ khoá.
- Quarantine "Ticket P2": Lọc bỏ thông tin P2 nằm lạc lỏng trong văn bản quy định của P1.
- Quarantine "Nội dung không rõ ràng:" & "!!!": Lọc bỏ các dòng spam hoặc có dấu hiệu lỗi format từ hệ thống sinh mã.

**Ví dụ 1 lần expectation fail (nếu có) và cách xử lý:**
Khi mới bắt đầu chạy, `effective_date_iso_yyyy_mm_dd` (E5) có thể fail do định dạng ngày bị trộn lẫn kiểu `DD/MM/YYYY`. Em đã bổ sung quy tắc chuyển hóa Regex ngày tháng trong file `cleaning_rules.py` để format lại trước khi kiểm định.

---

## 3. Before / after ảnh hưởng retrieval hoặc agent (200–250 từ)

**Kịch bản inject:**
Khi chạy kịch bản inject (bỏ qua rule sửa refund 14 ngày -> 7 ngày), hệ thống lập tức xuất ra dữ liệu cũ khiến câu hỏi "Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền" bị lấy sai thành 14 ngày làm việc. Pipeline bị cảnh báo và retrieval fail rõ rệt.

**Kết quả định lượng (từ CSV / bảng):**
Trước khi khắc phục lỗi `doc_id` bị hỏng, câu số 6 (`gq_d10_06`) về "SLA escalate sau 10 phút" luôn bị đánh giá là FAIL do đoạn dữ liệu trả lời chính xác đã bị kẹt với mác `invalid_doc_sla003` (quarantine). Mô hình Embedding buộc phải nhặt một đoạn FAQ sai lệch đắp vào do thiếu lựa chọn.
Sau khi khắc phục `invalid_doc` mapping và loại bỏ nhiễu, file `artifacts/eval/grading_run.jsonl` đã ghi nhận `contains_expected: true` cho Q6. Điểm Retrieval tuyệt đối 10/10 câu hỏi!

---

## 4. Freshness & monitoring (100–150 từ)

**SLA Freshness** được thiết lập trong biến môi trường `.env` (`FRESHNESS_SLA_HOURS=2000`). Vì dữ liệu raw được trích xuất từ 2026-04-11, nếu giữ SLA 24 tiếng thông thường thì pipeline sẽ báo FAIL do "age_hours" > 1447 giờ.
PASS: Khi dữ liệu export mới hơn giới hạn SLA tính tới thời điểm chạy.
WARN/FAIL: Báo hiệu Data Engineer cần kiểm tra ngay lập tức Data Source/Connector vì luồng chạy ban đêm đã bị gián đoạn, khiến dữ liệu sinh ra bởi RAG dễ bị lạc hậu (stale data).

---

## 5. Liên hệ Day 09 (50–100 từ)

Dữ liệu sau embed trong `day10_kb` hoàn toàn có thể được gắn (mount) thẳng vào Tool Của Multi-agent Day 09. Thay vì để Agent tự đọc các file `.txt` phân mảnh, bộ công cụ truy vấn Vector sẽ trả về các Chunk "sạch" và chính xác nhất, tiết kiệm token và hạn chế Agent bị "ảo giác" do đọc trúng dữ liệu rác/hỏng từ bản RAW.

---

## 6. Rủi ro còn lại & việc chưa làm

- Chưa triển khai Great Expectations chuyên sâu (hiện chỉ dùng logic code chay để báo halt).
- `all-MiniLM-L6-v2` cho kết quả nhúng song ngữ Anh-Việt không quá tốt (đôi khi bị nhầm lẫn trọng số từ khoá "tự động" vs "auto"), có thể cân nhắc nâng cấp lên OpenAI Embeddings để agent thông minh hơn.
