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
  In `app/page.tsx` (subcategory `useEffect`), include the selected category in the request:

  ```ts
  fetch(`/api/subcategories?category=${encodeURIComponent(selectedCategory)}`)
  ```

- **Why this approach**  
  The API already supported the `category` query; only the frontend was missing it. Sending `selectedCategory` is a minimal change that matches the API contract. `encodeURIComponent` keeps the value safe in the query string.


#### 2. 

- **Issue**
- **Impact**
- **How I found it**  
- **Root cause**  
- **Fix**  
- **Why I chose this approach**  

#### 3. 

- **Issue**
- **Impact**
- **How I found it**  
- **Root cause**  
- **Fix**  
- **Why I chose this approach**  