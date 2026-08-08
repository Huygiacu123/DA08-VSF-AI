# TODO — Ý tưởng tương lai (chưa triển khai)

> Ghi chú: các mục dưới đây là ý tưởng cải tiến chatbot được đề xuất, **chưa code, chưa
> triển khai**. Chỉ lưu lại để tham khảo khi làm tiếp.

## 1. Agent routing bằng LLM (intent classification)

Dùng một LLM (có thể là model nhẹ/rẻ) đọc câu hỏi đầu vào của người dùng, phân loại intent,
và quyết định route sang agent/luồng xử lý nào — trước khi vào orchestrator chính
(`query-service`).

Ví dụ luồng:
```
User input
  → LLM router (phân loại: RAG / HR / small-talk / khác)
  → route sang agent tương ứng
```

Câu hỏi cần trả lời khi triển khai: dùng model nào cho bước router (ưu tiên nhanh/rẻ vì chỉ
làm classification), fallback khi router không chắc chắn (confidence thấp) thì xử lý sao,
có cần few-shot examples hay fine-tune riêng cho bước phân loại này không.

## 2. RAG self-critique loop (answerer + verifier lặp tối đa 3 vòng)

Thay vì trả lời RAG một lượt, thêm một vòng lặp tự kiểm tra chất lượng câu trả lời trước khi
trả về người dùng:

```
1. Lấy top-5 chunk theo similarity từ Qdrant
2. LLM #1 (answerer): sinh câu trả lời dựa trên 5 chunk đó
3. LLM #2 (verifier): kiểm tra câu trả lời (đúng ngữ cảnh? đủ thông tin? có bịa không?)
4. Nếu verifier chưa chấp nhận:
   → đưa phản hồi của verifier ngược lại cho answerer
   → answerer sinh lại câu trả lời mới
   → lặp lại bước 3
5. Dừng khi verifier chấp nhận, hoặc đã lặp đủ 3 vòng (lấy câu trả lời tốt nhất trong 3 vòng)
```

Câu hỏi cần trả lời khi triển khai: tiêu chí "tốt nhất" verifier chấm dựa trên gì (độ chính
xác so với chunk, độ đầy đủ, không hallucination...), chi phí tăng thêm (tối đa gấp 3 lần số
lệnh gọi LLM cho 1 câu hỏi) có chấp nhận được không, có cần stream câu trả lời tạm trong lúc
đang lặp hay đợi xong mới trả về.

## 3. Harness đánh giá tự động để cải thiện chatbot

Xây dựng một bộ khung đánh giá/test tự động (harness) chạy độc lập với luồng chat chính:
chạy một bộ câu hỏi mẫu (golden set) qua chatbot, chấm điểm chất lượng câu trả lời (có thể
bằng LLM-as-judge hoặc so khớp với đáp án mẫu/metric retrieval như recall@k), rồi dùng kết
quả đó để lặp cải thiện prompt/routing/retrieval theo thời gian — chạy định kỳ (vd mỗi lần
đổi prompt hoặc mỗi tuần) để phát hiện regression sớm.

Liên quan tới bug thật đã gặp ở `ai-router` (fallback embedding sang model khác làm giảm
chất lượng retrieval mà không có exception nào) — một harness đánh giá định kỳ sẽ bắt được
loại lỗi "âm thầm" này sớm hơn nhiều so với chỉ dựa vào báo cáo thủ công từ người dùng.
