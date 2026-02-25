# Local Testing Guide - Production Challenge Scenarios

This guide helps you test all production challenges using local HTML pages.

## Setup

### 1. Start the Local Test Server

```bash
cd test-pages
python server.py 8080
```

The server will be available at `http://localhost:8080`

### 2. Verify Server is Running

Open your browser and visit:
- http://localhost:8080/pricing-us.html

You should see a pricing page with 3 plans.

---

## Test Scenarios

### Challenge 1(a): Regional Pricing - Different URLs

**Scenario:** Test scraping different regional pricing pages.

#### Step 1: Scrape US Pricing

```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }' | jq '.snapshot_id'
```

**Expected:**
- Prices in USD ($29, $99, $299)
- Tax note: "Sales tax calculated at checkout"
- Region badge: "🇺🇸 United States"

#### Step 2: Scrape EU Pricing

```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_eu",
    "target_url": "http://localhost:8080/pricing-eu.html"
  }' | jq '.snapshot_id'
```

**Expected:**
- Prices in EUR (€29, €99, €299)
- Tax note: "VAT included (20%)"
- Region badge: "🇪🇺 Europe"

#### Step 3: Compare Regional Snapshots

```bash
# Get the last 2 snapshot IDs
./get-snapshots.sh

# Use LLM analyzer to compare
curl -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "previous_snapshot_id": "US_SNAPSHOT_ID",
    "current_snapshot_id": "EU_SNAPSHOT_ID",
    "ignore_competitor": false
  }' | jq '.analysis.price_comparison'
```

**What to verify:**
- ✅ Both snapshots stored separately
- ✅ Same price amounts but different currencies
- ✅ Different tax treatment detected
- ✅ LLM recognizes this as competitive comparison

**SQL Query to verify:**
```sql
-- Check both regions stored
docker exec price_ai_bot-postgres-1 psql -U postgres -d priceai -c "
  SELECT source_id, content_hash, created_at
  FROM snapshots
  WHERE source_id LIKE 'testcompany_pricing_%'
  ORDER BY created_at DESC
  LIMIT 5;
"
```

---

### Challenge 2: Language-Specific Changes

**Scenario:** Same pricing, different language shouldn't trigger false positives.

#### Step 1: Scrape English Version

```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_en",
    "target_url": "http://localhost:8080/pricing-us.html"
  }' | jq
```

Save the `snapshot_id`.

#### Step 2: Scrape French Version

```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_fr",
    "target_url": "http://localhost:8080/pricing-fr.html"
  }' | jq
```

Save the `snapshot_id`.

#### Step 3: Compare with LLM Analyzer

```bash
curl -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "previous_snapshot_id": "ENGLISH_SNAPSHOT_ID",
    "current_snapshot_id": "FRENCH_SNAPSHOT_ID",
    "ignore_competitor": true
  }' | jq '.analysis'
```

**What to verify:**
- ✅ Structured pricing data (amounts) should be identical
- ✅ LLM should note: "Same pricing, different language"
- ✅ Plan names: "Starter" vs "Débutant" (translated)
- ✅ Features: "Up to 5 users" vs "Jusqu'à 5 utilisateurs"

**Expected LLM Output:**
```json
{
  "analysis": {
    "price_changes": [],
    "language_difference": true,
    "summary": "No material pricing changes. Content is the same pricing structure presented in French instead of English."
  }
}
```

---

### Challenge 3: VAT / Tax Inclusion Differences

**Scenario:** Compare US (tax excluded) vs EU (VAT included) pricing.

#### Step 1: Scrape Both Regions

```bash
# US (tax excluded)
US_ID=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }' | jq -r '.snapshot_id')

# EU (VAT included)
EU_ID=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_eu",
    "target_url": "http://localhost:8080/pricing-eu.html"
  }' | jq -r '.snapshot_id')

echo "US Snapshot: $US_ID"
echo "EU Snapshot: $EU_ID"
```

#### Step 2: Use LLM for Tax-Aware Comparison

```bash
curl -X POST http://localhost:8004/competitive-analysis \
  -H "Content-Type: application/json" \
  -d "{
    \"competitors\": {
      \"TestCompany US\": \"$US_ID\",
      \"TestCompany EU\": \"$EU_ID\"
    }
  }" | jq '.analysis'
```

