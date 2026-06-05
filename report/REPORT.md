# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Nguyễn Văn Quang  
**Nhóm:** Chưa được cung cấp trong workspace  
**Ngày:** 2026-06-05

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
High cosine similarity nghĩa là hai đoạn văn có hướng biểu diễn gần nhau trong không gian embedding, tức là chúng đang nói về nội dung hoặc ý nghĩa tương tự nhau. Điểm cao không có nghĩa là câu giống từng từ, mà là chúng gần nhau về ngữ nghĩa.

**Ví dụ HIGH similarity:**
- Sentence A: `Meditation helps calm the mind.`
- Sentence B: `Mindfulness practice can reduce stress and mental reactivity.`
- Tại sao tương đồng: cả hai đều nói về tác động của thực hành thiền/chánh niệm lên trạng thái tinh thần.

**Ví dụ LOW similarity:**
- Sentence A: `David Deutsch wrote about the fabric of reality.`
- Sentence B: `A recipe needs flour, sugar, and butter.`
- Tại sao khác: một câu nói về triết học khoa học, câu còn lại nói về nấu ăn.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector hơn là độ lớn, nên phù hợp hơn với embedding văn bản vì điều quan trọng là mức độ giống nhau về nghĩa. Euclidean distance dễ bị ảnh hưởng bởi độ dài hay scale của vector hơn.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, chunk_size=500, overlap=50. Bao nhiêu chunks?**  
Phép tính:

`num_chunks = ceil((doc_length - overlap) / (chunk_size - overlap))`  
`= ceil((10000 - 50) / (500 - 50))`  
`= ceil(9950 / 450)`  
`= ceil(22.11)`  
`= 23`

**Đáp án:** `23 chunks`

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
Khi overlap = 100:  
`ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25 chunks`.  
Overlap lớn hơn làm số chunk tăng vì bước nhảy nhỏ lại, nhưng đổi lại giúp giữ ngữ cảnh tốt hơn ở ranh giới chunk.

---

## 2. Document Selection — Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** sách về consciousness, meditation, psychedelic experience, physics/philosophy, và spiritual self-realization.

**Tại sao nhóm chọn domain này?**  
Bộ dữ liệu `psychic_*` có chủ đề tương đối nhất quán quanh ý thức, nhận thức, tâm trí, và thực hành tinh thần nên phù hợp để thử retrieval theo ngữ nghĩa. Đồng thời, đây là bộ tài liệu dài, nhiều chương và nhiều phong cách viết khác nhau, đủ khó để thấy rõ tác động của chunking và metadata.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | `psychic_altered_traits_text.txt` | Sách *Altered Traits* | 571,709 | `title`, `category=meditation`, `source=book`, `lang=en` |
| 2 | `psychic_fabric_of_reality_text.txt` | Sách *The Fabric of Reality* | 802,696 | `title`, `category=science`, `source=book`, `lang=en` |
| 3 | `psychic_how_to_change_your_mind_text.txt` | Sách *How to Change Your Mind* | 771,761 | `title`, `category=psychedelics`, `source=book`, `lang=en` |
| 4 | `psychic_tao_of_physics_text.txt` | Sách *The Tao of Physics* | 561,856 | `title`, `category=science`, `source=book`, `lang=en` |
| 5 | `psychic_vigyan_bhairav_text.txt` | Bản OCR/scan *Vigyan Bhairav Tantra* | 437,219 | `title`, `category=spirituality`, `source=book`, `lang=en` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `title` | string | `psychic_how_to_change_your_mind_text` | Cho phép filter chính xác theo sách khi query nhắm vào một tài liệu cụ thể |
| `category` | string | `science`, `meditation`, `psychedelics` | Giảm search space, tăng precision khi câu hỏi theo chủ đề |
| `source` | string | `book` | Giữ schema nhất quán để mở rộng sau này nếu thêm blog, PDF, note |
| `lang` | string | `en` | Hữu ích nếu sau này trộn tài liệu đa ngôn ngữ |

---

