# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Thị Hải Mi  **MSSV:** 2A202602667  **Ngày:** 2026-10-05

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
Chat model: gemini:gemini-3.5-flash-lite | Embedding: gemini:gemini-embedding-001 | top_k=3 | chunk_size=800 | chunks=176 | KG: 205 nodes / 387 rels

== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    132.5
graph       196     34619     5741   0.00000    229.4

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       73   0.00000     5.04
graph       1.00   2.00     6149      164   0.00000     5.49
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00000 | không tính được (xem ghi chú 1) |
| Indexing giây | 132.5 | 229.4 | ×1.73 |
| Indexing: số lần gọi API | 176 | 196 | ×1.11 (+20 lần gọi LLM) |
| Mỗi câu: USD | 0.00000 | 0.00000 | không tính được (xem ghi chú 1) |
| Mỗi câu: giây | 5.04 | 5.49 | ×1.09 |
| Mỗi câu: in_tok | 696 | 6149 | ×8.83 |
| Mỗi câu: out_tok | 73 | 164 | ×2.25 |

**Ghi chú đọc số liệu:**
1. **USD = 0 không phải chi phí thật.** Bảng giá `PRICES_PER_M` trong `src/llm.py` chưa có `gemini-3.5-flash-lite` / `gemini-embedding-001`, nên `price()` trả 0. Vì vậy dùng **token** làm thước đo chi phí: chi phí chat tỉ lệ với `in_tok`/`out_tok`. Embedding của Gemini không trả `usage` nên `in_tok` của indexing flat hiển thị 0.
2. **Cột giây bị đội lên do giãn nhịp gọi API.** Để không vượt hạn mức gói miễn phí của Gemini (100 request embedding/phút, 15 request chat/phút), mỗi lần gọi embedding được chờ tối thiểu 0,75 s và mỗi lần gọi chat tối thiểu 5 s. Vì thế: indexing flat ≈ 176 × 0,75 s ≈ 132 s; phần graph thêm ≈ 20 × 5 s ≈ 97 s; mỗi câu hỏi ≈ 5 s ở cả hai pipeline. **So sánh độ trễ giữa hai pipeline không có ý nghĩa** trong lần chạy này; so sánh token và số lần gọi thì vẫn đúng.

**Chi phí tăng thêm đến từ đâu?**
> Indexing: GraphRAG dùng lại nguyên index vector của Flat và **thêm 20 lần gọi LLM** (một lần/bài báo) để trích xuất vụ việc thành JSON — 34.619 token vào, 5.741 token ra; phần luật dựng bằng regex nên không tốn gì. Querying: số lần gọi API như nhau, nhưng prompt của GraphRAG dài **gấp ~8,8 lần** (6.149 so với 696 token) vì ngoài 3 đoạn văn bản còn chứa tóm tắt các vụ việc, toàn văn các khoản luật được giữ lại và các cạnh quanh node hạt giống; câu trả lời cũng dài hơn (~2,2 lần) vì nêu thêm Điều/khoản/khung hình phạt.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa "tiền chất" nằm trọn trong 1 đoạn của Điều 2 Luật PCMT nên vector search lấy được ngay. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên hai bị cáo tử hình nằm ngay câu đầu bài báo vụ 36kg; không cần đi sang KB luật. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat có án 36 tháng nhưng tự nhận "không đủ thông tin" về Điều luật; Graph đi `Person→Case→Crime←Điều 251→khoản 1` ra "02 năm đến 07 năm". |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat biết hành vi nhưng không có văn bản Điều 255; Graph khớp alias "Hoàng Nato", đi qua cầu `Crime` và dòng khung hình phạt ra "khoản 4 … tù chung thân". |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Flat không có khoản luật để so khối lượng; Graph đưa đủ khoản 1–4 Điều 250 (các khoản nhắc MDMA) nên LLM so được 9,6kg ≥ 100g → khoản 4, tử hình. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | Graph (có lỗi) | Top-3 của Flat chỉ chạm 2 vụ và thiếu Viện Pháp y; Graph lấy mọi `Case` nối MDMA nên đủ 3 vụ, nhưng thừa vụ Hoàng Nato do trích xuất sai chất (lỗi E5) và judge=2 của Flat là chấm sai (lỗi E4). |

