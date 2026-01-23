# SEP: Enhanced Checkout Capabilities

**Status:** Draft  
**Version:** 2026-01-22  
**Authors:** Founding Maintainers

---

## 1. Overview

This SEP extends the Agentic Checkout Specification to support a broader range of commerce scenarios, including advanced fulfillment options, richer product metadata, subscription commerce, B2B transactions, promotional campaigns, and international commerce.

### 1.1 Goals

**Core Protocol Enhancements:**
- Add protocol versioning and capability discovery for API evolution and feature negotiation
- Improve API clarity with standardized field naming (`line_items`, required `currency`)
- Enhance session lifecycle management with expiration, escalation flows, and recovery URLs
- Enable hierarchical product structures (quantity at line level, parent-child relationships)
- Strengthen messaging system with severity levels, warnings, and expanded error codes
- Add fraud prevention signals for secure payment processing

**Commerce Capabilities:**
- Support omnichannel fulfillment (pickup, local delivery, split shipments) with rich metadata
- Enable promotional campaigns (coupon codes, automatic discounts, stackable promotions)
- Facilitate B2B transactions (purchase orders, net terms, approval workflows, tax exemptions)
- Support subscription and recurring billing models with trial periods
- Provide comprehensive product metadata (SKUs, variants, inventory status, physical attributes)
- Enable international commerce (multi-currency, localization, currency conversion)
- Enhance buyer experience (gift services, special instructions, flexible name handling, loyalty programs)

---

## 2. Line Item Enhancements

### 2.1 Core Structure Changes

**Item** schema (simplified):
```typescript
{
  "id": "item_123"  // Only contains item reference
}
```

**LineItem** schema with quantity and hierarchy:
```typescript
{
  "id": "line_item_123",
  "item": {
    "id": "item_123"
  },
  "quantity": 2,                        // Moved from Item to LineItem
  "parent_id": "line_item_100",         // For bundled/related products
  
  // Pricing
  "base_amount": 5000,
  "discount": 500,
  "subtotal": 4500,
  "tax": 360,
  "total": 4860,
  
  // Line-level totals breakdown
  "totals": [
    {
      "type": "subtotal",
      "display_text": "Line Subtotal",
      "amount": 4500
    },
    {
      "type": "tax",
      "display_text": "Line Tax",
      "amount": 360
    },
    {
      "type": "total",
      "display_text": "Line Total",
      "amount": 4860
    }
  ]
}
```

### 2.2 Product Identification & Metadata

**LineItem** schema extensions:

```typescript
{
  // ... core fields from 2.1 ...
  
  // Product identification
  "product_id": "prod_12345",           // Merchant's product identifier
  "sku": "DENIM-M-BLUE",                // Stock keeping unit
  "variant_id": "var_67890",            // Specific variant identifier
  
  // Product categorization
  "category": "Apparel > Outerwear",    // Product category path
  "tags": ["vintage", "sustainable"],   // Product tags
  
  // Physical attributes
  "weight": {
    "value": 500,                       // Weight in grams
    "unit": "g"
  },
  "dimensions": {
    "length": 30,                       // Dimensions in cm
    "width": 25,
    "height": 5,
    "unit": "cm"
  },
  
  // Availability & inventory
  "availability_status": "in_stock",    // in_stock | low_stock | out_of_stock | backorder | pre_order
  "available_quantity": 47,             // Current available quantity
  "max_quantity_per_order": 10,        // Maximum allowed per order
  "fulfillable_on": "2026-02-15T00:00:00Z",  // For pre-orders/backorders
  
  // Variant options
  "variant_options": [
    {
      "name": "Size",
      "value": "Medium"
    },
    {
      "name": "Color",
      "value": "Blue"
    }
  ],
  
  // Subscription information
  "subscription": {
    "is_subscription": true,
    "interval": "monthly",              // daily | weekly | monthly | yearly
    "interval_count": 1,                // Bill every N intervals
    "trial_period_days": 30,
    "billing_cycles": null              // null for unlimited, number for fixed
  }
}
```

### 2.3 Discount Metadata

