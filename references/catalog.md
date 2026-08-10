# Catalog & Inventory Reference

All paths relative to `https://api.bigcommerce.com/stores/{STORE_HASH}`.
OAuth scope: **Products** (read-only or modify) for catalog; **Products / Inventory** for inventory endpoints.

## Contents
- [Products](#products)
- [Variants, options, modifiers](#variants-options-modifiers)
- [Categories & category trees](#categories--category-trees)
- [Brands](#brands)
- [Images & videos](#images--videos)
- [Custom fields, metafields, bulk pricing](#custom-fields-metafields-bulk-pricing)
- [Inventory (locations API)](#inventory-locations-api)
- [Recipes](#recipes)

## Products

| Action | Endpoint |
|---|---|
| List/search | `GET /v3/catalog/products` |
| Create | `POST /v3/catalog/products` |
| Get one | `GET /v3/catalog/products/{id}` |
| Update one | `PUT /v3/catalog/products/{id}` |
| **Batch update (≤10)** | `PUT /v3/catalog/products` — array of objects each containing `id` |
| Delete one / by filter | `DELETE /v3/catalog/products/{id}` or `?id:in=1,2,3` (**confirm first**) |
| Catalog summary | `GET /v3/catalog/summary` — counts, inventory value, cheapest/priciest |

Required on create: `name`, `type` (`physical`|`digital`), `price`, `weight` (physical only, but include it anyway). Everything else optional. High-value optional fields: `sku`, `categories` (array of IDs), `brand_id`, `inventory_level` + `inventory_tracking` (`none`|`product`|`variant`), `is_visible` (defaults true — set `false` when staging), `sort_order`, `sale_price`, `retail_price`, `custom_url: {"url": "/my-product/", "is_customized": true}`, SEO fields (`page_title`, `meta_description`), `availability` (`available`|`disabled`|`preorder`).

Useful list filters: `sku=`, `name:like=`, `categories:in=`, `brand_id=`, `is_visible=`, `inventory_level:less=`, `date_modified:min=`, `keyword=` (searches name/sku/description), `availability=`. Sub-resources via `include=variants,images,custom_fields,bulk_pricing_rules,primary_image,modifiers,options,videos`. Trim with `include_fields=name,sku,price,inventory_level`.

Products can be created with a full variant matrix in one call by passing an inline `variants` array (each with `sku`, `price`, and `option_values: [{"option_display_name": "Color", "label": "Red"}]`). This is the fastest way to build an optioned product — BigCommerce creates the options automatically.

## Variants, options, modifiers

**Variants** = purchasable SKUs generated from **variant options** (Color, Size). **Modifiers** (engraving text, gift wrap checkbox) never create SKUs. Pick the right one before building.

| Action | Endpoint |
|---|---|
| List variants of product | `GET /v3/catalog/products/{pid}/variants` |
| Create/update/delete variant | `POST/PUT/DELETE /v3/catalog/products/{pid}/variants/{vid}` |
| **All variants store-wide** | `GET /v3/catalog/variants` — supports `sku=`; great for SKU lookups |
| **Batch variant update** | `PUT /v3/catalog/variants` — array with `id`; ideal for price/inventory sweeps |
| Options | `.../products/{pid}/options` and `.../options/{oid}/values` |
| Modifiers | `.../products/{pid}/modifiers` and `.../modifiers/{mid}/values` |

Variant-level overrides: `price`, `sale_price`, `inventory_level` (only honored when product `inventory_tracking=variant`), `purchasing_disabled`, `image_url`, dimensions/weight.

## Categories & category trees

Modern (multi-storefront-aware) endpoints work on **trees**:

- `GET /v3/catalog/trees` — list trees; each channel maps to a tree
- `GET /v3/catalog/trees/{tree_id}/categories` — the tree's categories
- `POST/PUT/DELETE /v3/catalog/trees/categories` — batch create/update/delete across trees

Classic single-tree endpoints still work: `GET/POST /v3/catalog/categories`, `PUT/DELETE /v3/catalog/categories/{id}`. Required on create: `name`, `parent_id` (0 = top level). A category's products are assigned from the **product** side (`categories` array) or via `PUT /v3/catalog/products/{id}` — there is no "add product to category" endpoint under categories.

Product order within a category (merchandising!): `GET/PUT /v3/catalog/categories/{id}/products/sort-order` with `[{"product_id": 123, "sort_order": 0}, ...]`.

## Brands

`GET/POST /v3/catalog/brands`, `PUT/DELETE /v3/catalog/brands/{id}`. Create requires only `name` (must be unique — 409 on duplicate; search with `name:like=` and reuse). Brand image: `POST /v3/catalog/brands/{id}/image` (multipart) or set `image_url` on the brand.

## Images & videos

- `GET/POST /v3/catalog/products/{pid}/images`; `PUT/DELETE .../images/{iid}`
- JSON body with `image_url` (must be publicly fetchable) **or** multipart with `image_file` (use `bc_api.py --file`)
- `is_thumbnail: true` sets the primary image; `sort_order` controls gallery order; `description` is the alt text
- Variant images: `POST /v3/catalog/products/{pid}/variants/{vid}/image`
- Videos: `.../products/{pid}/videos` — YouTube IDs only (`video_id`)

## Custom fields, metafields, bulk pricing

- **Custom fields** (visible on storefront): `.../products/{pid}/custom-fields`, body `{"name": "Material", "value": "Cotton"}`. 250 char value limit.
- **Metafields** (structured, permissioned): `.../products/{pid}/metafields` — required: `namespace`, `key`, `value`, `permission_set`. Batch endpoints exist at `/v3/catalog/products/metafields` (and equivalents for variants, categories, brands, carts, orders, channels, store). Duplicate (namespace,key) → 409; update instead.
- **Bulk pricing rules**: `.../products/{pid}/bulk-pricing-rules`, body `{"quantity_min": 10, "quantity_max": 0, "type": "percent"|"fixed"|"price", "amount": 15}`.

## Inventory (locations API)

For multi-location stores; simple stores can just set `inventory_level` on products/variants.

- `GET /v3/inventory/locations`, `POST/PUT/DELETE /v3/inventory/locations`
- `GET /v3/inventory/locations/{loc}/items` and `GET /v3/inventory/items?sku:in=...`
- **Absolute set**: `PUT /v3/inventory/adjustments/absolute` — `{"items": [{"location_id": 1, "sku": "ABC", "quantity": 40}]}`
- **Relative (+/-)**: `POST /v3/inventory/adjustments/relative` — same shape, `quantity` may be negative. Prefer relative for concurrent-safe decrements.

## Recipes

**Bulk price update from a list of SKUs** — resolve SKUs to variant IDs via `GET /v3/catalog/variants?sku:in=...` (or paged `sku=` lookups), then send chunks to `PUT /v3/catalog/variants`. Check the response's per-item results for partial failures.

**Clone a product** — `GET` it with `include=variants,images,custom_fields,options,modifiers`, strip `id` fields, adjust `name`/`sku` (must be unique), `POST` it back. Images copy by re-sending each `url_zoom` as `image_url`.

**Hide out-of-stock products** — `GET /v3/catalog/products?inventory_level:less=1&inventory_tracking=product&include_fields=name`, confirm the list with the user, then batch `PUT` with `{"id": ..., "is_visible": false}` in chunks of 10.
