# Payment Handlers Context - Full Agent Reference

**Purpose:** Complete context dump for AI agent working on ACP payment handlers. Contains all details from UCP exploration and handler analysis.

**Date:** 2026-01-22

---

## Overview

This document contains comprehensive context on UCP payment handlers to inform ACP payment handler design. It includes:
1. UCP handler framework structure
2. Three complete handler examples (Stripe SPT, Shop Pay, Google Pay)
3. Schema patterns and relationships
4. Security models
5. Integration flows

---

## UCP Repository Structure

**Location:** `/Users/prasadw/code/ucp`

### Key Files Explored

```
/Users/prasadw/code/ucp/
├── spec/
│   ├── schemas/
│   │   ├── capability.json                           # Capability negotiation schema
│   │   ├── ucp.json                                  # Base UCP definitions
│   │   └── shopping/
│   │       ├── types/
│   │       │   ├── payment_handler_resp.json         # Handler response schema
│   │       │   ├── payment_handler.create_req.json   # Handler create request
│   │       │   ├── payment_instrument_base.json      # Base instrument schema
│   │       │   ├── payment_instrument.json           # Instrument discriminator
│   │       │   ├── card_payment_instrument.json      # Card-specific instrument
│   │       │   ├── payment_credential.json           # Credential discriminator
│   │       │   ├── token_credential_resp.json        # Token credential schema
│   │       │   └── card_credential.json              # Card credential schema
│   └── handlers/
│       └── tokenization/
│           └── openapi.json                          # Tokenization API spec
└── docs/
    └── specification/
        ├── payment-handler-guide.md                  # Handler specification guide
        ├── payment-handler-template.md               # Template for new handlers
        └── examples/
            ├── platform-tokenizer-payment-handler.md
            ├── business-tokenizer-payment-handler.md
            └── encrypted-credential-handler.md
```

---

## UCP Handler Framework (Core Concepts)

### Handler Declaration Structure

Every payment handler in UCP follows this standard structure when advertised by businesses:

```json
{
  "payment": {
    "handlers": [
      {
        "id": "unique_instance_id",
        "name": "reverse.dns.format",
        "version": "YYYY-MM-DD",
        "spec": "https://example.com/handler-spec",
        "config_schema": "https://example.com/config.json",
        "instrument_schemas": ["https://example.com/instrument.json"],
        "config": {
          // Handler-specific configuration
        }
      }
    ]
  }
}
```

**Field Definitions:**

- **`id`** (string, required): Unique identifier for this handler instance within payment.handlers. Used by payment instruments to reference which handler produced them.
- **`name`** (string, required): Specification name using reverse-DNS format (e.g., `dev.ucp.delegate_payment`, `com.google.pay`)
- **`version`** (string, required): Handler version in YYYY-MM-DD format
- **`spec`** (URI, required): URL to human-readable specification document
- **`config_schema`** (URI, required): URL to JSON Schema for validating the config object
- **`instrument_schemas`** (array of URIs, required): URLs to schemas defining valid instrument structures
- **`config`** (object, required): Handler-specific configuration (structure validated by config_schema)

**Schema Source:** `/Users/prasadw/code/ucp/spec/schemas/shopping/types/payment_handler_resp.json`

---

### Base Payment Instrument Schema

All payment instruments extend this base schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/schemas/shopping/types/payment_instrument_base.json",
  "title": "Payment Instrument Base",
  "type": "object",
  "required": ["id", "handler_id", "type"],
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique identifier for this instrument instance, assigned by the Agent."
    },
    "handler_id": {
      "type": "string",
      "description": "The unique identifier for the handler instance that produced this instrument."
    },
    "type": {
      "type": "string",
      "description": "The broad category of the instrument (e.g., 'card', 'tokenized_card')."
    },
    "billing_address": {
      "$ref": "postal_address.json"
    },
    "credential": {
      "$ref": "payment_credential.json"
    }
  }
}
```

**Key Pattern:** The `handler_id` field links the instrument back to which handler produced it.

---

### Base Credential Schemas

**Token Credential:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/schemas/shopping/types/token_credential.json",
  "title": "Token Credential Response",
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": {
      "type": "string",
      "description": "The specific type of token (e.g., 'stripe_token')."
    }
  },
  "additionalProperties": true
}
```

