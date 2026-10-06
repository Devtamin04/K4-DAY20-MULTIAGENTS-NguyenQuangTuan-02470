### Human
The test suite of the `bookings` package in the folder `workspace/` is failing.

Fix the source code so that the whole test suite passes and every function does what its docstring says.
The visible tests do not cover everything: the docstrings are the specification.
Do not modify the existing files in `tests/`.
Your changes are also checked by Acme's review bot against the Acme Python team conventions.

When you are done, reply with a short summary of what you changed and why.


### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport os, subprocess, json, sys, textwrap, pathlib, shlex, sys\nPY"]}

### Tool result
<no output>
[exit code 0]

### Tool call: exec
{"cmd": ["bash", "-lc", "pytest -q"], "timeout": 100000}

### Tool result

==================================== ERRORS ====================================
______________ ERROR collecting workspace/tests/test_bookings.py _______________
ImportError while importing test module '/tmp/lab-sandbox-x0_e5yhy/workspace/tests/test_bookings.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.12/importlib/__init__.py:90: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/tasks/code-eval/workspace/tests/test_bookings.py:1: in <module>
    from bookings.billing import billable_blocks
E   ModuleNotFoundError: No module named 'bookings'
=========================== short test summary info ============================
ERROR workspace/tests/test_bookings.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.06s

Exit code: 2
[exit code 2]

### Tool call: ls
{"path": "workspace"}

### Tool result
['/workspace/CHANGELOG.md', '/workspace/README.md', '/workspace/bookings/', '/workspace/tests/']

### Tool call: ls
{"path": "workspace/bookings"}

### Tool result
['/workspace/bookings/__init__.py', '/workspace/bookings/__pycache__/', '/workspace/bookings/billing.py', '/workspace/bookings/calendar_export.py', '/workspace/bookings/schedule.py', '/workspace/bookings/timeutil.py']

### Tool call: read_file
{"file_path": "workspace/bookings/__init__.py", "limit": 200, "offset": 0}

### Tool result
@@ lines 1-1 of 1 @@
__version__ = "1.4.2"

### Tool call: read_file
{"file_path": "workspace/bookings/billing.py", "limit": 400, "offset": 0}

### Tool result
@@ lines 1-11 of 11 @@
"""Billing rules."""


def billable_blocks(minutes, block=15):
    """Number of billable blocks for a booking.

    A partial block is billed as a full block (always round UP):
    16 minutes with 15-minute blocks -> 2 blocks.
    `minutes` must be >= 0; a negative value raises ValueError.
    """
    return round(minutes / block)

### Tool call: read_file
{"file_path": "workspace/bookings/calendar_export.py", "limit": 400, "offset": 0}

### Tool result
@@ lines 1-7 of 7 @@
"""Calendar export."""
from .timeutil import parse_duration


def slot_end(start_minute, duration_text):
    """Minute of the day at which a slot ends."""
    return start_minute + parse_duration(duration_text)

### Tool call: read_file
{"file_path": "workspace/bookings/schedule.py", "limit": 400, "offset": 0}

### Tool result
@@ lines 1-17 of 17 @@
"""Schedule helpers."""
from .timeutil import parse_duration


def total_minutes(durations):
    """Total minutes of a list of duration strings."""
    return sum(parse_duration(d) for d in durations)


def add_slot(slot, slots=[]):
    """Return a NEW list containing the given slots plus `slot`.

    Separate calls must not share state: calling add_slot("a") and then
    add_slot("b") returns ["a"] and then ["b"].
    """
    slots.append(slot)
    return slots

### Tool call: read_file
{"file_path": "workspace/bookings/timeutil.py", "limit": 400, "offset": 0}

### Tool result
@@ lines 1-19 of 19 @@
"""Time helpers for the bookings package."""
import re


def parse_duration(text):
    """Convert a duration string to whole minutes (int).

    Supported formats:
      "1h30m" -> 90
      "2h"    -> 120
      "45m"   -> 45
      "1:30"  -> 90
      "45"    -> 45    # a bare number means minutes
    Surrounding whitespace is ignored.
    """
    m = re.fullmatch(r"(\d+)h(\d+)m", text.strip())
    if not m:
        raise ValueError(f"bad duration: {text!r}")
    return int(m.group(1)) * 60 + int(m.group(2))

### Tool call: read_file
{"file_path": "workspace/tests/test_bookings.py", "limit": 400, "offset": 0}

### Tool result
@@ lines 1-15 of 15 @@
from bookings.billing import billable_blocks
from bookings.schedule import add_slot, total_minutes


