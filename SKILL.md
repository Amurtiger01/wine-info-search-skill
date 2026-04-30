---
name: wine-info-search
version: 1.5.0
description: >
  This skill should be used when the user wants to search for wine or other alcohol information,
  ratings, prices, or buying recommendations. It supports searching by brand name, vintage year,
  and series name across major domestic and international platforms. It also supports image-based
  wine label recognition using OCR. Trigger scenarios include: looking up wine ratings,
  comparing wine prices across platforms (京东/天猫/Wine-Searcher/etc.), checking vintage comparisons
  for a specific wine, getting detailed wine info (grape varieties, taste profile, food pairing),
  getting wine & winery background information, getting vintage recommendations by year,
  getting health-related drinking advice by age group and medical conditions, getting staple food
  and main dish pairing recommendations, getting drinking-window advice for aged wines, identifying
  a wine from a label photo, or asking for purchase recommendations.
---

# Wine Info Search

## Overview

Search for wine and other alcohol detailed information, community/professional ratings, and prices across 16+ major platforms worldwide. Primary data sources are **Wine-Searcher via WebFetch** and **Vivino via Firecrawl**. **Firecrawl integration (v1.4)** restores Vivino access by using US proxy IPs + JavaScript rendering, bypassing Vivino's China IP blockade. **Wikipedia API integration (v1.5)** provides wine & winery background information (history, region, winery stories) from both English and Chinese Wikipedia, accessible from China without API keys. **Open Food Facts API** is a supplementary free data source. Supports Chinese/English bilingual name mapping (110+ common wine names) with multi-segment replacement for automatic cross-language search. WebFetch-assisted price fetching for real-time prices from JD.com, Wine-Searcher, etc. Image-based label recognition via pytesseract or easyocr. **Vintage recommendations** with rating-based labels (卓越/优秀/良好/一般/不佳) and year-specific buying advice. Health drinking advice customized by age group and medical conditions. Staple food & main dish pairing recommendations. Also generates direct search links for all major domestic (京东/天猫/淘宝/苏宁/拼多多/1919/也买酒/酒仙网) and international (Vivino/Wine.com/Drizly/Total Wine/Wine-Searcher/Wine Spectator/CellarTracker/Decántalo) platforms.

## Data Sources

||| Source | Type | Key Required | Data Provided | Status |
|||--------|------|-------------|---------------|--------|
||| Wine-Searcher (WebFetch) | Web + AI parsing | No | Ratings, prices, vintages, grape info, tasting notes | ? Primary |
||| Vivino (Firecrawl) | Firecrawl scrape | Yes (API Key) | Ratings, taste profile, grapes, food pairing, prices | ? Secondary (restored) |
||| Wikipedia API | REST API | No | Wine & winery background, history, region info | ? Tertiary (v1.5) |
||| Open Food Facts API | REST API | No | Basic wine metadata (ABV, grape, image) | ? Supplementary |
||| Vivino API | REST API | No | Wine search, details, ratings | ? Blocked (403) |
||| Vivino Web (fallback) | Web scraping | No | Basic search when API is blocked | ? Timeout (CN) |
||| WebFetch Price Hints | URL + AI parsing | No | Real-time prices from JD.com, Tmall, Wine-Searcher | ? Works |
||| Direct Scrape (legacy) | Web scraping | No | Best-effort prices from JD.com, Wine-Searcher | ?? Low rate |
||| Platform Link Generator | URL builder | No | Direct search links for 16+ platforms | ? Works |
||| Health & Food Database | Built-in data | No | Age-group drinking limits, 10 health conditions, 6 wine-type food pairings | ? Works |

### Data Source Strategy (v1.5)

The script uses a **cascading fallback** approach for wine search:

1. **Firecrawl → Vivino** — If `FIRECRAWL_API_KEY` is configured, uses Firecrawl's US proxy + JS rendering to access Vivino search page. **Best option for China users** — returns rich data (ratings, taste profile, grape varieties, food pairing, prices).
2. **Vivino API** — Attempted next (best-case: rich data). Currently returns 403 Forbidden.
3. **Vivino Web Search** — Best-effort fallback. Often times out from China mainland.
4. **Wine-Searcher direct scrape** — Best-effort. Often times out from China mainland.
5. **Open Food Facts API** — Always accessible, but limited to basic metadata (no ratings/prices).

**Wine & Winery Background** is fetched from **Wikipedia API** (both English and Chinese), which is:
- Free, no API key required
- Accessible from China mainland
- Provides historical background, winery stories, region appellation info
- Bilingual: automatically searches both `en.wikipedia.org` and `zh.wikipedia.org`

**Vintage Recommendations** use the existing Vivino vintage data but add:
- Rating-based recommendation labels: ? 卓越 (≥4.5), ? 优秀 (≥4.0), ? 良好 (≥3.5), ? 一般 (≥3.0), ?? 不佳 (<3.0)
- Confidence notes for low rating counts
- Year-specific buying advice when user specifies a vintage
- Summary of best vintages (outstanding + very good)

**For the AI agent**: The most reliable approach is:
- **With Firecrawl**: Firecrawl → Vivino provides rich data directly from the script.
- **Without Firecrawl**: Use **WebFetch on Wine-Searcher** as the primary data source. The script outputs WebFetch-ready hints with URLs and extraction instructions.

### Firecrawl Configuration

To enable Firecrawl-based Vivino access, configure the API key using one of:

```bash
# Option 1: Environment variable
set FIRECRAWL_API_KEY=fc-xxxx     # Windows
export FIRECRAWL_API_KEY=fc-xxxx  # Linux/macOS

# Option 2: Command-line argument
python scripts/wine_search.py "拉菲" --firecrawl-key fc-xxxx
```