**Payment Credential (Discriminator):**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/schemas/shopping/types/payment_credential.json",
  "title": "Payment Credential",
  "description": "Container for sensitive payment data.",
  "oneOf": [
    {"$ref": "token_credential_resp.json"},
    {"$ref": "card_credential.json"}
  ]
}
```

---

### Capability Schema

UCP uses a capability negotiation system with this structure:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/schemas/capability.json",
  "title": "UCP Capability",
  "description": "Schema for UCP capabilities and extensions.",
  "$defs": {
    "base": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "pattern": "^[a-z][a-z0-9]*(?:\\.[a-z][a-z0-9_]*)+$",
          "description": "Stable capability identifier in reverse-domain notation."
        },
        "version": {
          "$ref": "ucp.json#/$defs/version",
          "description": "Capability version in YYYY-MM-DD format."
        },
        "spec": {
          "type": "string",
          "format": "uri",
          "description": "URL to human-readable specification document."
        },
        "schema": {
          "type": "string",
          "format": "uri",
          "description": "URL to JSON Schema for this capability's payload."
        },
        "extends": {
          "type": "string",
          "pattern": "^[a-z][a-z0-9]*(?:\\.[a-z][a-z0-9_]*)+$",
          "description": "Parent capability this extends."
        },
        "config": {
          "type": "object",
          "description": "Capability-specific configuration."
        }
      }
    }
  }
}
```

---

## UCP Payment Handler Framework (Specification Guide)

**Source:** `/Users/prasadw/code/ucp/docs/specification/payment-handler-guide.md`

### Core Elements Every Handler Must Define

#### 1. Participants

The distinct actors that participate in the handler's lifecycle:

| Participant | Role |
|:------------|:-----|
| **Business** | Advertises handler configuration, processes payment instruments |
| **Platform** | Discovers handlers, acquires payment instruments, submits checkout |
| **Extended Participants** | Handler-specific (e.g., Tokenizer, PSP) |

**Note:** While the guide refers to "Business," technical schema fields use `merchant_*` nomenclature.

#### 2. Prerequisites

**Definition:** The onboarding, setup, or configuration a participant must complete before participating.

**Signature:**
```
PREREQUISITES(participant, onboarding_input) → prerequisites_output
```

**Prerequisites Output:**
- At minimum: an **identity** (maps to `PaymentIdentity`)
- May include additional configuration, credentials, settings
- Identity typically appears in handler's `config` object (e.g., as `merchant_id`)

**Key Point:** Participants receiving raw credentials must complete security acknowledgements during onboarding.

#### 3. Handler Declaration

**Definition:** The configuration a business advertises to indicate support for this handler.

**Signature:**
```
HANDLER_DECLARATION(prerequisites_output) → handler_declaration
```

**Output Structure:** Conforms to `PaymentHandler` schema with:
- Handler metadata (id, name, version, spec)
- `config_schema`: Points to JSON Schema for config validation
- `instrument_schemas`: Array of accepted instrument schemas
- `config`: Handler-specific configuration from prerequisites

**Best Practice:** Include `environment` field (sandbox/production) in config schema to standardize testing flows.

#### 4. Instrument Acquisition

**Definition:** The protocol a platform follows to acquire a payment instrument.

**Signature:**
```
INSTRUMENT_ACQUISITION(
  platform_prerequisites_output,
  handler_declaration,
  binding,
  buyer_input
) → checkout_instrument
```

**Key Points:**
- Specs should document how to apply handler's `config` to construct valid `checkout_instrument`
- Must explain how to create effective credential binding for security
- Binding ties credential to specific checkout and business

#### 5. Processing

**Definition:** Steps to process a received payment instrument.

**Signature:**
```
PROCESSING(
  identity,
  checkout_instrument,
  binding,
  transaction_context
) → processing_result
```

**Required:** Specs MUST define error mapping for common failures (declined, insufficient funds, network error) to standard UCP errors.

#### 6. Binding

**Definition:** A cryptographic or logical association of a payment instrument to a specific checkout transaction and business identity. Prevents replay attacks.

---

## Complete Handler Examples

### 1. Stripe Shared Payment Token (SPT) Handler

**Handler Name:** `com.stripe.shared_payment_token`

**Schemas Provided:**

