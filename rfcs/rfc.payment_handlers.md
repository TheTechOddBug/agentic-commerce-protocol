# RFC: Agentic Commerce — Payment Handlers

**Status:** Draft  
**Version:** 2026-01-22  
**Scope:** Payment handler framework for standardized payment method integration and credential delegation

This RFC introduces a **payment handler framework** for ACP that enables standardized, extensible payment method integration. Payment handlers consolidate payment method capabilities, credential flows, and security controls into a unified discovery and negotiation mechanism.

---

## 1. Motivation

As agentic commerce evolves, the ecosystem needs a standardized way to integrate diverse payment methods—from traditional cards to digital wallets, agentic tokens, and emerging payment types. The current approach has payment methods as simple identifiers, which limits:

- **Extensibility**: Adding new payment methods requires protocol changes
- **Configuration richness**: Payment methods need varying configuration (merchant IDs, network identifiers, tokenization specs)
- **Security clarity**: The relationship between payment methods and credential delegation is implicit
- **N-to-N compatibility**: Agents and Sellers need to discover detailed payment method capabilities before checkout

This RFC introduces **payment handlers**—a framework that:

1. **Unifies payment methods and capabilities**: Each handler fully describes a payment method including configuration, credential schemas, and requirements
2. **Standardizes delegation**: Handlers explicitly declare whether credential delegation is required, recommended, or optional
3. **Enables extensibility**: New payment methods can be added without protocol changes by defining new handler specifications
4. **Supports diverse flows**: Handlers accommodate different integration patterns (delegated tokens, wallet credentials, direct card data)

---

## 2. Goals and Non-Goals

### 2.1 Goals

1. **Unified payment method framework**: Replace simple payment method identifiers with rich handler declarations
2. **Explicit delegation semantics**: Make credential delegation requirements clear and enforceable at the handler level
3. **Provider flexibility**: Support both same-PSP and cross-PSP scenarios
4. **Wallet integration**: Enable standardized integration of digital wallets (Stripe Link, Apple Pay, Shop Pay, Google Pay)
5. **Security by default**: Make `delegate_payment` the recommended approach while allowing handlers to opt out when appropriate
6. **Forward compatibility**: Enable new payment methods through handler specifications without protocol changes

### 2.2 Non-Goals

- Defining specific handler implementations (covered in separate handler specification documents)
- Changing the `delegate_payment` API endpoint (endpoint remains as-is)
- Backward compatibility with the current payment_methods array (this is a breaking change)
- Standardizing PSP-specific APIs or token formats
- Creating a centralized handler registry (handlers are discovered per-session)

---

## 3. Design Rationale

### 3.1 Why consolidate payment_methods into handlers?

The current `payment_methods` array in `seller_capabilities` provides limited information:

```json
"payment_methods": ["card", "card.agentic_token", "wallet.stripe.link"]
```

This approach cannot express:
- Configuration needed to use each method (merchant IDs, network identifiers)
- Credential schemas and requirements
- Whether delegation is required
- PSP-specific constraints
- Environment settings (sandbox vs production)

Payment handlers address these limitations by making each payment method a **rich, self-describing object** with configuration, schemas, and security requirements.

### 3.2 Why handlers, not just enhanced payment methods?

