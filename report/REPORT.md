# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên:
- Mã sinh viên:

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

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
(dán bảng ở đây)
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Bạn đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

> Nêu ít nhất 3 hạn chế và ảnh hưởng của từng hạn chế đến kết luận (ví dụ: chỉ 3 tác vụ mỗi vai trò, mỗi cấu hình chạy một lần, nhiễu của mô hình, tác vụ do giảng viên thiết kế sẵn quy ước, chỉ một mô hình).

1.
2.
3.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
