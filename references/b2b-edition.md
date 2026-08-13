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

## Linking a quote to an order

Orders themselves are created via the normal core `POST /v2/orders` (see `references/orders-customers.md` — and note that endpoint *does* honor a backdated `date_created`, unlike quotes). To associate a resulting order with the quote it came from:

```
POST /api/v3/io/rfq/{quoteId}/ordered
{"orderId": "123"}   // the core BigCommerce order id, as a string
```

This is what makes the quote show "Ordered" / links to the order in the B2B buyer portal and control panel.

There is also `PUT /api/v3/io/orders/{id}` for attaching a `poNumber` etc. to the *B2B-side* order record, but note `{id}` here is the B2B Edition's own internal order id, **not** the core `bcOrderId`. Resolve it first: `GET /api/v3/io/orders?bcOrderId={coreOrderId}` → `data[0].id`. (Even with the correct internal id this endpoint still returned 404 in testing on a fresh store — treat it as unreliable/best-effort, not required for a working quote↔order link.)

## Gotchas worth remembering

- Every field name in this API is inconsistently cased across resources (`firstName` vs `first_name`, `companyId` vs `company_id`) — when a call 400s citing a field as blank/missing, suspect casing first before assuming the field doesn't exist.
- A 405 ("Method X not allowed") on a path you guessed means the resource exists under a different HTTP method — worth probing `GET`/`PUT`/`POST` on the same path rather than assuming 404.
- WebFetch against `docs.bigcommerce.com`/`developer.bigcommerce.com` B2B Edition pages tends to return only the prose overview, not the actual endpoint/schema tables (likely rendered client-side) — don't rely on it for exact request shapes; empirical probing (send a minimal/empty body, read the validation error, iterate) got real answers faster in practice.