Enhanced discount information in **LineItem**:

```typescript
{
  // ... existing fields ...
  
  "discount_details": [
    {
      "code": "SUMMER20",               // Promo code if applicable
      "type": "percentage",             // percentage | fixed | bogo | volume
      "amount": 60,                     // Amount of discount in minor units
      "description": "20% Summer Sale",
      "source": "coupon"                // coupon | automatic | loyalty
    }
  ]
}
```

---

## 3. Enhanced Fulfillment Options

### 3.1 Additional Fulfillment Types

Extend fulfillment options beyond `shipping` and `digital`:

**FulfillmentOptionPickup** for in-store and curbside pickup:

```typescript
{
  "type": "pickup",
  "id": "pickup_store_sf_001",
  "title": "In-Store Pickup",
  "subtitle": "Ready in 2 hours",
  "description": "Pick up your order at our San Francisco location",
  "location": {
    "name": "San Francisco Store",
    "address": {
      "line_one": "123 Market Street",
      "city": "San Francisco",
      "state": "CA",
      "country": "US",
      "postal_code": "94102"
    },
    "phone": "14155551234",
    "instructions": "Enter through main entrance, pickup counter on left"
  },
  "pickup_type": "in_store",          // in_store | curbside | locker
  "ready_by": "2026-01-22T16:00:00Z",
  "pickup_by": "2026-01-29T20:00:00Z"   // Latest pickup time
  // Free pickup - no subtotal/tax/total needed
}
```

**FulfillmentOptionLocalDelivery** for same-day and local delivery:

```typescript
{
  "type": "local_delivery",
  "id": "local_del_001",
  "title": "Same-Day Delivery",
  "subtitle": "Today by 8 PM",
  "description": "Fast local delivery within San Francisco",
  "delivery_window": {
    "start": "2026-01-22T18:00:00Z",
    "end": "2026-01-22T20:00:00Z"
  },
  "service_area": {
    "radius_miles": 10,
    "center_postal_code": "94102"
  },
  "subtotal": 999,
  "tax": 80,
  "total": 1079
}
```

### 3.2 Fulfillment Grouping

Support for complex multi-destination and split shipments:

```typescript
{
  "fulfillment_groups": [
    {
      "id": "group_1",
      "item_ids": ["item_123", "item_456"],
      "destination_type": "shipping",    // shipping | pickup | local_delivery | digital
      "fulfillment_details": {
        "name": "John Doe",
        "phone_number": "15551234567",
        "email": "john@example.com",
        "address": { /* address object */ }
      },
      "instructions": "Leave at front door"
    },
    {
      "id": "group_2",
      "item_ids": ["item_789"],
      "destination_type": "pickup",
      "location_id": "store_sf_001"
    }
  ]
}
```

---

## 4. Promotions & Discounts

### 4.1 Coupon Application

Support for promotional codes in checkout requests:

**CheckoutSessionCreateRequest** / **CheckoutSessionUpdateRequest**:

```typescript
{
  "items": [ /* ... */ ],
  "coupons": ["SUMMER20", "FREESHIP"],
  // ... other fields ...
}
```

### 4.2 Promotion Details in Response

**CheckoutSession** response includes applied promotions:

```typescript
{
  // ... existing fields ...
  
  "applied_coupons": [
    {
      "code": "SUMMER20",
      "status": "applied",              // applied | invalid | expired | limit_reached
      "discount_amount": 600,           // Total discount from this coupon
      "description": "20% off your order",
      "error": null                     // Error message if status != applied
    },
    {
      "code": "FREESHIP",
      "status": "applied",
      "discount_amount": 500,
      "description": "Free standard shipping"
    }
  ],
  
  "automatic_discounts": [
    {
      "id": "auto_discount_001",
      "name": "Buy 2, Get 1 Free",
      "discount_amount": 300,
      "description": "Automatically applied based on cart contents"
    }
  ],
  
  "available_promotions": [
    {
      "code": "VIP10",
      "description": "10% off for VIP members",
      "requirements": "Must be logged in as VIP member"
    }
  ]
}
```