The term "handler" reflects that each payment method defines:
- How to **handle** credential acquisition (agent's responsibility)
- How to **handle** credential processing (seller's responsibility)  
- How to **handle** delegation and security
- Complete **handler** specifications that can evolve independently

This aligns with industry patterns (EMVCo, payment networks) where payment methods are implemented as handlers with defined behaviors.

### 3.3 Why require delegate_payment by default?

The `delegate_payment` endpoint provides critical security properties:
- **Allowance constraints**: Clear limits on amount, currency, expiry
- **Risk signals**: Standardized risk assessment data
- **Token lifecycle**: Explicit creation, binding, and expiry
- **Auditability**: Clear separation of credential vault from payment processing

Making `delegate_payment` the **recommended default** (via `requires_delegate_payment: true`) ensures these properties are preserved while allowing handlers to opt out when the pattern doesn't fit (e.g., certain wallet flows where the wallet provider's token already embeds these constraints).

### 3.4 Handler as specification + instance

Each payment handler serves dual roles:

1. **Specification**: A documented standard (e.g., "com.google.pay") defining schemas, flows, and requirements
2. **Instance**: A seller's specific configuration for that handler (merchant IDs, accepted networks, environment settings)

This duality enables:
- Agents to implement handler specifications once and work with any seller supporting that handler
- Sellers to advertise their specific configuration for each handler
- The ecosystem to add new handlers without protocol changes

---

## 4. Terminology

### 4.1 Core Concepts

**Payment Handler**: A specification and configuration for a payment method, including:
- Unique identifier (specification name in reverse-DNS format)
- Configuration schema (what sellers must provide)
- Instrument schema (what agents submit at checkout)
- Credential schema (payment data structure)
- Delegation requirements
- Processing requirements

**Handler Specification**: The documented standard for a handler (e.g., "com.google.pay"), typically hosted at a public URL, defining all schemas and flows.

**Handler Instance**: A seller's specific configuration of a handler, advertised in `seller_capabilities.payment_handlers[]`.

**Handler ID**: A unique identifier for a handler instance within a checkout session (e.g., "gpay_main", "card_stripe").

**Payment Instrument**: The complete payment data submitted by an agent, conforming to a handler's instrument schema.

**Credential**: The sensitive payment data within an instrument (e.g., card number, wallet token, vault token).

**Delegation**: The process of using the `delegate_payment` endpoint to vault credentials and obtain a token with allowance constraints.

### 4.2 Terminology Clarifications

| ACP Term | Also Known As | Description |
|----------|---------------|-------------|
| Seller | Business / Merchant | Party accepting payment |
| Seller Platform | Platform | Infrastructure/system that sellers use for commerce (e.g., Shopify) |
| PSP | Payment Service Provider | Service processing payment transactions (e.g., Stripe, Adyen) |
| Agent | Facilitator / Orchestrator | Party orchestrating checkout on behalf of user |
| Payment Handler | Payment Method Handler | Specification + configuration for payment method |
| Handler Instance | Handler Declaration | Seller's advertised handler config |
| Requires Delegation | Delegation Requirement | Whether delegate_payment must be used |

---

## 5. Handler Structure

### 5.1 Handler Declaration (in seller_capabilities)

Each handler in `seller_capabilities.payment_handlers[]` follows this structure:

```json
{
  "id": "handler_instance_id",
  "name": "reverse.dns.format",
  "version": "YYYY-MM-DD",
  "spec": "https://example.com/handler-spec",
  "requires_delegate_payment": true,
  "requires_pci_compliance": false,
  "psp": "stripe",
  "config_schema": "https://example.com/config-schema.json",
  "instrument_schemas": ["https://example.com/instrument-schema.json"],
  "config": {
    // Handler-specific configuration
  }
}
```

**Field Definitions:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier for this handler instance within the session. Used by payment instruments to reference which handler they conform to. |
| `name` | string | ✅ | Handler specification name in reverse-DNS format (e.g., `dev.acp.card`, `com.google.pay`, `dev.shopify.shop_pay`). |
| `version` | string | ✅ | Handler specification version in YYYY-MM-DD format. |
| `spec` | URI | ✅ | URL to human-readable handler specification document. |
| `requires_delegate_payment` | boolean | ✅ | Whether this handler requires use of the `/agentic_commerce/delegate_payment` endpoint. **Default recommendation: `true`**. |
| `requires_pci_compliance` | boolean | ✅ | Whether this handler processes PCI DSS sensitive data. **RECOMMENDED: `false`**. Handlers SHOULD leverage tokenization to avoid passing PCI-sensitive data. However, if a handler absolutely requires PCI-sensitive data, this field MUST be set to `true` so Seller can route requests to PCI-compliant infrastructure. **Default: `false`**. |
| `psp` | string | ✅ | Payment Service Provider identifier (e.g., `stripe`, `adyen`, `braintree`). Enables Agent to determine if they share the same PSP with Seller for credential vaulting decisions. |
| `config_schema` | URI | ✅ | URL to JSON Schema for validating the `config` object. |
| `instrument_schemas` | array of URIs | ✅ | URLs to JSON Schemas defining valid payment instrument structures for this handler. |
| `config` | object | ✅ | Handler-specific configuration (structure validated by `config_schema`). |

### 5.2 Payment Instrument Structure

All payment instruments extend this base schema:

```json
{
  "id": "instrument_id",
  "handler_id": "handler_instance_id",
  "type": "handler_specific_type",
  "credential": {
    // Credential structure defined by handler
  },
  "billing_address": {
    // Optional billing address
  }
}
```

**Base Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier for this instrument instance, assigned by the Agent. |
| `handler_id` | string | ✅ | References the `id` of the handler that produced this instrument. |
| `type` | string | ✅ | Payment instrument type identifier (defined by handler specification). |
| `credential` | object | ✅ | Credential payload (structure defined by handler's credential schema). |
| `billing_address` | Address | ❌ | Optional billing address associated with payment method. |

### 5.3 Handler Discovery Flow

```
1. Agent → POST /checkout_sessions
   {
     "agent_capabilities": { ... }
   }

2. Seller → Response with seller_capabilities
   {
     "seller_capabilities": {
       "payment_handlers": [
         {
           "id": "card_primary",
           "name": "dev.acp.card",
           "requires_delegate_payment": true,
           "config": { "accepted_brands": ["visa", "mastercard"] }
         },
         {
           "id": "link_checkout",
           "name": "com.stripe.link",
           "requires_delegate_payment": false,
           "config": { "merchant_id": "...", ... }
         }
       ]
     }
   }

3. Agent discovers available handlers and their configurations

4. Agent acquires payment credential (via user input, wallet, etc.)

5a. If handler requires_delegate_payment = true:
    Agent → POST /agentic_commerce/delegate_payment
    { credential, allowance, ... }
    ← Response with vault token
    
5b. If handler requires_delegate_payment = false:
    Agent can choose to use delegate_payment (recommended) or bypass it

6. Agent → POST /checkout_sessions/{id}/complete
   {
     "payment_data": {
       "handler_id": "card_primary",
       "credential": { /* vault token or direct credential */ },
       ...
     }
   }
```

---

## 6. Relationship with delegate_payment

### 6.1 The `requires_delegate_payment` Field

Each handler declares whether credential delegation is required:

**`requires_delegate_payment: true`** (RECOMMENDED DEFAULT)
- Agent **MUST** call `/agentic_commerce/delegate_payment` before checkout completion
- Agent submits the vault token in the payment instrument's credential
- Seller uses vault token with their PSP to charge payment
- Provides allowance constraints, risk signals, and security guarantees

**`requires_delegate_payment: false`**
- Agent **MAY** call `/agentic_commerce/delegate_payment` for added security (still recommended)
- OR Agent **MAY** submit credentials directly in the payment instrument
- Useful for payment methods where the provider's token already includes security constraints (e.g., some wallet tokens)

### 6.2 Why Delegation is Recommended

Regardless of `requires_delegate_payment` setting, using the `delegate_payment` endpoint provides:

1. **Explicit allowance bounds**: `max_amount`, `currency`, `expires_at`, `checkout_session_id`
2. **Risk signal standardization**: Consistent risk data format across payment methods
3. **Credential scoping**: Tokens bound to specific checkout sessions and merchants
4. **Audit trail**: Clear record of credential creation and usage
5. **Security separation**: Vault infrastructure separate from payment processing

Handlers SHOULD set `requires_delegate_payment: true` unless there's a specific reason the delegation pattern doesn't fit their flow.

### 6.3 delegate_payment API Remains Unchanged

The existing `/agentic_commerce/delegate_payment` endpoint defined in the Delegate Payment RFC remains unchanged. Payment handlers extend how it's used, not the endpoint itself.

### 6.4 PSP-Based Routing

The `psp` field in handler declarations enables Agents to make intelligent routing decisions:

**Same PSP Scenario** (Agent and Seller both use `stripe`):
1. Agent sees handler has `psp: "stripe"` 
2. Agent recognizes it shares the same PSP
3. Agent calls `delegate_payment` with credential
4. PSP (Stripe) creates vault token (SPT) accessible to both parties
5. Agent submits SPT to Seller
6. Seller unwraps SPT using shared PSP infrastructure

**Different PSP Scenario** (Agent uses `stripe`, Seller uses `adyen`):
1. Agent sees handler has `psp: "adyen"`
2. Agent recognizes different PSP
3. **If `requires_delegate_payment: true`:** Agent must still call `delegate_payment` with own PSP, then coordinate cross-PSP token translation
4. **If `requires_delegate_payment: false`:** Agent MAY pass credential directly to Seller (who processes with their PSP)

This explicit PSP declaration eliminates ambiguity and enables optimal credential routing.

---

## 7. Handler Types Overview

This framework supports multiple handler integration patterns based on credential types and tokenization strategies:

### 7.1 Type 1: Card Handlers (Agentic Tokens)

**Credential Types**:
- **Agentic Tokens**: MC/Visa tokens specifically designed for agentic commerce
- **FPAN**: Funding Primary Account Number (raw card number)

**Flow**:
1. Agent acquires card credential (FPAN or agentic token)
2. Agent calls `delegate_payment` with credential + allowance
3. PSP creates vault token (SPT) with allowance constraints
4. Agent submits vault token to Seller
5. Seller uses vault token with PSP to unwrap and charge

**Handler Names**:
- `dev.acp.card` — Generic card handler
- `dev.acp.card.agentic_token` — Agentic tokens (MC/Visa)

**`requires_delegate_payment`**: `true`

**`requires_pci_compliance`**: `true` (when handling FPAN) or `false` (when handling only tokens)

**`psp`**: Seller's PSP identifier (e.g., `"stripe"`, `"adyen"`)

**Cross-PSP Capability**: 
- **Same PSP (Agent and Seller)**: Direct SPT usage - both parties access same vault
- **Different PSPs**: SPT header contains routing information; cross-PSP token translation required OR Seller's PSP unwraps via tokenization service
- Agent uses `psp` field to determine routing strategy

**Key Properties**:
- Allowance enforcement by PSP
- Risk signals included in delegation
- Single-use tokens with checkout binding
- Test mode support
- PCI-compliant infrastructure required for FPAN processing

### 7.2 Type 2: Wallet Handlers

**Wallet Types**:
- **Platform Wallets**: Stripe Link, Shop Pay
- **Digital Wallets**: Apple Pay, Google Pay
- **BNPL Wallets**: Affirm, Klarna (when using native tokens)

**Flow**:
1. Agent orchestrates wallet interaction (API/SDK)
2. Wallet provider returns payment token (scope depends on wallet)
3. Agent calls `delegate_payment` with wallet token + allowance (RECOMMENDED but optional)
4. If delegation used: PSP mints SPT wrapping wallet token
5. If delegation bypassed: Agent submits wallet token directly with allowance metadata
6. Seller's PSP processes wallet token (validates allowance if present)

**Handler Names**:
- `com.stripe.link` — Stripe Link
- `com.apple.pay` — Apple Pay
- `dev.shopify.shop_pay` — Shop Pay
- `com.google.pay` — Google Pay

**`requires_delegate_payment`**: `false` (delegation still recommended for security/allowance enforcement)

**`requires_pci_compliance`**: `false` (wallet tokens do not contain raw PCI data)

**`psp`**: Wallet provider or Seller's PSP handling wallet processing

**Cross-PSP Capability**:
- Wallet tokens contain PSP-specific routing (e.g., Stripe Link configuration, Apple Pay token specifications)
- Agent checks `psp` field to determine if delegation needed
- **Same PSP**: Agent can vault wallet token with shared PSP for standardized SPT wrapper
- **Different PSPs**: Wallet token passed directly; PSP-agnostic routing via wallet network
- When delegation bypassed: Allowance enforcement depends on wallet provider

**Key Properties**:
- No PCI-sensitive data exposed to non-compliant systems
- Wallet-specific tokenization handles card data securely
- Seller can route to standard (non-PCI) infrastructure

### 7.3 Type 3: BNPL Handlers (Buy Now Pay Later)

**BNPL Providers**: Affirm, Klarna, Afterpay, Cash App Pay

**Flow**:
1. Agent initiates BNPL auth with provider
2. BNPL provider returns authorization token
3. Agent calls `delegate_payment` with BNPL token + allowance
4. PSP mints SPT containing BNPL authorization
5. Agent submits SPT to Seller
6. Seller unwraps SPT and processes with BNPL provider

**Handler Names**:
- `com.affirm.pay` — Affirm
- `com.klarna.pay` — Klarna
- `com.afterpay.pay` — Afterpay

**`requires_delegate_payment`**: `true`

**`requires_pci_compliance`**: `false` (BNPL tokens do not contain raw PCI data)

**`psp`**: Seller's PSP identifier (may differ from BNPL provider)

**Cross-PSP Capability**:
- SPT standardizes BNPL integration across agents/sellers
- Agent checks `psp` field to determine delegation strategy
- **On-Stripe (same PSP)**: Direct processing with negotiated buy rates
- **Off-Stripe (different PSP)**: SPT routing to merchant's BNPL integration OR virtual card fallback
- Virtual card fallback enables cross-PSP compatibility at higher cost

**Virtual Card Fallback**:
- When direct BNPL integration unavailable, PSP may issue virtual card
- Virtual card charged to Seller's processor; BNPL funds virtual card
- Higher fees due to two-leg transaction

**Key Properties**:
- No PCI-sensitive data in BNPL authorization tokens
- Seller can route to standard (non-PCI) infrastructure
- Virtual card fallback may require PCI compliance depending on implementation

### 7.4 Tokenization Standards

All handlers leverage tokenization to secure credentials:

| Token Type | Description | Scope | Example Handler Types |
|------------|-------------|-------|----------------------|
| **SPT (Shared Payment Token)** | ACP's standardized delegation token | Cross-agent, cross-seller | All handlers when `requires_delegate_payment: true` |
| **Agentic Token** | MC/Visa tokens for agentic commerce | Network-scoped | Card handlers |
| **Wallet Token** | Provider-specific token (Apple, Google, Shop) | Provider-scoped | Wallet handlers |
| **BNPL Auth Token** | Provider authorization token | Provider-scoped | BNPL handlers |

**SPT as Universal Wrapper**:
- SPT can wrap any underlying token type
- Provides consistent allowance constraints across payment methods
- Enables standardized unwrapping at Seller's PSP
- Facilitates cross-PSP routing via header metadata

**Note**: Detailed handler specifications, cross-PSP token translation, and schema definitions will be addressed in subsequent steps.

---

## 8. Example Handler Declarations

### 8.1 Card Handler (Delegated, Same-PSP)

```json
{
  "id": "card_primary",
  "name": "dev.acp.card",
  "version": "2026-01-22",
  "spec": "https://acp.dev/specs/handlers/card",
  "requires_delegate_payment": true,
  "requires_pci_compliance": true,
  "config_schema": "https://acp.dev/schemas/handlers/card/config.json",
  "instrument_schemas": [
    "https://acp.dev/schemas/handlers/card/instrument.json"
  ],
  "config": {
    "accepted_brands": ["visa", "mastercard", "amex"],
    "accepted_funding_types": ["credit", "debit"],
    "psp": "stripe",
    "environment": "production"
  }
}
```

### 8.2 Stripe Link Handler (Wallet)

```json
{
  "id": "link_checkout",
  "name": "com.stripe.link",
  "version": "2026-01-22",
  "spec": "https://stripe.com/docs/acp/handlers/link",
  "requires_delegate_payment": false,
  "requires_pci_compliance": false,
  "config_schema": "https://stripe.com/acp/schemas/link/config.json",
  "instrument_schemas": [
    "https://stripe.com/acp/schemas/link/instrument.json"
  ],
  "config": {
    "merchant_id": "acct_1234567890",
    "supported_payment_methods": ["card", "bank_account"],
    "environment": "production"
  }
}
```

### 8.3 Apple Pay Handler (Wallet)

```json
{
  "id": "apple_pay_main",
  "name": "com.apple.pay",
  "version": "2026-01-22",
  "spec": "https://apple.com/apple-pay/acp-integration",
  "requires_delegate_payment": false,
  "requires_pci_compliance": false,
  "config_schema": "https://apple.com/acp/schemas/apple-pay/config.json",
  "instrument_schemas": [
    "https://apple.com/acp/schemas/apple-pay/instrument.json"
  ],
  "config": {
    "merchant_id": "merchant.com.example",
    "supported_networks": ["visa", "mastercard", "amex"],
    "merchant_capabilities": ["supports3DS"],
    "environment": "production"
  }
}
```

### 8.4 Shop Pay Handler (Wallet)

```json
{
  "id": "shop_pay_main",
  "name": "dev.shopify.shop_pay",
  "version": "2026-01-22",
  "spec": "https://shopify.dev/docs/acp/shop-pay-handler",
  "requires_delegate_payment": false,
  "requires_pci_compliance": false,
  "config_schema": "https://shopify.dev/acp/shop-pay/config.json",
  "instrument_schemas": [
    "https://shopify.dev/acp/shop-pay/instrument.json"
  ],
  "config": {
    "shop_id": "shopify-559128571",
    "environment": "production"
  }
}
```

### 8.5 Google Pay Handler (Wallet)

```json
{
  "id": "gpay_checkout",
  "name": "com.google.pay",
  "version": "2026-01-22",
  "spec": "https://pay.google.com/acp/handlers/google-pay",
  "requires_delegate_payment": false,
  "requires_pci_compliance": false,
  "config_schema": "https://pay.google.com/acp/schemas/config.json",
  "instrument_schemas": [
    "https://pay.google.com/acp/schemas/instrument.json"
  ],
  "config": {
    "api_version": 2,
    "api_version_minor": 0,
    "environment": "PRODUCTION",
    "merchant_info": {
      "merchant_name": "Example Merchant",
      "merchant_id": "01234567890123456789"
    },
    "allowed_payment_methods": [
      {
        "type": "CARD",
        "parameters": {
          "allowed_auth_methods": ["PAN_ONLY", "CRYPTOGRAM_3DS"],
          "allowed_card_networks": ["VISA", "MASTERCARD"]
        },
        "tokenization_specification": {
          "type": "PAYMENT_GATEWAY",
          "parameters": {
            "gateway": "stripe",
            "gatewayMerchantId": "acct_1234567890"
          }
        }
      }
    ]
  }
}
```

---

## 9. Migration from payment_methods

### 9.1 Breaking Change Notice

This RFC introduces a **breaking change**: `seller_capabilities.payment_methods[]` is **removed** and **replaced** with `seller_capabilities.payment_handlers[]`.

**Before (capability_negotiation RFC):**
```json
{
  "seller_capabilities": {
    "payment_methods": ["card", "card.agentic_token", "wallet.stripe.link"],
    "features": { ... }
  }
}
```

**After (payment_handlers RFC):**
```json
{
  "seller_capabilities": {
    "payment_handlers": [
      {
        "id": "card_main",
        "name": "dev.acp.card",
        "version": "2026-01-22",
        "requires_delegate_payment": true,
        "config": { ... }
      },
      {
        "id": "link_main",
        "name": "com.stripe.link",
        "version": "2026-01-22",
        "requires_delegate_payment": false,
        "config": { ... }
      }
    ],
    "features": { ... }
  }
}
```

### 9.2 Migration Strategy

**For Sellers:**
1. Map each payment method to a handler specification
2. Define handler configuration for each supported method
3. Replace `payment_methods[]` with `payment_handlers[]` in responses

**For Agents:**
1. Update session creation logic to parse `payment_handlers[]`
2. Implement handler-specific credential acquisition flows
3. Update completion logic to reference handler IDs in payment instruments

---

## 10. Open Questions for Next Steps

These questions will be addressed in subsequent steps:

1. **Cross-PSP token translation**: How do wallet tokens work when Agent and Seller use different PSPs?
2. **Handler specification template**: What's the standard structure for documenting new handlers?
3. **Credential schema standards**: What are the base credential types (card, token, wallet_credential)?
4. **Allowance enforcement**: When delegation is bypassed, how are allowance limits communicated and enforced?
5. **Handler versioning**: How do agents handle multiple versions of the same handler?
6. **Error handling**: What errors occur when handler requirements aren't met?

---

## 11. Next Steps

**Step 2**: Define base schemas (handler, credential, instrument)  
**Step 3**: Specify Type 1 handler (delegated payment, same-PSP)  
**Step 4**: Specify Type 2 handler (wallet handlers, cross-PSP)  
**Step 5**: Cross-PSP considerations and token handling  
**Step 6**: Complete examples, OpenAPI updates, and handler specification template

---

## 12. Change Log

- **2026-01-22**: Initial draft — Foundation and terminology for payment handlers framework

---

**Status**: Ready for review — Step 1 complete, awaiting feedback before proceeding to Step 2.
