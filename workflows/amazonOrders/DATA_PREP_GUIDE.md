# Amazon Orders to Odoo - Data Preparation Guide

## Overview
This guide explains how to transform Amazon SP-API order data into the format required by your existing Odoo injection workflow (Chilly Water orders).

## Current Workflow Requirements

Your existing Odoo workflow expects data in this structure:

```json
{
  "order": {
    "state": "OPEN",
    "reference_id": "order_reference",
    "id": "order_id",
    "fulfillments": [{
      "uid": "fulfillment_uid",
      "type": "SHIPMENT",
      "state": "PROPOSED",
      "shipment_details": {
        "recipient": {
          "display_name": "First Last",
          "email_address": "email@example.com",
          "phone_number": "+1234567890",
          "address": {
            "address_line_1": "123 Main St",
            "address_line_2": "Apt 4",
            "locality": "City",
            "administrative_district_level_1": "STATE",
            "postal_code": "12345",
            "country": "US",
            "first_name": "First",
            "last_name": "Last"
          }
        },
        "shipping_type": "FLAT",
        "placed_at": "2026-01-13T16:01:16.209Z"
      }
    }],
    "source": {
      "name": "Amazon"
    }
  },
  "line_items_structured": [
    {
      "name": "24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)",
      "quantity": "1"
    }
  ]
}
```

## Amazon SP-API Response Structure

The Amazon SP-API `/orders/v0/orders` endpoint returns:

```json
{
  "payload": {
    "Orders": [
      {
        "AmazonOrderId": "123-4567890-1234567",
        "PurchaseDate": "2026-01-13T10:30:00Z",
        "LastUpdateDate": "2026-01-13T10:35:00Z",
        "OrderStatus": "Unshipped",
        "FulfillmentChannel": "MFN",
        "SalesChannel": "Amazon.com",
        "ShipServiceLevel": "Std US D2D Dom",
        "OrderTotal": {
          "CurrencyCode": "USD",
          "Amount": "0.00"
        },
        "ShippingAddress": {
          "Name": "John Doe",
          "AddressLine1": "123 Main Street",
          "AddressLine2": "Apt 4",
          "City": "San Francisco",
          "StateOrRegion": "CA",
          "PostalCode": "94115",
          "CountryCode": "US"
        },
        "BuyerInfo": {
          "BuyerEmail": "buyer@example.com"
        }
      }
    ]
  }
}
```

**Note:** You'll need to make a separate API call to get order items:
`GET /orders/v0/orders/{orderId}/orderItems`

This returns:
```json
{
  "payload": {
    "OrderItems": [
      {
        "ASIN": "B0ABCDEFGH",
        "SellerSKU": "CW-24PACK-REG",
        "OrderItemId": "12345678901234",
        "Title": "24 Pack - Purified Chilly Water - 16oz Cans",
        "QuantityOrdered": 1,
        "QuantityShipped": 0
      }
    ]
  }
}
```

## Required Data Transformations

### 1. Order Basic Info
- `order.id` ← `AmazonOrderId`
- `order.reference_id` ← `AmazonOrderId`
- `order.state` ← Always "OPEN" (map from `OrderStatus`)
- `order.source.name` ← "Amazon"

### 2. Fulfillment Structure
- `fulfillments[0].uid` ← Generate unique ID or use `AmazonOrderId`
- `fulfillments[0].type` ← "SHIPMENT"
- `fulfillments[0].state` ← "PROPOSED"
- `fulfillments[0].shipment_details.placed_at` ← `PurchaseDate`
- `fulfillments[0].shipment_details.shipping_type` ← "FLAT"

### 3. Recipient Address Mapping

| Target Field | Amazon Field | Transformation |
|-------------|--------------|----------------|
| `recipient.display_name` | `ShippingAddress.Name` | Direct copy |
| `recipient.email_address` | `BuyerInfo.BuyerEmail` | May be masked by Amazon |
| `recipient.phone_number` | `ShippingAddress.Phone` | May not be available |
| `address.address_line_1` | `ShippingAddress.AddressLine1` | Direct copy |
| `address.address_line_2` | `ShippingAddress.AddressLine2` | Direct copy (may be empty) |
| `address.locality` | `ShippingAddress.City` | Direct copy |
| `address.administrative_district_level_1` | `ShippingAddress.StateOrRegion` | Direct copy |
| `address.postal_code` | `ShippingAddress.PostalCode` | Direct copy |
| `address.country` | `ShippingAddress.CountryCode` | Direct copy |
| `address.first_name` | `ShippingAddress.Name` | Split name (first word) |
| `address.last_name` | `ShippingAddress.Name` | Split name (rest) |

### 4. Line Items - SKU to Product Name Mapping

You need to map Amazon SKUs to your product names:

```javascript
const skuToProductName = {
  // 12-Pack Regular
  "CW-12PACK-REG": "12 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)",

  // 24-Pack Regular
  "CW-24PACK-REG": "24 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)",

  // 12-Pack Sparkling
  "CW-12PACK-SPARK": "12 Pack - Sparkling Chilly Water - 16oz Cans (SHIPPING INCLUDED)",

  // 6-Pack Regular
  "CW-6PACK-REG": "6 Pack - Purified Chilly Water - 16oz Cans (SHIPPING INCLUDED)"
};
```

Transform:
```json
{
  "line_items_structured": [
    {
      "name": skuToProductName[item.SellerSKU] || item.Title,
      "quantity": item.QuantityOrdered.toString()
    }
  ]
}
```

## Potential Issues & Solutions

### Issue 1: Missing Phone/Email
Amazon may mask or not provide buyer contact info.
**Solution:** Set default values or handle as optional in Odoo workflow.

### Issue 2: Multiple Order Items
Amazon orders can have multiple different products.
**Solution:** Aggregate all items into `line_items_structured` array.

### Issue 3: Name Splitting
Amazon provides full name as single field.
**Solution:** Use JavaScript to split:
```javascript
const nameParts = shippingAddress.Name.trim().split(' ');
const firstName = nameParts[0];
const lastName = nameParts.slice(1).join(' ') || firstName;
```

### Issue 4: Order Status Mapping
Amazon uses different status values (Unshipped, PartiallyShipped, Shipped, etc.)
**Solution:** Map to your system's status:
- "Unshipped" → "OPEN"
- "PartiallyShipped" → "OPEN"
- Others → Skip or handle separately

## Recommended Workflow Steps

1. **Get Orders** - Fetch unshipped orders from Amazon SP-API
2. **Loop Through Orders** - Process each order individually
3. **Get Order Items** - Fetch line items for each order
4. **Transform Data** - Apply all mappings above
5. **Filter by SKU** - Only process Chilly Water products
6. **Inject into Odoo** - Pass transformed data to existing workflow

## Next Steps

1. Create workflow to fetch Amazon orders
2. Create workflow to fetch order items
3. Create transformation/mapping workflow
4. Test with sample data
5. Connect to existing Odoo injection workflow
