# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Nguyễn Quang Tuấn
- Mã sinh viên: 2A202602470

- Nhà cung cấp và mô hình: Ollama Cloud, endpoint tương thích OpenAI (`LAB_BASE_URL=https://ollama.com/v1`), `LAB_MODEL=gpt-oss:120b`; `LAB_TEMPERATURE=0`; `recursion_limit=60` (mặc định của `lab.runner`) cho mọi lần chạy chính thức.
- Deep Agents 0.7.21, Python 3.12, Linux; chạy trực tiếp (không Docker), sandbox là thư mục tạm.
- Số lần chạy tác vụ đã dùng / ngân sách: xem Phụ lục (kể cả các lần chạy hỏng do hạ tầng, được lưu trong `results/pre-fix/`).
- Commit của tag `freeze`: xem `git rev-parse freeze` (điền ở mục Phụ lục sau khi tạo tag).
- **Điều chỉnh harness bắt buộc với mô hình này** (áp dụng giống nhau cho cả 3 điều kiện, ghi trong `src/lab/agent.py`, `src/lab/runner.py`):
  1. gpt-oss được huấn luyện với một công cụ shell dựng sẵn tên `exec` (`{"cmd": ["bash","-lc", ...]}`) và vẫn gọi nó dù chỉ có `execute`. Endpoint Ollama trả **HTTP 500** cho lời gọi công cụ lạ này (tái hiện 0/12 lần thành công ở mọi nhiệt độ, cả trên API gốc `/api/chat`; thêm công cụ `exec` thì 2/2 thành công). Harness thêm công cụ `exec` là bí danh của `execute` trên cùng backend.
  2. `ModelRetryMiddleware` (5 lần, backoff 5-60 s) chỉ cho lỗi 5xx/429/mạng, để lỗi hạ tầng tạm thời không bị tính là lỗi của tác tử.
  3. `run_task` dùng `agent.stream(stream_mode="values")` (mở rộng gợi ý trong `03_runner.md`) để vẫn có `trace.md` và `tool_calls` khi gặp `GraphRecursionError`.

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

Viết trước khi có bất kỳ điểm nào của tác vụ đánh giá; chỉ dựa trên tác vụ học (mục 4-6).

