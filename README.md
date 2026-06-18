# 📊 Supplier Quotation Analyzer

Automated AI-powered tool that extracts structured data from supplier quotation PDFs, computes 4-year Total Cost of Ownership (TCO), scores and ranks suppliers, and delivers a reasoned recommendation — all in a Streamlit web UI.

Built for AUMOVIO as part of the LLM Innovation Intern Task.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Python | 3.10+ |
| [uv](https://github.com/astral-sh/uv) | Any recent version (used by Makefile) |
| Google Gemini API key | Free tier works |

> **Note:** The `Makefile` uses `uv pip` for installation. If you prefer plain pip, replace `uv pip install` with `pip install` in the install step.

---

## Setup

### 1. Clone the repository

```bash
cd <file-folder>
```

### 2. Create and activate a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate      # macOS / Linux
.venv\Scripts\activate         # Windows
```

### 3. Install dependencies

```bash
make install
# or without uv:
pip install -r requirements.txt
```

### 4. Configure your API key

Create a `.env` file in the project root:

```bash
GOOGLE_API_KEY=your_gemini_api_key_here
```

Get a free key at [https://aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey).

### 5. Add your supplier PDFs

Place all supplier quotation PDFs into a folder (default: `sample/`):

```
sample/
  supplier_a.pdf
  supplier_b.pdf
  supplier_c.pdf
  ...
```

---

## Running the App

### Option A — Streamlit Web UI (recommended)

```bash
make run
# or:
streamlit run app.py
```

Open [http://localhost:8501](http://localhost:8501) in your browser.

**In the sidebar:**
1. Set the **PDF folder path** (default: `sample/`)
2. Choose the **reference year**, **TCO currency**, and **USD→EUR rate**
3. Click **🔍 Run Analysis**

### Option B — CLI (prints raw JSON)

```bash
make pipeline
# or:
python analyze.py
```

Outputs the full extraction + analysis result as JSON to stdout.

---

## Project Structure

```
.
├── app.py              # Streamlit UI
├── analyze.py          # Extraction & analysis pipeline
├── requirements.txt    # Python dependencies
├── Makefile            # Developer shortcuts
├── .env                # API key (not committed)
└── sample/             # Place your supplier PDFs here
```

---

## How It Works

### Step 1 — Extraction (`analyze.py: extract_all_quotes`)

Each PDF is base64-encoded and sent directly to **Gemini 2.5 Flash** as native PDF input. The model extracts a structured `SupplierQuote` Pydantic schema via LangChain's `with_structured_output()`.

Key fields extracted per supplier:

- Supplier name, quote reference, date, part name
- Annual price table (year, unit price, volume) — 2027–2030
- Tooling cost & NRE / development cost (one-time)
- MOQ, delivery terms, payment deadlines
- Production lead time, tooling setup time, packaging cost
- Notes & conditions, validity days

### Step 2 — Analysis (`analyze.py: compare_and_recommend`)

All extracted quotes are serialized to JSON and sent to a second LLM call that computes:

| Criterion | Description | Max Score |
|---|---|---|
| Price Decline Rate | % price reduction 2027→2030 | 10 |
| 4-Year TCO | Lowest total cost (parts + tooling + NRE), in EUR | 10 |
| Production Lead Time | Shortest lead time in weeks | 10 |
| Payment Terms | Longest payment deadline (cash flow benefit) | 10 |
| Data Completeness | Fewest missing fields | 10 |

**TCO formula:** `Σ(unit_price × volume per year) + tooling_cost + NRE_cost`  
Total score (0–50) is scaled to 0–100 for the supplier scorecard.

### Step 3 — UI (`app.py`)

**Tab 1 — Extraction:** Each supplier shown with the original PDF (embedded viewer) alongside extracted fields for human verification.

**Tab 2 — Comparison & Recommendation:**
- Price Breakdown & TCO table
- Delivery Timeline comparison
- Scoring Breakdown (per-criterion)
- Supplier Scorecards (strengths + risk flags)
- Unit Price Trend chart (2027–2030)
- Recommended winner with reasoning and key insight

---

## Makefile Commands

```bash
make help       # Show all available commands
make install    # Install dependencies via uv pip
make run        # Launch Streamlit app
make pipeline   # Run CLI analysis and print JSON
make clean      # Remove __pycache__ and .pyc files
```

---

## Configuration

All runtime settings are adjustable in the Streamlit sidebar — no code changes needed:

| Setting | Default | Description |
|---|---|---|
| PDF folder path | `sample/` | Where your supplier PDFs live |
| Reference year | `2027` | Year used for unit price comparison |
| TCO currency | `EUR` | Output currency for all cost calculations |
| 1 USD = ? EUR | `0.92` | Exchange rate for USD→EUR conversion |

---

## Key Design Decisions

**Direct PDF input over RAG:** Sending PDFs as binary to Gemini preserves table structure. RAG/chunking was abandoned because it broke price tables and caused missed fields.

**Structured output over JSON parsing:** `model.with_structured_output(SupplierQuote)` delegates all parsing to LangChain + Pydantic — no manual JSON parsing, no crashes on malformed output.

**Gemini 2.5 Flash:** Chosen for its free tier, native PDF understanding, and reliable structured output. An Ollama fallback is retained in the code (commented out) but is not recommended — it cannot accept PDF binary input and produces unreliable structured output.

---

## Troubleshooting

**`FileNotFoundError: No PDF files found in sample/`**  
→ Make sure your PDFs are in the folder shown in the sidebar and the path ends with `/`.

**`GOOGLE_API_KEY` errors**  
→ Check that your `.env` file is in the project root and the key is valid.

**Gemini quota exceeded**  
→ The free tier has rate limits. Wait a few minutes and retry, or upgrade to a paid plan.

**`uv` not found**  
→ Install uv (`pip install uv`) or use `pip install -r requirements.txt` directly.

**Supplier name shows as filename**  
→ The LLM fell back to filename-based extraction. This is expected for ambiguous PDF layouts and is handled gracefully.
