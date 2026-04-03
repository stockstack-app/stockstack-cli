# Disclosure Analysis Patterns

Example patterns for retrieving and analyzing annual securities reports (有価証券報告書) using the StockStack CLI. These are starting points — adapt the `jq` filters to suit your specific analysis needs.

## Workflow

Disclosure content retrieval follows a 3-step flow:

1. `filings list <ticker>` → Identify `filingId` from the filing list
2. `disclosure sections <filingId>` → Identify `blockKey` from section/block index
3. `disclosure block <filingId> <blockKey>` → Retrieve block content

## Finding and Retrieving Filings

### List Filings

```bash
# List all annual reports
stockstack filings list 7203

# Get the latest filing ID
stockstack filings list 7203 | jq -r '.filings[0].filingId'
# → "S100VWVY"
```

### Browse Sections and Blocks

```bash
# Section list with block index
stockstack disclosure sections S100VWVY

# List all block keys and labels
stockstack disclosure sections S100VWVY | jq '.sections[].blocks[] | {key, label}'
```

### Retrieve Block Content

```bash
# Get block content (markdown format, default)
stockstack disclosure block S100VWVY BusinessRisksTextBlock

# Get as plain text
stockstack disclosure block S100VWVY BusinessRisksTextBlock --content-format text
```

## Analysis Patterns

### Business Risk Investigation

```bash
# Find risk-related blocks in the "business" section
stockstack disclosure sections S100VWVY | \
  jq '.sections[] | select(.key == "business") | .blocks[] | {key, label}'

# Retrieve risk content as text
stockstack disclosure block S100VWVY BusinessRisksTextBlock --content-format text
```

### Officer Information

```bash
# Find blocks in the "reporting_company" section
stockstack disclosure sections S100VWVY | \
  jq '.sections[] | select(.key == "reporting_company") | .blocks[] | {key, label}'

# Retrieve officer information
stockstack disclosure block S100VWVY InformationAboutOfficersTextBlock
```

### Year-over-Year Comparison

Output is JSON — always extract text with `jq` before processing. Do not pipe raw JSON through `head` as it breaks the structure.

```bash
# Compare business risks across the last 3 filings
for filing_id in $(stockstack filings list 7203 | jq -r '.filings[:3][].filingId'); do
  echo "=== $filing_id ==="
  stockstack disclosure block "$filing_id" BusinessRisksTextBlock | \
    jq -r '.block.items[0].content' | head -30
  echo ""
done
```

### Peer Comparison

```bash
# Compare business risks between Toyota and Honda
for ticker in 7203 7267; do
  filing_id=$(stockstack filings list "$ticker" | jq -r '.filings[0].filingId')
  echo "=== $ticker ($filing_id) ==="
  stockstack disclosure block "$filing_id" BusinessRisksTextBlock | \
    jq -r '.block.items[0].content' | head -20
done
```

## JSON Response Structures

### Sections List

```json
{
  "sections": [
    {
      "key": "business",
      "label": "事業の状況",
      "blocks": [
        { "key": "BusinessRisksTextBlock", "label": "事業等のリスク" },
        { "key": "BusinessPolicyBusinessEnvironmentIssuesToAddressEtcTextBlock", "label": "経営方針、経営環境及び対処すべき課題等" }
      ]
    }
  ]
}
```

### Block Content

```json
{
  "filing": { "filingId": "S100VWVY", "ticker": "7203", "companyName": "トヨタ自動車", "periodEnd": "2025-03-31" },
  "block": {
    "key": "BusinessRisksTextBlock",
    "label": "事業等のリスク",
    "items": [{ "content": "当社グループの事業に関するリスク..." }]
  },
  "format": "markdown"
}
```

### jq Recipes for Disclosure

```bash
# List block labels within a specific section
stockstack disclosure sections S100VWVY | \
  jq '.sections[] | select(.key == "business") | .blocks[].label'

# Extract text content from a block
stockstack disclosure block S100VWVY BusinessRisksTextBlock | jq -r '.block.items[].content'

# Blocks with multiple items (dimensioned) — check titles
stockstack disclosure block S100VWVY SomeTextBlock | \
  jq '.block.items[] | {title, snippet: (.content | .[0:80])}'
```

## Major Shareholders

Available as structured data via a separate command (not disclosure blocks).

```bash
stockstack corporate shareholders 7203

# Top 5 shareholders by holding ratio (latest period)
stockstack corporate shareholders 7203 | \
  jq '.periods[0].shareholders[:5][] | {name, holdingRatio}'
```

## Tips

- **markdown vs text**: Default is markdown, which preserves tables and lists. Use `--content-format text` for plain text
- **3-step flow**: Always follow `filings list` → `disclosure sections` → `disclosure block`
- **Block keys**: Correspond to XBRL element names; `label` is the Japanese display name
- **Granularity**: Blocks are the smallest retrievable unit — more efficient than fetching entire sections
