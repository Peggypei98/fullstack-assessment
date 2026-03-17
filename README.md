# Stackline Full Stack Assignment

## Overview

This is a sample eCommerce website that includes:

- Product List Page
- Search Results Page
- Product Detail Page

The application contains various bugs including UX issues, design problems, functionality bugs, and potential security vulnerabilities.

## Getting Started

```bash
yarn install
yarn dev
```

## Scope

Given the 2-hour time constraint, I focused on identifying and fixing high-impact issues related to:

- Data correctness
- API behavior
- Core user flows

## What I Identified

### Critical bugs

#### 1. Subcategory filter was not scoped to the selected category

- **Issue**  
Selecting a category updated the product results, but the subcategory dropdown still showed subcategories from unrelated categories.
- **Impact**  
Inconsistent filtering: users could pick subcategories that did not belong to the active category, making the UI feel unreliable and sometimes showing confusing or empty results.
- **How I found it**  
Selected a category on the home page and checked network requests in DevTools. The products request correctly sent the selected category, but the subcategory request was sent as `/api/subcategories` with no `category` query parameter.
- **Root cause**  
In `app/page.tsx`, the subcategory fetch called `/api/subcategories` without the selected category. The API in `app/api/subcategories/route.ts` already accepts an optional `category` and filters by it; the frontend never sent it, so the API returned all subcategories.
- **Fix**  
In `app/page.tsx` (subcategory `useEffect`), include the selected category in the request (line 55-56):
  ```ts
  fetch(`/api/subcategories?category=${encodeURIComponent(selectedCategory)}`)
  ```
- **Why this approach**  
The API already supported the `category` query; only the frontend was missing it. Sending `selectedCategory` is a minimal change that matches the API contract. `encodeURIComponent` keeps the value safe in the query string.

#### 2. Product detail page: data passed via URL and no defensive rendering

- **Issue**  
The list page sent the full product object in the URL with `query: { product: JSON.stringify(product) }`, and the detail page parsed it from the query. The detail page also used `product.featureBullets`, `product.imageUrls`, and `product.retailerSku` without checking if they existed.
- **Impact**  
  - Long and fragile URLs
  - Broken refresh and sharing behavior
  - Unnecessary data exposure
  - Risk of runtime errors when fields are missing
