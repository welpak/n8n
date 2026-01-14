# Amazon Orders Workflow Architecture

## Overview

This document outlines the complete workflow architecture for processing Amazon orders and injecting them into Odoo, matching the existing Chilly Water Square orders workflow.

## Workflow Components

### 1. Get Amazon Orders (Fetch)
**File:** `get_amazon_orders.json` (your current workflow)

**Purpose:** Fetch unshipped orders from Amazon SP-API

**Current Implementation:**
- Authenticates with Amazon OAuth2
- Fetches orders from last 5 days
- Filters by status: Unshipped

**Output:** Array of Amazon orders with basic info (no line items yet)

---

### 2. Get Order Items (Enrich)
**File:** `get_order_items.json` (needs to be created)

**Purpose:** For each order, fetch the line items (products)

**Implementation Needed:**
```javascript
// Loop through orders from step 1
// For each order, call: GET /orders/v0/orders/{orderId}/orderItems
// Merge order data with its items
```

**API Endpoint:**
```
GET https://sellingpartnerapi-na.amazon.com/orders/v0/orders/{orderId}/orderItems
Headers:
  - x-amz-access-token: {access_token}
```

---

### 3. Transform Data (Normalize)
**File:** `transform_amazon_to_odoo.json` (provided in this folder)

**Purpose:** Convert Amazon data structure to match Odoo workflow format

**Key Transformations:**
1. **Order Structure**
   - Map Amazon order fields to Square-like structure
   - Add `order.source.name = "Amazon"`

2. **Address Normalization**
   - Split full name into first/last
   - Map address fields
   - Handle missing phone/email

3. **Line Items Mapping**
   - Filter for Chilly Water products only (by SKU or title)
   - Map to standardized product names
   - Ensure quantity is string format

4. **Fulfillment Details**
   - Create fulfillment object with shipment details
   - Set shipping type to "FLAT"

**Output:** Data structure identical to Square orders format

---

### 4. Filter Chilly Water Orders (Quality Gate)
**File:** Part of transform step

**Purpose:** Only process orders containing Chilly Water products

**Logic:**
- Check if any line items match known SKUs
- Skip orders with no Chilly Water products
- Alert on unknown SKUs (for manual review)

---

### 5. Inject into Odoo (Execute)
**File:** Use existing `2_main.json` workflow

**Purpose:** Create contacts, orders, and shipments in Odoo

**Integration Point:**
- The transformed data from step 3 can be directly passed to your existing workflow
- The existing workflow already handles:
  - Country/State lookup
  - Contact creation/lookup
  - Order creation
  - Product mapping
  - Shipment setup

---

## Complete Workflow Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. FETCH ORDERS (Existing)                                  │
│    - Get Amazon OAuth token                                 │
│    - Fetch unshipped orders (last 5 days)                  │
│    Output: Orders without line items                        │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. ENRICH WITH LINE ITEMS (New)                            │
│    - Loop through each order                                │
│    - Fetch order items for each                            │
│    - Merge order + items                                    │
│    Output: Complete order data                              │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. TRANSFORM DATA (New)                                     │
│    - Map Amazon structure to Square-like format            │
│    - Split name, normalize address                          │
│    - Map SKUs to product names                             │
│    - Filter Chilly Water products only                      │
│    Output: Odoo-ready format                                │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. QUALITY CHECK (New)                                      │
│    - Verify required fields present                         │
│    - Check for valid products                               │
│    - Alert on anomalies                                     │
│    Output: Validated orders                                 │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. INJECT INTO ODOO (Existing)                             │
│    - Execute existing workflow (2_main.json)                │
│    - Creates contacts                                       │
│    - Creates orders                                         │
│    - Sets up shipments                                      │
│    Output: Odoo sales orders                                │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Strategy

### Phase 1: Build & Test Data Pipeline
1. ✅ Create `amazonOrders` folder
2. ⏳ Update `get_amazon_orders.json` to fetch order items
3. ⏳ Create transformation workflow
4. ⏳ Test with sample data
5. ⏳ Verify output matches Square format exactly