---

## 5. Buyer Enhancements

### 5.1 Buyer Identity & Account Status

Extended **Buyer** schema:

```typescript
{
  // Name fields (flexible - can provide any combination)
  "first_name": "John",              // Optional
  "last_name": "Doe",                // Optional
  "full_name": "John Doe",           // Optional alternative to first/last
  "email": "john@example.com",       // Required
  "phone_number": "15551234567",
  
  // Account fields
  "customer_id": "cus_123456",        // Merchant's customer identifier
  "account_type": "guest",            // guest | registered | business
  "authentication_status": "authenticated",  // authenticated | guest | requires_signin
  
  // B2B fields (when account_type = business)
  "company": {
    "name": "Acme Corporation",
    "tax_id": "12-3456789",            // EIN, VAT number, etc.
    "department": "Engineering",
    "cost_center": "CC-001"
  },
  
  // Loyalty information
  "loyalty": {
    "tier": "gold",
    "points_balance": 1500,
    "member_since": "2024-06-15T00:00:00Z"
  },
  
  // Tax exemption
  "tax_exemption": {
    "certificate_id": "cert_12345",
    "certificate_type": "resale",      // resale | exempt_organization | government
    "exempt_regions": ["CA", "NY"],
    "expires_at": "2027-12-31T00:00:00Z"
  }
}
```

---

## 6. Tax Enhancements

### 6.1 Tax Exemptions

Support tax-exempt transactions:

```typescript
{
  // In LineItem
  "tax_exempt": true,
  "tax_exemption_reason": "resale_certificate",
  
  // In Buyer (for wholesale/B2B)
  "tax_exemption": {
    "certificate_id": "cert_12345",
    "certificate_type": "resale",      // resale | exempt_organization | government
    "exempt_regions": ["CA", "NY"],    // Regions where exemption applies
    "expires_at": "2027-12-31T00:00:00Z"
  }
}
```

### 7.2 Tax Breakdown

Detailed tax information in **Total**:

```typescript
{
  "type": "tax",
  "display_text": "Tax",
  "amount": 850,
  "breakdown": [
    {
      "jurisdiction": "California State",
      "rate": 0.0625,
      "amount": 625
    },
    {
      "jurisdiction": "San Francisco County",
      "rate": 0.0125,
      "amount": 125
    },
    {
      "jurisdiction": "San Francisco City",
      "rate": 0.01,
      "amount": 100
    }
  ]
}
```

---

## 7. Order Notes & Special Instructions

### 7.1 Gift Services

Add gift options to checkout:

```typescript
{
  // In CheckoutSessionCreateRequest / UpdateRequest
  "gift_options": {
    "is_gift": true,
    "gift_message": "Happy Birthday! Love, Mom",
    "gift_wrap": {
      "enabled": true,
      "style": "birthday",              // birthday | holiday | elegant
      "charge": 500                     // Additional charge for gift wrap
    },
    "hide_prices": true                 // Hide prices on packing slip
  }
}
```

### 8.2 Order Notes

General notes and delivery instructions:

```typescript
{
  // In CheckoutSessionCreateRequest / UpdateRequest
  "order_notes": "Please ring doorbell twice",
  "special_instructions": {
    "delivery": "Leave package at side entrance",
    "personalization": "Monogram: JD"
  }
}
```

---

## 8. Session Management & Protocol Versioning

### 8.1 Protocol Version & Capabilities

Add protocol versioning to **CheckoutSession** responses:

```typescript
{
  "id": "sess_abc123",
  "acp": {
    "version": "2026-01-22",           // Protocol version
    "capabilities": [
      "pickup_fulfillment",
      "split_payments", 
      "subscription_billing",
      "multi_currency"
    ]
  },
  // ... other fields ...
}
```

**Note:** This initial version provides basic capability signaling via a simple array of strings. A future version of this specification will introduce structured capability objects with detailed agent and seller capability declarations, including supported features, constraints, and negotiation parameters.

### 9.2 Session Expiry & Recovery

Add expiration and recovery to **CheckoutSession**:

```typescript
{
  // ... existing fields ...
  
  "created_at": "2026-01-22T10:00:00Z",
  "updated_at": "2026-01-22T10:15:00Z",
  "expires_at": "2026-01-22T16:00:00Z",  // Default: 6 hours from creation
  
  "continue_url": "https://merchant.example.com/checkout/recover/sess_abc123",
  
  "metadata": {
    "source": "mobile_app",
    "session_attributes": {
      "utm_source": "google",
      "utm_campaign": "summer_sale"
    }
  }
}
```

### 9.3 Enhanced Status Values

Expanded checkout session statuses:

```typescript
{
  "status": "incomplete" |              // Missing required information
            "not_ready_for_payment" |   // Not all requirements met
            "requires_escalation" |     // Needs merchant intervention
            "authentication_required" |  // 3DS or similar required
            "ready_for_payment" |       // Ready to complete
            "pending_approval" |        // Awaiting approval (B2B)
            "complete_in_progress" |    // Processing completion
            "completed" |               // Successfully completed
            "canceled" |                // Canceled
            "in_progress" |             // General processing state
            "expired"                   // Session expired
}
```

---

## 9. Multi-Currency & Localization

### 9.1 Currency Conversion

Support for displaying converted amounts:

```typescript
{
  // In CheckoutSession
  "currency": "usd",
  "presentment_currency": "eur",       // Currency shown to buyer
  "exchange_rate": 0.92,               // Exchange rate applied
  "exchange_rate_timestamp": "2026-01-22T10:00:00Z",
  
  "totals": [
    {
      "type": "total",
      "display_text": "Total",
      "amount": 10000,                  // Amount in settlement currency (USD)
      "presentment_amount": 9200        // Amount in presentment currency (EUR)
    }
  ]
}
```

### 9.2 Locale Preferences

Support localization:

```typescript
{
  // In CheckoutSessionCreateRequest
  "locale": "fr-CA",                   // IETF BCP 47 language tag
  "timezone": "America/Toronto"
}
```

---

## 10. Subscription & Recurring Billing

### 10.1 Subscription Line Items

Already covered in Section 2.1, but complete subscription response:

```typescript
{
  // In CheckoutSession for subscription products
  "subscription_details": {
    "id": "sub_12345",
    "status": "active",                 // trial | active | paused | canceled
    "current_period_start": "2026-01-22T00:00:00Z",
    "current_period_end": "2026-02-22T00:00:00Z",
    "trial_end": "2026-02-21T00:00:00Z",
    "cancel_at_period_end": false,
    "next_billing_date": "2026-02-22T00:00:00Z"
  }
}
```

---

## 11. B2B Features

### 11.1 Purchase Orders

Support PO-based payments:

```typescript
{
  // In PaymentData
  "type": "purchase_order",
  "purchase_order_number": "PO-2026-001234",
  "payment_terms": "net_30",           // immediate | net_15 | net_30 | net_60 | net_90
  "due_date": "2026-02-21T00:00:00Z",
  "approval_required": true
}
```

### 12.2 Approval Workflow

Checkout status for approval workflows:

```typescript
{
  "status": "pending_approval",         // New status value
  "approval_details": {
    "approver_email": "manager@acme.com",
    "approval_threshold": 100000,       // Amount requiring approval
    "requested_at": "2026-01-22T10:00:00Z",
    "approval_url": "https://merchant.example.com/approvals/appr_123"
  }
}
```

### 12.3 Quote to Order

Support quote conversion:

```typescript
{
  // In CheckoutSessionCreateRequest
  "quote_id": "quote_12345",
  "quote_expires_at": "2026-02-22T00:00:00Z"
}
```

---

## 12. Enhanced Order Confirmation

### 12.1 Extended Order Object

Enhanced **Order** in completion response:

