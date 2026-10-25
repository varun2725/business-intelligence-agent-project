# Business Intelligence Agent Project

A 3-agent pipeline that turns a sales CSV into a plain-English weekly business report.

CSV -> Data Fetcher -> Insight Analyser -> Report Writer -> `weekly_report.md`

| Agent | Owner | File |
|---|---|---|
| Data Fetcher | Balveer | `agents/data_fetcher.py` (temporary stand-in for now) |
| Insight Analyser | Elava | `agents/insight_analyser.py` (temporary stand-in for now) |
| Report Writer | Varun S | `agents/report_writer.py` |

## Setup

```
pip install -r requirements.txt
```
Create a `.env` file containing `GEMINI_API_KEY=your-key`.

## Run

```
python make_sample_data.py        # creates sales.csv with planted anomalies
python main.py sales.csv          # mock LLM, no key needed
python main.py sales.csv live     # real Gemini report
streamlit run app.py              # dashboard with charts and report
```

## Data contract

- Fetcher output: `date_range, weekly_totals, daily_totals, by_category, raw_stats`
- Analyser output: `metrics, trends, anomalies`
- Writer output: `weekly_report.pdf`

Anomalies are found with statistics in code; the LLM only writes the report.