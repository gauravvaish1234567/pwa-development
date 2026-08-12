# Magento 2 PWA Studio Cart Page Developer Guide & Reference

This guide provides the complete developer reference for the **Shopping Cart Page (`/cart`)** in Magento 2 PWA Studio. It details the component architecture, Peregrine talons, coupon code mutations, line-item quantity updating/removal, and practical instructions for **updating**, **removing**, or **completely replacing** any component on the cart page.

---

## Table of Contents
1. [Architecture & Component Hierarchy](#1-architecture--component-hierarchy)
2. [Internal Sub-Components Breakdown](#2-internal-sub-components-breakdown)
3. [Peregrine Talons, State & GraphQL Operations](#3-peregrine-talons-state--graphql-operations)
4. [How to UPDATE Cart Components (With Examples)](#4-how-to-update-cart-components-with-examples)
5. [How to REMOVE Cart Components (With Examples)](#5-how-to-remove-cart-components-with-examples)
6. [How to REPLACE Components (3 Architectural Approaches)](#6-how-to-replace-components-3-architectural-approaches)
7. [Full Hands-On Example: Building a 100% Custom Cart Page](#7-full-hands-on-example-building-a-100-custom-cart-page)

---

## 1. Architecture & Component Hierarchy

The default Venia Cart Page is located in `@magento/venia-ui/lib/components/CartPage/cartPage.js`.

```
<CartPage> (cartPage.js)
├── <StoreTitle />                       (Updates document <title>)
├── <h1 className={classes.heading}>     "Shopping Cart"
├── <div className={classes.items_container}>
│   └── <ProductListing />               (ProductListing/productListing.js)
│       └── <Product />                  (ProductListing/product.js - line item)
│           ├── <Image />                (Product thumbnail image)
│           ├── <Link />                 (Product title linking to PDP)
│           ├── <Section />              (Selected options: Color, Size)
│           ├── <Quantity />             (QuantityStepper for line item)
│           ├── <Price />                (Line item unit & row total price)
│           └── <button>                 (Remove item trash icon trigger)
├── <div className={classes.price_adjustments_container}>
│   └── <PriceAdjustments />             (PriceAdjustments/priceAdjustments.js)
│       ├── <CouponCode />               (CouponCode/couponCode.js - apply/remove promo)
│       ├── <GiftCards />                (GiftCards/giftCards.js - redeem gift card)
│       └── <ShippingMethods />          (ShippingMethods/shippingMethods.js - estimator)
└── <div className={classes.summary_container}>
    └── <PriceSummary />                 (PriceSummary/priceSummary.js)
        ├── Subtotal
        ├── Estimated Shipping & Tax
        ├── Applied Discounts
        ├── Grand Total
        └── <Button> "Proceed to Checkout" (Links to /checkout)
```

---

## 2. Internal Sub-Components Breakdown

| Component | Source File Path | Responsibility |
| :--- | :--- | :--- |
| `<CartPage />` | `@magento/venia-ui/lib/components/CartPage/cartPage.js` | Top-level container coordinating line items, adjustments, and summary. |
| `<ProductListing />` | `@magento/venia-ui/lib/components/CartPage/ProductListing` | Iterates over items in cart and renders `<Product />` rows. |
| `<Product />` | `@magento/venia-ui/lib/components/CartPage/ProductListing/product.js` | Single item row with quantity updater, variant labels, and item removal trigger. |
| `<PriceAdjustments />`| `@magento/venia-ui/lib/components/CartPage/PriceAdjustments` | Accordion container for coupon codes, gift cards, and shipping estimation. |
| `<CouponCode />` | `@magento/venia-ui/lib/components/CartPage/PriceAdjustments/CouponCode` | Form to apply promo codes (`applyCouponToCart`) and display active coupon tag. |
| `<ShippingMethods />`| `@magento/venia-ui/lib/components/CartPage/PriceAdjustments/ShippingMethods`| Postal code / country selector to calculate estimated shipping before checkout. |
| `<PriceSummary />` | `@magento/venia-ui/lib/components/CartPage/PriceSummary` | Calculates breakdown of totals and contains the "Proceed to Checkout" button. |

---

## 3. Peregrine Talons, State & GraphQL Operations

### 3.1 `useCartPage` Talon
* **Module Path**: `@magento/peregrine/lib/talons/CartPage/useCartPage.js`
* **Returns**:
  ```javascript
  const {
      hasItems,           // Boolean: false if cart is empty
      isCartUpdating,     // Boolean: true when quantities or coupons are mutating
      setIsCartUpdating,  // State updater
      shouldShowLoadingIndicator
  } = useCartPage();
  ```

### 3.2 `useProduct` (Line Item Operations)
* **Module Path**: `@magento/peregrine/lib/talons/CartPage/ProductListing/useProduct.js`
* **Returns**:
  ```javascript
  const {
      errorMessage,       // Stock limit error
      handleEditItem,     // Opens option edit modal
      handleRemoveItem,   // Calls removeItemFromCart mutation
      handleUpdateItemQuantity, // Calls updateCartItems mutation
      isFavorite,
      isLoading
  } = useProduct({ item });
  ```

### 3.3 Core GraphQL Mutations

#### Update Item Quantity:
```graphql
mutation UpdateCartItemQuantity($cartId: String!, $itemId: ID!, $quantity: Float!) {
    updateCartItems(
        input: {
            cart_id: $cartId
            cart_items: [{ cart_item_id: $itemId, quantity: $quantity }]
        }
    ) {
        cart {
            total_quantity
            prices {
                grand_total { value currency }
            }
        }
    }
}
```

#### Remove Item:
```graphql
mutation RemoveCartItem($cartId: String!, $itemId: ID!) {
    removeItemFromCart(
        input: {
            cart_id: $cartId
            cart_item_id: $itemId
        }
    ) {
        cart {
            total_quantity
            prices {
                grand_total { value }
            }
        }
    }
}
```

#### Apply / Remove Coupon:
```graphql
# Apply Coupon:
mutation ApplyCoupon($cartId: String!, $couponCode: String!) {
    applyCouponToCart(input: { cart_id: $cartId, coupon_code: $couponCode }) {
        cart {
            applied_coupons { code }
            prices { grand_total { value } }
        }
    }
}

# Remove Coupon:
mutation RemoveCoupon($cartId: String!) {
    removeCouponFromCart(input: { cart_id: $cartId }) {
        cart {
            applied_coupons { code }
            prices { grand_total { value } }
        }
    }
}
```

---

## 4. How to UPDATE Cart Components (With Examples)

### Example 4.1: Adding a "Free Shipping Progress Bar"
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const cartPage = targetables.reactComponent(
        '@magento/venia-ui/lib/components/CartPage/cartPage.js'
    );

    // 1. Add FreeShippingBar import
    cartPage.addImport(
        "import FreeShippingBar from '../../../../src/components/FreeShippingBar'"
    );

    // 2. Insert above line items
    cartPage.insertBeforeJSX(
        '<div className={classes.items_container}>',
        '<FreeShippingBar />'
    );
};
```

---

### Example 4.2: Adding a Cross-Sell / "You May Also Like" Product Slider
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const cartPage = targetables.reactComponent(
        '@magento/venia-ui/lib/components/CartPage/cartPage.js'
    );

    cartPage.addImport(
        "import CrossSells from '../../../../src/components/CrossSells'"
    );

    // Insert below cart summary
    cartPage.insertAfterJSX(
        '<div className={classes.summary_container}>',
        '<CrossSells />'
    );
};
```

---

## 5. How to REMOVE Cart Components (With Examples)

### Example 5.1: Removing the Shipping Estimator from Cart
If you prefer users to calculate shipping only during Checkout:

In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const priceAdjustments = targetables.reactComponent(
        '@magento/venia-ui/lib/components/CartPage/PriceAdjustments/priceAdjustments.js'
    );

    // Remove ShippingMethods accordion section
    priceAdjustments.removeJSX('<ShippingMethods setPageIsUpdating={setPageIsUpdating} />');
};
```

---

### Example 5.2: Removing Gift Cards Section
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const priceAdjustments = targetables.reactComponent(
        '@magento/venia-ui/lib/components/CartPage/PriceAdjustments/priceAdjustments.js'
    );

    // Remove GiftCards accordion section
    priceAdjustments.removeJSX('<GiftCards setPageIsUpdating={setPageIsUpdating} />');
};
```

---

## 6. How to REPLACE Components (3 Architectural Approaches)

### Method A: Targetables (AST Interception)
* **Best for**: Adding custom trust badges, loyalty points widgets, or coupon tags.
* **File**: `local-intercept.js`

---

### Method B: Webpack Aliasing (Global Replacement)
* **Best for**: Swapping `@magento/venia-ui/lib/components/CartPage/PriceSummary` with your own design.
* **File**: `webpack.config.js`

```javascript
// webpack.config.js
config.resolve.alias['@magento/venia-ui/lib/components/CartPage/PriceSummary'] = path.resolve(
    __dirname,
    'src/components/CustomPriceSummary'
);
```

---

### Method C: Standalone Custom Cart Route
* **Best for**: Total freedom over layout, tabular desktop vs card mobile line items, interactive quantity steppers, and instant coupon verification.

---

## 7. Full Hands-On Example: Building a 100% Custom Cart Page

Here is a complete, copy-pasteable custom Shopping Cart component featuring:
* Tabular line-item layout with image, product title, SKU, variant options, unit price, quantity stepper (+/-), and delete trigger
* Live Quantity mutation with optimistic updates
* Promo Coupon Code input with apply & remove functionality
* Real-time Free Shipping Progress Bar (e.g. Free shipping on orders over $140)
* Order Summary with Subtotal, Discount deductions, and Checkout CTA button

### Step 1: Create `src/components/CustomCart/CustomCart.js`
```javascript
import React, { useState } from 'react';
import { useQuery, useMutation, gql } from '@apollo/client';
import { Link } from 'react-router-dom';
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useToasts } from '@magento/peregrine';
import BrowserPersistence from '@magento/peregrine/lib/util/simplePersistence';
import classes from './customCart.module.css';

const GET_CART_DETAILS = gql`
    query GetCartPageDetails($cartId: String!) {
        cart(cart_id: $cartId) {
            id
            total_quantity
            applied_coupons {
                code
            }
            items {
                id
                quantity
                product {
                    id
                    name
                    sku
                    url_key
                    small_image {
                        url
                    }
                }
                prices {
                    price {
                        value
                        currency
                    }
                    row_total {
                        value
                        currency
                    }
                }
            }
            prices {
                subtotal_excluding_tax {
                    value
                    currency
                }
                discounts {
                    amount {
                        value
                    }
                    label
                }
                grand_total {
                    value
                    currency
                }
            }
        }
    }
`;

const UPDATE_ITEM_QTY = gql`
    mutation UpdateQty($cartId: String!, $itemId: ID!, $quantity: Float!) {
        updateCartItems(
            input: {
                cart_id: $cartId
                cart_items: [{ cart_item_id: $itemId, quantity: $quantity }]
            }
        ) {
            cart { id total_quantity }
        }
    }
`;

const REMOVE_ITEM = gql`
    mutation RemoveItem($cartId: String!, $itemId: ID!) {
        removeItemFromCart(input: { cart_id: $cartId, cart_item_id: $itemId }) {
            cart { id total_quantity }
        }
    }
`;

const APPLY_COUPON = gql`
    mutation ApplyPromo($cartId: String!, $coupon: String!) {
        applyCouponToCart(input: { cart_id: $cartId, coupon_code: $coupon }) {
            cart { id applied_coupons { code } }
        }
    }
`;

const REMOVE_COUPON = gql`
    mutation RemovePromo($cartId: String!) {
        removeCouponFromCart(input: { cart_id: $cartId }) {
            cart { id applied_coupons { code } }
        }
    }
`;

const FREE_SHIPPING_THRESHOLD = 140;

const CustomCart = () => {
    const [, { addToast }] = useToasts();
    const [cartState, cartApi] = useCartContext();
    const storage = new BrowserPersistence();
    const cartId = cartState?.cartId || storage.getItem('cartId');

    const [couponInput, setCouponInput] = useState('');
    const [isUpdating, setIsUpdating] = useState(false);

    const { data, loading, refetch } = useQuery(GET_CART_DETAILS, {
        variables: { cartId },
        skip: !cartId,
        fetchPolicy: 'cache-and-network'
    });

    const [updateQty] = useMutation(UPDATE_ITEM_QTY);
    const [removeItem] = useMutation(REMOVE_ITEM);
    const [applyCoupon] = useMutation(APPLY_COUPON);
    const [removeCoupon] = useMutation(REMOVE_COUPON);

    const cart = data?.cart;
    const items = cart?.items || [];
    const subtotal = cart?.prices?.subtotal_excluding_tax?.value || 0;
    const grandTotal = cart?.prices?.grand_total?.value || 0;
    const discounts = cart?.prices?.discounts || [];
    const appliedCoupon = cart?.applied_coupons?.[0]?.code;

    // Quantity Update
    const handleQtyChange = async (itemId, newQty) => {
        if (newQty < 1) return;
        try {
            setIsUpdating(true);
            await updateQty({ variables: { cartId, itemId, quantity: parseFloat(newQty) } });
            await refetch();
            await cartApi.getCartDetails({ cartId });
        } catch (e) {
            addToast({ type: 'error', message: e.message || 'Error updating quantity' });
        } finally {
            setIsUpdating(false);
        }
    };

    // Item Removal
    const handleRemoveItem = async (itemId, name) => {
        try {
            setIsUpdating(true);
            await removeItem({ variables: { cartId, itemId } });
            await refetch();
            await cartApi.getCartDetails({ cartId });
            addToast({ type: 'info', message: `Removed ${name} from your cart.` });
        } catch (e) {
            addToast({ type: 'error', message: e.message || 'Error removing item' });
        } finally {
            setIsUpdating(false);
        }
    };

    // Apply Promo Code
    const handleApplyCoupon = async (e) => {
        e.preventDefault();
        if (!couponInput.trim()) return;

        try {
            setIsUpdating(true);
            await applyCoupon({ variables: { cartId, coupon: couponInput.trim() } });
            await refetch();
            addToast({ type: 'info', message: `Coupon "${couponInput}" applied!` });
            setCouponInput('');
        } catch (e) {
            addToast({ type: 'error', message: e.message || 'Invalid coupon code' });
        } finally {
            setIsUpdating(false);
        }
    };

    // Remove Promo Code
    const handleRemoveCoupon = async () => {
        try {
            setIsUpdating(true);
            await removeCoupon({ variables: { cartId } });
            await refetch();
            addToast({ type: 'info', message: 'Coupon removed.' });
        } catch (e) {
            addToast({ type: 'error', message: e.message || 'Error removing coupon' });
        } finally {
            setIsUpdating(false);
        }
    };

    if (loading && !cart) return <div className={classes.loading}>Loading cart...</div>;

    if (!items.length) {
        return (
            <div className={classes.emptyState}>
                <span className={classes.emptyIcon}>🛒</span>
                <h2>Your Shopping Cart is Empty</h2>
                <p>Looks like you haven't added any items to your cart yet.</p>
                <Link to="/" className={classes.shopBtn}>Start Shopping</Link>
            </div>
        );
    }

    const freeShippingProgress = Math.min(100, (subtotal / FREE_SHIPPING_THRESHOLD) * 100);
    const amountLeft = Math.max(0, FREE_SHIPPING_THRESHOLD - subtotal);

    return (
        <div className={classes.root}>
            <h1 className={classes.title}>Shopping Cart ({cart.total_quantity} items)</h1>

            {/* Free Shipping Progress Bar */}
            <div className={classes.shippingBarBox}>
                <p className={classes.shippingBarText}>
                    {amountLeft > 0
                        ? `Add $${amountLeft.toFixed(2)} more to get FREE SHIPPING!`
                        : '🎉 Congratulations! You unlocked FREE SHIPPING!'}
                </p>
                <div className={classes.progressBar}>
                    <div
                        className={classes.progressFill}
                        style={{ width: `${freeShippingProgress}%` }}
                    />
                </div>
            </div>

            <div className={classes.layout}>
                {/* Left: Items Table */}
                <div className={classes.tableWrapper}>
                    <div className={classes.tableHeader}>
                        <span>Product</span>
                        <span>Price</span>
                        <span>Quantity</span>
                        <span>Subtotal</span>
                        <span></span>
                    </div>

                    {items.map(item => (
                        <div key={item.id} className={classes.row}>
                            {/* Product Info */}
                            <div className={classes.productCol}>
                                <img
                                    src={item.product.small_image?.url}
                                    alt={item.product.name}
                                    className={classes.itemImg}
                                />
                                <div>
                                    <Link to={`/${item.product.url_key}.html`} className={classes.itemName}>
                                        {item.product.name}
                                    </Link>
                                    <span className={classes.itemSku}>SKU: {item.product.sku}</span>
                                </div>
                            </div>

                            {/* Unit Price */}
                            <div className={classes.col}>
                                ${item.prices.price.value}
                            </div>

                            {/* Quantity Stepper */}
                            <div className={classes.col}>
                                <div className={classes.stepper}>
                                    <button
                                        type="button"
                                        onClick={() => handleQtyChange(item.id, item.quantity - 1)}
                                        disabled={item.quantity <= 1 || isUpdating}
                                    >
                                        -
                                    </button>
                                    <span>{item.quantity}</span>
                                    <button
                                        type="button"
                                        onClick={() => handleQtyChange(item.id, item.quantity + 1)}
                                        disabled={isUpdating}
                                    >
                                        +
                                    </button>
                                </div>
                            </div>

                            {/* Row Total */}
                            <div className={`${classes.col} ${classes.rowTotal}`}>
                                ${item.prices.row_total.value}
                            </div>

                            {/* Delete Button */}
                            <div className={classes.col}>
                                <button
                                    onClick={() => handleRemoveItem(item.id, item.product.name)}
                                    className={classes.deleteBtn}
                                    title="Remove item"
                                    disabled={isUpdating}
                                >
                                    ✕
                                </button>
                            </div>
                        </div>
                    ))}

                    <div className={classes.cartActions}>
                        <Link to="/" className={classes.returnBtn}>← Return to Shop</Link>
                    </div>
                </div>

                {/* Right: Summary Card */}
                <div className={classes.summaryCol}>
                    {/* Coupon Form */}
                    <div className={classes.couponCard}>
                        <h3>Coupon Code</h3>
                        {appliedCoupon ? (
                            <div className={classes.activeCoupon}>
                                <span>Code <strong>{appliedCoupon}</strong> is active</span>
                                <button onClick={handleRemoveCoupon} className={classes.removeCouponBtn}>
                                    Remove
                                </button>
                            </div>
                        ) : (
                            <form onSubmit={handleApplyCoupon} className={classes.couponForm}>
                                <input
                                    type="text"
                                    placeholder="Enter Coupon Code"
                                    value={couponInput}
                                    onChange={e => setCouponInput(e.target.value)}
                                    className={classes.couponInput}
                                />
                                <button type="submit" className={classes.applyBtn} disabled={isUpdating}>
                                    Apply
                                </button>
                            </form>
                        )}
                    </div>

                    {/* Order Total Breakdown */}
                    <div className={classes.summaryCard}>
                        <h3>Cart Total</h3>
                        <div className={classes.summaryRow}>
                            <span>Subtotal:</span>
                            <span>${subtotal.toFixed(2)}</span>
                        </div>

                        {discounts.map((disc, idx) => (
                            <div key={idx} className={`${classes.summaryRow} ${classes.discountRow}`}>
                                <span>Discount ({disc.label}):</span>
                                <span>-${disc.amount.value.toFixed(2)}</span>
                            </div>
                        ))}

                        <div className={classes.summaryRow}>
                            <span>Shipping:</span>
                            <span>{subtotal >= FREE_SHIPPING_THRESHOLD ? 'Free' : 'Calculated at checkout'}</span>
                        </div>

                        <div className={`${classes.summaryRow} ${classes.grandTotalRow}`}>
                            <span>Total:</span>
                            <span>${grandTotal.toFixed(2)}</span>
                        </div>

                        <Link to="/checkout" className={classes.checkoutBtn}>
                            Proceed to Checkout →
                        </Link>
                    </div>
                </div>
            </div>
        </div>
    );
};

export default CustomCart;
```

### Step 2: Create `src/components/CustomCart/customCart.module.css`
```css
.root {
  max-width: 1200px;
  margin: 0 auto;
  padding: 40px 16px 80px;
  font-family: var(--font-primary, sans-serif);
}

.title {
  font-size: 28px;
  font-weight: 700;
  color: #111827;
  margin-bottom: 24px;
}

/* Free Shipping Bar */
.shippingBarBox {
  background: #fdf2f2;
  border: 1px solid #fee2e2;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 32px;
}

.shippingBarText {
  font-size: 14px;
  font-weight: 600;
  color: #db4444;
  margin: 0 0 10px;
  text-align: center;
}

.progressBar {
  width: 100%;
  height: 8px;
  background: #fee2e2;
  border-radius: 999px;
  overflow: hidden;
}

.progressFill {
  height: 100%;
  background: #db4444;
  transition: width 0.3s ease;
}

.layout {
  display: grid;
  grid-template-columns: 1.6fr 1fr;
  gap: 40px;
}

@media (max-width: 900px) {
  .layout {
    grid-template-columns: 1fr;
  }
}

/* Items Table */
.tableWrapper {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.tableHeader {
  display: grid;
  grid-template-columns: 2.5fr 1fr 1fr 1fr 40px;
  padding: 14px 16px;
  background: #f9fafb;
  border-radius: 6px;
  font-weight: 600;
  font-size: 14px;
  color: #4b5563;
}

@media (max-width: 600px) {
  .tableHeader {
    display: none;
  }
}

.row {
  display: grid;
  grid-template-columns: 2.5fr 1fr 1fr 1fr 40px;
  align-items: center;
  padding: 16px;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

@media (max-width: 600px) {
  .row {
    grid-template-columns: 1fr;
    gap: 12px;
  }
}

.productCol {
  display: flex;
  align-items: center;
  gap: 16px;
}

.itemImg {
  width: 60px;
  height: 60px;
  object-fit: contain;
  background: #f9fafb;
  border-radius: 4px;
}

.itemName {
  font-size: 14px;
  font-weight: 600;
  color: #111827;
  text-decoration: none;
}

.itemName:hover {
  color: #db4444;
}

.itemSku {
  display: block;
  font-size: 12px;
  color: #6b7280;
  margin-top: 4px;
}

.col {
  font-size: 14px;
  color: #374151;
}

.rowTotal {
  font-weight: 700;
  color: #111827;
}

.stepper {
  display: inline-flex;
  align-items: center;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.stepper button {
  background: transparent;
  border: none;
  width: 28px;
  height: 28px;
  cursor: pointer;
}

.stepper span {
  padding: 0 8px;
  font-weight: 600;
  font-size: 13px;
}

.deleteBtn {
  background: transparent;
  border: none;
  color: #9ca3af;
  font-size: 16px;
  cursor: pointer;
}

.deleteBtn:hover {
  color: #ef4444;
}

.cartActions {
  display: flex;
  justify-content: space-between;
  margin-top: 16px;
}

.returnBtn {
  color: #374151;
  text-decoration: none;
  font-weight: 600;
  font-size: 14px;
}

/* Summary Column */
.summaryCol {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.couponCard,
.summaryCard {
  border: 1.5px solid #111827;
  border-radius: 8px;
  padding: 24px;
}

.couponCard h3,
.summaryCard h3 {
  font-size: 18px;
  font-weight: 700;
  margin: 0 0 16px;
}

.couponForm {
  display: flex;
  gap: 12px;
}

.couponInput {
  flex: 1;
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  font-size: 14px;
  outline: none;
}

.applyBtn {
  background: #db4444;
  color: #ffffff;
  border: none;
  padding: 10px 20px;
  border-radius: 4px;
  font-weight: 600;
  cursor: pointer;
}

.activeCoupon {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #f3f4f6;
  padding: 10px 14px;
  border-radius: 4px;
  font-size: 14px;
}

.removeCouponBtn {
  background: none;
  border: none;
  color: #ef4444;
  cursor: pointer;
  text-decoration: underline;
}

.summaryRow {
  display: flex;
  justify-content: space-between;
  padding: 10px 0;
  font-size: 14px;
  color: #4b5563;
  border-bottom: 1px solid #f3f4f6;
}

.discountRow {
  color: #10b981;
  font-weight: 600;
}

.grandTotalRow {
  font-size: 18px;
  font-weight: 800;
  color: #111827;
  border-top: 2px solid #111827;
  border-bottom: none;
  margin-top: 8px;
  padding-top: 16px;
}

.checkoutBtn {
  display: block;
  text-align: center;
  background: #db4444;
  color: #ffffff;
  text-decoration: none;
  padding: 14px;
  border-radius: 6px;
  font-weight: 700;
  margin-top: 20px;
  transition: background-color 0.2s;
}

.checkoutBtn:hover {
  background: #b91c1c;
}

.emptyState {
  max-width: 500px;
  margin: 60px auto;
  text-align: center;
}

.emptyIcon {
  font-size: 64px;
}

.shopBtn {
  display: inline-block;
  background: #db4444;
  color: #ffffff;
  text-decoration: none;
  padding: 12px 32px;
  border-radius: 6px;
  font-weight: 600;
  margin-top: 16px;
}
```

---

*Authored for Magento 2 PWA Studio Storefront Development.*