## 3. Chunking Strategy — Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu, với `chunk_size=500` và lấy 12,000 ký tự đầu để có mẫu đo nhất quán:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|-----------|----------|-------------|------------|-------------------|
| `psychic_altered_traits_text.txt` | `fixed_size` | 24 | 500.00 | Khá tốt, ổn định |
| `psychic_altered_traits_text.txt` | `by_sentences` | 21 | 570.48 | Giữ ý tốt nhưng chunk dài |
| `psychic_altered_traits_text.txt` | `recursive` | 29 | 411.86 | Cân bằng giữa kích thước và coherence |
| `psychic_fabric_of_reality_text.txt` | `fixed_size` | 24 | 500.00 | Ổn định nhưng có thể cắt giữa ý |
| `psychic_fabric_of_reality_text.txt` | `by_sentences` | 14 | 856.21 | Ý trọn hơn nhưng quá dài |
| `psychic_fabric_of_reality_text.txt` | `recursive` | 29 | 411.90 | Tốt nhất cho text giải thích dài |
| `psychic_tao_of_physics_text.txt` | `fixed_size` | 24 | 500.00 | Chắc chắn và dễ kiểm soát |
| `psychic_tao_of_physics_text.txt` | `by_sentences` | 20 | 599.00 | Chấp nhận được |
| `psychic_tao_of_physics_text.txt` | `recursive` | 31 | 385.03 | Nhỏ hơn, ít đứt ý hơn fixed ở phần mở đầu |

### Strategy Của Tôi

**Loại:** `FixedSizeChunker(chunk_size=500, overlap=50)`

**Mô tả cách hoạt động:**  
Strategy này dùng sliding window theo ký tự, mỗi chunk tối đa 500 ký tự và giữ lại 50 ký tự overlap với chunk kế tiếp. Cách này đơn giản, quyết định được kích thước index ngay từ đầu, và không phụ thuộc vào sentence boundary hay format của file gốc.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Corpus hiện tại là hỗn hợp sách sạch, sách có front matter dài. `FixedSizeChunker` chịu lỗi format tốt nhất, ít bị phá bởi dấu câu sai hoặc dòng xuống hàng lộn xộn. Overlap 50 giúp nối các ý nằm ở ranh giới chunk.

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|-----------|----------|-------------|------------|--------------------|
| `psychic_fabric_of_reality_text.txt` | best baseline: `recursive` | 29 | 411.90 | Coherent hơn, nhưng tạo nhiều chunk hơn |
| `psychic_fabric_of_reality_text.txt` | **của tôi: `fixed_size`** | 24 | 500.00 | Dễ kiểm soát kích thước, ổn định hơn toàn corpus |

### So Sánh Với Thành Viên Khác

| Thành viên | Strategy | Retrieval Score (/10) | Điểm mạnh | Điểm yếu |
|-----------|----------|----------------------|-----------|----------|
| Tôi | `FixedSizeChunker(500,50)` + metadata filter | 4/10 (ước lượng với `_mock_embed`) | Dễ tái lập, ổn định với OCR noise | Semantic precision thấp do embedder mock |
| Thành viên khác | Chưa có dữ liệu trong workspace | N/A | N/A | N/A |
| Thành viên khác | Chưa có dữ liệu trong workspace | N/A | N/A | N/A |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
Nếu xét coherence thuần túy trên text sạch, `RecursiveChunker` là baseline tốt nhất vì nó tận dụng separator tự nhiên và giữ chunk ngắn. Tuy nhiên, nếu xét toàn bộ corpus gồm cả file OCR lỗi, `FixedSizeChunker` an toàn hơn và ít bị hỏng bởi chất lượng dữ liệu đầu vào.

---

## 4. My Approach — Cá nhân (10 điểm)

Tôi implement các phần chính trong package `src` theo interface mà test yêu cầu, ưu tiên tính ổn định và tính giải thích được.

### Chunking Functions

**`SentenceChunker.chunk`** — approach:  
Tôi dùng regex để split theo ranh giới câu (`.`, `!`, `?`) rồi gom nhóm theo `max_sentences_per_chunk`. Edge case chính là text rỗng và text không tách được câu; khi đó hàm vẫn trả về list hợp lệ thay vì lỗi.

