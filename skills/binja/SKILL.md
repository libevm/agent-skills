---
name: binja
description: Use this skill for Binary Ninja reverse engineering tasks including inspecting binaries, enumerating functions, walking cross-references, analyzing strings, reading BNIL (LLIL/MLIL/HLIL), and building Binary Ninja Python scripts/plugins. Covers headless analysis, symbol/import/section listing, annotations, .bndb workflows, and PluginCommand development.
---

# Binary Ninja

Use this skill when you need to inspect binaries, enumerate functions, walk cross-references, analyze strings, read BNIL (LLIL/MLIL/HLIL), or build Binary Ninja scripts/plugins for reverse engineering.

## What this skill is for

This skill helps an AI agent use the **Binary Ninja Python API** effectively and safely.

Prefer this skill for tasks such as:
- Opening a binary and running analysis headlessly
- Listing functions, symbols, imports, strings, and sections
- Collecting code/data cross-references
- Reading LLIL, MLIL, mapped MLIL, and HLIL
- Renaming functions, adding comments, or applying annotations
- Saving analysis to a `.bndb`
- Writing Binary Ninja Python plugins with `PluginCommand`

Do **not** use this skill when the user wants raw disassembly from another toolchain, dynamic debugging outside Binary Ninja, or Ghidra/IDA-specific behavior.

## Ground rules for agents

1. **Prefer the Python API.** It is the most heavily documented API and is the best reference point even when concepts later map to Rust or C++.
2. **Treat `BinaryView` (`bv`) as the top-level analysis object.** Most work starts from an open `BinaryView`.
3. **Use headless-friendly code unless the user explicitly wants UI-only behavior.**
4. **Do not rely on Python console magic variables** such as `current_function` or `bv` in scripts or headless automation. They are convenient in the UI console, but `current_function` is explicitly documented as console-only and not for headless plugins.
5. **Generate IL on demand.** LLIL/MLIL/HLIL generation can be expensive; use it only when needed.
6. **Wait for analysis before trusting results.** If analysis is deferred or you changed analysis inputs, call `bv.update_analysis_and_wait()`.
7. **Keep writes deliberate.** Renaming, comments, type changes, and saved databases should be explicit.

## Required knowledge

### Core mental model

- `load(...)` opens a file or byte stream and returns a `BinaryView`.
- `BinaryView` represents the loaded binary in virtual memory and is the main query surface.
- A `Function` exposes disassembly, metadata, and IL views.
- Binary Ninja offers several IL layers:
  - **LLIL**: low-level, close to machine semantics
  - **MLIL**: normalized medium-level operations
  - **Mapped MLIL / MMLIL**: mapped medium-level form
  - **HLIL**: higher-level reconstruction
- Cross-references and string recovery live primarily off the `BinaryView`.

### Important APIs to know

- Open binary:
  - `from binaryninja import load`
  - `with load(path) as bv:`
- Force analysis:
  - `bv.update_analysis_and_wait()`
- Enumerate functions:
  - `bv.functions`
  - `bv.get_functions_containing(addr)`
  - `bv.get_functions_by_name(name)`
- Enumerate strings:
  - `bv.get_strings()`
- Cross-references:
  - `bv.get_code_refs(addr)`
  - `bv.get_data_refs(addr)`
  - `bv.get_code_refs_from(addr)` / `bv.get_data_refs_from(addr)` when outgoing refs matter
- IL access on `Function`:
  - `func.llil`, `func.mlil`, `func.mmlil`, `func.hlil`
  - `func.llil_if_available`, `func.mlil_if_available`, `func.hlil_if_available` when you want to avoid forcing generation
- Save database:
  - `bv.create_database("file.bndb")`
  - `bv.save_auto_snapshot()` after a database already exists
- Plugin registration:
  - `binaryninja.plugin.PluginCommand.register(...)`
  - `register_for_address(...)`
  - `register_for_function(...)`

## Environment expectations

The agent should assume:
- Binary Ninja is installed locally.
- The Python API is available via `import binaryninja`.
- For stand-alone headless usage, the Binary Ninja API has been installed into the local Python environment.
- Some plugin categories only load at Binary Ninja startup and are not hot-reloadable.

If `import binaryninja` fails, stop and report that the Binary Ninja API is not installed in the current Python environment.

## Installing the Binary Ninja API into a Python environment

When `import binaryninja` is unavailable, prefer installing the API into the exact Python interpreter the user will run. Binary Ninja ships an `install_api.py` helper for this purpose. This is especially useful for headless scripting and for project-local environments such as `venv` or `uv` environments.

### macOS

