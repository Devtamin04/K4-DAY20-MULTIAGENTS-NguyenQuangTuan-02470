---
name: code-quality-and-change-management
description: Use when you modify package code so that type hints, regression tests, and changelog entries are correctly maintained.
---
1. Identify every public function (its name does **not** start with an underscore) that you added or edited.  
2. For each identified function, add **type annotations** for **all** parameters **and** the return value; import from `typing` as needed.  
3. Run the project's type‑checking command (e.g., `mypy` or the CI lint step) and confirm that no missing‑annotation warnings appear.  
4. Create or update `tests/test_regressions.py`:  
   a. Write one test function for each bug you fixed.  
   b. Ensure each test asserts the corrected behaviour.  
   c. Provide at least three distinct regression tests if you fixed three bugs.  
5. Execute the full test suite (`pytest -q`) and verify that **all** tests, including the new regression tests, pass.  
6. Open `CHANGELOG.md` and locate the `## Unreleased` heading (create it if it does not exist).  
7. Under that heading, add a bullet for every function you fixed, exactly in the form:  
   `- fix(<function name>): <short description>`  
8. Ensure there are at least three such bullets when you have fixed three bugs.  
9. Commit the changes together so that code, tests, and changelog stay in sync.  
10. **Self‑check**: every public function now has complete type hints, `tests/test_regressions.py` exists with ≥3 passing tests, and `CHANGELOG.md` contains the required `## Unreleased` bullets. If any check fails, return to step 1.
