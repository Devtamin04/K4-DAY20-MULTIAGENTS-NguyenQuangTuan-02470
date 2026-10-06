---
name: ensure-codebase-conventions
description: Use when finalizing any code change to guarantee repository‑wide conventions and testability.
---
1. **Run the full test suite** (`pytest -q`).  
   - Ensure *all* tests pass, including the visible suite and any hidden tests.  
   - If failures appear, fix the code before proceeding.

2. **Check type hints for public functions**:  
   - Open every module in the package.  
   - For each function whose name does **not** start with an underscore, verify that **all parameters** and the **return value** have explicit type annotations.  
   - Add missing annotations or refactor the function signature.

3. **Add regression tests**:  
   - Create or update `tests/test_regressions.py`.  
   - For each bug you have fixed, write **one** test that would fail before the fix and pass after.  
   - Ensure the file contains **at least three** such tests and that the file itself passes `pytest`.

4. **Update the changelog**:  
   - Open `CHANGELOG.md`.  
   - Under the heading `## Unreleased`, add a bullet for **each** fix in the exact format:  
     `- fix(<function name>): <short description>`  
   - Include **at least three** bullets if you fixed three or more issues.

5. **Verify importability for the test runner**:  
   - The package root (`inventory/` or equivalent) must expose the implementation without requiring manual `sys.path` manipulation.  
   - If you use a namespace package, ensure `__path__` is correctly extended in `__init__.py` and that the implementation directory exists.  
   - Run `python -c "import inventory"` from the repository root to confirm no `ImportError`.

6. **Optional static checks** (if tools are available):  
   - Run `mypy` or similar to confirm type‑hint compliance.  
   - Run a linter (e.g., `ruff`) to catch obvious style issues.

7. **Self‑check before committing**:  
   - All public functions are typed.  
   - Regression test file exists with ≥3 tests and passes.  
   - `CHANGELOG.md` has the required bullets under `## Unreleased`.  
   - `pytest` reports 0 failures.  
   - The package imports cleanly from the repository root.
