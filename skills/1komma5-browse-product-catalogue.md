---
name: Browse the 1KOMMA5° product catalogue and pricebook
description: List and filter the 1KOMMA5° hardware catalogue — modules, inverters,
  batteries, heat pumps, wallboxes — by category, brand and country, and read the
  tenant-scoped pricebook behind a quote.
api: openapi/1komma5-offer-tool-openapi-original.json
generated: '2026-08-02'
method: generated
source: openapi/1komma5-offer-tool-openapi-original.json
operations:
- ProductGetAllController_handle_v1
- GetProductController_get_v1
- GetAdminProductsController_getAdminProducts_v1
- GetAdminProductController_getProduct_v1
- GetAdminCategoriesController_getCategories_v1
- GetAdminCategoriesController_getBrands_v1
- GetAdminPricebookController_getAdminPricebook_v1
- GetCountriesController_getCountries_v1
- MegasearchController_search_v1
---

# Browse the 1KOMMA5° product catalogue and pricebook

Use this to answer "what can 1KOMMA5° quote, in which country, at what price".

## Before you start

- Base URL: `https://api.offer.1komma5grad.com`; Auth0 bearer JWT in `authorization`.
- Two parallel surfaces exist and they are not the same thing:
  - the **sales** catalogue — `/api/v1/product`, what a sales user may sell;
  - the **admin** catalogue — `/api/v1/admin/products`, the full editable master with
    pricebook, categories and brands.
  Prefer the sales surface for read-only work.

## Steps

1. **Establish country context.** `GetCountriesController_getCountries_v1`
   (`GET /api/v1/admin/countries`). Country drives locale, currency and market rules, and
   most catalogue endpoints take a `countryId` filter.
2. **List sellable products.** `ProductGetAllController_handle_v1`
   (`GET /api/v1/product`). Filters observed in the spec: `countryId`, `categoryId`,
   `brandId`, `public`, `canbesold`, `type`, plus `search`.
3. **Read one product.** `GetProductController_get_v1` (`GET /api/v1/product/{id}`).
   Returns the `ProductDTO` with `PriceDTO` and `MediaDTO`. A `404` is
   "Product not found".
4. **Taxonomy.** `GetAdminCategoriesController_getCategories_v1`
   (`GET /api/v1/admin/categories`) and `GetAdminCategoriesController_getBrands_v1`
   (`GET /api/v1/admin/brands`) resolve `categoryId` / `brandId`.
5. **Pricebook.** `GetAdminPricebookController_getAdminPricebook_v1`
   (`GET /api/v1/admin/pricebook`) — tenant and country scoped. Two prices for the same
   product in different tenants is expected, not a bug.
6. **Admin master data.** `GetAdminProductsController_getAdminProducts_v1` supports
   `page`, `limit`, `search`, `sortBy` and `sortOrder`; read one with
   `GetAdminProductController_getProduct_v1`.
7. **Search across everything.** `MegasearchController_search_v1`
   (`GET /api/v1/admin/megasearch`) when you do not know which entity you are looking for.

## Paging and sorting

Page-number paging only — `page` + `limit`. There is no cursor and no offset. **Not every
list endpoint accepts them**: `ProductGetAllController_handle_v1` is filter-driven, while
the admin list operations take `page`/`limit`/`sortBy`/`sortOrder`. Check the operation
before assuming a list is pageable. Response paging fields are undocumented; look for the
shared `PaginationMetaDto` schema, which is the most-referenced schema in the spec.

## Watch out

- Two naming conventions for the same concepts coexist in the schemas (`CategoryDTO` /
  `BrandDTO` / `PriceDTO` alongside `CategoryDto` / `BrandDto`). Match on the schema the
  specific operation references, not on the name you saw elsewhere.
- The admin endpoints are role-gated; check `GetAssignableRolesController_getAssignableRoles_v1`
  and the `AccessLevel` schema if a call is refused.