```typescript
{
  "id": "ord_abc123",
  "checkout_session_id": "sess_xyz789",
  "order_number": "10001234",          // Human-readable order number
  "permalink_url": "https://merchant.example.com/orders/ord_abc123",
  
  "status": "confirmed",                // confirmed | processing | shipped | delivered
  
  "estimated_delivery": {
    "earliest": "2026-01-25T00:00:00Z",
    "latest": "2026-01-27T00:00:00Z"
  },
  
  "confirmation": {
    "confirmation_number": "CONF-ABC123",
    "confirmation_email_sent": true,
    "receipt_url": "https://merchant.example.com/receipts/ord_abc123.pdf",
    "invoice_number": "INV-2026-001234"
  },
  
  "support": {
    "email": "support@merchant.example.com",
    "phone": "18005551234",
    "hours": "Monday-Friday, 9am-5pm PST",
    "help_center_url": "https://help.merchant.example.com"
  }
}
```

---

## 13. Enhanced Messaging System

### 13.1 Message Types with Severity

Support for info, warning, and error messages with severity levels:

```typescript
{
  "messages": [
    {
      "type": "info",
      "severity": "low",
      "content_type": "plain",
      "content": "This item ships from our partner warehouse"
    },
    {
      "type": "warning",
      "code": "low_stock",
      "severity": "medium",
      "param": "$.line_items[0]",
      "content_type": "plain",
      "content": "Only 3 items left in stock!"
    },
    {
      "type": "error",
      "code": "quantity_exceeded",
      "severity": "high",
      "param": "$.line_items[1].quantity",
      "content_type": "plain",
      "content": "Maximum quantity per order is 10"
    }
  ]
}
```

### 14.2 Warning Codes

New warning type with specific codes:

```typescript
{
  "type": "warning",
  "code": "low_stock" | 
          "high_demand" | 
          "shipping_delay" | 
          "price_change" | 
          "expiring_promotion" | 
          "limited_availability",
  "severity": "info" | "low" | "medium" | "high" | "critical"
}
```

### 14.3 Expanded Error Codes

Extended error codes:

```typescript
{
  "type": "error",
  "code": "missing" | "invalid" | "out_of_stock" | 
          "payment_declined" | "requires_sign_in" | "requires_3ds" | 
          "low_stock" | "quantity_exceeded" | "coupon_invalid" | 
          "coupon_expired" | "minimum_not_met" | "maximum_exceeded" | 
          "region_restricted" | "age_verification_required" | 
          "approval_required" | "unsupported" | "not_found" | 
          "conflict" | "rate_limited" | "expired",
  "severity": "info" | "low" | "medium" | "high" | "critical"
}
```

---

## 14. Risk Signals & Fraud Prevention

### 14.1 Risk Signals in Complete Request

Add fraud prevention signals to checkout completion:

```typescript
{
  // In CheckoutSessionCompleteRequest
  "risk_signals": {
    "ip_address": "192.0.2.1",
    "user_agent": "Mozilla/5.0...",
    "accept_language": "en-US,en;q=0.9",
    "session_id": "sess_browser_abc123",
    "device_fingerprint": "fp_device_xyz789"
  },
  "payment_data": { /* ... */ },
  "buyer": { /* ... */ }
}
```

---

## 15. Enhanced Links

### 15.1 Expanded Link Types with Titles

Support for additional link types and optional titles:

```typescript
{
  "links": [
    {
      "type": "terms_of_use",
      "title": "Terms of Service",
      "url": "https://merchant.example.com/terms"
    },
    {
      "type": "privacy_policy",
      "title": "Privacy Policy",  
      "url": "https://merchant.example.com/privacy"
    },
    {
      "type": "return_policy",
      "title": "Returns & Refunds",
      "url": "https://merchant.example.com/returns"
    },
    {
      "type": "shipping_policy",
      "title": "Shipping Information",
      "url": "https://merchant.example.com/shipping"
    },
    {
      "type": "contact_us",
      "title": "Contact Support",
      "url": "https://merchant.example.com/contact"
    },
    {
      "type": "support",
      "title": "Help Center",
      "url": "https://help.merchant.example.com"
    }
  ]
}
```

---

## 16. Request/Response Field Changes

### 16.1 Renamed Fields

Key field naming updates for clarity:

