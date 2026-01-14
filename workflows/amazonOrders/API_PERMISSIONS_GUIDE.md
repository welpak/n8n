# Amazon SP-API Permissions for Customer Shipping Information (3PL)

## The Problem

You're seeing empty/missing:
- Customer name (`display_name: "Unknown"`)
- Street address (`address_line_1: ""`)

This is because **Amazon restricts access to Personally Identifiable Information (PII)** by default.

## Solution: Request Restricted Data Tokens (RDT)

### Step 1: Verify Your SP-API Application Has the Right Roles

1. Go to **Amazon Seller Central** → **Apps & Services** → **Develop Apps**
2. Find your SP-API application
3. Check that it has these **Data Access** permissions:
   - ✅ **Direct-to-Consumer Shipping** (required for shipping addresses)
   - ✅ **Order Information** (required for order details)
   - ✅ May also need: **Buyer Information**

### Step 2: Use Restricted Data Token (RDT) API

Amazon requires you to get a special token to access PII. You need to make an additional API call BEFORE getting the orders.

#### Get Restricted Data Token Endpoint:

```
POST https://sellingpartnerapi-na.amazon.com/tokens/2021-03-01/restrictedDataToken
```

**Request Body:**
```json
{
  "restrictedResources": [
    {
      "method": "GET",
      "path": "/orders/v0/orders",
      "dataElements": ["buyerInfo", "shippingAddress"]
    },
    {
      "method": "GET",
      "path": "/orders/v0/orders/{orderId}",
      "dataElements": ["buyerInfo", "shippingAddress"]
    }
  ]
}
```

**Response:**
```json
{
  "restrictedDataToken": "Atzr|...",
  "expiresIn": 3600
}
```

#### Updated Workflow Flow:

```
[1. Get OAuth Token]
       ↓
[2. Get Restricted Data Token] ← NEW STEP!
       ↓
[3. Get Orders] (use RDT in headers)
       ↓
[4. Split Orders]
       ↓
[5. Get Order Items]
       ↓
[6. Merge & Transform]
```

### Step 3: Update Your n8n Workflow

#### Add New Node After "Get OAuth Token":

**Node Name:** Get Restricted Data Token
**Type:** HTTP Request
**Method:** POST
**URL:** `https://sellingpartnerapi-na.amazon.com/tokens/2021-03-01/restrictedDataToken`

**Authentication:** AWS (Assume Role) - same as your other nodes

**Headers:**
```
x-amz-access-token: {{ $json.access_token }}
Content-Type: application/json
```

**Body (JSON):**
```json
{
  "restrictedResources": [
    {
      "method": "GET",
      "path": "/orders/v0/orders",
      "dataElements": ["buyerInfo", "shippingAddress"]
    }
  ]
}
```

#### Update "Get Orders" Node:

Change the headers to include BOTH tokens:

**Headers:**
```
x-amz-access-token: {{ $('Get Restricted Data Token').item.json.restrictedDataToken }}
```

**OR keep both:**
```
x-amz-access-token: {{ $('Get OAuth Token').first().json.access_token }}
x-amzn-restricted-data-token: {{ $('Get Restricted Data Token').first().json.restrictedDataToken }}
```

## Alternative: Check If You Have the Right API Application Type

### For 3PL Companies:

1. Your Amazon SP-API application should be registered as a **"Selling Partner"** or **"3PL Provider"** type
2. You need to be authorized by the seller to access their order data
3. The seller must grant you permission to view PII

### Verify Authorization:

- The selling partner must have authorized your app
- Your app must be in "Published" status (not just Draft)
- Check in Seller Central: **Apps & Services** → **Manage Your Apps** → verify your app is listed and authorized

## Quick Test

### Check What Data You're Currently Getting:

In your Transform node, add this at the beginning:

```javascript
const amazonOrder = $('Split Orders').item.json;
console.log('=== CHECKING AVAILABLE DATA ===');
console.log('ShippingAddress exists?', !!amazonOrder.ShippingAddress);
console.log('ShippingAddress keys:', Object.keys(amazonOrder.ShippingAddress || {}));
console.log('Name field:', amazonOrder.ShippingAddress?.Name);
console.log('AddressLine1:', amazonOrder.ShippingAddress?.AddressLine1);
console.log('BuyerInfo exists?', !!amazonOrder.BuyerInfo);
console.log('BuyerInfo keys:', Object.keys(amazonOrder.BuyerInfo || {}));
console.log('================================');
```

Run the workflow and check the console. If you see:
- ✅ **Keys include Name, AddressLine1** → Good! Just need to handle empty values
- ❌ **Keys missing or ShippingAddress doesn't exist** → Need RDT token
- ❌ **ShippingAddress exists but all values are empty** → Permission issue

## Common Issues & Solutions

### Issue 1: "Access Denied" when requesting RDT
**Solution:** Your SP-API app doesn't have the required roles. Go to Seller Central and add "Direct-to-Consumer Shipping" permission.

### Issue 2: RDT token works but still no address
**Solution:** The seller hasn't authorized your app for PII access. They need to re-authorize with additional permissions.

### Issue 3: Some fields populated, others empty
**Solution:** Amazon may be masking certain fields based on:
- Order age (older orders may have restricted data)
- Order type (Prime, Business, etc.)
- Buyer privacy settings

### Issue 4: "This operation requires a Restricted Data Token"
**Solution:** You're on the right track! Add the RDT request as shown above.

## For 3PL Companies Specifically

### Amazon's 3PL Program Requirements:

1. **Register as 3PL Provider** in Seller Central
2. **Get authorization from each seller** you fulfill for
3. **Request the right permissions:**
   - Fulfillment Outbound (for shipment creation)
   - Order Information (for order data)
   - Shipping Labels (if you print labels)

### Authorization Process:

1. Seller logs into their Seller Central
2. Goes to **Settings** → **User Permissions**
3. Invites your 3PL company as a user
4. Grants permissions including "View Customer Information"

## Testing Checklist

- [ ] SP-API app has "Direct-to-Consumer Shipping" role
- [ ] SP-API app has "Order Information" role
- [ ] Selling partner has authorized your app
- [ ] Added RDT token request node to workflow
- [ ] Updated Get Orders node to use RDT token
- [ ] Verified token in headers
- [ ] Tested with recent order (less than 30 days old)
- [ ] Checked console logs for actual data received

## Documentation Links

- [Amazon SP-API: Tokens API](https://developer-docs.amazon.com/sp-api/docs/tokens-api-use-case-guide)
- [Restricted Data Tokens Guide](https://developer-docs.amazon.com/sp-api/docs/tokens-api-use-case-guide)
- [Orders API v0 Reference](https://developer-docs.amazon.com/sp-api/docs/orders-api-v0-reference)
- [Data Protection Policy](https://developer-docs.amazon.com/sp-api/docs/data-protection-policy)

## Next Steps

1. **Today:** Add RDT token request to your workflow
2. **Verify:** Your SP-API app has the right roles/permissions
3. **Test:** Run workflow and check if name/address now appear
4. **If still failing:** Contact Amazon Developer Support to verify your 3PL authorization

## Updated Complete Workflow Needed?

Would you like me to create an updated `complete_workflow_with_items.json` that includes the Restricted Data Token request?