### Phase 2: SKU Mapping
1. ⏳ Get actual Amazon SKUs from Seller Central
2. ⏳ Update `SKU_MAPPING.md` with real SKUs
3. ⏳ Test SKU→Product name mapping
4. ⏳ Add fallback logic for title matching

### Phase 3: Integration
1. ⏳ Connect transformation output to existing Odoo workflow
2. ⏳ Test end-to-end with real order
3. ⏳ Verify order created correctly in Odoo
4. ⏳ Check shipment details are accurate

### Phase 4: Automation & Monitoring
1. ⏳ Set up scheduled trigger (every X hours)
2. ⏳ Add error handling & notifications
3. ⏳ Create logging/audit trail
4. ⏳ Set up alerts for failed orders

## Key Differences: Amazon vs Square

| Aspect | Square Orders | Amazon Orders |
|--------|--------------|---------------|
| **Data Source** | Square API | Amazon SP-API |
| **Order ID** | reference_id | AmazonOrderId |
| **Line Items** | Included in order | Separate API call |
| **Customer Info** | Full details | May be masked |
| **Address** | Structured | Structured (similar) |
| **Product ID** | By name | By SKU (must map) |
| **Status** | OPEN/CLOSED | Unshipped/Shipped/etc |

## Critical Considerations

### 1. Amazon Contact Info Limitations
- **Issue:** Amazon may mask buyer email
- **Solution:** Use Amazon-provided email proxy or set default
- **Code:**
  ```javascript
  email_address: buyerInfo.BuyerEmail || 'amazon-order@yourcompany.com'
  ```

### 2. Phone Number Availability
- **Issue:** Phone may not be in API response
- **Solution:** Make phone optional or use placeholder
- **Code:**
  ```javascript
  phone_number: shippingAddress.Phone || '+10000000000'
  ```

### 3. Name Splitting
- **Issue:** Amazon provides full name, Odoo needs first/last
- **Solution:** Smart splitting with fallback
- **Code:**
  ```javascript
  const parts = name.trim().split(' ');
  first_name: parts[0] || 'Customer',
  last_name: parts.slice(1).join(' ') || parts[0]
  ```

### 4. Unknown SKUs
- **Issue:** New products or SKU changes
- **Solution:** Alert + manual review workflow
- **Code:**
  ```javascript
  if (!productMapping[sku]) {
    // Log to error channel
    // Skip order or create ticket
  }
  ```

### 5. Duplicate Orders
- **Issue:** Re-processing same order
- **Solution:** Check `client_order_ref` in Odoo before creating
- **Implementation:** Add duplicate check node before order creation

## Testing Checklist

Before going live, test these scenarios:

- [ ] Order with single 12-pack product
- [ ] Order with multiple quantities
- [ ] Order with multiple different products
- [ ] Order with sparkling variant
- [ ] Order with missing phone number
- [ ] Order with masked email
- [ ] International order (non-US)
- [ ] Order with apartment/suite in address
- [ ] Order with hyphenated or special name
- [ ] Order that's already in Odoo (duplicate test)

## Monitoring & Alerts

Set up notifications for:
- Failed order fetches
- Unknown SKUs detected
- Transformation errors
- Odoo creation failures
- Missing required fields
- Duplicate order attempts

## Next Steps

1. **Immediate:** Get actual Amazon SKUs and update mapping
2. **Today:** Create order items fetch workflow
3. **Tomorrow:** Test transformation with real data
4. **This Week:** Complete end-to-end integration test
5. **Next Week:** Deploy to production with monitoring

## Support & Troubleshooting

### Common Issues

**Problem:** "Order items not found"
- Check order ID is correct
- Verify API permissions include order items
- Check order status allows item access

**Problem:** "Product not found in Odoo"
- Verify SKU mapping is correct
- Check Odoo product IDs are accurate
- Ensure product is active in Odoo

**Problem:** "Address validation failed"
- Check state/country mapping
- Verify address format
- Look for special characters in address

## Documentation References

- Amazon SP-API Orders: https://developer-docs.amazon.com/sp-api/docs/orders-api-v0-reference
- Your existing workflow: `/workflows/2_main.json`
- SKU mapping: `/workflows/amazonOrders/SKU_MAPPING.md`
- Data prep guide: `/workflows/amazonOrders/DATA_PREP_GUIDE.md`
