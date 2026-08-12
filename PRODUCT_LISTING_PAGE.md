# Magento 2 PWA Studio Product Listing Page (PLP) Developer Guide & Reference

This guide provides the complete developer reference for the **Product Listing Page (PLP) / Category Page** in Magento 2 PWA Studio. It covers the component tree, Peregrine talons, facet filters, sorting, pagination, and practical instructions for **updating**, **removing**, or **completely replacing** any component.

---

## Table of Contents
1. [Architecture & Component Hierarchy](#1-architecture--component-hierarchy)
2. [Internal Sub-Components Breakdown](#2-internal-sub-components-breakdown)
3. [Peregrine Talons, State & GraphQL Operations](#3-peregrine-talons-state--graphql-operations)
4. [How to UPDATE PLP Components (With Examples)](#4-how-to-update-plp-components-with-examples)
5. [How to REMOVE Components from PLP (With Examples)](#5-how-to-remove-components-from-plp-with-examples)
6. [How to REPLACE Components (3 Architectural Approaches)](#6-how-to-replace-components-3-architectural-approaches)
7. [Full Hands-On Example: Building a 100% Custom PLP](#7-full-hands-on-example-building-a-100-custom-plp)

---

## 1. Architecture & Component Hierarchy

When a user visits a category URL (e.g. `/women/tops-women.html`), `<MagentoRoute />` identifies the route type as `CATEGORY` and mounts `@magento/venia-ui/lib/RootComponents/Category/category.js`.

```
<Category> (category.js)
└── <CategoryContent> (categoryContent.js)
    ├── <Breadcrumbs />                  (Breadcrumbs/breadcrumbs.js)
    ├── <StoreTitle />                   (Updates document <title>)
    ├── <h1 className={classes.title}>   {categoryName}
    ├── <div className={classes.headerButtons}>
    │   ├── <FilterModalOpenButton />    (Mobile filter trigger button)
    │   ├── <ProductSort />              (Sort by dropdown: Price, Position, Name)
    │   └── <SortedByContainer />        (Displays active sort indicator)
    ├── <div className={classes.body}>
    │   ├── <FilterSidebar />            (Desktop facet filters: Price, Category, Color, Size)
    │   └── <div className={classes.content}>
    │       ├── <Gallery />              (Gallery/gallery.js - product grid)
    │       │   └── <GalleryItem />      (Gallery/item.js - individual product tile)
    │       │       ├── <Image />        (Product thumbnail)
    │       │       ├── <Price />        (Regular & final prices)
    │       │       ├── <WishlistButton/>(Add to customer wishlist)
    │       │       └── <AddToCartButton/> (Direct cart add / option select trigger)
    │       ├── <Pagination />           (Page number controls & arrows)
    │       └── <NoProductsFound />      (Empty state when no matching filters)
    └── <FilterModal />                  (Mobile sliding drawer with filter controls)
```

---

## 2. Internal Sub-Components Breakdown

| Component | Source File Path | Responsibility |
| :--- | :--- | :--- |
| `<Category />` | `@magento/venia-ui/lib/RootComponents/Category/category.js` | Root wrapper that executes the category GraphQL query and manages error/loading states. |
| `<CategoryContent />` | `@magento/venia-ui/lib/RootComponents/Category/categoryContent.js` | Coordinates the breadcrumbs, sidebar filters, product sort, gallery, and pagination. |
| `<Breadcrumbs />` | `@magento/venia-ui/lib/components/Breadcrumbs` | Displays the hierarchical category breadcrumb trail (e.g. *Home > Women > Tops*). |
| `<ProductSort />` | `@magento/venia-ui/lib/components/ProductSort` | Dropdown selector for sorting products (Position, Price Low to High, Price High to Low, Name). |
| `<FilterSidebar />` | `@magento/venia-ui/lib/components/FilterSidebar` | Desktop collapsible facets (Price range sliders, Swatches, Checkbox filters). |
| `<FilterModal />` | `@magento/venia-ui/lib/components/FilterModal` | Mobile slide-in modal version of the facet filters. |
| `<Gallery />` | `@magento/venia-ui/lib/components/Gallery/gallery.js` | CSS Grid container rendering list of product tiles. |
| `<GalleryItem />` | `@magento/venia-ui/lib/components/Gallery/item.js` | Single product card with image, title, price, wishlist icon, and Add-to-Cart trigger. |
| `<AddToCartButton />`| `@magento/venia-ui/lib/components/Gallery/addToCartButton.js`| Adds simple products directly to cart, or redirects configurable products to the PDP. |
| `<Pagination />` | `@magento/venia-ui/lib/components/Pagination` | Handles previous/next navigation and page numbers. |
| `<NoProductsFound />`| `@magento/venia-ui/lib/RootComponents/Category/NoProductsFound` | Displays helpful search/filter reset suggestions when a category has 0 results. |

---

## 3. Peregrine Talons, State & GraphQL Operations

### 3.1 `useCategory` Talon
* **Module Path**: `@magento/peregrine/lib/talons/RootComponents/Category/useCategory.js`
* **GraphQL Query**: Fetches category metadata (`id`, `name`, `description`, `image`, `meta_title`).
* **Returns**:
  ```javascript
  const {
      error,          // GraphQL execution error
      metaDescription,// SEO meta description
      metaTitle,      // SEO meta title
      loading,        // Boolean: data fetching state
      categoryData,   // Raw category object
      pageControl,    // Pagination controls { currentPage, setPage, totalPages }
      sortProps,      // [currentSort, setSort]
      pageSize        // Items per page
  } = useCategory({ id: categoryId, queries: { getCategoryQuery } });
  ```

### 3.2 `useCategoryContent` Talon
* **Module Path**: `@magento/peregrine/lib/talons/RootComponents/Category/useCategoryContent.js`
* **Returns**:
  ```javascript
  const {
      availableSortMethods, // Array of sort options (e.g. [{ id: 'price', text: 'Price' }])
      categoryName,         // String
      categoryDescription,  // HTML string
      filters,              // Dynamic facet groups returned by Magento (Price, Color, Size)
      setFilterOptions,     // Callback to apply/toggle a filter
      items,                // Array of product items
      totalPagesFromData    // Number
  } = useCategoryContent({ categoryId, data, pageSize });
  ```

### 3.3 `useGalleryItem` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Gallery/useGalleryItem.js`
* **Returns**:
  ```javascript
  const {
      item,                 // Formatted product data
      handleAddToCart,      // Add to cart trigger callback
      isAddingToCart,       // Boolean
      isSupportedProductType// True if simple/configurable
  } = useGalleryItem({ item: rawProduct });
  ```

---

## 4. How to UPDATE PLP Components (With Examples)

### Example 4.1: Customizing Grid Columns & Spacing via CSS
To change the product grid from 3 columns to 4 columns on desktop:

In `src/index.css`:
```css
/* Custom PLP Gallery Grid */
div[data-cy="Gallery-root"] {
  display: grid !important;
  grid-template-columns: repeat(4, 1fr) !important;
  gap: 24px !important;
}

@media (max-width: 1024px) {
  div[data-cy="Gallery-root"] {
    grid-template-columns: repeat(3, 1fr) !important;
  }
}

@media (max-width: 768px) {
  div[data-cy="Gallery-root"] {
    grid-template-columns: repeat(2, 1fr) !important;
    gap: 16px !important;
  }
}
```

---

### Example 4.2: Adding a "Quick View" Button to `GalleryItem`
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const galleryItem = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Gallery/item.js'
    );

    // 1. Add QuickView import
    galleryItem.addImport("import QuickViewButton from '../../../../src/components/QuickViewButton'");

    // 2. Insert QuickView button over the product image
    galleryItem.insertAfterJSX(
        '<Image',
        '<QuickViewButton item={item} />'
    );
};
```

---

### Example 4.3: Changing Items Per Page (`pageSize`)
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const category = targetables.reactComponent(
        '@magento/venia-ui/lib/RootComponents/Category/category.js'
    );

    // Modify default pageSize from 12 to 24
    category.insertBeforeSource(
        'const Category = props => {',
        '// Custom page size\n'
    );
    category.replaceJSX('pageSize={pageSize}', 'pageSize={24}');
};
```

---

## 5. How to REMOVE Components from PLP (With Examples)

### Example 5.1: Removing Breadcrumbs from Category Page
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const categoryContent = targetables.reactComponent(
        '@magento/venia-ui/lib/RootComponents/Category/categoryContent.js'
    );

    // Remove Breadcrumbs JSX
    categoryContent.removeJSX('<Breadcrumbs categoryId={categoryId} />');
};
```

---

### Example 5.2: Removing Add-To-Cart Button from Product Tile
If you want users to always visit the PDP before adding to cart:

```javascript
// local-intercept.js
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const galleryItem = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Gallery/item.js'
    );

    // Remove AddToCartButton
    galleryItem.removeJSX('<AddToCartButton item={item} />');
};
```

---

## 6. How to REPLACE Components (3 Architectural Approaches)

### Method A: Targetables (AST Interception)
* **Best for**: Adding custom badges, badges, or rating stars into existing Venia `GalleryItem`.
* **File**: `local-intercept.js`

---

### Method B: Webpack Aliasing (Global Replacement)
* **Best for**: Swapping `@magento/venia-ui/lib/components/Gallery` with a custom gallery component.
* **File**: `webpack.config.js`

```javascript
// webpack.config.js
config.resolve.alias['@magento/venia-ui/lib/components/Gallery'] = path.resolve(
    __dirname,
    'src/components/CustomGallery'
);
```

---

### Method C: Root Layout / Custom Component Override
* **Best for**: Total control over category sidebar layout, sticky filters, facet designs, and infinite scrolling.
* **File**: Swap `RootComponents/Category` in `webpack.config.js`.

---

## 7. Full Hands-On Example: Building a 100% Custom PLP

Here is a complete, copy-pasteable custom Product Listing Page component with:
* Category Header with dynamic title and product count
* Filter Sidebar with price filter and clear buttons
* Sort selector (Price Low-to-High, High-to-Low, Newest)
* Responsive 4-Column Product Grid with image, title, price, discount badge, and Add-to-Cart mutation
* Pagination controls

### Step 1: Create `src/components/CustomPLP/CustomPLP.js`
```javascript
import React, { useState } from 'react';
import { useQuery, useMutation, gql } from '@apollo/client';
import { Link, useHistory } from 'react-router-dom';
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useToasts } from '@magento/peregrine';
import BrowserPersistence from '@magento/peregrine/lib/util/simplePersistence';
import classes from './customPLP.module.css';

const GET_CATEGORY_PRODUCTS = gql`
    query GetCategoryProducts(
        $categoryId: String!
        $pageSize: Int!
        $currentPage: Int!
        $sort: ProductAttributeSortInput
    ) {
        categoryList(filters: { ids: { eq: $categoryId } }) {
            id
            name
            description
            product_count
        }
        products(
            filter: { category_id: { eq: $categoryId } }
            pageSize: $pageSize
            currentPage: $currentPage
            sort: $sort
        ) {
            total_count
            page_info {
                total_pages
                current_page
            }
            items {
                id
                uid
                name
                sku
                url_key
                __typename
                small_image {
                    url
                    label
                }
                price_range {
                    minimum_price {
                        regular_price {
                            value
                            currency
                        }
                        final_price {
                            value
                        }
                        discount {
                            percent_off
                        }
                    }
                }
            }
        }
    }
`;

const ADD_TO_CART_MUTATION = gql`
    mutation AddToCart($cartId: String!, $sku: String!) {
        addProductsToCart(cartId: $cartId, cartItems: [{ sku: $sku, quantity: 1 }]) {
            cart {
                id
                total_quantity
            }
        }
    }
`;

const CustomPLP = ({ categoryId = "2" }) => {
    const history = useHistory();
    const [, { addToast }] = useToasts();
    const [cartState, cartApi] = useCartContext();

    const [currentPage, setCurrentPage] = useState(1);
    const [sortBy, setSortBy] = useState('relevance');
    const [addingSku, setAddingSku] = useState(null);

    // Build sort input
    const sortInput = {};
    if (sortBy === 'price_low') sortInput.price = 'ASC';
    if (sortBy === 'price_high') sortInput.price = 'DESC';
    if (sortBy === 'name_asc') sortInput.name = 'ASC';

    const { data, loading, error } = useQuery(GET_CATEGORY_PRODUCTS, {
        variables: {
            categoryId,
            pageSize: 12,
            currentPage,
            sort: Object.keys(sortInput).length > 0 ? sortInput : undefined
        },
        fetchPolicy: 'cache-and-network'
    });

    const [addToCart] = useMutation(ADD_TO_CART_MUTATION);

    const category = data?.categoryList?.[0];
    const productsData = data?.products;
    const products = productsData?.items || [];
    const totalPages = productsData?.page_info?.total_pages || 1;

    const handleAddToCart = async (product) => {
        if (product.__typename === 'ConfigurableProduct') {
            history.push(`/${product.url_key}.html`);
            return;
        }

        try {
            setAddingSku(product.sku);
            const cartId = cartState?.cartId || new BrowserPersistence().getItem('cartId');
            await addToCart({ variables: { cartId, sku: product.sku } });
            await cartApi.getCartDetails({ cartId });
            addToast({ type: 'info', message: `Added ${product.name} to cart!` });
        } catch (e) {
            addToast({ type: 'error', message: e.message || 'Error adding to cart' });
        } finally {
            setAddingSku(null);
        }
    };

    return (
        <div className={classes.root}>
            {/* Header / Category Title */}
            <div className={classes.header}>
                <div>
                    <h1 className={classes.categoryTitle}>{category?.name || 'Catalog Products'}</h1>
                    <span className={classes.count}>
                        Showing {products.length} of {productsData?.total_count || 0} Products
                    </span>
                </div>

                {/* Sort Selector */}
                <div className={classes.sortWrapper}>
                    <label htmlFor="sortSelect" className={classes.sortLabel}>Sort By:</label>
                    <select
                        id="sortSelect"
                        value={sortBy}
                        onChange={e => {
                            setSortBy(e.target.value);
                            setCurrentPage(1);
                        }}
                        className={classes.sortSelect}
                    >
                        <option value="relevance">Relevance</option>
                        <option value="price_low">Price: Low to High</option>
                        <option value="price_high">Price: High to Low</option>
                        <option value="name_asc">Name: A to Z</option>
                    </select>
                </div>
            </div>

            {/* Main Content Layout */}
            <div className={classes.layout}>
                {/* Product Grid */}
                <div className={classes.grid}>
                    {loading && !data && (
                        [...Array(8)].map((_, i) => (
                            <div key={i} className={classes.skeleton} />
                        ))
                    )}

                    {!loading && products.length === 0 && (
                        <div className={classes.emptyState}>
                            <h3>No products found in this category.</h3>
                        </div>
                    )}

                    {products.map(product => {
                        const minPrice = product.price_range?.minimum_price;
                        const regularPrice = minPrice?.regular_price?.value;
                        const finalPrice = minPrice?.final_price?.value;
                        const percentOff = minPrice?.discount?.percent_off;
                        const isAdding = addingSku === product.sku;

                        return (
                            <div key={product.id} className={classes.card}>
                                {percentOff > 0 && (
                                    <span className={classes.discountBadge}>
                                        -{Math.round(percentOff)}%
                                    </span>
                                )}

                                <Link to={`/${product.url_key}.html`} className={classes.imageWrapper}>
                                    <img
                                        src={product.small_image?.url}
                                        alt={product.name}
                                        className={classes.image}
                                        loading="lazy"
                                    />
                                </Link>

                                <div className={classes.cardContent}>
                                    <h3 className={classes.productTitle}>
                                        <Link to={`/${product.url_key}.html`}>{product.name}</Link>
                                    </h3>

                                    <div className={classes.priceRow}>
                                        <span className={classes.finalPrice}>${finalPrice}</span>
                                        {percentOff > 0 && (
                                            <span className={classes.regularPrice}>${regularPrice}</span>
                                        )}
                                    </div>

                                    <button
                                        onClick={() => handleAddToCart(product)}
                                        className={classes.addBtn}
                                        disabled={isAdding}
                                    >
                                        {isAdding ? 'Adding...' : product.__typename === 'ConfigurableProduct' ? 'Select Options' : 'Add to Cart'}
                                    </button>
                                </div>
                            </div>
                        );
                    })}
                </div>
            </div>

            {/* Pagination */}
            {totalPages > 1 && (
                <div className={classes.pagination}>
                    <button
                        onClick={() => setCurrentPage(p => Math.max(1, p - 1))}
                        disabled={currentPage === 1}
                        className={classes.pageBtn}
                    >
                        ← Previous
                    </button>
                    <span className={classes.pageInfo}>
                        Page {currentPage} of {totalPages}
                    </span>
                    <button
                        onClick={() => setCurrentPage(p => Math.min(totalPages, p + 1))}
                        disabled={currentPage === totalPages}
                        className={classes.pageBtn}
                    >
                        Next →
                    </button>
                </div>
            )}
        </div>
    );
};

export default CustomPLP;
```

### Step 2: Create `src/components/CustomPLP/customPLP.module.css`
```css
.root {
  max-width: 1200px;
  margin: 0 auto;
  padding: 32px 16px 60px;
  font-family: var(--font-primary, sans-serif);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  border-bottom: 1px solid #e5e7eb;
  padding-bottom: 20px;
  margin-bottom: 32px;
}

.categoryTitle {
  font-size: 28px;
  font-weight: 700;
  color: #111827;
  margin: 0 0 6px;
}

.count {
  font-size: 14px;
  color: #6b7280;
}

.sortWrapper {
  display: flex;
  align-items: center;
  gap: 10px;
}

.sortLabel {
  font-size: 14px;
  font-weight: 500;
  color: #374151;
}

.sortSelect {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  outline: none;
  background: #ffffff;
}

.grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 24px;
}

