# Day 18 Lab — Human-Centered AI Design

**Trần Nguyễn Anh Thư - 2A202600915**

## Feature: Claim Verification & Routing (Step ⑧⑨ — Academic Research Assistant)

**Dự án gốc:** AI hỗ trợ literature review & research-gap detection (LLM + RAG + Semantic Scholar API)
**Feature slice được chọn:** Claim Verification & Routing — màn hình `ClaimVerifier.jsx`


---

## 1. Tổng quan bối cảnh

Sau khi LLM viết xong nội dung cho 1 theme (Step ⑦), hệ thống chạy verification 3 tầng cho từng claim trong bài (Step ⑧):
- **Case A (snippet):** Semantic Scholar trả về đoạn trích trực tiếp — độ tin cậy cao nhất (~30% coverage)
- **Case B (arxiv fulltext):** Lấy toàn văn qua ar5iv — độ tin cậy cao (~80%+ coverage cho CS/AI)
- **Case C (abstract-only):** Chỉ có tóm tắt — không bao giờ trả "Supported", luôn cần con người xem lại

Routing logic (Step ⑨) quyết định: claim nào tự giữ, claim nào tự xóa (có log + khôi phục), claim nào bắt buộc đưa cho user quyết. Feature slice này thiết kế lại toàn bộ trải nghiệm xung quanh việc user xem, hiểu, và can thiệp vào quyết định của AI ở bước này.

---

## 2. Bản đồ 5 kịch bản theo Generic Scenario Pack (P0–P6)

| # | Pack | Loại | Tên kịch bản |
|---|---|---|---|
| 1 | P0 | Onboarding (bắt buộc) | Lần đầu mở màn hình Claim Verifier |
| 2 | P2 | During — multiple valid options | Claim "Partially Supported" |
| 3 | P3 | During — agency boundary | AI tự xóa claim "Unsupported" |
| 4 | P4 | Error/Uncertainty | Nhiều claim Case C (abstract-only) bị đẩy review |
| 5 | P5/P6 | Error & Recovery | AI đoán sai "contrasting intent" + vòng feedback |

Đủ điều kiện tối thiểu của đề: 1 onboarding + ≥2 during + ≥2 error/recovery, toàn bộ nối liền thành một luồng Claim Verification thống nhất (không phải các đoạn rời rạc).

---

## 3. Chi tiết 5 kịch bản

### Kịch bản 1 — P0: Onboarding

**Bối cảnh:** User vừa hoàn thành Step ⑦, hệ thống verify xong, chuyển sang `ClaimVerifier.tsx` lần đầu trong session.

**Trạng thái 1 — AI đang xử lý:** Thanh progress "Đang kiểm tra nguồn cho 12 claims..." kèm số đếm. Không có nút bấm — hành động AI tự làm (Act), không ảnh hưởng nội dung, dễ hoàn tác.

**Trạng thái 2 — Màn hình chính khi verify xong:**
- Header giải thích 1 lần: "AI đã kiểm tra từng câu trong bài viết với nguồn gốc của nó. Mỗi câu được gắn 1 trong 4 mức: Đã xác minh / Xác minh một phần / Không xác minh được / Cần bạn xem lại."
- Danh sách claim dạng card: đoạn text claim, badge nguồn (snippet/arxiv/abstract, abstract có icon "độ tin cậy thấp"), trạng thái, nút hành động — chỉ hiện nút với claim cần user quyết; claim "Supported + nguồn mạnh" chỉ có nhãn "Đã tự động giữ lại" + link xem nguồn.
- Dòng tóm tắt cuối danh sách: "8 claims tự động giữ, 3 claims cần bạn xem, 1 claim đã tự xóa."

**User bấm được gì:** Bấm badge nguồn → mở rộng xem trích đoạn gốc (evidence/explainability layer). Với claim cần quyết: Approve/Reject. Mọi xóa khôi phục được qua tab "Đã xóa".

**Vì sao thiết kế vậy:** AI không tự quyết hết (hậu quả sai ảnh hưởng academic integrity) nhưng cũng không hỏi mọi claim (tránh quá tải) — tự lọc theo độ tin cậy 3-tier.