#### Config Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://ucp.dev/schemas/handlers/stripe_shared_payment_token/config.json",
  "title": "StripeSharedPaymentTokenHandlerConfig",
  "description": "Configuration schema for the com.stripe.shared_payment_token handler.",
  "type": "object",
  "required": ["stripe_network_id"],
  "properties": {
    "stripe_network_id": {
      "type": "string",
      "description": "The Stripe network identifier of the merchant who will process payments.",
      "examples": ["rocket_rides"]
    },
    "supported_payment_methods": {
      "type": "array",
      "description": "Optional list of Stripe payment method types the merchant accepts.",
      "items": {
        "type": "string"
      }
    }
  },
  "additionalProperties": false
}
```

#### Credential Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stripe.com/ucp/handlers/stripe_shared_payment_token/credential.json",
  "title": "StripeSharedPaymentTokenCredential",
  "description": "Credential structure for Stripe Shared Payment Tokens (SPT).",
  "allOf": [
    {
      "$ref": "https://ucp.dev/schemas/shopping/types/token_credential.json"
    },
    {
      "type": "object",
      "properties": {
        "type": {
          "const": "stripe_shared_payment_token",
          "description": "The credential type for Stripe Shared Payment Tokens."
        }
      }
    }
  ]
}
```

#### Instrument Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://stripe.com/ucp/handlers/stripe_shared_payment_token/instrument.json",
  "title": "StripeSharedPaymentTokenInstrument",
  "description": "Payment instrument structure for Stripe Shared Payment Token.",
  "allOf": [
    { "$ref": "https://ucp.dev/schemas/shopping/types/payment_instrument.json" },
    {
      "type": "object",
      "properties": {
        "type": {
          "const": "stripe_shared_payment_token",
          "description": "The payment instrument type identifier."
        },
        "credential": {
          "$ref": "https://ucp.dev/schemas/stripe_shared_payment_token/credential.json"
        }
      }
    }
  ]
}
```

**Key Characteristics:**
- Delegated token flow: Agent obtains SPT from Stripe on merchant's behalf
- Merchant identified by `stripe_network_id`
- Token bound to specific checkout and merchant
- Agent must have relationship with Stripe to obtain SPTs

---

### 2. Shop Pay Handler

**Handler Name:** `dev.shopify.shop_pay`
**Version:** `2026-01-11`

**Full Specification Summary:**

#### Introduction
- Enables Shop Pay as accelerated checkout option
- Delegated payment flow: agents generate Shop Token, merchants process
- Single integration works across all UCP-compatible merchants

#### Merchant Integration

**Requirements:**
- Shopify merchants: Automatic (Shopify advertises handlers)
- External merchants: Register for Shop Pay to obtain `shop_id`

**Handler Configuration:**

```json
{
  "payment": {
    "handlers": [
      {
        "id": "shop_pay",
        "name": "dev.shopify.shop_pay",
        "version": "2026-01-11",
        "spec": "https://shopify.dev/docs/agents/checkout/shop-pay-handler",
        "config_schema": "https://shopify.dev/ucp/shop-pay-handler/2026-01-11/config.json",
        "instrument_schemas": [
          "https://shopify.dev/ucp/shop-pay-handler/2026-01-11/instrument.json"
        ],
        "config": {
          "shop_id": "shopify-559128571"
        }
      }
    ]
  }
}
```

**Config Schema:**

| Field | Type | Description |
|-------|------|-------------|
| `shop_id` | String (required) | Merchant's unique Shop Pay identifier |

**Instrument Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier for payment instrument |
| `handler_id` | string | Yes | Must match handler's id (e.g., `shop_pay`) |
| `type` | string | Yes | Must be `shop_pay` |
| `credential` | Object | Yes | Shop Pay credential containing token |
| `credential.type` | string | Yes | Must be `shop_token` |
| `credential.token` | string | Yes | Shop Token from delegated payment flow |
| `billing_address` | Object | Yes | Billing address |
| `email` | string | No | Buyer's email |

**Example Payment Instrument:**

```json
{
  "payment": {
    "instruments": [
      {
        "id": "instr_shop_pay_1",
        "handler_id": "shop_pay",
        "type": "shop_pay",
        "credential": {
          "type": "shop_token",
          "token": "shop_abc123xyz789..."
        },
        "email": "buyer@example.com",
        "billing_address": {
          "full_name": "Jane Doe",
          "street_address": "123 Main St",
          "address_locality": "San Francisco",
          "address_region": "CA",
          "postal_code": "94102",
          "address_country": "US"
        }
      }
    ],
    "selected_instrument_id": "instr_shop_pay_1"
  }
}
```

#### Agent Integration

**Requirements:**
- Register with Shop Pay to obtain `client_id` for delegated experiences

**Delegated Payment Protocol:**

1. **Discover handler:** Find `dev.shopify.shop_pay` in merchant's payment.handlers[]
2. **Build payment request:**
   - Initialize delegated payment context with merchant's `shop_id` and agent's `client_id`
   - Construct payment request with UCP checkout data (totals, fulfillment, line items, currency, locale)
   - Present to buyer through Shop Pay interface
3. **Complete checkout:**
   - Receive Shop Token from Shop Pay
   - Wrap token in UCP payment instrument
   - Submit checkout completion request

**Example Checkout Submission:**

```json
POST /checkout-sessions/{checkout_id}/complete
Content-Type: application/json
UCP-Agent: profile="https://agent.example/profile"

