---
name: crawl4ai
description: This skill should be used when users need to scrape websites, extract structured data, handle JavaScript-heavy pages, crawl multiple URLs, or build automated web data pipelines. Includes optimized extraction patterns with schema generation for efficient, LLM-free extraction.
---

# Crawl4AI

## Overview

Crawl4AI provides comprehensive web crawling and data extraction capabilities. This skill supports both **CLI** (recommended for quick tasks) and **Python SDK** (for programmatic control).

**Choose your interface:**

- **CLI** (`crwl`) - Quick, scriptable commands: [CLI Guide](references/cli-guide.md)
- **Python SDK** - Full programmatic control: [SDK Guide](references/sdk-guide.md)

---

## Quick Start

### Installation

```bash
pip install crawl4ai
crawl4ai-setup

# Verify installation
crawl4ai-doctor
```

### CLI (Recommended)

```bash
# Basic crawling - returns markdown
crwl https://example.com

# Get markdown output
crwl https://example.com -o markdown

# JSON output with cache bypass
crwl https://example.com -o json -v --bypass-cache

# See more examples
crwl --example
```

### Python SDK

```python
import asyncio
from crawl4ai import AsyncWebCrawler

async def main():
    async with AsyncWebCrawler() as crawler:
        result = await crawler.arun("https://example.com")
        print(result.markdown[:500])

async def main():
    async with AsyncWebCrawler() as crawler:
        result = await crawler.arun("https://example.com")
        print(result.markdown[:500])

asyncio.run(main())
```

For SDK configuration details: [SDK Guide - Configuration](references/sdk-guide.md#configuration) (lines 61-150)

---

## Core Concepts

### Configuration Layers

Both CLI and SDK use the same underlying configuration:

| Concept | CLI | SDK |
|---------|-----|-----|
| Browser settings | `-B browser.yml` or `-b "param=value"` | `BrowserConfig(...)` |
| Crawl settings | `-C crawler.yml` or `-c "param=value"` | `CrawlerRunConfig(...)` |
| Extraction | `-e extract.yml -s schema.json` | `extraction_strategy=...` |
| Content filter | `-f filter.yml` | `markdown_generator=...` |

### Key Parameters

**Browser Configuration:**

- `headless`: Run with/without GUI
- `viewport_width/height`: Browser dimensions
- `user_agent`: Custom user agent
- `proxy_config`: Proxy settings

**Crawler Configuration:**

- `page_timeout`: Max page load time (ms)
- `wait_for`: CSS selector or JS condition to wait for
- `cache_mode`: bypass, enabled, disabled
- `js_code`: JavaScript to execute
- `css_selector`: Focus on specific element

For complete parameters: [CLI Config](references/cli-guide.md#configuration) | [SDK Config](references/sdk-guide.md#configuration)

### Output Content

Every crawl returns:

- **markdown** - Clean, formatted markdown
- **html** - Raw HTML
- **links** - Internal and external links discovered
- **media** - Images, videos, audio found
- **extracted_content** - Structured data (if extraction configured)

---

## Markdown Generation (Primary Use Case)

Crawl4AI excels at generating clean, well-formatted markdown:

### CLI

```bash
# Basic markdown
crwl https://docs.example.com -o markdown

# Filtered markdown (removes noise)
crwl https://docs.example.com -o markdown-fit

# With content filter
crwl https://docs.example.com -f filter_bm25.yml -o markdown-fit
```

**Filter configuration:**

```yaml
# filter_bm25.yml (relevance-based)
type: "bm25"
query: "machine learning tutorials"
threshold: 1.0
```

### Python SDK

```python
from crawl4ai.content_filter_strategy import BM25ContentFilter
from crawl4ai.markdown_generation_strategy import DefaultMarkdownGenerator

bm25_filter = BM25ContentFilter(user_query="machine learning tutorials")
markdown_generator = DefaultMarkdownGenerator(content_filter=bm25_filter)
```
