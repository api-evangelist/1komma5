---
name: Build and send a 1KOMMA5° solar offer
description: Create a customer and site in the 1KOMMA5° Offer Tool, price a configuration
  against the German effective-price simulation, render the offer PDF and send it to the
  customer for signature — then handle rejection back into the CRM.
api: openapi/1komma5-offer-tool-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/1komma5-offer-tool-openapi-original.json
operations:
- CreateCustomerController_handle_v1
- CreateCustomerSiteController_handle_v1
- GetCustomerDetailsController_handle_v1
- QueueSiteEnergySiteInfoRefreshController_queue_v1
- GetConceptsController_getAll_v1
- GetConfigController_get_v1
- RefreshConfigEnergySiteInfoController_refresh_v1
- EffectivePriceController_checkSimulationDataDe_v1
- EffectivePriceController_getEffectivePriceDe_v1
- GetAllPaymentOptionsController_handle_v1
- PdfController_generatePdf_v1
- SendOfferController_sendOffer_v1
- GetDealController_getDeal_v1
- RejectOfferController_handle_v1
---

# Build and send a 1KOMMA5° solar offer

Use this when quoting a residential PV / battery / heat-pump / wallbox system through the
1KOMMA5° Offer Tool API.

## Before you start

- Base URL: `https://api.offer.1komma5grad.com`
- Every call needs an Auth0 bearer JWT in the `authorization` header. Get one from
  `https://auth.1komma5grad.com/oauth/token`. See
  `authentication/1komma5-authentication.yml`.
- Confirm which tenant you are acting in first — the Offer Tool is multi-tenant and list
  results are scoped to the caller's active tenant. Use
  `TenantController_getTenants_v1` and `TenantController_switchActiveTenant_v1`.
- **There is no idempotency contract.** Do not blindly retry `POST` operations; a retried
  `CreateCustomerController_handle_v1` will create a second customer. See
  `conventions/1komma5-conventions.yml`.

## Steps

1. **Create or find the customer.** `CreateCustomerController_handle_v1`
   (`POST /api/v1/customers`). To check for an existing record first, list with
   `GetAllCustomersController_handle_v1` (`GET /api/v1/customers`) using the `search`
   query parameter; it also accepts `zohoIdSearch` if you already hold the Zoho CRM id.
   Read the full record back with `GetCustomerDetailsController_handle_v1`.
2. **Attach the installation site.** `CreateCustomerSiteController_handle_v1`
   (`POST /api/v1/customers/{customerId}/sites`). The site carries the address that drives
   country, locale and currency.
3. **Pull the site's energy-service data.**
   `QueueSiteEnergySiteInfoRefreshController_queue_v1`
   (`POST /api/v1/sites/{id}/lookup-energy-service-data`) queues the lookup. This is
   asynchronous — it returns before the data lands.
4. **Choose a concept (offer template).** `GetConceptsController_getAll_v1`
   (`GET /api/v1/concepts-public`) returns the templates visible to the caller.
5. **Read the configuration.** `GetConfigController_get_v1` (`GET /api/v1/configs/{id}`).
   Refresh its energy-service data with
   `RefreshConfigEnergySiteInfoController_refresh_v1` — note this operation declares
   `security: [{bearer: []}]` and a required `authorization` header parameter explicitly.
   > **Gap to be aware of:** the public description exposes no operation that *creates* a
   > configuration. Configs appear to be created by the Offer Tool front end through a
   > surface that is not in the published spec. Do not invent one — treat the config id as
   > an input to this flow.
6. **Price it for the German market.** Validate the inputs with
   `EffectivePriceController_checkSimulationDataDe_v1`
   (`POST /api/v1/effective-price/de/check-simulation-data`), then compute with
   `EffectivePriceController_getEffectivePriceDe_v1` (`POST /api/v1/effective-price/de`).
   A `400` here means the simulation payload failed to parse; a `404` means the scenario
   or its post-processed result is missing.
7. **Add payment options.** `GetAllPaymentOptionsController_handle_v1`
   (`GET /api/v1/payment-options`).
8. **Render the PDF.** `PdfController_generatePdf_v1`
   (`POST /api/v1/configs/{configId}/pdf`). This is the one operation that documents the
   full failure set: `400` (configId not an integer / unparseable data), `401`
   (unauthorized), `503` (PDF service unavailable). Retry `503` with backoff.
9. **Send the offer.** `SendOfferController_sendOffer_v1`
   (`POST /api/v1/configs/{id}/send-offer`). The response carries a
   `getAcceptDocumentId` — delivery and e-signature go through GetAccept.
10. **Track it in the CRM.** `GetDealController_getDeal_v1`
    (`GET /api/v1/crm/deals/{dealId}`) reads the Zoho deal. If the customer declines, call
    `RejectOfferController_handle_v1` (`PATCH /api/v1/crm/reject-offer/{crmOfferId}`).

## Error handling

Errors come back as `{"message":…,"path":…,"statusCode":…,"success":false}` — **not** RFC
9457 problem+json, and the Offer Tool envelope carries no trace id. Full catalogue in
`errors/1komma5-problem-types.yml`.

| Status | Meaning here | Do |
|---|---|---|
| 400 | Bad path/query type or unparseable body | Fix the payload; `configId` must be an integer |
| 401 | Missing/expired bearer token | Re-mint at the Auth0 token endpoint |
| 404 | Not found in the caller's active tenant | Check tenant context before assuming it is gone |
| 409 | Email already exists (user admin only) | Update the existing record instead |
| 503 | PDF renderer down | Retry with backoff |

## Do not

- Do not assume a webhook or event will tell you when the async energy-service lookup
  finishes — 1KOMMA5° publishes no event surface. Poll.
- Do not call anything under `/api/v1/migration-frozen-state/` as part of a customer
  flow; those are one-shot internal data migrations.
- Do not expect rate-limit headers. None are returned; back off on your own schedule.
