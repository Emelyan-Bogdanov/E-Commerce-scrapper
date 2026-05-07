# E-Commerce Scraper - Architecture & Data Flow

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         INPUT: scrapeme.json                             │
│                    (URLs + CSS selectors config)                        │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                    LAYER 1: DATA INGESTION                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ WebScraper (src/scrapers/scraper.py)                            │   │
│  │  • Fetch HTML from configured URLs                              │   │
│  │  • Retry failed requests (up to 3 times)                        │   │
│  │  • Respect rate limits (delay between requests)                │   │
│  │  • Handle HTTP errors gracefully                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                        Raw HTML
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                   LAYER 2: DATA EXTRACTION                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ProductParser (src/scrapers/scraper.py)                         │   │
│  │  • Parse HTML with BeautifulSoup                                │   │
│  │  • Apply CSS selectors to extract fields                        │   │
│  │  • For each product container:                                  │   │
│  │    - Extract product_id, name, price, rating, etc.             │   │
│  │    - Validate required fields are present                       │   │
│  │    - Preserve raw extracted data                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                    Raw product data
                    (unstructured, messy)
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                   LAYER 3: DATA CLEANING                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ DataCleaner (src/processors/cleaner.py)                         │   │
│  │  • clean_price() - Extract $99.99 → 99.99                       │   │
│  │  • clean_discount() - Parse "50% OFF" → 50.0                    │   │
│  │  • clean_rating() - Normalize "4.5 stars" → 4.5                 │   │
│  │  • clean_text() - Remove extra spaces, HTML tags               │   │
│  │  • clean_stock_status() - "In Stock" → "in_stock"              │   │
│  │  • clean_count() - Parse "1.5k reviews" → 1500                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                   Cleaned product data
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                  LAYER 4: DATA VALIDATION                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ DataValidator (src/processors/cleaner.py)                       │   │
│  │  • Validate required fields present (id, name, price, url)     │   │
│  │  • Price validation: price > 0 and < 10 million               │   │
│  │  • Rating validation: 1-5 scale                                 │   │
│  │  • Discount validation: 0-100 %                                 │   │
│  │  • URL validation: proper format                                │   │
│  │  • Collect validation issues                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                   Validated product data
                   + quality issues list
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                  LAYER 5: QUALITY SCORING                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ QualityScorer (src/processors/cleaner.py)                       │   │
│  │                                                                  │   │
│  │  Completeness Score (0-100):                                    │   │
│  │    = (required_fields_present / total_required) * 0.7           │   │
│  │    + (optional_fields_present / total_optional) * 0.3           │   │
│  │                                                                  │   │
│  │  Accuracy Score (0-100):                                        │   │
│  │    = 100 - (validation_issues * 10)                             │   │
│  │                                                                  │   │
│  │  Overall Quality Score (0-100):                                 │   │
│  │    = (Completeness * 0.5) + (Accuracy * 0.5)                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                    Product with quality
                    score & metadata
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                  LAYER 6: PYDANTIC VALIDATION                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ProductData Model (src/models/product.py)                       │   │
│  │  • Type validation (str, float, int, datetime)                  │   │
│  │  • Range validation (price > 0, rating 1-5)                    │   │
│  │  • Required field checking                                      │   │
│  │  • Final data integrity check                                   │   │
│  │  Output: Structured ProductData object                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                   Final structured
                   ProductData object
                             │
┌────────────────────────────▼────────────────────────────────────────────┐
│                 LAYER 7: STORAGE & REPORTING                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Optional: PostgreSQL Database                                    │   │
│  │  ├─ Bronze Layer: bronze_raw_products                           │   │
│  │  │  └─ Raw data, full audit trail                              │   │
│  │  └─ Silver Layer: silver_products                              │   │
│  │     └─ Cleaned, deduplicated data                              │   │
│  │                                                                  │   │
│  │ Always: JSON Report & Logs                                      │   │
│  │  ├─ scrape_report_*.json - Summary                             │   │
│  │  └─ scraper.log - Detailed logs                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                       OUTPUT FILES
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
  scraper.log    scrape_report_*.json    PostgreSQL DB
  (detailed      (summary JSON)          (if enabled)
   logs)
