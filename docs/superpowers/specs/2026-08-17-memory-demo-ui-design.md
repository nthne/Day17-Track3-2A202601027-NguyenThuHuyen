# Lab 17 Memory Demo UI Design

## Goal

Hoàn thiện mini-product Streamlit trong `src/demo_ui.py` để đạt đủ rubric bonus UI: load case, hiển thị metadata, chạy retrieval với evidence từng layer/merged context, và chat tiếp trên cùng user/thread với history được giữ lại.

## Architecture

UI tiếp tục dùng shell hiện có. `retrieve_for_case` là orchestration boundary duy nhất: nó tạo bốn layer text rỗng, lấy short-term từ fixture/session cộng history chat, gọi các contract của `StudentMemory` cho layer được case yêu cầu, rồi gọi `assemble_context` để trả về merged context và breakdown. Case `mixed` chỉ truy vấn các layer trong `retrieve_layers`, tránh gọi thừa.

Chat không ghi transcript mới lên Zep; mỗi lượt dùng cùng `user_id`/`thread_id` của case, thêm history ngắn vào short-term local và query mới vào retrieval. Gemini chỉ dùng sau retrieval để tạo câu trả lời grounded; khi thiếu Gemini key, UI hiển thị context đã retrieve để phần demo retrieval vẫn hoạt động.

## Testing and evidence

- Contract test dùng fake `StudentMemory` và case tối giản để kiểm tra layer selection, short-term history, merged context và budget.
- Chạy toàn bộ pytest sau implementation.
- Chạy Streamlit trong Docker bằng `docker compose run --rm --service-ports app streamlit run src/demo_ui.py --server.address 0.0.0.0 --server.port 8501`.
- Kiểm tra thủ công đủ bốn rubric: chọn case, xem metadata, run retrieval/evidence, gửi chat follow-up và thấy history/assistant reply.
- Không commit `.env`, API key, `data/golden_eval.json`, hoặc thay đổi evaluator/dataset/reference.
