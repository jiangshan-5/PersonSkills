---
name: multi-source-audio-crawler
description: This skill covers the design, implementation, and deployment of a multi-source dynamic audio crawler and synchronized backend downloader. It details search engine query scraping, specialized parsing (Mixkit, Tabletop Audio), generic link extraction, WAF bypass (Referer + User-Agent), and backend downloader hardening with retries and exponential backoff.
---

# Multi-Source Audio Crawler & Backend Sync Pipeline

## Overview

This skill details a robust, end-to-end audio crawling and push-synchronization system. The system dynamically crawls free nature/ambient audio websites, extracts track metadata, pushes configurations to a FastAPI backend API, and hardens the backend downloader to stream, download, and cache tracks locally.

---

## 1. Dynamic Web Search & Site Discovery

To avoid hardcoded lists or raw GitHub resource dependencies (which can be blocked or go stale), the crawler uses a dynamic search-engine query discovery model.

### China Network (GFW) Bypass
- **Problem**: Popular global search APIs (like DuckDuckGo or Google) are network-unreachable on servers inside China (e.g., Baidu Cloud IP).
- **Solution**: Use **`cn.bing.com`** as a fallback search engine. It is fully accessible within China and offers comprehensive search indexing.

### Search Result Parsing
Query Bing using `requests` with a Chrome User-Agent:
```python
import urllib.parse
from bs4 import BeautifulSoup

query = "免费 白噪音 mp3 下载"
url = f"https://cn.bing.com/search?q={urllib.parse.quote(query)}"
r = requests.get(url, headers=headers, timeout=10)
soup = BeautifulSoup(r.text, "html.parser")

for h2 in soup.find_all("h2"):
    a = h2.find("a")
    if a and a.get("href").startswith("http"):
        # Filter out portal sites like wikipedia, taobao, iqiyi, etc.
        ...
```

---

## 2. Specialized & Generic Crawler Engines

The crawler operates a multi-site parsing engine. Visited domains are routed to their matching parser.

### A. Tabletop Audio Scraper
Extracts loop ID and title from custom DOM tags:
- Target: `https://tabletopaudio.com/`
- Parsing strategy: Extracts `track_id` from the JavaScript function `saveAs('id')` within the DOM, mapping direct MP3 assets: `https://sounds.tabletopaudio.com/{track_id}.mp3`.

### B. Mixkit Audio Scraper
Extracts tracks from Mixkit's free nature sfx section:
- Target: `https://mixkit.co/free-sound-effects/nature/`
- Parsing strategy: Finds player tags containing `data-audio-player-preview-url-value` and traverses up to find the nearest `audio-player__title` text.
- ID Sanitization: Extracts sfx index (e.g. `mixkit_2393`) and generates metadata.

### C. Generic Fallback Parser
Handles any arbitrary website:
- Automatically scans all `<a>` tags where `href` ends in `.mp3` or `.wav`.
- Filters tracks strictly against a nature keyword pattern (e.g., rain, sea, ocean, wind, campfire, forest, stream).
- Cleans and formats link text to act as the title and unique alphanumeric `id` (matching `^[a-zA-Z0-9_]+$`).

---

## 3. WAF Bypass (Referer & User-Agent)

Many high-quality hosting sites use Cloudflare or mod_security WAF (Web Application Firewall) to prevent hotlinking and crawling.

### Referer Header Enforcement
- **Crucial Finding**: Tabletop Audio's Cloudflare setup returns `403 Forbidden` if a request lacks the `Referer` header matching `https://tabletopaudio.com/`, even if a valid Chrome `User-Agent` is present.
- **Solution**: The backend downloader must dynamically set the `Referer` header based on the target domain:
```python
parsed_url = urllib.parse.urlparse(source_url)
domain = parsed_url.netloc.lower()
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..."
}
if "tabletopaudio.com" in domain:
    headers["Referer"] = "https://tabletopaudio.com/"
elif "mixkit.co" in domain:
    headers["Referer"] = "https://mixkit.co/"
else:
    headers["Referer"] = f"{parsed_url.scheme}://{parsed_url.netloc}/"
```

---

## 4. Backend Downloader Hardening

Pushing many tracks simultaneously will cause the backend server to fire off dozens of concurrent network requests to the same target domain. This triggers Cloudflare rate-limiting, resulting in subsequent `403 Forbidden` failures.

### Concurrency Protection & Backoff Retry
1. **Delay**: Throttle push requests from the crawler (e.g. `time.sleep(1)`) to stagger background downloader tasks.
2. **Retry Loop**: Implement a 3-attempt download loop with exponential backoff:
```python
max_retries = 3
for attempt in range(max_retries):
    try:
        response = requests.get(source_url, headers=headers, stream=True, timeout=60)
        response.raise_for_status()
        break
    except Exception as e:
        if attempt == max_retries - 1:
            raise e
        # Sleep increases with attempts: 2s, 4s
        time.sleep(2 * (attempt + 1))
```
3. **Atomic Writes**: Download to a temporary file (e.g., `.tmp`) and swap/rename it to `.mp3` on completion to prevent file corruption.

---

## 5. Scheduler & Deployment

### Virtual Environment Setup
Ensure Playwright dependencies are set up inside a local Python virtual environment to keep global modules clean:
```bash
python3 -m venv /root/crawl4ai_venv
/root/crawl4ai_venv/bin/pip install crawl4ai requests beautifulsoup4
/root/crawl4ai_venv/bin/python3 -m playwright install chromium
/root/crawl4ai_venv/bin/python3 -m playwright install-deps
```

### Crontab Scheduling
Add the execution script to the system cron tab to check hourly at minute 0:
```bash
0 * * * * /root/crawl4ai_venv/bin/python3 /root/crawl_white_noise.py >> /var/log/crawl_white_noise.log 2>&1
```