{
  "payment_data": {
    "id": "instr_shop_pay_1",
    "handler_id": "shop_pay",
    "type": "shop_pay",
    "credential": {
      "type": "shop_token",
      "token": "shop_abc123xyz789..."
    },
    "email": "buyer@example.com",
    "billing_address": {
      "full_name": "Jane Doe",
      "street_address": "123 Main St",
      "address_locality": "San Francisco",
      "address_region": "CA",
      "postal_code": "94102",
      "address_country": "US"
    }
  }
}
```

#### Merchant Processing

1. Validate handler: Confirm `handler_id` matches configured Shop Pay handler
2. Extract token: Retrieve token from `credential.token`
3. Process payment: Use Shop Token with Shop Pay's payment processing API
4. Return response: Finalized checkout state with order confirmation

#### Security Considerations

**Token Security:**
- Single-use tokens (cannot be reused)
- Time-limited validity
- Checkout-scoped (prevents usage outside verified checkout)
- TLS 1.2+ required for all exchanges

**Agent Authorization:**
- Must register for valid `client_id`
- `client_id` enables authorization tracking

**Schema References:**
- Config: `https://shopify.dev/ucp/shop-pay-handler/2026-01-11/config.json`
- Instrument: `https://shopify.dev/ucp/shop-pay-handler/2026-01-11/instrument.json`
- Credential: `https://shopify.dev/ucp/shop-pay-handler/2026-01-11/credential.json`

---

### 3. Google Pay Handler

**Handler Name:** `com.google.pay`
**Version:** `2026-01-11`

**Full Specification Summary:**

#### Introduction
- Enables Google Pay as accelerated checkout option
- Headless integration: businesses provide config, platforms handle client-side Google Pay API
- Leverages Google's tokenization to pass encrypted credentials to business's PSP

**Key Benefits:**
- Universal configuration: Businesses configure once with standard JSON
- Decoupled frontend: Platforms handle Google Pay JS API/SDK complexity
- Secure tokenization: Encrypted credentials pass directly to PSP

#### Business Integration

**Requirements:**
- Obtain Google Pay Merchant ID (register in Google Pay & Wallet Console for PRODUCTION)
- Verify PSP is in list of supported processors/gateways

**Handler Configuration:**

```json
{
  "payment": {
    "handlers": [
      {
        "id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
        "name": "com.google.pay",
        "version": "2026-01-11",
        "spec": "https://pay.google.com/gp/p/ucp/2026-01-11/",
        "config_schema": "https://pay.google.com/gp/p/ucp/2026-01-11/schemas/config.json",
        "instrument_schemas": [
          "https://pay.google.com/gp/p/ucp/2026-01-11/schemas/card_payment_instrument.json"
        ],
        "config": {
          "api_version": 2,
          "api_version_minor": 0,
          "environment": "TEST",
          "merchant_info": {
            "merchant_name": "Example Merchant",
            "merchant_id": "01234567890123456789",
            "merchant_origin": "checkout.merchant.com"
          },
          "allowed_payment_methods": [
            {
              "type": "CARD",
              "parameters": {
                "allowed_auth_methods": ["PAN_ONLY"],
                "allowed_card_networks": ["VISA", "MASTERCARD"]
              },
              "tokenization_specification": {
                "type": "PAYMENT_GATEWAY",
                "parameters": {
                  "gateway": "example",
                  "gatewayMerchantId": "exampleGatewayMerchantId"
                }
              }
            }
          ]
        }
      }
    ]
  }
}
```

