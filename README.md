# 🏨 Concurrent Hotel Scraper

A concurrent web scraper built with [Scrapling](https://github.com/D4Vinci/Scrapling)** that pulls hotel data from 10 MakeMyTrip hotel detail pages using 2 Chrome browser instances running in parallel.


## Table of Contents

- [Project Overview]
- [Architecture Notes]
- [Project Structure]
- [Installation]
- [Virtual Environment Setup]
- [Proxy Configuration]
- [Usage]
- [Output Format]
- [Logging]
- [Troubleshooting]


## Project Overview

 Feature | Detail |
 Framework | Scrapling (`StealthyFetcher`) |
 Browser | Chromium via Patchright (stealth-patched Playwright) |
 Concurrency | 2 parallel worker threads (`ThreadPoolExecutor`) |
 Target site | MakeMyTrip hotel detail pages |
 Data extracted | Hotel name, price, rating, review count |
 Anti-bot measures | Stealth browser, ad blocking, canvas noise, WebRTC masking |
 Output format | Excel (XLSX) |
 Proxy support | Residential / datacenter, per-worker or shared |



## Architecture Notes

### Concurrency Approach

I went with `ThreadPoolExecutor(max_workers=2)` to spin up 2 browser instances at the same time. Each thread gets its own `StealthyFetcher` session so there's no overlap or conflict between them. Both are kicked off together and I collect results using `as_completed()` as each one wraps up.

### URL Distribution Strategy

I split the URLs round-robin across both workers so each one gets exactly 5:

```
urls = [u1, u2, u3, u4, u5, u6, u7, u8, u9, u10]

Worker 1 → [u1, u3, u5, u7, u9]   (indices 0, 2, 4, 6, 8)
Worker 2 → [u2, u4, u6, u8, u10]  (indices 1, 3, 5, 7, 9)
```

This keeps the load balanced regardless of what order the URLs are in.

### Proxy Handling Mechanism

Proxy credentials are picked up from the `.env` file at runtime using `python-dotenv`. I've set it up to support two modes — a single shared proxy for both workers, or separate proxies per worker (`PROXY_SERVER_1`, `PROXY_SERVER_2`) if you want independent IPs for each session. The proxy config gets passed directly into each `StealthyFetcher.fetch()` call.

### Anti-Bot Strategy

| Technique | Implementation |
|---|---|
| Stealth browser | Patchright-patched Chromium (`StealthyFetcher`) |
| HTTP/1.1 forced | `--disable-http2` flag to avoid MMT's protocol-level blocking |
| Ad & tracker blocking | `block_ads=True` blocks ~3,500 known ad/tracking domains |
| Canvas fingerprint noise | `hide_canvas=True` |
| WebRTC IP leak prevention | `block_webrtc=True` |
| Google referrer spoofing | `google_search=True` |
| Indian locale & timezone | `locale="en-IN"`, `timezone_id="Asia/Kolkata"` to match MMT's audience |
| Network idle wait | `network_idle=True` + `wait=3000` ensures JS-rendered content is ready |
| Retry on failure | Up to 3 attempts per URL with exponential back-off |



## Project Structure

```
hotel-scraper/
├── scraper.py          # Main scraper (orchestrator + worker logic)
├── urls.yaml           # 10 target hotel URLs (loaded at runtime)
├── .env                # Proxy credentials
├── requirements.txt    # Python dependencies
├── README.md           # This file
├── output.xlsx         # Generated after run
├── scraper.log         # Main process log (generated after run)
├── worker_1.log        # Worker 1 log (generated after run)
└── worker_2.log        # Worker 2 log (generated after run)
```



## Installation

### Prerequisites

- Python 3.10+
- Node.js 18+ (needed internally by Playwright/Patchright)
- A working internet connection

### Steps

```bash
# 1. Create and activate a virtual environment (see next section)

# 2. Install Python dependencies
pip install -r requirements.txt

# 3. Install the Patchright Chromium browser
python -m patchright install chromium

# 4. Configure proxy credentials if needed (see Proxy Configuration)
```



## Virtual Environment Setup

```bash
# Create virtual environment
python3 -m venv .venv

# Activate — Linux / macOS
source .venv/bin/activate

# Activate — Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Install the browser binary (only needed once)
python -m patchright install chromium
```



## Proxy Configuration

Edit the `.env` file in the project root:

### Option A — Single proxy shared by both workers

```dotenv
PROXY_SERVER=http://proxy.example.com:8080
PROXY_USERNAME=your_username
PROXY_PASSWORD=your_password
```

### Option B — Per-worker proxies (for rotating IPs)

```dotenv
PROXY_SERVER_1=http://gate.provider.com:10000
PROXY_USERNAME_1=user-session-1
PROXY_PASSWORD_1=secret1

PROXY_SERVER_2=http://gate.provider.com:10000
PROXY_USERNAME_2=user-session-2
PROXY_PASSWORD_2=secret2
```

### Residential proxy providers

The scraper works with any provider that supports HTTP/HTTPS/SOCKS5 proxies:

- [Bright Data](https://brightdata.com)
- [Oxylabs](https://oxylabs.io)
- [SmartProxy](https://smartproxy.com)
- [IPRoyal](https://iproyal.com)

Leave everything commented out to run without a proxy. Fine for local testing, but MMT may block requests coming from a regular IP.



## Usage

```bash
python scraper.py
```

### Expected output

```
15:30:00  [MainThread]  INFO      ============================================================
15:30:00  [MainThread]  INFO      Concurrent Hotel Scraper — starting
15:30:00  [MainThread]  INFO      Workers : 2  (threads)
15:30:00  [MainThread]  INFO      Loaded 10 URLs from urls.yaml
15:30:00  [MainThread]  INFO      Worker 1 will scrape 5 URLs
15:30:00  [MainThread]  INFO      Worker 2 will scrape 5 URLs
15:30:00  [MainThread]  INFO      Both browser threads started — waiting for results …
15:31:10  [MainThread]  INFO      Worker 1 completed — returned 5 results
15:31:20  [MainThread]  INFO      Worker 2 completed — returned 5 results
15:31:20  [MainThread]  INFO      Excel saved → output.xlsx
15:31:20  [MainThread]  INFO      ============================================================
15:31:20  [MainThread]  INFO      Execution summary
15:31:20  [MainThread]  INFO        Total URLs : 10
15:31:20  [MainThread]  INFO        Succeeded  : 10
15:31:20  [MainThread]  INFO        Failed     : 0
15:31:20  [MainThread]  INFO        Duration   : 80.2 s
15:31:20  [MainThread]  INFO      ============================================================
```



## Output Format

The scraper produces a single **`output.xlsx`** file with all the scraped results.

| Column | Type | Description |
|---|---|---|
| `url` | string | The scraped hotel page URL |
| `hotel_name` | string \| null | Full hotel name |
| `price` | string \| null | Displayed room price (e.g. `₹4,999`) |
| `rating` | string \| null | User rating score (e.g. `4.3`) |
| `reviews` | string \| null | Number of reviews (e.g. `1,250`) |
| `error` | string \| null | Error message if the URL failed, else null |



## Logging

Three log files get created when you run the scraper:

| File | Content |
|---|---|
| `scraper.log` | Main thread: URL distribution, worker results, final summary |
| `worker_1.log` | Worker 1: per-URL progress, retries, extraction results |
| `worker_2.log` | Worker 2: per-URL progress, retries, extraction results |

Everything is also printed to the terminal in real time so you can follow along as it runs.


## Troubleshooting

### `ModuleNotFoundError: No module named 'patchright'`

```bash
pip install patchright
python -m patchright install chromium
```

### `ModuleNotFoundError: No module named 'msgspec'`

```bash
pip install msgspec
```

### `Browser not found` / Chromium launch failure

```bash
python -m patchright install chromium --force
```

### All URLs returning `None` for all fields

MMT's page structure may have changed. Open the hotel page in your browser, inspect the element you want, and update the relevant selector list in `scraper.py` (`HOTEL_NAME_SELECTORS`, `PRICE_SELECTORS`, etc.).

### Bot detection / empty pages

- Add a residential proxy in `.env`.
- Increase `timeout` to `90_000` and make sure `network_idle=True` is set.
- Try dropping to 1 worker temporarily to isolate whether it's a concurrency issue.

### `openpyxl` not installed

```bash
pip install openpyxl
```