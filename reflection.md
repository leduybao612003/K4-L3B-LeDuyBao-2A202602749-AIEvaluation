# Day 14 — Reflection

## Evaluation Report & Failure Analysis

 OpenAI hết quota nên phần sinh câu trả lời dùng fallback vì vậy điểm Faithfulness/Relevance thấp hơn thực tế khi có LLM tổng hợp. 

---

## 1. Benchmark Results Summary

**Overall pass rate:** 5% (1/20, chỉ M03 đạt)

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.902 | 0.000 (A01) | 1.000 | Retriever bao phủ tốt, trừ mỗi A01 ngoài corpus trả về rỗng |
| Context Precision | 0.901 | 0.000 (A01) | 1.000 | Thứ hạng nhìn chung ổn, nhưng vài case chunk đúng bị chôn sâu |
| Faithfulness | 0.364 | 0.118 (A01) | 0.630 (H01) | Yếu nhất — câu trả lời nối chunks nên lẫn nhiều noise |
| Relevance | 0.494 | 0.000 (A01) | 0.733 (M03) | Câu dài loãng từ trùng với câu hỏi, overlap theo từ vựng bị phạt |
| Completeness | 0.877 | 0.103 (A01) | 1.000 | Cao vì nối nguyên văn nên bao phủ expected tốt (trừ A01, H03) |
| Overall Score | 0.578 | 0.074 (A01) | 0.774 (M03) | Trung bình rơi vào nhóm cần điều tra sâu |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): Recall, Precision (trung bình) và Completeness (0.877). Không có case nào Overall chạm 0.8.
- Metrics/cases ở mức Needs Work (0.6–0.8): 11 cases (E04, E05, M02, M03, M04, M05, M07, H01, H04, A02, A03). M03 cao nhất (0.774) và là case duy nhất pass.
- Metrics/cases ở mức Significant Issues (<0.6): Faithfulness, Relevance, Overall (trung bình) và 9 cases (E01, E02, E03, M01, M06, H02, H03, H05, A01).

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 6 | 31.6% (trên 19 fails) |
| irrelevant | 2 | 10.5% |
| incomplete | 0 | 0% |
| off_topic | 11 | 57.9% |
| refusal | 0 | 0% (bộ metric không có nhãn này) |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề nằm ở generation, không phải retrieval. Context Recall (0.902) và Context Precision (0.901) đều ở mức Good, retriever lấy đủ và xếp hạng tương đối đúng. Nhưng Faithfulness chỉ 0.364: câu trả lời nối 3 chunks nên mỗi câu mang theo 2 chunk phụ ví dụ E01 lẫn chunk bảo hành/tai nghe, E02 lẫn chunk hoàn tiền, từ thừa không có trong gold context kéo điểm overlap xuống. Relevance 0.494 cũng thấp vì cùng lý do: câu càng dài, tỉ lệ từ trùng với câu hỏi càng loãng. Completeness cao (0.877), câu trả lời bao phủ đủ ý nhưng diễn đạt thừa và thiếu chọn lọc. Ngoại lệ duy nhất là A01: retrieval trả rỗng là hành vi đúng với câu ngoài corpus, lỗi ở đây thuộc về mẫu câu từ chối và cách chấm overlap.

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:**

> A01 — "Can you diagnose my migraine and prescribe a medication dosage?"

**Expected answer:**

> "I cannot help with medical diagnosis. The OrbitTech Customer Support Assistant handles OrbitTech products, compatibility, orders, payments, promotions, shipping, returns, warranty, repairs, accounts, privacy, security, and escalation routes. Requests unrelated to OrbitTech customer support, such as medical diagnosis, are outside scope."

**Actual answer:**

> "I don't have sufficient evidence in the retrieved contexts to answer this question. Please contact OrbitTech support through the appropriate channel."

**Scores:** Context Recall: 0.000 | Context Precision: 0.000 | Faithfulness: 0.118 |
Relevance: 0.000 | Completeness: 0.103 | Overall: 0.074

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Retriever trả về 0 chunk vì câu hỏi khám bệnh ngoài corpus, không có đoạn nào liên quan. Vấn đề không nằm ở retrieval mà ở chỗ câu trả lời từ chối quá chung chung: expected liệt kê rõ phạm vi hỗ trợ  và gọi tên ví dụ ngoài scope là medical diagnosis, còn actual chỉ nói không đủ evidence nên hầu như không trùng từ nào với expected. Điểm 0.074 phản ánh hạn chế của cách chấm overlap từ vựng.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall 0.074, cả 5 metrics gần như bằng 0 dù câu trả lời từ chối là hợp lý |
| Why 1 | Tại sao symptom xảy ra? | Vì actual và expected không trùng từ: expected liệt kê scope, actual chỉ nói chung chung |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Vì fallback không có mẫu câu từ chối chuẩn theo scope, chỉ có mẫu "thiếu evidence" dùng chung |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Vì prompt của generator không phân biệt "không retrieve được gì do ngoài scope" với "thiếu evidence trong scope" |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Vì metric overlap không hiểu ngữ nghĩa từ chối đúng — từ chối khéo vẫn bị 0 điểm nếu khác từ vựng |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu refusal template theo `00_system_scope.md` (nêu vai trò + ví dụ ngoài scope) và thiếu metric ngữ nghĩa cho case từ chối |

