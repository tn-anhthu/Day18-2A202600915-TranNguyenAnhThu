# Day 18 — Human-Centered AI Design
## Lab Report: AI Literature Review System

> **Track:** Track 1 — AI Product Management  
> **Họ và tên:** Trần Nguyễn Anh Thư
> **Mã học viên:** 2A202600915
> **Dự án:** Option 4 — Dự án nhóm đang phát triển  
> **Prototype:** `day18-prototype.html`

---

## 1. Người dùng, vấn đề và lát cắt tính năng

### Ai là người dùng?

Hai nhóm người dùng chính:

- **Researchers** — người đang viết bài báo khoa học, cần literature review chặt chẽ về nguồn trích dẫn
- **Thesis students** — sinh viên sau đại học cần tổng quan tài liệu cho luận văn, thường không quen với quy trình tìm paper thủ công

Điểm chung: họ **biết mình cần gì** (literature review về một topic cụ thể) nhưng **không có thời gian và công cụ** để đọc hàng trăm paper, tổng hợp claims và format citations một cách thủ công.

### Vấn đề cụ thể

Literature review truyền thống có 3 pain point lớn:

1. **Tìm paper mất rất nhiều thời gian** — query Google Scholar, đọc abstract, lọc thủ công
2. **Citations dễ sai hoặc thiếu** — nhất là khi viết dưới áp lực deadline
3. **Khó kiểm soát hướng nghiên cứu** — nếu AI tự generate toàn bộ mà không có checkpoint, output có thể lệch hoàn toàn so với góc mà researcher muốn

### Lát cắt được chọn

> **"User được can thiệp vào định hướng của hệ thống xuyên suốt quy trình tạo literature review"**

Đây là lát cắt từ lúc user nhập topic cho đến khi nhận được full literature review — nhưng quan trọng hơn là **3 điểm dừng có kiểm soát** ở giữa, nơi user có thể chỉnh hướng trước khi AI đi tiếp.

**Tại sao không chọn lát cắt khác?**

Lúc đầu, nhóm nghĩ đến lát cắt "save → edit LaTeX → export" — nhưng lát cắt đó chủ yếu là user action, AI đóng vai trò passive. Không có đủ AI interaction moments để thiết kế. Lát cắt hiện tại phong phú hơn vì AI đưa ra quyết định ở mỗi bước và user phải có khả năng kiểm tra, chỉnh sửa, rescue.

---

## 2. Kiến trúc hệ thống: 4 Requests, 3 Interrupts

Hệ thống hoạt động theo pipeline 4 API requests với 3 điểm dừng bắt buộc:

```
R1: POST /api/research/stream
    Body: {"query": "...", "thread_id": null}
    → AI tạo plan: 5 sub-queries + nguồn tìm paper
    ⏸ INTERRUPT 1 — Plan Review (user approve/edit)

R2: POST /api/research/resume
    Body: {"thread_id": "<id>", "resume_value": true}
    → AI fetch papers, đọc abstracts, cluster thành outline
    ⏸ INTERRUPT 2 — Outline Approval (user approve/split/merge)

R3: POST /api/research/resume
    Body: {"thread_id": "<id>", "resume_value": true}
    → AI phân loại 57 claims: included / needs-review / removed
    ⏸ INTERRUPT 3 — Claims Routing (user review removed claims)

R4: POST /api/research/resume
    Body: {"thread_id": "<id>", "resume_value": true}
    → AI generate full literature review
    ✅ OUTPUT — Edit LaTeX, export PDF/MD/BibTeX
```

**Tại sao thiết kế theo pipeline interrupt?**

Nếu AI chạy thẳng từ query đến output (không dừng), thì khi output sai — user phải restart toàn bộ từ đầu. Chi phí rất cao. Ba interrupt cho phép user "rẽ nhánh" tại điểm rủi ro cao nhất, trước khi AI đã tiêu tốn quá nhiều compute.

---

## 3. Vòng đời trải nghiệm (4 giai đoạn bắt buộc)

```
[A] Onboarding
      ↓
[B] Trong khi AI hỗ trợ — R1 → Interrupt 1 → R2 → Interrupt 2
      ↓
[C] Sau hành động / Review — R3 → Interrupt 3 → R4 → Output
      ↓
[D] Feedback, sửa lỗi & khôi phục ↺ (quay lại kết quả tốt hơn)
```