- **Request:** `items` → `line_items` (in create/update requests)
- **Request:** Added required `currency` field to create request
- **Item:** Removed `quantity` (moved to LineItem)
- **LineItem:** Added `quantity`, `parent_id`, `totals`
- **Buyer:** Made `first_name` and `last_name` optional, added `full_name`
- **All responses:** Added `acp` object with version and capabilities
- **All responses:** Added `expires_at`, `continue_url`, `payment` object

---

## 17. Complete Schema Updates

### 17.1 Updated Status Values

Extend checkout session status enum:

```typescript
"status": "not_ready_for_payment" | "authentication_required" | 
          "ready_for_payment" | "pending_approval" | 
          "completed" | "canceled" | "in_progress" | "expired"
```

### 17.2 Updated Total Types

Additional total types:

```typescript
"type": "items_base_amount" | "items_discount" | "subtotal" | "discount" | 
        "fulfillment" | "tax" | "fee" | "gift_wrap" | "tip" | 
        "store_credit" | "total"
```

---

## 18. Implementation Examples

### 18.1 Subscription Checkout with Discount

**Request:**
```json
{
  "line_items": [
    {
      "id": "sub_monthly_premium"
    }
  ],
  "currency": "usd",
  "coupons": ["FIRST30"],
  "buyer": {
    "email": "jane@example.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "customer_id": "cus_456789"
  }
}
```

**Response:**
```json
{
  "id": "sess_sub_001",
  "acp": {
    "version": "2026-01-22",
    "capabilities": ["subscription_billing", "coupons"]
  },
  "status": "ready_for_payment",
  "currency": "usd",
  "expires_at": "2026-01-22T16:00:00Z",
  "line_items": [
    {
      "id": "line_sub_001",
      "item": { "id": "sub_monthly_premium" },
      "quantity": 1,
      "product_id": "prod_premium",
      "name": "Premium Monthly Subscription",
      "base_amount": 2999,
      "discount": 900,
      "subtotal": 2099,
      "tax": 168,
      "total": 2267,
      "subscription": {
        "is_subscription": true,
        "interval": "monthly",
        "interval_count": 1,
        "trial_period_days": 0
      },
      "discount_details": [
        {
          "code": "FIRST30",
          "type": "percentage",
          "amount": 900,
          "description": "30% off first month",
          "source": "coupon"
        }
      ]
    }
  ],
  "applied_coupons": [
    {
      "code": "FIRST30",
      "status": "applied",
      "discount_amount": 900,
      "description": "30% off your first month"
    }
  ],
  "totals": [
    { "type": "subtotal", "display_text": "Subtotal", "amount": 2999 },
    { "type": "discount", "display_text": "Discount (FIRST30)", "amount": -900 },
    { "type": "tax", "display_text": "Tax", "amount": 168 },
    { "type": "total", "display_text": "Total Today", "amount": 2267 }
  ],
  "fulfillment_options": [
    {
      "type": "digital",
      "id": "digital_immediate",
      "title": "Instant Access",
      "subtitle": "Access immediately after purchase",
      "description": "Get instant access to all premium features",
      "total": 0
    }
  ],
  "messages": [],
  "links": []
}
```

### 18.2 B2B Checkout with Purchase Order

**Request:**
```json
{
  "line_items": [
    { "id": "item_bulk_001" }
  ],
  "currency": "usd",
  "buyer": {
    "email": "robert@acme.com",
    "first_name": "Robert",
    "last_name": "Johnson",
    "account_type": "business",
    "company": {
      "name": "Acme Corporation",
      "tax_id": "12-3456789",
      "department": "Operations"
    },
    "tax_exemption": {
      "certificate_id": "cert_acme_001",
      "certificate_type": "resale",
      "exempt_regions": ["CA", "TX"]
    }
  },
  "fulfillment_details": {
    "name": "Acme Warehouse",
    "address": {
      "name": "Acme Warehouse",
      "line_one": "500 Industrial Blvd",
      "city": "San Jose",
      "state": "CA",
      "country": "US",
      "postal_code": "95113"
    }
  }
}
```

