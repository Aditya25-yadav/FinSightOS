# 📊 FinSight OS
### Agentic Financial Intelligence Platform

> A production-grade multi-agent system that fuses company annual reports, live stock market data, financial news sentiment, and competitor benchmarks into a structured investment intelligence brief — complete with sentiment-vs-price divergence charts and interactive Plotly dashboards.

![Python](https://img.shields.io/badge/Python-3.11-blue) ![LangGraph](https://img.shields.io/badge/LangGraph-0.2-green) ![FastAPI](https://img.shields.io/badge/FastAPI-0.111-teal) ![React](https://img.shields.io/badge/React-18-blue) ![FinBERT](https://img.shields.io/badge/FinBERT-NLP-purple) ![Docker](https://img.shields.io/badge/Docker-ready-blue)

---

## 📌 Table of Contents
- [What This Project Does](#what-this-project-does)
- [Live Demo](#live-demo)
- [System Architecture](#system-architecture)
- [Agent Design](#agent-design)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Learning Roadmap](#learning-roadmap)
- [Build Plan (Week by Week)](#build-plan-week-by-week)
- [Setup & Installation](#setup--installation)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [Financial Metrics Reference](#financial-metrics-reference)
- [Evaluation & Testing](#evaluation--testing)
- [Deployment](#deployment)
- [Disclaimer](#disclaimer)
- [What I Learned](#what-i-learned)
- [Future Improvements](#future-improvements)

---

## What This Project Does

Most finance tools show you data. This one *reasons* about it.

Enter a company ticker like **RELIANCE.NS** and optionally upload its annual report PDF. The system:

1. **Analyses** the annual report — extracts revenue trends, margins, debt, cash flow, management risks
2. **Monitors** live market data — calculates P/E, ROE, EV/EBITDA, compares to 2-year price history
3. **Reads the news** — runs FinBERT sentiment analysis on 30 days of headlines, builds a sentiment timeline
4. **Benchmarks** against competitors — side-by-side ratio comparison with colour-coded performance flags
5. **Synthesises** a structured brief — Bull case, Bear case, Key risks, with an LLM reasoning over all four data sources

The killer output: a **Plotly chart overlaying news sentiment against stock price** — showing divergences that often precede major price moves. Everything is wrapped in a deployed React dashboard with live agent activity updates.

---

## Live Demo

🔗 [Live App — Render/Vercel](https://your-app.onrender.com) *(update after deployment)*

![Demo GIF — replace with actual recording](docs/demo.gif)

**Try it with:** `RELIANCE.NS`, `TCS.NS`, `INFY.NS` (NSE Indian stocks via yfinance)
Or US stocks: `AAPL`, `MSFT`, `NVDA`

---

## System Architecture

```
React UI (Ticker input + PDF upload + Competitor tickers)
        │
        ▼
FastAPI Gateway
        │
        ▼
LangGraph Orchestrator ──────────────────────── SSE Stream → React AgentFeed
        │
   ┌────┴──────────────────────────────────┐
   │           PARALLEL EXECUTION          │
   ▼                                       ▼
PDFAgent                            MarketAgent
│  ├── PyMuPDF PDF parsing           │  ├── yfinance price history (2yr)
│  ├── Section extraction            │  ├── Financial ratio calculation
│  ├── ChromaDB storage              │  ├── Trend detection
│  └── RAG over financials           │  └── Price momentum scoring
   │                                       │
   └──────────────┬────────────────────────┘
                  │
         ┌────────┴────────┐
         ▼                 ▼
  SentimentAgent    CompetitorAgent
  │  ├── NewsAPI     │  ├── yfinance × N tickers
  │  ├── FinBERT     │  ├── Same ratio calc
  │  ├── Timeline    │  └── Comparison dict
  │  └── Divergence  │
         │                 │
         └────────┬─────────┘
                  │
                  ▼
          Fusion + LLM Synthesis (Groq)
          │  ├── Bull case / Bear case / Risks
          │  ├── Structured JSON brief
          │  └── Chart data for Plotly
                  │
                  ▼
          React Dashboard
          ├── Plotly: price + sentiment overlay chart
          ├── Ratio comparison table (colour-coded)
          ├── LLM brief in cards
          ├── Competitor radar chart
          └── PDF export (jsPDF)
```

---

## Agent Design

### Agent 1 — PDFAgent
**Responsibility:** Extract structured financial intelligence from the annual report.

- Accepts PDF upload or auto-downloads from company investor relations page (if URL known)
- Parses with PyMuPDF — detects sections: MD&A, Risk Factors, Financial Statements
- Extracts: Revenue (3yr), EBITDA, Net Profit, Total Debt, Cash & Equivalents, Capex
- Identifies management commentary sentiment (forward guidance language)
- Stores full report in ChromaDB for RAG queries
- RAG: "What does management say about their debt reduction plan?" → retrieves, LLM summarises
- Output: `{financials_3yr, key_ratios_from_report, risk_factors[], guidance_statements[], mgmt_tone}`

### Agent 2 — MarketAgent
**Responsibility:** Analyse live and historical market data.

- Fetches 2-year daily OHLCV data via `yfinance`
- Calculates ratios programmatically:
  - P/E = Current Price / TTM EPS
  - ROE = Net Income / Shareholders' Equity
  - Debt-to-Equity = Total Debt / Total Equity
  - EV/EBITDA = (Market Cap + Debt - Cash) / EBITDA
  - Current Ratio = Current Assets / Current Liabilities
- Detects: price trend (50-day vs 200-day MA crossover), earnings vs price divergence
- Computes a "valuation score": is the stock cheap/fair/expensive vs its 2yr history?
- Output: `{ratios{}, price_history[], trend_signal, valuation_score, moving_averages{}}`

### Agent 3 — SentimentAgent
**Responsibility:** Build a news sentiment timeline and detect divergences from price.

- Fetches last 30 days of news headlines via NewsAPI (free tier: 100 req/day)
- Each headline scored by `ProsusAI/finbert` → positive / negative / neutral + confidence
- Aggregates to daily sentiment score (weighted avg of headline scores for that day)
- **Divergence detection:** Pearson correlation between sentiment and 3-day-lagged price change
- Flags days where sentiment and price moved in opposite directions
- Output: `{daily_sentiment[], correlation_score, divergence_events[], sentiment_trend}`

### Agent 4 — CompetitorAgent
**Responsibility:** Benchmark the company against 2–3 competitors.

- Receives competitor tickers from user
- Runs same ratio calculations (from MarketAgent) for each competitor
- Builds a comparison matrix: company vs each competitor for each ratio
- Flags: outperforming ratios (green), lagging ratios (red), within 10% (yellow)
- Calculates relative valuation: is this company cheap vs peers?
- Output: `{comparison_matrix{}, relative_valuation, outperforming[], lagging[]}`

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Agent orchestration | LangGraph 0.2 | Parallel agent fan-out, shared state |
| LLM inference | Groq (llama-3.1-70b) | Free, fast, great at structured output |
| Financial NLP | ProsusAI/FinBERT | Pretrained on financial text, much better than general BERT |
| Market data | yfinance | Free Yahoo Finance wrapper, covers NSE + global |
| News data | NewsAPI | Free tier 100 req/day, good for headlines |
| Embeddings | sentence-transformers | Free, local, no API cost |
| Vector store | ChromaDB | Stores annual report for RAG |
| PDF parsing | PyMuPDF (fitz) | Fastest, handles messy annual report layouts |
| Backend | FastAPI + uvicorn | Async, SSE support, fast |
| Frontend | React 18 + Vite | Fast development |
| Charts | Plotly (React) | Interactive financial charts, built-in candlestick |
| Styling | Tailwind CSS | Utility-first |
| PDF export | jsPDF + html2canvas | Client-side PDF generation |
| CI/CD | GitHub Actions | Auto-deploy |
| Deployment | Render + Vercel | Free tier |

---

## Project Structure

```
finsight-os/
│
├── backend/
│   ├── main.py                          # FastAPI entry point
│   ├── requirements.txt
│   ├── Dockerfile
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── pdf_agent.py                 # Annual report parsing + RAG
│   │   ├── market_agent.py              # yfinance + ratio calculations
│   │   ├── sentiment_agent.py           # NewsAPI + FinBERT pipeline
│   │   └── competitor_agent.py          # Multi-ticker benchmarking
│   │
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── workflow.py                  # LangGraph state machine
│   │   ├── state.py                     # FinSightState TypedDict
│   │   └── nodes.py                     # Node wrappers + parallel fan-out
│   │
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── embedder.py
│   │   ├── vector_store.py
│   │   └── chunker.py
│   │
│   ├── finance/
│   │   ├── __init__.py
│   │   ├── ratios.py                    # All financial ratio calculations
│   │   ├── market_data.py               # yfinance wrapper + data cleaning
│   │   └── divergence.py               # Sentiment-price correlation logic
│   │
│   ├── nlp/
│   │   ├── __init__.py
│   │   ├── finbert.py                   # FinBERT model wrapper
│   │   └── news_fetcher.py              # NewsAPI client
│   │
│   ├── pdf/
│   │   ├── __init__.py
│   │   └── parser.py                    # PyMuPDF annual report parser
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py
│   │   └── sse.py
│   │
│   ├── models/
│   │   ├── company.py                   # Company, Ratios, Sentiment Pydantic models
│   │   ├── brief.py                     # InvestmentBrief, ChartData models
│   │   └── state.py                     # LangGraph state model
│   │
│   └── tests/
│       ├── test_market_agent.py
│       ├── test_sentiment_agent.py
│       ├── test_ratios.py
│       └── test_finbert.py
│
├── frontend/
│   ├── package.json
│   ├── vite.config.js
│   ├── Dockerfile
│   │
│   ├── src/
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   │
│   │   ├── components/
│   │   │   ├── TickerInput.jsx           # Ticker + competitor input form
│   │   │   ├── PDFUpload.jsx             # Drag-and-drop PDF uploader
│   │   │   ├── AgentFeed.jsx             # Live SSE agent activity
│   │   │   ├── charts/
│   │   │   │   ├── SentimentPriceChart.jsx  # THE key chart — overlay
│   │   │   │   ├── RatioRadarChart.jsx      # Competitor radar chart
│   │   │   │   └── PriceHistoryChart.jsx    # Candlestick + MA chart
│   │   │   ├── RatioTable.jsx            # Colour-coded ratio comparison
│   │   │   ├── InvestmentBrief.jsx       # Bull/Bear/Risk cards
│   │   │   └── ExportButton.jsx          # jsPDF export
│   │   │
│   │   ├── hooks/
│   │   │   ├── useSSE.js
│   │   │   └── useAnalysis.js
│   │   │
│   │   └── utils/
│   │       ├── chartUtils.js             # Plotly data formatting
│   │       └── formatUtils.js            # Currency, % formatting (INR/USD)
│   │
│   └── public/
│
├── docker-compose.yml
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── docs/
│   ├── architecture.md
│   ├── financial_metrics.md
│   └── demo.gif
│
└── README.md
```

---

## Learning Roadmap

### Phase 1 — Financial concepts (Week 0 — before coding)
**You cannot build a finance tool without understanding what you're calculating.**

What to learn:
- P/E ratio — what it means, high vs low, industry context
- ROE — why Warren Buffett loves this metric
- EV/EBITDA — enterprise valuation, how it differs from P/E
- Debt-to-Equity — why high debt is risky, how to read it
- Moving averages — 50-day vs 200-day, golden cross / death cross
- What an earnings surprise is and why it moves stock prices

**Resources:**
- [Investopedia — Financial Ratio Tutorial](https://www.investopedia.com/financial-ratios-4689817) (3 hours — read all ratio articles)
- [Zerodha Varsity — Fundamental Analysis](https://zerodha.com/varsity/module/fundamental-analysis/) (4 hours — Module 3 and 5 specifically)
- [Zerodha Varsity — Technical Analysis](https://zerodha.com/varsity/module/technical-analysis/) (Module 3 — Moving Averages)

**Checkpoint:** You can explain P/E, ROE, and EV/EBITDA to someone without looking them up.

---

### Phase 2 — yfinance + financial data (Week 1)
**What to learn:**
- yfinance: download price history, get financials, access info dict
- pandas: datetime indexing, rolling averages, percentage change
- How to calculate each ratio from raw yfinance data

**Resources:**
- [yfinance documentation](https://python-yfinance.readthedocs.io/) (1 hour)
- [Pandas time series — Real Python](https://realpython.com/pandas-plot-python/) (2 hours)
- Build a small script: for any ticker, print P/E, ROE, 50-day MA — this teaches you exactly where each number comes from

**Checkpoint:** Running `get_ratios("TCS.NS")` returns a dict of all 5 ratios correctly.

---

### Phase 3 — FinBERT + NewsAPI (Week 2)
**What to learn:**
- How BERT models work (at a conceptual level — attention, tokenisation, classification head)
- Why FinBERT is better than general sentiment models for finance
- HuggingFace pipeline API — `pipeline("text-classification", model="...")`
- NewsAPI — search endpoint, date filtering, source filtering

**Resources:**
- [FinBERT paper abstract](https://arxiv.org/abs/1908.10063) — read the abstract + introduction only (20 mins)
- [HuggingFace pipeline tutorial](https://huggingface.co/docs/transformers/pipeline_tutorial) (1 hour)
- [NewsAPI documentation](https://newsapi.org/docs/endpoints/everything) (30 mins)

**Checkpoint:** Script that fetches 20 headlines about "Reliance Industries" and prints the FinBERT sentiment score for each.

---

### Phase 4 — LangGraph parallel execution (Week 3)
**What to learn:**
- LangGraph fan-out: running multiple nodes in parallel
- How to merge parallel outputs back into a single state
- Async execution in LangGraph
- LLM structured output — how to get an LLM to always return valid JSON

**Resources:**
- [LangGraph — How to create branches for parallel node execution](https://langchain-ai.github.io/langgraph/how-tos/branching/) (2 hours)
- [LangGraph — Subgraphs](https://langchain-ai.github.io/langgraph/concepts/subgraphs/) (1 hour)
- [Groq structured output docs](https://console.groq.com/docs/structured-outputs) (30 mins)

**Checkpoint:** A LangGraph graph where MarketAgent and SentimentAgent run simultaneously (not sequentially) and their results merge into a single state dict.

---

### Phase 5 — Plotly React charts (Week 4–5)
**What to learn:**
- Plotly.js vs React-Plotly — use `react-plotly.js`
- Dual-axis charts — one y-axis for price, one for sentiment
- Candlestick charts — OHLC data format
- Radar/spider charts — for competitor comparison
- Conditional table formatting in React — green/red/yellow cells

**Resources:**
- [Plotly React documentation](https://plotly.com/javascript/react/) (2 hours)
- [Plotly — Dual axis charts](https://plotly.com/javascript/multiple-axes/) (1 hour)
- [React-Plotly GitHub](https://github.com/plotly/react-plotly.js) — look at the examples (1 hour)

**Checkpoint:** A standalone React component that renders a dual-axis Plotly chart with dummy price and sentiment data. This is the hardest chart in the project — get it working in isolation first.

---

### Phase 6 — PDF parsing of annual reports (Week 3, parallel with LangGraph)
**What to learn:**
- Annual report structure: MD&A section, Notes to Accounts, standalone vs consolidated P&L
- PyMuPDF: table extraction, section heading detection
- Handling messy formatting in real Indian company PDFs (they're notoriously messy)

**Resources:**
- Download a real annual report — Infosys FY2024 annual report is well-formatted and freely available on their investor relations page. Use it as your test case.
- [PyMuPDF — Working with tables](https://pymupdf.readthedocs.io/en/latest/page.html#Page.find_tables) (1 hour)

**Checkpoint:** Script that opens the Infosys annual report PDF and extracts the revenue and profit figures from the financial statements.

---

## Build Plan (Week by Week)

> Total time estimate: **6–7 weeks** at 2–3 hours per day.
> This project shares architecture with Agentic Research OS — if you build that first, Weeks 1 and 2 here go faster because you're reusing RAG + LangGraph patterns.

### Week 0 — Financial literacy (before coding)
**Goal:** Understand every metric you're going to calculate.

Tasks:
- [ ] Read all Zerodha Varsity — Fundamental Analysis (Module 3)
- [ ] Read Investopedia articles on: P/E, ROE, EV/EBITDA, D/E ratio, current ratio
- [ ] Create a `docs/financial_metrics.md` file with your own explanation of each metric
- [ ] Download the Infosys FY2024 annual report — read the MD&A section and find revenue manually
- [ ] Look up Reliance Industries on screener.in — understand what each number means on that page

**Why this matters:** In your internship interviews, you will be asked "what does P/E mean and what does a high P/E imply?" If you can't answer this, the finance-facing project loses all credibility.

---

### Week 1 — Market data pipeline
**Goal:** Working pipeline that fetches data and calculates all ratios for any ticker.

Tasks:
- [ ] Install: `yfinance`, `pandas`, `numpy`
- [ ] Write `finance/market_data.py` — download 2yr price history, handle NaN values, calculate daily returns
- [ ] Write `finance/ratios.py` — calculate P/E, ROE, EV/EBITDA, D/E, current ratio from yfinance `.info` dict
- [ ] Write `finance/divergence.py` — given a price series and a sentiment series, calculate 3-day lagged Pearson correlation
- [ ] Write `market_agent.py` — wraps all the above, returns structured JSON
- [ ] Test on 5 different tickers: 2 Indian (NSE), 2 US, 1 that doesn't exist (test error handling)
- [ ] Commit to GitHub

**Definition of done:** `market_agent.analyse("INFY.NS")` returns all ratios, price history, and trend signal without errors.

---

### Week 2 — FinBERT sentiment pipeline
**Goal:** Given a ticker, fetch 30 days of news and produce a daily sentiment timeline.

Tasks:
- [ ] Sign up for NewsAPI free key
- [ ] Install: `transformers`, `torch` (or `torch-cpu` to save space)
- [ ] Write `nlp/news_fetcher.py` — fetch headlines for a query, filter by date, deduplicate
- [ ] Write `nlp/finbert.py` — load `ProsusAI/finbert`, run batch inference on list of headlines, return scores
- [ ] Write `finance/divergence.py` — aggregate scores by day, calculate rolling average, compute price correlation
- [ ] Write `sentiment_agent.py` — orchestrates fetcher + FinBERT + divergence
- [ ] Test on Reliance Industries — plot sentiment vs price on paper to verify it looks sensible
- [ ] Add caching: store fetched headlines in a JSON file to avoid re-fetching during development

**Note on FinBERT:** First load will download ~440MB. After that it's cached locally.

**Definition of done:** `sentiment_agent.analyse("Reliance Industries", "RELIANCE.NS")` returns a 30-day sentiment timeline and a divergence score.

---

### Week 3 — PDF Agent + LangGraph wiring
**Goal:** Annual report parsing works; all 4 agents wired into parallel LangGraph graph.

Tasks:
- [ ] Write `pdf/parser.py` — section detection for Indian annual reports (search for "Management Discussion", "Risk Factors", "Standalone Statement of Profit")
- [ ] Extend `rag/` from Project 1 (or rewrite fresh) — same ChromaDB + embeddings setup
- [ ] Write `pdf_agent.py` — parse report, store in ChromaDB, RAG-query for key financials
- [ ] Write `competitor_agent.py` — takes list of tickers, runs MarketAgent ratios on each, builds comparison matrix
- [ ] Write `graph/state.py` — `FinSightState` TypedDict with fields for all 4 agents
- [ ] Write `graph/workflow.py` — PDFAgent + MarketAgent run in parallel (fan-out), then SentimentAgent + CompetitorAgent (second parallel step), then Fusion node
- [ ] Test full graph: `workflow.invoke({"ticker": "TCS.NS", "competitors": ["INFY.NS", "WIPRO.NS"]})`

**Definition of done:** Full graph runs end to end without errors. State object populated with all 4 agent outputs.

---

### Week 4 — LLM fusion + SSE backend
**Goal:** LLM generates a structured investment brief; frontend receives live updates.

Tasks:
- [ ] Write the fusion node: take all agent outputs, build a single context JSON, call Groq LLM
- [ ] Prompt the LLM to return structured JSON: `{bull_case: str, bear_case: str, key_risks: [], summary: str, confidence: float}`
- [ ] Write `api/sse.py` — async queue, each LangGraph node emits status events to the queue
- [ ] Write `api/routes.py` — `POST /analyse` (full result) and `GET /analyse/stream` (SSE)
- [ ] Test SSE with curl — verify events stream correctly
- [ ] Add `POST /upload-pdf` endpoint — accepts multipart file, saves to temp dir, passes path to PDFAgent

**Definition of done:** Curling the SSE endpoint streams agent statuses and ends with a full investment brief in the final event.

---

### Week 5 — React frontend + Plotly charts
**Goal:** Full UI with the sentiment-price chart, ratio table, and LLM brief.

Tasks:
- [ ] Set up React + Vite + Tailwind + react-plotly.js
- [ ] Build `TickerInput.jsx` + `PDFUpload.jsx` — form with drag-and-drop
- [ ] Build `AgentFeed.jsx` — SSE live updates (reuse pattern from Project 1)
- [ ] Build `SentimentPriceChart.jsx` — **dual-axis Plotly chart** (price line + sentiment bar, same x-axis)
  - Price on left y-axis (line trace)
  - Sentiment on right y-axis (bar trace)
  - Divergence events marked with vertical lines
- [ ] Build `RatioTable.jsx` — company vs competitor for each ratio, conditional background colour
- [ ] Build `PriceHistoryChart.jsx` — candlestick + 50-day and 200-day MA lines
- [ ] Build `InvestmentBrief.jsx` — Bull / Bear / Risks displayed as cards with icons
- [ ] Add `ExportButton.jsx` — uses `jsPDF` + `html2canvas` to export the visible dashboard as PDF
- [ ] Connect everything in `App.jsx`

**The sentiment-price chart is the centrepiece — spend extra time on this. Make the divergence annotations clearly visible.**

**Definition of done:** Full flow works: enter TCS.NS + INFY.NS + WIPRO.NS, see agents run, see charts render, see brief.

---

### Week 6 — Polish, Docker, deploy, eval
**Goal:** Deployed, live, tested, and documented.

Tasks:
- [ ] Write 4 unit tests per major module (ratios, FinBERT, divergence calculation, LLM output parsing)
- [ ] Add input validation: what if a ticker doesn't exist? What if the PDF has no text layer?
- [ ] Add a loading skeleton in React so the UI doesn't look blank while agents run
- [ ] Write `backend/Dockerfile` and `frontend/Dockerfile`
- [ ] Write `docker-compose.yml`
- [ ] Test locally with `docker-compose up --build`
- [ ] Deploy backend to Render, frontend to Vercel
- [ ] Run the app on 5 different companies, manually verify the brief makes sense
- [ ] Record demo GIF
- [ ] Fill in "What I Learned" section of README

**Definition of done:** Public URL works. Demo GIF is in the README. GitHub Actions deploys on push.

---

## Setup & Installation

### Prerequisites
- Python 3.11+
- Node.js 18+
- Docker Desktop
- [Groq API key](https://console.groq.com) (free)
- [NewsAPI key](https://newsapi.org) (free, 100 req/day)

### Local Development

```bash
# 1. Clone the repo
git clone https://github.com/yourusername/finsight-os.git
cd finsight-os

# 2. Backend setup
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 3. FinBERT model — first run downloads ~440MB, then cached
# No action needed — downloads automatically on first sentiment_agent call

# 4. Environment variables
cp .env.example .env
# Add your GROQ_API_KEY and NEWS_API_KEY

# 5. Run backend
uvicorn main:app --reload --port 8000

# 6. Frontend (new terminal)
cd frontend
npm install
npm run dev
```

### Docker

```bash
docker-compose up --build
# App at http://localhost:3000
```

---

## Environment Variables

```env
# backend/.env
GROQ_API_KEY=your_key_here
NEWS_API_KEY=your_key_here

CHROMA_PERSIST_PATH=./chroma_db
EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
FINBERT_MODEL=ProsusAI/finbert
LLM_MODEL=llama-3.1-70b-versatile

NEWS_LOOKBACK_DAYS=30
MAX_HEADLINES_PER_DAY=5
PRICE_HISTORY_YEARS=2
```

---

## API Reference

### `POST /analyse`
Full analysis (non-streaming).

```json
Request:
{
  "ticker": "TCS.NS",
  "competitors": ["INFY.NS", "WIPRO.NS"],
  "pdf_path": null          // optional — null if no PDF uploaded
}

Response:
{
  "session_id": "abc123",
  "brief": {
    "bull_case": "...",
    "bear_case": "...",
    "key_risks": ["...", "..."],
    "summary": "...",
    "confidence": 0.72
  },
  "charts": {
    "sentiment_price": { ... },   // Plotly trace data
    "ratio_comparison": { ... },
    "price_history": { ... }
  },
  "ratios": { ... },
  "sentiment_timeline": [ ... ]
}
```

### `GET /analyse/stream`
SSE streaming version. Query params: `ticker`, `competitors` (comma-separated).

```
data: {"agent": "MarketAgent", "status": "running", "message": "Fetching 2yr price history..."}
data: {"agent": "SentimentAgent", "status": "running", "message": "Running FinBERT on 87 headlines..."}
data: {"agent": "MarketAgent", "status": "done", "ratios": {...}}
...
data: {"type": "complete", "brief": {...}, "charts": {...}}
```

### `POST /upload-pdf`
Upload annual report PDF. Returns `pdf_path` to use in `/analyse`.

### `GET /health`
Health check.

---

## Financial Metrics Reference

A full explanation of every metric calculated — useful for interviews.

| Metric | Formula | What it means | Good range |
|---|---|---|---|
| P/E Ratio | Price / EPS | How much you pay per ₹1 of earnings | 15–25 for most sectors |
| ROE | Net Income / Shareholders' Equity | How efficiently equity generates profit | >15% is good |
| EV/EBITDA | (Mkt Cap + Debt - Cash) / EBITDA | Enterprise valuation multiple | 8–15 depends on sector |
| Debt-to-Equity | Total Debt / Total Equity | Financial leverage risk | <1 is generally safe |
| Current Ratio | Current Assets / Current Liabilities | Short-term liquidity | >1.5 is comfortable |

*See `docs/financial_metrics.md` for in-depth explanations with Indian market context.*

---

## Evaluation & Testing

### Unit Tests
```bash
cd backend
pytest tests/ -v --cov=. --cov-report=term-missing
```

### Manual Evaluation: Sentiment-Price Correlation
After running the app on 5 companies:
- [ ] Does the sentiment chart visually correlate with price direction?
- [ ] Do flagged divergences correspond to real events (earnings misses, news shocks)?
- [ ] Check: compare the flagged risk factors against the actual "Risk Factors" section in the PDF manually

Log results in `docs/eval_log.md` — this is impressive to show in interviews.

### Backtest (Optional but impressive)
For 3 companies, run the analysis as if it were 6 months ago (use historical data). Check:
- Did the Bull/Bear case turn out to be right?
- Did divergence events predict subsequent price moves?

Document the results honestly — even if wrong, showing you evaluated your model is the point.

---

## Deployment

### Backend → Render
1. New Web Service → connect GitHub → root: `backend/`
2. Build: `pip install -r requirements.txt`
3. Start: `uvicorn main:app --host 0.0.0.0 --port $PORT`
4. Add env vars: `GROQ_API_KEY`, `NEWS_API_KEY`
5. Free tier: 512MB RAM — FinBERT fits (~440MB). If it doesn't, use `distilbert-base-uncased-finetuned-sst-2-english` as a lighter fallback.

### Frontend → Vercel
1. Import repo → root: `frontend/`
2. Framework: Vite
3. Env var: `VITE_API_URL=https://your-render-app.onrender.com`

---

## Disclaimer

> **This tool is for educational and research purposes only. It does not constitute financial advice. All analysis is generated by AI models that can produce errors. Never make investment decisions based solely on this tool's output. Always consult a SEBI-registered financial advisor for investment decisions.**

This disclaimer is intentional — it shows awareness of responsible AI deployment in a regulated domain, which is something FinTech companies specifically look for in candidates.

---

## What I Learned

> *(Fill this in as you build — recruiters read this section carefully)*

- **FinBERT vs general sentiment** — why domain-specific pre-training matters: general BERT scored "merger announcement" as negative (neutral word "agreement"), FinBERT correctly scored it positive
- **yfinance quirks** — `.info` dict keys are inconsistent across tickers; built defensive fallbacks for every ratio
- **LangGraph parallel fan-out** — how to use `Send` nodes and merge outputs in a reducer function
- **Sentiment-price divergence** — using 3-day lagged correlation rather than same-day because news sentiment affects price over subsequent days, not immediately
- **Annual report parsing** — Indian company PDFs often have scanned sections; added a PyMuPDF OCR fallback
- **Responsible AI in finance** — why confidence scores and disclaimers are not optional decorations but architectural requirements

---

## Future Improvements

- [ ] Add support for screener.in as a data source for Indian stocks (more reliable than yfinance for Indian data)
- [ ] Insider trading signal: flag unusual volume spikes before major announcements
- [ ] Add a "portfolio mode" — analyse multiple holdings at once
- [ ] Integrate earnings call transcripts (from Project EarningsEdge) as a 5th data source
- [ ] Add email alerts: if sentiment divergence exceeds threshold, send an alert
- [ ] Replace NewsAPI with a dedicated financial news source (Moneycontrol RSS, ET Markets RSS)

---

## License

MIT — see [LICENSE](LICENSE)

---

*Built by [Your Name] — 3rd year CS student | [LinkedIn](https://linkedin.com) | [GitHub](https://github.com)*
