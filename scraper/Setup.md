# E-Commerce Web Scraper - Minimum Viable Product (MVP)

A production-ready web scraper for e-commerce sites with data cleaning, validation, and storage capabilities.

## 🎯 Features

✅ **Multi-site scraping** - Configure multiple e-commerce sites
✅ **CSS selector-based extraction** - Flexible, easy to adapt to new sites
✅ **Data cleaning** - Normalize prices, ratings, discounts automatically
✅ **Data validation** - Pydantic models ensure data quality
✅ **Bronze/Silver layers** - Medallion architecture for data governance
✅ **Quality scoring** - Completeness and accuracy metrics
✅ **Error handling** - Robust retry logic and error logging
✅ **Database storage** - PostgreSQL integration (optional)

## 📋 Project Structure

```
ecommerce-scraper/
├── scrapeme.json              # Configuration: URLs and CSS selectors
├── main.py                    # Entry point
├── requirements.txt           # Python dependencies
├── scraper.log               # Application logs
│
├── src/
│   ├── scrapers/
│   │   └── scraper.py        # Web scraping pipeline
│   │
│   ├── processors/
│   │   └── cleaner.py        # Data cleaning & validation
│   │
│   ├── models/
│   │   └── product.py        # Pydantic data models
│   │
│   ├── database/
│   │   └── db.py             # Database connection & schema
│   │
│   └── config/
│       └── settings.py       # Configuration management
│
└── tests/
    └── test_*.py             # Unit tests
```

## 🚀 Quick Start

### 1. Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install packages
pip install -r requirements.txt
```

### 2. Configure Sites (scrapeme.json)

Edit `scrapeme.json` with your target e-commerce sites:

```json
{
  "sites": [
    {
      "site_name": "My Store",
      "base_url": "https://example.com",
      "product_pages": [
        {
          "url": "https://example.com/products/electronics",
          "site_id": "my_store_1",
          "selectors": {
            "product_container": "div.product-item",
            "product_id": "data-product-id",
            "product_name": "h2.title",
            "product_price": "span.price",
            "product_url": "a.product-link"
          },
          "pagination": {
            "enabled": true,
            "next_page_selector": "a.next",
            "max_pages": 5
          }
        }
      ]
    }
  ],
  "global_settings": {
    "user_agent": "Mozilla/5.0...",
    "timeout": 10,
    "retry_count": 3,
    "delay_between_requests": 2
  }
}
```

### 3. Run Scraper

```bash
# Simple run (no database required)
python main.py

# Or with database:
# 1. Ensure PostgreSQL is running
# 2. Create database: createdb ecommerce_scraper
# 3. Set credentials in main.py or environment variables
# 4. Run: python main.py
```

### 4. Output

The scraper generates:
- **scrape_report_*.json** - Summary of scraping results
- **scraper.log** - Detailed logs
- **Database tables** (if enabled):
  - `bronze_raw_products` - Raw scraped data
  - `bronze_scrape_logs` - Scraping metadata
  - `silver_products` - Cleaned, validated data

## 📝 scrapeme.json Configuration Guide

### Global Settings

```json
{
  "global_settings": {
    "user_agent": "Mozilla/5.0...",      // Browser user agent
    "timeout": 10,                        // Request timeout in seconds
    "retry_count": 3,                     // Retry failed requests
    "delay_between_requests": 2,          // Delay between requests (prevent blocking)
    "use_proxy": false,                   // Enable proxy rotation
    "proxy_list": [],                     // List of proxies
    "headless_browser": false,            // Use headless browser (for JS)
    "browser_type": "chromium"            // Browser type if headless enabled
  }
}
```

### Site Configuration

Each site must specify:
- `site_name` - Human-readable site name
- `base_url` - Base URL of the site
- `product_pages` - List of pages to scrape

### CSS Selectors

For each product page, specify CSS selectors for data extraction:

```json
{
  "selectors": {
    "product_container": "div.product",    // Container for each product
    "product_id": "data-id",               // Can be attribute or selector
    "product_name": "h2.title",            // Product title
    "product_price": "span.price",         // Current price
    "product_original_price": "span.old",  // Original price (optional)
    "product_discount": "span.discount",   // Discount % (optional)
    "product_stock": "span.stock",         // Stock status (optional)
    "product_rating": "span.rating",       // Rating (optional)
    "product_review_count": "span.reviews", // Review count (optional)
    "product_image": "img.product-img",    // Product image (optional)
    "product_url": "a.link",               // Product URL
    "product_description": "p.desc"        // Description (optional)
  }
}
```

### Finding CSS Selectors

1. Open site in browser
2. Right-click on product element → "Inspect" (or press F12)
3. Copy the CSS class or data attribute
4. Test in browser console: `document.querySelectorAll('your-selector')`

Example:
```html
<div class="product-item" data-sku="ABC123">
  <h2 class="product-title">Product Name</h2>
  <span class="current-price">$99.99</span>
  <a href="/products/abc" class="product-url">View</a>