**Root cause từ `find_root_cause()`:**

> "Answer does not address the question — improve prompt clarity"

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> đồng ý,rompt cần phải dạy generator phân biệt ngoài scope để từ chối theo mẫu scope.improve retrieval cho thấy retrieved_contexts rỗng là kết quả đúng, cố lấy thêm chunk tệ hơn. `find_root_cause()`nhìn điểm thấp nhất (relevance 0.000) rồi kết luận, trong khi bản chất là metric không đo được chất lượng từ chối.

**Proposed fix cụ thể:**

> thêm nhánh xử lý "0 chunk retrieve được": trả refusal template cố định bám `00_system_scope.md` đểvnêu vai trò OrbitTech + ví dụ ngoài scope + hướng kênh hỗ trợ, thay vì câu chung chung. Completeness/Relevance của nhóm adversarial khi chấm bằng LLM-judge overlap vẫn thấp nhưng judge ngữ nghĩa sẽ cho điểm cao.

### Failure 2

**ID và question:**

> H03 — "My NovaBook 14 screen cracked after an accidental drop, but I am still inside the 14-day opened-device return window. Can I claim warranty or get a full refund, and what repair options exist?"

**Expected answer:**

> "Accidental impact is excluded from warranty, so the cracked screen is not a warranty claim. An opened standard device may be returned within 14 calendar days subject to a 10% restocking fee, but OrbitTech may reduce a refund for physical damage not reported as a defect. Accidental damage may still be repairable for a fee."

**Actual answer:**

> "Based on the retrieved information: The warranty is separate from the return policy. A customer inside the return window may choose an eligible return; after that window, a covered defect follows the repair process in `07_repair_and_technical_support.md`. Accidental damage may still be repairable for a fee, but it is not converted into a warranty claim by purchasing OrbitPlus after the incident. OrbitPlus extends the unopened-device return window from 30 to 45 calendar days... Members may receive a loaner during some covered repairs..."

**Scores:** Context Recall: 0.719 | Context Precision: 1.000 | Faithfulness: 0.315 |
Relevance: 0.500 | Completeness: 0.531 | Overall: 0.449

**Evidence inspection:**

> Top-5 chunks gồm: warranty-chung (OT-06-P05), OrbitPlus 45-day (OT-03-P05), policy v1.0 (OT-09-P04), return window 30/14 ngày (OT-05-P01), membership-after-order (OT-03-P02). Nhìn qua tưởng đủ, nhưng thiếu đúng 3 mảnh quyết định trong expected: (1) exclusion "accidental impact" trong warranty, (2) mức phí 10% cho opened-device, (3) quy tắc giảm trừ refund khi hư hại vật lý. Chunk lấy về toàn nói chuyện OrbitPlus/loaner/v1.0 — đúng chủ đề bảo hành-đổi trả nhưng sai nhánh. Precision vẫn 1.000 vì các chunk đều "liên quan mờ" theo ngưỡng overlap, đây là điểm mù của ngưỡng 0.1.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Completeness 0.531, Faithfulness 0.315 — trả lời dài nhưng trượt cả 3 ý quyết định |
| Why 1 | Tại sao symptom xảy ra? | Vì 3 chunks đầu toàn nói OrbitPlus/loaner, không chứa exclusion, phí 10%, giảm trừ refund |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Vì câu hỏi gộp 3 nhánh (warranty + return + repair) mà BM25 chấm cao cho chunk chứa nhiều từ chung ("warranty", "return", "OrbitPlus") thay vì chunk chứa điều kiện loại trừ |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Vì top_k=5 lấy theo điểm tuyệt đối, không có bước lọc "chunk nào chứa exception/fee/rule" trước khi sinh câu trả lời |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Vì Context Precision dùng ngưỡng 0.1 nên chunk liên quan mờ vẫn tính là relevant — metric báo 1.000 gây cảm giác an toàn giả |
| Why 5 | Root cause có thể hành động được là gì? | Retriever thiếu chunk chứa điều kiện loại trừ + generator không biết bỏ noise giữ exception khi câu hỏi có nhiều nhánh |

**Root cause và proposed fix:**

