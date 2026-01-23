# Payment Handlers Context - Brief Summary

**Date:** 2026-01-22  
**Purpose:** Quick briefing for starting new agent conversation on ACP payment handlers

---

## Context Summary

We're designing a payment handler framework for ACP (Agentic Commerce Protocol) based on patterns from UCP (Universal Checkout Protocol).

### What are Payment Handlers?

Payment handlers are standardized specifications that enable N-to-N interoperability between:
- **Sellers/Merchants** (who accept payments)
- **Agents/Platforms** (who orchestrate checkout flows)
- **Payment Providers** (like Stripe, Shop Pay, Google Pay)

Each handler defines:
1. **Config Schema** - What sellers advertise to indicate support
2. **Credential Schema** - Structure of payment tokens/data
3. **Instrument Schema** - Complete payment instrument submitted at checkout
4. **Integration flows** - How each participant interacts with the handler

---

## Key Patterns from UCP

### Handler Declaration Structure

```json
{
  "payment": {
    "handlers": [
      {
        "id": "unique_instance_id",
        "name": "reverse.dns.format",        // e.g., com.google.pay
        "version": "2026-01-11",             // Date format
        "spec": "https://...",               // Specification URL
        "config_schema": "https://...",      // Config validation schema
        "instrument_schemas": ["https://..."], // Instrument schemas
        "config": {
          // Handler-specific config (e.g., merchant_id, network_id)
        }
      }
    ]
  }
}
```

### Three Integration Models

1. **Delegated Token** (Stripe SPT, Shop Pay)
   - Agent obtains token from payment provider
   - Merchant processes token
   - Token bound to checkout + merchant

2. **Headless** (Google Pay)
   - Merchant provides configuration
   - Agent handles UI/API interaction
   - Returns encrypted payment data

3. **Tokenization** (Platform/Business tokenizers)
   - One party stores credentials, issues tokens
   - Other party detokenizes for processing

---

## Example Handlers Analyzed

### 1. Stripe Shared Payment Token (SPT)
- **Config:** `stripe_network_id`, optional `supported_payment_methods[]`
- **Flow:** Agent gets SPT from Stripe → Merchant processes with network_id
- **Credential:** `stripe_shared_payment_token` type

### 2. Shop Pay
- **Config:** `shop_id` (minimal)
- **Flow:** Agent orchestrates Shop Pay UI → gets `shop_token` → Merchant processes
- **Credential:** `shop_token` type
- **Registration:** Agent needs `client_id`, Merchant needs `shop_id`

### 3. Google Pay
- **Config:** Full Google Pay API config (complex nested structure)
- **Flow:** Merchant provides config → Agent handles Google Pay JS API → Returns gateway token
- **Credential:** `PAYMENT_GATEWAY` type with encrypted payload

---

## Common Security Patterns

- **Single-use tokens**
- **Time-limited validity**
- **Checkout binding** (prevents replay attacks)
- **Environment isolation** (TEST vs PRODUCTION)
- **Participant registration** (client_id, merchant_id, network_id)
- **TLS 1.2+ required**

---

## ACP Context

**Repository:** `/Users/prasadw/code/agentic-commerce-protocol`

### Relevant RFCs
- **`rfc.capability_negotiation.md`** - Existing bidirectional capability negotiation
- **`rfc.delegate_payment.md`** - Current payment delegation approach
- **`rfc.agentic_checkout.md`** - Main checkout flow

### ACP Already Has
- Capability negotiation (agent_capabilities ↔ seller_capabilities)
- Payment methods in seller_capabilities
- Intervention handling (3DS, OTP, etc.)
- Checkout session flow

### What We Need to Design
1. **Where handlers fit** - New field? Extension of payment_methods?
2. **Handler declaration structure** - Adapt UCP pattern to ACP
3. **Base schemas** - Handler, instrument, credential schemas for ACP
4. **Integration with capabilities** - How handlers relate to existing capability negotiation
5. **Discovery flow** - When/how agents discover available handlers
6. **Payment submission** - Format for submitting handler-based payments
7. **Documentation** - Handler specification guide, examples

---

## Key Design Questions

1. **Handler location:** Where in CheckoutSession should handlers be advertised?
2. **Relationship to capabilities:** Are handlers a type of capability or separate?
3. **Schema hosting:** Centralized in ACP repo vs. provider-hosted?
4. **Discovery timing:** Session creation? Separate endpoint?
5. **Backward compatibility:** How do existing payment flows coexist with handlers?
6. **Error handling:** How do handler errors map to ACP's message system?

---

## Terminology Mapping

| UCP Term | ACP Term |
|----------|----------|
| Business | Seller |
| Platform | Agent |
| Merchant | Seller |

---

## Next Steps

1. **Read** `rfc.delegate_payment.md` to understand current ACP payment architecture
2. **Analyze** ACP's checkout session structure
3. **Design** handler declaration structure for ACP
4. **Define** base schemas
5. **Create** RFC for ACP payment handlers
6. **Build** example handler specification

---

## Files to Reference

**UCP:**
- `/Users/prasadw/code/ucp/docs/specification/payment-handler-guide.md`
- `/Users/prasadw/code/ucp/spec/schemas/shopping/types/payment_handler_resp.json`
- Full details in `payment-handlers-context-full.md`

**ACP:**
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.capability_negotiation.md`
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.delegate_payment.md`
- `/Users/prasadw/code/agentic-commerce-protocol/rfcs/rfc.agentic_checkout.md`

---

**For complete details, schemas, and examples, see:** `payment-handlers-context-full.md`
