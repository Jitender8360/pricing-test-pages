# Local Testing Environment for price_ai_bot

This directory contains HTML pricing pages and testing infrastructure to validate all production challenge scenarios locally before monitoring real competitor websites.

## Quick Start

### 1. Start the Test Server

```bash
cd test-pages
python server.py 8080
```

The server will display all available test pages at startup.

### 2. Run the Tests

Follow the comprehensive guide in **[LOCAL_TESTING_GUIDE.md](LOCAL_TESTING_GUIDE.md)** which covers:

- ✅ Regional pricing variations (US, EU)
- ✅ Language handling (English, French)
- ✅ Tax treatment differences (VAT included vs excluded)
- ✅ Shadow DOM content extraction
- ✅ Price change detection
- ✅ Content hashing validation

## Test Pages

| File | Purpose | URL |
|------|---------|-----|
| **pricing-us.html** | US pricing (tax excluded) | http://localhost:8080/pricing-us.html |
| **pricing-eu.html** | EU pricing (VAT included) | http://localhost:8080/pricing-eu.html |
| **pricing-fr.html** | French version (same pricing) | http://localhost:8080/pricing-fr.html |
| **pricing-shadow-dom.html** | Shadow DOM test | http://localhost:8080/pricing-shadow-dom.html |
| **pricing-us-v2.html** | Updated pricing (for change detection) | http://localhost:8080/pricing-us-v2.html |

## What Each Test Page Simulates

### pricing-us.html
- **Challenge:** Regional pricing - US market
- **Features:**
  - Prices in USD
  - Tax excluded (sales tax at checkout)
  - 3 plans: Starter ($29), Professional ($99), Enterprise ($299)
  - 14-day free trial
  - 20% annual discount

### pricing-eu.html
- **Challenge:** Regional pricing - EU market
- **Features:**
  - Prices in EUR
  - VAT included (20%)
  - Same plan structure as US
  - Tests tax treatment differences

### pricing-fr.html
- **Challenge:** Language variations
- **Features:**
  - Same pricing as EU version
  - All text in French
  - Tests that language changes don't trigger false positives

### pricing-shadow-dom.html
- **Challenge:** JavaScript rendering and Shadow DOM
- **Features:**
  - Pricing cards rendered in Shadow DOM
  - Requires `SCRAPBEE_RENDER_JS=true`
  - Tests content extraction from web components

### pricing-us-v2.html
- **Challenge:** Price change detection
- **Features:**
  - Updated prices (Starter: $39, Professional: $119)
  - New "Business" plan added ($199)
  - Extended trial period (30 days)
  - Increased annual discount (25%)
  - Changed CTA text
  - Tests comprehensive change detection

## Example Test Workflows

### Test 1: Scrape All Regions

```bash
# Scrape US
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }'

# Scrape EU
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_eu",
    "target_url": "http://localhost:8080/pricing-eu.html"
  }'

# View all snapshots
./get-snapshots.sh testcompany_us testcompany_eu
```

### Test 2: Detect Price Changes

```bash
# Scrape version 1
V1=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-us.html"}' \
  | jq -r '.snapshot_id')

# Scrape version 2
V2=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-us-v2.html"}' \
  | jq -r '.snapshot_id')

# Analyze changes
curl -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d "{
    \"previous_snapshot_id\": \"$V1\",
    \"current_snapshot_id\": \"$V2\",
    \"ignore_competitor\": true
  }" | jq
```

### Test 3: Compare Languages

```bash
# Scrape English
EN=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test_en", "target_url": "http://localhost:8080/pricing-us.html"}' \
  | jq -r '.snapshot_id')

# Scrape French
FR=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test_fr", "target_url": "http://localhost:8080/pricing-fr.html"}' \
  | jq -r '.snapshot_id')

# Compare with LLM
curl -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d "{
    \"previous_snapshot_id\": \"$EN\",
    \"current_snapshot_id\": \"$FR\",
    \"ignore_competitor\": true
  }" | jq '.analysis.summary'
```

## Limitations

### ScrapingBee and Localhost

ScrapingBee is a cloud service and cannot directly access `localhost:8080`. To work around this:

**Option 1: Use ngrok** (Recommended for testing)
```bash
# Install ngrok: https://ngrok.com/download
ngrok http 8080

# Use the ngrok URL:
# https://abc123.ngrok.io/pricing-us.html
```

**Option 2: Deploy to public hosting**
- Upload HTML files to GitHub Pages, Netlify, or Vercel
- Use the public URLs in your tests

**Option 3: Bypass ScrapingBee for local testing**
Temporarily modify `shared/scrape_client.py` to use local httpx for localhost URLs.

## Validating Test Results

### Check Snapshots Created
```bash
docker exec price_ai_bot-postgres-1 psql -U postgres -d priceai -c "
  SELECT source_id, COUNT(*) as count
  FROM snapshots
  GROUP BY source_id
  ORDER BY count DESC;
"
```

### View S3 Storage
```bash
aws s3 ls s3://your-bucket/sources/ --recursive | grep testcompany
```

### Compare Pricing Data
```bash
# Download pricing.json for comparison
SNAPSHOT_ID="your-snapshot-id"
aws s3 cp s3://your-bucket/sources/testcompany_us/snapshots/$SNAPSHOT_ID/pricing.json - | jq
```

## Expected Outcomes

After running all tests, you should have:

1. **Multiple snapshots** for different regions/languages
2. **Validated** that:
   - Same content produces same hash
   - Different content produces different hash
   - Language changes are handled correctly
   - Tax treatments are detected
   - Shadow DOM content is extracted
   - Price changes are accurately detected

3. **LLM analysis** showing:
   - Tax-aware comparisons
   - Language recognition
   - Price change summaries
   - Business implications

## Next Steps

Once local testing is complete:

1. **Deploy test pages publicly** (GitHub Pages, Netlify)
2. **Test with real ScrapingBee** using public URLs
3. **Configure Parallel Monitor** for delta service testing
4. **Apply learnings** to real competitor monitoring

## Troubleshooting

### Port Already in Use
```bash
# Use a different port
python server.py 8090
```

### JS Rendering Not Working
```bash
# Ensure in .env:
SCRAPBEE_RENDER_JS=true

# Restart services
./services.sh restart
```

### No Snapshots Created
```bash
# Check service logs
./services.sh logs pricing-scrape-full

# Verify database connection
docker compose ps
```

## Additional Resources

- **[LOCAL_TESTING_GUIDE.md](LOCAL_TESTING_GUIDE.md)** - Comprehensive step-by-step guide
- **[explanation.md](../explanation.md)** - Full architecture documentation
- **[TESTING.md](../TESTING.md)** - General testing guide

---

**Happy Testing!** 🚀

These local test pages let you validate the entire pricing monitoring system before pointing it at real competitor websites.