</div>
```

CSS selectors:
```json
{
  "product_container": "div.product-item",
  "product_id": "data-sku",
  "product_name": "h2.product-title",
  "product_price": "span.current-price",
  "product_url": "a.product-url"
}
```

## 🔍 Data Extraction Examples

### Price Handling
- **Input**: "$99.99", "€50,50", "100 USD"
- **Output**: 99.99, 50.50, 100.00 (float)

### Discount Handling
- **Input**: "50%", "50% OFF", "Save 50%"
- **Output**: 50.0 (percentage)

### Rating Handling
- **Input**: "4.5", "4.5/5", "4.5 stars"
- **Output**: 4.5 (float, 1-5 scale)

### Stock Status
- **Input**: "In Stock", "out of stock", "Limited"
- **Output**: "in_stock", "out_of_stock", "limited_stock"

## 📊 Data Quality

Each product gets a quality score (0-100):
- **Completeness**: % of fields populated
- **Accuracy**: Validation checks (no negative prices, valid rating range, etc.)
- **Overall**: Average of completeness and accuracy

Quality issues are logged in `quality_issues` field:
```json
{
  "quality_issues": [
    "Missing product_rating",
    "Price seems unrealistic"
  ]
}
```

## 🗄️ Database Schema (Optional)

If using PostgreSQL, three layers are created:

### Bronze Layer (Raw)
- Immutable original data
- Full HTML backup for re-processing
- Timestamps and source tracking

### Silver Layer (Cleaned)
- Normalized data types
- Quality flags
- Deduplication tracking

### Gold Layer (Analytics)
- Aggregated daily prices
- Category statistics
- Price history & trends

## 🛠️ Customization

### Add New Site

1. Find product container CSS selector (e.g., `div.product-item`)
2. Find all field selectors (name, price, rating, etc.)
3. Add to `scrapeme.json`:

```json
{
  "site_name": "New Store",
  "product_pages": [
    {
      "url": "https://newstore.com/products",
      "site_id": "new_store_1",
      "selectors": {
        "product_container": "...",
        "product_name": "...",
        ...
      }
    }
  ]
}
```

4. Run scraper: `python main.py`

### Adjust Cleaning Rules

Edit `src/processors/cleaner.py`:
- `DataCleaner` - Price, rating, discount extraction
- `DataValidator` - Validation rules
- `QualityScorer` - Quality metrics

### Add Custom Fields

1. Update `src/models/product.py` - Add field to ProductData
2. Update `scrapeme.json` - Add selector
3. Update `src/scrapers/scraper.py` - Add cleaning logic

## 📈 Next Steps (Production Features)

- [ ] **Airflow orchestration** - Schedule daily scrapes
- [ ] **Discount optimization** - ML model for optimal prices
- [ ] **Streamlit dashboard** - Visualize price trends
- [ ] **API endpoint** - Expose data via REST API
- [ ] **Proxy rotation** - Advanced anti-detection
- [ ] **Playwright integration** - Handle JavaScript
- [ ] **Cloud deployment** - Docker + Kubernetes

## ⚠️ Ethical Scraping

- ✅ Check site's `robots.txt` and terms of service
- ✅ Respect rate limits (add delays between requests)
- ✅ Use appropriate User-Agent
- ✅ Don't overload servers
- ✅ Credit the source
- ✅ Consider alternatives (APIs, data feeds)

## 🐛 Troubleshooting

### "Connection timeout"
- Increase `timeout` in settings
- Check internet connection
- Site may be blocking requests

### "No products found"
- Verify CSS selectors are correct
- Check site structure hasn't changed
- Enable browser (set `headless_browser: true`)

### "Data looks wrong"
- Review `quality_issues` in output
- Check data cleaning rules
- Inspect actual HTML in browser

### "Database connection failed"
- Ensure PostgreSQL is running
- Check credentials
- Create database: `createdb ecommerce_scraper`

## 📚 Resources

- [BeautifulSoup Documentation](https://www.crummy.com/software/BeautifulSoup/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [CSS Selectors Guide](https://www.w3schools.com/cssref/selectors.php)

## 📝 License

MIT License - Use freely for academic and commercial projects

## 🤝 Contributing

Contributions welcome! Areas to improve:
- Better error handling
- More cleaning rules
- Database optimization
- Test coverage
- Documentation

---

**Built for academic and professional data engineering projects**