**Config Schema Structure:**
- Based on Google Pay's API initialization requirements
- Includes environment, merchant info, allowed payment methods
- Tokenization specification defines how token is encrypted

**Instrument Schema:**

Extends base Card Payment Instrument with Google Pay specifics:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier (platform-assigned) |
| `handler_id` | string | Yes | Google Pay handler identifier |
| `type` | string | Yes | Constant = `card` |
| `brand` | string | Yes | Card network (mapped from Google Pay's `info.cardNetwork`) |
| `last_digits` | string | Yes | Last 4 digits (from `info.cardDetails`) |
| `rich_text_description` | string | No | Optional card description |
| `rich_card_art` | string (uri) | No | Optional issuer card art URI |
| `billing_address` | Postal Address | No | Billing address |
| `credential` | Credential Payload | No | Google Pay tokenization data |

**Credential Payload Structure:**

Maps directly to Google Pay Tokenization Data:

```json
{
  "type": "PAYMENT_GATEWAY",
  "token": "{\"signature\":\"...\",\"protocolVersion\":\"ECv2\"...}"
}
```

#### Platform Integration

**Requirements:**
- Capability to load Google Pay API for Web (or Android equivalent)
- Adhere to Google Pay Brand Guidelines

**Payment Protocol:**

1. **Discover & Configure:** Initialize PaymentsClient to manage API lifecycle
2. **Check Readiness:** Call `isReadyToPay()` to verify user can pay before showing button
3. **Build Payment Request:** Assemble PaymentDataRequest with merchant config, payment methods, transaction details
4. **Invoke User Interaction:** Trigger payment sheet with `loadPaymentData()` on button interaction
5. **Complete Checkout:** Map PaymentInstrument response to card_payment_instrument schema and submit

**Example Checkout Submission:**

```json
POST /checkout-sessions/{checkout_id}/complete

{
  "payment_data": {
    "id": "pm_1234567890abc",
    "handler_id": "8c9202bd-63cc-4241-8d24-d57ce69ea31c",
    "type": "card",
    "brand": "visa",
    "last_digits": "4242",
    "billing_address": {
      "street_address": "123 Main Street",
      "extended_address": "Suite 400",
      "address_locality": "Charleston",
      "address_region": "SC",
      "postal_code": "29401",
      "address_country": "US",
      "first_name": "Jane",
      "last_name": "Smith"
    },
    "credential": {
      "type": "PAYMENT_GATEWAY",
      "token": "{\"signature\":\"...\",\"protocolVersion\":\"ECv2\"...}"
    }
  },
  "risk_signals": {
    // ...
  }
}
```

#### Business Processing

1. **Validate Handler:** Confirm handler_id corresponds to Google Pay handler
2. **Extract Token:** Retrieve token string from `payment_data.credential.token`
3. **Process Payment:** Pass token string and transaction details to PSP endpoint
   - Note: Most PSPs have specific field for "Google Pay Payload" or "Network Token"
4. **Return Response:** Finalized checkout state (Success/Failure)

#### Security Considerations

**Token Security:**
- **PAYMENT_GATEWAY:** Token encrypted specifically for tokenization party; pass-through party cannot decrypt
- **DIRECT:** Business receives encrypted card data and must decrypt (increases PCI DSS scope, not recommended)

**Environment Isolation:**
- **TEST Mode:** Returns dummy tokens that cannot be charged
- **PRODUCTION Mode:** Real cards; PSP credentials must match environment

---

## Handler Comparison Matrix

| Aspect | Stripe SPT | Shop Pay | Google Pay |
|--------|-----------|----------|------------|
| **Integration Model** | Delegated Token | Delegated Token | Headless |
| **Platform Role** | Obtains SPT from Stripe | Orchestrates Shop Pay UI | Handles Google Pay JS API |
| **Business Role** | Processes SPT | Processes shop_token | Provides config, processes gateway token |
| **Config Complexity** | Simple (network_id + optional methods) | Minimal (shop_id only) | Complex (full Google Pay API config) |
| **Credential Type** | `stripe_shared_payment_token` | `shop_token` | `PAYMENT_GATEWAY` or `DIRECT` |
| **Token Security** | Checkout-bound to merchant | Single-use, checkout-scoped | PSP-encrypted |
| **Platform Registration** | With Stripe | With Shop Pay (client_id) | With Google Pay |
| **Business Registration** | Stripe network_id | Shop Pay shop_id | Google Pay merchant_id |
| **Environment Support** | Yes (via config or separate handlers) | Yes (environment field) | Yes (TEST/PRODUCTION) |

---

## Common Patterns Across All Handlers

### 1. Handler Declaration Structure
```json
{
  "id": "instance_id",
  "name": "reverse.dns.name",
  "version": "YYYY-MM-DD",
  "spec": "https://...",
  "config_schema": "https://...",
  "instrument_schemas": ["https://..."],
  "config": { /* handler-specific */ }
}
```

### 2. Three-Schema Approach
- **Config Schema:** What businesses advertise (merchant identifiers, capabilities)
- **Credential Schema:** Token/payment data structure
- **Instrument Schema:** Complete payment instrument (extends base + adds credential)

### 3. Delegated Payment Flow Pattern
1. Agent discovers handler in merchant's `payment.handlers[]`
2. Agent uses merchant's config to construct payment request
3. Payment provider/API returns token or encrypted payload
4. Agent wraps in UCP instrument schema
5. Merchant processes using handler-specific API

### 4. Security Model
- **Single-use tokens** (Shop Pay, Stripe SPT)
- **Time-limited validity**
- **Checkout/transaction binding** (prevents replay attacks)
- **Environment isolation** (TEST vs PRODUCTION)
- **Participant registration** (client_id, merchant_id, network_id)
- **TLS 1.2+ required**

### 5. Documentation Structure
1. Introduction & key benefits
2. Merchant/Business integration
   - Requirements
   - Handler configuration
   - Schema definitions
   - Processing flow
3. Agent/Platform integration
   - Requirements
   - Protocol/flow steps
   - Example requests
4. Security considerations
5. Schema references

### 6. Naming Conventions
- **Handler names:** Reverse-DNS format (`com.google.pay`, `dev.shopify.shop_pay`)
- **Versions:** Date format (YYYY-MM-DD)
- **Credential types:** Descriptive constants (`shop_token`, `stripe_shared_payment_token`, `PAYMENT_GATEWAY`)
- **Instrument types:** Match credential or base type (`shop_pay`, `card`)

### 7. Schema Extension Pattern
All handlers extend base schemas:

```json
{
  "allOf": [
    { "$ref": "base_schema.json" },
    {
      "type": "object",
      "properties": {
        "type": { "const": "handler_specific_type" },
        // Handler-specific fields
      }
    }
  ]
}
```

### 8. Config Schema Best Practices
- Include `environment` field (sandbox/production)
- Use descriptive field names
- Include examples
- Minimal required fields
- `additionalProperties: false` for strict validation

### 9. Error Handling
- Map handler-specific errors to standard UCP error codes
- Provide clear error messages
- Handle environment mismatches
- Validate handler_id matches expected handler

### 10. Risk Signals
Platforms may include risk signals in checkout submission:
```json
{
  "payment_data": { /* ... */ },
  "risk_signals": {
    // Platform-specific risk data
  }
}
```

---

## ACP Context

**ACP Repository:** `/Users/prasadw/code/agentic-commerce-protocol`

### Relevant Files

```
/Users/prasadw/code/agentic-commerce-protocol/
├── rfcs/
│   ├── rfc.capability_negotiation.md    # Existing capability negotiation RFC
│   ├── rfc.delegate_payment.md          # Current delegate payment RFC
│   ├── rfc.agentic_checkout.md          # Agentic checkout RFC
│   └── rfc.affiliate_attribution.md
├── spec/
│   ├── json-schema/
│   │   ├── schema.agentic_checkout.json
│   │   └── schema.delegate_payment_schema.json
│   └── openapi/
│       ├── openapi.agentic_checkout.yaml
│       ├── openapi.agentic_checkout_webhook.yaml
│       └── openapi.delegate_payment.yaml
└── examples/
    ├── examples.agentic_checkout.json
    ├── examples.capability_negotiation.json
    └── examples.delegate_payment.json
```

### ACP Capability Negotiation (Existing)

**Source:** `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.capability_negotiation.md`

**Key Concepts:**
- **Bidirectional negotiation:** Agents declare capabilities, Sellers advertise capabilities
- **Agent capabilities:** Interventions (3DS, OTP, biometric), features (async_completion, session_persistence)
- **Seller capabilities:** Payment methods, interventions, features
- **Location:** Top-level fields in CheckoutSession request/response
- **Requirement:** REQUIRED when extension is implemented

**Structure:**

```json
{
  "agent_capabilities": {
    "interventions": {
      "3ds": { "versions": ["2.1", "2.2"], "channels": ["browser", "mobile"] },
      "otp": {},
      "max_redirects": 1,
      "redirect_context": "in_app"
    },
    "features": {
      "async_completion": true,
      "session_persistence": true
    }
  }
}
```

```json
{
  "seller_capabilities": {
    "payment_methods": [
      {
        "method": "card",
        "brands": ["visa", "mastercard"],
        "funding_types": ["credit", "debit"],
        "providers": ["stripe"]
      },
      "card.network_token",
      "wallet.apple_pay"
    ],
    "interventions": {
      "required": ["3ds"],
      "3ds": { "versions": ["2.1", "2.2", "2.3"] }
    },
    "features": {
      "partial_auth": true,
      "saved_payment_methods": true
    }
  }
}
```

**Key Difference from UCP:**
- ACP uses broader capability negotiation at session level
- UCP uses handler-specific declarations in payment.handlers[]
- Both can coexist: ACP capabilities for high-level features, handlers for specific payment methods

### ACP Terminology Mapping

| UCP Term | ACP Term |
|----------|----------|
| Business | Seller |
| Platform | Agent |
| Merchant | Seller |
| Agent | Platform (in some contexts) |

---

## Key Design Questions for ACP Payment Handlers

### 1. Where do handlers fit in ACP?

**Options:**
- A. New top-level field in CheckoutSession: `payment_handlers[]`
- B. Extension of `seller_capabilities.payment_methods`
- C. Separate discovery endpoint
- D. Embedded in payment method objects

**Recommendation:** Consider Option A or B based on ACP's architecture philosophy.

### 2. How do handlers relate to capability negotiation?

**Options:**
- A. Handlers are a type of capability
- B. Handlers are separate from capabilities
- C. Handlers declare capabilities they require/support

**Recommendation:** Likely Option C - handlers can reference intervention/feature requirements.

### 3. Schema hosting and versioning

**Questions:**
- Who hosts handler schemas? (Centralized in ACP repo vs. handler providers)
- How are handler versions managed?
- How are schema URLs structured?

### 4. Handler discovery flow

**Questions:**
- When do Agents learn about available handlers? (Session creation response? Separate endpoint?)
- Can Agents filter/request specific handlers?
- How do Agents signal handler support?

### 5. Payment submission format

**Questions:**
- Does ACP adopt UCP's `payment_data` structure?
- How does it integrate with existing `delegate_payment` RFC?
- Where does `payment_instrument` fit in ACP's checkout completion?

### 6. Backward compatibility

**Questions:**
- How do existing ACP payment flows coexist with handlers?
- Migration path for existing implementations?
- Optional vs. required handler support?

### 7. Error handling and messages

**Questions:**
- How do handler-specific errors map to ACP's message system?
- Standard error codes for handler failures?
- Intervention handling for handler-specific auth requirements?

---

## Implementation Considerations

### For ACP Spec Authors

1. **Start with delegate_payment RFC:** Review existing delegate payment design to understand current payment flow
2. **Define handler declaration location:** Decide where handlers are advertised in CheckoutSession
3. **Create base handler schema:** Define ACP's equivalent of UCP's `payment_handler` schema
4. **Define credential/instrument schemas:** Base schemas that handlers can extend
5. **Integration with capability negotiation:** How handlers declare their capability requirements
6. **Example handler specification:** Create one complete example (e.g., Stripe SPT for ACP)
7. **Update OpenAPI specs:** Add handler-related endpoints and schemas
8. **Create JSON schemas:** Define validation schemas for handlers
9. **Documentation:** Handler specification guide (like UCP's payment-handler-guide.md)

### For Handler Authors

1. **Follow naming conventions:** Use reverse-DNS format
2. **Define three schemas:** Config, credential, instrument
3. **Document all flows:** Prerequisites, acquisition, processing
4. **Specify error mappings:** Handler errors → ACP error codes
5. **Include security considerations:** Token lifecycle, binding, environment isolation
6. **Provide complete examples:** Full JSON examples for all schemas and flows
7. **Reference base schemas:** Extend ACP base schemas, don't redefine

### For Implementers

1. **Sellers/Merchants:**
   - Determine which handlers to support
   - Complete handler prerequisites (registration, onboarding)
   - Advertise handlers in checkout sessions
   - Implement handler-specific processing logic
   - Map handler errors to checkout responses

2. **Agents/Platforms:**
   - Discover available handlers from sellers
   - Implement handler-specific acquisition flows
   - Handle handler registration requirements
   - Map handler responses to checkout instruments
   - Implement fallback for unsupported handlers

---

## Next Steps for ACP Payment Handlers

### Phase 1: Foundation
1. Read ACP's `rfc.delegate_payment.md` to understand current payment architecture
2. Analyze ACP's checkout session structure and payment flow
3. Design handler declaration structure for ACP
4. Define base schemas (handler, instrument, credential)
5. Determine relationship with capability negotiation

### Phase 2: Specification
1. Create RFC for ACP payment handlers
2. Define handler specification template
3. Create example handler (Stripe SPT or Shop Pay for ACP)
4. Update JSON schemas
5. Update OpenAPI specifications

### Phase 3: Documentation
1. Handler authoring guide
2. Implementation guide for sellers
3. Implementation guide for agents
4. Migration guide (if applicable)
5. Security best practices

### Phase 4: Examples
1. Create 2-3 complete handler specifications
2. Create example implementations
3. Create test vectors
4. Create validation tools

---

## File References

**UCP Files:**
- `/Users/prasadw/code/ucp/spec/schemas/capability.json`
- `/Users/prasadw/code/ucp/spec/schemas/shopping/types/payment_handler_resp.json`
- `/Users/prasadw/code/ucp/spec/schemas/shopping/types/payment_instrument_base.json`
- `/Users/prasadw/code/ucp/spec/schemas/shopping/types/payment_credential.json`
- `/Users/prasadw/code/ucp/spec/schemas/shopping/types/token_credential_resp.json`
- `/Users/prasadw/code/ucp/docs/specification/payment-handler-guide.md`
- `/Users/prasadw/code/ucp/docs/specification/examples/platform-tokenizer-payment-handler.md`

**ACP Files:**
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.capability_negotiation.md`
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.delegate_payment.md`
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.agentic_checkout.md`

---

## Glossary

| Term | Definition |
|------|------------|
| **Handler** | A payment method implementation specification (e.g., `com.google.pay`, `dev.shopify.shop_pay`) |
| **Config Schema** | JSON Schema defining handler configuration structure advertised by businesses/sellers |
| **Instrument Schema** | JSON Schema defining payment instrument structure submitted by platforms/agents |
| **Credential Schema** | JSON Schema defining credential structure within payment instrument |
| **Binding** | Cryptographic/logical association of payment instrument to specific checkout and business |
| **Delegated Payment** | Flow where agent/platform orchestrates payment on behalf of merchant, obtaining token |
| **Headless Integration** | Merchant provides config, platform handles UI/API interaction |
| **SPT** | Shared Payment Token (Stripe-specific) |
| **PSP** | Payment Service Provider (e.g., Stripe, Adyen) |
| **Tokenization** | Process of replacing sensitive payment data with non-sensitive token |
| **Prerequisites** | Onboarding/setup required before participating in handler flows |
| **Payment Identity** | Participant's identity within payment handler system |
| **Intervention** | User interaction required during payment (3DS, OTP, biometric) |

---

**End of Full Context Document**

---

**Last Updated:** 2026-01-22
**Document Version:** 1.0
**Purpose:** AI agent context for ACP payment handler design