---

### Kịch bản 2 — P2: Claim "Partially Supported"

**Bối cảnh:** Claim ở mức "Partially Supported" — nguồn tìm được chỉ khớp một phần claim (ví dụ claim tổng quát hóa quá mức so với nguồn).

**Màn hình:** Badge cam "Partially Supported". Hiển thị 2 đoạn song song: "Claim viết:" / "Nguồn tìm được:" — để user tự so sánh, AI không diễn giải hộ. Dòng gợi ý: "AI thấy nguồn chỉ khớp một phần với claim này — có thể do claim tổng quát hóa quá mức, hoặc do nguồn chưa đủ chi tiết."

**3 nút hành động:** Giữ nguyên / Sửa claim (mở ô edit) / Xóa. Ba lựa chọn vì đây không có 1 đáp án đúng duy nhất (P2 — multiple valid options).

**Act/Ask:** Ask bắt buộc — AI tự nhận mơ hồ, hậu quả sai liên quan academic integrity.

---

### Kịch bản 3 — P3: AI tự xóa claim "Unsupported"

**Bối cảnh:** Claim "Unsupported" (nguồn xác nhận mâu thuẫn) → hệ thống tự xóa + ghi log, không hỏi trước.

**Màn hình:** Claim biến khỏi danh sách chính, để lại dòng mờ: "Đã tự động xóa 1 câu do không tìm được nguồn ủng hộ — [Xem chi tiết]". Bấm vào → câu đã xóa, nguồn đã kiểm tra + lý do mâu thuẫn, 2 nút: Khôi phục / Đồng ý xóa. Tất cả xóa tự động gom vào tab "Đã xóa tự động (3)".

**Vì sao là Act (có giám sát), không phải Ask:** Tín hiệu mâu thuẫn rõ ràng nhất trong 3-tier verification (dễ phát hiện đúng), dễ hoàn tác (chỉ ẩn khỏi markdown, không xóa dữ liệu gốc), hậu quả sai chỉ là thiếu sót sửa được. Nhưng không Act âm thầm — luôn hiện log + nút khôi phục ngay tại chỗ.

---

### Kịch bản 4 — P4: Nhiều claim Case C (abstract-only) bị đẩy review

**Bối cảnh:** Case C không bao giờ trả "Supported". Với theme dùng nhiều paper ít phổ biến, nhiều claim hợp lệ vẫn bị đẩy vào hàng đợi review — không phải vì sai, mà vì thiếu dữ liệu.

**Màn hình:** Badge riêng màu xám-vàng "Chưa đủ dữ liệu để xác minh" (phân biệt rõ với cam/đỏ). Dòng giải thích: "AI chỉ tìm được tóm tắt (abstract) của nguồn này, không có toàn văn — nên không thể xác minh chi tiết. Không có nghĩa là claim sai." Nếu >30% claim trong theme thuộc case này → banner tổng đầu trang gợi ý user tự kiểm tra kỹ hơn (tín hiệu Mức 2 — gợi ý theo ngữ cảnh).

**Vì sao tách badge riêng:** Lý do mơ hồ ở đây khác (thiếu dữ liệu, không phải nguồn mâu thuẫn) — gộp chung với "AI có thể sai" sẽ làm user hiểu sai bản chất rủi ro và cách xử lý (tự tìm thêm thông tin, không phải nghi ngờ AI).

**Act/Ask:** Ask — AI biết rõ giới hạn của mình (Case C luôn route review theo thiết kế), không có lý do tự quyết.

---

### Kịch bản 5 — P5/P6: AI đoán sai "contrasting intent" + vòng feedback

**Bối cảnh:** Hệ thống phát hiện claim có vẻ mâu thuẫn với lập luận chung của theme → đẩy vào `human_review_priority` bất kể kết quả verify nguồn. Đây là suy đoán dễ sai nhất — claim trái chiều có thể là chủ đích của user (ví dụ viết phần "hạn chế của phương pháp").