## 3. Phân tích lỗi (20 điểm)

Graph phân tích: lần chạy `python bench_kg.py --judge` trong `ket_qua_benchmark_kg.txt` (205 node / 387 cạnh).

### Lỗi E3: Trùng thực thể — một vụ án thành nhiều node `Case`, kể cả vụ "ma" sinh từ nhiễu cuối bài

- **Hiện tượng:** Vụ "Hoàng Nato" (4 bài báo) thành **4 node `Case`** khác tên; vụ Viện Pháp y tâm thần (2 bài) thành **2 node**. Nghiêm trọng hơn, bài về Lê Minh Thành (`news-100260918080821054`) sinh thêm một vụ **Cái Quang Huy** không hề được bài này tường thuật. Ngược lại, `Person` gộp tốt: `Dương Minh Tuấn` (alias "Hoàng Nato") là 1 node nối tới cả 4 vụ.
- **Bằng chứng:**

```cypher
MATCH (k:Case) RETURN k.doc_id, k.name ORDER BY k.doc_id
```

```
news-100260917203001265  Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài
news-100260918080821054  Vụ mua bán ma túy liên quan đến Lê Minh Thành và đồng phạm tại Hà Nội
news-100260918080821054  Vụ vận chuyển ma túy qua sân bay Nội Bài do Cái Quang Huy thực hiện   ← vụ "ma"
news-100260920221957595  Vụ bắt giang hồ Hoàng Nato và các đường dây ma túy tại TP.HCM
news-100260922111804786  Vụ triệt phá 8 đường dây ma túy liên quan TikToker Phannhibeauty và Hoàng Nato tại TP.HCM
news-100260924095400982  Vụ sử dụng pod chill chứa ma túy của 'Hoàng Nato' và TikToker Phannhibeauty
news-100260925144412498  Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM
news-100260924105118645  Vụ sai phạm tại Viện Pháp y tâm thần Trung ương và tiệc ma túy tại Sầm Sơn
news-100260930085028036  Vụ tổ chức sử dụng trái phép chất ma túy tại Sầm Sơn và Viện Pháp y tâm thần Trung ương
```

  Câu trả lời Q6 (graph) lộ ra lỗi này: *"Vụ vận chuyển ma túy qua sân bay Nội Bài do Cái Quang Huy thực hiện (hoặc 'Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam qua sân bay Nội Bài')"* và liệt kê vụ Viện Pháp y dưới 2 tên.
- **Nguyên nhân:** (1) **Thiết kế ontology:** `Case` được `MERGE` theo `name` do LLM tự đặt cho từng bài, nên mỗi bài đặt một tên khác → không bao giờ gộp. (2) **Crawl:** file `news-100260918080821054.md` bị dính đoạn sapo của bài khác ở cuối ("Từ mối quen biết khi cùng làm bếp tại một nhà hàng ở Berlin… Cái Quang Huy…"), prompt trích xuất đọc cả đoạn đó nên tạo vụ thứ hai.
- **Đề xuất sửa:** (a) Crawler cắt phần "bài liên quan" sau thân bài (hoặc `extract_news_cases` chỉ đọc đến đoạn cuối của thân bài). (b) Khóa `Case` ổn định hơn: sau khi trích xuất, gộp các `Case` có chung ≥ 1 bị cáo/bị can **và** chung tội danh (Cypher `MATCH (a:Case)<-[:INVOLVED_IN]-(p)-[:INVOLVED_IN]->(b:Case) WHERE a<>b …`), giữ danh sách `doc_ids`. Đánh đổi: có thể gộp nhầm hai vụ khác nhau của cùng một người (ví dụ người có nhiều tiền án), nên cần thêm điều kiện thời gian/địa điểm.

### Lỗi E5: LLM lệch với dữ kiện thật — Q6 graph liệt kê thừa vụ "Hoàng Nato" có MDMA