Tất cả 6 scenarios đều nằm trong một lát cắt thống nhất — không phải 5 mini-app rời rạc.

---

## 4. Scenarios

### P0 — Onboarding (Giai đoạn A) `[Bắt buộc]`

**Bối cảnh:** Researcher lần đầu mở tính năng AI Literature Review. Họ chưa biết AI làm được gì cụ thể, cần dữ liệu gì, và giới hạn ở đâu.

**Thiết kế onboarding:**

UI hiển thị hai cột song song rõ ràng ngay từ đầu:

| ✅ AI có thể hỗ trợ | ❌ AI không tự làm |
|---|---|
| Tạo search queries từ topic | Submit/publish bài của bạn |
| Tìm và filter papers liên quan | Truy cập paper bị paywall |
| Summarize + verify claims | Đảm bảo 100% citation chính xác |
| Generate draft literature review | Xóa claim mà không hỏi bạn |
| Format citations (APA/IEEE/BibTeX) | Quyết định final thay bạn |

Ngoài ra, có **cảnh báo citation risk** nổi bật:

> ⚠️ AI có thể tạo citation không tồn tại (hallucination). Hệ thống sẽ đánh dấu các claim chưa được verify — bạn nên kiểm tra trước khi dùng.

Cảnh báo này quan trọng vì nếu researcher dùng fake citation trong paper, hậu quả là mất uy tín học thuật. Phải set expectation từ trước.

Sau onboarding, user chỉ cần làm một việc: **nhập research topic** và nhấn Bắt đầu. Không có form dài.

**Design rationale — P0:**

- **AI biết gì?** Chưa biết gì về user, topic, scope nghiên cứu
- **AI không biết gì?** Research angle cụ thể, loại output cần, nguồn ưu tiên
- **Agency:** `Don't Act` — AI không được làm gì cho đến khi user input topic
- **Expectation setting:** Show rõ giới hạn TRƯỚC khi bắt đầu, tránh tình trạng user phát hiện limitation ở bước cuối

---

### S1 — Plan Review (Giai đoạn B, Interrupt 1) `[Trong tương tác]`

**Bối cảnh:** User nhập topic "Efficient Transformer Architectures in NLP". AI gọi R1 và tạo plan gồm 5 sub-queries. Tuy nhiên, sub-query #3 quá rộng: "transformer NLP applications general survey" — nó sẽ kéo về hàng trăm paper không liên quan đến angle "efficiency" của user.

**Vấn đề thiết kế:** Nếu AI tự fetch paper với sub-query sai, toàn bộ pipeline từ R2 trở đi sẽ bị lệch. Chi phí restart rất cao.

**Giải pháp:**

- Hệ thống dừng lại và hiện **Interrupt Banner** vàng: "AI đã tạo plan gồm 5 sub-queries. Vui lòng kiểm tra trước khi AI fetch papers."
- Sub-query bị flag sẽ có màu vàng kèm ghi chú: "⚠️ Query chưa cụ thể, có thể ảnh hưởng đến kết quả tra cứu."
- User có thể click **✏️ Sửa** ngay trên query đó, edit inline và save
- Sau khi save, query chuyển sang màu xanh với dấu ✓

**Design rationale — S1:**

- **AI biết gì?** Research topic từ user. Không biết angle cụ thể (efficiency vs applications vs survey)
- **AI đang assume gì?** Query #3 assume user muốn general survey — nhưng user có thể chỉ muốn efficiency papers
- **Điều gì có thể sai?** Sub-query quá rộng → AI fetch sai papers → toàn bộ pipeline đi sai hướng từ bước đầu
- **Hậu quả nếu sai?** Rất đắt — phải restart từ đầu, mất paper results đã fetch
- **Agency:** `Ask` — bắt buộc phải review TRƯỚC khi fetch papers. Chi phí khi sai quá cao để cho AI tự quyết
- **Kiểm soát:** User có thể edit từng query, thêm query mới, chọn/bỏ chọn nguồn paper

---

### S2 — Outline Approval với Explainability (Giai đoạn B, Interrupt 2) `[Trong tương tác + Explainability]`