**Màn hình:** Badge "🔺 Ưu tiên xem" + giải thích: "Câu này có vẻ trái chiều với phần lập luận chung của theme — AI đẩy lên ưu tiên để bạn xác nhận có đúng ý không." 2 nút: **Đồng ý** (claim này cố ý trái chiều, không phải lỗi) / **Không đồng ý** (AI hiểu sai, đây không trái chiều).

Khi bấm "Không đồng ý" → ô input không bắt buộc: "Bạn có thể cho biết vì sao không? (giúp AI tránh lặp lại)" (explicit feedback). Badge biến mất ngay trong session (phản hồi tức thì).

**Vòng Feedback → Recovery:** Ở lần verify theme tiếp theo, nếu pattern tương tự xuất hiện lại, AI hiển thị: "Trước đó bạn đã đánh dấu 1 case tương tự là 'không phải lỗi' — AI vẫn đẩy lên review vì muốn chắc chắn, nhưng đã giảm độ ưu tiên." AI **không tắt hẳn** cảnh báo chỉ sau 1 feedback, vì hành vi đơn lẻ không phải ground truth tuyệt đối.

**Act/Ask:** Ask — suy đoán "ý định trái chiều" có độ tin cậy thấp nhất trong toàn pipeline (dựa ngữ nghĩa, không dựa verify nguồn).

---

## 4. Bảng Act / Ask / Don't Act (tổng hợp)

| Tình huống | Mức độ | Vì sao |
|---|---|---|
| Verify nguồn cho từng claim (chạy ngầm) | **Act** | Bước nội bộ, không ảnh hưởng nội dung cuối, không cần user thấy |
| Claim Supported + nguồn mạnh (snippet/arxiv) | **Act** | Độ tin cậy cao nhất theo 3-tier, dễ hoàn tác (user vẫn đọc lại được) |
| Claim Unsupported (nguồn mâu thuẫn rõ) | **Act có giám sát** | Tín hiệu mâu thuẫn rõ ràng, hậu quả sai chỉ là thiếu sót, có nút khôi phục ngay |
| Claim Partially Supported | **Ask** | AI tự nhận mơ hồ, hậu quả sai liên quan academic integrity |
| Claim chỉ có abstract (Case C) | **Ask** | Thiếu dữ liệu để xác minh, AI biết rõ giới hạn của mình |
| Claim "contrasting intent" (đẩy ưu tiên review) | **Ask** | Suy đoán có độ tin cậy thấp nhất (dựa ngữ nghĩa, không dựa verify nguồn) |
| Tự động xóa hẳn dữ liệu gốc (không cho khôi phục) | **Don't Act** | Không nằm trong thiết kế — mọi xóa đều giữ log + khôi phục được |

---

## 5. Bảng 2×2 Feedback Matrix (đầy đủ rationale 8 điểm)

| | **Explicit** | **Implicit** |
|---|---|---|
| **User → System** | Bấm "Đồng ý"/"Không đồng ý" ở claim contrasting-intent + ô input lý do (kịch bản 5) | Bấm "Sửa claim" thay vì "Giữ nguyên"/"Xóa" ở claim Partially Supported (kịch bản 2) |
| **System → User** | Banner tổng khi >30% claim là Case C (kịch bản 4) | Progress bar verify + badge màu phân biệt loại uncertainty (kịch bản 1, 4) |

### 5.1 User → System / Explicit — "Đồng ý / Không đồng ý" + lý do
1. **Thời điểm:** Ngay sau khi bấm, lúc lý do còn rõ trong đầu user.
2. **Mục tiêu:** Biết AI có đang suy đoán "ý định trái chiều" sai pattern không.
3. **Vì sao explicit:** Lý do "không đồng ý" có thể rất khác nhau — chỉ suy luận từ hành vi sẽ không phân biệt được.
4. **Cách hiểu khác:** User có thể bấm vì thực sự nghĩ AI sai, hoặc vì muốn bỏ qua nhanh không đọc kỹ.
5. **Phản hồi tức thì:** Badge "Ưu tiên xem" biến mất ngay trong session.
6. **Dùng để làm gì:** Giảm priority cho pattern tương tự ở lần verify sau, không tắt hẳn cảnh báo.
7. **User biết/kiểm soát:** Có — dòng "Trước đó bạn đã đánh dấu..." cho thấy rõ phản hồi cũ đang ảnh hưởng gì.
8. **Overload:** Không — ô lý do không bắt buộc.

