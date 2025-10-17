# Bug Fixes and Changes

## Issues I Fixed

### 1. Subcategory Filtering Bug

**Problem:** When selecting a category, the subcategory dropdown was showing ALL subcategories from all categories instead of only the subcategories belonging to the selected category.

**Root Cause:** The API call to fetch subcategories was missing the required category parameter.

**Impact:** Users saw irrelevant subcategories, creating confusion and poor UX.

#### What I Changed
I added the missing category parameter to the API call:

```typescript
// Before
fetch(`/api/subcategories`)

// After  
fetch(`/api/subcategories?category=${encodeURIComponent(selectedCategory)}`)
```

I used `encodeURIComponent()` to properly handle special characters and spaces in category names. Now when a user selects "Electronics", the API call becomes `GET /api/subcategories?category=Electronics` and only returns relevant subcategories.

---

### 2. Search API Spam Prevention

**Problem:** The search input was making API calls on every keystroke, causing unnecessary server load and poor performance.

**Root Cause:** No debouncing mechanism for search input.

**Impact:** Excessive API calls, potential server overload, and poor user experience.

#### What I Changed
I implemented a custom debounce hook to delay API calls until the user stops typing:

```typescript
// Added custom hook
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value);
  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  return debouncedValue;
}

// Used it in the component
const debouncedSearch = useDebounce(search, 500);
```

Now the search API is only called 500ms after the user stops typing, reducing server load while keeping the UI responsive.

---

### 3. Clear Filters Button State Management

**Problem:** Inconsistent state management when clearing filters - using empty strings vs undefined for category/subcategory reset.

**Root Cause:** Mixed usage of empty strings and undefined for filter state reset.

**Impact:** Potential inconsistencies in filter state and UI behavior.

#### What I Changed
I standardized the clear filters button to use `undefined` instead of empty strings:

```typescript
// Before
setSelectedCategory(undefined);
setSelectedSubCategory(undefined);

// After
setSelectedCategory("");
setSelectedSubCategory("");
```

This ensures consistent state management and proper Select component behavior, where undefined shows the placeholder text.

---

### 4. Security Vulnerability: XSS Attack via URL Parameters

**Problem:** Product data was being passed through URL query parameters using `JSON.stringify()`, creating a potential XSS vulnerability and exposing sensitive data in URLs.

**Root Cause:** The application was serializing entire product objects into URL parameters for navigation to product detail pages.

**Impact:** XSS vulnerability, data exposure in URLs, poor performance, and unshareable URLs.

#### What I Changed
I replaced the unsafe JSON approach with secure SKU-based navigation:

```typescript
// Before - Vulnerable
<Link href={{
  pathname: "/product",
  query: { product: JSON.stringify(product) }
}}>

// After - Secure
<Link href={{
  pathname: "/product", 
  query: { sku: product.stacklineSku }
}}>
```

And updated the product page to fetch data via API instead of parsing JSON:

```typescript
// Before - Unsafe parsing
const productParam = searchParams.get('product');
const parsedProduct = JSON.parse(productParam);

// After - Secure API call
const sku = searchParams.get('sku');
fetch(`/api/products/${sku}`)
```

This eliminates the XSS vulnerability, creates clean URLs like `/product?sku=ABC123`, and ensures fresh data from the API.

---

### 5. Landing Page Missing Products

**Problem:** Only 20 products displayed in the landing page so users had to search exactly what they were looking for.

**Root Cause:** The home page fetched a fixed `limit` of 20 items without passing an `offset`, and there were no UI controls for paging.

**Impact:** Users couldn’t browse beyond the first page of results, harming discoverability and UX. From a business standpoint, this prevented them from being introducted to new products and potentially purchasing.

#### What I Changed
I added a pagination mechanism on the client, wiring it to the existing API which already supports `limit` and `offset`. The feature is available at the top right and bottom of each page.

```typescript
// In app/page.tsx
// Added state
const [currentPage, setCurrentPage] = useState<number>(1);
const [totalPages, setTotalPages] = useState<number>(1);
const LIMIT = 20;

// Include offset in fetch
params.append("limit", String(LIMIT));
params.append("offset", String((currentPage - 1) * LIMIT));

// Use API response to compute total pages
setTotalPages(Math.max(1, Math.ceil(data.total / data.limit)));

// Simple controls
<Button onClick={() => setCurrentPage(Math.max(1, currentPage - 1))}>
  Previous
</Button>
<span>Page {currentPage} of {totalPages}</span>
<Button onClick={() => setCurrentPage(Math.min(totalPages, currentPage + 1))}>
  Next
</Button>
```

I also synced the page number into the URL (`?page=`) to preserve navigation state:

```typescript
function updateURL(nextPage: number) {
  const url = new URL(window.location.href);
  url.searchParams.set("page", String(nextPage));
  window.history.replaceState({}, "", url.toString());
}
```

Now users can browse all products page-by-page, with predictable performance and URL state that can be shared or revisited.

---

### 6. Image Optimization Host Configuration (next/image)

**Problem:** Runtime error: `Invalid src prop (...) on next/image, hostname "images-na.ssl-images-amazon.com" is not configured under images in your next.config.js`.

**Root Cause:** The `images.remotePatterns` configuration was missing the `images-na.ssl-images-amazon.com` host used by some Amazon product images.

**Impact:** Product images from that host failed to render/optimize, breaking visuals and causing runtime errors.

#### What I Changed
I updated the Next.js image configuration to allow the Amazon host:

```typescript
// Before
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'm.media-amazon.com',
    },
  ],
},

// After
images: {
  remotePatterns: [
    {
      protocol: 'https',
      hostname: 'm.media-amazon.com',
    },
    {
      protocol: 'https',
      hostname: 'images-na.ssl-images-amazon.com',
    },
  ],
},
```

I then restarted the dev server to apply the change.

---