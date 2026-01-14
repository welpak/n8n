# Visual Workflow Guide - Amazon Orders to Odoo

## 🔴 Current Problem

```
[Get Orders] → [Transform] → ❌ No Output
                    ↑
                    Missing OrderItems!
```

Your transformation node has **no order items** to process, so it returns nothing.

## ✅ Complete Solution

```
[Get OAuth Token]
       ↓
[Get Orders] ──────────┐
       ↓                │
[Split Orders]         │ (passes order data)
       ↓                │
[Get Order Items]      │ (fetches line items for each order)
       ↓                │
[Merge]←───────────────┘ (combines order + items)
       ↓
[Transform to Odoo Format] ← Now has complete data!
       ↓
[Your Existing Odoo Workflow]
```

---

## Step-by-Step Fix

### Step 1: Add "Split Orders" Node

**Why?** Amazon returns an array of orders. You need to process each order individually.

**Node Type:** Split Out
**Field to split:** `payload.Orders`

### Step 2: Add "Get Order Items" Node

**Why?** Amazon doesn't include line items in the orders response. It's a separate API call.

**Node Type:** HTTP Request

**Configuration:**
```
URL: https://sellingpartnerapi-na.amazon.com/orders/v0/orders/{{ $json.AmazonOrderId }}/orderItems

Method: GET

Headers:
  x-amz-access-token: {{ $node['Get OAuth Token'].json.access_token }}

Authentication: AWS (Assume Role) - use your existing credentials
```

### Step 3: Add "Merge" Node

**Why?** Combine the order data with its items into one object.

**Node Type:** Merge
**Mode:** Combine
**Combination Mode:** Merge By Position

**Inputs:**
- Input 1: Order data (from Split Orders)
- Input 2: Order items (from Get Order Items)

### Step 4: Update Transform Node

Your transform node should now access data like this:

```javascript
// Get order from first input
const amazonOrder = $('Split Orders').item.json;

// Get items from second input (merged)
const orderItemsResponse = $json;
const orderItems = orderItemsResponse.payload?.OrderItems || [];
```

---

## Quick Import Instructions

### Option A: Import Complete Workflow

1. Copy content from `complete_workflow_with_items.json`
2. In n8n, click **"+ Add workflow"** → **"Import from JSON"**
3. Paste the JSON
4. Update your AWS credentials
5. Test!

### Option B: Add Missing Nodes to Existing Workflow

Your current workflow:
```
[Get OAuth Token] → [Get Orders] → [Transform] ❌
```

Add these nodes:
```
[Get OAuth Token] → [Get Orders] → [NEW: Split Orders] → [NEW: Get Order Items]
                                            ↓                        ↓
                                    [NEW: Merge] ← ─────────────────┘
                                            ↓
                                    [Transform] ✅
```

---

## Data Flow Diagram

### What Amazon Gives You

**Step 1 - Get Orders Response:**
```json
{
  "payload": {
    "Orders": [
      {
        "AmazonOrderId": "123-456",
        "ShippingAddress": {...},
        "BuyerInfo": {...}
        // ❌ NO OrderItems HERE!
      }
    ]
  }
}
```

**Step 2 - Get Order Items Response:**
```json
{
  "payload": {
    "OrderItems": [
      {
        "SellerSKU": "CW-24PACK",
        "Title": "24 Pack Chilly Water",
        "QuantityOrdered": 1
      }
    ]
  }
}
```

**Step 3 - After Merge:**
```json
{
  // Order data
  "AmazonOrderId": "123-456",
  "ShippingAddress": {...},

  // Items data (from second input)
  "payload": {
    "OrderItems": [...]
  }
}
```

**Step 4 - After Transform:**
```json
{
  "order": {
    "id": "123-456",
    "fulfillments": [{...}],
    "source": {"name": "Amazon"}
  },
  "line_items_structured": [
    {
      "name": "24 Pack - Purified Chilly Water...",
      "quantity": "1"
    }
  ]
}
```

---

## Testing Checklist

After adding the missing nodes:

- [ ] Token fetched successfully
- [ ] Orders retrieved (check: `payload.Orders` exists)
- [ ] Orders split (should see multiple items if multiple orders)
- [ ] Order items fetched for each order
- [ ] Data merged correctly
- [ ] Transform produces output
- [ ] Output matches expected Odoo format

---

## Common Issues & Solutions

### Issue: "Node execution failed"
**Cause:** Missing AWS credentials on "Get Order Items" node
**Fix:** Add the same AWS (Assume Role) credentials as "Get Orders"

### Issue: "AmazonOrderId is undefined"
**Cause:** Wrong data reference in URL
**Fix:** Make sure URL uses: `{{ $json.AmazonOrderId }}`

### Issue: "No OrderItems in response"
**Cause:** Order might be too old or already shipped
**Fix:** Check order status, try with recent Unshipped orders

### Issue: "Transform still returns no output"
**Cause:** SKU mapping doesn't match your actual products
**Fix:**
1. Log the SKUs: `console.log(item.SellerSKU)`
2. Update `SKU_TO_PRODUCT` mapping
3. Or rely on title matching fallback

### Issue: "Cannot read property 'access_token'"
**Cause:** Wrong node reference in headers
**Fix:** Use: `{{ $node['Get OAuth Token'].json.access_token }}`

---

## Performance Notes

- **Rate Limits:** Amazon SP-API has rate limits. Add error handling for 429 errors.
- **Batching:** If processing many orders, consider adding delays between requests.
- **Caching:** The OAuth token is valid for 1 hour. Consider caching it.

---

## Next Steps

1. **Today:** Import the complete workflow JSON or add missing nodes
2. **Test:** Run with one order and verify output
3. **Configure:** Update SKU mappings with your actual Amazon SKUs
4. **Integrate:** Connect output to your existing Odoo workflow
5. **Automate:** Add schedule trigger to run every few hours

---

## Need Help?

### Debug Mode

Add this at the start of your Transform code to see what data you're getting:

```javascript
console.log('===== DEBUG =====');
console.log('Order:', JSON.stringify($('Split Orders').item.json, null, 2));
console.log('Items Response:', JSON.stringify($json, null, 2));
console.log('=================');
```

Then check browser console (F12) when running the workflow.

### Test with Mock Data

Create a manual trigger with pinned test data:

```json
{
  "AmazonOrderId": "TEST-123",
  "ShippingAddress": {
    "Name": "Test Customer",
    "AddressLine1": "123 Main St",
    "City": "San Francisco",
    "StateOrRegion": "CA",
    "PostalCode": "94115",
    "CountryCode": "US"
  },
  "BuyerInfo": {
    "BuyerEmail": "test@example.com"
  },
  "PurchaseDate": "2026-01-14T10:00:00Z",
  "payload": {
    "OrderItems": [
      {
        "SellerSKU": "YOUR-24PACK-SKU",
        "Title": "24 Pack Chilly Water",
        "QuantityOrdered": 1
      }
    ]
  }
}
```

This lets you test the transformation without calling Amazon API.

---

## Files in This Folder

- `README.md` - Quick start guide
- `complete_workflow_with_items.json` - **Use this!** Complete working workflow
- `TROUBLESHOOTING_NO_OUTPUT.md` - Detailed explanation of the no-output issue
- `VISUAL_WORKFLOW_GUIDE.md` - This file (visual guide)
- `DATA_PREP_GUIDE.md` - Data transformation details
- `SKU_MAPPING.md` - Product mapping reference
- `WORKFLOW_ARCHITECTURE.md` - System architecture

**Start here:** Import `complete_workflow_with_items.json` into n8n!
