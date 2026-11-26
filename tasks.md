# Tasks: Field Service Report Summarizer

Ordered list of individually completable tasks for the implementation phase.

## Task 1: Project setup
- Create `requirements.txt` with dependencies (`anthropic`)
- Create `.env.example` showing required environment variables
- Create `.gitignore` for Python and environment files

## Task 2: Report loader
- Read `data/service_reports.jsonl` and parse each line as a JSON object
- Validate that each report has the required fields
- Return a list of report dictionaries

## Task 3: Pre-processing and validation
- **Duration check**: Compute actual duration from `arrived_at` and `departed_at`. Compare to `stated_duration_hours`. Flag if difference exceeds 15 minutes.
- **Parts contradiction check**: Detect when `parts_used` is non-empty but `resolution` indicates no parts (or vice versa).
- **Insufficient data check**: Flag reports where `resolution` is fewer than 10 characters and `technician_notes` is empty or fewer than 15 characters.
- **Prompt injection check**: Scan `technician_notes` for instruction-like patterns. Flag any matches.
- Attach flags to each report dictionary.

## Task 4: LLM summarizer
- Build the system prompt encoding all rules from spec.md (audience, required content, prohibited content, notes-as-data).
- Build the per-report user prompt, including the report data and any pre-detected flags.
- Call Claude API for each report.
- Parse the structured response.
- Handle API errors gracefully (retry once, then mark as failed).

## Task 5: Post-validation redaction check
- Apply regex patterns against each generated summary to catch:
  - UK phone numbers
  - Email addresses
  - Standalone numeric codes near access-related keywords
  - Street addresses / flat references / postcodes
  - Names extracted from the original technician_notes
- If any match is found, strip it from the summary and add a flag.

## Task 6: Output generation
- Write all summaries to `output/summaries.json` as a formatted JSON array.
- Generate `output/portal.html` — a styled, static customer portal page showing each summary as a card.
- Write `output/analysis.md` — a per-report analysis covering all flagged reports, what was found, and what the tool did.

## Task 7: End-to-end run and manual review
- Run the tool against all 20 reports.
- Read every generated summary against the requirements.
- Specifically verify:
  - FSR-3003 and FSR-3014: zero PII in output
  - FSR-3005 and FSR-3006: contradictions flagged
  - FSR-3007 and FSR-3008: marked as insufficient
  - FSR-3009: prompt injection ignored, instruction attempt flagged
  - FSR-3011: recommendations at the end of the long report survive in output

## Task 8: AI output review
- Review all generated code across five lenses:
  1. **Intent**: Does the code do what the spec says?
  2. **Tests**: Are there tests, and do they cover edge cases?
  3. **Security**: Does the tool ever leak PII or access codes in its output?
  4. **Performance**: Is the tool efficient enough for the 20-report dataset?
  5. **Maintainability**: Is the code readable and structured for future changes?
- Document at least one real issue found and corrected.
- Write findings to `review.md`.