Binary Ninja is typically installed at `/Applications/Binary Ninja.app`. To install the API into a specific environment, run Binary Ninja's installer script with that environment's Python interpreter.

Example with a standard virtual environment:

```bash
python /Applications/Binary\ Ninja.app/Contents/Resources/scripts/install_api.py
```

Example with `uv` and a project-local environment:

```bash
uv venv .venv --python 3.11
uv run --python .venv/bin/python -- \
  python /Applications/Binary\ Ninja.app/Contents/Resources/scripts/install_api.py
uv run --python .venv/bin/python -- \
  python -c "import binaryninja; print(binaryninja.__file__)"
```

If Binary Ninja is installed somewhere else, locate the script first and substitute the actual path:

```bash
find /Applications/Binary\ Ninja.app -name install_api.py
```

### Linux

On Linux, Binary Ninja also ships `install_api.py`, and many installations additionally include `scripts/linux-setup.sh`. For a specific virtual environment, prefer `install_api.py` with that environment's Python interpreter.

Example with a standard virtual environment:

```bash
python /path/to/binaryninja/scripts/install_api.py
```

Example with `uv` and a project-local environment:

```bash
uv venv .venv --python 3.11
uv run --python .venv/bin/python -- \
  python /path/to/binaryninja/scripts/install_api.py
uv run --python .venv/bin/python -- \
  python -c "import binaryninja; print(binaryninja.__file__)"
```

For system-wide shell setup on Linux, `./binaryninja/scripts/linux-setup.sh` may still be appropriate, but for isolated project environments use `install_api.py` against the exact interpreter that should import `binaryninja`.

### Agent guidance

- Prefer `install_api.py` when the user wants the API available inside a particular interpreter, virtual environment, CI job, or `uv` project.
- On macOS, use the app bundle path under `/Applications/Binary Ninja.app/...` unless the user has installed it elsewhere.
- On Linux, replace `/path/to/binaryninja` with the user's actual extraction or install directory.
- After installation, verify with `python -c "import binaryninja; print(binaryninja.__file__)"`.

## Default workflow

### 1. Open and analyze the target

Use a context manager and keep the script deterministic.

```python
from binaryninja import load

with load("./target.bin") as bv:
    bv.update_analysis_and_wait()
    print(f"Functions: {len(list(bv.functions))}")
```

Notes:
- `load(...)` defaults to running analysis unless told otherwise.
- If the script changes analysis inputs or relies on finished analysis, call `bv.update_analysis_and_wait()`.

### 2. Start with `BinaryView`-level facts

Prefer quick inventory before expensive IL work:
- filename / architecture / platform
- functions
- symbols
- strings
- sections / segments if relevant
- imports / external references

Example:

```python
from binaryninja import load

with load("./target.bin") as bv:
    for func in bv.functions:
        print(hex(func.start), func.name)

    for s in bv.get_strings():
        print(hex(s.start), str(s))
```

### 3. Only then move into IL

Use IL when the task needs semantics rather than raw addresses.

Example: dump HLIL per function.

```python
from binaryninja import load

with load("./target.bin") as bv:
    for func in bv.functions:
        print(f"\n== {func.name} @ {func.start:#x} ==")
        if func.hlil is None:
            continue
        for block in func.hlil:
            for instr in block:
                print(instr)
```

### 4. Use xrefs for navigation and evidence

Example: find code references to a string.

```python
from binaryninja import load

with load("./target.bin") as bv:
    for s in bv.get_strings():
        text = str(s)
        if "password" in text.lower():
            print(f"String {text!r} at {s.start:#x}")
            for ref in bv.get_code_refs(s.start):
                print("  code ref from", hex(ref.address))
                for func in bv.get_functions_containing(ref.address):
                    print("    in function", func.name, hex(func.start))
```

### 5. Persist work when useful

Create a BNDB when analysis or annotations should be reopened later.

```python
from binaryninja import load

with load("./target.bin") as bv:
    bv.update_analysis_and_wait()
    ok = bv.create_database("./target.bndb")
    print("saved" if ok else "save failed")
```

## Common task patterns

### Enumerate all functions

```python
for func in bv.functions:
    print(func.start, func.name)
```

### Find the function containing an address

```python
funcs = bv.get_functions_containing(addr)
```

### Get strings in the binary

```python
for s in bv.get_strings():
    print(s.start, str(s))
```

### Collect incoming code xrefs to an address

```python
for ref in bv.get_code_refs(addr):
    print(ref.address)
```

### Work with IL only if already generated

```python
if func.hlil_if_available is not None:
    for block in func.hlil_if_available:
        for instr in block:
            print(instr)
```

