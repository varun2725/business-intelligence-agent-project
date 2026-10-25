# Business Intelligence Agent Project

A 3-agent pipeline that turns a raw sales CSV into a plain-English weekly business report. It ships with a Streamlit dashboard that is protected by Google sign-in (SSO), shows charts, and lets you download the finished report as a PDF.

**Author:** Varun S (designed, built and secured end to end)

```
CSV  ->  Data Fetcher  ->  Insight Analyser  ->  Report Writer  ->  Weekly report (Markdown / PDF)
```

---

## Table of contents

1. [Overview](#overview)
2. [Key features](#key-features)
3. [How it works](#how-it-works)
4. [Project structure](#project-structure)
5. [Tech stack](#tech-stack)
6. [Prerequisites](#prerequisites)
7. [Installation](#installation)
8. [Configuration](#configuration)
9. [Google sign-in (SSO) setup](#google-sign-in-sso-setup)
10. [Running the project](#running-the-project)
11. [Using the dashboard](#using-the-dashboard)
12. [Data contract](#data-contract)
13. [Input CSV format](#input-csv-format)
14. [Security](#security)
15. [Troubleshooting](#troubleshooting)
16. [Roadmap](#roadmap)

---

## Overview

Business teams often have sales data but no time to read it. This project automates the weekly review:

1. It reads a sales CSV.
2. It calculates totals, trends and unusual days using plain statistics.
3. It asks an LLM (Google Gemini) to turn those verified numbers into a readable report with trends, anomalies, possible causes and recommendations.

The key design choice is that **the numbers are never invented by the LLM**. All maths and anomaly detection happen in code. The LLM only writes the narrative around results it has been given, which keeps the report accurate and reduces hallucination.

---

## Key features

- **3-agent pipeline:** separate Data Fetcher, Insight Analyser and Report Writer agents with a clear data contract between them
- **Statistics in code, language from the LLM:** anomalies are found with statistical checks, not by the model
- **Live and mock modes:** use real Gemini output, or run a mock LLM that needs no API key (useful for demos and testing)
- **Interactive Streamlit dashboard:** upload a CSV, generate the report, and view charts for revenue by category and weekly revenue
- **PDF download:** export the generated report as `weekly_report.pdf` from the dashboard
- **Google sign-in (SSO):** only approved Google accounts can open the dashboard, using Streamlit's built-in authentication
- **Sample data generator:** creates a `sales.csv` with planted anomalies so the analyser always has something to find
- **Secrets kept out of git:** API keys and OAuth credentials live in git-ignored files

---

## How it works

### 1. Data Fetcher (`agents/data_fetcher.py`)
Loads the sales CSV and prepares the numbers the other agents need: the date range, weekly totals, daily totals, revenue by category and basic raw statistics (such as mean and standard deviation).

### 2. Insight Analyser (`agents/insight_analyser.py`)
Takes the fetcher output and works out what happened. It calculates headline metrics, identifies week-on-week trends and flags anomalous days using statistics in code (days that sit far from the daily average). It outputs `metrics`, `trends` and `anomalies`.

### 3. Report Writer (`agents/report_writer.py`)
Builds a prompt from the analyser's insights and sends it to the LLM. The model writes a report with sections such as Key Trends, Anomalies and Possible Causes, and Recommendations. The writer works with either the real Gemini model or the mock model from `test_writer.py`.

### 4. Orchestration
- `main.py` runs the full pipeline from the command line and saves `weekly_report.md`.
- `app.py` runs the same pipeline inside the Streamlit dashboard, adds charts, handles login and offers the PDF download.

---

## Project structure

```
bi-agent-report-writer/
├── agents/
│   ├── __init__.py
│   ├── data_fetcher.py        # Agent 1: reads CSV, builds totals and breakdowns
│   ├── insight_analyser.py    # Agent 2: metrics, trends, anomaly detection
│   └── report_writer.py       # Agent 3: prompt building and report generation
├── .streamlit/
│   └── secrets.toml           # OAuth settings (git-ignored, never commit)
├── app.py                     # Streamlit dashboard with Google login and PDF export
├── main.py                    # Command-line entry point
├── make_sample_data.py        # Generates sales.csv with planted anomalies
├── test_writer.py             # real_llm (Gemini) and mock_llm
├── sample_insights.json       # Example analyser output for testing the writer
├── sales.csv                  # Sample input data
├── requirements.txt           # Python dependencies
├── .env                       # GEMINI_API_KEY (git-ignored, never commit)
├── .gitignore
└── README.md
```

---

## Tech stack

| Area | Tools |
|---|---|
| Language | Python 3.12 |
| Dashboard | Streamlit (built-in `st.login` authentication) |
| Data and charts | pandas, Altair |
| LLM | Google Gemini via the `google-genai` SDK |
| Authentication | Google OAuth 2.0 / OpenID Connect through Streamlit and Authlib |
| PDF export | fpdf2 |
| Configuration | python-dotenv, Streamlit secrets |

---

## Prerequisites

- Python 3.12 (3.10 or higher should also work)
- Git
- A Google Gemini API key for live mode (not needed for mock mode)
- A Google account to create the OAuth client for sign-in
- Streamlit 1.42 or newer (required for the built-in login feature)

---

## Installation

1. Clone the repository:

```
git clone https://github.com/varun2725/business-intelligence-agent-project.git
cd business-intelligence-agent-project
```

2. Create and activate a virtual environment (Windows PowerShell):

```
python -m venv venv
.\venv\Scripts\Activate.ps1
```

If PowerShell blocks the activation script, run this once in the same window and activate again:

```
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

On macOS or Linux:

```
python3 -m venv venv
source venv/bin/activate
```

3. Install the dependencies:

```
pip install -r requirements.txt
```

The prompt should start with `(venv)` before you run any command below. Running from outside the virtual environment is the most common cause of "module not found" errors.

---

## Configuration

### Gemini API key

Create a file named `.env` in the project root:

```
GEMINI_API_KEY=your-key
```

The `.env` file is listed in `.gitignore`. Never commit it. Mock mode does not need this key.

---

## Google sign-in (SSO) setup

The dashboard requires login. Only Google accounts that you approve can open it.

### Step 1: Create the OAuth consent screen

1. Open [console.cloud.google.com](https://console.cloud.google.com) and select a project.
2. Search for **Google Auth Platform** and click **Get started**.
3. Fill in the app name (for example, `Business Intelligence Agent Project`), a support email and a contact email.
4. Choose **External** as the audience and accept the policy.
5. Open **Audience** and add every teammate's Gmail address under **Test users**. While the app is in testing mode, only these accounts can sign in.

Creating the OAuth client does not require a billing account or a payment card. If Google shows a free-trial banner, dismiss it.

### Step 2: Create the OAuth client

1. Open **Clients** and click **Create client**.
2. Choose **Web application** and give it a name.
3. Under **Authorized redirect URIs**, add:

```
http://localhost:8501/oauth2callback
```

4. Click **Create** and copy the **Client ID** and **Client secret** straight away. Google will not show the secret again.

### Step 3: Generate a cookie secret

```
python -c "import secrets; print(secrets.token_hex(32))"
```

### Step 4: Create the secrets file

Create `.streamlit/secrets.toml` and fill in your own values:

```toml
[auth]
redirect_uri = "http://localhost:8501/oauth2callback"
cookie_secret = "YOUR_COOKIE_SECRET"
client_id = "YOUR_CLIENT_ID.apps.googleusercontent.com"
client_secret = "YOUR_CLIENT_SECRET"
server_metadata_url = "https://accounts.google.com/.well-known/openid-configuration"
```

Each value must sit on a single line and end with a closing quote. A value split over two lines makes the file invalid and login will not work.

### Step 5: Match the port

The port in `redirect_uri` must match the port you run the app on, and the same URI must be listed in the Google client. To run on another port (for example 8503), update both places and start Streamlit with `--server.port 8503`.

### How the login works in `app.py`

```python
if not st.user.is_logged_in:
    st.title("Business Intelligence Agent Project")
    st.button("Sign in with Google", on_click=st.login)
    st.stop()

st.sidebar.write(f"Signed in as {st.user.email}")
st.sidebar.button("Log out", on_click=st.logout)
```

`st.stop()` halts the page for anyone who is not signed in, so nothing below the gate (uploads, charts, reports) is shown or executed for them.

---

## Running the project

Always run from inside the virtual environment.

```
python make_sample_data.py        # creates sales.csv with planted anomalies
python main.py sales.csv          # mock LLM, no key needed, saves weekly_report.md
python main.py sales.csv live     # real Gemini report
python -m streamlit run app.py    # dashboard with login, charts and PDF report
```

To run the dashboard on a specific port:

```
python -m streamlit run app.py --server.port 8501
```

Open `http://localhost:8501`, click **Sign in with Google** and choose an approved account.

---

## Using the dashboard

1. **Sign in** with an approved Google account. Your email appears in the sidebar with a **Log out** button.
2. **Upload a CSV** in the sidebar (`date, category, revenue`). If you upload nothing, the app falls back to `sales.csv`.
3. **Choose the model mode** with the **Use Gemini (live)** checkbox:
   - Ticked: the report is written by Gemini
   - Unticked: the report is written by the mock LLM (instant, no API needed)
4. Click **Generate weekly report**. The agents run in order and the app shows a spinner while they work.
5. **Review the output:**
   - Revenue by category (bar chart)
   - Weekly revenue (bar chart)
   - The written report with Key Trends, Anomalies and Possible Causes, and Recommendations
6. Click **Download report (PDF)** to save `weekly_report.pdf`.

---

## Data contract

Each agent passes a fixed structure to the next, so any agent can be improved or replaced without breaking the others.

| Stage | Output |
|---|---|
| Data Fetcher | `date_range`, `weekly_totals`, `daily_totals`, `by_category`, `raw_stats` |
| Insight Analyser | `metrics`, `trends`, `anomalies` |
| Report Writer | Weekly report text (saved as `weekly_report.md` by `main.py`, downloadable as PDF in the dashboard) |

Anomalies are found with statistics in code. The LLM only writes the report.

---

## Input CSV format

The uploaded file must contain these columns:

| Column | Description | Example |
|---|---|---|
| `date` | Day of the sale (YYYY-MM-DD) | `2026-10-23` |
| `category` | Product category | `Electronics` |
| `revenue` | Revenue for that row | `4800` |

Example:

```
date,category,revenue
2026-10-22,Clothing,300
2026-10-23,Electronics,4800
```

`python make_sample_data.py` generates a valid file with planted anomalies for testing.

---

## Security

- **Authentication:** the dashboard is gated behind Google sign-in. Unauthenticated visitors only see the login button.
- **Authorised users:** while the OAuth app is in testing mode, only Gmail addresses added as test users can sign in.
- **Secrets stay out of git:** `.env` and `.streamlit/secrets.toml` are listed in `.gitignore`. Check `git status` before every commit.
- **No credentials in code:** the Gemini key is read from the environment and OAuth settings from Streamlit secrets.
- **Rotate exposed secrets:** if the client secret, cookie secret or Gemini key is ever pasted into a chat, screenshot or commit, replace it:
  - Client secret: Google Auth Platform, Clients, open your client, add a new secret and disable the old one
  - Cookie secret: generate a new one with the command in the SSO setup section
  - Gemini key: create a new key and delete the old one
- **Before going public:** publishing the OAuth app, or deploying it, needs your live URL added as an authorised redirect URI and the same secrets entered in your host's secrets settings.

---

## Troubleshooting

| Problem | Cause and fix |
|---|---|
| Dashboard opens without a login button | The app is running old code or a stale instance. Stop it with Ctrl+C and restart. Check that the login block is saved in `app.py`. |
| `To use authentication features, you need to install Authlib` | Streamlit is running from the wrong Python. Activate the venv and start with `python -m streamlit run app.py`. |
| `redirect_uri_mismatch` | The URI in Google Cloud and `secrets.toml` do not match exactly. Check the port, `http` and the `/oauth2callback` path, and wait a few minutes after saving. |
| `Access blocked` | The signing-in account is not listed under **Test users** in Google Auth Platform, Audience. |
| `Google hasn't verified this app` | Normal in testing mode. Click **Advanced**, then **Continue**. |
| `Port 8501 is already in use` | Another copy of the app is running. Stop it, or use `--server.port` with a free port and update the redirect URI. |
| `503 UNAVAILABLE` from Gemini | Google's servers are busy. Wait a minute and retry, or untick **Use Gemini (live)** to use mock mode. |
| Login stops working after editing `secrets.toml` | Restart Streamlit. Secrets are only read at startup. Check that every value is on one line. |
| `ModuleNotFoundError: No module named 'fpdf'` | Dependencies are not installed in the active environment. Run `pip install -r requirements.txt` inside the venv. |

---

## Roadmap

- Automatic retry with backoff when Gemini returns a temporary 503
- Fallback to the mock model if the live call fails
- Restrict access by email domain or an allow-list instead of the testing-mode user list
- Microsoft sign-in alongside Google
- Deployment to Streamlit Community Cloud with a private app setting
- Scheduled weekly report generation
- Tests for the anomaly detection logic

---

## Author

**Varun S**: architecture, agents, dashboard, authentication and documentation.