**Bối cảnh:** Sau khi R2 hoàn thành, AI cluster 847 papers thành 4 sections. Cluster 3 ("Attention + Positional Encoding") thực ra chứa 2 sub-topics khác nhau: Rotary/Relative position encoding (17 papers) và Cross-attention efficiency (14 papers). AI gộp lại nhưng người đọc sẽ thấy section này incoherent.

**Vấn đề thiết kế:** User không biết tại sao AI cluster theo cách này. Nếu không có explainability, user không có đủ thông tin để quyết định có nên split hay không.

**Giải pháp:**

Mỗi cluster có nút **"▸ Xem lý do AI cluster"** — khi click, expand ra một box giải thích:

> "AI grouped 31 papers nhưng phát hiện 2 sub-clusters rõ ràng: (a) Rotary/Relative position encoding (17 papers) và (b) Cross-attention efficiency (14 papers). AI gộp lại nhưng recommend Thư xem xét split thành 2 sections riêng."

Cluster bị flag màu vàng kèm nút **✂️ Split** — khi click sẽ tách ra thành 2 sections.

**Design rationale — S2:**

- **AI biết gì?** 847 papers với abstracts và keywords. Biết có 2 sub-clusters trong cluster 3
- **AI đang assume gì?** Gộp 2 sub-topics vào 1 section cho ngắn gọn — nhưng user muốn review mạch lạc hơn
- **Explainability:** "Xem lý do cluster" → show keywords chung, % overlap. User hiểu WHY, không chỉ thấy WHAT
- **Agency:** `Ask` — AI tự cluster nhưng phải hỏi user approve. Với cluster suspect, AI chủ động flag thay vì chờ user phát hiện
- **Kiểm soát:** Reorder, Split, Merge, Thêm section thủ công — user giữ toàn quyền với structure

---

### S3 — Rescue Claim bị xóa sai (Giai đoạn C, Interrupt 3) `[Failure & Recovery]`

**Bối cảnh:** R3 phân loại 57 claims. Claim "Sparse attention reduces memory complexity from O(n²) to O(n√n) for sequences >2048 tokens" bị đưa vào bucket **Removed** vì AI không tìm thấy 2+ papers verify trong database. Nhưng thực ra đây là một claim quan trọng, có trong Child et al. 2019 — AI chỉ đang miss source.

**Vấn đề với thiết kế ban đầu ("informational only"):** Nếu Interrupt 3 chỉ là thông tin (không cho rescue), user sẽ mất claim quan trọng mà không biết. Đây là lý do nhóm đổi design từ "informational" sang "có rescue loop".

**Giải pháp:**

- Hiện tổng hợp 3 buckets: **38 Included / 12 Needs Review / 7 Removed**
- Removed claims được dim nhưng KHÔNG bị xóa hoàn toàn — kèm lý do AI loại
- User có nút **🔄 Rescue** trên từng claim
- Khi click Rescue: mở panel để user paste DOI hoặc title của paper nguồn
- Sau khi submit: AI verify → nếu tìm thấy, claim move sang "Needs Review" để user confirm lại → sau đó vào "Included"
- Hệ thống confirm: "✓ Source provided: Child et al., 2019. AI đang verify..."

**Design rationale — S3:**

- **Tại sao không để "informational only"?** AI có thể loại sai claim quan trọng. "Informational only" = user mất claim mà không biết. Hậu quả: literature review thiếu evidence quan trọng
- **Agency:** `Act` cho included claims (rủi ro thấp) · `Ask` cho removed claims (hậu quả lớn nếu xóa sai)
- **Recovery loop hoàn chỉnh:** User cung cấp source → AI verify → move sang "needs review" → user confirm → included. Không có dead end
- **Implicit feedback:** Khi user rescue claim, AI ghi nhận loại source nào bị miss → cải thiện future search

---

### S4 — Citation Hallucination Recovery (Giai đoạn C/D) `[Failure & Recovery + Explainability]`

**Bối cảnh:** Trong output của R4, có một citation "[Lin et al., 2021]" được AI synthesize nhưng paper này không tồn tại trong bất kỳ database nào đã fetch. Citation nghe hợp lý nhưng là hallucination.