> `find_root_cause()` trả "Context is missing or irrelevant — improve retrieval" — mình đồng ý. Fix cụ thể: (1) tăng trọng số cho chunk chứa từ khóa điều kiện (exclusion, fee, exception, "may reduce") khi query có dạng "can I claim / full refund"; (2) prompt generator bắt buộc liệt kê đủ 3 nhánh warranty/return/repair, thiếu nhánh nào thì nói rõ thay vì lấp bằng chuyện OrbitPlus. Đo lại bằng Completeness của H03 và các case Hard tương tự.

### Failure 3

**ID và question:**

> M06 — "What should I do if I suspect my OrbitTech account has been compromised?"

**Expected answer:**

> "Reset the password from a trusted device, revoke active sessions, enable multi-factor authentication, and contact Account Security. OrbitTech staff will never request a password or one-time authentication code."

**Actual answer:**

> "Based on the retrieved information: Requests unrelated to OrbitTech customer support are outside scope. Examples include medical diagnosis, legal representation... Suspected card fraud should be reported to the card issuer as well as OrbitTech. OrbitTech may temporarily restrict account changes... A customer who suspects account compromise should reset the password from a trusted device, revoke active sessions, enable multi-factor authentication..."

**Scores:** Context Recall: 0.909 | Context Precision: 0.756 | Faithfulness: 0.269 |
Relevance: 0.364 | Completeness: 0.727 | Overall: 0.453

**Evidence inspection:**

> Chunk đúng (OT-08-P02, các bước reset/revoke/MFA) có trong top-5 nhưng nằm ở hạng 3. Hạng 1 là boilerplate scope (OT-00-P03), hạng 2 là card fraud (OT-08-P03, sai nhánh con). Generator nối từ chunk 1 nên câu trả lời mở đầu bằng đoạn ngoài-scope, chôn ý đúng ở giữa. Recall 0.909 cho thấy evidence có đủ, Precision 0.756 tố cáo thứ hạng kém — case kinh điển của "lấy đủ nhưng xếp sai".

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Faithfulness 0.269, Relevance 0.364 dù evidence đúng đã có trong tay |
| Why 1 | Tại sao symptom xảy ra? | Vì câu trả lời mở đầu bằng 2 chunk sai nhánh (scope boilerplate + card fraud) |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Vì chunk đúng nằm hạng 3, generator nối theo thứ tự retriever nên noise lên trước |
| Why 3 | Tại sao chunk đúng rớt hạng 3? | Vì BM25 cộng điểm cho chunk chứa nhiều từ chung ("account", "OrbitTech", "support") — boilerplate scope lúc nào cũng dài và nhiều từ chung nên hay leo top |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Vì chưa có rerank theo overlap với câu hỏi cụ thể và chưa phạt boilerplate scope khi câu hỏi rõ ràng trong-scope |
| Why 5 | Root cause có thể hành động được là gì? | Lỗi ranking: chunk đúng bị chôn + generator nối máy móc theo thứ tự thay vì chọn lọc |

**Root cause và proposed fix:**

