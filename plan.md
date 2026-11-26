# Implementation Plan: Field Service Report Summarizer

## Tech stack

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Python 3.11+ | Standard for data processing; rich standard library for JSON, regex, datetime |
| LLM | Claude (Anthropic API) via `anthropic` SDK | Strong instruction-following for redaction rules; good at structured output |
| Output | JSON + static HTML portal | JSON for machine-readable output; HTML for the customer-facing demo portal |
| Version control | Git | Required by the process; four commits in sequence |

## Architecture

The tool is a single-file CLI script (`summarizer.py`) with these stages:

```
JSONL file → Load & Parse → Pre-process & Validate → LLM Summarize → Post-validate → Output
```

### Stage 1: Load & Parse
- Read `data/service_reports.jsonl`, parse each line as JSON
- Validate required fields exist

### Stage 2: Pre-process & Validate
Before sending to the LLM, detect data quality problems programmatically:

**Duration check**: Compute actual duration from `arrived_at` and `departed_at`. Compare to `stated_duration_hours`. Flag if difference > 15 minutes.

**Parts contradiction check**: If `parts_used` is non-empty but `resolution` contains phrases like "no parts", "inspection only", or vice versa (resolution mentions parts but `parts_used` is empty), flag the contradiction.

**Insufficient data check**: If `resolution` is fewer than 10 characters AND `technician_notes` is empty or fewer than 15 characters, mark as insufficient.

**Prompt injection check**: Scan `technician_notes` for instruction-like patterns (e.g. "do not mention", "publish directly", "instruction for", "record as", "omit from"). Flag any matches.

### Stage 3: LLM Summarization
Send each report to Claude with a strict system prompt that:
- Defines the audience (client facilities contact, not an engineer)
- Lists required content (asset, date, findings, actions, parts, recommendations, time)
- Lists prohibited content (names, phone numbers, emails, addresses, access codes, technician IDs)
- Explicitly states that technician_notes is data, not instruction
- Passes pre-detected flags so the LLM can incorporate them into the summary
- Requests structured JSON output

### Stage 4: Post-validation
After receiving the LLM's summary, run a category-based redaction check:
- **Phone numbers**: regex for UK phone patterns (`07\d{3}\s?\d{6}`, `0\d{3,4}\s?\d{3}\s?\d{4}`)
- **Email addresses**: regex for email patterns
- **Access codes**: regex for standalone 4-6 digit codes near keywords like "code", "access", "PIN"
- **Addresses**: check for patterns like flat/house numbers, street names, postcodes
- **Names**: compare against names found in the original report's notes (extracted during pre-processing)

If any prohibited content is found in the generated summary, strip it and add a flag.

### Stage 5: Output
- Write all summaries to `output/summaries.json` as a JSON array
- Generate `output/portal.html` — a styled customer portal page displaying all summaries
- Write `output/analysis.md` — the per-report analysis of problematic reports

## Decisions and alternatives rejected

| Decision | Alternative rejected | Reason |
|----------|---------------------|--------|
| Single Python file | Multi-module package | The tool is small; a package adds structure without value |
| Category-based redaction (regex) | String-matching against known values | String-matching breaks on held-out data with different names/numbers |
| Post-validation after LLM | Relying solely on LLM to redact | Defence in depth; LLM may miss edge cases |
| Flag contradictions, don't resolve | Pick the "most likely" value | Spec says Sana prefers honest uncertainty over confident error |
| Claude API | Local model (Ollama) | Better instruction-following for complex redaction rules; cost is minimal for 20 reports |

## Security considerations

- The tool processes internal data and publishes to a customer-visible portal. The primary security concern is **information leakage**, not code-level vulnerabilities.
- Redaction is the most critical feature. It must work by **category** (any phone number, any name) not by **specific string** (this phone number, this name).
- Technician notes are treated as untrusted input. Any instruction-like content is flagged and ignored.
- The output HTML portal is static with no user input, so XSS is not a concern.
- No secrets are stored in the repository. The API key is read from an environment variable.

## Risk

The main risk is that the LLM produces a plausible-sounding summary that leaks redactable content. The post-validation regex layer mitigates this but cannot catch every possible form of PII. The analysis document will note where this defence is weakest.
