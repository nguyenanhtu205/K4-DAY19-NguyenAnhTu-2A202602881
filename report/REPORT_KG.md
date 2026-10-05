# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Nguyễn Anh Tú  **MSSV:** 2A202602881  **Ngày:** 05/10/2026

## 1. Chi phí

Kết quả từ `ket_qua_benchmark_kg.txt` (Gemini 3.5 Flash-Lite, Gemini Embedding 001; 176 chunks; KG 201 node/385 cạnh):

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    132.5
graph       196     34619     5579   0.02433    166.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       72   0.00039     1.91
graph       0.94   1.83     5479      165   0.00206     2.03
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | $0.00000* | $0.02433 | Không xác định* |
| Indexing giây | 132.5 | 166.3 | 1.25× |
| Mỗi câu: USD | $0.00039 | $0.00206 | 5.28× |
| Mỗi câu: giây | 1.91 | 2.03 | 1.06× |
| Mỗi câu: in_tok | 696 | 5,479 | 7.87× |

\*Meter của repo chưa có bảng giá cho `gemini-embedding-001`, nên USD của embedding được ghi là 0; vì vậy không dùng tỉ lệ indexing USD để kết luận chi phí thực tế. Chi phí Graph tăng chủ yếu do 20 lần trích xuất tin bằng LLM khi dựng graph và prompt truy vấn chứa thêm facts multi-hop.

## 2. Từng câu hỏi

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Đáp án đã nằm trong một Điều luật/chunk. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên hai bị cáo cùng nằm trong bài báo. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Graph nối Lê Minh Thành → tội danh → Điều 251/khoản 1. |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | Graph (một phần) | Graph tìm được Điều 255 nhưng lọc khoản thiếu khung tối đa. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Quan hệ `INVOLVES` với MDMA dẫn đúng đến khoản 4 Điều 250. |
| Q6 | aggregation | 0.00 / 1 | 1.00 / 2 | Graph | Graph gom các `Case` chung node `Substance: MDMA`. |

## 3. Phân tích lỗi

### Lỗi E2: Thiếu ngữ cảnh luật cho khung phạt tối đa

- **Hiện tượng:** Q4 GraphRAG nói Điều 255 có mức tối đa 07 năm, trong khi đáp án cần tù chung thân.
- **Bằng chứng:** Câu trả lời Q4 GraphRAG ghi: “khoản 1 ... từ 02 năm đến 07 năm (tối đa 07 năm tù)”. Trong graph, Điều 255 có khoản 4 là “phạt tù 20 năm hoặc tù chung thân”.

```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
RETURN cl.number AS khoan, cl.penalty AS khung_hinh_phat ORDER BY khoan;
```

```text
1 | phạt tù từ 02 năm đến 07 năm
2 | phạt tù từ 07 năm đến 15 năm
3 | phạt tù từ 15 năm đến 20 năm
4 | phạt tù 20 năm hoặc tù chung thân
```

- **Nguyên nhân:** KG-3 ưu tiên khoản 1 và khoản có chất ma túy của vụ. Vụ Hoàng Nato không có chất được trích xuất phù hợp để giữ các khoản khác, trong khi câu hỏi hỏi “tối đa”.
- **Đề xuất sửa:** nhận diện cụm “tối đa/khung cao nhất” trong câu hỏi để luôn thêm khoản có mức phạt cao nhất của Điều. Prompt dài hơn một ít nhưng tránh trả lời sai pháp lý.

### Lỗi E4: Recall theo từ khóa mâu thuẫn với đánh giá nội dung

- **Hiện tượng:** Q6 Flat RAG có `recall=0.00` nhưng judge=1. Câu trả lời vẫn liệt kê ba vụ có MDMA, trong đó có vụ Thành và các kiện hàng MDMA.
- **Bằng chứng:** `must_include` của Q6 cần nguyên văn “Cái Quang Huy”, “Lê Minh Thành”, “Pháp y tâm thần”; câu Flat dùng mô tả “Thành” và “các viên nén/kiện hàng” nên không khớp chuỗi bắt buộc dù có phần thông tin đúng.
- **Nguyên nhân:** `keyword_recall` chỉ kiểm tra chuỗi con, không nhận biết đồng tham chiếu, tên rút gọn hay đáp án diễn đạt tương đương.
- **Đề xuất sửa:** chuẩn hóa alias thực thể trước khi tính recall, hoặc kết hợp recall từ khóa với entity matching và judge thủ công/LLM. Đánh đổi là phép đo phức tạp và tốn thêm chi phí chấm.

## 4. Kết luận

KG đáng tiền cho câu hỏi xuyên nguồn/multi-hop: Q3 và Q5 tăng từ recall 0.33/0.40 lên 1.00; Q6 tăng từ 0.00 lên 1.00. Với câu đơn nguồn Q1–Q2, Flat RAG bằng GraphRAG (đều judge 2), nên không cần trả thêm 5.28× USD và 7.87× input token mỗi câu. Vì vậy nên dùng KG khi nhiều câu hỏi cần nối dữ kiện vụ án với Điều/khoản luật hoặc tổng hợp theo thực thể; Flat RAG đủ cho thông tin nằm trong một đoạn nguồn.

## 5. Tự kiểm

```text
$ pytest tests/ -q
48 passed in 0.04s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00255
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy.

## Vấn đề gặp phải

Lệnh tái chạy `python bench_kg.py --judge` sau cùng bị Gemini trả `403 PERMISSION_DENIED` với key thay thế. Trước đó, một lần chạy đầy đủ đã thành công và sinh `ket_qua_benchmark_kg.txt` đang dùng ở trên. Key Gemini trước đó cũng gặp quota embedding miễn phí; code đã thêm retry 429 cho Gemini.