**Đây là failure mode nghiêm trọng nhất** trong hệ thống này vì nếu researcher dùng fake citation trong paper → academic dishonesty → có thể bị retract.

**Giải pháp:**

Citations trong output được highlight theo 3 màu:
- 🟢 Xanh: verified (tìm thấy trong Semantic Scholar hoặc Arxiv)
- 🟡 Vàng: suspect (chưa verify đủ)
- 🔴 Đỏ: không tìm thấy source

Khi user click vào `[Lin et al., 2021]` (màu đỏ), mở ra panel giải thích:

> "AI không tìm thấy paper này trong bất kỳ nguồn nào đã fetch. Paper có thể không tồn tại, hoặc AI đã synthesize citation từ nhiều nguồn khác nhau."

Kèm theo phần **"Bằng chứng AI đang dùng"**:
- Claim "O(n√n) complexity" — tìm thấy concept này trong Child et al. 2019
- Nhưng "Lin et al., 2021" cụ thể → không có record trong database
- Đây có thể là hallucinated citation

User có 3 lựa chọn:
1. 🗑️ **Xóa citation này**
2. 🔄 **Thay bằng Child et al. 2019** (paper thật có claim tương tự)
3. 🔍 **Tôi sẽ tự tìm paper**

Sau khi fix, hệ thống confirm: "✓ Đã thay [Lin et al., 2021] → [Child et al., 2019]. AI sẽ ghi nhớ để không repeat lỗi này trong section tiếp theo."

**Design rationale — S4:**

- **Điều gì có thể sai?** AI synthesize citation không tồn tại
- **Hậu quả?** Rất nghiêm trọng — ảnh hưởng uy tín nghiên cứu, có thể bị retract paper
- **Sai sót có dễ phát hiện không?** Không, vì citation nghe rất hợp lý. Do đó hệ thống phải CHỦ ĐỘNG đánh dấu, không chờ user tự phát hiện
- **Explainability:** Show "bằng chứng AI đang dùng" → user thấy AI suy luận từ đâu, không chỉ nói "citation sai"
- **Recovery:** 3 paths rõ ràng, không dead end. AI confirm đã ghi nhớ fix để không lặp lại

---

### S5 — Auto-format Citations (Agency: Act) `[Agency]`

**Bối cảnh:** Sau khi output được generate, AI phát hiện citations trong document dùng 2 style khác nhau: 3 citations theo APA, 2 citations theo IEEE. User muốn export sang LaTeX với BibTeX.

**Thiết kế:**

AI chủ động đề xuất trong một notification nhỏ màu xanh:

> "🤖 AI phát hiện 3 citations dùng APA, 2 dùng IEEE. Muốn tôi tự động convert tất cả sang BibTeX/IEEE?"
> 
> "Dễ undo — bạn có thể revert bất kỳ lúc nào"

User có thể nhấn "✓ Auto-format" hoặc "Không, tôi tự làm".

**Design rationale — S5 (Act):**

Đây là một trong số ít chỗ trong prototype mà AI được **Act** (tự làm mà không cần xin phép trước), vì:

- **Rủi ro thấp:** format citations không ảnh hưởng đến content
- **Dễ undo:** có thể revert ngay lập tức
- **User nhìn thấy ngay:** output rõ ràng, dễ kiểm tra
- **Tiết kiệm thời gian:** format thủ công hàng chục citations rất tốn công

Nguyên tắc: "Hành động có rủi ro thấp, người dùng nhìn thấy ngay và có thể hoàn tác, vì vậy AI được tự thực hiện."

---

## 5. Mức độ tự chủ: Act / Ask / Don't Act

| Kịch bản | Level | Lý do |
|---|---|---|
| P0: Chưa có topic | `Don't Act` | AI không có đủ thông tin để làm bất kỳ điều gì |
| S1: Fetch papers | `Ask` | Chi phí restart cao, sub-query sai = sai từ gốc |
| S2: Cluster papers | `Ask` | User cần approve structure trước khi AI generate claims |
| S3: Included claims | `Act` | Rủi ro thấp, claim đã có 2+ sources verify |
| S3: Removed claims | `Ask` | Hậu quả lớn nếu xóa nhầm claim quan trọng |
| S4: Flag suspect citation | `Act` | AI phát hiện và đánh dấu, nhưng không tự xóa |
| S4: Thay thế citation | `Ask` | User phải chọn replacement, không được tự quyết |
| S5: Auto-format citations | `Act` | Rủi ro thấp, dễ undo, user thấy ngay |
| Publish/submit bài | `Don't Act` | Nằm ngoài thẩm quyền của AI, rủi ro cực cao |