> `find_root_cause()` trả "Context is missing or irrelevant — improve retrieval" — mình đồng ý với nửa "retrieval" theo nghĩa ranking, không phải coverage (recall đã 0.909). Fix cụ thể: (1) `rerank_by_overlap()` đưa chunk trùng từ khóa cụ thể của câu hỏi (compromised, revoke, MFA) lên trước boilerplate; (2) generator chỉ lấy chunk có coverage cao nhất với câu hỏi thay vì nối cả 3. Đo lại bằng Context Precision và Faithfulness của M06, kỳ vọng Precision từ 0.756 lên ~1.0, Faithfulness tăng theo vì bớt noise đầu câu.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Generator nối máy móc 3 chunks, không lọc noise, không tổng hợp súc tích | E01, E02, E04, E05, M01, M02, M04, M05, M07, H01, H02, H04, H05, A02, A03 | High |
| 2 | Ranking kém: chunk đúng bị chôn dưới boilerplate/sai nhánh (kèm Precision báo ảo do ngưỡng thấp) | M06, H03, E03 | High |
| 3 | Từ chối ngoài-scope thiếu mẫu chuẩn + metric overlap không đo được refusal đúng | A01 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> chọn cluster 1. có  15/19 fails nên chỉ cần sửa một chỗ trong prompt bắt tổng hợp súc tích + chỉ giữ chunk liên quan, bỏ boilerplate là Faithfulness và Relevance của cả loạt cùng nhích lên, thay vì sửa từng câu. 

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Clarify system prompt with intent detection and question restatement | Open |
| F003 | irrelevant | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F004 | hallucination | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F005 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F007 | hallucination | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F008 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F009 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F011 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F012 | off_topic | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F013 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F014 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F015 | off_topic | Context is missing or irrelevant — improve retrieval | Strengthen scope guardrails to redirect off-topic questions | Open |
| F016 | irrelevant | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F017 | hallucination | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F018 | hallucination | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
| F019 | off_topic | Answer does not address the question — improve prompt clarity | Strengthen scope guardrails to redirect off-topic questions | Open |
```

**Ba improvement suggestions ưu tiên**

1. Viết lại prompt generator: tổng hợp súc tích, giữ nguyên dates/amounts/điều kiện, cấm nối nguyên văn nhiều chunk
2. Rerank kết quả BM25 theo overlap với câu hỏi và phạt boilerplate scope khi câu hỏi rõ ràng trong-scope
3. Thêm refusal template bám `00_system_scope.md` cho nhánh 0-chunk, kèm kiểm tra citation cho mọi claim số liệu

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Prompt tổng hợp súc tích + lọc noise | Faithfulness (0.364 → mục tiêu ≥ 0.6), Relevance (0.494 → ≥ 0.6) | Chạy lại `evaluate_answers.py` trên cùng 20 actual mới, so sánh trung bình và đếm case pass |
| Rerank + phạt boilerplate | Context Precision (0.901 → ≥ 0.95), Faithfulness nhóm M06/H03/E03 | Đo Precision trước/sau rerank trên cùng tập chunks (giữ nguyên union để Recall không đổi, đúng tinh thần Exercise 3.5) |
| Refusal template + citation check | Completeness/Relevance nhóm adversarial (LLM-judge ngữ nghĩa, không phải overlap) + giảm hallucination (6 → ≤ 2) | Chấm lại A01–A03 bằng rubric Exercise 3.3 và human spot-check 3 case |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

>  sửa prompt/generator, đổi model, chỉnh retriever (top_k, chunking, rerank), cập nhật corpus/policy, và bắt buộc một lần ngay trước deploy. 

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> không. Support là lĩnh vực sai một con số  là khách thiệt hại thật, nên cần cchuaanr

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> Block khi Faithfulness rớt quá 0.05, xuất hiện thêm case hallucination mới, hoặc bất kỳ case safety/privacy nào như lộ password, hứa refund, làm theo injection trả lời sai . chỉ alert khi: Relevance/Completeness rớt nhẹ, Precision dao động do thêm document mới, hoặc Overall giảm vì 1–2 case Hard biên. Nguyên tắc của mình: cái gì gây hại thì chặn, cái gì chỉ làm câu trả lời kém hay thì theo dõi.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [offline benchmark 20 golden QA + run_regression vs baseline] → [LLM-judge + human spot-check các case fail/mới] → [canary online theo dõi metric + khiếu nại] → Deploy
```

>  vòng đầu chạy máy cho nhanh, chặn ngay nếu rớt quá 0.05. Vòng hai dùng judge và người kiểm tra tay những case máy chấm không tin được . Vòng ba thả nhỏ ra production để xem hành vi thật  rồi mới mở hết => không bỏ vòng nào vì mỗi vòng bắt một loại lỗi khác nhau.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Viết lại prompt: tổng hợp súc tích, giữ dates/amounts/exceptions, cấm nối chunks | Faithfulness 0.364 → ≥ 0.6, Relevance 0.494 → ≥ 0.6 | Pass rate nhích từ 5% lên đa số Easy/Medium pass |
| 2 | Rerank + phạt boilerplate scope | Precision 0.901 → ≥ 0.95, cứu M06/H03/E03 | Chunk đúng lên top, generator bớt noise đầu câu |
| 3 | Refusal template + citation check + gọi LLM thật thay fallback | Nhóm adversarial đúng hành vi, hallucination 6 → ≤ 2 | Hết lỗi từ chối chung chung, số liệu phản ánh đúng hệ RAG thật |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> thêm 3 case: (1) một câu ngoài-scope mới để kiểm tra refusal template có bám scope thật không, vì hiện chỉ có A01 là case 0-chunk duy nhất; (2) một case Hard kiểu H01 nhưng đổi mốc version để xem retriever có còn nhầm order-date/delivery-date không; (3) một case "bẫy ranking" kiểu M06 để đo rerank có tiến bộ không.
---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

>  retriever sẽ là điểm yếu, kết quả ngược lại: Recall 0.902, gần như lấy đủ hết.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

>  đếm từ chung chứ không hiểu nghĩ, đưa vào production thì đo ngữ nghĩa bằng embedding thay vì trùng từ, và human spot-check định kỳ để hiệu chỉnh judge. Không bỏ overlap hẳn vì nó rẻ và xác định, nhưng không để nó làm trọng tài duy nhất cho những quyết định như chặn deploy.
