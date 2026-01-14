# Amazon Orders to Odoo - Quick Start Guide

## What's in this folder?

This folder contains everything you need to process Amazon orders and inject them into Odoo, matching your existing Chilly Water Square orders workflow.

## Files Overview

### 📘 Documentation
- **README.md** (this file) - Quick start guide
- **DATA_PREP_GUIDE.md** - Detailed data transformation guide
- **SKU_MAPPING.md** - Product SKU to name mapping
- **WORKFLOW_ARCHITECTURE.md** - Complete system design

### 🔧 Workflows
- **transform_workflow.json** - Sample transformation code

## Quick Start (5 Steps)

### Step 1: Get Your Amazon SKUs ⚡
**Action Required:** You need to find the actual SKUs Amazon uses for your products.

1. Go to Amazon Seller Central
2. Navigate to **Inventory → Manage Inventory**
3. Find these products and note their SKUs:
   - 12 Pack - Purified Chilly Water (Regular)
   - 24 Pack - Purified Chilly Water (Regular)
   - 12 Pack - Sparkling Chilly Water
   - 6 Pack - Purified Chilly Water

4. Update `SKU_MAPPING.md` with actual SKUs

### Step 2: Enhance Order Fetching
**Current Status:** Your existing workflow fetches orders but not line items.

**What to do:**
1. Open your existing `get_amazon_orders.json` workflow
2. Add a new HTTP Request node after "Get Orders"
3. Configure it to fetch order items:

```
URL: https://sellingpartnerapi-na.amazon.com/orders/v0/orders/{{ $json.AmazonOrderId }}/orderItems
Method: GET
Headers:
  - x-amz-access-token: {{ $node["HTTP Request"].json["access_token"] }}
Authentication: AWS (Assume Role) - use your existing credentials
```

4. Add a Code node to merge order + items data

### Step 3: Add Transformation
1. Add a Code node to your workflow
2. Copy the transformation code from `transform_workflow.json`
3. Update the SKU mapping with your actual SKUs from Step 1
4. Test with a sample order

### Step 4: Connect to Existing Odoo Workflow
Your transformed data will be in this format:
```json
{
  "order": { ... },
  "line_items_structured": [ ... ]
}
```

This matches exactly what your existing Odoo workflow (`2_main.json`) expects!

**Integration Options:**

**Option A: Call as Sub-Workflow**
- Use "Execute Workflow" node
- Target: Your existing Odoo workflow
- Pass transformed data

**Option B: Direct Integration**
- Copy relevant nodes from your Odoo workflow
- Paste into Amazon workflow
- Connect transformation output to Odoo nodes

### Step 5: Test & Deploy
1. Test with 1 order manually
2. Verify order appears in Odoo correctly
3. Check contact, address, and products
4. Set up scheduled trigger (e.g., every 2 hours)
5. Add error notifications

## What Gets Transformed?

### Amazon Order → Your Format

```javascript
// Amazon SP-API provides this:
{
  AmazonOrderId: "123-456-789",
  ShippingAddress: {
    Name: "John Doe",
    AddressLine1: "123 Main St",
    City: "San Francisco",
    StateOrRegion: "CA",
    PostalCode: "94115"
  },
  OrderItems: [
    { SellerSKU: "CW-24PACK", QuantityOrdered: 1 }
  ]
}

// Gets transformed to:
{
  order: {
    id: "123-456-789",
    reference_id: "123-456-789",
    state: "OPEN",
    fulfillments: [{
      shipment_details: {
        recipient: {
          display_name: "John Doe",
          address: {
            address_line_1: "123 Main St",
            locality: "San Francisco",
            administrative_district_level_1: "CA",
            postal_code: "94115",
            country: "US",
            first_name: "John",
            last_name: "Doe"
          }
        },
        placed_at: "2026-01-13T10:30:00Z"
      }
    }],
    source: { name: "Amazon" }
  },
  line_items_structured: [
    {
      name: "24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)",
      quantity: "1"
    }
  ]
}
```

## Critical Things to Know

### ⚠️ Amazon Limitations
- **Email may be masked** - Amazon protects buyer privacy
- **Phone may be missing** - Not always provided
- **Solution:** Set defaults or make optional in Odoo

### ✅ What Works Great
- Address structure is similar to Square
- Order status mapping is straightforward
- Product quantity is always available
- Your existing Odoo workflow handles everything else

### 🎯 Product Filtering
Only these products will be processed:
- ✓ 12-Pack Regular Chilly Water
- ✓ 24-Pack Regular Chilly Water
- ✓ 12-Pack Sparkling Chilly Water
- ✓ 6-Pack Regular Chilly Water

All other products will be ignored (filtered out).

## Workflow Diagram

```
[Fetch Amazon Orders]
         ↓
[Get Order Items]
         ↓
[Transform to Odoo Format]
         ↓
[Filter Chilly Water Only]
         ↓
[Your Existing Odoo Workflow]
         ↓
[Order Created in Odoo! ✓]
```

## Troubleshooting

### Issue: "No line items found"
- Make sure you're fetching order items (separate API call)
- Check order status is "Unshipped"
- Verify API permissions

### Issue: "Product not recognized"
- Update SKU mapping with actual Amazon SKUs
- Check product title if SKU match fails
- Verify product is active in Odoo

### Issue: "Address validation failed"
- Check state abbreviation format
- Verify country code (should be "US")
- Look for special characters

### Issue: "Duplicate order"
- Check if order already exists in Odoo
- Look at `client_order_ref` field
- May need to add duplicate detection

## What to Do Next

### Today
- [ ] Get Amazon SKUs from Seller Central
- [ ] Update SKU_MAPPING.md

### This Week
- [ ] Add order items fetch to existing workflow
- [ ] Add transformation code node
- [ ] Test with 1-2 real orders manually

### Next Week
- [ ] Connect to Odoo workflow
- [ ] Run end-to-end test
- [ ] Set up automation & monitoring

## Need Help?

1. **Data Structure Questions?** → Read `DATA_PREP_GUIDE.md`
2. **SKU Mapping Issues?** → Check `SKU_MAPPING.md`
3. **Architecture Overview?** → See `WORKFLOW_ARCHITECTURE.md`
4. **Workflow Design?** → Review `transform_workflow.json`

## Pro Tips

💡 **Start Small:** Test with just one order type (e.g., 12-pack regular) first

💡 **Use Filters:** Add a filter node after transformation to only process specific SKUs during testing

💡 **Log Everything:** Add error logging to catch issues early

💡 **Duplicate Check:** Before creating Odoo order, check if `client_order_ref` already exists

💡 **Monitor SKUs:** Set up alerts when unknown SKUs are detected

## Success Checklist

You'll know it's working when:
- ✅ Amazon orders appear in Odoo automatically
- ✅ Customer addresses are correct
- ✅ Products match your Odoo catalog
- ✅ Shipment carriers are set correctly
- ✅ No duplicate orders are created
- ✅ Only Chilly Water orders are processed

---

**Questions?** Review the detailed guides in this folder or check your existing workflow for reference.

**Ready to start?** Begin with Step 1 above! 🚀
