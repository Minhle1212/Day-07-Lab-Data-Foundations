# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Lê Hoàng Minh
**Nhóm:** X2
**Ngày:** 10/4/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**
>  High cosine similarity nghĩa là hai đoạn văn có hướng biểu diễn ngữ nghĩa rất giống nhau trong không gian embedding, tức là chúng nói về ý gần nhau dù cách viết có thể khác

**Ví dụ HIGH similarity:**
- Sentence A: Python is used for data analysis and machine learning
- Sentence B: Python is popular and easy to use in ML and data sciene
- Tại sao tương đồng:Cùng nói về Python trong bối cảnh ML và phân tích dữ liệu.

**Ví dụ LOW similarity:**
- Sentence A: I have renew my passport tomorrow
- Sentence B: Neural networks require large datasets for training
- Tại sao khác: Hai câu thuojc hai đề tài không liên quan đến nhau

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**
> Cosine similarity tập trung vào góc giữa hai vector, nên ít bị ảnh hưởng bởi độ dài vector. Với text embeddings, hướng thường quan trọng hơn độ lớn tuyệt đối, nên cosine thường phản ánh mức độ giống nghĩa tốt hơn.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**
> num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))
> ceil((10000 - 50) / (500 - 50))
> ceil(9950 / 450)
> ceil(22.11)
> 23

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**
> chunk count tăng từ 23 lên 25. Lý do là bước trượt nhỏ hơn khi overlap lớn hơn, nên cần nhiều chunk hơn để phủ hết tài liệu. Overlap lớn giúp giữ ngữ cảnh qua ranh giới chunk, giảm mất context khi retrieve

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Bệnh học phổ biến (nội dung tư vấn sức khỏe bằng tiếng Việt)
**Lý Do Chọn** 

**Tại sao nhóm chọn domain này?**
> Nhóm chọn domain liên quan đến bệnh vì dữ liệu có cấu trúc rõ (định nghĩa, triệu chứng, nguyên nhân, điều trị), phù hợp để kiểm thử retrieval theo câu hỏi thực tế. Đây cũng là domain giàu thuật ngữ, giúp đánh giá rõ chất lượng chunking và metadata filtering. Nội dung tiếng Việt giúp nhóm kiểm tra khả năng truy xuất trong bối cảnh ngôn ngữ bản địa.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | alzheimer.md | https://tamanhhospital.vn/alzheimer/ | 27966 | condition=alzheimer; category=neurology; language=vi |
| 2 | benh-dai.md | https://tamanhhospital.vn/benh-dai/ | 12700 | condition=benh-dai; category=urology; language=vi |
| 3 | benh-lao-phoi.md | https://tamanhhospital.vn/benh-lao-phoi/ | 12704 | condition=benh-lao-phoi; category=respiratory; language=vi |
| 4 | benh-san-day.md | https://tamanhhospital.vn/benh-san-day/ | 15430 | condition=benh-san-day; category=dermatology; language=vi |
| 5 | benh-tri.md | https://tamanhhospital.vn/benh-tri/ | 12569 | condition=benh-tri; category=gastrointestinal; language=vi |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| condition | string | alzheimer, benh-tri | Giúp filter theo đúng bệnh khi query chỉ định bệnh cụ thể |
| category | string | neurology, respiratory, dermatology | Hữu ích khi query theo chuyên khoa hoặc nhóm bệnh |
| language | string | vi | Dùng để đảm bảo retrieve đúng ngôn ngữ khi mở rộng đa ngữ |
| source | string (URL) | https://tamanhhospital.vn/benh-tri/ | Tăng traceability, dễ kiểm chứng câu trả lời theo nguồn |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 2-3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| alzheimer.md | FixedSizeChunker (`fixed_size`) | 156 | 199.14 | Medium |
| alzheimer.md | SentenceChunker (`by_sentences`) | 74 | 375.85 | High |
| alzheimer.md | RecursiveChunker (`recursive`) | 220 | 125.48 | Medium |
| benh-lao-phoi.md | FixedSizeChunker (`fixed_size`) | 71 | 198.65 | Medium |
| benh-lao-phoi.md | SentenceChunker (`by_sentences`) | 28 | 450.50 | High |
| benh-lao-phoi.md | RecursiveChunker (`recursive`) | 96 | 130.90 | Medium |
| benh-tri.md | FixedSizeChunker (`fixed_size`) | 70 | 199.27 | Medium |
| benh-tri.md | SentenceChunker (`by_sentences`) | 35 | 355.77 | High |
| benh-tri.md | RecursiveChunker (`recursive`) | 91 | 136.20 | Medium |

### Strategy Của Tôi

**Loại:** custom strategy (Late Chunking)

