# Magento 2 PWA Studio Product Detail Page (PDP) Developer Guide & Reference

This guide provides the complete developer reference for the **Product Detail Page (PDP)** in Magento 2 PWA Studio. It details the component tree, configurable options handling, Peregrine talons, GraphQL queries/mutations, and practical instructions for **updating**, **removing**, or **completely replacing** any part of the PDP.

---

## Table of Contents
1. [Architecture & Component Hierarchy](#1-architecture--component-hierarchy)
2. [Internal Sub-Components Breakdown](#2-internal-sub-components-breakdown)
3. [Peregrine Talons, State & GraphQL Operations](#3-peregrine-talons-state--graphql-operations)
4. [How to UPDATE PDP Components (With Examples)](#4-how-to-update-pdp-components-with-examples)
5. [How to REMOVE Components from PDP (With Examples)](#5-how-to-remove-components-from-pdp-with-examples)
6. [How to REPLACE Components (3 Architectural Approaches)](#6-how-to-replace-components-3-architectural-approaches)
7. [Full Hands-On Example: Building a 100% Custom PDP](#7-full-hands-on-example-building-a-100-custom-pdp)

---

## 1. Architecture & Component Hierarchy

When a user visits a product URL (e.g. `/valeria-one-piece.html`), `<MagentoRoute />` resolves the route type as `PRODUCT` and mounts `@magento/venia-ui/lib/RootComponents/Product/product.js`.

```
<Product> (RootComponents/Product/product.js)
└── <ProductFullDetail> (ProductFullDetail/productFullDetail.js)
    ├── <Breadcrumbs />                  (Breadcrumbs/breadcrumbs.js)
    ├── <Form className={classes.root}>
    │   ├── <section className={classes.imageCarousel}>
    │   │   └── <Carousel />             (ProductImageCarousel/carousel.js)
    │   └── <section className={classes.productDetails}>
    │       ├── <h1 className={classes.productName}> {name}
    │       ├── <p className={classes.productSku}>   SKU: {sku}
    │       ├── <Price />                (Price/price.js - regular, final, discount)
    │       ├── <Options />              (ProductOptions/productOptions.js - swatches)
    │       │   ├── <TileList />         (Size selector pills: XS, S, M, L, XL)
    │       │   └── <SwatchList />       (Color swatches: Black, Blue, Red)
    │       ├── <QuantityStepper />      (QuantityStepper/quantityStepper.js)
    │       ├── <div className={classes.actions}>
    │       │   ├── <Button type="submit"> "Add to Cart"
    │       │   └── <WishlistButton />   (Add to Wishlist heart trigger)
    │       ├── <RichContent />          (Product HTML description)
    │       └── <CustomAttributes />     (Material, Style, Care instructions)
```

---

## 2. Internal Sub-Components Breakdown

| Component | Source File Path | Responsibility |
| :--- | :--- | :--- |
| `<Product />` | `@magento/venia-ui/lib/RootComponents/Product/product.js` | Root wrapper that queries product details and sets OpenGraph / Head meta tags. |
| `<ProductFullDetail />`| `@magento/venia-ui/lib/components/ProductFullDetail` | Coordinates image gallery, buy box, swatches, pricing, and submission form. |
| `<Carousel />` | `@magento/venia-ui/lib/components/ProductImageCarousel` | Multi-image gallery with thumbnail rail, touch swipe, and zoom. |
| `<ProductOptions />` | `@magento/venia-ui/lib/components/ProductOptions` | Renders configurable variant swatches (Color circles, Size pills, Dropdowns). |
| `<QuantityStepper />` | `@magento/venia-ui/lib/components/QuantityStepper` | Number stepper input with increment (+) and decrement (-) buttons. |
| `<Price />` | `@magento/venia-ui/lib/components/Price` | Formats currency, handles range pricing and strike-through discounts. |
| `<WishlistButton />` | `@magento/venia-ui/lib/components/Wishlist/AddToListButton` | Authenticated customer trigger to save product to wishlist. |
| `<CustomAttributes />`| `@magento/venia-ui/lib/components/ProductFullDetail/CustomAttributes`| Renders key-value attributes table (e.g. *Fabric: 100% Cotton, Fit: Regular*). |
| `<RichContent />` | `@magento/venia-ui/lib/components/RichContent` | Renders sanitized HTML product descriptions with Magento widget parsing. |

---

## 3. Peregrine Talons, State & GraphQL Operations

### 3.1 `useProduct` Talon
* **Module Path**: `@magento/peregrine/lib/talons/RootComponents/Product/useProduct.js`
* **Returns**:
  ```javascript
  const {
      error,              // Error object if product query fails
      loading,            // Boolean: query loading state
      product             // Complete product object from GraphQL
  } = useProduct({ mapProduct, queries: { getProductDetailByUrl } });
  ```

### 3.2 `useProductFullDetail` Talon
* **Module Path**: `@magento/peregrine/lib/talons/ProductFullDetail/useProductFullDetail.js`
* **Handles**: Option selection, dynamic media gallery switching when selecting colors, stock validation, and add-to-cart mutation.
* **Returns**:
  ```javascript
  const {
      breadcrumbCategoryId,
      errorMessage,           // Form submission validation error
      handleAddToCart,        // Submit handler triggered by Add to Cart button
      handleSelectionChange,  // Callback fired when user picks a color/size
      isOutOfStock,           // Boolean: true if selected variant is out of stock
      isAddToCartDisabled,    // Boolean: disabled state for submit button
      mediaGalleryEntries,    // Array of image objects for the selected variant
      productDetails,         // { name, sku, price, description }
      customAttributes,       // Formatted custom attributes array
      wishlistButtonProps     // Props to pass to <WishlistButton />
  } = useProductFullDetail({ product });
  ```

### 3.3 Adding Configurable vs Simple Products to Cart
```graphql
# Simple Product:
mutation AddSimpleProduct($cartId: String!, $sku: String!, $quantity: Float!) {
    addProductsToCart(
        cartId: $cartId
        cartItems: [{ sku: $sku, quantity: $quantity }]
    ) {
        cart { total_quantity }
    }
}

# Configurable Product (Requires selected option UIDs):
mutation AddConfigurableProduct(
    $cartId: String!
    $parentSku: String!
    $quantity: Float!
    $selectedOptions: [ID!]!
) {
    addProductsToCart(
        cartId: $cartId
        cartItems: [{
            sku: $parentSku
            quantity: $quantity
            selected_options: $selectedOptions
        }]
    ) {
        cart { total_quantity }
    }
}
```

---

## 4. How to UPDATE PDP Components (With Examples)

### Example 4.1: Customizing PDP 2-Column Grid Layout via CSS
In `src/index.css`:
```css
/* Modern 2-Column PDP Grid */
form.productFullDetail-root-3xU {
  display: grid !important;
  grid-template-columns: 1.2fr 1fr !important;
  gap: 48px !important;
  max-width: 1200px !important;
  margin: 0 auto !important;
  padding: 40px 16px !important;
}

@media (max-width: 860px) {
  form.productFullDetail-root-3xU {
    grid-template-columns: 1fr !important;
    gap: 24px !important;
  }
}
```

---

### Example 4.2: Injecting an "Estimated Delivery & Zipcode Checker"
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const productFullDetail = targetables.reactComponent(
        '@magento/venia-ui/lib/components/ProductFullDetail/productFullDetail.js'
    );

    // 1. Add import
    productFullDetail.addImport(
        "import DeliveryEstimator from '../../../../src/components/DeliveryEstimator'"
    );

    // 2. Insert below Add to Cart button
    productFullDetail.insertAfterJSX(
        '<div className={classes.actions}>',
        '<DeliveryEstimator sku={productDetails.sku} />'
    );
};
```

---

### Example 4.3: Adding a "Buy Now / 1-Click Checkout" Button
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const productFullDetail = targetables.reactComponent(
        '@magento/venia-ui/lib/components/ProductFullDetail/productFullDetail.js'
    );

    productFullDetail.addImport(
        "import BuyNowButton from '../../../../src/components/BuyNowButton'"
    );

    productFullDetail.insertAfterJSX(
        '<Button disabled={isAddToCartDisabled}',
        '<BuyNowButton product={product} />'
    );
};
```

---

## 5. How to REMOVE Components from PDP (With Examples)

### Example 5.1: Removing the SKU Display
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const productFullDetail = targetables.reactComponent(
        '@magento/venia-ui/lib/components/ProductFullDetail/productFullDetail.js'
    );

    // Remove SKU paragraph tag
    productFullDetail.removeJSX('<p className={classes.productSku}>');
};
```

---

### Example 5.2: Removing Custom Attributes or Breadcrumbs
In `local-intercept.js`:
```javascript
// Remove Custom Attributes table
productFullDetail.removeJSX('<CustomAttributes classes={{ list: classes.customAttributesList }} customAttributes={customAttributes} />');

// Remove Breadcrumbs
productFullDetail.removeJSX('{breadcrumbs}');
```

---

## 6. How to REPLACE Components (3 Architectural Approaches)

### Method A: Targetables (AST Interception)
* **Best for**: Small tweaks, injecting size charts, badges, and trust widgets.
* **File**: `local-intercept.js`

---

### Method B: Webpack Aliasing (Global Replacement)
* **Best for**: Swapping `@magento/venia-ui/lib/components/ProductFullDetail` or `@magento/venia-ui/lib/RootComponents/Product`.
* **File**: `webpack.config.js`

```javascript
// webpack.config.js
config.resolve.alias['@magento/venia-ui/lib/components/ProductFullDetail'] = path.resolve(
    __dirname,
    'src/components/CustomPDP/CustomProductFullDetail'
);
```

---

### Method C: Custom PDP Component with Direct GraphQL
* **Best for**: Total freedom over image zooming, swatch animations, sticky buy box, and related product carousels.

---

## 7. Full Hands-On Example: Building a 100% Custom PDP

Here is a complete, copy-pasteable custom Product Detail Page component featuring:
* Interactive multi-image gallery with thumbnail switcher
* Dynamic price calculation & discount badges
* Configurable variant swatches (Color & Size selectors)
* Custom Quantity Stepper (+ / -)
* Functional Add-to-Cart mutation with Peregrine Toast feedback
* Delivery & Guarantee accordion badges

### Step 1: Create `src/components/CustomPDP/CustomPDP.js`
```javascript
import React, { useState } from 'react';
import { useQuery, useMutation, gql } from '@apollo/client';
import { useParams } from 'react-router-dom';
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useToasts } from '@magento/peregrine';
import BrowserPersistence from '@magento/peregrine/lib/util/simplePersistence';
import classes from './customPDP.module.css';

const GET_PRODUCT_BY_URL = gql`
    query GetProductDetail($urlKey: String!) {
        products(filter: { url_key: { eq: $urlKey } }) {
            items {
                id
                uid
                name
                sku
                url_key
                stock_status
                __typename
                description {
                    html
                }
                media_gallery {
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
                            amount_off
                            percent_off
                        }
                    }
                }
                ... on ConfigurableProduct {
                    configurable_options {
                        id
                        attribute_code
                        label
                        values {
                            value_index
                            label
                            uid
                        }
                    }
                }
            }
        }
    }
`;

const ADD_TO_CART_MUTATION = gql`
    mutation AddItemToCart($cartId: String!, $cartItems: [CartItemInput!]!) {
        addProductsToCart(cartId: $cartId, cartItems: $cartItems) {
            cart {
                id
                total_quantity
            }
        }
    }
`;

const CustomPDP = () => {
    const { urlKey = '' } = useParams();
    const [, { addToast }] = useToasts();
    const [cartState, cartApi] = useCartContext();

    const [selectedImageIndex, setSelectedImageIndex] = useState(0);
    const [quantity, setQuantity] = useState(1);
    const [selectedOptions, setSelectedOptions] = useState({});
    const [isAdding, setIsAdding] = useState(false);

    const { data, loading, error } = useQuery(GET_PRODUCT_BY_URL, {
        variables: { urlKey: urlKey.replace('.html', '') },
        fetchPolicy: 'cache-and-network'
    });

    const [addToCart] = useMutation(ADD_TO_CART_MUTATION);

    const product = data?.products?.items?.[0];

    if (loading && !data) return <div className={classes.skeleton} />;
    if (error || !product) {
        return (
            <div className={classes.errorWrapper}>
                <h2>Product not found</h2>
            </div>
        );
    }

    const {
        name,
        sku,
        stock_status,
        description,
        media_gallery = [],
        price_range,
        configurable_options = [],
        __typename
    } = product;

    const minPrice = price_range?.minimum_price;
    const regularPrice = minPrice?.regular_price?.value;
    const finalPrice = minPrice?.final_price?.value;
    const percentOff = minPrice?.discount?.percent_off;
    const isConfigurable = __typename === 'ConfigurableProduct';
    const isOutOfStock = stock_status === 'OUT_OF_STOCK';

    const handleOptionSelect = (attributeCode, uid) => {
        setSelectedOptions(prev => ({
            ...prev,
            [attributeCode]: uid
        }));
    };

    const handleAddToCart = async () => {
        // Validate configurable options
        if (isConfigurable) {
            const requiredCount = configurable_options.length;
            const selectedCount = Object.keys(selectedOptions).length;
            if (selectedCount < requiredCount) {
                addToast({
                    type: 'warning',
                    message: 'Please select all product options before adding to cart.'
                });
                return;
            }
        }

        try {
            setIsAdding(true);
            const storage = new BrowserPersistence();
            const cartId = cartState?.cartId || storage.getItem('cartId');

            const cartItemInput = {
                sku,
                quantity: parseFloat(quantity)
            };

            if (isConfigurable) {
                cartItemInput.selected_options = Object.values(selectedOptions);
            }

            await addToCart({
                variables: {
                    cartId,
                    cartItems: [cartItemInput]
                }
            });

            await cartApi.getCartDetails({ cartId });

            addToast({
                type: 'info',
                message: `Added ${name} (${quantity}) to your shopping cart!`,
                timeout: 5000
            });
        } catch (e) {
            addToast({
                type: 'error',
                message: e.message || 'Failed to add item to cart.',
                timeout: 5000
            });
        } finally {
            setIsAdding(false);
        }
    };

    return (
        <div className={classes.root}>
            {/* Left: Image Gallery */}
            <div className={classes.gallerySection}>
                {/* Thumbnails */}
                <div className={classes.thumbnailRail}>
                    {media_gallery.map((img, idx) => (
                        <button
                            key={idx}
                            onClick={() => setSelectedImageIndex(idx)}
                            className={`${classes.thumbBtn} ${idx === selectedImageIndex ? classes.activeThumb : ''}`}
                        >
                            <img src={img.url} alt={img.label || name} />
                        </button>
                    ))}
                </div>

                {/* Main Preview Image */}
                <div className={classes.mainImageWrapper}>
                    <img
                        src={media_gallery[selectedImageIndex]?.url || '/placeholder.png'}
                        alt={name}
                        className={classes.mainImage}
                    />
                </div>
            </div>

            {/* Right: Buy Box / Product Information */}
            <div className={classes.infoSection}>
                <h1 className={classes.productTitle}>{name}</h1>
                <div className={classes.skuStockRow}>
                    <span className={classes.sku}>SKU: {sku}</span>
                    <span className={isOutOfStock ? classes.outOfStock : classes.inStock}>
                        {isOutOfStock ? 'Out of Stock' : 'In Stock'}
                    </span>
                </div>

                {/* Pricing */}
                <div className={classes.priceRow}>
                    <span className={classes.finalPrice}>${finalPrice}</span>
                    {percentOff > 0 && (
                        <>
                            <span className={classes.regularPrice}>${regularPrice}</span>
                            <span className={classes.discountPill}>-{Math.round(percentOff)}% OFF</span>
                        </>
                    )}
                </div>

                {/* Configurable Option Swatches */}
                {isConfigurable && (
                    <div className={classes.optionsWrapper}>
                        {configurable_options.map(option => (
                            <div key={option.id} className={classes.optionGroup}>
                                <span className={classes.optionLabel}>{option.label}:</span>
                                <div className={classes.swatchRow}>
                                    {option.values.map(val => {
                                        const isSelected = selectedOptions[option.attribute_code] === val.uid;
                                        return (
                                            <button
                                                key={val.uid}
                                                type="button"
                                                onClick={() => handleOptionSelect(option.attribute_code, val.uid)}
                                                className={`${classes.swatchPill} ${isSelected ? classes.selectedSwatch : ''}`}
                                            >
                                                {val.label}
                                            </button>
                                        );
                                    })}
                                </div>
                            </div>
                        ))}
                    </div>
                )}

                {/* Quantity & Add to Cart Controls */}
                <div className={classes.actionsRow}>
                    <div className={classes.stepper}>
                        <button
                            type="button"
                            onClick={() => setQuantity(q => Math.max(1, q - 1))}
                            className={classes.stepBtn}
                            disabled={quantity <= 1}
                        >
                            -
                        </button>
                        <span className={classes.qtyValue}>{quantity}</span>
                        <button
                            type="button"
                            onClick={() => setQuantity(q => q + 1)}
                            className={classes.stepBtn}
                        >
                            +
                        </button>
                    </div>

                    <button
                        onClick={handleAddToCart}
                        className={classes.addToCartButton}
                        disabled={isAdding || isOutOfStock}
                    >
                        {isAdding ? 'Adding...' : isOutOfStock ? 'Out of Stock' : 'Buy Now'}
                    </button>
                </div>

                {/* Trust & Delivery Badges */}
                <div className={classes.trustBox}>
                    <div className={classes.trustItem}>
                        <span className={classes.trustIcon}>🚚</span>
                        <div>
                            <strong>Free Delivery</strong>
                            <p>Enter your postal code for Delivery Availability</p>
                        </div>
                    </div>
                    <div className={classes.trustItem}>
                        <span className={classes.trustIcon}>🔄</span>
                        <div>
                            <strong>Return Delivery</strong>
                            <p>Free 30 Days Delivery Returns.</p>
                        </div>
                    </div>
                </div>

                {/* Product Description */}
                {description?.html && (
                    <div className={classes.descriptionBox}>
                        <h3>Product Details</h3>
                        <div dangerouslySetInnerHTML={{ __html: description.html }} />
                    </div>
                )}
            </div>
        </div>
    );
};

export default CustomPDP;
```

### Step 2: Create `src/components/CustomPDP/customPDP.module.css`
```css
.root {
  max-width: 1200px;
  margin: 0 auto;
  padding: 40px 16px 80px;
  display: grid;
  grid-template-columns: 1.2fr 1fr;
  gap: 48px;
  font-family: var(--font-primary, sans-serif);
}

@media (max-width: 860px) {
  .root {
    grid-template-columns: 1fr;
    gap: 32px;
  }
}

/* 1. Gallery Section */
.gallerySection {
  display: flex;
  gap: 16px;
}

@media (max-width: 600px) {
  .gallerySection {
    flex-direction: column-reverse;
  }
}

.thumbnailRail {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

@media (max-width: 600px) {
  .thumbnailRail {
    flex-direction: row;
    overflow-x: auto;
  }
}

.thumbBtn {
  width: 70px;
  height: 70px;
  border: 1px solid #e5e7eb;
  border-radius: 6px;
  background: #f9fafb;
  padding: 6px;
  cursor: pointer;
  outline: none;
  transition: border-color 0.2s;
}

.thumbBtn img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

.activeThumb {
  border: 2px solid #db4444;
}

.mainImageWrapper {
  flex: 1;
  background: #f9fafb;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 32px;
  min-height: 440px;
}

.mainImage {
  max-width: 100%;
  max-height: 440px;
  object-fit: contain;
}

/* 2. Info Section */
.infoSection {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.productTitle {
  font-size: 26px;
  font-weight: 700;
  color: #111827;
  margin: 0;
  line-height: 1.3;
}

.skuStockRow {
  display: flex;
  gap: 16px;
  font-size: 14px;
}

.sku {
  color: #6b7280;
}

.inStock {
  color: #10b981;
  font-weight: 600;
}

.outOfStock {
  color: #ef4444;
  font-weight: 600;
}

.priceRow {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 8px 0;
}

.finalPrice {
  font-size: 24px;
  font-weight: 700;
  color: #111827;
}

.regularPrice {
  font-size: 18px;
  color: #9ca3af;
  text-decoration: line-through;
}

.discountPill {
  background: #fee2e2;
  color: #db4444;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 700;
}

/* Options */
.optionsWrapper {
  display: flex;
  flex-direction: column;
  gap: 16px;
  border-top: 1px solid #e5e7eb;
  padding-top: 16px;
}

.optionGroup {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.optionLabel {
  font-size: 14px;
  font-weight: 600;
  color: #374151;
}

.swatchRow {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}

.swatchPill {
  border: 1px solid #d1d5db;
  background: #ffffff;
  padding: 6px 14px;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.swatchPill:hover {
  border-color: #db4444;
}

.selectedSwatch {
  background: #db4444;
  color: #ffffff;
  border-color: #db4444;
}

/* Controls */
.actionsRow {
  display: flex;
  gap: 16px;
  margin-top: 8px;
}

.stepper {
  display: flex;
  align-items: center;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.stepBtn {
  background: transparent;
  border: none;
  width: 38px;
  height: 44px;
  font-size: 18px;
  cursor: pointer;
}

.stepBtn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.qtyValue {
  padding: 0 12px;
  font-weight: 600;
}

.addToCartButton {
  flex: 1;
  background: #db4444;
  color: #ffffff;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s;
  height: 44px;
}

.addToCartButton:hover {
  background: #b91c1c;
}

.addToCartButton:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

/* Trust Box */
.trustBox {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin-top: 16px;
}

.trustItem {
  display: flex;
  gap: 16px;
  padding: 16px;
}

.trustItem:not(:last-child) {
  border-bottom: 1px solid #e5e7eb;
}

.trustIcon {
  font-size: 24px;
}

.trustItem strong {
  font-size: 14px;
  color: #111827;
}

.trustItem p {
  font-size: 12px;
  color: #6b7280;
  margin: 4px 0 0;
}

.descriptionBox {
  margin-top: 24px;
  border-top: 1px solid #e5e7eb;
  padding-top: 16px;
  color: #4b5563;
  line-height: 1.6;
}

.skeleton {
  max-width: 1200px;
  margin: 40px auto;
  height: 500px;
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