```

## Data Flow - Detailed Example

### Input (scrapeme.json)
```json
{
  "url": "https://example.com/products",
  "site_id": "example_1",
  "selectors": {
    "product_container": "div.product",
    "product_name": "h2.title",
    "product_price": "span.price"
  }
}
```

### Step 1: Fetch HTML
```
GET https://example.com/products
Response: 200 OK
Body: <html><div class="product"><h2 class="title">Laptop</h2>...
```

### Step 2: Parse HTML
```
CSS Selector: "div.product"
Found: 20 product containers
```

### Step 3: Extract Data (Raw)
```python
{
  'product_name': '  Laptop Pro   ',  # Extra spaces
  'product_price': '$1,299.99',       # Currency symbol
  'product_url': '/products/laptop',  # Relative URL
}
```

### Step 4: Clean Data
```python
{
  'product_name': 'Laptop Pro',           # Spaces removed
  'product_price': 1299.99,               # Converted to float
  'product_url': 'https://example.com/products/laptop',  # Absolute URL
}
```

### Step 5: Validate
```
✓ product_name present ✓ product_price > 0 ✓ product_url valid
→ All checks passed, is_valid = True
```

### Step 6: Quality Score
```
Completeness: 85% (5/6 fields present)
Accuracy: 100% (no validation issues)
Overall: 92.5%
```

### Step 7: Pydantic Model
```python
ProductData(
    product_id='...',
    product_name='Laptop Pro',
    product_price=1299.99,
    product_url='https://example.com/products/laptop',
    is_valid=True,
    quality_issues=[],
    scraped_at=datetime.now()
)
```

### Step 8: Store & Report
```
Bronze Table: INSERT INTO bronze_raw_products VALUES (...)
Silver Table: INSERT INTO silver_products VALUES (...)
JSON Report: scrape_report_20240115_103050.json
Log: 2024-01-15 10:30:50 - INFO - ✓ Stored product
```

## Error Handling Flow

```
                    Attempt to fetch URL
                           │
                    ┌──────┴──────┐
                    │             │
                  Success       Error
                    │             │
                    ▼             ▼
              Parse HTML      Check retry count
                    │             │
                    │      ┌──────┴──────┐
                    │      │             │
                    │    < 3 retries   ≥ 3 retries
                    │      │             │
                    │      ▼             ▼
                    │   Wait & Retry   Skip Page
                    │      │             │
                    └──────┴─────────────┘
                           │
                           ▼
                    Log error & continue
```

## Quality Metrics Calculation

### Completeness Example

```
Product: {
  'product_id': 'ABC123',              ✓ required
  'product_name': 'Laptop',            ✓ required
  'product_price': 999.99,             ✓ required
  'product_url': 'https://...',        ✓ required
  'product_rating': 4.5,               ✓ optional
  'product_review_count': 200,         ✓ optional
  # Missing: product_discount, product_description
}

Required: 4/4 = 100% → 100% * 0.7 = 70 points
Optional: 2/7 = 28.6% → 28.6% * 0.3 = 8.6 points
Total: 70 + 8.6 = 78.6% completeness
```

### Accuracy Example

```
Product: {
  'product_price': 999.99,             ✓ price > 0
  'product_rating': 4.5,               ✓ rating 1-5
  'product_original_price': 500.00,    ✗ original < current
}

Issues: 1
Accuracy = 100 - (1 * 10) = 90%
```

### Overall Score

```
Quality = (Completeness * 0.5) + (Accuracy * 0.5)
Quality = (78.6 * 0.5) + (90 * 0.5)
Quality = 39.3 + 45 = 84.3%
```

## Configuration Hierarchy

```
scrapeme.json
    │
    ├─ global_settings (applied to all sites)
    │   ├─ user_agent
    │   ├─ timeout
    │   ├─ retry_count
    │   └─ delay_between_requests
    │
    └─ sites[]
        ├─ site_name
        └─ product_pages[]
            ├─ url (specific page)
            ├─ site_id
            ├─ selectors (CSS extractors)
            └─ pagination (page-specific)
```

## Database Schema Layers

### Bronze Layer (Raw)
```
bronze_raw_products
├─ id (serial)
├─ site_id
├─ site_name
├─ raw_data (JSONB)      ← Full backup
├─ scraped_at            ← Timestamp
└─ (all product fields)

bronze_scrape_logs
├─ id
├─ site_id
├─ page_url
├─ products_scraped
├─ success (boolean)
└─ errors (text)
```

### Silver Layer (Cleaned)
```
silver_products
├─ id
├─ site_id
├─ product_id           ← UNIQUE per site
├─ product_name
├─ (cleaned fields)
├─ is_valid             ← Quality flag
├─ quality_issues[]     ← Array of problems
├─ is_duplicate         ← Dedup flag
└─ last_updated
```

### Gold Layer (Analytics)
```
gold_daily_prices
├─ product_id
├─ price_date
├─ current_price
├─ price_change
├─ discount_pct
└─ (aggregated metrics)

gold_category_stats
├─ category
├─ stat_date
├─ avg_price
├─ products_on_discount
└─ (category metrics)
```

---

This architecture ensures:
✅ **Traceability** - Every product has full history
✅ **Quality** - Multiple validation layers
✅ **Scalability** - Can add more sites/pages easily
✅ **Maintainability** - Clear separation of concerns
✅ **Robustness** - Comprehensive error handling