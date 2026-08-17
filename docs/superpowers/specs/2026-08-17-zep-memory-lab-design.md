# Day 17 Zep Memory Lab Design

## Goal

Hoàn thiện bốn contract retrieval trong `src/memory_student.py`, chứng minh đúng session/user/knowledge scope và token budget, rồi tạo các report/artefact bắt buộc của Day 17 mà không sửa evaluator, test, ground truth hoặc reference implementation.

## Architecture

`StudentMemory` giữ bốn ranh giới rõ ràng: long-term lấy Context Block theo user/thread; episodic tìm episode trên user graph; semantic tìm episode trên shared graph; và assembler dùng `ContextBudgetManager` để trim theo thứ tự ưu tiên short-term, long-term, episodic, semantic với tỷ lệ 10/4/3/3. Mọi truy vấn graph được giới hạn qua `cap_query` và kết quả graph được render bằng helper có sẵn.

Các bước benchmark chạy trên dữ liệu seed của lab. E01/E10 tiếp tục do short-term local evaluator xử lý; E02–E09 và E11 kiểm tra retrieval student. Privacy drill chỉ xóa user synthetic được chỉ định, không xóa shared semantic graph.

## Testing and evidence

- Chạy unit tests hiện có trước và sau thay đổi.
- Với mỗi contract, kiểm thử trực tiếp bằng test hiện có hoặc test harness nhỏ không sửa evaluator; quan sát lỗi trước khi implementation theo TDD.
- Chạy benchmark `student`, `no_memory`, và `compare_reports` sau khi seed thành công.
- Lưu `reports/benchmark.*`, `reports/benchmark_no_memory.*`, `reports/comparison.md`, `README_submission.md`, và ảnh minh chứng theo quy định nếu môi trường Zep khả dụng.
- Không commit `.env`, API key, `data/golden_eval.json`, hoặc thay đổi ngoài phạm vi cho phép.

## Scope boundaries

Chỉ sửa `src/memory_student.py` cho core retrieval, tạo README/report/submission artefacts được hướng dẫn cho phép, và chỉ sửa `src/demo_ui.py` nếu bonus UI thực sự được thực hiện. Không sửa tests, evaluator, dataset, context budget, zep helpers, short-term demo lâu dài, hay reference implementation.
