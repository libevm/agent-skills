---
name: duckduckgo-api
description: Use DuckDuckGo Instant Answer API (no API key) for quick facts, definitions, related topics, and lightweight reference links. Not a full SERP web search API.
allowed-tools: Bash(curl:*), Bash(jq:*)
---

# DuckDuckGo API-Only Search (No API Key)

This skill uses **DuckDuckGo's Instant Answer API**.

Best for:
- quick facts and definitions
- instant-answer summaries
- related topics and reference links

Not for:
- full web search / SERP-style ranked results

## Base endpoint

- `https://api.duckduckgo.com/`

## Core parameters

- `q` — query text
- `format=json` — JSON output (recommended)
- `pretty=1` — pretty output (optional)
- `no_html=1` — strips HTML (recommended for scripts)
- `skip_disambig=1` — reduces disambiguation noise (optional)

## Basic request

```bash
curl -sG "https://api.duckduckgo.com/" \
  --data-urlencode "q=python argparse" \
  --data-urlencode "format=json" \
  --data-urlencode "no_html=1"
```

## Useful jq extracts

### Instant answer text + source

```bash
curl -sG "https://api.duckduckgo.com/" \
  --data-urlencode "q=what is photosynthesis" \
  --data-urlencode "format=json" \
  --data-urlencode "no_html=1" \
| jq -r '
  "Abstract: \(.AbstractText // "")",
  "Source: \(.AbstractSource // "")",
  "URL: \(.AbstractURL // "")"
'
```

### Definition

```bash
curl -sG "https://api.duckduckgo.com/" \
  --data-urlencode "q=define: entropy" \
  --data-urlencode "format=json" \
  --data-urlencode "no_html=1" \
| jq -r '.Definition // empty'
```

### Related topics (including nested topics)

```bash
curl -sG "https://api.duckduckgo.com/" \
  --data-urlencode "q=linux systemd" \
  --data-urlencode "format=json" \
  --data-urlencode "no_html=1" \
| jq -r '
  def items:
    .RelatedTopics[]
    | if has("Topics") then .Topics[] else . end;
  items
  | select(.Text? and .FirstURL?)
  | "\(.Text)\n\(.FirstURL)\n"
'
```

## Query patterns that work well

- concept/definition queries (e.g. `what is zero trust`)
- entity lookups (e.g. `Marie Curie`)
- quick programming concept lookups (e.g. `python dataclass`)

## Quality guidance

- If `AbstractText` is empty, inspect `Answer` and `RelatedTopics`.
- Validate important facts by opening `AbstractURL` / `FirstURL`.
- If the user needs full web results, use a dedicated search skill/API.

## Helper shell function

```bash
ddg_api () {
  curl -sG "https://api.duckduckgo.com/" \
    --data-urlencode "q=$*" \
    --data-urlencode "format=json" \
    --data-urlencode "no_html=1"
}
```