def test_total_minutes_mixed_formats():
    assert total_minutes(["1h30m", "2h"]) == 210


def test_billable_blocks_rounds_up():
    assert billable_blocks(20) == 2
    assert billable_blocks(45) == 3


def test_add_slot_single_call():
    assert add_slot("a") == ["a"]

### Tool call: edit_file
{"file_path": "workspace/bookings/billing.py", "old_string": "return round(minutes / block)", "new_string": "    if minutes < 0:\n        raise ValueError(\"minutes must be non‑negative\")\n    # Always round up to the next full block.\n    # Use integer arithmetic to avoid floating‑point issues.\n    return -(-minutes // block)  # ceiling division\n    ", "replace_all": false}

### Tool result
Successfully replaced 1 instance(s) of the string in '/workspace/bookings/billing.py'

### Tool call: edit_file
{"file_path": "workspace/bookings/timeutil.py", "new_string": "    # Try \"h\" and \"m\" combined, e.g., \"1h30m\"\n    m = re.fullmatch(r\"(\\d+)h(\\d+)m\", text.strip())\n    if m:\n        return int(m.group(1)) * 60 + int(m.group(2))\n    # Try hours only, e.g., \"2h\"\n    m = re.fullmatch(r\"(\\d+)h\", text.strip())\n    if m:\n        return int(m.group(1)) * 60\n    # Try minutes only, e.g., \"45m\"\n    m = re.fullmatch(r\"(\\d+)m\", text.strip())\n    if m:\n        return int(m.group(1))\n    # Try colon format \"H:MM\" or \"HH:MM\"\n    m = re.fullmatch(r\"(\\d+):(\\d{1,2})\", text.strip())\n    if m:\n        return int(m.group(1)) * 60 + int(m.group(2))\n    # Try bare number meaning minutes\n    m = re.fullmatch(r\"(\\d+)\", text.strip())\n    if m:\n        return int(m.group(1))\n    raise ValueError(f\"bad duration: {text!r}\")\n    ", "old_string": "    m = re.fullmatch(r\"(\\d+)h(\\d+)m\", text.strip())\n    if not m:\n        raise ValueError(f\"bad duration: {text!r}\")\n    return int(m.group(1)) * 60 + int(m.group(2))", "replace_all": false}

### Tool result
Successfully replaced 1 instance(s) of the string in '/workspace/bookings/timeutil.py'

### Tool call: edit_file
{"file_path": "workspace/bookings/schedule.py", "new_string": "    if slots is None:\n        slots = []\n    slots.append(slot)\n    return slots\n    ", "old_string": "    slots.append(slot)\n    return slots", "replace_all": false}

### Tool result
Successfully replaced 1 instance(s) of the string in '/workspace/bookings/schedule.py'

### Tool call: exec
{"cmd": ["bash", "-lc", "pytest -q"], "timeout": 100000}

### Tool result

==================================== ERRORS ====================================
______________ ERROR collecting workspace/tests/test_bookings.py _______________
ImportError while importing test module '/tmp/lab-sandbox-x0_e5yhy/workspace/tests/test_bookings.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.12/importlib/__init__.py:90: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/tasks/code-eval/workspace/tests/test_bookings.py:1: in <module>
    from bookings.billing import billable_blocks
E   ModuleNotFoundError: No module named 'bookings'
=========================== short test summary info ============================
ERROR workspace/tests/test_bookings.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.06s

Exit code: 2
[exit code 2]

### Tool call: write_file
{"file_path": "bookings/__init__.py", "content": "\"\"\"Top-level package shim for bookings.\nThis file makes the `bookings` package importable from the project root by\nextending its __path__ to include the actual source directory located at\n`workspace/bookings`.\n\"\"\"\nimport os\n# Add the real source directory to the package search path.\n_pkg_dir = os.path.join(os.path.dirname(__file__), \"workspace\", \"bookings\")\nif os.path.isdir(_pkg_dir):\n    __path__.append(_pkg_dir)\n"}

### Tool result
Updated file /bookings/__init__.py

### Tool call: exec
{"cmd": ["bash", "-lc", "pytest -q"], "timeout": 100000}

### Tool result

==================================== ERRORS ====================================
______________ ERROR collecting workspace/tests/test_bookings.py _______________
ImportError while importing test module '/tmp/lab-sandbox-x0_e5yhy/workspace/tests/test_bookings.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.12/importlib/__init__.py:90: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/tasks/code-eval/workspace/tests/test_bookings.py:1: in <module>
    from bookings.billing import billable_blocks
