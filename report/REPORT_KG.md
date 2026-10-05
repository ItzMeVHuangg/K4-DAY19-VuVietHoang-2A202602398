# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Vũ Việt Hoàng  **MSSV:** 2A202602398  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu khớp `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

**Cấu hình chạy:** chat `gemini:gemini-3.5-flash-lite`, embedding `gemini:gemini-embedding-001`, top_k=3, chunk_size=800, 176 chunk, KG 201 node / 384 cạnh. Ontology gợi ý.

> **Hai lưu ý về số liệu.** (1) `src/llm.py` chưa có bảng giá cho `gemini-3.5-flash-lite` nên cột USD luôn là 0,00000; các so sánh chi phí dưới đây dùng **token và giây**. (2) Model mặc định của lab `gemini-2.5-flash-lite` đã bị Google gỡ, nên chạy với `GEMINI_CHAT_MODEL=gemini-3.5-flash-lite`. Tài khoản free tier giới hạn 15 request/phút nên mình đặt `max_retries=12` trong `_openai_client` (`src/llm.py`); thời gian chờ retry nằm trong cột `seconds`, nhất là ở indexing.

## 1. Chi phí (10 điểm)

```
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 201 nodes / 384 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    103.1
graph       196     34619     5598   0.00000    143.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       73   0.00000     2.09
graph       1.00   2.00     5692      181   0.00000     2.22
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0 (chưa có giá) | 0 (chưa có giá) | không tính được |
| Indexing giây | 103,1 | 143,8 | ×1,39 |
| Indexing lần gọi API | 176 | 196 | ×1,11 |
| Mỗi câu: USD | 0 (chưa có giá) | 0 (chưa có giá) | không tính được |
| Mỗi câu: giây | 2,09 | 2,22 | ×1,06 |
| Mỗi câu: in_tok | 696 | 5692 | ×8,18 |
| Mỗi câu: out_tok | 73 | 181 | ×2,48 |

**Chi phí tăng thêm đến từ đâu?**
> Lúc dựng: Flat chỉ embed 176 chunk, Graph embed thêm 20 lần gọi LLM trích xuất (34.619 token vào, 5.598 token ra) cho 20 bài báo, còn phần luật dựng bằng regex nên không tốn LLM. Lúc hỏi: input tăng ×8,2 vì ngoài 3 chunk còn có khoảng 20–44 dòng dữ kiện graph (câu Q6 có 44 dòng), và output dài hơn ×2,5 vì câu trả lời có cấu trúc, liệt kê nhiều ý. Độ trễ gần như không đổi (×1,06) vì thời gian bị chi phối bởi lần gọi LLM, còn truy vấn Cypher chỉ tốn vài ms.
>
> Ước tính điểm hòa vốn: nếu tính theo giá `gpt-4o-mini` (0,15 / 0,60 USD mỗi triệu token, chỉ là giả định để so sánh, không phải số đã đo), dựng graph tốn thêm ≈ 0,0086 USD và mỗi câu hỏi tốn thêm ≈ 0,0008 USD. Graph **không rẻ hơn ở điểm nào**, nên không có "điểm hòa vốn" về tiền; cái được là độ đúng (recall 0,51 → 1,00) trên câu cần xuyên 2 KB.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1,00 / 2 | 1,00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một chunk, vector search đủ |
| Q2 | single-hop-news | 1,00 / 2 | 1,00 / 2 | Hòa | Tên hai bị cáo tử hình nằm trong một bài báo |
| Q3 | cross-kb | 0,33 / 1 | 1,00 / 2 | Graph | Flat không có Điều 251 và khung 02–07 năm ("Ngữ cảnh không đủ thông tin"); graph đi Person → Case → Crime → Article → Clause |
| Q4 | cross-kb | 0,33 / 1 | 1,00 / 2 | Graph | Flat chỉ biết hành vi, không biết khung tối đa; graph đưa khoản tù nặng nhất của Điều 255 (20 năm / chung thân) |
| Q5 | cross-kb-multi-hop | 0,40 / 1 | 1,00 / 2 | Graph | Graph nối vụ → khối lượng MDMA → khoản 4 Điều 250 → tử hình; flat "không đủ thông tin" ở khoản và khung |
| Q6 | aggregation | 0,00 / 2 | 1,00 / 2 | Graph (recall); judge hòa | Graph liệt kê mọi `Case` có `INVOLVES` MDMA; flat chỉ thấy 3 chunk. Điểm judge của flat cao bất thường, xem lỗi E4 |

**Quy luật:** câu nằm gọn trong một đoạn (Q1, Q2) thì Flat RAG đủ và rẻ hơn ×8 token. Câu cần ghép dữ liệu từ hai KB (Q3–Q5) hoặc gom nhiều tài liệu (Q6) thì graph thắng, vì thông tin cần thiết không cùng một chunk nên vector top-3 không thể lấy đủ.

## 3. Phân tích lỗi (20 điểm)

Dữ liệu dưới đây lấy từ graph đầy đủ ngay sau `python bench_kg.py --judge` (201 node / 384 cạnh).

### Lỗi E3: Trùng thực thể (một vụ án thành 2 node `Case`)

- **Hiện tượng:** vụ vận chuyển MDMA qua Nội Bài của Cái Quang Huy bị tách thành 2 node `Case` vì hai bài báo khác nhau cùng viết về vụ đó và LLM đặt tên vụ khác nhau; người chấm sẽ thấy `Person` Cái Quang Huy có 2 đường tới luật.
- **Bằng chứng:**

```cypher
MATCH (k:Case) WHERE k.name CONTAINS 'Nội Bài' RETURN k.name, k.doc_id;
MATCH p=(:Person {name:'Cái Quang Huy'})-[:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(:Article) RETURN count(p) AS paths;
```

```
Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài              news-100260917203001265
Vụ vận chuyển trái phép chất ma túy qua sân bay Nội Bài liên quan đến Cái Quang Huy  news-100260918080821054
paths = 2
```

  Ảnh hưởng thấy được ở Q6 (câu trả lời GraphRAG gộp 2 node này thành một mục "Vụ vận chuyển hơn 10kg … (hoặc …)"). Truy vấn `MATCH (k:Case)-[r:INVOLVES]->(s:Substance) WHERE toLower(s.name) CONTAINS 'mdma'` trả về **6** `Case`, trong khi đáp án chuẩn có **3** vụ.
- **Nguyên nhân:** thiết kế ontology. `Case` khóa theo `name` do LLM tự đặt cho từng bài (`MERGE (k:Case {name: $name})`), nên cùng một vụ ở hai bài sẽ ra hai tên và hai node; không có bước gộp theo thực thể ngoài đời (người + tội + ngày).
- **Đề xuất sửa:** (1) cho `Case` một khóa ổn định, ví dụ băm của tập tên bị cáo + tội danh + ngày, thay vì tên tự do; (2) hoặc thêm bước hậu xử lý gộp các `Case` có chung `Person` và chung `Crime`; (3) hoặc đưa danh sách `Case` đã có vào prompt để LLM tái sử dụng tên. Đánh đổi: (1)–(2) không tốn token LLM nhưng có thể gộp nhầm hai vụ khác nhau của cùng một người; (3) tốn thêm token mỗi bài và làm trích xuất tuần tự (không chạy song song được).

### Lỗi E5: Câu trả lời của LLM lệch với graph (Q6 thừa vụ, thừa Điều luật)

- **Hiện tượng:** Q6 hỏi "những vụ nào liên quan MDMA". Đáp án chuẩn có 3 vụ (Cái Quang Huy, Lê Minh Thành, Viện Pháp y tâm thần). GraphRAG liệt kê **5** mục, trong đó có "Vụ bắt giữ giang hồ Hoàng Nato…" và nêu "các điều luật liên quan đến MDMA là Điều 250, 249, 251, 252 BLHS" cho vụ Nội Bài, dù vụ này chỉ bị truy tố tội vận chuyển (Điều 250).
- **Bằng chứng:** trích câu trả lời Q6 của pipeline `graph`: *"Các điều luật liên quan đến MDMA trong dữ kiện gồm Điều 250, Điều 249, Điều 251, Điều 252 BLHS (các khoản 1, 2, 3, 4 tùy theo hành vi và khối lượng)."* Gọi trực tiếp `context()` cho câu hỏi Q6:

```
len(facts) = 44
(Case ...) -[INVOLVES]-> (Substance: MDMA)      × 6 vụ (gồm cả vụ Hoàng Nato, amount rỗng)
(Clause: Điều 249/250/251/252 BLHS khoản N) -[MENTIONS]-> (Substance: MDMA)   × 18 dòng
[Điều 249/250/251/255 …] khoản …                 × 14 dòng khoản luật
```

  Tự viết Cypher trả lời thẳng câu hỏi (`MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'}) RETURN k.name`) cho ra đúng 6 `Case` (5 vụ thật, vì 2 trong số đó là cùng vụ Nội Bài, xem E3); graph chứa thật cả vụ Hoàng Nato nối tới MDMA nên phần "thừa" một phần là do trích xuất (`amount` rỗng, bài chỉ nhắc MDMA chung chung).
- **Nguyên nhân:** hai nguồn. (a) Trích xuất: LLM gán `INVOLVES MDMA` cho vụ Hoàng Nato dù bài không cho số lượng cụ thể. (b) KG-3 / `seed_facts`: câu hỏi chứa chữ "MDMA" nên node `Substance: MDMA` thành seed, kéo theo toàn bộ cạnh `MENTIONS` của mọi khoản luật nhắc MDMA, nên prompt có Điều 249/251/252 không liên quan tới vụ nào cụ thể và LLM gộp chúng vào câu trả lời.
- **Đề xuất sửa:** (1) trong `context()`, với seed là `Substance`, chỉ hiển thị cạnh `Case → Substance`, không mở rộng sang `Clause` (dùng tham số `skip_labels=("Clause",)` của `seed_facts`); (2) trong prompt trích xuất, yêu cầu chỉ ghi `INVOLVES` khi bài nêu rõ chất đó trong vụ. Đánh đổi: (1) giảm token (~44 → ~25 dòng) nhưng câu hỏi dạng "khoản nào quy định MDMA" sẽ mất dữ kiện, phải dựa vào nhánh "Điều N" của `context()`; (2) có thể bỏ sót vụ có nhắc chất ngắn gọn (recall thấp hơn).

### Lỗi E4: Phép đo sai (Q6 Flat: recall 0,00 nhưng judge 2)

- **Hiện tượng:** câu trả lời Flat cho Q6 bị `recall = 0,00` nhưng `judge = 2` (đúng đủ), hai thước đo mâu thuẫn hoàn toàn. Cả hai đều sai theo hướng khác nhau.
- **Bằng chứng:** `must_include` của Q6 là `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Câu trả lời Flat (từ `ket_qua_benchmark_kg.txt`, Q6 flat): *"1. … liên quan đến các nhân vật như Đạt và Huy. 2. … bắt quả tang Thành mang 5 viên nén … 3. … kiện hàng do Huy gửi là MDMA (tổng khối lượng hơn 5,3kg)."*
- **Nguyên nhân:** phép đo. `recall` khớp chuỗi nguyên văn nên không nhận "Huy" / "Thành" là "Cái Quang Huy" / "Lê Minh Thành" (đúng một phần, bị phạt oan), và chắc chắn không có "Pháp y tâm thần". Ngược lại `judge` cho 2 điểm (đúng đủ) dù Flat **thiếu vụ Viện Pháp y tâm thần** và mục 1 và 3 thực chất là cùng một người (Huy), nên đáng ra phải là 0–1. Bên nào đúng: câu trả lời đúng một phần, nên cả 0,00 lẫn 2 đều lệch.
- **Đề xuất sửa:** (1) `recall`: khớp theo họ tên gốc không phân biệt từ phụ (hoặc chấp nhận alias trong `must_include`); (2) `judge`: dùng thang chấm có kiểm đếm số vụ so với đáp án chuẩn (ví dụ yêu cầu liệt kê từng mục đáp án và đánh dấu có/không), hoặc chạy judge 2 lần và lấy giá trị thấp hơn. Đánh đổi: thêm 1 lần gọi LLM mỗi câu (+6 lần gọi cho cả benchmark).

### Ghi nhận thêm (E1, E6)

- **E1 (cầu nối gãy):** `MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name, k.doc_id` trả về 2/13 `Case`: "Vụ vận chuyển hơn 800kg chất nghi ma túy tại Preah Sihanouk" (vụ bắt giữ ở Campuchia, bài không nêu tội danh theo luật Việt Nam, đúng khi không nối) và "Vụ triệt phá chuyên án A3-626P" (bài tin về hội nghị của Bộ đội Biên phòng, không phải vụ án; đây là lỗi trích xuất: lẽ ra phải trả `{"cases": []}`).
- **E6 (thuộc tính thiếu):** `INVOLVED_IN` có 51 cạnh, `charge` rỗng ở 19 cạnh, `sentence` rỗng ở 41 cạnh. Nhiều cạnh `role = 'cán bộ'` rỗng `charge` là hợp lý (người liên quan, không bị truy tố ma túy), nhưng `sentence` rỗng của Cái Quang Huy (bài chỉ nói bị truy tố, chưa xét xử) và các bị cáo chưa xét xử là đúng sự thật chứ không phải lỗi.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> **Dùng Flat RAG** khi đáp án nằm gọn trong một đoạn: Q1, Q2 cả hai đều đạt recall 1,00 / judge 2, mà Flat chỉ tốn ~696 token vào mỗi câu so với ~5.692 của Graph (×8,2), và indexing nhanh hơn (103 s so với 144 s, không cần 20 lần gọi LLM trích xuất). **Dùng KG** khi câu hỏi phải ghép dữ liệu giữa các nguồn khác loại (tin ↔ luật) hoặc gom nhiều tài liệu: ở Q3–Q5 recall của Flat chỉ 0,33–0,40 và judge 1, còn Graph đạt 1,00 / 2; ở Q6 (tổng hợp) recall tăng 0,00 → 1,00. Trung bình trên 6 câu: recall 0,51 → 1,00 và judge 1,50 → 2,00, nhưng đổi lại input token ×8, và graph có lỗi riêng (trùng `Case`, nhiễu từ `MENTIONS`, trích xuất thừa). Điều kiện đáng đầu tư: tập dữ liệu có **khóa chung** để làm node cầu nối (ở đây là tên tội danh) và đủ nhiều câu hỏi xuyên nguồn để bù chi phí dựng; với dữ liệu chỉ có câu hỏi một nguồn thì Flat RAG đủ. Lưu ý: số liệu USD chưa đo được (xem phần đầu báo cáo), nên so sánh chi phí ở đây là theo token và giây.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.05s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy**.

## Vấn đề gặp phải (không tính điểm)

- `bench_kg.py --check` lần đầu lỗi `404 … gemini-2.5-flash-lite is no longer available to new users`. Cách xử lý: đặt biến môi trường `GEMINI_CHAT_MODEL=gemini-3.5-flash-lite` khi chạy (không sửa `.env`).
- `bench_kg.py --judge` lần đầu lỗi `429 … generate_content_free_tier_requests, limit: 15` (free tier Gemini 15 request/phút). Cách xử lý: tăng `max_retries` lên 12 trong `_openai_client` (`src/llm.py`); SDK tự chờ theo `Retry-After`. Thời gian chờ nằm trong cột `seconds`.
- `llm.py` không có giá `gemini-3.5-flash-lite` và `gemini-embedding-001`, nên mọi cột USD bằng 0.
