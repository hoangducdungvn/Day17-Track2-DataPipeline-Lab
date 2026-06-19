# Bonus Challenge: Thiết kế Data Flywheel cho Chatbot CSKH Sàn TMĐT tuân thủ Nghị định 13 (PDPL)

## 1. Bài toán & Ràng buộc thực tế
**Bài toán:** Xây dựng một Data Pipeline (Data Flywheel) để tự động thu thập lịch sử trò chuyện (traces) của người dùng với Chatbot AI Chăm sóc khách hàng (CSKH) trên một sàn thương mại điện tử lớn tại Việt Nam (như Shopee, Tiki). Mục tiêu là biến hàng trăm ngàn lượt chat mỗi ngày thành tập dữ liệu đánh giá (Eval Set) và tập dữ liệu huấn luyện tăng cường (DPO Preference Pairs) nhằm làm cho bot ngày càng thông minh hơn.

**Ràng buộc thực tiễn:**
- **Khối lượng lớn & Nhiễu:** Tiếp nhận khoảng 500.000 tin nhắn/ngày, trong đó có rất nhiều tin nhắn cộc lốc ("hi", "ơi") hoặc khách hàng mất bình tĩnh do thất lạc đơn.
- **Rủi ro rò rỉ dữ liệu nhạy cảm (PII):** Khách hàng Việt Nam có thói quen gửi thẳng Số điện thoại, CCCD, Số tài khoản ngân hàng, hoặc Địa chỉ nhà chi tiết vào khung chat.
- **Tuân thủ pháp luật:** Bắt buộc tuân thủ Nghị định 13/2023/NĐ-CP về Bảo vệ dữ liệu cá nhân. Tuyệt đối không đưa dữ liệu PII chưa mã hóa (masking) vào huấn luyện mô hình.

## 2. Sơ đồ kiến trúc (Architecture Sketch)

```text
[User Chat] --> (Chatbot Agent) --> [Raw Traces (JSON OTel)]
                                         |
                                         v
                                  [Bronze Layer] (Append-only Data Lake)
                                         |
                            (Nightly Masking Job) <-- [Regex/PhoBERT NER]
                            *Nơi che mờ PII (Nghị định 13)*
                                         |
                                  [Silver Layer] (Cleaned, Masked Spans)
                                         |
                   +---------------------+---------------------+
                   |                                           |
           [Quality Gate]                               [Eval Router]
    (Drop short/meaningless chats)                   (Holdout 10% for Eval)
                   |                                           |
         [DPO Pairs Generator]                         [Eval Golden Set]
 (Pair: Prompt, Chosen, Rejected)                              |
                   |                                           |
                   +-----------> [Decontamination] <-----------+
                                (MinHash LSH matching)
                                         |
                                   [Gold Layer] 
                           (Training-ready DPO Dataset)
```

## 3. Chi tiết 10 câu hỏi thiết kế (Phiên tự phỏng vấn Brainstorming)

Dưới đây là lời giải và những đánh đổi cho cả 10 câu hỏi cốt lõi của việc thiết kế kiến trúc này:

### 1. Nguồn & hình dạng. Dữ liệu đến từ đâu, dạng gì, bẩn cỡ nào? Schema có ổn định không?
- **Nguồn:** Dữ liệu đến từ hệ thống Telemetry (OpenTelemetry) của Chatbot, lưu dưới dạng cây span JSON lồng nhau.
- **Độ bẩn & Schema:** Rất bẩn (chứa PII, chat vô nghĩa). Schema thường xuyên bị trôi (drift) mỗi khi team Dev cập nhật phiên bản bot (thêm/bớt các field trong `metadata`). Do đó, Bronze layer phải lưu nguyên cục JSON, đến Silver layer mới parse (flatten) và validate bằng Pandera để bắt các lỗi drift.

### 2. Batch hay streaming? Độ tươi cần bao nhiêu là đủ?
- **Quyết định:** Chọn **Batch hằng đêm (Nightly Batch)** thay vì Streaming (Kappa/Lambda).
- **Lý do (Trade-off):** Mô hình LLM không nên được train liên tục (Online learning) vì dễ bị "học lệch" (catastrophic forgetting) và hacker chèn mã độc (data poisoning). Flywheel chỉ cần gom DPO để fine-tune vào cuối tuần, nên việc duy trì cụm Streaming 24/7 là tốn kém vô ích (overkill).

### 3. Cái gì vỡ khi scale? Ở 10×, 100× dữ liệu, bottleneck đầu tiên là gì?
- **Vấn đề:** Trong mùa Sale 11/11, lượng dữ liệu x100. Nút thắt (Bottleneck) đầu tiên sẽ là "Small-files problem" nếu ta ghi từng trace JSON nhỏ xíu lên hệ thống lưu trữ (như S3).
- **Giải pháp:** Cần có một service gom cụm (compaction) các tin nhắn lẻ tẻ thành những block Parquet lớn (~128MB) trước khi ingest vào lớp Bronze để tối ưu I/O.