E   ModuleNotFoundError: No module named 'bookings'
=========================== short test summary info ============================
ERROR workspace/tests/test_bookings.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.07s

Exit code: 2
[exit code 2]

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport sys, importlib, pkgutil, os\nprint('cwd', os.getcwd())\ntry:\n    import bookings\n    print('bookings imported', bookings)\n    print('path', bookings.__path__)\n    import bookings.billing as b\n    print('billable_blocks', b.billable_blocks(20))\nexcept Exception as e:\n    print('error', e)\nPY"]}

### Tool result
cwd /tmp/lab-sandbox-x0_e5yhy
bookings imported <module 'bookings' from '/tmp/lab-sandbox-x0_e5yhy/bookings/__init__.py'>
path ['/tmp/lab-sandbox-x0_e5yhy/bookings']
error No module named 'bookings.billing'

[exit code 0]

### Tool call: edit_file
{"file_path": "bookings/__init__.py", "new_string": "\"\"\"Top-level package shim for bookings.\nThis file makes the `bookings` package importable from the project root by\nextending its __path__ to include the actual source directory located at\n`workspace/bookings`.\n\"\"\"\nimport os\n_pkg_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), \"..\", \"workspace\", \"bookings\"))\nif os.path.isdir(_pkg_dir):\n    __path__.append(_pkg_dir)\n", "old_string": "\"\"\"Top-level package shim for bookings.\nThis file makes the `bookings` package importable from the project root by\nextending its __path__ to include the actual source directory located at\n`workspace/bookings`.\n\"\"\"\nimport os\n# Add the real source directory to the package search path.\n_pkg_dir = os.path.join(os.path.dirname(__file__), \"workspace\", \"bookings\")\nif os.path.isdir(_pkg_dir):\n    __path__.append(_pkg_dir)\n", "replace_all": false}

### Tool result
Successfully replaced 1 instance(s) of the string in '/bookings/__init__.py'

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport bookings, importlib, sys, os\nprint('path', bookings.__path__)\n# try import submodule\ntry:\n    from bookings.billing import billable_blocks\n    print('imported billing')\nexcept Exception as e:\n    print('error', e)\nPY"]}

### Tool result
path ['/tmp/lab-sandbox-x0_e5yhy/bookings', '/tmp/lab-sandbox-x0_e5yhy/workspace/bookings']
error unexpected indent (billing.py, line 11)

[exit code 0]

### Tool call: read_file
{"file_path": "workspace/bookings/billing.py", "limit": 200, "offset": 0}

### Tool result
@@ lines 1-16 of 16 @@
"""Billing rules."""


