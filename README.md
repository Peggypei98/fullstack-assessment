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

## Your Task

1. **Identify and fix bugs** - Review the application thoroughly and fix any issues you find
2. **Document your work** - Create a comprehensive README that includes:
   - What bugs/issues you identified
   - How you fixed each issue
   - Why you chose your approach
   - Any improvements or enhancements you made

We recommend spending no more than 2 hours on this assignment. We are more interested in the quality of your work and your communication than the amount of time you spend or how many bugs you fix!

## Submission

- Fork this repository
- Make your fixes and improvements
- **Replace this README** with your own that clearly documents all changes and your reasoning
- Provide your Stackline contact with a link to a git repository where you have committed your changes

We're looking for clear communication about your problem-solving process as much as the technical fixes themselves.


This README documents: what bugs and issues I identified, how I fixed each one, why I chose that approach, and any other improvements I made.


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
  Long or broken URLs, refresh or shared links failed when only the SKU was in the URL, unnecessary data exposure, and risk of runtime errors when data was missing or malformed.


- **How I found it**  
  Reviewed how the product link and detail page work: the list uses `Link` with a serialized product in the query, and the detail page reads it with `useSearchParams` and `JSON.parse`. The API `GET /api/products/[sku]` was not used, and there was no optional chaining or fallbacks for arrays or optional fields.


- **Root cause**  
  In `app/page.tsx`, the product link passed the whole product in the URL. In `app/product/page.tsx` (old), data came only from that query; there was no call to the existing products-by-SKU API. The UI assumed `featureBullets`, `imageUrls`, and `retailerSku` were always present.


- **Fix**  
  - **List page** (`app/page.tsx`): Change the product link to `href={\`/product/${product.stacklineSku}\`}` so the URL only contains the SKU.


  - **Detail page**: Add `app/product/[sku]/page.tsx`. Use `useParams()` to get `sku`, fetch with `GET /api/products/[sku]`, and add loading and error states. Use defensive rendering: `product.imageUrls ?? []`, `product.featureBullets ?? []`, only show SKU when `product.retailerSku` is present, and use optional chaining where needed. Remove the old `app/product/page.tsx` that read from the query.


  - **Turbopack**: Use a named function and export it at the end (`export default ProductDetailBySkuPage`) so the page is recognized as a valid React component.



- **Why this approach**  
  Using the path and API keeps URLs short and stable, supports refresh and sharing, and avoids sending full product data in the URL. Defensive checks prevent crashes when the API or sample data is incomplete. The separate default export avoids Turbopack’s issue with `export default function Name()` in this dynamic route.



  

#### 3. 

- **Issue**
- **Impact**
- **How I found it**  
- **Root cause**  
- **Fix**  
- **Why I chose this approach**  