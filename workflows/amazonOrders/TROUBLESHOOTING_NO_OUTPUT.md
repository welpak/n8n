# Quick Fix: Why No Output?

## The Problem

Your transformation code is returning no output because:

1. **Missing Order Items**: The code expects `OrderItems` in the input data, but Amazon requires a **separate API call** to get them
2. **Empty line items**: Without OrderItems, the code creates an empty `lineItemsStructured` array
3. **Filter returns nothing**: The code has this line: `if (lineItemsStructured.length === 0) { return []; }`

## The Solution (2 Options)

### Option A: Add "Get Order Items" Node (Recommended)

You need to add a node BEFORE the transformation that fetches order items for each order.

**Steps:**
1. Add a new HTTP Request node between "Get Orders" and "Transform"
2. Configure it as shown below
3. Then run the transformation

### Option B: Test with Mock Data First

Update your transformation code to handle missing OrderItems and use test data.

---

## Immediate Fix: Updated Transformation Code

Replace your current code with this version that:
- ✅ Handles missing OrderItems gracefully
- ✅ Shows helpful error messages
- ✅ Works with your current data structure
- ✅ Provides debugging output

```javascript
// This node transforms Amazon SP-API order data into the format expected by Odoo workflow

// SKU to Product Mapping - UPDATE WITH YOUR ACTUAL SKUS
const SKU_TO_PRODUCT = {
  'YOUR-12PACK-SKU': {
    name: '12 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6648
  },
  'YOUR-24PACK-SKU': {
    name: '24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6794
  },
  'YOUR-12SPARK-SKU': {
    name: '12 Pack - Sparkling Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6827
  },
  'YOUR-6PACK-SKU': {
    name: '6 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6983
  }
};

// Alternative: Match by title if SKUs are inconsistent
function matchProductByTitle(title) {
  const titleLower = title.toLowerCase();

  if (titleLower.includes('sparkling') && titleLower.includes('12')) {
    return SKU_TO_PRODUCT['YOUR-12SPARK-SKU'];
  }
  if (titleLower.includes('24')) {
    return SKU_TO_PRODUCT['YOUR-24PACK-SKU'];
  }
  if (titleLower.includes('6')) {
    return SKU_TO_PRODUCT['YOUR-6PACK-SKU'];
  }
  if (titleLower.includes('12')) {
    return SKU_TO_PRODUCT['YOUR-12PACK-SKU'];
  }
  return null;
}

// Get Amazon order data from various possible input structures
const inputData = $input.item.json;
const amazonOrder = inputData.Order || inputData.Orders?.[0] || inputData;

// Check if OrderItems exist
let orderItems = inputData.OrderItems || inputData.payload?.OrderItems || amazonOrder.OrderItems || [];

// DEBUGGING: Log what we received
console.log('===== DEBUG INFO =====');
console.log('Input keys:', Object.keys(inputData));
console.log('Amazon Order ID:', amazonOrder.AmazonOrderId);
console.log('OrderItems found:', orderItems.length);
console.log('OrderItems:', JSON.stringify(orderItems, null, 2));

// If no order items, return error message
if (!orderItems || orderItems.length === 0) {
  throw new Error(`❌ No OrderItems found for order ${amazonOrder.AmazonOrderId}.

You need to fetch order items first using:
GET /orders/v0/orders/${amazonOrder.AmazonOrderId}/orderItems

Current input structure: ${Object.keys(inputData).join(', ')}

Add a 'Get Order Items' HTTP Request node BEFORE this transformation node.`);
}

// Split full name into first and last
function splitName(fullName) {
  const parts = (fullName || '').trim().split(' ');
  return {
    firstName: parts[0] || '',
    lastName: parts.slice(1).join(' ') || parts[0] || ''
  };
}

const shippingAddress = amazonOrder.ShippingAddress || {};
const buyerInfo = amazonOrder.BuyerInfo || {};
const nameParts = splitName(shippingAddress.Name);

// Transform line items
const lineItemsStructured = orderItems
  .map(item => {
    console.log(`Processing item: ${item.SellerSKU} - ${item.Title}`);

    // Try SKU mapping first
    let product = SKU_TO_PRODUCT[item.SellerSKU];

    // Fallback to title matching
    if (!product) {
      product = matchProductByTitle(item.Title);
      if (product) {
        console.log(`✓ Matched by title: ${item.Title}`);
      }
    } else {
      console.log(`✓ Matched by SKU: ${item.SellerSKU}`);
    }

    // Skip if not a Chilly Water product
    if (!product) {
      console.log(`✗ Skipped (not Chilly Water): ${item.Title}`);
      return null;
    }

    return {
      name: product.name,
      quantity: item.QuantityOrdered.toString()
    };
  })
  .filter(item => item !== null);

console.log('Chilly Water items found:', lineItemsStructured.length);
console.log('======================');

// If no Chilly Water products found, skip this order
if (lineItemsStructured.length === 0) {
  throw new Error(`⚠️ Order ${amazonOrder.AmazonOrderId} has no Chilly Water products. Skipping.

OrderItems received: ${orderItems.map(i => i.Title).join(', ')}`);
}

// Build the transformed order object
const transformedOrder = {
  order: {
    state: 'OPEN',
    reference_id: amazonOrder.AmazonOrderId,
    id: amazonOrder.AmazonOrderId,
    fulfillments: [
      {
        uid: amazonOrder.AmazonOrderId,
        type: 'SHIPMENT',
        state: 'PROPOSED',
        line_item_application: 'ALL',
        shipment_details: {
          recipient: {
            display_name: shippingAddress.Name || 'Unknown',
            email_address: buyerInfo.BuyerEmail || '',
            phone_number: shippingAddress.Phone || '',
            address: {
              address_line_1: shippingAddress.AddressLine1 || '',
              address_line_2: shippingAddress.AddressLine2 || '',
              locality: shippingAddress.City || '',
              administrative_district_level_1: shippingAddress.StateOrRegion || '',
              postal_code: shippingAddress.PostalCode || '',
              country: shippingAddress.CountryCode || 'US',
              first_name: nameParts.firstName,
              last_name: nameParts.lastName
            }
          },
          shipping_type: 'FLAT',
          placed_at: amazonOrder.PurchaseDate
        }
      }
    ],
    source: {
      name: 'Amazon'
    },
    net_amount_due_money: {
      amount: 0,
      currency: 'USD'
    }
  },
  line_items_structured: lineItemsStructured
};

console.log('✅ Transformation successful!');
console.log('Output:', JSON.stringify(transformedOrder, null, 2));

return [{ json: transformedOrder }];
```

## What This New Code Does

1. **Better Input Detection**: Checks multiple possible locations for order data
2. **Clear Error Messages**: Tells you exactly what's missing
3. **Debug Logging**: Shows what it's processing (check browser console)
4. **Helpful Errors**: Explains what to do next

## Next: Add "Get Order Items" Node

You'll see an error message after running this that tells you exactly what API call to make. Follow that to add the missing node.

Would you like me to create the complete workflow with the "Get Order Items" node included?