- H1 (subagents so với baseline): `subagents` **không** cao hơn `baseline` trên tác vụ đánh giá (chênh lệch điểm trung bình trong khoảng ±0,05) nhưng tốn token hơn khoảng 20-50%. Căn cứ: trên tác vụ học hai điều kiện có điểm giống hệt nhau (code 1/10, data 5/8, logs 6/9), tác tử chính chỉ giao việc 1/3 lần, và lời giao việc duy nhất không chứa quy tắc nào của đề; các lỗi chiếm đa số là quy ước ẩn (nhóm E), mà subagent cũng không biết. Token trung bình 98k so với 76k (+30%). Bài của Anthropic về hệ đa tác tử ghi nhận chi phí token tăng mạnh, còn lợi ích chỉ xuất hiện ở việc song song hóa được.
- H2 (skills-auto so với baseline): `skills-auto` đạt điểm trung bình cao nhất trên tác vụ đánh giá, nhưng lợi ích tập trung ở họ `logs` (các check `rule_` về tên service, thứ tự sắp xếp, header schema nếu tác vụ đánh giá dùng lại quy ước) và có thể ở `code` nếu tác tử đọc skill và còn đủ bước; **không** cải thiện họ `data` vì curator không sinh được skill hợp lệ cho họ này. Check quy ước **mới** của tác vụ đánh giá sẽ thất bại ở cả 3 điều kiện. Dự đoán chênh lệch trung bình +0,05 đến +0,15. Căn cứ: Phần 3.4 cho logs-learn 9/9 so với 6/9 (`skills_read`=1); SkillsBench ghi nhận skill tự sinh trung bình không có lợi, nhưng ở đây phản hồi `RULE:` phát biểu tường minh quy ước nên skill mang đúng thông tin còn thiếu.
- H3 (tác vụ học so với tác vụ đánh giá): mức cải thiện của `skills-auto` trên tác vụ học lớn hơn trên tác vụ đánh giá (khoảng cách tổng quát hóa), vì skill log còn lặp lại khá sát đề bài học và vì tác vụ đánh giá có thêm một quy ước không học được (SkillEvolBench: lợi ích trên tác vụ học thường không chuyển sang tác vụ mới). Họ `code` có nhiễu lớn (chạm `recursion_limit`, phản hồi rỗng của mô hình) nên mọi chênh lệch ở đó dưới 0,2 coi như nhiễu.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: nhóm tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; shell `execute`; subagent `task`. Chỉ `execute` cho phép chạy lệnh (lệnh shell thật, thư mục làm việc là gốc sandbox).
2. Mô tả của `task` giới thiệu `general-purpose` là tác tử đa dụng để nghiên cứu câu hỏi phức tạp, tìm tệp/nội dung và thực thi tác vụ nhiều bước, "has access to all tools as the main agent". Về ngữ cảnh: "Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report", tức subagent **không** thấy lịch sử hội thoại của tác tử chính, chỉ thấy lời giao việc.
3. System prompt mặc định là `''`. Câu hướng dẫn hành vi trong mô tả `task`: "Put full detail in the prompt and state exactly what it should return". Câu trong mô tả `execute`: "You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail."

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Nguồn: `results/baseline/<tác vụ học>/run.json` và `trace.md` (lần chạy với harness cuối cùng, `recursion_limit=60`). 15 check thất bại.

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| code-learn | `visible_suite_passes` | G (chạy test sai thư mục) + C | Tác tử chạy `pytest -q` ở gốc sandbox -> `ModuleNotFoundError: No module named 'inventory'`; thay vì chạy trong `workspace/`, nó tạo gói giả `inventory/__init__.py` ở gốc để nối `__path__` (vá triệu chứng), tốn ~20 bước. `detail`: "2 failed, 4 passed". |
| code-learn | `parse_price_all_formats` | G (hết `recursion_limit`) | `error`: `GraphRecursionError: Recursion limit of 60`; vết không có `edit_file` nào lên `workspace/inventory/pricing.py`. `detail`: "wrong for: ['$1,299.50', '(12.00)', ...]". |
| code-learn | `other_caller_fixed` | G (hết `recursion_limit`) | Như trên: hàm dùng chung `parse_price` chưa sửa nên `to_csv_row` vẫn lỗi (`'<InvalidOperation>'`). |
| code-learn | `discount_rounds_half_up` | G (hết `recursion_limit`) | Không có chỉnh sửa `apply_discount`; `detail`: "wrong for: [('10.05', 10, '9.05'), ...]". |
| code-learn | `low_stock_follows_docstring` | G (hết `recursion_limit`) | `detail`: "low_stock returned ['b', 'A', 'c']" (chưa sắp xếp theo docstring). |
| code-learn | `csv_quoting_follows_docstring` | A (bỏ qua đặc tả trong docstring) | `detail`: "to_csv_row returned 'Desk, large \"oak\",10.00,2'". Ngay cả lần chạy thử với `recursion_limit=100` (`results/pre-fix/limit100/`, 6/10) đã sửa mọi lỗi khác nhưng vẫn trượt check này: docstring ghi rõ RFC 4180 nhưng không có test hiển thị. |
| code-learn | `rule_type_hints` | E | `detail`: "RULE: every public function ... has type annotations on all parameters and on the return value." |
| code-learn | `rule_regression_tests` | E | `detail`: "RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3)". |
| code-learn | `rule_changelog` | E | `detail`: "RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' ...". Vết: tác tử không mở `workspace/README.md` lẫn `workspace/CHANGELOG.md`. |
| data-learn | `rule_money_in_cents` | E | `detail`: "RULE: money values in answer.json are integer cents (1606.67 USD is written 160667)." |
| data-learn | `rule_meta_block` | E | `detail`: "RULE: answer.json has an object `meta` = {source, rows_in, rows_used}". |
| data-learn | `rule_clean_csv` | E | `detail`: "RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents ...". |
| logs-learn | `rule_service_names` | E | `detail`: "RULE: service names in the output are lower-case with '-' replaced by '_'". |
| logs-learn | `rule_sorted_errors` | E | `detail`: "RULE: `errors` is sorted by service, then by timestamp_utc, ascending." |
| logs-learn | `rule_schema_header` | E | `detail`: "RULE: the top-level object has \"schema_version\": 2 and \"generated_by\": \"log-triage\"." |

