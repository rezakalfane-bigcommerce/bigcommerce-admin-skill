# Merchandising Reference

Promotions, pricing, segmentation, and channel merchandising.
Scopes: **Marketing** (promotions, coupons, banners, gift certs), **Products** (price lists), **Customers** (segments), **Channel listings** (channels).

## Contents
- [Promotions (v3)](#promotions-v3)
- [Coupon codes for promotions](#coupon-codes-for-promotions)
- [Classic coupons (v2)](#classic-coupons-v2)
- [Price lists](#price-lists)
- [Customer segments](#customer-segments)
- [Banners & gift certificates](#banners--gift-certificates)
- [Channels & listings](#channels--listings)
- [Merchandising levers cheat sheet](#merchandising-levers-cheat-sheet)

## Promotions (v3)

The modern engine — use this over v2 coupons for anything new.

| Action | Endpoint |
|---|---|
| List | `GET /v3/promotions` (filters: `status=`, `redemption_type=`, `name:like=`) |
| Create | `POST /v3/promotions` |
| Update / Delete | `PUT/DELETE /v3/promotions/{id}` |
| Archive / restore | `POST /v3/promotions/archives` / `DELETE /v3/promotions/archives` (bulk, by IDs) |
| Global settings | `GET/PUT /v3/promotions/settings` |

Anatomy of a promotion:

```json
{
  "name": "20% off Sale category",
  "redemption_type": "AUTOMATIC",        // or "COUPON"
  "status": "ENABLED",
  "start_date": "2026-08-15T00:00:00Z",
  "end_date": "2026-08-31T23:59:59Z",    // optional
  "max_uses": 500,                        // optional
  "stop": false,                          // true = don't stack later promos
  "channels": [{ "id": 1 }],              // optional; omit/[] = all channels
  "rules": [{
    "action": {
      "cart_items": {
        "discount": { "percentage_amount": "20" },
        "items": { "categories": [42] }
      }
    },
    "apply_once": false,
    "condition": { "cart": { "minimum_spend": "50" } }   // optional
  }],
  "notifications": []
}
```

Action shapes worth knowing: `cart_items` (item-level discount, target by `categories`/`brands`/`products`/`variants` or `{"and":[...]}` combos), `cart_value` (order subtotal discount), `shipping` (free/discounted shipping, `zone_ids` or `"*"`), `gift_item` (free gift `product_id`/`variant_id`), `fixed_price_set` (bundle pricing). Conditions can nest `and`/`or`/`not` over cart contents, spend, and item quantities. BYGO = condition on qty of X + `gift_item` or `cart_items` action with `apply_once: true`.

`cart_value` worked example — "N% off the whole order once spend crosses a threshold" (a common ask):

```json
{
  "name": "5% Off Orders Over $500",
  "redemption_type": "AUTOMATIC",
  "status": "ENABLED",
  "stop": true,
  "rules": [{
    "action": { "cart_value": { "discount": { "percentage_amount": "5" } } },
    "apply_once": true,
    "condition": { "cart": { "minimum_spend": "500" } }
  }]
}
```

Currency note: monetary values are strings in the store's default (or promotion's specified) currency.

**Channel scoping.** `channels` is a real field on the promotion body: an array of objects keyed by `id` (channel ID) — `[{"id": 1}]`, **not** a bare ID array (`[1]` → 422 "must contain a collection of objects") and **not** `channel_id` (→ 422 "Please provide a id"). Omit the field or send `[]` to apply storewide across every channel (this is also what the control panel calls "all channels"); include one or more `{"id": N}` entries to restrict to specific channels ("selected channels" in the UI, under **Marketing → Promotions → This promotion applies to**). Look up channel IDs with `GET /v3/channels` first — ask the user which named channel(s) they mean, since IDs aren't self-explanatory. `channels` is also usable as a `GET /v3/promotions?channels=` query filter, separately from this create/update field.

## Coupon codes for promotions

For `redemption_type: "COUPON"` promotions:

- `GET /v3/promotions/{id}/codes` — list
- `POST /v3/promotions/{id}/codes` — `{"code": "SUMMER20", "max_uses": 100, "max_uses_per_customer": 1}`
- `POST /v3/promotions/{id}/codes/bulk` — generate many random codes: `{"quantity": 500, "code_prefix": "VIP-", "max_uses": 1}` (single-use code drops)
- `DELETE /v3/promotions/{id}/codes/{code_id}` or bulk delete via `?id:in=`

## Classic coupons (v2)

Legacy but still common; needed when the user says "coupon" and the store predates v3 promotions.

- `GET/POST /v2/coupons`, `PUT/DELETE /v2/coupons/{id}`, `DELETE /v2/coupons` (**deletes ALL — confirm loudly**)
- Create body: `{"name": "...", "type": "per_item_discount"|"per_total_discount"|"percentage_discount"|"shipping_discount"|"free_shipping", "code": "SAVE10", "amount": "10", "applies_to": {"entity": "products"|"categories", "ids": [...]}, "enabled": true}`
- Optional: `min_purchase`, `expires` (RFC 2822 date), `max_uses`, `max_uses_per_customer`

## Price lists

Per-customer-group / per-channel price overrides — the tool for wholesale/VIP pricing.

| Action | Endpoint |
|---|---|
| Lists | `GET/POST /v3/pricelists`, `PUT/DELETE /v3/pricelists/{id}` |
| Records (per variant+currency) | `GET /v3/pricelists/{id}/records` |
| **Bulk upsert records (≤1000)** | `PUT /v3/pricelists/{id}/records` |
| Assignments | `GET/POST/DELETE /v3/pricelists/assignments` |

Record shape: `{"variant_id": 123, "currency": "usd", "price": 8.5, "sale_price": 7.99, "retail_price": 12, "map_price": 8, "bulk_pricing_tiers": [...]}` — can also key by `sku`. Assignments bind a price list to a `customer_group_id` and/or `channel_id`: `POST /v3/pricelists/assignments` with `{"price_list_id": 1, "customer_group_id": 5}`.

## Customer segments

Segments drive targeted promotions (Enterprise feature).

- `GET/POST/PUT/DELETE /v3/segments`
- Shopper profiles: `GET/POST /v3/shopper-profiles` (a profile wraps a `customer_id`)
- Membership: `GET/POST/DELETE /v3/segments/{seg_id}/shopper-profiles`

Flow: ensure a shopper profile exists per customer → add profile IDs to the segment → reference the segment in a promotion rule condition (`"customer": {"segments": {"id": [...]}}`).

## Banners & gift certificates

- **Banners (v2)**: `GET/POST /v2/banners`, `PUT/DELETE /v2/banners/{id}`. Body: `{"name", "content" (HTML), "page": "home_page"|"category_page"|"brand_page"|"search_page", "location": "top"|"bottom", "date_type": "always"|"custom", "visible": "1"}`. `item_id` required for category/brand pages.
- **Gift certificates (v2)**: `GET/POST /v2/gift_certificates`, `PUT/DELETE /v2/gift_certificates/{id}`. Create: `{"code": "XXX-YYY", "amount": "50.00", "balance": "50.00", "to_email", "to_name", "from_email", "from_name", "status": "active"}`. `DELETE /v2/gift_certificates` nukes all — confirm.

## Channels & listings

Multi-storefront / marketplace merchandising.

- `GET/POST /v3/channels`, `PUT /v3/channels/{id}`
- Product↔channel assignment: `GET/PUT/DELETE /v3/catalog/products/channel-assignments` — `[{"product_id": 1, "channel_id": 2}]`
- Per-channel overrides (name/description/state): `GET/POST/PUT /v3/channels/{id}/listings`
- Category assignment per channel happens via category **trees** (see catalog.md)

## Merchandising levers cheat sheet

| Goal | Mechanism |
|---|---|
| Reorder products in a category | `PUT /v3/catalog/categories/{id}/products/sort-order` |
| Feature a product on homepage | `PUT /v3/catalog/products/{id}` with `"is_featured": true` |
| Related products | `PUT /v3/catalog/products/{id}` `"related_products": [-1]` (auto) or explicit ID array |
| Sitewide sale | v3 promotion, `redemption_type: AUTOMATIC`, `cart_items` with `items: {"and": []}` or category targeting |
| VIP pricing | Price list + assignment to customer group |
| Flash sale on schedule | Promotion `start_date`/`end_date` (times are UTC) |
| Single-use influencer codes | `POST /v3/promotions/{id}/codes/bulk` |
