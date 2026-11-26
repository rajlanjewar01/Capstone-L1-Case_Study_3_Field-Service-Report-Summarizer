# AI Output Review — Field Service Report Summarizer

Review of AI-generated code and outputs across five lenses, as required by the L1 AI-Augmented SDLC capstone.

## 1. Intent — Does the code do what the spec says?

**Finding**: The tool follows the spec's requirements. It:
- Produces summaries containing all six required elements (asset, date, findings, actions, parts, recommendations, time on site)
- Excludes prohibited content via LLM instructions + post-validation regex
- Flags inconsistent reports (FSR-3005 duration mismatch, FSR-3006 parts contradiction)
- Marks insufficient reports (FSR-3008) as incomplete rather than generating a confident summary
- Detects and ignores the prompt injection attempt in FSR-3009
- Applies the documented conflict-resolution decision (present both values, don't pick one)

**Issue found**: FSR-3007 has resolution "Attended site" and notes "See job sheet" — this contains minimal but slightly more than the 10-character threshold for `resolution` length. The insufficient-data check uses `len(res) < 10` and "Attended site." is 14 characters, so it passed. However, "See job sheet" provides no actionable information.

**Correction**: The insufficient-data check should also flag reports where notes refer only to external documents (e.g. "see job sheet", "see attached", "refer to") without any substantive content. This was fixed in a subsequent commit by adding a check for external-reference-only notes.

## 2. Tests — Are there tests, and do they cover edge cases?

**Finding**: No automated test suite was written. For a tool this size (single-file CLI processing 20 reports), the manual verification against known problem reports is the primary test. The pre-processing validation functions (`detect_flags`) are deterministic and testable.

**Recommendation**: For production use, unit tests for `detect_flags`, `post_validate_summary`, and `compute_duration` would be valuable. For this capstone, the manual review against all 20 reports with specific attention to the 8 known problem reports serves as the test.

## 3. Security — Does the tool ever leak PII or access codes in its output?

This is the most critical lens for this tool. The security concern is **information leakage**, not code-level vulnerabilities.

**Defence-in-depth approach**:
1. **LLM system prompt** explicitly prohibits names, phone numbers, emails, addresses, access codes, technician IDs
2. **Post-validation regex** catches any PII the LLM missed — UK phone patterns, email addresses, access code patterns, address patterns, and names extracted from the original notes
3. **HTML output uses `_escape_html()`** to prevent XSS in the portal

**Issue found and corrected**: During initial review, I discovered the AI-generated code's `extract_names_from_notes` function only looked for names preceded by keywords like "contact is" or "manager". A name appearing in a different context (e.g. "spoke to John Smith about the issue") would not be caught. The function was broadened to also check standalone capitalized multi-word patterns, with a list of common false positives (equipment names like "Cooling tower") to exclude.

**Specific verification**:
- **FSR-3003**: Contains Margaret Oyelaran (name), 07700 900412 (mobile), 14 Alderman Court flat 3B (address), 4471 (access code). The published summary contains none of these.
- **FSR-3014**: Contains Dev Ramaswamy (name), d.ramaswamy@northgate-fm.example (email), 0161 496 0221 (phone). The published summary contains none of these.
- **All reports**: No technician IDs (T-118, T-204, T-311) appear in any summary.

**Residual risk**: The regex-based post-validation cannot catch every possible form of PII. A name that doesn't follow capitalization conventions, or a phone number in an unusual format, might pass through. For production use, a dedicated PII detection service would be stronger.

## 4. Performance — Is the tool efficient enough?

**Finding**: The tool processes 20 reports sequentially, making one API call per report. Total runtime is approximately 2-3 minutes for 20 reports. This is acceptable for the use case (batch processing a backlog, not real-time).

**Potential improvement**: Reports could be processed in parallel using `asyncio` and the Anthropic async client. For 20 reports this is unnecessary; for the production backlog of hundreds, it would reduce wall-clock time significantly.

**Cost**: Each report uses approximately 500-800 input tokens and 200-400 output tokens. At Claude Sonnet pricing via Portkey/Bedrock, the total cost for 20 reports is negligible (under $0.05).

## 5. Maintainability — Is the code readable and structured for future changes?

**Finding**: The code is a single 400-line Python file with clear function separation:
- `load_reports()` — data loading
- `detect_flags()` — pre-processing validation
- `post_validate_summary()` — post-LLM redaction check
- `summarize_report()` — LLM orchestration
- `generate_portal_html()` — output rendering
- `generate_analysis()` — analysis document generation

Each function has a single responsibility. The system prompt and redaction patterns are defined as module-level constants for easy modification.

**Issue found**: The HTML generation uses string formatting (f-strings) to build the portal page. For a larger template this would become hard to maintain. For 20 reports it is adequate, but a template engine (Jinja2) would be the right choice at scale.

## Summary of Issues Found and Corrected

| Issue | Lens | Severity | Action |
|-------|------|----------|--------|
| FSR-3007 not flagged as insufficient despite minimal content | Intent | Medium | Added external-reference detection to insufficient-data check |
| Name extraction too narrow — only keyword-preceded names caught | Security | High | Broadened to standalone capitalized name patterns |
| FSR-3011 truncated due to 1024 max_tokens | Intent | High | Increased to 2048 for long reports (notes > 500 chars) |
| No automated tests | Tests | Low | Acceptable for capstone scope; documented as improvement area |

## Declared Effort

This tool was built in a single session using Claude Code as the AI coding assistant. The spec, plan, and tasks were written before the code. The implementation was generated with AI assistance, then reviewed and corrected across all five lenses documented above.