**Complete Request with PO:**
```json
{
  "buyer": {
    "email": "robert@acme.com",
    "first_name": "Robert",
    "last_name": "Johnson"
  },
  "payment_data": {
    "type": "purchase_order",
    "purchase_order_number": "PO-2026-001234",
    "payment_terms": "net_30",
    "due_date": "2026-02-21T00:00:00Z"
  },
  "risk_signals": {
    "ip_address": "203.0.113.42",
    "user_agent": "Mozilla/5.0...",
    "session_id": "sess_b2b_123"
  }
}
```

### 18.3 Omnichannel Checkout with Split Fulfillment

**Request:**
```json
{
  "line_items": [
    { "id": "item_001" },
    { "id": "item_002" },
    { "id": "item_003" }
  ],
  "currency": "usd",
  "fulfillment_groups": [
    {
      "id": "group_ship",
      "item_ids": ["item_001"],
      "destination_type": "shipping",
      "fulfillment_details": {
        "name": "Jane Doe",
        "phone_number": "15551234567",
        "email": "jane@example.com",
        "address": {
          "name": "Jane Doe",
          "line_one": "123 Main St",
          "city": "Portland",
          "state": "OR",
          "country": "US",
          "postal_code": "97201"
        }
      }
    },
    {
      "id": "group_pickup",
      "item_ids": ["item_002", "item_003"],
      "destination_type": "pickup",
      "location_id": "store_pdx_001"
    }
  ],
  "gift_options": {
    "is_gift": true,
    "gift_message": "Congrats on your new home!",
    "gift_wrap": {
      "enabled": true,
      "style": "elegant"
    }
  }
}
```

### 18.4 International Checkout with Currency Conversion

**Request:**
```json
{
  "line_items": [
    { "id": "item_intl_001" }
  ],
  "currency": "usd",
  "locale": "fr-FR",
  "buyer": {
    "email": "marie@example.fr",
    "full_name": "Marie Dubois"
  },
  "fulfillment_details": {
    "name": "Marie Dubois",
    "address": {
      "name": "Marie Dubois",
      "line_one": "15 Rue de la Paix",
      "city": "Paris",
      "state": "Île-de-France",
      "country": "FR",
      "postal_code": "75002"
    }
  }
}
```

**Response (showing currency conversion):**
```json
{
  "id": "sess_intl_001",
  "acp": {
    "version": "2026-01-22",
    "capabilities": ["multi_currency", "international_shipping"]
  },
  "currency": "usd",
  "presentment_currency": "eur",
  "exchange_rate": 0.92,
  "exchange_rate_timestamp": "2026-01-22T10:00:00Z",
  "expires_at": "2026-01-22T16:00:00Z",
  "line_items": [
    {
      "id": "line_intl_001",
      "item": { "id": "item_intl_001" },
      "quantity": 1,
      "base_amount": 5000,
      "tax": 1000,
      "total": 6000
    }
  ],
  "totals": [
    {
      "type": "subtotal",
      "display_text": "Sous-total",
      "amount": 5000,
      "presentment_amount": 4600
    },
    {
      "type": "tax",
      "display_text": "TVA",
      "amount": 1000,
      "presentment_amount": 920,
      "breakdown": [
        {
          "jurisdiction": "France VAT",
          "rate": 0.20,
          "amount": 1000
        }
      ]
    },
    {
      "type": "total",
      "display_text": "Total",
      "amount": 6000,
      "presentment_amount": 5520
    }
  ]
}
```

---

## 19. Validation Rules

### 19.1 Request Field Requirements

**Create Checkout:**
- `line_items` array MUST contain at least one item
- `currency` field is REQUIRED
- Each item in `line_items` MUST have valid `id`

**Update Checkout:**
- At least one field MUST be provided for update
- When updating `line_items`, entire array replaces previous state

**Complete Checkout:**
- `payment_data` is REQUIRED
- When session status is `authentication_required`, `authentication_result` MUST be provided

### 19.2 Line Items