### 4. Hợp đồng & chất lượng. Bạn validate gì trước khi dữ liệu vào model? Dòng xấu đi đâu?
- **Validate:** Quan trọng nhất là bóc tách PII (Che số thẻ, SĐT, Địa chỉ). 
- **Cách ly (Quarantine):** Các cuộc hội thoại chứa quá nhiều ngôn từ thô tục (toxic) hoặc không bóc tách được PII sẽ bị ném thẳng vào `quarantine/`.
- **Alert:** Cấu hình cảnh báo bắn vào Slack của Data Engineer nếu tỷ lệ quarantine vượt mức 5%, báo hiệu hệ thống bot đang bị spam hàng loạt.

### 5. Train/serve parity. Chỗ nào có thể rò rỉ tương lai? Cần point-in-time ở đâu?
- **Nguy cơ rò rỉ:** Khi tạo dataset, nếu ta JOIN thêm cột "Hạng thành viên của khách hàng" (Đồng/Bạc/Vàng) để model học cách ưu tiên trả lời.
- **Giải pháp:** Bắt buộc dùng `ASOF JOIN`. Nếu hôm nay khách lên hạng Vàng, nhưng đoạn chat xảy ra tháng trước lúc họ hạng Bạc, ta phải lấy đúng hạng Bạc tại thời điểm (point-in-time) của sự kiện chat. Nếu không, mô hình sẽ rò rỉ tương lai và có thiên kiến sai lệch.

### 6. Phi cấu trúc -> RAG hay KG? Dùng cái nào? Chi phí/Độ trễ?
- **RAG vs KG:** Để xử lý các chính sách đổi trả phức tạp chéo nhau ("Tôi mua áo sơ mi ở kho Hà Nội, nhưng muốn đổi trả ở TP.HCM sau 5 ngày thì phí bao nhiêu?"), việc dùng Knowledge Graph (KG) là tốt nhất để truy xuất (hopping) qua các nút "Sản phẩm" -> "Kho" -> "Chính sách vùng miền".
- **Đánh đổi:** Dùng Vector bình thường rẻ và độ trễ thấp nhưng dễ trả lời sai ngớ ngẩn (hallucination). KG tốn token để extract thực thể và tốn công xây dựng đồ thị, nhưng giảm hẳn rủi ro pháp lý khi tư vấn chính sách sai cho khách hàng.

### 7. Flywheel. Làm sao biến trace thành eval set mà không tự đầu độc bằng leakage?
- **Vấn đề:** Tạo DPO Pairs (Prompt, Chọn câu Bot nói đúng, Bỏ câu Bot nói sai/bị vote 1 sao).
- **Decontamination:** Khách hay hỏi những câu tương tự nhau ("hàng của tôi đâu" vs "hàng tôi đâu"). Nếu dùng Exact Match (so khớp tuyệt đối) thì sẽ sót rác. Cần dùng **Fuzzy Matching** (như thuật toán MinHash LSH) để đo khoảng cách N-gram. Bất kỳ câu nào trong DPO có độ tương đồng > 0.8 với tập Eval sẽ bị loại để tránh rò rỉ đề thi (leakage).

### 8. Failure semantics. Chạy lại có idempotent không? Backfill an toàn thế nào?
- **Idempotent:** Lớp Bronze là Append-only. Pipeline từ Bronze -> Silver -> Gold dùng tính năng Overwrite Partition (ghi đè phân vùng theo ngày). 
- **Backfill:** Khi logic bắt PII thay đổi (ví dụ nhà nước đổi luật số CMND), ta có thể backfill (chạy lại) an toàn bằng cách xóa partition cũ ở Silver và chạy lại từ Bronze mà không bao giờ bị nhân đôi dòng dữ liệu. Phương án này (Overwrite) an toàn hơn so với việc cố update (Mutate) trực tiếp.

### 9. Chi phí & vận hành. Ai trả tiền? Đâu là 80% chi phí? Cắt ở đâu?
- **Ai trả tiền:** Khối Dịch vụ Khách hàng (Customer Service) trả chi phí vì hệ thống này giúp họ giảm bớt điện thoại viên.
- **80% Chi phí:** Nằm ở khâu nhận diện PII. Việc đẩy toàn bộ text cho LLM (như GPT-4) cực kỳ đắt đỏ.
- **Cắt giảm (Trade-off):** Phương án bị loại là dùng LLM to. Phương án thực hiện là Hybrid: Dùng Regex (gần như miễn phí) bắt các định dạng tĩnh như Email, SĐT, Số thẻ. Chỉ những đoạn text lộn xộn chứa địa chỉ, tên người mới được đẩy qua mô hình PhoBERT NER nhỏ gọn chạy ở server local. Việc này cắt 90% chi phí mà chất lượng vẫn đảm bảo.

### 10. Bối cảnh Việt Nam. Điều gì đổi so với một bài blog tiếng Anh?
- **PDPL (Luật Bảo vệ Dữ liệu Cá nhân VN):** Áp dụng nghiêm ngặt Nghị định 13/2023/NĐ-CP. Việc đẩy data thô ra API LLM nước ngoài chưa qua mã hóa là vi phạm luật chuyển dữ liệu xuyên biên giới. Bắt buộc phải xử lý Masking PII ngay tại Local Server.
- **Tiếng Việt sai chính tả:** Khách hàng gõ không dấu, teencode ("gjao hang lau the"). Pipeline bắt buộc phải có bước chuẩn hóa (vi-tokenizer) ở Silver Layer để các khâu Fuzzy Match (Decontamination) ở trên mới chạy chính xác được.