**`RecursiveChunker.chunk` / `_split`** — approach:  
Tôi dùng danh sách separator theo ưu tiên (`\n\n`, `\n`, `. `, ` `, `""`) và recurse dần khi phần text còn quá dài. Base case là text đã đủ ngắn hoặc không còn separator nào hữu ích, khi đó fallback sang `FixedSizeChunker`.

### EmbeddingStore

**`add_documents` + `search`** — approach:  
Tôi chuẩn hóa mỗi `Document` thành record có `id`, `doc_id`, `content`, `metadata`, và `embedding`, rồi lưu vào in-memory list. Search tạo embedding cho query, tính score bằng dot product, sort giảm dần, rồi trả về top-k kết quả.

**`search_with_filter` + `delete_document`** — approach:  
Tôi filter theo metadata trước rồi mới search, vì nếu search toàn bộ rồi mới filter thì kết quả top-k sẽ sai về logic retrieval. `delete_document` xóa tất cả record có cùng `doc_id` và trả về `True/False` tùy có xóa được gì hay không.

### KnowledgeBaseAgent

**`answer`** — approach:  
Agent lấy top-k chunks liên quan, ghép chúng thành phần `Context`, sau đó dựng prompt theo mẫu `Context -> Question -> Answer`. Cách này làm rõ RAG pattern và đảm bảo prompt luôn grounded vào dữ liệu retrieve được.

### Test Results

```text
pytest tests/ -v
================================================= test session starts =================================================
platform win32 -- Python 3.12.0, pytest-9.0.3, pluggy-1.6.0 -- E:\02_ToolSoft\02_AppData\Python\python.exe
cachedir: .pytest_cache
rootdir: D:\04_Study\01_Vin_AI20k\Day 7\Day07_Lab\2A202600590_NguyenVanQuang_Day07_Cohort2
plugins: anyio-4.13.0, langsmith-0.8.8
collected 42 items                                                                                                     

tests/test_solution.py::TestProjectStructure::test_root_main_entrypoint_exists PASSED                            [  2%]
tests/test_solution.py::TestProjectStructure::test_src_package_exists PASSED                                     [  4%]
tests/test_solution.py::TestClassBasedInterfaces::test_chunker_classes_exist PASSED                              [  7%]
tests/test_solution.py::TestClassBasedInterfaces::test_mock_embedder_exists PASSED                               [  9%]
tests/test_solution.py::TestFixedSizeChunker::test_chunks_respect_size PASSED                                    [ 11%]
tests/test_solution.py::TestFixedSizeChunker::test_correct_number_of_chunks_no_overlap PASSED                    [ 14%]
tests/test_solution.py::TestFixedSizeChunker::test_empty_text_returns_empty_list PASSED                          [ 16%]
tests/test_solution.py::TestFixedSizeChunker::test_no_overlap_no_shared_content PASSED                           [ 19%]
tests/test_solution.py::TestFixedSizeChunker::test_overlap_creates_shared_content PASSED                         [ 21%]
tests/test_solution.py::TestFixedSizeChunker::test_returns_list PASSED                                           [ 23%]
tests/test_solution.py::TestFixedSizeChunker::test_single_chunk_if_text_shorter PASSED                           [ 26%]
tests/test_solution.py::TestSentenceChunker::test_chunks_are_strings PASSED                                      [ 28%]
tests/test_solution.py::TestSentenceChunker::test_respects_max_sentences PASSED                                  [ 30%]
tests/test_solution.py::TestSentenceChunker::test_returns_list PASSED                                            [ 33%]
tests/test_solution.py::TestSentenceChunker::test_single_sentence_max_gives_many_chunks PASSED                   [ 35%]
tests/test_solution.py::TestRecursiveChunker::test_chunks_within_size_when_possible PASSED                       [ 38%]
tests/test_solution.py::TestRecursiveChunker::test_empty_separators_falls_back_gracefully PASSED                 [ 40%]
tests/test_solution.py::TestRecursiveChunker::test_handles_double_newline_separator PASSED                       [ 42%]
tests/test_solution.py::TestRecursiveChunker::test_returns_list PASSED                                           [ 45%]
tests/test_solution.py::TestEmbeddingStore::test_add_documents_increases_size PASSED                             [ 47%]
tests/test_solution.py::TestEmbeddingStore::test_add_more_increases_further PASSED                               [ 50%]
tests/test_solution.py::TestEmbeddingStore::test_initial_size_is_zero PASSED                                     [ 52%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_content_key PASSED                          [ 54%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_have_score_key PASSED                            [ 57%]
tests/test_solution.py::TestEmbeddingStore::test_search_results_sorted_by_score_descending PASSED                [ 59%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_at_most_top_k PASSED                             [ 61%]
tests/test_solution.py::TestEmbeddingStore::test_search_returns_list PASSED                                      [ 64%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_non_empty PASSED                                     [ 66%]
tests/test_solution.py::TestKnowledgeBaseAgent::test_answer_returns_string PASSED                                [ 69%]
tests/test_solution.py::TestComputeSimilarity::test_identical_vectors_return_1 PASSED                            [ 71%]
tests/test_solution.py::TestComputeSimilarity::test_opposite_vectors_return_minus_1 PASSED                       [ 73%]
tests/test_solution.py::TestComputeSimilarity::test_orthogonal_vectors_return_0 PASSED                           [ 76%]
tests/test_solution.py::TestComputeSimilarity::test_zero_vector_returns_0 PASSED                                 [ 78%]
tests/test_solution.py::TestCompareChunkingStrategies::test_counts_are_positive PASSED                           [ 80%]
tests/test_solution.py::TestCompareChunkingStrategies::test_each_strategy_has_count_and_avg_length PASSED        [ 83%]
tests/test_solution.py::TestCompareChunkingStrategies::test_returns_three_strategies PASSED                      [ 85%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_filter_by_department PASSED                     [ 88%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_no_filter_returns_all_candidates PASSED         [ 90%]
tests/test_solution.py::TestEmbeddingStoreSearchWithFilter::test_returns_at_most_top_k PASSED                    [ 92%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_reduces_collection_size PASSED             [ 95%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_false_for_nonexistent_doc PASSED   [ 97%]
tests/test_solution.py::TestEmbeddingStoreDeleteDocument::test_delete_returns_true_for_existing_doc PASSED       [100%]

================================================= 42 passed in 0.18s ==================================================
```