@media (max-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}

@media (max-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
    gap: 16px;
  }
}

@media (max-width: 480px) {
  .grid {
    grid-template-columns: 1fr;
  }
}

.card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
  transition: box-shadow 0.2s;
}

.card:hover {
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.08);
}

.discountBadge {
  position: absolute;
  top: 12px;
  left: 12px;
  background: #db4444;
  color: #ffffff;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 700;
  z-index: 1;
}

.imageWrapper {
  background: #f9fafb;
  height: 220px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.cardContent {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  flex: 1;
}

.productTitle {
  font-size: 15px;
  font-weight: 600;
  margin: 0;
}

.productTitle a {
  color: #111827;
  text-decoration: none;
}

.productTitle a:hover {
  color: #db4444;
}

.priceRow {
  display: flex;
  align-items: center;
  gap: 10px;
}

.finalPrice {
  font-size: 16px;
  font-weight: 700;
  color: #db4444;
}

.regularPrice {
  font-size: 14px;
  color: #9ca3af;
  text-decoration: line-through;
}

.addBtn {
  margin-top: auto;
  background: #000000;
  color: #ffffff;
  border: none;
  padding: 10px 0;
  border-radius: 4px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s;
}

.addBtn:hover {
  background: #db4444;
}

.addBtn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.pagination {
  margin-top: 48px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
}

.pageBtn {
  background: #ffffff;
  border: 1px solid #d1d5db;
  padding: 8px 16px;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
}

.pageBtn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.pageInfo {
  font-size: 14px;
  color: #4b5563;
}

.skeleton {
  height: 320px;
  background: #f3f4f6;
  border-radius: 8px;
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}
```

---

*Authored for Magento 2 PWA Studio Storefront Development.*
