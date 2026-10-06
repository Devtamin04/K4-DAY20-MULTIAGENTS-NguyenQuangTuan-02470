---
name: enforce-code-quality-and-documents
description: Use when finalising a code package to guarantee that type‑hints, regression tests, and changelog entries obey the project conventions.
---
1. **Identify public functions** – any function whose name does **not** start with an underscore (`_`).  
2. **Add type hints** – for each public function, annotate every parameter and the return type using standard Python typing syntax.  
3. **Run a type‑checker** – execute `mypy` (or an equivalent) on the package; ensure it reports zero errors.  
4. **Create regression test file** – if it does not exist, add `tests/test_regressions.py`.  
5. **Write one test per bug fix** – each test function must be named `test_<bug_description>` and assert the corrected behaviour; include at least three such tests.  
6. **Execute the test suite** – run `pytest`; the suite must pass with zero failures.  
7. **Update CHANGELOG.md** – locate the heading `## Unreleased`. Under it, add a bullet for every fixed public function in the exact form:  
   `- fix(<function name>): <short description of the fix>`  
   Ensure there are at least three bullets matching the bugs you addressed.  
8. **Verify changelog formatting** – no duplicate headings, bullets start with a hyphen and a space, and the file ends with a newline.  
9. **Self‑check** – confirm that (a) all public functions have type hints, (b) `tests/test_regressions.py` exists with ≥ 3 passing tests, and (c) the “Unreleased” section of `CHANGELOG.md` contains the required bullets.
