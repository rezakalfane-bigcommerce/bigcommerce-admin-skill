# B2B Edition Reference

B2B Edition (companies, buyer-portal users, quotes/RFQs, sales reps) lives on a **completely separate API surface** from the rest of this skill — different host, different auth header. Only use this when the store actually has B2B Edition installed (check: `GET /v3/customers` shows buyer accounts with a non-zero `customer_group_id`, or the control panel sidebar has a "B2B Edition" section).

## Base URL and auth

```
https://api-b2b.bigcommerce.com/api/v3/io/...
```

As of September 30, 2025 the old `authToken`/`auth/backend` token-exchange flow is deprecated. Server-to-server calls now reuse the **same store API token**, just with a different header pair:

```
X-Auth-Token: {ACCESS_TOKEN}      (same token bc_api.py already resolves)
X-Store-Hash: {STORE_HASH}
Content-Type: application/json
```

No exchange call needed. The store's API account needs the **B2B Edition** scope (visible on the API Accounts page under "Scopes" — if it's not listed, B2B calls will 401/404 in confusing ways).

Do **not** try `POST /api/io/auth/backend` (that path exists but is the *customer storefront login* endpoint — wants `email`/`password`/hashes, not a staff exchange) or version-prefixed variants like `/api/v3/auth/backend` (404). There is no working "get a B2B token" endpoint anymore; just send the two headers above directly.