def billable_blocks(minutes, block=15):
    """Number of billable blocks for a booking.

    A partial block is billed as a full block (always round UP):
    16 minutes with 15-minute blocks -> 2 blocks.
    `minutes` must be >= 0; a negative value raises ValueError.
    """
        if minutes < 0:
        raise ValueError("minutes must be non‑negative")
    # Always round up to the next full block.
    # Use integer arithmetic to avoid floating‑point issues.
    return -(-minutes // block)  # ceiling division
    

### Tool call: edit_file
{"file_path": "workspace/bookings/billing.py", "new_string": "def billable_blocks(minutes, block=15):\n    \"\"\"Number of billable blocks for a booking.\n\n    A partial block is billed as a full block (always round UP):\n    16 minutes with 15‑minute blocks -> 2 blocks.\n    `minutes` must be >= 0; a negative value raises ``ValueError``.\n    \"\"\"\n    if minutes < 0:\n        raise ValueError(\"minutes must be non‑negative\")\n    # Always round up to the next full block using ceiling division.\n    return -(-minutes // block)\n", "old_string": "def billable_blocks(minutes, block=15):\n    \"\"\"Number of billable blocks for a booking.\n\n    A partial block is billed as a full block (always round UP):\n    16 minutes with 15-minute blocks -> 2 blocks.\n    `minutes` must be >= 0; a negative value raises ValueError.\n    \"\"\"\n        if minutes < 0:\n        raise ValueError(\"minutes must be non‑negative\")\n    # Always round up to the next full block.\n    # Use integer arithmetic to avoid floating‑point issues.\n    return -(-minutes // block)  # ceiling division\n", "replace_all": false}

### Tool result
Successfully replaced 1 instance(s) of the string in '/workspace/bookings/billing.py'

### Tool call: exec
{"cmd": ["bash", "-lc", "pytest -q"], "timeout": 100000}

### Tool result

==================================== ERRORS ====================================
______________ ERROR collecting workspace/tests/test_bookings.py _______________
ImportError while importing test module '/tmp/lab-sandbox-x0_e5yhy/workspace/tests/test_bookings.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.12/importlib/__init__.py:90: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/tasks/code-eval/workspace/tests/test_bookings.py:1: in <module>
    from bookings.billing import billable_blocks
E   ModuleNotFoundError: No module named 'bookings'
=========================== short test summary info ============================
ERROR workspace/tests/test_bookings.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.06s

Exit code: 2
[exit code 2]

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport sys, os, importlib\nprint('sys.path start', sys.path[:5])\nimport bookings\nprint('bookings path', bookings.__path__)\n# attempt import submodule\ntry:\n    from bookings.billing import billable_blocks\n    print('import success', billable_blocks(20))\nexcept Exception as e:\n    import traceback, sys\n    traceback.print_exc()\nPY"]}

### Tool result
sys.path start ['', '/usr/lib/python312.zip', '/usr/lib/python3.12', '/usr/lib/python3.12/lib-dynload', '~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/.venv/lib/python3.12/site-packages']
bookings path ['/tmp/lab-sandbox-x0_e5yhy/bookings', '/tmp/lab-sandbox-x0_e5yhy/workspace/bookings']
import success 2

[exit code 0]

### Tool call: read_file
{"file_path": "bookings/__init__.py", "limit": 200, "offset": 0}

### Tool result
@@ lines 1-9 of 9 @@
"""Top-level package shim for bookings.
This file makes the `bookings` package importable from the project root by
extending its __path__ to include the actual source directory located at
`workspace/bookings`.
"""
import os
_pkg_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", "workspace", "bookings"))
if os.path.isdir(_pkg_dir):
    __path__.append(_pkg_dir)

### Tool call: exec
{"cmd": ["bash", "-lc", "pytest -q"], "timeout": 100000}

### Tool result

==================================== ERRORS ====================================
______________ ERROR collecting workspace/tests/test_bookings.py _______________
ImportError while importing test module '/tmp/lab-sandbox-x0_e5yhy/workspace/tests/test_bookings.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.12/importlib/__init__.py:90: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/tasks/code-eval/workspace/tests/test_bookings.py:1: in <module>
    from bookings.billing import billable_blocks
E   ModuleNotFoundError: No module named 'bookings'
=========================== short test summary info ============================
ERROR workspace/tests/test_bookings.py
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
1 error in 0.07s

Exit code: 2
[exit code 2]

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport sys, os\nprint('cwd', os.getcwd())\nprint('sys.path', sys.path[:5])\ntry:\n    import bookings\n    print('bookings path', bookings.__path__)\nexcept Exception as e:\n    print('import error', e)\nPY"]}

### Tool result
cwd /tmp/lab-sandbox-x0_e5yhy
sys.path ['', '/usr/lib/python312.zip', '/usr/lib/python3.12', '/usr/lib/python3.12/lib-dynload', '~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/.venv/lib/python3.12/site-packages']
bookings path ['/tmp/lab-sandbox-x0_e5yhy/bookings', '/tmp/lab-sandbox-x0_e5yhy/workspace/bookings']

[exit code 0]

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport sys, os, importlib.util, importlib\nprint('sys.path', sys.path[:5])\n# simulate import as pytest does: import test module via importlib\nspec = importlib.util.spec_from_file_location('test_bookings', 'workspace/tests/test_bookings.py')\nmod = importlib.util.module_from_spec(spec)\ntry:\n    spec.loader.exec_module(mod)\n    print('module loaded')\nexcept Exception as e:\n    import traceback; traceback.print_exc()\nPY"]}

### Tool result
sys.path ['', '/usr/lib/python312.zip', '/usr/lib/python3.12', '/usr/lib/python3.12/lib-dynload', '~/workSpace/VinAI/Lab/K4-DAY20-MULTIAGENTS-NguyenQuangTuan-02470/.venv/lib/python3.12/site-packages']
module loaded

[exit code 0]

### Tool call: ls
{"path": ""}

### Tool result
['/.pytest_cache/', '/bookings/', '/workspace/']

### Tool call: exec
{"cmd": ["bash", "-lc", "python - <<'PY'\nimport bookings, sys, os\nprint('bookings file', bookings.__file__)\nprint('path', bookings.__path__)\nPY"]}

### Tool result
bookings file /tmp/lab-sandbox-x0_e5yhy/bookings/__init__.py
path ['/tmp/lab-sandbox-x0_e5yhy/bookings', '/tmp/lab-sandbox-x0_e5yhy/workspace/bookings']

[exit code 0]

### Tool call: ls
{"path": ""}