### 5.2 User → System / Implicit — bấm "Sửa claim"
1. **Thời điểm:** Ngay khi user click "Sửa claim" thay vì giữ/xóa.
2. **Mục tiêu:** Phát hiện claim Partially Supported thường cần sửa hơn giữ/xóa nguyên trạng.
3. **Vì sao implicit:** Hành vi tự nhiên trong luồng làm việc, không cần hỏi thêm để không gây gián đoạn.
4. **Cách hiểu khác:** User có thể sửa vì lý do văn phong, không liên quan độ tin cậy nguồn.
5. **Phản hồi tức thì:** Ô edit mở ngay, không cần xác nhận thêm.
6. **Dùng để làm gì:** Theo dõi tỷ lệ Partially Supported cần sửa, ở mức thống kê tổng hợp.
7. **User biết/kiểm soát:** Không trực tiếp — đây là điểm rủi ro cần lưu ý, nên chỉ dùng implicit signal này ở mức thống kê, không dùng để tự động cá nhân hóa quyết định cho từng user.
8. **Overload:** Không — là hành động bình thường trong luồng.

### 5.3 System → User / Explicit — banner tổng Case C
1. **Thời điểm:** Ngay khi mở danh sách claim, trước khi đọc chi tiết.
2. **Mục tiêu:** Giúp user hiểu nhanh vì sao có nhiều claim cần review, tránh suy luận sai "bài có vấn đề."
3. **Vì sao nói rõ trực tiếp:** Cảnh báo quan trọng cần nói thẳng, không để user tự đoán qua màu badge.
4. **Cách hiểu khác (rủi ro phía người nhận):** User có thể hiểu nhầm "nhiều claim không xác minh được" = "bài có nhiều lỗi" nếu không giải thích rõ.
5. **Phản hồi tức thì:** Banner hiện ngay, mất khi user đã dismiss hoặc xem qua hết case trong theme.
6. **Dùng để làm gì:** Định hướng nơi cần đọc kỹ, không thay quyết định của user.
7. **User biết/kiểm soát:** Có thể dismiss, không lặp lại trong cùng theme.
8. **Overload:** Chỉ hiện khi vượt ngưỡng 30%, tránh hiện liên tục không cần thiết.

### 5.4 System → User / Implicit — progress bar + badge màu
1. **Thời điểm:** Liên tục trong suốt quá trình verify và đọc danh sách.
2. **Mục tiêu:** Giữ user luôn biết trạng thái xử lý/loại uncertainty mà không cần đọc text dài.
3. **Vì sao implicit:** Tín hiệu thị giác nền, không cần đọc để hiểu tiến trình.
4. **Cách hiểu khác:** Màu sắc có thể bị hiểu lầm nếu user không quen hệ thống màu — cần giải thích 1 lần ở onboarding (đã có trong header kịch bản 1).
5. **Phản hồi tức thì:** Cập nhật theo thời gian thực.
6. **Dùng để làm gì:** Cải thiện UX hiển thị, không thu thập/lưu trữ phân tích hành vi.
7. **User biết/kiểm soát:** Không cần kiểm soát — đây thuần là hiển thị trạng thái, không phải thu thập dữ liệu.
8. **Overload:** Không — là visual nền, không yêu cầu hành động.

---

## 6. Lớp Evidence / Explainability

Ở mọi kịch bản, user đều có thể bấm vào badge nguồn để xem trích đoạn gốc (snippet/arxiv fulltext/abstract) mà AI dùng để verify — không cần giải thích cơ chế model, chỉ cần đủ thông tin để user tự quyết định giữ/sửa/xóa. Đây thỏa yêu cầu "ít nhất 1 kịch bản cho phép user xác minh qua evidence hiển thị" — áp dụng xuyên suốt cả 5 kịch bản, rõ nhất ở kịch bản 1 (xem nguồn) và kịch bản 2 (so sánh song song claim vs nguồn).