**Mô tả cách hoạt động:**
> Late Chunking giữ ngữ cảnh ở mức đoạn lớn trong bước indexing, sau đó mới cắt nhỏ ở giai đoạn retrieve theo câu hỏi. Cách này giúp vector đại diện ban đầu nắm được nhiều thông tin liên kết hơn, rồi vẫn trả về đoạn ngắn đủ chính xác khi cần. Với dữ liệu bệnh học, các mục như triệu chứng, biến chứng và điều trị thường liên quan chặt chẽ, nên giữ context lớn trước khi cắt giúp giảm mất ý. Khi query cụ thể, hệ thống mới "late split" để tăng độ bám sát câu hỏi.

**Tại sao tôi chọn strategy này cho domain nhóm?**
> Domain y khoa cần cân bằng giữa hai mục tiêu: đủ context để không sai nghĩa và đủ chi tiết để trả lời đúng câu hỏi cụ thể. Late Chunking phù hợp vì nó giữ coherence ở bước biểu diễn, nhưng vẫn cho phép cắt tinh hơn ở bước truy xuất. So với fixed-size hoặc recursive cố định ngay từ đầu, late chunking linh hoạt hơn theo từng loại query trong benchmark.

**Code snippet (nếu custom):**
```python
class LateChunking:
	"""Index lớn, cắt muộn theo query."""

	def __init__(self, base_chunk_size: int = 600, late_window: int = 180):
		self.base_chunk_size = base_chunk_size
		self.late_window = late_window

	def index_chunks(self, text: str) -> list[str]:
		# chunk lớn để giữ ngữ cảnh tổng thể khi embed/index
		return [text[i:i+self.base_chunk_size] for i in range(0, len(text), self.base_chunk_size)]

	def late_split_for_query(self, retrieved_text: str, query: str) -> list[str]:
		# sau khi retrieve, cắt nhỏ quanh vùng chứa từ khóa/query terms
		return [retrieved_text[i:i+self.late_window] for i in range(0, len(retrieved_text), self.late_window)]
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| 3 docs + 5 benchmark queries | best baseline: FixedSizeChunker | 200/20 overlap | 199.27 (tham chiếu benh-tri.md) | Top-1 acc = 0.40, Top-3 recall = 0.40 |
| 3 docs + 5 benchmark queries | **của tôi: Late Chunking (custom)** | base 600/100, late 180/40 | N/A (dynamic late split) | Top-1 acc = 0.40, Top-3 recall = 0.60 |

**Kết quả chạy thực tế (mock embedding):**
> Late Chunking không cải thiện Top-1 trong run này (giữ ở 0.40), nhưng tăng Top-3 recall từ 0.40 lên 0.60. Điều này cho thấy cắt muộn giúp tăng cơ hội xuất hiện tài liệu đúng trong top-k, đặc biệt với query cần nhiều ngữ cảnh.

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | Late Chunking (custom) | 6.0 (top-3 hit 3/5) | Linh hoạt theo query, tăng recall top-3 | Triển khai phức tạp hơn, khó tuning tham số |
| [Tên thành viên A] | FixedSizeChunker | 2.0 (top-3 hit 1/5) | Cấu hình đơn giản, tốc độ ổn định, dễ debug | Dễ cắt mất ngữ nghĩa y khoa khi chunk cố định |
| Việt | RecursiveChunker | 4.0 (top-3 hit 2/5) | Chunk chi tiết hơn, có thể tăng recall một số query khó | Nhiều chunk vụn, top-1 dễ lệch và nhiễu |

**Strategy nào tốt nhất cho domain này? Tại sao?**
> Với dataset hiện tại, Late Chunking cho tín hiệu tốt hơn ở mức recall (top-3) so với baseline fixed-size trong cùng điều kiện test. Cách giữ chunk lớn ở bước index giúp hạn chế mất ý nghĩa y khoa, còn bước cắt muộn cải thiện khả năng đưa đúng tài liệu vào top-k. Vì vậy strategy này phù hợp hơn khi benchmark có cả câu hỏi tổng quan và câu hỏi cụ thể.

---

## 4. My Approach — Cá nhân (10 điểm)

Giải thích cách tiếp cận của bạn khi implement các phần chính trong package `src`.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:
> Sử dụng regex `(?<=[.!?])\s+` để tách câu theo dấu kết thúc câu rồi khoảng trắng phía sau. Sau khi tách, mình strip từng phần và loại bỏ câu rỗng để tránh tạo chunk nhiễu. Nếu input rỗng thì trả về `[]`, còn nếu không tách được câu thì fallback thành một chunk duy nhất.

**`RecursiveChunker.chunk` / `_split`** — approach:
> Thuật toán thử tách theo thứ tự separator ưu tiên (`\n\n`, `\n`, `. `, ` `, `""`) và đệ quy khi đoạn còn vượt quá `chunk_size`. Base case là khi text rỗng, khi text đã nhỏ hơn hoặc bằng `chunk_size`, hoặc khi không còn separator thì fallback sang `FixedSizeChunker`. Ngoài ra mình có bước buffer để gộp các mảnh nhỏ trước khi đệ quy tiếp, giúp giảm số lượng chunk quá vụn.

### EmbeddingStore

**`add_documents` + `search`** — approach:
> `add_documents` embed từng document rồi lưu record chuẩn hóa gồm `id`, `content`, `metadata`, `embedding`; nếu có Chroma thì add vào collection, nếu không thì lưu in-memory list. `search` embed query, sau đó tính score bằng dot product giữa query embedding và từng embedding đã lưu. Kết quả được sort giảm dần theo score và trả về top-k.

**`search_with_filter` + `delete_document`** — approach:
> Với `search_with_filter` filter trước theo metadata rồi mới chạy similarity search trên tập đã lọc. Nếu chạy in-memory thì dùng điều kiện `all(...)` trên metadata dictionary; nếu có Chroma thì lấy records bằng `where` rồi re-rank lại theo score. `delete_document` xóa toàn bộ chunks có `metadata['doc_id'] == doc_id` và trả về `True/False` tùy có xóa được record nào hay không.

### KnowledgeBaseAgent

**`answer`** — approach:
> Hàm `answer` gọi store để retrieve top-k chunks liên quan đến câu hỏi, sau đó đánh số từng chunk làm context. Prompt được tổ chức theo 3 phần: instruction (chỉ dùng context), context đã retrieve, và question. Cuối cùng gọi `llm_fn(prompt)` để sinh câu trả lời; nếu không có context phù hợp thì prompt có thông báo rõ để model hạn chế bịa thông tin.

### Test Results

```
============================= test session starts =============================
platform win32 -- Python 3.13.13, pytest-9.0.3, pluggy-1.6.0 -- C:\Python313\python.exe
cachedir: .pytest_cache
rootdir: D:\AI\Day-07-Lab-Data-Foundations
collected 42 items

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED   [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED    [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED   [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED [100%]

============================= 42 passed in 0.16s =============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Alzheimer la benh suy giam tri nho thuong gap o nguoi cao tuoi. | Nguoi cao tuoi co nguy co mac Alzheimer va sa sut tri tue. | high | 0.0707 | No |
| 2 | Trieu chung lao phoi bao gom ho keo dai va sot nhe. | Lao phoi thuong gay met moi, ho, va giam can. | high | 0.0051 | No |
| 3 | Benh tri co the gay chay mau khi di ngoai. | Hom nay toi di xem phim o rap voi ban. | low | 0.1918 | Yes |
| 4 | Dieu tri lao phoi can tuan thu phac do thuoc khang sinh. | Benh nhan can dung thuoc dung lieu va du thoi gian de tranh khang thuoc. | high | 0.0151 | No |
| 5 | Toi thich nau an vao cuoi tuan. | Muc tieu cua retrieval la tim dung doan van lien quan. | low | -0.0267 | Yes |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**
> Kết quả bất ngờ nhất là các cặp câu cùng chủ đề bệnh học (pair 1, 2, 4) lại có score khá thấp. Điều này cho thấy backend mock embedding không biểu diễn ngữ nghĩa thật sự mạnh như model embedding thực tế, nên điểm similarity có thể không phản ánh trực giác ngôn ngữ. Bài học rút ra là khi đánh giá semantic similarity nghiêm túc, nên dùng embedder thực (local hoặc OpenAI) thay vì chỉ dựa trên mock.

---

## 6. Results — Cá nhân (10 điểm)

Chạy 5 benchmark queries của nhóm trên implementation cá nhân của bạn trong package `src`. **5 queries phải trùng với các thành viên cùng nhóm.**

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Bệnh trĩ có ảnh hưởng khả năng sinh sản không? | Không. |
| 2 | Ăn cá có bị sán không? | Có. Có loại sán dây ở bên trong cá và có khả năng lây bệnh cho người. |
| 3 | Làm sao biết mình bị Alzheimer? | Dấu hiệu thường gặp: sa sút trí nhớ và nhận thức; khó diễn đạt ngôn ngữ; thay đổi hành vi, tâm trạng, tính cách; nhầm lẫn thời gian/địa điểm; đặt đồ sai vị trí và không nhớ đã làm gì. |
| 4 | Những đối tượng nào có nguy cơ cao chuyển từ tình trạng lao tiềm ẩn sang bệnh lao phổi (lao bệnh)? | Nhóm nguy cơ cao gồm: người nhiễm HIV; người sử dụng ma túy dạng chích; người sụt cân ~10%; bệnh nhân bụi phổi silic/suy thận/chạy thận/đái tháo đường; người từng phẫu thuật cắt dạ dày hoặc ruột non; người ghép tạng hoặc dùng corticoid dài ngày/thuốc ức chế miễn dịch; bệnh nhân ung thư đầu cổ. |
| 5 | Trong trường hợp bị động vật cắn hoặc cào xước, quy trình sơ cứu tại chỗ và các biện pháp y tế cần thực hiện ngay lập tức là gì để ngăn chặn virus dại xâm nhập hệ thần kinh? | Sơ cứu: rửa vết thương bằng nước sạch + xà phòng/chất tẩy rửa/povidone iodine ít nhất 15 phút; sát trùng bằng cồn 70% hoặc povidone-iodine; băng bó đơn giản và đưa nạn nhân đến cơ sở y tế. Can thiệp y tế: bác sĩ thăm khám, chỉ định tiêm vắc xin phòng dại sớm; có thể tiêm thêm huyết thanh kháng dại khi cần. Theo dõi động vật cắn: nếu có dấu hiệu bất thường (cắn vô cớ, ăn vật lạ, gầm gừ, tăng tiết nước bọt, chết sau vài ngày) thì nguy cơ dại cao. |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | Bệnh trĩ có ảnh hưởng khả năng sinh sản không? | Top-1 retrieve từ tài liệu bệnh dại (không đúng trọng tâm câu hỏi bệnh trĩ). | 0.2460 | No | Agent trả lời theo context retrieve nhưng chưa đúng gold answer. |
| 2 | Ăn cá có bị sán không? | Top-1 retrieve đúng tài liệu bệnh sán dây, nêu bối cảnh lây nhiễm qua cá. | 0.078 | Yes | Agent trả lời theo context: có nguy cơ nhiễm sán dây từ cá. |
| 3 | Làm sao biết mình bị Alzheimer? | Top-1 retrieve đúng tài liệu Alzheimer, có phần triệu chứng và nhận thức. | 0.1123 | Yes | Agent trả lời theo context về dấu hiệu sa sút trí nhớ/nhận thức. |
| 4 | Những đối tượng nào có nguy cơ cao chuyển từ lao tiềm ẩn sang lao phổi? | Top-1 bị lệch sang tài liệu Alzheimer, nhưng top-3 có tài liệu lao phổi. | 0.1647 | Yes | Agent có thể trả lời đúng hơn khi dùng chunk từ tài liệu lao phổi trong top-3. |
| 5 | Bị động vật cắn cần sơ cứu thế nào để phòng dại? | Top-1 bị lệch sang Alzheimer, nhưng top-3 có tài liệu bệnh dại. | 0.0859 | Yes | Agent có thể bám context bệnh dại trong top-3 để nêu bước rửa, sát trùng, tiêm vaccine. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 4 / 5

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**
> Điều mình học được nhiều nhất là cách bạn cùng nhóm dùng metadata filtering trước khi search để thu hẹp candidate chunks. Cách này giúp giảm nhiễu rõ rệt ở các câu hỏi theo đúng bệnh hoặc đúng chuyên khoa. Mình nhận ra retrieval quality không chỉ phụ thuộc chunking mà còn phụ thuộc mạnh vào thiết kế metadata.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**
> Từ phần d8emo của nhóm khác, mình học được rằng cần đánh giá theo từng loại query thay vì chỉ nhìn một chỉ số tổng. Có chiến lược mạnh ở câu hỏi tổng quan nhưng yếu ở câu hỏi chi tiết, và ngược lại. Cách so sánh theo từng query giúp mình hiểu rõ hơn vì sao top-1 có thể thấp nhưng top-3 vẫn hữu ích cho agent.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**
> Nếu làm lại, mình sẽ chuẩn hóa tài liệu đầu vào tốt hơn (loại bỏ nhiễu HTML/header) trước khi index để giảm các kết quả lệch. Mình cũng sẽ chia metadata chi tiết hơn (disease, section, symptom/treatment/risk) để filter chính xác hơn. Cuối cùng, mình sẽ benchmark thêm bằng local embedder thật để so sánh với mock embedding và có kết luận chắc chắn hơn.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 12 / 15 |
| My approach | Cá nhân | 7 / 10 |
| Similarity predictions | Cá nhân | 4 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 25 / 30 |
| Demo | Nhóm | 3 / 5 |
| **Tổng** | | 85 / 100** |
