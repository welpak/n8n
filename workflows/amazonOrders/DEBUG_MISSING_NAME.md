# Debugging: Missing Ship-To Name from Amazon

## Quick Debug Code

Add this at the start of your Transform node to see what Amazon is actually sending:

```javascript
// Add this RIGHT at the beginning of your Transform code
console.log('===== DEBUGGING SHIP-TO NAME =====');
console.log('Full Amazon Order:', JSON.stringify($('Split Orders').item.json, null, 2));

const amazonOrder = $('Split Orders').item.json;
const shippingAddress = amazonOrder.ShippingAddress || {};

console.log('ShippingAddress object:', JSON.stringify(shippingAddress, null, 2));
console.log('Name field:', shippingAddress.Name);
console.log('Name exists?', !!shippingAddress.Name);
console.log('Name length:', shippingAddress.Name?.length);
console.log('==================================');
```

Then check your browser console (F12) when running the workflow.

## Common Causes & Solutions

### Cause 1: Amazon Uses Different Field Name

Amazon might use `Name` at the order level instead of in ShippingAddress.

**Check these locations:**
```javascript
// Try these in order:
const shipToName =
  amazonOrder.ShippingAddress?.Name ||           // Standard location
  amazonOrder.BuyerInfo?.BuyerName ||            // Buyer info
  amazonOrder.Name ||                            // Order level
  amazonOrder.ShippingAddress?.AddressLine1 ||   // Fallback to address
  'Unknown Customer';                            // Last resort

console.log('Ship-to name found:', shipToName);
```

### Cause 2: Amazon Business Orders

Business orders sometimes have different fields:
```javascript
const shipToName =
  amazonOrder.ShippingAddress?.Name ||
  amazonOrder.ShippingAddress?.CompanyName ||
  amazonOrder.DefaultShipFromLocationAddress?.Name ||
  'Unknown Customer';
```

### Cause 3: Empty/Whitespace

Name might be an empty string or just whitespace:
```javascript
const rawName = amazonOrder.ShippingAddress?.Name || '';
const cleanName = rawName.trim();

if (!cleanName) {
  console.warn('⚠️ Ship-to name is empty!');
  // Use fallback
}

const shipToName = cleanName || 'Unknown Customer';
```

### Cause 4: International Orders

Some international orders have name in different encoding:
```javascript
// Check for special characters
const name = amazonOrder.ShippingAddress?.Name || '';
console.log('Name charCodes:', Array.from(name).map(c => c.charCodeAt(0)));
```

## Updated Transform Code Section

Replace the name extraction part of your transform with this more robust version:

```javascript
// Better name extraction with fallbacks
function getShipToName(amazonOrder) {
  const shippingAddress = amazonOrder.ShippingAddress || {};
  const buyerInfo = amazonOrder.BuyerInfo || {};

  // Try multiple locations
  const possibleNames = [
    shippingAddress.Name,
    buyerInfo.BuyerName,
    amazonOrder.Name,
    shippingAddress.CompanyName,
    shippingAddress.AddressLine1  // Last resort
  ];

  // Find first non-empty value
  for (const name of possibleNames) {
    if (name && typeof name === 'string' && name.trim().length > 0) {
      console.log(`✓ Found name in: ${possibleNames.indexOf(name)} - "${name}"`);
      return name.trim();
    }
  }

  console.warn('⚠️ No ship-to name found in any location');
  return 'Amazon Customer';  // Default
}

const shipToName = getShipToName(amazonOrder);
const nameParts = splitName(shipToName);

console.log('Using ship-to name:', shipToName);
console.log('First name:', nameParts.firstName);
console.log('Last name:', nameParts.lastName);
```

## Test Scenarios

Run these checks:

1. **Log the raw API response** from "Get Orders" node
2. **Check if Name is in the response at all**
3. **Try with different order types** (regular, business, international)
4. **Verify Amazon API permissions** - some fields require specific permissions

## Quick Fix for Now

If you need a temporary workaround while debugging:

```javascript
// Temporary fallback using address as name if needed
const shipToName = amazonOrder.ShippingAddress?.Name ||
                   `Customer at ${amazonOrder.ShippingAddress?.City || 'Unknown'}`;
```

Or use a generic name:
```javascript
const shipToName = amazonOrder.ShippingAddress?.Name || 'Amazon Customer';
```

## What to Share

To help debug further, share:
1. The output from "Get Orders" node (with name redacted if needed)
2. What you see in the console logs
3. The order type (FBA vs MFN, domestic vs international)

Would you like me to create an updated transform code with all these improvements?
