# Báo Cáo Nhóm — Lab 7: Embedding & Vector Store

**Nhóm:** [Tên nhóm]
**Thành viên:** [Họ tên từng thành viên]
**Ngày:** [Ngày nộp]

> **Nộp 1 bản / nhóm.** Phần cá nhân (hướng tiếp cận, kết quả riêng, dự đoán…) mỗi thành viên nộp riêng trong `REPORT_CANHAN.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần nhóm: 40** = Lựa chọn tài liệu (10) + Thiết kế chiến lược (15) + Chất lượng truy xuất (10) + Thuyết trình (5).

---

## 1. Lựa chọn tài liệu (Document Set Quality) — Nhóm (10 điểm)

### Chủ đề (Domain) & Lý Do Chọn

**Chủ đề:** Chính sách đổi trả và bảo hành trên Shopee Việt Nam.

**Tại sao nhóm chọn chủ đề này?**
Nhóm chọn chủ đề này vì có cả tài liệu dành cho `buyer` và `seller`, phù hợp để kiểm tra metadata filtering. Nội dung chính sách có nhiều mốc thời gian, điều kiện và ngoại lệ nên giúp quan sát rõ ảnh hưởng của chunking đến retrieval.

### Danh sách tài liệu (Data Inventory)

| # | Tên tài liệu | Nguồn (Source URL) | Ngày lấy / Phiên bản | Số ký tự | Metadata đã gán |
|---|--------------|------------|--------------------|----------|-----------------|
| 1 | Những quy định chung về trả hàng và hoàn tiền | [Shopee Help Center](https://help.shopee.vn/portal/4/article/188931-%5BTr%E1%BA%A3-h%C3%A0ng/Ho%C3%A0n-ti%E1%BB%81n%5D-Nh%E1%BB%AFng-quy-%C4%91%E1%BB%8Bnh-chung-v%E1%BB%81-Tr%E1%BA%A3-h%C3%A0ng/Ho%C3%A0n-ti%E1%BB%81n-c%E1%BB%A7a-Shopee) | 2026-09-19 / not-stated | 1.4K | buyer, returns-policy, vi |
| 2 | Trách nhiệm bảo hành của người bán | [Shopee Help Center](https://help.shopee.vn/portal/4/article/77245) | 2026-09-19 / not-stated | 1.0K | seller, warranty-policy, vi |
| 3 | Thời hạn yêu cầu trả hàng và hoàn tiền | [Shopee Đảm Bảo](https://help.shopee.vn/portal/4/article/79314) | 2026-09-19 / not-stated | 0.9K | buyer, returns-deadline, vi |
| 4 | Quy trình gửi yêu cầu trả hàng và hoàn tiền | [Shopee Help Center](https://help.shopee.vn/portal/4/article/79233) | 2026-09-19 / not-stated | 0.9K | buyer, returns-process, vi |
| 5 | Thời hạn phản hồi yêu cầu trả hàng của người bán | [Shopee Help Center](https://help.shopee.vn/portal/4/article/77251) | 2026-09-19 / 2026-03-04→2026-03-11 | 1.0K | seller, seller-obligations, vi |
| 6 | Chi phí vận chuyển khi trả hàng | [Shopee Help Center](https://help.shopee.vn/portal/4/article/77251) | 2026-09-19 / 2026-03-04→2026-03-11 | 1.0K | seller, return-shipping, vi |
| 7 | Điều kiện bảo hành cơ bản | [Shopee Help Center](https://help.shopee.vn/portal/4/article/77245) | 2026-09-19 / not-stated | 0.9K | buyer, warranty-conditions, vi |

**Danh sách kiểm tra quản trị dữ liệu (Data governance checklist):**
- [x] Tập tài liệu (Corpus) hiện chỉ dùng nội dung diễn giải từ nguồn công khai, không chứa dữ liệu cá nhân, thông tin đăng nhập hoặc tài liệu nội bộ; nhóm cần xác nhận quy định sử dụng nguồn với giảng viên trước khi nộp.
- [x] Mỗi tài liệu có `source_url`, `retrieved_at`, `document_version` (hoặc ngày hiệu lực) trong metadata.

> Ghi chú: có 7 file tài liệu cục bộ và 5 URL chính thức duy nhất; một URL được tách thành hai tài liệu theo góc nhìn/thuộc tính cần benchmark (ví dụ thời hạn phản hồi và chi phí vận chuyển).

### Cấu trúc Metadata (Metadata Schema)

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho truy xuất (retrieval)? |
|----------------|------|---------------|-------------------------------|
| `audience` | string | `buyer` / `seller` | Lọc đúng đối tượng hỏi |
| `category` | string | `returns-deadline` | Tách loại chính sách |
| `language` | string | `vi` | Hỗ trợ kiểm tra ngôn ngữ |
| `source_url` | string | URL Shopee Help Center | Truy vết nguồn |
| `retrieved_at` | date | `2026-09-19` | Kiểm tra độ mới |
| `document_version` | string | `not-stated` hoặc ngày hiệu lực | Phân biệt phiên bản |

---

## 2. Thiết kế chiến lược (Strategy Design) — Nhóm (15 điểm)

> Mỗi thành viên thử **một chiến lược khác nhau** trên cùng bộ tài liệu; nhóm tổng hợp và so sánh ở đây.

### Phân tích đường cơ sở (Baseline Analysis)

Thống kê trên toàn bộ 7 tài liệu của corpus; kết quả benchmark chi tiết nằm ở `report/BENCHMARK_GEMINI.md`:

| Tài liệu | Chiến lược (Strategy) | Số lượng Chunk | Độ dài trung bình | Giữ được ngữ cảnh không? |
|-----------|----------|-------------|------------|-------------------|
| 7 tài liệu | FixedSizeChunker (`fixed_size`) | 15 | 276.5 | Giữ được overlap nhưng có thể cắt giữa câu |
| 7 tài liệu | SentenceChunker (`by_sentences`) | 16 | 234.1 | Dễ đọc, giữ ranh giới câu |
| 7 tài liệu | RecursiveChunker (`recursive`) | 16 | 234.1 | Ưu tiên paragraph/heading, giữ cấu trúc tốt |
| 7 tài liệu | Heading-aware custom | 16 | 256.1 | Giữ heading cùng nội dung điều khoản |

### Chiến lược của từng thành viên

> Mỗi thành viên điền một khối dưới đây (copy thêm nếu nhóm có nhiều hơn 3 người).

**Thành viên 1 — [Tên]**
- **Loại chiến lược:** [FixedSize / Sentence / Recursive / custom]
- **Mô tả & lý do chọn cho chủ đề này:** Fixed-size với overlap giúp giữ thông tin ở ranh giới chunk, phù hợp khi điều khoản dài.
- **Code snippet (nếu custom):**
```python
# Dán mã nguồn (implementation) vào đây
```

**Thành viên 2 — [Tên]**
- **Loại chiến lược:**
- **Mô tả & lý do chọn:** Sentence chunking giữ câu hoàn chỉnh, dễ đọc và dễ kiểm tra nguồn.
- **Code snippet (nếu custom):**

**Thành viên 3 — [Tên]**
- **Loại chiến lược:** Heading-aware custom chunker
- **Mô tả & lý do chọn:** Tách theo heading/section của chính sách, sau đó dùng recursive fallback cho section dài; heading được gắn lại vào các mảnh con để không mất ngữ cảnh.
- **Code snippet (nếu custom):**

### So Sánh Giữa Các Thành Viên

| Thành viên | Chiến lược (Strategy) | Điểm truy xuất (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Thành viên 1 | Fixed-size | 5/5 gold ở top-1 với Gemini | Có overlap | Có thể cắt câu |
| Thành viên 2 | Sentence | 5/5 gold ở top-1 với Gemini | Chunk dễ đọc | Cần nhiều chunk hơn fixed-size |
| Thành viên 3 | Heading-aware custom | 5/5 gold ở top-1 với Gemini | Giữ cấu trúc điều khoản | Section dài cần fallback |

**Chiến lược nào tốt nhất cho chủ đề này? Tại sao?**
Với `gemini-embedding-001`, cả bốn chiến lược đều có gold chunk ở top-1 cho 5/5 câu hỏi. Vì tập corpus còn nhỏ và câu hỏi bám sát gold answer, chưa thể kết luận chiến lược thắng chỉ từ recall@3; nhóm sẽ so sánh thêm độ mạch lạc chunk, câu hỏi khó hơn và corpus lớn hơn.

---

## 3. Câu hỏi đánh giá & Chất lượng truy xuất (Retrieval Quality) — Nhóm (10 điểm)

### Câu hỏi đánh giá & Câu trả lời chuẩn (nhóm thống nhất)

> **Đúng 5 câu hỏi**, đa dạng, có thể kiểm chứng; **ít nhất 1 câu** cần lọc metadata mới trả lời tốt. Đây là bộ câu hỏi chung cho mọi thành viên chạy.

| # | Câu hỏi (Query) | Câu trả lời chuẩn (Gold Answer) | Chunk nào chứa thông tin? |
|---|-------|-------------------------------|--------------------------|
| 1 | Trong bao lâu người mua có thể yêu cầu trả hàng hoặc hoàn tiền? | 15 ngày kể từ khi đơn cập nhật giao hàng thành công trong trường hợp phù hợp. | `buyer-return-window#0` |
| 2 | Người bán phải phản hồi yêu cầu trả hàng trong bao lâu? | Trong vòng 02 ngày lịch hoặc thời hạn khác Shopee quy định. | `seller-return-response#0` |
| 3 | Nếu yêu cầu được chấp nhận thì tiền hoàn thường được chuyển trong bao lâu? | Khoảng 1–14 ngày làm việc, tùy phương thức thanh toán. | `refund-request-process#1` |
| 4 | Các điều kiện cơ bản để được bảo hành là gì? | Còn thời hạn, còn tem/phiếu và lỗi kỹ thuật không do người mua gây ra. | `warranty-conditions#0` |
| 5 | Trong một số trường hợp, ai chịu chi phí vận chuyển chiều hoàn trả? | Người bán chịu trong các trường hợp được Shopee chấp thuận theo chính sách. | `return-shipping-costs#0` |