Nhận xét:

- **Nhóm E chiếm đa số: 9/15 check thất bại** (100% check `rule_` thất bại: 0/9 đạt). Nguyên nhân chung: quy ước Acme không có trong đề và không có trong workspace; tác tử không thể tự suy ra. Đây đúng là loại lỗi mà skill (tri thức thủ tục được ghi lại từ phản hồi) có thể phòng ngừa.
- **Bằng chứng phủ định cho A-D ở data và logs** (`python scripts/check_breakdown.py`): check kỹ thuật baseline học đạt **12/18**, trong đó data 5/5 và logs 6/6. Tác tử đã đọc README trước khi làm (vết: `read_file workspace/README.md` ở cả hai), xử lý đúng trùng lặp, giá trị thiếu `-999`, nhiều định dạng ngày, múi giờ, stack trace nhiều dòng và dòng lặp, nên không có lỗi nhóm A, B, D ở hai họ này. Không có lỗi F: câu trả lời cuối chỉ nhắc tệp thật sự đã ghi.
- **Họ code là ngoại lệ:** 5/7 check kỹ thuật thất bại nhưng nguyên nhân gốc là **G: chạy test sai thư mục rồi vá import (C) đến khi hết `recursion_limit`**, không phải không hiểu lỗi. Lỗi A (`csv_quoting`) là lỗi kỹ thuật "thật" duy nhất còn lại khi có đủ bước. Skill có thể giúp nếu nói "chạy test trong thư mục của gói", nhưng curator không rút ra quy tắc này (xem mục 6).

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa (`src/lab/subagents.py`): `explorer` (chỉ đọc README, CHANGELOG, docstring, mẫu dữ liệu; báo cáo quy ước và dữ liệu bẩn; nhắm vào nhóm lỗi A/D), `implementer` (sửa nguyên nhân gốc, viết script, chạy test; nhắm vào nhóm C), `reviewer` (kiểm tra độc lập từng yêu cầu, không sửa; nhắm vào nhóm B/F). `description` của mỗi subagent nêu **khi nào** gọi (FIRST / to make the change / LAST before replying) và nhắc tác tử chính gửi kèm toàn bộ quy tắc.
- `subagent_calls`: code-learn **1** (`implementer`), data-learn **0**, logs-learn **0**. Với data và logs, tác tử chính (gpt-oss) đọc README và dữ liệu rồi tự viết một script trong 5-11 lời gọi công cụ; tác vụ đủ nhỏ để làm một mạch, nên mô hình bỏ qua lời khuyến khích của `SUBAGENTS_NOTE`. Đây là kết quả hợp lệ: quyết định giao việc thuộc về tác tử chính, và việc giao một tác vụ ~10 bước không có lợi về ngữ cảnh.
- Thông tin khi giao việc: lời giao việc duy nhất (code-learn) là *"Create a top-level inventory package that points to the existing workspace/inventory implementation so that imports like 'import inventory.export' work ... No other changes needed."* Nó **không chứa quy tắc nào của đề** (docstring là đặc tả, không sửa `tests/`) dù `SUBAGENTS_NOTE` yêu cầu, và giao đúng cái "mẹo" vá triệu chứng ở mục 4. Subagent báo cáo "Implemented a top-level `inventory` package ..."; tác tử chính có kiểm tra lại bằng `pytest -q` nhưng vẫn không nhận ra nên chạy test trong `workspace/`. Kết quả: subagent khuếch đại lỗi thay vì sửa nó; `explorer` và `reviewer` không bao giờ được gọi.
- Ảnh hưởng đến token và thời gian (tác vụ học): code 161.856 -> 192.005 (+19%), data 42.227 -> 62.192 (+47%), logs 22.791 -> 40.346 (+77%); trung bình 75.624 -> 98.181 (+30%). Ngay cả khi không giao việc, token vẫn tăng vì system prompt dài hơn (mô tả công cụ `task` liệt kê 4 subagent + `SUBAGENTS_NOTE`) và tác tử làm thêm bước. Điểm không đổi (12/18 kỹ thuật, 0/9 quy ước ở cả hai).

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator: **3** (1 lần + 2 lần chạy lại, đúng giới hạn). Mọi đầu ra được lưu nguyên trạng trong `results/curator-attempts/run{1,2,3}/`; không sửa tay nội dung skill nào.
  - Lần 1: 2 skill hợp lệ (`code-quality-and-change-management`, `log-triage-output-conventions`); skill thứ ba `structured-data-output` bị `validate_skill` từ chối (*mentions evaluation material: orders*). **Xóa cả hai và chạy lại** vì (a) họ `data` không có skill, (b) skill log có giá trị viết bằng dấu gạch Unicode U+2011 (`"log‑triage"` ở bước tự kiểm tra, `YYYY‑MM‑DD`), tác tử chép nguyên sẽ ra giá trị sai.
  - Lần 2: 2 skill hợp lệ, ngắn hơn, giá trị `"log-triage"` dùng ASCII; skill data `produce-correct-data-output` lại bị từ chối vì `orders`. Trước lần 3, prompt curator được bổ sung hai quy tắc tổng quát (không dùng danh từ nghiệp vụ của dữ liệu, chỉ dùng ASCII); prompt **không** nêu định danh nào của tác vụ đánh giá.
  - Lần 3: skill data vẫn bị từ chối (`orders`); hai skill còn lại **kém hơn lần 2**: skill code có bước 5 "expose `inventory/` at the repository root ... extend `__path__`" (chép lại đúng mẹo sai trong vết baseline - có hại, và nêu tên gói của tác vụ học), skill log dài 71 dòng và nêu tên tệp `app.log` của tác vụ học (quá khớp). **Giữ bộ skill của lần 2** (khôi phục nguyên văn từ `results/curator-attempts/run2/`).
  - Nhận xét: bộ lọc rò rỉ dùng chuỗi con nên `orders` (tên tệp của tác vụ đánh giá họ data) chặn mọi skill nói về "orders" - một từ không thể tránh khi phản hồi học là về `order_id`. Đây là dương tính giả của bộ lọc, nhưng nó đúng là đang làm việc của nó: kết quả là họ `data` không có skill.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `enforce-code-quality-and-documents` | Tổng quát cho mọi gói Python: không nêu tên hàm, tệp hay gói của tác vụ; chỉ nêu tên do quy ước Acme yêu cầu (`tests/test_regressions.py`, `CHANGELOG.md`, `## Unreleased`, `- fix(<function name>): ...`). | Đúng với cả 3 `RULE:` của code-learn (type hint cho **mọi** hàm public, >= 3 test hồi quy, gạch đầu dòng changelog). Chỗ yếu: bước 3 bắt chạy `mypy` (không có trong sandbox, tốn bước); bước 5 thêm ràng buộc đặt tên `test_<bug_description>` không ai yêu cầu (vô hại). Không nói gì về nguyên nhân gốc của thất bại code (chạy test sai thư mục). | 15 dòng, 9 bước, có tự kiểm tra. `description` "Use when finalising a code package ..." mô tả thời điểm *kết thúc*, trong khi `SKILLS_NOTE` bắt đọc skill ở bước đầu; đủ rộng nên **data-learn đọc nhầm** skill này (`skills_read`=1, không liên quan). Ở code-learn `skills_read`=0: tác tử dừng ở bước 8 với phản hồi rỗng của mô hình trước khi kịp đọc skill. |