### Rename or annotate deliberately

When writing scripts that change the database, make the action explicit and log it.

```python
func.name = "init_uart"
func.set_comment_at(func.start, "Recovered from xref/string pattern")
```

## Plugin development guidance

Use plugins when the user wants reusable in-UI actions instead of one-off scripts.

### Minimal plugin command

```python
from binaryninja import PluginCommand


def summarize_function(bv, func):
    print(f"Function {func.name} starts at {func.start:#x}")
    if func.hlil is not None:
        for block in func.hlil:
            for instr in block:
                print(instr)


PluginCommand.register_for_function(
    "Agent Tools\\Summarize Function",
    "Print a simple HLIL-based summary for the current function",
    summarize_function,
)
```

Use:
- `register(...)` for a generic command
- `register_for_address(...)` when the callback should operate on the current address
- `register_for_function(...)` when the callback should operate on the current function

### Plugin iteration

For reloadable Python plugins, a practical test loop is:

```python
import importlib
import pluginname
importlib.reload(pluginname)
```

Remember that some plugin types, including architectures and BinaryViews, are only loaded at launch and cannot be reloaded in a running session.

## Agent decision rules

### Prefer these starting points

- **Need a full-binary overview?** Start from `bv.functions`, `bv.get_strings()`, symbols, sections.
- **Need semantic behavior?** Inspect `func.mlil` or `func.hlil`.
- **Need precise machine-level effects?** Use `func.llil`.
- **Need to connect evidence?** Use `get_code_refs`, `get_data_refs`, and `get_functions_containing`.
- **Need repeatable UI actions?** Build a plugin command.
- **Need batch automation?** Write a stand-alone Python script using `load(...)`.

### Avoid these mistakes

- Using `current_function` or other UI console magic variables in headless scripts
- Iterating HLIL for every function before knowing it is needed
- Assuming analysis is complete without waiting when the script depends on finished results
- Confusing `get_code_refs` with `get_data_refs`
- Calling `save_auto_snapshot()` before a database exists
- Treating `.bndb` save APIs as if they write the original binary

## Recommended response style for agents

When helping a user, structure work in this order:
1. State what artifact or binary you will inspect.
2. Start with a small inventory script.
3. Escalate to IL only for the functions or addresses that matter.
4. Show exact addresses, function names, and evidence sources.
5. Keep edits opt-in and easy to undo.

## Ready-to-use script templates

### Inventory script

```python
from binaryninja import load

with load("./target.bin") as bv:
    print("File:", bv.file.filename)
    print("Functions:", len(list(bv.functions)))
    print("Strings:", len(bv.get_strings()))

    print("\nFunctions:")
    for func in bv.functions:
        print(f"{func.start:#x} {func.name}")
```

### Strings to xrefs script

```python
from binaryninja import load

NEEDLES = ["password", "token", "key", "error", "uart"]

with load("./target.bin") as bv:
    for s in bv.get_strings():
        text = str(s)
        if any(n in text.lower() for n in NEEDLES):
            print(f"\n{text!r} @ {s.start:#x}")
            for ref in bv.get_code_refs(s.start):
                print(f"  xref from {ref.address:#x}")
                for func in bv.get_functions_containing(ref.address):
                    print(f"    function {func.name} @ {func.start:#x}")
```

### HLIL dump for one function by name

```python
from binaryninja import load

TARGET = "main"

with load("./target.bin") as bv:
    funcs = bv.get_functions_by_name(TARGET)
    for func in funcs:
        print(f"\n== {func.name} @ {func.start:#x} ==")
        if func.hlil is None:
            continue
        for block in func.hlil:
            for instr in block:
                print(instr)
```

## Verification checklist

Before returning a result to the user, verify:
- The binary loaded successfully
- Analysis completed if the result depends on it
- The reported address belongs to the stated function
- Xrefs are code refs vs data refs as claimed
- IL level matches the claim being made
- Any save operation wrote the intended artifact (`.bndb` vs original file)

## Sources used to build this skill

- Binary Ninja Python API Reference: https://api.binary.ninja/
- BinaryView module docs: https://api.binary.ninja/binaryninja.binaryview-module.html
- Function module docs: https://api.binary.ninja/binaryninja.function-module.html
- Plugin module docs: https://api.binary.ninja/binaryninja.plugin-module.html
- Binary Ninja developer docs: https://docs.binary.ninja/dev/api.html
- Writing Python Plugins: https://docs.binary.ninja/dev/plugins.html
- BNIL Guide / cookbook pages in the Binary Ninja docs
