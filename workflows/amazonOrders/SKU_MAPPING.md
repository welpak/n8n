# Amazon SKU to Product Name Mapping

## Current Active SKUs (From Screenshot)

Based on your Amazon inventory, here are the Chilly Water products to process:

### 12-Pack Products
```
SKU: CW-12PACK-REG (or similar)
Product Name: "12 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)"
Odoo Product ID: 6648
```

### 24-Pack Products
```
SKU: CW-24PACK-REG (or similar)
Product Name: "24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)"
Odoo Product ID: 6794
```

### 12-Pack Sparkling
```
SKU: CW-12PACK-SPARK (or similar)
Product Name: "12 Pack - Sparkling Chilly Water - 16oz Cans (SHIPPING INCLUDED)"
Odoo Product ID: 6827
```

### 6-Pack Products
```
SKU: CW-6PACK-REG (or similar)
Product Name: "6 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)"
Odoo Product ID: 6983
```

## Action Required

**IMPORTANT:** You need to verify the actual SKU values Amazon uses for these products.

To find your actual SKUs:
1. Go to Amazon Seller Central
2. Navigate to Inventory → Manage Inventory
3. Note the exact SKU for each Chilly Water product
4. Update the mapping below

## SKU Mapping Object (for n8n Code Node)

```javascript
// TODO: Replace these SKU keys with actual Amazon SKUs
const SKU_TO_PRODUCT = {
  // 12-Pack Regular (Replace 'YOUR-12PACK-SKU' with actual SKU)
  'YOUR-12PACK-SKU': {
    name: '12 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6648,
    packSize: 12,
    type: 'regular'
  },

  // 24-Pack Regular (Replace 'YOUR-24PACK-SKU' with actual SKU)
  'YOUR-24PACK-SKU': {
    name: '24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6794,
    packSize: 24,
    type: 'regular'
  },

  // 12-Pack Sparkling (Replace 'YOUR-12SPARK-SKU' with actual SKU)
  'YOUR-12SPARK-SKU': {
    name: '12 Pack - Sparkling Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6827,
    packSize: 12,
    type: 'sparkling'
  },

  // 6-Pack Regular (Replace 'YOUR-6PACK-SKU' with actual SKU)
  'YOUR-6PACK-SKU': {
    name: '6 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
    odooId: 6983,
    packSize: 6,
    type: 'regular'
  }
};

// Function to check if SKU is a Chilly Water product
function isChillyWaterProduct(sku) {
  return SKU_TO_PRODUCT.hasOwnProperty(sku);
}

// Function to get product info
function getProductInfo(sku) {
  return SKU_TO_PRODUCT[sku] || null;
}
```

## Alternative: Product Title Matching

If Amazon SKUs are inconsistent, you can match by product title keywords:

```javascript
function matchProductByTitle(title) {
  const titleLower = title.toLowerCase();

  // Check for sparkling first (more specific)
  if (titleLower.includes('sparkling') || titleLower.includes('spark')) {
    if (titleLower.includes('12')) {
      return {
        name: '12 Pack - Sparkling Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
        odooId: 6827,
        packSize: 12,
        type: 'sparkling'
      };
    }
  }

  // Check pack sizes for regular
  if (titleLower.includes('24')) {
    return {
      name: '24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
      odooId: 6794,
      packSize: 24,
      type: 'regular'
    };
  }

  if (titleLower.includes('6 pack') || titleLower.includes('6-pack')) {
    return {
      name: '6 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
      odooId: 6983,
      packSize: 6,
      type: 'regular'
    };
  }

  if (titleLower.includes('12')) {
    return {
      name: '12 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)',
      odooId: 6648,
      packSize: 12,
      type: 'regular'
    };
  }

  return null; // Not a recognized Chilly Water product
}
```

## Notes

- The screenshot shows these are the main SKUs to focus on
- All products include shipping in the name
- The existing workflow filters products by name, so exact name matching is critical
- Make sure to update this mapping with actual Amazon SKUs before deployment
