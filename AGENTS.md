# AGENTS.md

Guidelines for AI coding agents working in this repository. Follow these rules
for all generated or modified code. When a language-specific linter/config
exists (e.g. `.editorconfig`, `pyproject.toml`, `.shellcheckrc`), it takes
precedence over this file.

## General Principles

- Write code for readers, not just the compiler. Optimize for clarity first.
- Fail fast and loudly: validate inputs early, surface errors with actionable messages.
- Prefer standard-library and well-established tooling over novel dependencies.
- Never commit secrets, tokens, or machine-specific paths.
- Keep functions short and single-purpose; extract when logic branches on more than two conditions.
- Match the style of nearby code when editing existing files; do not reformat untouched code.

## Comments

- Comments explain **why**, not **what**. The code already says what happens.
- Every file starts with a brief header comment: purpose, author/date optional, license if required.
- Every non-trivial function gets a docstring or header comment covering: purpose, parameters, return value, side effects, and thrown/exit errors.
- Use `TODO(owner): description` for incomplete work. Never leave a vague `TODO` without context.
- Delete commented-out code instead of committing it; version control keeps history.
- Do not state the obvious (`i = i + 1  # increment i`).
- Keep comment lines under ~100 characters, matching code line length.

---

## Bash

### Style

- Shebang: `#!/usr/bin/env bash`. Never rely on `sh` aliases.
- Strict mode at the top of every script:

```bash
set -Eeuo pipefail
```

- Use 4 spaces, no tabs. Max line length 100.
- Quote all variable expansions: `"$var"`, `"${var}"`. Use `"${var:-default}"` for defaults.
- Prefer `[[ ]]` conditionals over `[ ]`; use `(( ))` for arithmetic.
- Use lowercase for variables; reserve uppercase for exported/environment variables and constants.
- Name scripts and functions in `snake_case`. Prefix internal functions with `_`.
- Prefer `local` variables in functions; declare globals explicitly and minimally.
- Use `printf '%s\n'` over `echo` for untrusted data.
- Use `command -v` to test for binaries; `readonly` for constants; traps for cleanup:

```bash
cleanup() { rm -f "$tmpfile"; }
trap cleanup EXIT
```

- Run `shellcheck` (add a directive on line 1 only when needed) and `shfmt` with default settings. CI must pass both.

### Commenting

- Header block after the shebang: purpose, usage (`Usage: script.sh <arg>`), exit codes.
- Comment each function above its definition:

```bash
# Deploys the built artifact to the staging bucket.
# Args:
#   $1 - target environment (staging|prod)
# Returns: 0 on success, 1 on invalid environment.
deploy_artifact() { ... }
```

- Explain non-obvious flags, regexes, and any suppression of `shellcheck`.
- Document exit codes; use `exit 1` for usage errors and distinct codes for distinct failures.

### Example

```bash
#!/usr/bin/env bash
# Backs up the config directory to a timestamped archive.
# Usage: backup.sh [source_dir]
# Exit codes: 0 success, 1 missing source, 2 archive failure.
set -Eeuo pipefail

readonly source_dir="${1:-./config}"

# Fail early if the source is missing (why: silent no-op backups hid failures).
if [[ ! -d "$source_dir" ]]; then
  printf 'Error: source directory not found: %s\n' "$source_dir" >&2
  exit 1
fi
```

---

## PowerShell

### Style

- Use `#!/usr/bin/env pwsh` (PowerShell 7+, cross-platform). Avoid Windows-only 5.1 constructs when possible.
- Use full cmdlet names in scripts: `Test-Path`, not `tp`. Approved aliases are fine interactively, never in committed code.
- Use `Verb-Noun` naming for functions with approved verbs (`Get-`, `Set-`, `New-`, `Test-`).
- Use `PascalCase` for functions and cmdlets, `$PascalCase` for variables. Avoid abbreviations (`$ServerName`, not `$SrvNm`).
- Use 4 spaces, no tabs. Max line length 120.
- Prefer `"..."` strings only when interpolation or escapes are needed; otherwise `'...'`.
- Use splatting for calls with more than 3 parameters:

```powershell
$params = @{
    Path    = $path
    Recurse = $true
    Filter  = '*.log'
}
Get-ChildItem @params
```

- Always use `-ErrorAction Stop` on cmdlets inside `try` blocks, and handle explicitly:

```powershell
try { Copy-Item $src $dst -ErrorAction Stop }
catch { Write-Error "Copy failed: $_"; exit 1 }
```

- Use `[CmdletBinding()]` and `param()` blocks with type constraints and `[Parameter()]` attributes in reusable functions.
- Add `#Requires -Version 7.0` when using modern features.
- Run `PSScriptAnalyzer` with the default rule set; fix all warnings.

### Commenting

- Use comment-based help for every function and script:

```powershell
<#
.SYNOPSIS
    Archives rotated logs older than the retention window.
.DESCRIPTION
    Finds .log files under $Path older than $Days and compresses them.
.PARAMETER Path
    Root directory to scan.
.EXAMPLE
    Archive-OldLogs -Path C:\logs -Days 30
#>
```

- Explain any `-ErrorAction SilentlyContinue` with a justification comment.
- Do not leave commented-out commands; delete them.

---

## C / C++

### Style

- Target **C17** and **C++20** unless the project pins otherwise; declare the standard explicitly in the build system, never rely on compiler defaults.
- Format with `clang-format` using a committed `.clang-format` file; run `clang-tidy` in CI. Machine formatting is non-negotiable in review.
- Use 4 spaces, no tabs. Max line length 100.
- Naming (choose one convention per project and stick to it; suggested default):
  - C: `snake_case` for functions and variables, `SCREAMING_SNAKE_CASE` for macros and constants, `UpperCamelCase` for types/structs.
  - C++: `UpperCamelCase` for types, `camelCase` or `snake_case` for functions and variables (match the codebase), `SCREAMING_SNAKE_CASE` for macros, `g_` prefix for globals, `m_` or trailing `_` for members.
- Braces on their own line (C) or same line (C++/Google style); never mix within a file.
- In C++, prefer `constexpr`/`const` over macros, `enum class` over plain enums, `nullptr` over `NULL`/`0`, `using` over `typedef`.
- Use `<stdint.h>`/`<cstdint>` fixed-width types for anything stored, transmitted, or hardware-facing.
- Prefer stack and RAII over manual `new`/`delete`; use `std::unique_ptr`/`std::shared_ptr` when heap allocation is required. In C, every `malloc`/`open`/`fopen` must have an obvious matching `free`/`close`/`fclose` on all paths.
- Never return or store raw pointers to local variables; document pointer ownership at API boundaries (caller owns, callee owns, or borrowed).
- Initialize every variable at declaration; do not read uninitialized memory.
- Compile with warnings fully enabled (`-Wall -Wextra -Werror`, or `/W4 /WX` on MSVC) and treat warnings as errors in CI.
- Sanitizers in debug builds: ASan + UBSan always, TSan for threaded code.
- No `using namespace` in headers. Headers must be self-contained (compile standalone) and idempotent (include guards or `#pragma once`).

### Commenting

- Header files document the public contract; source files document implementation. Every non-trivial function declaration in a header gets a comment or Doxygen-style block:

```c
/**
 * Parse a telemetry frame into a struct.
 *
 * @param buf      Raw frame bytes (at least FRAME_LEN long).
 * @param out      Destination struct; untouched on failure.
 *
 * @return 0 on success, negative errno-style code on failure.
 *         Frame CRC is validated before any field is copied.
 */
int frame_parse(const uint8_t *buf, frame_t *out);
```