- **Hiện tượng:** Q6 hỏi các vụ liên quan MDMA; đáp án chuẩn có 3 vụ (Cái Quang Huy, Lê Minh Thành, Viện Pháp y). GraphRAG trả lời thêm *"Vụ bắt giang hồ Hoàng Nato… có liên quan đến khoảng 100g ma túy tổng hợp là MDMA"*. Bài báo gốc chỉ ghi "khoảng 100g ma túy tổng hợp các loại" (và "thuốc lắc" trong danh sách loại ma túy của các đường dây), không khẳng định 100g là MDMA. `recall` vẫn = 1,00 vì phép đo không phạt thông tin thừa.
- **Bằng chứng:** cùng một tang vật "khoảng 100g ma túy tổng hợp" được trích thành **hai chất khác nhau** ở hai bài:

```cypher
MATCH (k:Case)-[r:INVOLVES]->(s) WHERE k.name CONTAINS 'Nato' RETURN k.name, s.name, r.amount
```

```
Vụ bắt giang hồ Hoàng Nato và các đường dây ma túy tại TP.HCM                     MDMA             khoảng 100g ma túy tổng hợp
Vụ bắt giang hồ Hoàng Nato và các đường dây ma túy tại TP.HCM                     Ketamine
Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM                 etomidate        hơn 1.000 đầu pod chill
Vụ triệt phá 8 đường dây ma túy liên quan TikToker Phannhibeauty và Hoàng Nato…   Methamphetamine  khoảng 100g ma túy tổng hợp
```

  Cypher trả lời thẳng Q6 (`MATCH (k:Case)-[:INVOLVES]->(:Substance {name:'MDMA'}) RETURN k.name`) ra 6 node `Case`, tương ứng 4 vụ ngoài đời — tức chính graph đã sai trước khi tới LLM trả lời.
- **Nguyên nhân:** **Prompt trích xuất:** `NEWS_EXTRACTION_PROMPT` yêu cầu "dùng tên chuẩn trong DANH SÁCH CHẤT nếu khớp", nên LLM bị đẩy về việc *chọn một* tên chuẩn cho cụm chung chung "ma túy tổng hợp" (lúc MDMA, lúc Methamphetamine) thay vì để trống. Graph lưu suy đoán này như sự thật; bước trả lời tin graph.
- **Đề xuất sửa:** Thêm vào prompt quy tắc "chỉ ghi chất khi bài nêu đích danh; cụm chung chung ('ma túy tổng hợp', 'các loại') thì bỏ qua", và lưu nguyên văn câu chứa chất vào `INVOLVES.evidence` để kiểm chứng được. Đánh đổi: prompt dài hơn một chút; có thể bỏ sót chất ở bài viết mơ hồ (giảm recall của Q6 ở trường hợp biên).

### Lỗi E4: Phép đo sai — Q6 flat `recall = 0.00` nhưng `judge = 2`