**What to verify:**
- ✅ LLM recognizes tax treatment differences
- ✅ Shows: "$29 (tax excluded)" vs "€29 (VAT included 20%)"
- ✅ Calculates: EU base price = €29 / 1.20 = €24.17 before VAT
- ✅ Insight: "EU customers pay similar base price but with VAT included"

**Manual Verification:**

Download and compare pricing.json files:
```bash
# Get S3 keys from response
aws s3 cp s3://your-bucket/sources/testcompany_us/snapshots/$US_ID/pricing.json us-pricing.json
aws s3 cp s3://your-bucket/sources/testcompany_eu/snapshots/$EU_ID/pricing.json eu-pricing.json

# Compare
diff <(jq -S '.plans[0].prices' us-pricing.json) <(jq -S '.plans[0].prices' eu-pricing.json)
```

---

### Challenge 4: JavaScript Rendering (Shadow DOM)

**Scenario:** Test that Shadow DOM content is properly extracted.

#### Step 1: Scrape Shadow DOM Page

**Important:** Ensure `SCRAPBEE_RENDER_JS=true` in your `.env` file!

```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_shadow_dom",
    "target_url": "http://localhost:8080/pricing-shadow-dom.html"
  }' | jq
```

#### Step 2: Verify Content Was Extracted

```bash
# Get the snapshot ID from previous response
SHADOW_ID="YOUR_SNAPSHOT_ID"

# Download the clean.md to verify content
aws s3 cp s3://your-bucket/sources/testcompany_shadow_dom/snapshots/$SHADOW_ID/clean.md - | grep -i "shadow dom"
```

**What to verify:**
- ✅ Plan names include "(Shadow DOM)" text
- ✅ Prices are extracted: $29, $99, $299
- ✅ Features are captured
- ✅ `chunks_stored` > 0 (indicates content was extracted)

**If JS rendering is disabled:**
```bash
# You'll see empty or minimal content
aws s3 cp s3://your-bucket/sources/testcompany_shadow_dom/snapshots/$SHADOW_ID/clean.md - | wc -l
# Should be > 50 lines if JS rendering works
# Will be < 10 lines if it doesn't
```

**Test with JS rendering disabled (to see the difference):**

1. Edit `.env`: `SCRAPBEE_RENDER_JS=false`
2. Restart service
3. Scrape again:
```bash
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_shadow_dom_no_js",
    "target_url": "http://localhost:8080/pricing-shadow-dom.html"
  }' | jq '.chunks_stored'
# Should be 0 or very low
```

---

### Challenge 5: Price Change Detection

**Scenario:** Detect actual pricing changes between versions.

#### Step 1: Scrape Version 1 (Original Pricing)

```bash
V1_ID=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }' | jq -r '.snapshot_id')

echo "Version 1 Snapshot: $V1_ID"
```

#### Step 2: Scrape Version 2 (Updated Pricing)

```bash
V2_ID=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us-v2.html"
  }' | jq -r '.snapshot_id')

echo "Version 2 Snapshot: $V2_ID"
```

#### Step 3: Analyze Changes with Deterministic Analyzer

```bash
curl -X POST http://localhost:8003/analyze \
  -H "Content-Type: application/json" \
  -d "{
    \"previous_snapshot_id\": \"$V1_ID\",
    \"current_snapshot_id\": \"$V2_ID\"
  }" | jq
```

**Expected Signals:**
```json
{
  "signals": {
    "price_changes": {
      "Starter": {"previous": 29, "current": 39, "change_pct": 34.5},
      "Professional": {"previous": 99, "current": 119, "change_pct": 20.2}
    },
    "plans_added": ["Business"],
    "plans_removed": [],
    "trial_changes": {
      "trial_duration_increased": true,
      "previous": "14-day",
      "current": "30-day"
    },
    "emphasis_changes": {
      "annual_discount_increased": true,
      "previous": "20%",
      "current": "25%"
    }
  }
}
```

#### Step 4: Analyze Changes with LLM Analyzer