- `quantity` MUST be >= 1
- `product_id`, `sku`, `variant_id` SHOULD be provided when available
- `available_quantity` SHOULD be updated in real-time
- When `availability_status` is `pre_order` or `backorder`, `fulfillable_on` MUST be provided
- When `subscription.is_subscription` is true, `subscription.interval` and `subscription.interval_count` MUST be provided
- When `parent_id` is set, referenced parent line item MUST exist in same checkout

### 19.3 Fulfillment

- When `fulfillment_groups` is provided, all line items MUST be assigned to exactly one group
- Pickup fulfillment MUST include valid `location` with address
- Local delivery MUST include `delivery_window` or estimated delivery time
- All fulfillment options MUST include `description` field for clarity

### 19.4 Coupons

- Servers MUST validate all codes in `coupons` array
- Applied coupons MUST appear in `applied_coupons` with status
- Invalid/expired coupons MUST have status != "applied" with error message

### 19.5 Payment

- For B2B purchase orders, `account_type` MUST be "business"
- For tax-exempt transactions, `tax_exemption.certificate_id` MUST be valid

### 19.6 Currency

- When `presentment_currency` differs from `currency`, all totals MUST include `presentment_amount`
- `exchange_rate` MUST be current (typically < 1 hour old)

### 19.7 Buyer Information

- `email` is REQUIRED
- At least one of `first_name`/`last_name` OR `full_name` SHOULD be provided
- When `account_type` is "business", `company.name` is REQUIRED

### 20.8 Messages

- All messages MUST include `type` and `severity`
- `warning` type MUST include `code`
- `error` type MUST include `code`
- `param` SHOULD use RFC 9535 JSONPath notation when referencing specific fields

### 20.9 Protocol Versioning

- All responses SHOULD include `acp` object with `version`
- `acp.capabilities` array SHOULD list supported optional features
- Clients SHOULD respect `expires_at` and handle session expiration
- When status is `requires_escalation`, `continue_url` MUST be provided

---

## 20. Migration & Backward Compatibility

### 20.1 Breaking Changes

This SEP includes some breaking changes:

**Field Renames:**
- `items` → `line_items` in request bodies
- `currency` now REQUIRED in create request

**Structure Changes:**
- `Item.quantity` moved to `LineItem.quantity`
- `Buyer.first_name` and `Buyer.last_name` now optional (only `email` required)

**Required Field Changes:**
- LineItem: changed from `[id, item, base_amount, discount, subtotal, tax, total]` to `[id, item, quantity, total]`

### 20.2 Additive Changes

All other changes are additive and optional:
- New fulfillment types
- New status values
- New message types
- New fields on all objects

### 20.3 Migration Strategy

Merchants implementing this SEP should:
1. Update request construction to use `line_items` instead of `items`
2. Include `currency` in all create requests
3. Move `quantity` from Item to LineItem level
4. Handle new status values gracefully
5. Optionally adopt new features (coupons, gift options, etc.)

---

## 21. OpenAPI Schema Changes

All schemas have been updated in `spec/openapi/openapi.agentic_checkout.yaml` and `spec/json-schema/schema.agentic_checkout.json` to reflect these additions.

---

## 22. Summary

This SEP significantly expands the capabilities of the Agentic Checkout Specification to support:

✅ Protocol versioning and capability discovery  
✅ Enhanced messaging with severity levels and warnings  
✅ Risk signals for fraud prevention  
✅ Hierarchical line item structures with quantity at line level  
✅ Modern omnichannel fulfillment (pickup, local delivery)  
✅ Rich product metadata and variants  
✅ Promotional campaigns with coupons  
✅ B2B commerce (PO, net terms, approvals)  
✅ Subscription and recurring billing  
✅ Gift services and special instructions  
✅ International commerce with multi-currency  
✅ Advanced tax scenarios  
✅ Enhanced session management with expiration and recovery  
✅ Real-time inventory awareness  
✅ Expanded link types for merchant communication  
✅ Flexible buyer name handling

These enhancements enable ACP to support a comprehensive range of commerce scenarios with improved API clarity and extensibility.