Prototype có đủ cả 3 levels, phân bổ theo logic rõ ràng.

---

## 6. Feedback hai chiều: Ma trận 2×2

### User → System (Explicit)

Người dùng **chủ động** đưa tín hiệu cho hệ thống:

- **S1:** User edit sub-query #3 "quá rộng" → submit revised query → AI biết angle research cụ thể hơn
- **S3:** User click "Rescue" + cung cấp DOI cho removed claim → AI biết đang miss loại source nào
- **S4:** User click "Thay bằng paper thật" → AI biết cách tìm evidence đúng cho claim loại này
- **Output:** Nút 👍/👎 trên từng section (explicit rating)

*Rationale: Tường minh vì user cần deliberate action — AI không tự đoán ý định từ behavior.*

### User → System (Implicit)

Hệ thống **quan sát hành vi** của user mà không ngắt:

- User **approve outline ngay không sửa** → signal: AI hiểu đúng topic, không cần hỏi thêm ở lần sau
- User **spend >3 min ở Interrupt 1** → signal: uncertain về sub-queries, có thể cần thêm gợi ý
- User **rescue nhiều claims** từ "Removed" → signal: AI search sources quá hẹp, cần mở rộng database
- User **undo auto-format** → signal: AI dùng sai citation style cho lĩnh vực này

*Rationale: Không làm gián đoạn flow của user. Observe hành vi nhưng không tự coi là ground truth — user undo có thể vì AI sai, hoặc đơn giản là họ đổi ý.*

### System → User (Explicit)

Hệ thống **nói thẳng** cho user biết trạng thái:

- Progress steps: "Bước 2/4: Clustering..." với step indicator rõ ràng
- Badge "⚠️ 3 citations suspect" trên output panel
- Warning box: "Citation not found in verified sources. Click to investigate."
- Confidence bar cho từng claim: 25% / 60% / 95%
- Confirmation sau fix: "✓ Đã ghi nhớ fix này cho section tiếp theo"

*Rationale: Explicit vì user cần biết trạng thái AI để ra quyết định có informed, không phải đoán.*

### System → User (Implicit)

Hệ thống truyền đạt thông tin **qua thiết kế giao diện**:

- **Color coding:** xanh/vàng/đỏ cho citations theo confidence → user học pattern mà không cần đọc hướng dẫn
- **Dim** removed claims thay vì xóa hẳn → signal "vẫn có thể rescue, chưa mất hoàn toàn"
- **"▸ arrow"** trên cluster reason → signal "có thể expand để xem thêm"
- **Step indicator mờ** cho pending steps → user biết còn bước nữa phía trước

*Rationale: Progressive disclosure — dùng visual cue trước, chỉ interrupt khi user thực sự cần help.*

### Nguyên tắc feedback trong prototype này

1. **Đặt feedback đúng ngữ cảnh** — Rescue button nằm ngay trên claim bị removed, không ở menu khác
2. **Show benefit của việc feedback** — "AI sẽ ghi nhớ fix này cho section tiếp theo" → user có lý do để feedback
3. **Đừng ngắt khi không cần thiết** — Implicit signals được observe im lặng, không popup liên tục
4. **Hành vi ≠ ý định** — User undo ≠ AI sai. Hệ thống ghi nhận nhưng không tự label

---

## 7. Bằng chứng và Explainability

Prototype có **ít nhất 2 lớp explainability**:

**Lớp 1 — Interrupt 2: Cluster reasoning**

Khi user click "▸ Xem lý do AI cluster", hệ thống show:
- Keywords chung giữa các papers trong cluster
- % overlap với cluster khác
- Sub-clusters tiềm ẩn bên trong (nếu có)
- Recommendation của AI về việc có nên split không

Điều này giúp user đưa ra quyết định split/merge dựa trên **data**, không dựa vào cảm giác.

