# TODO — Lab Self-evolving Agentic (Deep Agents)

> Quy tắc xuyên suốt: KHÔNG mở `tasks/*/check.py`, `tasks/*-eval/`, `results/*/…-eval/`; KHÔNG sửa tay `skills/auto/`; KHÔNG sửa `tests/`, `tasks/`, `scripts/`, file có sẵn; KHÔNG commit `.env`. Ngân sách gợi ý ≤ 30 lần chạy task.

## Phần 0 — Cài đặt & làm quen
- [ ] `python3 -m venv .venv && source .venv/bin/activate && pip install -e .`
- [ ] `cp .env.example .env` → điền `LAB_MODEL` + API key (model có tool calling, giá rẻ)
- [ ] `mkdir -p report && cp REPORT_TEMPLATE.md report/REPORT.md`
- [ ] `pytest tests/test_01_provided.py` → `15 passed`
- [ ] Kiểm tra model: `python -c "from lab.model import make_model; print(make_model().invoke('Reply with OK').content)"`
- [ ] `python scripts/tour.py` → trả lời 3 câu hỏi vào **mục 3** REPORT.md

## Phần 1 — Cài đặt harness (30đ, chấm tự động)
- [ ] 1.1 `src/lab/subagents.py::get_subagents` (≥2 subagent, vai trò khác nhau, `description` nêu khi nào gọi) — đọc `guides/pseudocode/02_subagents.md` → `pytest tests/test_02_agent.py -k subagents`
- [ ] 1.2 `src/lab/agent.py`: TODO 1 imports, `make_backend` (đặt `PATH`, không lộ key), `build_agent` — `01_agent.md` → `pytest tests/test_02_agent.py` (9 test)
- [ ] 1.3 `src/lab/runner.py::run_task` — `03_runner.md` → `pytest tests/test_03_runner.py` (6 test)
- [ ] Chạy thật: `python -m lab.runner --condition baseline --tasks data-learn` → kiểm tra `run.json`, `trace.md`, `tokens.total > 0`

## Phần 2 — Chạy tác vụ học & phân loại lỗi
- [ ] `python -m lab.runner --condition baseline --tasks code-learn logs-learn`
- [ ] `python -m lab.runner --condition subagents --tasks learn`
- [ ] Đọc `run.json`/`trace.md` của 3 task học → bảng phân loại lỗi A–G (≥4 check thất bại, có bằng chứng) → **mục 4**
- [ ] `python scripts/check_breakdown.py` → bằng chứng phủ định cho nhóm A–D nếu lỗi chủ yếu là E
- [ ] Phân tích subagents (`subagent_calls`, nội dung giao việc, token so với baseline) → **mục 5**

## Phần 3 — Curator tự viết skill
- [ ] Cài `src/lab/curator.py::curate_skills` — `04_curator.md`, `05_skill_quality.md` → `pytest tests/test_04_curator.py`
- [ ] `python -m lab.curator` → có ≥1 skill hợp lệ trong `skills/auto/`
- [ ] Đánh giá từng skill (tổng quát? đúng? độ dài & `description`?) → **mục 6**; nếu xóa/chạy lại curator (tối đa 2 lần) thì ghi lý do
- [ ] `python -m lab.runner --condition skills-auto --tasks learn` → xem `skills_read`, trace
- [ ] Sao lưu: `mv results/skills-auto results/skills-auto-dev`

## Phần 4 — Giả thuyết, đóng băng, đo lại
- [ ] Viết H1–H3 vào **mục 2** REPORT.md (chưa xem bất kỳ kết quả eval nào)
- [ ] `git add -A && git commit -m "hypotheses"`
- [ ] `git add -A && git commit --allow-empty -m "freeze skills" && git tag freeze` — từ đây không đụng `skills/auto/`
- [ ] `python -m lab.runner --condition baseline --tasks eval`
- [ ] `python -m lab.runner --condition subagents --tasks eval`
- [ ] `python -m lab.runner --condition skills-auto --tasks all`
- [ ] `python scripts/verify_freeze.py` → `OK`
- [ ] `python -m lab.compare > report/table.md`
- [ ] `python scripts/check_breakdown.py` → số liệu cho mục 4, 7, 8

## Phần 5 — Báo cáo (`report/REPORT.md`)
- [ ] Mục 1–7 (dán bảng `table.md` vào mục 7)
- [ ] Mục 8: phân tích — tách learn/eval, tách check kỹ thuật vs `rule_`, `skills_read` + trace, chi phí token, quá khớp/rò rỉ, nhiễu (skills-auto-dev vs sau freeze)
- [ ] Mục 9: ≥3 hạn chế + ảnh hưởng đến kết luận
- [ ] Mục 10 + ghi rõ model, tham số, lệnh chạy, commit

## Phần 6 — Mở rộng (tùy chọn, +5)
- [ ] Chọn 1 hướng (6a–6e), kết quả ở thư mục `--results` riêng, ghi vào phụ lục

## Trước khi nộp
- [ ] `pytest` toàn bộ đạt
- [ ] `results/{baseline,subagents,skills-auto}/` đủ 6 task (run.json + trace.md)
- [ ] Rà `trace.md`/báo cáo: không có API key, không lộ thông tin cá nhân
- [ ] `git status` sạch, `.env` không bị commit; push kho