```bash
curl -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d "{
    \"previous_snapshot_id\": \"$V1_ID\",
    \"current_snapshot_id\": \"$V2_ID\",
    \"ignore_competitor\": true
  }" | jq '.analysis'
```

**Expected LLM Output:**
```json
{
  "analysis": {
    "price_changes": [
      {
        "plan": "Starter",
        "change_type": "increase",
        "previous_amount": 29,
        "current_amount": 39,
        "percentage_change": 34.5,
        "context": "Significant price increase on entry-level plan"
      }
    ],
    "plan_changes": {
      "added": ["Business"],
      "removed": [],
      "renamed": []
    },
    "business_implications": {
      "revenue_impact": "Price increases and new Business tier suggest upmarket positioning",
      "customer_impact": "Entry-level customers face 34.5% price increase; may cause churn",
      "strategic_direction": "Adding mid-tier Business plan to capture SMB segment"
    },
    "summary": "Significant pricing restructure with new Business tier and 20-35% price increases"
  }
}
```

#### Step 5: Generate Comparison Report

```bash
curl -X POST http://localhost:8004/generate-report \
  -H "Content-Type: application/json" \
  -d "{
    \"previous_snapshot_id\": \"$V1_ID\",
    \"current_snapshot_id\": \"$V2_ID\",
    \"ignore_competitor\": true
  }" | jq -r '.report_markdown' > pricing-change-report.md

# View the report
cat pricing-change-report.md
```

---

## Content Hash Testing

**Scenario:** Verify that identical content produces same hash.

### Test 1: Same Content = Same Hash

```bash
# Scrape the same page twice
HASH1=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-us.html"}' \
  | jq -r '.content_hash')

sleep 2

HASH2=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-us.html"}' \
  | jq -r '.content_hash')

if [ "$HASH1" = "$HASH2" ]; then
  echo "✅ Content hashing works: Same content = Same hash"
  echo "Hash: $HASH1"
else
  echo "❌ Problem: Same content produced different hashes"
  echo "Hash 1: $HASH1"
  echo "Hash 2: $HASH2"
fi
```

### Test 2: Different Content = Different Hash

```bash
# Scrape two different pages
HASH_US=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-us.html"}' \
  | jq -r '.content_hash')

HASH_EU=$(curl -s -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{"source_id": "test", "target_url": "http://localhost:8080/pricing-eu.html"}' \
  | jq -r '.content_hash')

if [ "$HASH_US" != "$HASH_EU" ]; then
  echo "✅ Content hashing works: Different content = Different hash"
  echo "US Hash:  $HASH_US"
  echo "EU Hash:  $HASH_EU"
else
  echo "❌ Problem: Different content produced same hash"
fi
```

---

## Delta Service Testing

**Scenario:** Test two-tier change detection (Parallel + Hash comparison).

**Note:** This requires Parallel Monitor setup, which needs a publicly accessible URL. For local testing, you can simulate the hash comparison part.

### Simulate Hash Comparison

```bash
# Setup: Store initial state
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }'

# Query the hash from database
docker exec price_ai_bot-postgres-1 psql -U postgres -d priceai -c "
  SELECT content_hash
  FROM snapshots
  WHERE source_id = 'testcompany_pricing_us'
  ORDER BY created_at DESC
  LIMIT 1;
"

# Scrape again (same content)
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us.html"
  }' | jq '.content_hash'

# ✅ Hash should match - no new snapshot created (deduplication works)

# Scrape v2 (different content)
curl -X POST http://localhost:8001/run \
  -H "Content-Type: application/json" \
  -d '{
    "source_id": "testcompany_pricing_us",
    "target_url": "http://localhost:8080/pricing-us-v2.html"
  }' | jq '.content_hash'

# ✅ Hash should differ - new snapshot created
```

---

## Complete Workflow Test

Run all scenarios in sequence:

```bash
#!/bin/bash

echo "🧪 Running Complete Production Challenge Test Suite"
echo "===================================================="

# 1. Regional Pricing
echo "\n📍 Test 1: Regional Pricing"
US_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_us", "target_url": "http://localhost:8080/pricing-us.html"}' | jq -r '.snapshot_id')
EU_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_eu", "target_url": "http://localhost:8080/pricing-eu.html"}' | jq -r '.snapshot_id')
echo "✅ US Snapshot: $US_ID"
echo "✅ EU Snapshot: $EU_ID"

# 2. Language Variations
echo "\n🌐 Test 2: Language Variations"
EN_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_en", "target_url": "http://localhost:8080/pricing-us.html"}' | jq -r '.snapshot_id')
FR_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_fr", "target_url": "http://localhost:8080/pricing-fr.html"}' | jq -r '.snapshot_id')
echo "✅ English Snapshot: $EN_ID"
echo "✅ French Snapshot: $FR_ID"

# 3. Shadow DOM
echo "\n👻 Test 3: Shadow DOM Content Extraction"
SHADOW_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_shadow", "target_url": "http://localhost:8080/pricing-shadow-dom.html"}' | jq -r '.snapshot_id')
echo "✅ Shadow DOM Snapshot: $SHADOW_ID"

# 4. Price Changes
echo "\n💰 Test 4: Price Change Detection"
V1_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_changes", "target_url": "http://localhost:8080/pricing-us.html"}' | jq -r '.snapshot_id')
V2_ID=$(curl -s -X POST http://localhost:8001/run -H "Content-Type: application/json" -d '{"source_id": "test_changes", "target_url": "http://localhost:8080/pricing-us-v2.html"}' | jq -r '.snapshot_id')
echo "✅ V1 Snapshot: $V1_ID"
echo "✅ V2 Snapshot: $V2_ID"

# 5. Run Analysis
echo "\n📊 Test 5: Running LLM Analysis on Price Changes"
curl -s -X POST http://localhost:8004/analyze \
  -H "Content-Type: application/json" \
  -d "{\"previous_snapshot_id\": \"$V1_ID\", \"current_snapshot_id\": \"$V2_ID\", \"ignore_competitor\": true}" \
  | jq '.analysis.summary'

echo "\n✅ All tests completed!"
echo "\n📈 Database Summary:"
docker exec price_ai_bot-postgres-1 psql -U postgres -d priceai -c "
  SELECT source_id, COUNT(*) as snapshot_count
  FROM snapshots
  WHERE source_id LIKE 'test_%'
  GROUP BY source_id;
"
```

Save this as `run-all-tests.sh` and execute:
```bash
chmod +x run-all-tests.sh
./run-all-tests.sh
```

---

## Troubleshooting

### Server won't start
```bash
# Check if port 8080 is already in use
lsof -i :8080

# Use a different port
python server.py 8090
```

### ScrapingBee can't reach localhost

**Problem:** ScrapingBee is a cloud service and can't access `localhost:8080`.

**Solutions:**

**Option 1: Use ngrok to expose local server**
```bash
# Install ngrok: https://ngrok.com/download
ngrok http 8080

# Use the ngrok URL in your tests:
# https://abc123.ngrok.io/pricing-us.html
```

**Option 2: Deploy test pages to a public server**
```bash
# Upload to GitHub Pages, Netlify, or Vercel
# Then use the public URL
```

**Option 3: Skip ScrapingBee for local testing**

Modify `shared/scrape_client.py` to detect localhost and use local fetch:
```python
if "localhost" in url or "127.0.0.1" in url:
    # Use local httpx fetch instead of ScrapingBee
    response = httpx.get(url)
    return [{"html": response.text}]
```

### Shadow DOM content not extracted

Ensure:
1. `SCRAPBEE_RENDER_JS=true` in `.env`
2. Restart services after changing `.env`
3. Wait 2-3 seconds for JS to execute

---

## Next Steps

After testing locally:

1. **Deploy test pages** to a public URL (GitHub Pages, Netlify)
2. **Configure Parallel Monitor** to watch the public URLs
3. **Test delta service** with real change detection
4. **Scale to real competitors** using the same patterns

---

## Summary

You now have a complete local testing environment for:
- ✅ Regional pricing variations
- ✅ Language handling
- ✅ Tax treatment differences
- ✅ Shadow DOM content extraction
- ✅ Price change detection
- ✅ Content hashing and deduplication

These tests validate your system works correctly before monitoring real competitor sites!