**Lớp 2 — S4: Citation hallucination evidence**

Khi user click vào citation suspect, hệ thống show:
- Claim AI đang cố prove
- Paper nào AI "dùng" làm basis cho claim đó
- Tại sao citation cụ thể không tìm thấy được
- Gợi ý paper thật có thể thay thế

Điều này giúp user **không chỉ biết citation sai, mà hiểu TẠI SAO nó sai** — từ đó đưa ra quyết định có informed hơn (xóa, thay, hay tự tìm).

---

## 8. Điều kiện tối thiểu — Checklist

| Yêu cầu | Trạng thái | Thể hiện ở đâu |
|---|---|---|
| ✅ Onboarding lần đầu dùng tính năng | Đạt | Tab 01 — P0 |
| ✅ Ít nhất 4 kịch bản ngoài onboarding | Đạt | S1, S2, S3, S4, S5 (5 kịch bản) |
| ✅ Lát cắt xuyên suốt Onboarding → During → After → Feedback | Đạt | Flow 4 request pipeline |
| ✅ Đủ Act / Ask / Don't Act | Đạt | Bảng agency section 5 |
| ✅ Ít nhất 1 vòng feedback & recovery hoàn chỉnh | Đạt | S3 rescue loop, S4 hallucination fix |
| ✅ Đủ 4 loại feedback trong ma trận 2×2 | Đạt | Tab 06 — Feedback 2×2 |
| ✅ Ít nhất 1 lớp bằng chứng / explainability | Đạt | Cluster reasoning + Citation evidence |
| ✅ Design rationale đặt cạnh flow | Đạt | Annotation box dưới mỗi screen |

---

## 9. Demo Path (5 phút)

| Thời gian | Nội dung | Click path |
|---|---|---|
| 0:00–0:30 | Người dùng, vấn đề, lát cắt | Tab 00 — flow map tổng quan |
| 0:30–1:15 | Onboarding → Plan review | Tab 01 → Tab 02: click "✏️ Sửa" query #3, edit, save, approve |
| 1:15–2:15 | Outline + Explainability | Tab 03: click "▸ Xem lý do cluster 3" → yellow warning → click "✂️ Split" → approve |
| 2:15–3:45 | Failure & Recovery (×2) | Tab 04: rescue claim "O(n√n)" → Tab 05: click [Lin et al., 2021] → replace với Child et al. |
| 3:45–4:45 | Feedback 2×2 | Tab 06: walk through 4 ô, highlight implicit system: color coding = progressive disclosure |
| 4:45–5:00 | Quyết định thiết kế quan trọng nhất | "Interrupt 3 không được là informational-only vì AI có thể xóa sai claim quan trọng" |

---

## 10. Quyết định thiết kế quan trọng nhất và rủi ro còn lại

### Quyết định quan trọng nhất

**Đổi Interrupt 3 từ "informational only" sang "có rescue loop"**

Thiết kế gốc của team là Interrupt 3 chỉ hiện thống kê (38/12/7) như một dashboard — user xem rồi nhấn Continue. Nhưng điều này tạo ra vấn đề: nếu AI xóa sai một claim quan trọng, user không có cách nào lấy lại.

Chi phí của lỗi này cực cao trong ngữ cảnh nghiên cứu — một claim bị thiếu có thể làm yếu toàn bộ argument của paper.

Quyết định: **Bất kỳ claim nào trong "Removed" đều phải cho user khả năng rescue**, với evidence loop hoàn chỉnh (user provide source → AI verify → move sang needs-review → confirm). Điều này tăng 1 interaction step nhưng giảm đáng kể risk của false negative.

### Rủi ro còn lại

1. **Citation confidence score không phải ground truth** — Confidence 25% không đồng nghĩa với "sai 75% khả năng". User có thể overtrust hoặc undertrust màu đỏ
2. **Implicit user feedback dễ misinterpret** — User approve outline nhanh có thể vì họ hiểu rõ, hoặc đơn giản là vội. Hệ thống không phân biệt được hai trường hợp
3. **Pipeline bị interrupted ở giữa** — Nếu user rời session sau Interrupt 2 và quay lại sau, thread_id có thể expire. Chưa có thiết kế cho trường hợp resume từ mid-pipeline

---