Use `scripts/b2b_api.py` (mirrors `bc_api.py`'s CLI and credential resolution, just pointed at this host/headers):

```bash
python scripts/b2b_api.py GET  /api/v3/io/companies
python scripts/b2b_api.py POST /api/v3/io/rfq --data @quote.json
```

or import `b2b_request(method, path, params, body, env)` from it in a script. Note B2B Edition uses `offset`/`limit` pagination inside `meta.pagination`, not `page` — the rest of this skill's pagination conventions don't apply here.

## Companies, users, sales reps

| Resource | Endpoint |
|---|---|
| List companies | `GET /api/v3/io/companies` — includes `companyId`, address, `bcGroupId` (the linked core customer_group_id), `parentCompany` (for sub-org hierarchies) |
| Get one company | `GET /api/v3/io/companies/{companyId}` |
| List company **buyer** users | `GET /api/v3/io/users?companyId={id}` (or no filter for all) — these are the storefront buyer-portal accounts (Admin/Senior Buyer/Junior Buyer roles), each with a `customerId` linking to the core `/v3/customers` record |
| List Sales Reps | `GET /api/v3/io/sales-staffs` — **only shows users with `Sales staff: Yes` on their store-level System Users record.** This is not creatable via API; it's a genuine BigCommerce control-panel staff login (Settings → B2B Edition → System users and roles) that a human must flag as Sales Rep in the UI. If this list is empty and you need a quote's `userEmail`, ask the user to check that screen rather than guessing — every quote-create call fails with `"Email must belong to an existing B2B Control Panel system user"` otherwise, and this is a distinct concept from Super Admins. |
| Create a Super Admin | `POST /api/v3/io/super-admins` body `{"firstName", "lastName", "email"}` — **camelCase, not snake_case** (snake_case fields are silently ignored and reported as blank). Returns `{"userId", "customerId"}`. Super Admins manage companies from the *buyer portal* side; they are a different role from Sales Reps and do **not** satisfy the quote `userEmail` requirement. |
| Assign a Super Admin to companies | `PUT /api/v3/io/super-admins/{userId}` body `{"companies": [{"companyId": N, "isAssigned": true}, ...]}` — both `companyId` and `isAssigned` are required per entry; omitting either gives a field-specific 400 that names the missing key. |
| Create a company | `POST /api/v3/io/companies` — **snake_case** body: `{"company_name", "company_email", "company_phone", "country", "admin_email", "admin_first_name", "admin_last_name", "admin_phone"}`. All eight are required (send a `{}` body first to see the exact 422 field list for the store's config — some stores may require address fields too). Creating a company always creates its first buyer user as company-role **Admin** on the core customer record tied to `admin_email`. Returns `{"companyId", "customerId", "customerGroupId"}` where `customerId` is the new Admin's core `/v3/customers` id. |
| List company roles | `GET /api/v3/io/companies/roles` → fixed 3-role list, ids are stable per store but confirm rather than hardcode: `Admin` (id 27605), `Senior Buyer` (27606), `Junior Buyer` (27607), each `roleType: 1, roleLevel: 1`. |
| Create a buyer user (Admin/Senior/Junior) on an existing company | `POST /api/v3/io/users` — **camelCase** body (opposite casing from company create): `{"email", "firstName", "lastName", "companyId", "companyRoleId"}`. Sending the snake_case equivalents 422s citing them as null even though *some* snake_case keys were present — this endpoint only accepts camelCase, full stop. Returns `{"userId", "bcId"}` where `bcId` is the core `/v3/customers` id. |
| Link a company as a subsidiary of another (company hierarchy / sub-companies) | `POST /api/v3/io/companies/{subCompanyId}/parent` body `{"parentCompanyId": N}` — plain JSON object (not an array), **camelCase**. This endpoint is undiscoverable by guessing body fields on `PUT /api/v3/io/companies/{id}` (silently ignored, no error, 200 with unrelated field echoed back) — it only exists as this dedicated sub-path. `GET .../parent` 405s (confirming POST is the right method); wrapping the body in an array 500s. Verify the link via `GET /api/v3/io/companies/{parentId}/hierarchy`, which returns a nested `subsidiaries` tree (`GET .../hierarchy` on the child returns its own empty `subsidiaries: []`, not the parent — hierarchy is always rooted at whatever id you query). There's no unlink/re-parent-to-null endpoint found yet. |
| Set/reset a buyer user's or Super Admin's password | Not a B2B-host call — B2B users are core BC customers under the hood. Use the normal core Customers API: `PUT /v3/customers` (see `references/orders-customers.md`) with `[{"id": <customerId from bcId/customerId above>, "authentication": {"new_password": "..."}}]`. Batches fine like any other core customer PUT. |
| List a company's saved addresses | `GET /api/v3/io/addresses?companyId={id}&isShipping=true` or `&isBilling=true` — confirmed live (200, empty array on a company with none). Address book entries are what the buyer portal offers at checkout; fields on each entry are `firstName`/`lastName`/`company`/`addressLine1`/`addressLine2`/`city`/`stateName`/`stateCode`/`countryCode`/`zipCode`/`phoneNumber`/`addressId`/`label`/`apartment` (camelCase, per the `Add Billing Address to Checkout` mapping in the quote-to-invoice Postman collection — no live store had populated addresses to fully confirm every field name). |

## Quotes (RFQ)

`POST /api/v3/io/rfq` creates a real quote. Full worked example (fields confirmed against a live store):

```json
{
  "quoteTitle": "string", "notes": "string", "createdBy": "string",
  "referenceNumber": "string", "expiredAt": "MM/DD/YYYY",
  "legalTerms": "string",
  "subtotal": 0, "discount": 0, "grandTotal": 0,
  "userEmail": "sales-rep@example.com",
  "companyId": 13937529, "storeHash": "...",
  "currency": {"token": "$", "location": "left", "currencyCode": "USD", "decimalToken": ".", "decimalPlaces": 2, "thousandsToken": ",", "currencyExchangeRate": "1.0000000000"},
  "contactInfo": {"name": "...", "email": "...", "companyName": "...", "phoneNumber": "..."},
  "companyInfo": {"companyId": 13937529, "companyName": "...", "companyAddress": "...", "companyCountry": "...", "companyState": "", "companyCity": "...", "companyZipCode": "...", "phoneNumber": "...", "companyEmail": "...", "bcId": 4},
  "productList": [{
    "sku": "...", "basePrice": 0, "discount": 0, "offeredPrice": 0, "quantity": 1,
    "imageUrl": "", "productId": 0, "variantId": 0, "productName": "...",
    "options": [], "notes": "", "purchaseHandled": false,
    "orderQuantityMaximum": 0, "orderQuantityMinimum": 0, "costPrice": 0,
    "inventoryTracking": "none", "inventoryLevel": 0, "type": "physical",
    "isFreeShipping": false, "availability": "available", "isPriceHidden": false,
    "purchasingDisabled": false, "foundInBc": true, "hasInvalidOptions": false,
    "productUrl": "..."
  }],
  "fileList": [], "extraFields": [], "recipients": [], "allowCheckout": false
}
```

Success response: `{"data": {"quoteId": N, "quoteUrl": "..."}}`.

**Fuller optional field set** (sourced from a `POST /api/v3/io/rfq` "copy a quote" call in a working BigCommerce-provided Postman collection, *"BigCommerce B2B Quote to Invoice Flow - Demo"* — not independently re-verified against a live store, but internally consistent with the confirmed fields above so treat as reliable unless a call rejects one): a quote can also carry `shippingAddress`/`billingAddress` objects (`{"city","label","state","address","country","zipCode","lastName","addressId","apartment","firstName","stateCode","countryCode","phoneNumber"}`), `trackingHistory` (array, read-only-looking), `bcOrderId`/`orderId` (strings, populated once ordered — don't set on create), `shippingTotal`/`taxTotal`/`totalAmount`, `discountType`/`discountValue`, `shippingMethod` (object), `storefrontAttachFiles`/`backendAttachFiles` (arrays, alternative/successor to `fileList`), `displayDiscount` (bool), `storeInfo` (`{"storeName","storeAddress","storeCountry","storeLogo","storeUrl"}`), `salesRepInfo` (`{"salesRepName","salesRepEmail","salesRepPhoneNumber"}` — a denormalized copy of the sales rep shown on the quote, separate from the required top-level `userEmail`), and `extraFieldsInfo` (full custom-field *definitions* — `id`, `uuid`, `fieldName`, `fieldType`, `isRequired`, `isUnique`, `visibleToEnduser`, `configType`, `defaultValue`, `labelName`, `listOfValue`/`maximumLength`, `valueConfigs` — distinct from `extraFields`, which is just the `[{"fieldName","fieldValue"}]` pairs of actual values). `quoteLogo` (a URL) is also accepted.

- `variantId` is required per line item even for simple products with no options — every BC product has a `base_variant_id` (visible on the product object, or via `include=variants`); use that.
- `userEmail` must be an existing Sales Rep — see above.
- `channelId`/`channelName`: can only be set **at creation time**, never afterward. `PUT /api/v3/io/rfq/{id}` with a `channelId` rejects with `"Invalid channelId"` even for a channel id that works fine on create (e.g. `1`, a single-storefront store's only channel) — this is a create-vs-update asymmetry in the API, not evidence the id itself is invalid. `GET /api/v3/io/channels` 422s with `"Multi storefront is not enabled"` on single-storefront stores, which is a red herring: it doesn't mean no valid `channelId` exists, just that this particular list endpoint doesn't work there.
- **`createdAt` cannot be set on create or changed via `PUT /api/v3/io/rfq/{id}` afterward** — confirmed empirically: a `PUT` with a different `createdAt` returns 200 but silently no-ops. Every quote is stamped with the real creation time. If you need historical/backdated *B2B* records, that's only possible for orders (below), not quotes — say so explicitly rather than faking it.
- **Once a quote's `status` becomes 4 (Ordered — i.e. after a successful `.../ordered` link call), the entire quote record is permanently immutable.** Not just `status` or `createdAt` — *every* field, including `notes`/`quoteTitle`, rejects with `"Quote has already been ordered"` on any `PUT`, and `DELETE` rejects with `"You can't delete a quote that has been ordered."` There is no API-side undo. If a quote you're about to link to an order might need further edits (including a data fix you discover later), make every fix **before** calling the `.../ordered` endpoint — once linked, recreating means leaving the old one behind forever as an orphaned duplicate (this happened once; the 5 stray quotes had to be left in place with no cleanup path).
- Read one: `GET /api/v3/io/rfq/{id}`. List: `GET /api/v3/io/rfq` (supports the usual `offset`/`limit`). Delete: `DELETE /api/v3/io/rfq/{id}` (hard delete, not an archive — works fine for cleaning up test quotes, **but only before they've been linked to an order**, see above).

### Known unresolved issue: "undefined" in B2B Merchant Dashboard product links

Quote line items sometimes render their product deep-link in the B2B Merchant Dashboard UI as `https://microapps.bigcommerce.com/b2b-merchant-dashboard/undefined/{productUrl}` — the `undefined` segment where some identifier should be. **Tried and ruled out** as the cause:
- Setting `channelId`/`channelName` at creation time (confirmed persisted correctly via `GET`) — link still broken.
- Overriding `productUrl` to a full absolute URL — the API accepts the `PUT` (200) but silently reverts to the catalog-derived relative path on the next `GET`; this field can't be overridden from the quote side at all, so it isn't the lever either.

This looks like a client-side bug in the Merchant Dashboard microapp itself (not something fixable via quote data through this API), but that's not confirmed — nothing here has actually identified the real cause yet. If you hit this again, don't re-try the two things above; if you want to keep digging, the productive next step is inspecting the microapp's actual frontend requests/JS (e.g. via browser devtools) to see what field name and value it expects, rather than guessing more quote payload fields.

## Quote → checkout → order → invoice (full flow)

The following end-to-end pipeline is transcribed from a working BigCommerce-provided Postman collection, *"BigCommerce B2B Quote to Invoice Flow - Demo"*. **Caveat: this collection predates the Sept 2025 auth change** — every B2B-host request in it uses a bare `authToken: {{BE_Token}}` header instead of the current `X-Auth-Token` + `X-Store-Hash` pair documented above. Use the current pair; the endpoint paths and body shapes below are otherwise what to follow. Only the address-list endpoint (see table above) and quote-create (already documented) have been independently re-confirmed against a live store in this skill's own testing — the rest of this section (checkout conversion onward) is transcribed, not re-verified, so treat it as a strong starting point and confirm against a real order before relying on it for anything destructive.

**1. Convert an existing quote straight into a checkout** (an alternative to building a cart from scratch):

```
POST /api/v3/io/rfq/{quoteId}/checkout
{}
```

Returns `{"data": {"cartId": "...", "checkoutUrl": "...", "cartUrl": "..."}}`. This creates a real core cart pre-populated with the quote's `productList`.

**2. Manipulate the cart/checkout with the normal core Storefront Cart/Checkout v3 API** (not B2B-host — back on `api.bigcommerce.com/stores/{hash}/...`, `X-Auth-Token` only, no `X-Store-Hash`):

| Step | Call |
|---|---|
| Read the cart | `GET /v3/checkouts/{cartId}` |
| Add a non-catalog line item | `POST /v3/checkouts/{cartId}/items` body `{"custom_items": [{"sku", "name", "quantity", "list_price", "image_url"}]}` |
| Apply a per-line discount | `POST /v3/checkouts/{cartId}/discounts` body `{"cart": {"line_items": [{"id": "<line_item_id>", "discounted_amount": N}, ...]}}` |
| Attach the cart to a known customer | `PUT /v3/carts/{cartId}` (note: **Carts** API, not Checkouts) body `{"customer_id": N}` |
| Set billing address | `POST /v3/checkouts/{cartId}/billing-address` body `{"first_name","last_name","email","company","address1","address2","city","state_or_province","state_or_province_code","country_code","postal_code","phone","custom_fields":[]}` — snake_case, standard core-address shape (different casing from the B2B-host `GET /api/v3/io/addresses` fields — map camelCase → snake_case between the two) |
| Add a shipping consignment | `POST /v3/checkouts/{cartId}/consignments?include=consignments.available_shipping_options` body: an **array** `[{"address": {...same snake_case shape as billing...}, "line_items": [{"item_id": "<line_item_id>", "quantity": N}, ...]}]`. Line item ids come from `cart.line_items.physical_items[].id` on the cart object. Response includes `consignments[0].available_shipping_options` — pick one. |
| Pick the shipping option | `PUT /v3/checkouts/{cartId}/consignments/{consignmentId}` body `{"shipping_option_id": "<id from available_shipping_options>"}` |
| Convert checkout to a real order | `POST /v3/checkouts/{cartId}/orders` (empty body) → `{"data": {"id": <coreOrderId>, ...}}` |

**3. Attach the resulting order to the B2B side and finish the paper trail:**

```
PUT /api/v3/io/orders/{coreOrderId}          # note: pass the CORE bcOrderId as {id} here — see caveat below
{"bcOrderId": <coreOrderId>, "customerId": <core customer id>, "poNumber": "..."}
```

Response is `{"data": {"id": <b2bOrderId>, ...}}` — save `b2bOrderId`, it's a *different* id used only for the invoice call below. This contradicts the older note that used to live here (that `{id}` must be the B2B-internal id resolved via `GET /api/v3/io/orders?bcOrderId=...` first) — the demo collection instead PUTs straight to the core order id and reads the internal id back out of the response. Both may be valid depending on whether a B2B order record already exists for that `bcOrderId`; if the direct PUT ever 404s, fall back to the resolve-first approach from that older note.

```
PUT /v2/orders/{coreOrderId}
{"status_id": N}   # e.g. 1 = Pending — set whatever core order status fits
```

```
POST /api/v3/io/rfq/{quoteId}/ordered
{"orderId": "{coreOrderId}"}   # as a string — links the quote to the order (see quote immutability warning above: do this LAST)
```

```
POST /api/v3/io/ip/orders/{b2bOrderId}/invoices
{}
```

Note the path prefix is `/api/v3/io/ip/...` (a distinct "invoicing" sub-API), not `/api/v3/io/...` — issues an invoice against the B2B order. Body and response shape are unconfirmed; probe with an empty body first the way the rest of this reference does, and check both `include`/`GET` variants if it 405s.

**Alternate path — creating a backdated/manual order without a quote or checkout:** `POST /v2/orders` (core, see `references/orders-customers.md`) accepts a `company_info` object directly on the order body (`{"companyId","companyName","companyAddress","companyCountry","companyState","companyCity","companyZipCode","phoneNumber","companyEmail","bcId"}`) alongside the normal `customer_id`/`billing_address`/`shipping_addresses`/`products`. This is how the demo collection creates a *historical* B2B-linked order (since, unlike quotes, `POST /v2/orders` does honor `date_created`) — then feeds that order's id through the same "attach to B2B" → status update → (optionally) quote-link steps above.

**Not BigCommerce API calls — skip these:** the demo collection's `New Request`, `Set session vars`, and `Get session vars` items hit `localhost:3000/api/storefront/...` — those are routes on the demo's own Catalyst/Next.js storefront app (session-stored quote/discount state for its UI), not BigCommerce endpoints. Don't try to replicate them against a real store.

## Gotchas worth remembering

- Every field name in this API is inconsistently cased across resources (`firstName` vs `first_name`, `companyId` vs `company_id`) — when a call 400s citing a field as blank/missing, suspect casing first before assuming the field doesn't exist.
- A 405 ("Method X not allowed") on a path you guessed means the resource exists under a different HTTP method — worth probing `GET`/`PUT`/`POST` on the same path rather than assuming 404.
- WebFetch against `docs.bigcommerce.com`/`developer.bigcommerce.com` B2B Edition pages tends to return only the prose overview, not the actual endpoint/schema tables (likely rendered client-side) — don't rely on it for exact request shapes; empirical probing (send a minimal/empty body, read the validation error, iterate) got real answers faster in practice. Confirmed again: `https://docs.bigcommerce.com/developer/api-reference/rest/overview` (the general REST overview, not B2B-specific) and `https://docs.bigcommerce.com/llms.txt` (the doc-site index) both return zero B2B Edition content via WebFetch — general API fundamentals and nav links only, no B2B endpoint list. If someone hands you a docs.bigcommerce.com URL for B2B specifics, don't spend a round-trip re-confirming this — go straight to empirical probing or ask for a Postman collection/HAR capture instead.
- A vendor-provided Postman collection (e.g. a "Quote to Invoice" demo) is a much higher-signal source than the docs site for this API — concrete request bodies with real values beat prose every time here. When given one, read it in full (including test/pre-request scripts — they reveal which response field feeds the next request, e.g. `pm.environment.set('quoteId', pm.response.json().data.quoteId)`) before probing anything yourself. But still flag collection-sourced info as unverified against your actual target store until you've made the call yourself, and watch for staleness (auth scheme, deprecated fields) the same way you'd treat any sample code found online.