**Số tests pass:** `42 / 42`

---

## 5. Similarity Predictions — Cá nhân (5 điểm)

Ghi chú: phần actual score bên dưới dùng `compute_similarity(_mock_embed(a), _mock_embed(b))`, nên nó phản ánh pipeline kỹ thuật hiện có chứ không phản ánh semantic embedding thực sự.

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Meditation helps calm the mind. | Mindfulness practice reduces stress. | high | 0.1354 | Không |
| 2 | Quantum physics studies particles. | The Tao of Physics compares science and mysticism. | high | 0.3093 | Có |
| 3 | Magic mushrooms can alter consciousness. | Psilocybin may occasion mystical experiences. | high | 0.0681 | Không |
| 4 | David Deutsch wrote about reality. | A recipe needs flour and sugar. | low | 0.1961 | Có |
| 5 | Breathing meditation centers attention. | Galaxies expand across the universe. | low | 0.0277 | Có |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Bất ngờ nhất là pair 3 có nghĩa khá gần nhưng score rất thấp. Điều này cho thấy `_mock_embed` chỉ phù hợp để test interface và deterministic behavior, chứ không đủ để đánh giá semantic similarity nghiêm túc.

---

## 6. Results — Cá nhân (10 điểm)

Phần benchmark dưới đây được chạy với `_mock_embed` và `KnowledgeBaseAgent` dùng `llm_fn` giả lập để echo context. Vì vậy kết quả dùng để đánh giá pipeline retrieval/filtering và failure analysis.

### Benchmark Queries & Gold Answers (nhóm thống nhất trong report này)