| `generate-correct-log-triage-json` | Nửa tổng quát: bước 11-13 là quy ước Acme (sắp xếp theo service rồi thời gian, `schema_version`/`generated_by`, tên service chữ thường + `_`), áp dụng được cho mọi tệp log; bước 1-10 phần lớn **lặp lại đề bài học** (định dạng repeat line, cách lấy exception) - dấu hiệu quá khớp vào định dạng log cụ thể. | Đúng với cả 3 `RULE:` của logs-learn. Rủi ro nhỏ: `YYYY‑MM‑DDTHH:MM:SSZ` viết bằng dấu gạch U+2011 (đề bài đã ghi đúng định dạng nên ít ảnh hưởng). | 23 dòng, 15 bước, có tự kiểm tra. `description` nêu đúng tình huống ("converting a raw log file into the structured `errors.json` ... Acme log-triage conventions"). logs-learn: `skills_read`=1, vết cho thấy làm theo đủ 3 quy tắc -> **9/9** (baseline 6/9). |

## 7. Kết quả so sánh (Phần 4.3, 4.4)

`python -m lab.compare` (sau tag `freeze`; `python scripts/verify_freeze.py` -> `checked 6 runs of skill conditions: OK`):

| Task | baseline | subagents | skills-auto |
|---|---|---|---|
| code-learn | 1/10 | 1/10 | 5/10 |
| data-learn | 5/8 | 5/8 | 5/8 |
| logs-learn | 6/9 | 6/9 | 9/9 |
| code-eval | 7/11 | 6/11 | 7/11 |
| data-eval | 5/9 | 5/9 | 5/9 |
| logs-eval | 6/10 | 6/10 | 9/10 |
| **Mean score - learning tasks** | 0.46 | 0.46 | 0.71 |
| **Mean score - evaluation tasks** | 0.60 | 0.57 | 0.70 |
| **Mean tokens per run** | 101,713 | 96,054 | 119,982 |
| **Runs that read a skill** | 0/6 | 0/6 | 4/6 |