### Tổng hợp chất lượng truy xuất của nhóm

> Cách chấm (theo `docs/SCORING.md`): **2 điểm/câu** — top-3 chứa chunk liên quan + agent trả lời đúng (2), có liên quan nhưng thiếu/không ở top-1 (1), không có trong top-3 (0).

| # | Câu hỏi | Chiến lược tốt nhất cho câu này | Có chunk liên quan trong top-3? | Ghi chú |
|---|---------|-------------------------------|-------------------------------|---------|
| 1 | Buyer return window | Cả 4 chiến lược | Có, top-1 với Gemini embedding |
| 2 | Seller response deadline | Cả 4 chiến lược | Có, top-1 với Gemini embedding |
| 3 | Refund timing | Cả 4 chiến lược | Có, top-1 với Gemini embedding |
| 4 | Warranty conditions | Cả 4 chiến lược | Có, top-1 với Gemini embedding |
| 5 | Seller return shipping | Cả 4 chiến lược | Có, top-1 với Gemini embedding |

**Lọc bằng metadata có giúp ích không? Ở câu hỏi nào?**
Có. Filter `audience` được áp dụng cho cả 5 câu benchmark để loại chunk sai đối tượng trước khi xếp hạng. Điều này làm rõ use case buyer/seller; tuy nhiên filter quá chặt có thể làm mất tài liệu `both`, nên cần kiểm tra A/B giữa filtered và unfiltered khi corpus được mở rộng.