- **How I found it**  
Reviewed how the product link and detail page work: the list uses `Link` with a serialized product in the query, and the detail page reads it with `useSearchParams` and `JSON.parse`. The API `GET /api/products/[sku]` was not used, and there was no optional chaining or fallbacks for arrays or optional fields.
- **Root cause**  
In `app/page.tsx`, the product link passed the whole product in the URL. In `app/product/page.tsx` (old), data came only from that query; there was no call to the existing products-by-SKU API. The UI assumed `featureBullets`, `imageUrls`, and `retailerSku` were always present.
- **Fix**  
  - **List page** (`app/page.tsx`): Change the product link to `href={\`/product/${product.stacklineSku}`}` so the URL only contains the SKU.
  - **Detail page**: Add `app/product/[sku]/page.tsx`. Use `useParams()` to get `sku`, fetch with `GET /api/products/[sku]`, and add loading and error states. Use defensive rendering: `product.imageUrls ?? []`, `product.featureBullets ?? []`, only show SKU when `product.retailerSku` is present, and use optional chaining where needed. Remove the old `app/product/page.tsx` that read from the query.
  - **Turbopack**: Use a named function and export it at the end (`export default ProductDetailBySkuPage`) so the page is recognized as a valid React component.
- **Why this approach**  
Using the path and API keeps URLs short and stable, supports refresh and sharing, and avoids sending full product data in the URL. Defensive checks prevent crashes when the API or sample data is incomplete. The separate default export avoids Turbopack’s issue with `export default function Name()` in this dynamic route.

#### 3. API query validation missing for limit and offset

- **Issue**  
In `app/api/products/route.ts`, `limit` and `offset` were parsed with `parseInt` and passed through without checking for `NaN`, negative values, or unreasonable numbers. Invalid query strings (e.g. `?limit=abc` or `?offset=-1`) could lead to unstable API behavior.
- **Impact**  
Requests with invalid or negative `limit`/`offset` could produce unexpected results or rely on downstream `||` fallbacks. The API could return `NaN` in the response or use invalid values in `slice(offset, offset + limit)`, making behavior unreliable in edge cases.
- **How I found it**  
Reviewed the products API route and saw that `limit` and `offset` were read from query params with `parseInt` and used directly. Checked what happens when passing `?limit=abc`, `?limit=-1`, or `?offset=-1` and confirmed there was no validation.
- **Root cause**  
In `app/api/products/route.ts`, the code used `searchParams.get('limit') ? parseInt(...) : 20` (and similarly for `offset`). When a value was present but not a valid number, `parseInt` returned `NaN` and that value was passed into the service. Negative values were also passed through unchanged.
- **Fix**  
In `app/api/products/route.ts`, parse `limit` and `offset` then validate before use (lines 7–14):
  - Parse with `parseInt(..., 10)` and use `Number.isNaN()` to detect invalid numbers.
  - For `limit`: if NaN or < 1, use default 20; optionally cap with `Math.min(parsedLimit, 100)`.
  - For `offset`: if NaN or < 0, use default 0.
  - Use the validated `limit` and `offset` in the filters object so the response and `getAll` always receive valid numbers.
- **Why this approach**  
Validating at the API boundary keeps the service simple and guarantees the response never contains `NaN`. Using defaults for invalid input avoids breaking callers while keeping behavior predictable. An optional limit cap prevents overly large responses.

#### 4. Limit/offset falsy handling in product service

- **Issue**  
In `lib/products.ts`, `getAll` used `filters?.offset || 0` and `filters?.limit || filtered.length`. Because `0` is falsy in JavaScript, passing `limit: 0` or `offset: 0` was treated as "not provided" and got the default instead, so `limit: 0` returned all products instead of none.
- **Impact**  
Callers that explicitly pass `limit: 0` (e.g. "no results") would receive the full list. Any code passing `offset: 0` relied on the accident that `0 || 0` is still 0; the intent was unclear and fragile for pagination.
- **How I found it**  
Reviewed `lib/products.ts` for how limit/offset are applied; noticed the use of `||` for defaulting and considered the case where the value is explicitly `0`.
- **Root cause**  
`||` treats `0` as falsy, so `filters?.limit || filtered.length` becomes `filtered.length` when `limit` is `0`. The same pattern for `offset` is misleading even when the result happens to be correct.
- **Fix**  
In `lib/products.ts` (around the two lines that set `offset` and `limit`), use nullish coalescing so only `undefined`/`null` get the default:

```ts
  const offset = filters?.offset ?? 0;
  const limit = filters?.limit ?? filtered.length;
```

## Additional Observations (Deprioritized due to time constraints)

Due to time constraints, I focused on addressing issues that directly impact data correctness and core user flows. 

In addition, I identified several other areas for improvement that were not implemented:

### Reliability  
Potential approach: wrap fetches in try/catch, check res.ok and show error UI; move product detail to SKU-based route + API (already done for the main flow)

- Missing error handling in data fetching logic (e.g., no handling for failed requests or non-200 responses)
- Product detail page overly dependent on query parsing, making it fragile when query data is missing or malformed



### UX / Accessibility
Potential approach: add loading skeletons and error/empty states; avoid nesting <Button> inside <Link> or make the card a single focusable link.

- Incomplete handling of loading, error, and empty states
- Nested interactive elements (button inside a Link), which may cause accessibility and usability issues



### Code Quality
Potential approach: centralize Product type in a shared types file; replace placeholder metadata in layout.

- Inconsistent `Product` type definitions across components, which may lead to maintenance issues or runtime errors
- Application metadata still uses default placeholder values



### Tooling & Implementation Details
Potential approach: use next lint or eslint . in package.json; add explicit dimensions or aspect-ratio for thumbnail containers; add priority to above-the-fold product image where appropriate.

- The `lint` script in `package.json` is defined as `"eslint"` without specifying a target, which may result in incomplete linting. Using `next lint` or `eslint .` would be more reliable
- In `app/product/page.tsx`, thumbnail images use `<Image fill />` without a clearly defined width on the parent container, which may cause layout inconsistencies across screen sizes
- Minor CSS and layout inconsistencies (e.g., spacing, responsive behavior and the dark theme is not functioning) were observed but deprioritized
- Largest Contentful Paint (LCP) image is missing the `priority` attribute
  - Observed a Next.js console warning indicating that the LCP image is not marked with `priority`
  - Adding `priority` to the main product image would improve initial load performance and user-perceived speed

