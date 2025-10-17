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
setSelectedCategory("");
setSelectedSubCategory("");

// After
setSelectedCategory(undefined);
setSelectedSubCategory(undefined);
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
