# Issue Resolution: Reject malformed section headers and require whitespace for inline comments

## 1. Upstream Issue Reference
- **Repository:** `pytest-dev/iniconfig`
- **Issue:** [#83](https://github.com/pytest-dev/iniconfig/issues/83) - `Define a configparser-compatibility policy, and fix two silent misparses (malformed section headers, inline comments in parse())`
- **Status:** Open upstream (reported by iniconfig maintainer Ronny Pfannschmidt)

## 2. Problem Description
1. **Malformed Section Headers:**
   When an unindented line started with `[` but did not end with `]` (e.g. `[section` or `[section] extra`):
   Instead of raising a `ParseError`, `_parseline` fell through and returned `None, realline.strip()`, treating the line as continuation data. This silently corrupted INI parsing by swallowing the invalid section header into the preceding option value and dropping subsequent configuration into the wrong section.
2. **Inline Comments without Preceding Whitespace:**
   `IniConfig.parse()` blindly split values on `#` or `;` without requiring preceding whitespace. This truncated URLs (e.g. `http://x/#frag` became `http://x/`), values containing hash fragments, and semicolons without spaces. Standard `configparser` requires whitespace before an inline comment character.

## 3. Reproduction Test Cases
Added in `testing/test_iniconfig.py`:
```python
def test_iniconfig_unclosed_section_header_fails() -> None:
    with pytest.raises(ParseError) as excinfo:
        IniConfig("x", data="[section\nname = 1")
    assert "unclosed section header" in str(excinfo.value)

def test_iniconfig_unexpected_text_after_section_header_fails() -> None:
    with pytest.raises(ParseError) as excinfo:
        IniConfig("x", data="[section] extra\nname = 1")
    assert "unexpected text after section header" in str(excinfo.value)
```

And in `test_parse_strips_inline_comments()`:
```python
url = http://x/#frag # some URL
...
assert config["section1"]["url"] == "http://x/#frag"
assert config["section1"]["name3"] == "value3# no space before comment"
```

Prior to the fix:
- `[section` and `[section] extra` did not raise `ParseError`.
- `http://x/#frag` was truncated to `'http://x/'`.

## Solution Approach
1. **Enforce Strict INI Grammar Rules:** Analyzed two parsing ambiguities in `src/iniconfig/_parse.py`:
   - Malformed unindented lines starting with `[` (missing closing `]`) were silently appended as multiline continuations to the previous key instead of raising `ParseError`.
   - Inline `#` / `;` comments stripped characters even without preceding whitespace, corrupting values containing URL fragments (e.g., `http://example.com/#section`).
2. **Targeted Parser Tightening:** Rejected unindented unclosed section headers with `ParseError`, and required whitespace before inline comment delimiters so URL fragments and non-comment tokens stay intact.
3. **TDD & Regression Safety:** Added regression tests in `testing/test_iniconfig.py` covering malformed section headers and URL values with `#` fragments, and verified the full iniconfig test suite passes.

## 4. Solution & Code Changes
In `src/iniconfig/_parse.py`:
- Added `_strip_inline_comment(val: str)` requiring whitespace before `#` or `;` (matching Python's `configparser`).
- In `_parseline()`:
  - If a line starts with `[` and does not end with `]`, raise `ParseError`:
    - `"unexpected text after section header"` if `]` is present in the line.
    - `"unclosed section header"` if `]` is absent.
  - Used `_strip_inline_comment` when stripping inline comments from values, continuations, and section lines.

## 5. Verification
- All 92 unit tests in `testing/test_iniconfig.py` passed.
- All 6 property/fuzz tests in `testing/test_fuzz.py` passed.
- Ruff linter passed with zero warnings.