---

## 7. 10-câu hỏi Design Rationale theo từng kịch bản

### Kịch bản 1 — Onboarding
1. **AI đang biết gì?** Kết quả verify 3-tier (snippet/arxiv/abstract) cho từng claim, nguồn gốc trích dẫn.
2. **AI chưa biết/không chắc gì?** Claim có đúng về học thuật tổng thể không — chỉ biết nguồn có khớp văn bản hay không.
3. **AI đang giả định gì?** Semantic Scholar/arXiv index đầy đủ và đúng cho paper liên quan.
4. **Điều gì có thể sai?** Verify nhầm đoạn trích, hoặc claim đúng nhưng diễn đạt khác cách nguồn viết nên bị đánh giá thấp.
5. **Hậu quả nếu sai & ai bị ảnh hưởng?** User mất thời gian review oan, hoặc tin sai vào claim được tự giữ.
6. **Sai sót dễ phát hiện/hoàn tác không?** Có — luôn xem được nguồn gốc, mọi xóa khôi phục được.
7. **Act/Ask/Don't Act? Vì sao?** Mix theo độ tin cậy — Act cho case rõ, Ask cho case mơ hồ (xem bảng mục 4).
8. **User kiểm tra/sửa/từ chối/hoàn tác ở đâu?** Trên card claim — bấm badge xem nguồn, nút Approve/Reject.
9. **Phản hồi nào được thu thập?** Approve/Reject mỗi claim; mở rộng xem nguồn là tín hiệu implicit về claim cần xem kỹ.
10. **Flow giúp user tiếp tục mục tiêu thế nào?** Sau khi xử lý xong danh sách, user có bài review hoàn chỉnh để tiếp tục Step ⑨/⑩.

### Kịch bản 2 — Partially Supported
1. **AI biết gì?** Có nguồn liên quan nhưng chỉ khớp một phần nội dung claim.
2. **Chưa biết/không chắc gì?** Mức độ tổng quát hóa của claim có đúng ý tác giả gốc không.
3. **Giả định gì?** Khác biệt claim–nguồn là do tổng quát hóa, không phải do AI hiểu sai văn bản.
4. **Điều gì có thể sai?** AI đánh giá "khớp một phần" sai — nguồn có thể khớp hoàn toàn nhưng đoạn trích chưa đủ.
5. **Hậu quả & ai ảnh hưởng?** Academic integrity nếu giữ claim phóng đại; mất nội dung hữu ích nếu xóa nhầm.
6. **Dễ phát hiện/hoàn tác?** Tương đối dễ — user đọc song song để tự đánh giá; giữ/sửa/xóa đều đơn giản.
7. **Act/Ask?** Ask bắt buộc.
8. **User kiểm tra ở đâu?** 2 đoạn song song claim vs nguồn ngay trên card.
9. **Phản hồi thu thập?** Lựa chọn Giữ/Sửa/Xóa — tín hiệu rõ về việc claim dạng này cần xử lý đặc biệt.
10. **Flow tiếp tục mục tiêu?** Sau xử lý, claim cập nhật trạng thái cuối, bài tiếp tục sang các claim khác.

### Kịch bản 3 — AI tự xóa Unsupported
1. **AI biết gì?** Nguồn xác nhận mâu thuẫn trực tiếp với claim.
2. **Chưa biết/không chắc gì?** Claim có giá trị khác (ví dụ trích để phản biện) mà AI không nhận ra ý đồ.
3. **Giả định gì?** Claim được viết với ý định khẳng định, không phải dẫn ra để bác bỏ.
4. **Điều gì có thể sai?** AI xóa nhầm câu mà user chủ đích viết để phản biện/so sánh.
5. **Hậu quả & ai ảnh hưởng?** Mất nội dung user chủ đích viết, nhưng có thể khôi phục nên thiệt hại thấp.
6. **Dễ phát hiện/hoàn tác?** Rất dễ — có log + nút khôi phục ngay.
7. **Act/Ask?** Act có giám sát (log + undo).
8. **User kiểm tra ở đâu?** Tab "Đã xóa tự động", nút Khôi phục/Đồng ý xóa.
9. **Phản hồi thu thập?** Khôi phục = tín hiệu AI xóa sai; Đồng ý xóa = tín hiệu AI đúng.
10. **Flow tiếp tục mục tiêu?** Bài tự động gọn lại, user không cần dừng xử lý ngay, có thể xem cuối buổi.