`python scripts/check_breakdown.py`:

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      eval     18/18         0/12         127,803      0/3
baseline      learn    12/18         0/9           75,624      0/3
subagents     eval     17/18         0/12          93,928      0/3
subagents     learn    12/18         0/9           98,181      0/3
skills-auto   eval     18/18         3/12          95,984      2/3
skills-auto   learn    16/18         3/9          143,980      2/3
```

Lần chạy có `error`: mọi lần chạy tác vụ `code` (code-learn ở 3 điều kiện, code-eval ở 3 điều kiện) kết thúc bằng `GraphRecursionError` ở `recursion_limit=60`; tác vụ vẫn được chấm trên workspace hiện có (đúng quy định của `run_task`) và **không chạy lại**, vì đây là hành vi của tác tử (hết ngân sách bước), không phải lỗi hạ tầng. Không lần chạy nào có `skills_modified = true`. Các lần chạy hỏng do HTTP 500 trước khi sửa harness được lưu trong `results/pre-fix/` và không dùng trong bảng.

Chi tiết check thất bại trên tác vụ đánh giá:

| Tác vụ | baseline | subagents | skills-auto |
|---|---|---|---|
| code-eval | 4 `rule_` (type_hints, regression_tests, changelog, **version_bump**) | `add_slot_no_shared_state` + 4 `rule_` | 4 `rule_` như baseline (`skills_read`=0) |
| data-eval | 4 `rule_` (money_in_cents, meta_block, clean_csv, **sorted_keys_format**) | như baseline | như baseline (`skills_read`=2 nhưng không có skill data) |
| logs-eval | 4 `rule_` (service_names, sorted_errors, schema_header, **source_line**) | như baseline | chỉ **`rule_source_line`** (`skills_read`=1) |

(**in đậm**: quy ước mới chỉ có ở tác vụ đánh giá.)

## 8. Phân tích

1. **Học và đánh giá.** So với `baseline`, chỉ `skills-auto` cải thiện: tác vụ học 0,46 -> 0,71 (+0,25), tác vụ đánh giá 0,60 -> 0,70 (+0,10). `subagents` không cải thiện ở đâu (học 0,46 = 0,46; đánh giá 0,57 < 0,60, do một check kỹ thuật `add_slot_no_shared_state`). Mức cải thiện trên tác vụ học lớn gấp 2,5 lần trên tác vụ đánh giá, nhưng **phần lớn chênh lệch +0,25 ở tác vụ học là nhiễu** của họ code (code-learn 1/10 -> 5/10 mà `skills_read`=0, xem câu 6). Nếu chỉ tính phần do skill gây ra (họ logs), lợi ích trên học (+3 check /9) và đánh giá (+3 check /10) gần như bằng nhau: skill log **chuyển giao được** sang tác vụ mới cùng họ. Không có điều kiện nào "chỉ cải thiện tác vụ học", nên không thấy dấu hiệu quá khớp rõ ràng trong số liệu (dù nội dung skill log có dấu hiệu quá khớp, mục 6).
2. **Kỹ thuật và quy ước.** Check kỹ thuật trên tác vụ đánh giá gần như bão hòa ở mọi điều kiện (18/18, 17/18, 18/18), nên skill **không giúp được** nhóm này và không có chỗ để giúp. Toàn bộ lợi ích của `skills-auto` nằm ở check `rule_`: 0/12 -> 3/12 trên đánh giá, 0/9 -> 3/9 trên học - đều là 3 quy ước log (`rule_service_names`, `rule_sorted_errors`, `rule_schema_header`). Ba quy ước **mới** (`rule_version_bump`, `rule_sorted_keys_format`, `rule_source_line`) thất bại ở **cả 3 điều kiện** (0/9): skill chỉ chứa những gì curator thấy trong phản hồi học, nên không thể biết quy ước chưa từng xuất hiện. Đây là giới hạn cơ bản của tiến hóa ở tầng ngữ cảnh từ phản hồi quá khứ.
3. **Cơ chế (vết + `skills_read`).**
   - *Skill giúp:* logs-eval, `rule_service_names` / `rule_sorted_errors` / `rule_schema_header`. Vết `results/skills-auto/logs-eval/trace.md`: tác tử đọc `skills/generate-correct-log-triage-json/SKILL.md`, sau đó script sinh ra viết tên service chữ thường + `_`, sắp xếp theo `(service, timestamp_utc)`, thêm `"schema_version": 2, "generated_by": "log-triage"`. Kết quả 9/10 so với 6/10 của baseline, dù định dạng log của tác vụ đánh giá khác (mức `SEVERE`/`FATAL`, dấu ` | `), tức phần quy ước của skill tổng quát hóa, còn phần lặp lại đề bài học thì tác tử bỏ qua và làm theo đề mới.
   - *Skill không giúp (chưa được đọc):* code-eval, `rule_type_hints` / `rule_regression_tests` / `rule_changelog`. Skill `enforce-code-quality-and-documents` chứa đúng 3 quy tắc này, nhưng `skills_read`=0 ở cả code-learn và code-eval: hành động đầu tiên trong vết là `exec ls -R` rồi `pytest -q`, bỏ qua chỉ dẫn "As your FIRST action, read the SKILL.md" của `SKILLS_NOTE`; `description` "Use when **finalising** a code package" chỉ khớp với thời điểm kết thúc, mà tác tử hết `recursion_limit` trước khi tới đó.
   - *Skill được đọc nhưng sai chỗ:* ở data-learn và data-eval tác tử đọc **cả 2 skill** (code và log, `skills_read`=2), không skill nào liên quan đến data -> chỉ tốn token (data-learn 215.270 token so với 42.227 ở baseline, nhiều bước hơn), điểm không đổi.
4. **Chi phí.** Token trung bình mỗi lần chạy: baseline 101.713, subagents 96.054, skills-auto 119.982 (+18% so với baseline). Theo điểm trên mỗi token (trung bình 6 tác vụ: baseline 0,53, subagents 0,51, skills-auto 0,705): baseline 5,2 điểm/triệu token, subagents 5,3, **skills-auto 5,9** - hiệu quả nhất dù đắt nhất, vì lợi ích +3 check/tác vụ log lớn hơn chi phí đọc skill. Trên riêng tác vụ đánh giá, skills-auto vừa rẻ hơn baseline (95.984 so với 127.803 - baseline logs-eval tốn 175.777 token cho 13 lần gọi) vừa điểm cao hơn; chênh lệch token này chủ yếu là nhiễu theo lần chạy. Đa tác tử **không đáng chi phí** ở thí nghiệm này: giao việc 2/6 lần, điểm không tăng (thấp hơn ở code-eval), token trên tác vụ học +30%; lời giao việc thiếu quy tắc (mục 5; ở logs-eval lời giao việc cũng không chứa quy ước Acme nào vì tác tử chính không biết chúng). Subagent chỉ có ích khi tác vụ đủ lớn để cần cô lập ngữ cảnh hoặc song song hóa, mà 6 tác vụ ở đây đều làm xong trong 5-30 lời gọi.
5. **Rò rỉ và quá khớp.** Không có rò rỉ: `validate_skill` chặn mọi skill chứa định danh của tác vụ đánh giá (3 lần chặn skill data vì chữ `orders`); curator chỉ đọc `run.json` có `role == "learn"` (test `test_04` xác nhận); không mở `check.py`, `tasks/*-eval/` hay kết quả đánh giá trước tag `freeze` (lịch sử git: commit `hypotheses` -> `freeze skills` -> chạy đánh giá). Có dấu hiệu **quá khớp ở nội dung skill**: skill log lặp lại định dạng của log học (bước 1-10), ở lần chạy curator 3 skill nêu tên tệp `app.log` và tên gói `inventory` của tác vụ học, và chép lại mẹo sai `__path__` từ vết baseline - tôi phòng tránh bằng cách đọc từng skill và loại bộ của lần 3. Lượng quá khớp này không làm hại điểm đánh giá vì tác tử ưu tiên đề bài mới khi hai bên mâu thuẫn.
6. **Nhiễu.** Cùng bộ skill đóng băng, tác vụ học ở Phần 3.4 (`results/skills-auto-dev/`) so với sau đóng băng: code-learn 1/10 -> 5/10 (+0,40; cả hai lần `skills_read`=0, lần đầu dừng ở bước 8 vì phản hồi rỗng của mô hình, lần sau chạy tới `recursion_limit`), data-learn 5/8 -> 5/8, logs-learn 9/9 -> 9/9. Một tác vụ dao động **0,4 điểm chỉ do nhiễu** ở `temperature=0`; trung bình 3 tác vụ học dao động 0,58 -> 0,71 (0,13). Do đó các chênh lệch ở họ code trong bảng (1/10 so với 5/10, 6/11 so với 7/11) **không có ý nghĩa**; chỉ chênh lệch ở họ logs (lặp lại nhất quán ở 4/4 lần chạy có đọc skill: logs-learn x2, logs-eval, và 0/3 quy ước ở mọi lần không đọc skill) đủ tin cậy để quy cho skill.

## 9. Hạn chế và tính hợp lệ

1. **Cỡ mẫu rất nhỏ và một lần chạy cho mỗi cấu hình.** 3 tác vụ đánh giá x 1 lần chạy; nhiễu đo được (0,4 điểm trên một tác vụ code) lớn hơn chênh lệch trung bình giữa các điều kiện (0,10). Kết luận về `subagents` và về họ code vì thế chỉ là "không phát hiện khác biệt", không phải "không có khác biệt"; chỉ kết luận về skill log được hỗ trợ bởi tín hiệu lặp lại.
2. **Một mô hình, với các điều chỉnh harness riêng.** Mọi số liệu từ `gpt-oss:120b` qua Ollama Cloud. Mô hình này cần bí danh công cụ `exec`, thỉnh thoảng trả phản hồi rỗng, và bỏ qua `SKILLS_NOTE` ở tác vụ code; mô hình khác (tuân thủ chỉ dẫn tốt hơn) có thể đọc skill code và đạt thêm 3 check, nên không khái quát hóa con số "+0,10" sang mô hình khác. Harness có thêm công cụ `exec` nên `baseline` của tôi không giống hệt `baseline` của sinh viên khác.
3. **`recursion_limit=60` làm méo họ code.** 6/6 lần chạy code chạm giới hạn; điểm code đo "đi được bao xa trong 60 bước" hơn là năng lực sửa lỗi (lần thử với 100 bước: 6/10). Skill code (đọc ở bước đầu) có thể có tác dụng nếu đủ bước; thí nghiệm không kiểm chứng được điều này.
4. **Tác vụ và quy ước do giảng viên thiết kế.** Phần lớn lỗi (nhóm E) là quy ước ẩn được phát biểu tường minh trong `detail`, nên skill tự sinh ở đây có lợi thế hơn bối cảnh thực (SkillsBench: skill tự sinh trung bình không có lợi). Ngược lại, quy ước mới của tác vụ đánh giá được thiết kế để không học được, nên trần điểm của `skills-auto` bị giới hạn sẵn.
5. **Bộ lọc rò rỉ dựa trên chuỗi con** loại bỏ skill data vì một từ thông dụng (`orders`), nên thí nghiệm chỉ kiểm tra được skill cho 2/3 họ; kết quả "skill không giúp data" là do không có skill, không phải do skill kém.

## 10. Kết luận

Trên tác vụ đánh giá, `skills-auto` đạt điểm trung bình cao nhất (0,70 so với 0,60 baseline và 0,57 subagents), và toàn bộ lợi ích đến từ 3 quy ước log mà skill do curator sinh ra đã ghi lại và được tác tử đọc, làm theo (H2 được ủng hộ một phần). Đa tác tử không cải thiện điểm và giao việc hiếm, với lời giao việc thiếu quy tắc (H1 được ủng hộ). Quy ước mới của tác vụ đánh giá thất bại ở mọi điều kiện, và chênh lệch ở họ code nằm trong mức nhiễu đo được (0,4 điểm/tác vụ), nên không kết luận gì về skill code. Đề xuất tiếp theo: viết `description` của skill theo tình huống *bắt đầu* tác vụ ("Use when fixing or changing a Python package") và chạy mỗi cấu hình ít nhất 3 lần (hướng 6e) để tách tác dụng thật khỏi nhiễu.

## Phụ lục

- Lệnh đã chạy (theo thứ tự, harness cuối cùng):
  1. `pytest` (32 passed); `python scripts/tour.py`
  2. `python -m lab.runner --condition baseline --tasks learn`; `python -m lab.runner --condition subagents --tasks learn`
  3. `python -m lab.curator` (3 lần, mục 6); `python -m lab.runner --condition skills-auto --tasks learn`; `mv results/skills-auto results/skills-auto-dev`
  4. `git commit -m "hypotheses"` (e5b08c7); `git commit --allow-empty -m "freeze skills" && git tag freeze` (tag `freeze` = dd5277b)
  5. `python -m lab.runner --condition baseline --tasks eval`; `... --condition subagents --tasks eval`; `... --condition skills-auto --tasks all`
  6. `python scripts/verify_freeze.py` (OK); `python -m lab.compare > report/table.md`; `python scripts/check_breakdown.py`
- Lần chạy ngoài bảng (`results/pre-fix/`): các lần chạy trước khi thêm bí danh `exec` (nhiều lần HTTP 500), và một lần `baseline code-learn` với `--recursion-limit 100` (6/10, 387.124 token) chỉ để tham khảo. Tổng cộng khoảng 38 lần chạy tác vụ (vượt gợi ý 30 do lỗi hạ tầng).
- Thử thách mở rộng: không thực hiện.
- Ghi chú khác: sau đóng băng không sửa `skills/`; không có lần chạy nào `skills_modified = true`.
