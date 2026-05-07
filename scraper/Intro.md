# Quick Start Guide - 5 Minutes to Running Scraper

## Step 1: Setup (2 min)

```bash
# Clone/download the project
cd ecommerce-scraper

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

## Step 2: Configure Sites (2 min)

### Option A: Use Example Config
```bash
cp scrapeme.example.json scrapeme.json
# Edit scrapeme.json with your target sites
```

### Option B: Manual Config
Create `scrapeme.json`:

```json
{
  "sites": [
    {
      "site_name": "My Store",
      "base_url": "https://mystore.com",
      "product_pages": [
        {
          "url": "https://mystore.com/products",
          "site_id": "my_store_1",
          "selectors": {
            "product_container": "div.product",
            "product_id": "data-id",
            "product_name": "h2.name",
            "product_price": "span.price",
            "product_url": "a.link"
          },
          "pagination": {
            "enabled": true,
            "next_page_selector": "a.next",
            "max_pages": 3
          }
        }
      ]
    }
  ],
  "global_settings": {
    "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "timeout": 10,
    "retry_count": 3,
    "delay_between_requests": 2,
    "use_proxy": false,
    "proxy_list": [],
    "headless_browser": false,
    "browser_type": "chromium"
  }
}
```

## Step 3: Find CSS Selectors (1 min per site)

1. **Open the website** in your browser
2. **Right-click** on a product → **Inspect** (or press F12)
3. **Find the product container** - the `<div>` or element wrapping each product
   - Example: `<div class="product-item">` → selector is `div.product-item`
4. **Find each field selector**:
   - Product name: Look for `<h2 class="title">` → `h2.title`
   - Price: Look for `<span class="price">` → `span.price`
   - Link: Look for `<a href="...">` → `a`
5. **Copy selectors to scrapeme.json**

### Selector Format Tips
- **Class**: `div.class-name` or `.class-name`
- **ID**: `#id-name`
- **Attribute**: `[data-id]`
- **Nested**: `div.container h2.title`
- **Test**: Open console, type: `document.querySelectorAll('your-selector')`

## Step 4: Run Scraper (Immediate)

```bash
python main.py
```

### Expected Output:
```
2024-01-15 10:30:45 - __main__ - INFO - Loading configuration from scrapeme.json
2024-01-15 10:30:45 - __main__ - INFO - ✓ Configuration loaded and validated
2024-01-15 10:30:46 - __main__ - INFO - ============================================================
2024-01-15 10:30:46 - __main__ - INFO - STARTING SCRAPING PIPELINE
2024-01-15 10:30:46 - __main__ - INFO - ============================================================
2024-01-15 10:30:47 - src.scrapers.scraper - INFO - Fetching: https://mystore.com/products
2024-01-15 10:30:48 - src.scrapers.scraper - INFO - ✓ Successfully fetched
2024-01-15 10:30:48 - src.scrapers.scraper - INFO - Found 20 product containers
2024-01-15 10:30:48 - src.scrapers.scraper - INFO - ✓ Scrape complete: 18/20 valid in 2.45s

My Store - https://mystore.com/products
  Scraped: 20
  Valid: 18
  Invalid: 2
  Duration: 2.45s

============================================================
SCRAPING SUMMARY
============================================================
Total products scraped: 20
Total valid: 18
Total invalid: 2
Success rate: 90.0%
✓ Report generated: scrape_report_20240115_103050.json
✓ Application completed successfully
```

## Output Files

After running, check:
- **scraper.log** - Detailed logs
- **scrape_report_*.json** - Summary results
- **product data** - Formatted JSON output (if customized)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "No products found" | Check CSS selectors in browser - they might have changed |
| "Connection timeout" | Increase `timeout` setting, or check if site is blocking |
| "Invalid data" | Check cleaning rules in `src/processors/cleaner.py` |
| "Import errors" | Ensure virtual env is activated: `source venv/bin/activate` |

## What's in the Data?

Each product includes:
```json
{
  "product_id": "ABC123",
  "product_name": "Laptop Pro",
  "product_price": 999.99,
  "product_original_price": 1499.99,
  "product_discount": 33.33,
  "product_stock": "in_stock",
  "product_rating": 4.5,
  "product_review_count": 250,
  "product_image": "https://...",
  "product_url": "https://mystore.com/product/abc123",
  "product_description": "High-performance laptop...",
  "quality_issues": [],
  "is_valid": true
}
```

## Next Steps

1. **Add more sites** - Copy the `product_pages` section and add new URLs
2. **Enable database** - Follow database setup in README.md
3. **Schedule scrapes** - Use cron or Task Scheduler
4. **Build dashboard** - Use Streamlit to visualize data
5. **ML optimization** - Add discount recommendation engine

---

## Common CSS Selector Patterns

```html
<!-- Example HTML -->
<div class="product-item" data-id="123">
  <h2 class="product-title">Product Name</h2>
  <span class="price">$99.99</span>
  <a href="/products/123" class="product-link">View</a>
</div>
```

| Element | Selector |
|---------|----------|
| Div with class | `div.product-item` |
| Element with ID | `#product-123` |
| Data attribute | `[data-id]` |
| Nested element | `div.product-item h2.product-title` |
| Child element | `div.product-item > h2` |
| Attribute selector | `a[href*="/products"]` |

## Tips for Success

✅ **Always test selectors** in browser console first  
✅ **Add delays** between requests (default 2 sec)  
✅ **Check robots.txt** at `site.com/robots.txt`  
✅ **Handle pagination** - Set `max_pages` carefully  
✅ **Monitor logs** - Check `scraper.log` for issues  
✅ **Start small** - Test with 1 page before expanding  

---

**Ready to scrape!** Run `python main.py` 🚀