- **Hiện tượng:** Hai phép đo mâu thuẫn hoàn toàn trên cùng một câu trả lời.
- **Bằng chứng:** `must_include` của Q6 là `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Câu trả lời flat:

```
1. Vụ việc thứ nhất ([1]): … MDMA (khối lượng gần 4,3kg) liên quan đến Đạt và Huy.
2. Vụ việc thứ hai ([2]): Công an bắt quả tang Thành mang 5 viên nén màu trắng đi bán … MDMA.
3. Vụ việc thứ ba ([3]): … viên nén hình tam giác màu hồng - xám trong kiện hàng gửi bởi Huy là MDMA (khối lượng hơn 5,3kg).
```

- **Nguyên nhân:** **Phép đo.** `recall` so khớp chuỗi nguyên văn nên "Huy", "Thành" không được tính → 0,00 dù câu trả lời có nhắc đúng 2 vụ. Ngược lại `judge` cho điểm tối đa dù câu trả lời **thiếu hẳn vụ Viện Pháp y** và đếm **hai kiện hàng của cùng vụ Cái Quang Huy thành hai vụ**. Đánh giá đúng phải khoảng 1/2 (đúng một phần). Cả hai phép đo đều sai, theo hai hướng ngược nhau.
- **Đề xuất sửa:** `keyword_recall` chấp nhận biến thể tên (danh sách alias trong `benchmark_kg.json`, ví dụ `["Cái Quang Huy", "Huy"]`); prompt judge yêu cầu liệt kê từng ý trong đáp án chuẩn và đánh dấu có/không trước khi cho điểm. Đánh đổi: alias quá ngắn ("Huy") dễ khớp nhầm; judge dài hơn tốn thêm token mỗi câu.

### Lỗi E2: Thiếu ngữ cảnh luật — quy tắc lọc khoản theo chất bỏ sót khung cao nhất của Điều 255 (đã sửa)

- **Hiện tượng:** Q4 hỏi mức phạt tù **tối đa** cho hành vi tổ chức sử dụng trái phép chất ma túy. Quy tắc lọc gợi ý ở Bước 5 (giữ khoản 1 + khoản `MENTIONS` chất của vụ) chỉ giữ được khoản 1 (02–07 năm) của Điều 255.
- **Bằng chứng:** không khoản nào của Điều 255 nhắc tới chất:

```cypher
MATCH (:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl) OPTIONAL MATCH (cl)-[:MENTIONS]->(s)
RETURN cl.number, cl.penalty, collect(s.name) ORDER BY cl.number
```

```
1  phạt tù từ 02 năm đến 07 năm        []
2  phạt tù từ 07 năm đến 15 năm        []
3  phạt tù từ 15 năm đến 20 năm        []
4  phạt tù 20 năm hoặc tù chung thân   []
```

- **Nguyên nhân:** **Cypher KG-3 / độ chi tiết ontology:** Điều 255 tăng nặng theo tình tiết (số người, tái phạm…), không theo chất, nên điều kiện `MENTIONS` không bao giờ đúng.
- **Đề xuất sửa (đã áp dụng trong `Neo4jGraph.context`):** thêm cho mỗi Điều đi tới một dòng ngắn liệt kê mọi khung hình phạt (`[Điều 255 BLHS] các khung hình phạt: khoản 1: …; khoản 4: phạt tù 20 năm hoặc tù chung thân`). Kết quả: Q4 graph trả lời đúng *"khung hình phạt cao nhất quy định tại khoản 4 là phạt tù 20 năm hoặc tù chung thân"* (recall 1,00, judge 2). Đánh đổi: thêm khoảng 1 dòng (~60 token) cho mỗi Điều trong prompt.

### Ghi chú thêm (E1, E6)

- **E1 — vụ không nối sang luật:** 2 vụ không có `CHARGED_WITH`: *Vụ tông cảnh sát giao thông tại An Giang* (bị can bị khởi tố tội chống người thi hành công vụ; việc *sử dụng* ma túy không phải tội trong Chương XX) và *Triệt phá chuyên án A3-626P* (bài hội nghị chỉ nhắc "thu giữ 40kg ma túy", không nêu tội danh). Cả hai đều **hợp lý** không nối; `link_entity` trả `None` thay vì đoán bừa.
- **E6 — thuộc tính rỗng:** 21/52 cạnh `INVOLVED_IN` không có `charge`, 42/52 không có `sentence`. Phần lớn hợp lý (cán bộ, người liên quan, vụ chưa xét xử). Lỗi trích xuất rõ nhất: bài `news-100260924105118645` viết *"Đông mang theo loa, bàn DJ và ma túy để tổ chức 'bay lắc'"*, nhưng cạnh `Lê Văn Đông -[INVOLVED_IN]-> Vụ sai phạm tại Viện Pháp y…` có `charge` rỗng (bài `news-100260930085028036` thì trích đúng tội tổ chức sử dụng cho Đông) — cùng một người, hai bài, hai kết quả trích xuất khác nhau.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> **Flat RAG là đủ** khi đáp án nằm trọn trong một đoạn văn và câu hỏi có từ khóa trùng với đoạn đó: Q1 (luật) và Q2 (tin) cả hai pipeline đều đạt recall 1.00 / judge 2, trong khi Flat chỉ dùng 696 token đầu vào mỗi câu so với 6.149 của Graph (×8,8). Trả thêm gần 9 lần token cho những câu này là lãng phí.
>
> **Nên dùng KG** khi câu hỏi phải **nối hai nguồn không có chữ nào chung** hoặc **tổng hợp nhiều tài liệu**. Trên 4 câu cross-kb/aggregation (Q3–Q6), recall trung bình của Flat chỉ 0,27 (0.33, 0.33, 0.40, 0.00), của Graph là 1,00; judge trung bình 1,25 so với 2,00. Lý do có cấu trúc: báo không bao giờ ghi số Điều luật, nên vector search không có cách nào kéo Điều 250/251/255 về cho câu hỏi chỉ nêu tên người; node cầu nối `Crime` làm được việc đó. Chi phí một lần cho khả năng này là thêm 20 lần gọi LLM lúc dựng graph (≈40K token).
>
> **Nhưng KG không tự đúng.** Graph chỉ chính xác bằng bước trích xuất: cùng tang vật "100g ma túy tổng hợp" thành MDMA ở bài này và Methamphetamine ở bài kia (E5), một vụ thành 4 node `Case` (E3). Các lỗi này không làm giảm `recall` nên phép đo hiện tại đánh giá Graph cao hơn thực tế ở Q6 (E4). Kết luận: dùng hybrid (vector + graph) như GraphRAGAgent cho câu hỏi đa bước/đa nguồn, kèm kiểm tra chất lượng trích xuất; dùng Flat cho tra cứu một bước.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.03s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 25 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Trần Minh Tâm** (vụ mua bán hơn 36kg ma túy tại TP.HCM). Đường đi trên ảnh: `Trần Minh Tâm -INVOLVED_IN-> Vụ mua bán hơn 36kg… -CHARGED_WITH-> {mua bán / tổ chức sử dụng trái phép chất ma túy} <-DEFINES- {Điều 251, Điều 255 BLHS}`, kèm `LOCATED_IN TP.HCM` và `INVOLVES "ma túy"` (node chất chung chung, cũng là một dạng lỗi trích xuất ở mục 3).

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Không còn lỗi chưa giải quyết. Các vấn đề đã gặp khi dùng Gemini (gói miễn phí) thay cho OpenAI:
>
> 1. **Model bị ngừng cấp:** `GEMINI_CHAT_MODEL=gemini-2.5-flash-lite` (mặc định của lab) trả `404 - This model models/gemini-2.5-flash-lite is no longer available to new users`. Đã đổi sang `gemini-3.5-flash-lite`.
> 2. **Model quá chậm:** `gemini-3.5-flash` mất ~40 s cho một câu "Reply with exactly: OK"; `bench_kg.py --build --limit 2` treo hơn 5 phút. Đã bỏ, dùng bản `-lite` (2 bài báo mất 5,8 s).
> 3. **Vượt hạn mức theo phút (429 RESOURCE_EXHAUSTED):** embedding `embed_content_free_tier_requests, limit: 100` (176 chunk gọi liên tiếp) và chat `generate_content_free_tier_requests, limit: 15, model: gemini-3.5-flash-lite`. Đã xử lý bằng một script chạy kèm (ngoài repo) bọc `bench_kg.py`: giãn nhịp ≥ 0,75 s/lần embedding và ≥ 5 s/lần chat, timeout 60 s/lần gọi, gặp 429 thì chờ 30 s rồi thử lại. Không sửa code của lab. Hệ quả: cột `seconds` trong benchmark chủ yếu là thời gian chờ (xem mục 1).
> 4. Các model khác trên tài khoản (Gemini 2.5 Flash, 3 Flash…) chỉ có 20 request/ngày, không đủ cho một lần `--judge` (44 lần gọi chat), nên giữ `gemini-3.5-flash-lite` (500 request/ngày).