---

## 4. Thuyết trình (Demo) & Bài học nhóm — Nhóm (5 điểm)

**Những phân tích (insights) hay nhất nhóm sẽ trình bày:**
- Retrieval đúng chủ đề chưa chắc chứa đúng con số trả lời; cần kiểm tra nội dung chunk chứ không chỉ `doc_id`.
- Metadata filter có giá trị khi cùng một chủ đề có chính sách khác nhau cho buyer và seller.
- Gemini embedding đưa 5/5 gold chunk lên top-1 ở cả bốn chiến lược; tuy nhiên score không đủ để tuyên bố một chiến lược thắng khi corpus quá nhỏ.

**Bài học rút ra khi so sánh trong nhóm:**
Chunker quyết định đơn vị bằng chứng mà vector store có thể trả về. Sentence giữ khả năng đọc, fixed-size giữ overlap, còn recursive tận dụng cấu trúc văn bản; vì vậy cần đánh giá cả precision, coherence và grounding.

**Nếu làm lại, nhóm sẽ thay đổi gì trong chiến lược dữ liệu (data strategy)?**
Nhóm sẽ bổ sung thêm nguồn chính thức cùng chủ đề, tạo thêm câu hỏi có nhiễu và chạy A/B metadata filter. Mỗi gold answer sẽ gắn với một câu hoặc điều khoản đặc trưng để tránh chấm nhầm chỉ vì cùng `doc_id`.

---

## Tự Đánh Giá (Phần Nhóm)

| Tiêu chí | Điểm tự đánh giá |
|----------|-------------------|
| Lựa chọn tài liệu (Document Set Quality) | / 10 |
| Thiết kế chiến lược (Strategy Design) | / 15 |
| Chất lượng truy xuất (Retrieval Quality) | / 10 |
| Thuyết trình (Demo) | / 5 |
| **Tổng phần nhóm** | **/ 40** |