- Use `//` for inline comments (C99+ and C++); reserve `/* */` for file headers and doxygen blocks if the project uses them.
- Explain every `volatile`, memory barrier, raw cast, `reinterpret_cast`, bit manipulation, and magic number. Magic numbers become named `constexpr`/`#define` constants with a unit comment.
- Mark deliberate fallthrough and ignored return values explicitly (`// fall through`, `(void)result; // best-effort, failure OK`).
- Use `// TODO(owner): description` for known gaps; `// NOLINT` only with a justification on the same line.
- In hardware-facing code, comment register/buffer interactions with the datasheet or protocol reference (e.g. `// Per IMU datasheet §5.2: wait 10 ms after power-up`).
- Delete commented-out code; git keeps history.

---

## HTML

### Style

- Target modern HTML5: `<!DOCTYPE html>`, `<html lang="...">`, and UTF-8 `<meta charset="utf-8">` as the first element in `<head>`.
- 2 spaces indentation, no tabs. One block element per line; wrap attributes if the line exceeds 100 characters.
- Lowercase tags and attribute names; double quotes for attribute values.
- Always close elements; never add a trailing slash to void elements (`<br>`, `<img>`, `<input>`).
- Semantic elements first: `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<footer>`; use `<div>` only as a structural last resort.
- Accessibility is not optional: every image gets meaningful `alt` (or `alt=""` if decorative), every form control has an associated `<label>`, every interactive element is keyboard reachable, and heading hierarchy is flat (`h1` once, no skipped levels).
- Keep CSS in external stylesheets and JS in external files; no inline styles or inline event handlers in committed code.
- Validate markup with the W3C validator in CI when feasible.

### Commenting

- Use `<!-- ... -->` sparingly: section markers and non-obvious structure only:

```html
<!-- Site navigation (rendered from nav.yml) -->
<nav class="site-nav">...</nav>

<!-- End pricing table -->
```

- Use end-tag comments (`<!-- /main -->`) only in deeply nested markup.
- Never leave commented-out markup; delete it.
- Use `<!-- TODO(owner): ... -->` for incomplete sections.
- Conditional comments and browser hacks are forbidden.

---

## Python

### Style

- Target the newest stable Python 3; declare supported versions explicitly in `pyproject.toml`.
- Follow **PEP 8** and **PEP 257**. Enforce with `ruff` (lint + format) in CI; run `mypy` in strict mode for anything beyond one-off scripts.
- 4 spaces indentation. Max line length 100 (configured in ruff).
- Naming: `snake_case` for functions/variables/modules, `PascalCase` for classes, `SCREAMING_SNAKE_CASE` for constants, leading underscore for internal API.
- Type-hint all public functions and methods; always annotate return types.
- Prefer f-strings; `pathlib.Path` over `os.path` string wrangling; `with` blocks for all resource handling.
- Raise specific exceptions with informative messages; never use bare `except:`.
- Use dataclasses or `pydantic` models instead of loose dicts for structured data.
- Prefer comprehensions over `map`/`filter`; prefer generators for large pipelines.
- Imports: standard library, third-party, local, separated by blank lines, alphabetized, no wildcard imports.

### Commenting

- Docstrings on every public module, class, function, and method. Use Google style:

```python
def fetch_telemetry(channel: str, since: datetime) -> list[Reading]:
    """Fetch readings for a channel since a timestamp.

    Args:
        channel: Telemetry channel identifier, e.g. 'AV_BATT_V'.
        since: Inclusive lower bound for the query window.

    Returns:
        Readings ordered oldest-first.

    Raises:
        TelemetryError: If the backend is unreachable or the channel is unknown.
    """
```

- Inline comments: rare, only to explain *why* (workarounds, non-obvious math, performance choices). Single `# ` prefix.
- Use `# TODO(owner): description` for known gaps; `# type: ignore[code]` only with a justification comment.
- Never leave debug `print()` calls; use `logging` with a module-level `logger = logging.getLogger(__name__)`.
- Delete dead code and commented-out blocks; rely on git history.

---

## Commit Hygiene (all languages)

- Write imperative commit subjects: `Add retention filter to log archiver`.
- One logical change per commit; no drive-by reformatting mixed with behavior changes.
- Update comments and docstrings in the same commit as the code they describe.