### Kịch bản 4 — Case C (abstract-only)
1. **AI biết gì?** Chỉ có abstract, không có toàn văn nguồn.
2. **Chưa biết/không chắc gì?** Phần chi tiết cụ thể có ủng hộ claim hay không.
3. **Giả định gì?** Thiếu toàn văn = không thể xác minh, nên không tự tin báo Supported.
4. **Điều gì có thể sai?** Abstract thực ra đã đủ thông tin nhưng hệ thống quá conservative, gây review thừa.
5. **Hậu quả & ai ảnh hưởng?** User mất thời gian review claim thực ra ổn — ảnh hưởng hiệu suất, không sai bài.
6. **Dễ phát hiện/hoàn tác?** Dễ — user đọc abstract trực tiếp để tự quyết nhanh.
7. **Act/Ask?** Ask.
8. **User kiểm tra ở đâu?** Mở rộng card xem abstract, banner tổng quan đầu trang.
9. **Phản hồi thu thập?** Giữ/Xóa — cũng là tín hiệu để cân nhắc tích hợp thêm nguồn (OpenAlex) trong tương lai.
10. **Flow tiếp tục mục tiêu?** User xử lý nhanh nhóm claim này nhờ hiểu đúng lý do, không bị chặn lâu.

### Kịch bản 5 — Contrasting intent + recovery
1. **AI biết gì?** Câu có ngữ nghĩa khác biệt/ngược với lập luận chung của theme.
2. **Chưa biết/không chắc gì?** Sự khác biệt đó có phải chủ đích của user hay không.
3. **Giả định gì?** Trái chiều ngữ nghĩa = khả năng cao cần xem lại.
4. **Điều gì có thể sai?** Đoán nhầm chủ đích viết — user có thể chủ đích viết phần phản biện/hạn chế.
5. **Hậu quả & ai ảnh hưởng?** Làm phiền user không cần thiết; lặp lại nhiều lần dễ gây mất tin tưởng vào hệ thống.
6. **Dễ phát hiện/hoàn tác?** Dễ — chỉ cần bấm Đồng ý/Không đồng ý, không có hành động phá hủy nào xảy ra trước.
7. **Act/Ask?** Ask — độ tin cậy suy đoán thấp nhất trong toàn pipeline.
8. **User kiểm tra/từ chối ở đâu?** 2 nút Đồng ý/Không đồng ý + ô input lý do (không bắt buộc).
9. **Phản hồi thu thập?** Explicit feedback có lý do, dùng để giảm priority lần sau (không tắt hẳn cảnh báo).
10. **Flow tiếp tục mục tiêu?** Badge biến mất ngay, user tiếp tục đọc các claim còn lại không bị gián đoạn lâu.

---

## 8. Tổng kết theo điều kiện tối thiểu của đề bài

| Điều kiện | Đáp ứng |
|---|---|
| Có onboarding | ✅ Kịch bản 1 |
| ≥4 kịch bản ngoài onboarding | ✅ Kịch bản 2, 3, 4, 5 |
| Đủ slice Onboarding→During→After→Feedback | ✅ Onboarding (1) → During (2, 3) → After (xem lại tab Đã xóa) → Feedback&Recovery (5) |
| Đủ Act/Ask/Don't Act | ✅ Có cả 3 mức, với lý giải risk/recoverability |
| ≥1 vòng feedback & recovery hoàn chỉnh | ✅ Kịch bản 5 |
| Đủ 4 loại trong feedback matrix | ✅ Mục 5 |
| ≥1 lớp evidence/explainability | ✅ Mục 6 |
| Rationale đi kèm flow | ✅ Mục 7 |
