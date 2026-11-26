# Field Service Report Summarizer

**L1 Capstone — Case Study 03 | Perficient Global AI-First Academy**

A CLI tool that converts internal engineer field-service reports into client-ready summaries for a customer portal. Built using the spec-first four-commit process required by the L1 AI-Augmented SDLC certification.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Running the Tool](#running-the-tool)
- [Viewing the Output](#viewing-the-output)
- [How It Works](#how-it-works)
- [Four-Commit Process](#four-commit-process)
- [Key Design Decisions](#key-design-decisions)
- [Evaluation Dimensions Covered](#evaluation-dimensions-covered)

---

## Overview

Northgate FM sends engineers to client sites and publishes a summary of every visit to the client's portal. This tool automates that process by:

- Reading 20 structured field-service reports from a JSONL file
- Detecting data quality issues (inconsistencies, insufficient data, prompt injection attempts)
- Generating plain-language summaries using Claude (Anthropic API) via a Portkey gateway
- Redacting all personal information, access codes, and internal identifiers
- Producing a styled HTML customer portal, JSON output, and an analysis of flagged reports

## Project Structure

```
field-service-summarizer/
├── data/
│   ├── service_reports.jsonl      # 20 synthetic field-service reports (input)
│   └── summary_requirements.md    # What the customer summary must/must not contain
├── output/
│   ├── summaries.json             # Generated summaries for all 20 reports
│   ├── portal.html                # Customer-facing portal page
│   └── analysis.md                # Analysis of all flagged/insufficient reports
├── spec.md                        # Specification (committed before code)
├── plan.md                        # Implementation plan
├── tasks.md                       # Task decomposition
├── review.md                      # AI output review across 5 lenses
├── summarizer.py                  # Main CLI tool
├── requirements.txt               # Python dependencies
├── .env.example                   # Environment variable template
└── .gitignore
```

## Prerequisites

- **Python 3.11 or higher**
- **An Anthropic API key** (direct or via a Portkey gateway)
- **Git** (to verify the four-commit history)
- **pip** (Python package manager)

## Setup

### Step 1: Clone the repository

```bash
git clone https://github.com/rajlanjewar01/Capstone-L1-Case_Study_3_Field-Service-Report-Summarizer.git
cd Capstone-L1-Case_Study_3_Field-Service-Report-Summarizer
```

### Step 2: Create a virtual environment (recommended)

**Windows:**
```bash
python -m venv venv
venv\Scripts\activate
```

**macOS / Linux:**
```bash
python3 -m venv venv
source venv/bin/activate
```

### Step 3: Install dependencies

```bash
pip install -r requirements.txt
```

### Step 4: Configure environment variables

Copy the example file and fill in your credentials:

```bash
cp .env.example .env
```

Edit `.env` with your values:

```env
ANTHROPIC_API_KEY=your-api-key-here
PORTKEY_BASE_URL=https://your-portkey-gateway-url
LLM_MODEL=us.anthropic.claude-sonnet-4-6
PORTKEY_PROVIDER=@your-portkey-provider
```

**Configuration options:**

| Variable | Description | Default |
|----------|-------------|---------|
| `ANTHROPIC_API_KEY` | Your API key (Anthropic or Portkey) | *required* |
| `PORTKEY_BASE_URL` | Gateway URL | `https://api.anthropic.com` |
| `LLM_MODEL` | Model ID to use | `us.anthropic.claude-sonnet-4-6` |
| `PORTKEY_PROVIDER` | Portkey provider header | `@aws-bedrock-use2` |

**If using a direct Anthropic API key** (starts with `sk-ant-`), you only need:

```env
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

The tool defaults to `https://api.anthropic.com` when no gateway URL is set.

## Running the Tool

### Step 5: Run the summarizer

```bash
python summarizer.py
```

You will see output like:

```
Using gateway: https://api.anthropic.com
Using model: us.anthropic.claude-sonnet-4-6
Using provider: @aws-bedrock-use2
Loading reports from data/service_reports.jsonl...
Loaded 20 reports.

[1/20] Processing FSR-3001 (Chiller CH-04)... -> OK
[2/20] Processing FSR-3002 (AHU-11)... -> OK
[3/20] Processing FSR-3003 (Boiler BLR-02)... -> OK
...
[20/20] Processing FSR-3020 (Pump P-03)... -> OK

--- Summary ---
  Clean: 15  |  Flagged: 3  |  Insufficient: 2  |  Errors: 0
```

The tool processes all 20 reports and writes three output files.

## Viewing the Output

### Step 6: Open the customer portal

Open `output/portal.html` in your browser:

**Windows:**
```bash
start output\portal.html
```

**macOS:**
```bash
open output/portal.html
```

**Linux:**
```bash
xdg-open output/portal.html
```

The portal displays each service visit summary as a card with status badges:
- **Clean** (green) — summary generated normally
- **Flagged** (amber) — summary has warnings (data inconsistencies, redacted content, prompt injection)
- **Insufficient** (red) — not enough data for a meaningful summary

### Step 7: Review other outputs

| File | Description |
|------|-------------|
| `output/summaries.json` | All 20 summaries as a JSON array with status, flags, and metadata |
| `output/portal.html` | Styled customer-facing portal page |
| `output/analysis.md` | Detailed analysis of every flagged or insufficient report |

## How It Works

The tool runs a five-stage pipeline for each report:

```
JSONL File → Load & Parse → Pre-process & Validate → LLM Summarize → Post-validate → Output
```

### Stage 1: Load & Parse
Reads `data/service_reports.jsonl` and parses each line as JSON.

### Stage 2: Pre-process & Validate
Detects problems before sending to the LLM:
- **Duration mismatch** — stated hours vs. computed arrival/departure time (> 15 min difference)
- **Parts contradiction** — parts listed but resolution says "no parts" (or vice versa)
- **Insufficient data** — resolution is minimal and notes are empty or reference external documents
- **Prompt injection** — notes contain instruction-like text trying to manipulate the tool

### Stage 3: LLM Summarization
Sends each report to Claude with a strict system prompt that:
- Defines the audience (client facilities contact, plain language)
- Lists required content (asset, date, findings, actions, parts, recommendations, time)
- Lists prohibited content (names, phones, emails, addresses, access codes, technician IDs)
- States that technician notes are data, not instructions

### Stage 4: Post-validation
Applies category-based regex checks against each generated summary to catch any PII the LLM might have missed:
- UK phone number patterns
- Email addresses
- Access codes near related keywords
- Street addresses and postcodes
- Personal names extracted from the original notes

### Stage 5: Output
Writes `summaries.json`, `portal.html`, and `analysis.md` to the `output/` directory.

## Four-Commit Process

The project follows the required four-commit spec-first sequence. Verify with:

```bash
git log --oneline
```

```
84697ab Add outputs, review, and post-implementation fixes
9cfe440 04-implement
d4cc549 03-tasks
695005c 02-plan
c13447b 01-spec
578e718 Initial data: add service reports and summary requirements
```

| Commit | Contains | No Source Files Before 04-implement |
|--------|----------|-------------------------------------|
| `01-spec` | `spec.md` only | Verified |
| `02-plan` | adds `plan.md` | Verified |
| `03-tasks` | adds `tasks.md` | Verified |
| `04-implement` | `summarizer.py`, `requirements.txt`, `.env.example`, `.gitignore` | First source code |

To see full commit details with timestamps:

```bash
git log --stat --format="commit %H%nDate: %ai%n%n    %s%n"
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Flag contradictions, don't resolve** | Spec says Sana prefers honest uncertainty over a confident error |
| **Category-based redaction** | Regex by category (any phone, any name) not by specific strings — holds on unseen data |
| **Defence-in-depth** | LLM instructions + post-validation regex; neither layer alone is sufficient |
| **Notes as data, not instruction** | Prompt injection attempts are flagged and ignored |
| **Insufficient = refuse to summarize** | A confident summary from nothing is the worst failure mode |

## Evaluation Dimensions Covered

| Dimension | Evidence |
|-----------|----------|
| **Spec-Driven Development** | Four commits in order; spec written before code; git history proves sequence |
| **Context Engineering** | Structured system prompt with explicit rules; pre-detected flags passed as context |
| **AI Output Review** | `review.md` covers all 5 lenses; real issues found and corrected |
| **RAG Pipeline** | N/A for this case study |
| **n8n + AI Automation** | N/A for this case study |

---

**Perficient Global AI-First Academy** | L1 AI-Augmented SDLC Certification