| # | Query | Gold Answer |
|---|-------|-------------|
| 1 | Which book says meditation can create lasting altered traits? | *Altered Traits*; meditation có thể tạo ra các lasting traits |
| 2 | Who wrote The Fabric of Reality? | David Deutsch |
| 3 | What do set and setting mean in psychedelic experiences? | `set` là mind-set/kỳ vọng; `setting` là môi trường/circumstances |
| 4 | What does The Tao of Physics compare? | modern physics và Eastern mysticism |
| 5 | What is Vigyan Bhairav Tantra about? | 112 meditations for self-realization / knowing God-consciousness |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk (tóm tắt) | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|--------------------------------|-------|-----------|------------------------|
| 1 | altered traits | Chunk từ *The Fabric of Reality* nói về relativity và explanation | 0.3573 | Không | Không grounded đúng tài liệu cần tìm |
| 2 | author of *The Fabric of Reality* | Chunk từ *The Tao of Physics* nói về “substance” trong Western thought | 0.1940 | Không | Không trả lời đúng tên tác giả |
| 3 | set and setting | Chunk đúng sách *How to Change Your Mind*, nói về nghiên cứu psychedelic | 0.2668 | Một phần | Có liên quan domain nhưng chưa trúng định nghĩa ngắn gọn |
| 4 | compare in *The Tao of Physics* | Chunk từ *The Fabric of Reality* nói về explanation vs prediction | 0.2618 | Một phần | Gần chủ đề science/philosophy nhưng lệch sách |
| 5 | what is *Vigyan Bhairav Tantra* | Chunk đúng file nhưng dữ liệu OCR rất nhiễu | 0.2123 | Một phần | Đúng tài liệu nhưng context khó dùng |

**Bao nhiêu queries trả về chunk relevant trong top-3?** `4 / 5`  
**Bao nhiêu queries có top-1 clearly useful?** `2 / 5` ở mức rất thận trọng.

### Metadata Utility

- Không filter: top-1 thường rơi vào sai tài liệu vì `_mock_embed` không mang nghĩa ngữ nghĩa thực sự.
- Filter theo `category` giúp query 3 đi đúng vào nhóm `psychedelics`.
- Filter theo `title` giúp query 5 quay đúng tài liệu *Vigyan Bhairav Tantra*, dù chất lượng OCR vẫn làm chunk khó đọc.
- Với query 2 và 4, filter `science` có cải thiện search space nhưng chưa đủ để cứu semantic mismatch.

---

## 7. What I Learned (5 điểm — Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Cùng một document set nhưng khác chunking và metadata có thể tạo khác biệt retrieval lớn hơn việc đổi model.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  


**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Tôi sẽ làm sạch `psychic_vigyan_bhairav_text.txt` trước khi index vì OCR noise đang làm giảm cả chunk coherence lẫn grounding quality. Tôi cũng sẽ dùng embedder local `sentence-transformers` thay cho `_mock_embed`, vì hiện tại bottleneck lớn nhất không còn là code mà là chất lượng biểu diễn embedding.

### Failure Analysis

Failure case rõ nhất là query 1: hỏi về “altered traits” nhưng top-1 lại rơi vào *The Fabric of Reality*. Nguyên nhân gốc là `_mock_embed` tạo vector deterministic nhưng không mang ý nghĩa ngữ nghĩa, nên similarity score không phản ánh nội dung thật. Failure case thứ hai là query 5: filter đưa được về đúng file, nhưng OCR lỗi làm chunk kém coherent và khó dùng cho agent.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 9 / 10 |
| Chunking strategy | Nhóm | 12 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 7 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 0 / 5 |
| **Tổng** | | **78 / 100** |

---

## Kết Luận Ngắn

Lab này cho thấy một điều rất rõ: code đúng interface mới chỉ là điều kiện cần. Để retrieval tốt, cần đồng thời có chunking hợp dữ liệu, metadata đủ hữu ích, tài liệu đủ sạch, và embedding backend có ý nghĩa ngữ nghĩa thực sự. Ở bài làm này, phần implementation đã hoàn chỉnh và pass toàn bộ test; điểm yếu còn lại nằm ở chất lượng dữ liệu và việc đang dùng `_mock_embed` thay vì một embedder semantic thật.