Free tier provides **500 requests/month**. Register at [firecrawl.dev](https://firecrawl.dev).

## Core Capabilities

### 1. Wine Information Search (`--mode info`)

Search for wine details and ratings. Returns:
- Wine name, winery, vintage year
- Wine type (red/white/sparkling/rosé/dessert/fortified)
- Region and country of origin
- Community rating with visual bar (★★★★☆ 4.2/5)
- Number of ratings
- Reference price and currency
- Direct link (Wine-Searcher / Vivino)
- **Grape varieties** with blending percentages (e.g. "Cabernet Sauvignon 70%, Merlot 30%")
- **Taste profile**: body/tannin/acidity/sweetness with visual bars (█???? 1/5)
- **Food pairing** suggestions
- **Wine description** summary

**Script command:**
```bash
python scripts/wine_search.py "拉菲" 2018 --mode info
python scripts/wine_search.py "Lafite" 2018 "Rothschild" --mode info
```

### 2. Wine Price Comparison (`--mode price`)

Compare prices across platforms. Returns:
- WebFetch-ready price hints (URL + extraction instructions) for JD.com, Tmall, Wine-Searcher, Vivino (Firecrawl)
- Best-effort direct scraping results (legacy, low success rate)
- Direct search links for 8 domestic + 8 international platforms

**How WebFetch price hints work:**
The script outputs URLs and extraction instructions for each price platform. The AI agent should use its WebFetch tool to visit these URLs, parse the page content, and extract price data. This approach is far more reliable than direct HTML scraping because WebFetch handles JavaScript rendering and anti-scraping measures.

**Script command:**
```bash
python scripts/wine_search.py "奔富" 2020 "Bin 389" --mode price
python scripts/wine_search.py "Penfolds" --mode price
```

### 3. Full Search (`--mode all`, default)

Combines info + price + wine tips + health advice + food pairing in one search. Returns:
- All wine information from Capability 1
- Vintage comparison table for the best match (year × rating × price)
- All platform prices and links from Capability 2
- Wine tips: drinking window advice based on vintage age and wine type, purchase recommendations
- Health drinking advice by age group with recommended daily limits
- Health condition warnings (10 conditions: hypertension, diabetes, gout, liver disease, etc.)
- Staple food & main dish pairing recommendations

**Script command:**
```bash
python scripts/wine_search.py "拉菲" 2018
python scripts/wine_search.py "Lafite" 2018 "Rothschild" --mode all
```

### 4. Image-Based Search (`--image`)

Identify wines from label photos using OCR text extraction, then search with the extracted info.

**Script command:**
```bash
python scripts/wine_search.py --image "/path/to/wine_label.jpg"
```

**How it works:**
1. Attempts OCR via `pytesseract` (if installed) to extract text from the wine label image
2. Falls back to `easyocr` (if installed) for deep-learning-based text extraction
3. Falls back to filename-based hints if no OCR tool is available
4. Parses extracted text to identify brand name, vintage year, and series
5. Runs `search_wine()` automatically with the identified information
6. If no OCR tools are available, guides the user to install one or use the Vivino App

**Optional OCR dependencies:**
```bash
pip install pytesseract Pillow   # Requires Tesseract-OCR installed on system
pip install easyocr              # Deep learning OCR, no external install needed
```

## Workflow

1. **Collect parameters** — Extract brand name (required), year (optional), series name (optional), and mode (info/price/all, default all) from the user's query. If the user mentions a wine label photo, use `--image` mode.

2. **Resolve bilingual query** — The script automatically detects Chinese/English input and maps it to the corresponding language variant using a 110+ entry name dictionary with multi-segment replacement. For example, "拉菲 奥希耶黑鸢" → "Lafite Aussieres Noir". This ensures domestic platforms get Chinese queries and international platforms get English queries.

3. **Execute search** — Run `scripts/wine_search.py` with the collected parameters. The script will:
   - If Firecrawl API key is available, use Firecrawl to access Vivino (richest data source)
   - Fall back through Vivino API → Vivino Web → Wine-Searcher → Open Food Facts
   - Select the best matching result (preferring matching vintage year)
   - Fetch wine details (grape varieties, taste profile, food pairing, description) if available
   - Optionally fetch vintage comparison data
   - Generate WebFetch-ready hints for Wine-Searcher (primary) and domestic platforms
   - Generate direct search links for all platforms (CN query for domestic, EN for international)

4. **Use WebFetch for reliable data** — The AI agent should use its WebFetch tool to:
   - **Visit Wine-Searcher** first for the most comprehensive wine data (ratings, prices, tasting notes)
   - Visit domestic platforms (京东/天猫) for CNY prices (may be blocked by anti-scraping)
   - Parse the returned content and present it to the user

5. **Present results** — Display the structured output to the user, highlighting:
   - Best match with rating, price, and detailed wine profile
   - Key price differences across platforms
   - Drinking window advice if vintage year is provided

6. **Handle no results** — If all data sources return no results, provide:
   - Direct Wine-Searcher search link
   - Open Food Facts search link
   - Vivino search link (may require VPN or Firecrawl)
   - Suggest trying alternative spellings (Chinese ? English)
   - Suggest removing the series name to broaden the search
   - Suggest configuring Firecrawl API key for Vivino access

## Command Reference

```
python scripts/wine_search.py <brand> [year] [series] [--mode info|price|all]
python scripts/wine_search.py --image <image_path>
python scripts/wine_search.py <brand> --firecrawl-key <api_key>
```

||| Parameter | Required | Description |
|||-----------|----------|-------------|
||| `brand` | Yes | Wine brand name (Chinese or English), e.g. "拉菲", "Lafite", "奔富" |
||| `year` | No | Vintage year (1800-2100), e.g. 2018 |
||| `series` | No | Series/cuvée name, e.g. "Bin 389", "Rothschild" |
||| `--mode` | No | Search mode: `info` (details only), `price` (prices & links), `all` (default) |
||| `--image` | No | Path to wine label image for photo recognition guidance |
||| `--firecrawl-key` | No | Firecrawl API key for Vivino access (overrides env var) |
| `--insecure` | No | Disable SSL certificate verification (for restricted networks) |
| `--no-wiki` | No | Skip Wikipedia background lookup |

## Common Query Patterns

||| User Query | Suggested Command |
|||-----------|-------------------|
||| "查一下拉菲2018年的评分" | `python scripts/wine_search.py "拉菲" 2018 --mode info` |
||| "奔富Bin 389多少钱" | `python scripts/wine_search.py "奔富" 2020 "Bin 389" --mode price` |
||| "Lafite Rothschild 2018详细信息和价格" | `python scripts/wine_search.py "Lafite" 2018 "Rothschild" --mode all` |
||| "这瓶酒是什么" (with photo) | `python scripts/wine_search.py --image "<path>"` |
||| "帮我看看这款酒值不值得买" | `python scripts/wine_search.py "<brand>" <year> --mode all` |
||| "推荐买红酒的平台" | `python scripts/wine_search.py "<brand>" --mode price` |
||| "用Firecrawl搜索Vivino" | `python scripts/wine_search.py "<brand>" --firecrawl-key fc-xxxx` |

## Output Sections

When running in `--mode all`, the script outputs six structured sections:

### Section 1: ? 酒款信息 (Wine Information)
- Number of search results found
- Top 8 matches with: name, winery, type, region, rating bar, reference price, link
- Best match indicator (? 最佳匹配)
- **Detailed wine info for best match**:
  - Grape varieties with blending percentages
  - Taste profile (body/tannin/acidity/sweetness) with visual bars and Chinese labels
  - Food pairing suggestions
  - Wine description summary
- **Vintage comparison table with recommendations** (up to 15 years):
  - Rating-based recommendation labels: ? 卓越 (≥4.5), ? 优秀 (≥4.0), ? 良好 (≥3.5), ? 一般 (≥3.0), ?? 不佳 (<3.0)
  - Confidence notes for low rating counts
  - Year-specific buying advice when user specifies a vintage
  - Summary of best vintages (outstanding + very good)

### Section 1c: ?? 酒款与酒庄背景 (Wine & Winery Background) — **NEW in v1.5**
- Wine background from Wikipedia (history, region, appellation info)
- Winery/producer background from Wikipedia (founding, notable achievements)
- Bilingual search: automatically tries both English and Chinese Wikipedia
- Links to full Wikipedia articles for deeper reading

### Section 2: ? 各平台价格与购买链接 (Platform Prices & Links)
- WebFetch-ready price hints with URLs and extraction instructions (including Firecrawl-Vivino hint if configured)
- Best-effort direct scraping results (legacy)
- 8 domestic platform search links
- 8 international platform search links

### Section 3: ? 酒款小贴士 (Wine Tips)
- Drinking window advice based on vintage age **and wine type** (different windows for red/white/sparkling/dessert/fortified)
- Purchase recommendations

### Section 4: ? 健康饮用建议 (Health & Drinking Advice)
- Age-group-specific daily drinking limits (4 groups: 18-35 / 36-55 / 56-70 / 70+)
- Standard drink calculations based on wine ABV
- Health condition warnings (10 conditions with risk levels and max intake):
  - Hypertension, Diabetes, Gout, Liver disease, Gastritis, Heart disease, Kidney disease, Pregnancy, Medication, Obesity
- General safe drinking tips
- Wine type-specific notes (e.g., fortified wines: halve the amount; dessert wines: sugar warning)

### Section 5: ?? 餐饮搭配建议 (Food Pairing Recommendations)
- Wine-Searcher / Vivino food pairing suggestions (from API or Firecrawl, if available)
- Curated staple food recommendations by wine type (4 items each)
- Curated main dish recommendations with detailed pairing explanations (4 items each)
- Pairing principle for each wine type

## Wine Type Mapping

||| Code/Key | Display Name |
|||----------|-------------|
||| 1 / red | 红葡萄酒 (Red) |
||| 2 / white | 白葡萄酒 (White) |
||| 3 / sparkling | 起泡酒 (Sparkling) |
||| 4 / rose | 桃红葡萄酒 (Rosé) |
||| 5 / dessert | 甜酒 (Dessert) |
||| 6 / fortified | 加强酒 (Fortified) |

## Rating Scale

||| Range | Description |
|||-------|-------------|
||| 0 - 2.0 | 较差 (Poor) |
||| 2.0 - 3.0 | 一般 (Below Average) |
||| 3.0 - 3.5 | 尚可 (Average) |
||| 3.5 - 4.0 | 不错 (Good) |
||| 4.0 - 4.5 | 优秀 (Very Good) |
||| 4.5 - 5.0 | 卓越 (Outstanding) |

**Tip**: Ratings ≥ 4.0 (or 80/100 on Wine-Searcher) generally indicate good quality wines.

## Important Notes

- **Firecrawl restores Vivino access** — By configuring a Firecrawl API key, the script can access Vivino's rich data (ratings, taste profile, grape varieties, food pairing) via US proxy + JS rendering. This is the recommended approach for China-based users.
- **Wikipedia provides wine & winery background (v1.5)** — The script automatically fetches background information from both English and Chinese Wikipedia. No API key required, accessible from China. Provides wine history, winery stories, and region appellation info.
- **Vintage recommendations with buying advice (v1.5)** — The vintage comparison table now includes recommendation labels (卓越/优秀/良好/一般/不佳) and year-specific buying advice. When the user specifies a vintage, the script recommends whether to buy and suggests better alternatives if applicable.
- **Vivino API deprecated** — Vivino closed public API access in 2025 (returns 403 Forbidden). The script still attempts it as best-effort, but automatically falls back to Firecrawl/Vivino, Wine-Searcher and Open Food Facts.
- **Wine-Searcher is the primary data source** — The most reliable way to get wine data without Firecrawl is via the AI agent's WebFetch tool visiting Wine-Searcher. Direct script access to Wine-Searcher often times out from China mainland.
- **Open Food Facts as supplementary** — Free, public API accessible from China. Provides basic wine metadata (ABV, grape variety, image) but no ratings or prices.
- **API key required for Firecrawl** — Firecrawl requires an API key (free tier: 500 requests/month). Set via `FIRECRAWL_API_KEY` env var or `--firecrawl-key` argument.
- **Chinese/English bilingual name mapping** — The script contains a 110+ entry dictionary with multi-segment replacement. When "拉菲 奥希耶黑鸢" is entered, it maps to "Lafite Aussieres Noir" for international platforms.
- **Image search via OCR** — The `--image` flag uses pytesseract or easyocr (optional dependencies) to extract text from wine label images, then automatically parses the text to identify brand/year/series and runs a full search.
- **WebFetch-assisted price fetching** — The script outputs WebFetch-ready price hints (URL + extraction instructions). The AI agent should use its WebFetch tool to visit these URLs and parse the content for real-time prices. This is far more reliable than direct HTML scraping.
- **Health drinking advice** — The script provides age-group-specific daily drinking limits (4 age groups), health condition warnings (10 conditions with risk levels), and general safe drinking tips. Advice is automatically adjusted based on wine type ABV.
- **Food pairing recommendations** — The script provides curated staple food and main dish pairing suggestions for 6 wine types (red/white/sparkling/rosé/dessert/fortified), along with pairing principles.
- **Secure by default, fallback for compatibility** — The script validates SSL certificates by default. If certificate verification fails (e.g. corporate proxy), it automatically retries without verification. Users can also pass `--insecure` to skip verification entirely.
