# Magento 2 PWA Studio Homepage Developer Guide & Reference

This guide provides the complete developer reference for the **Homepage** in Magento 2 PWA Studio. It details how the default Venia CMS homepage works, how to fetch dynamic catalog data via GraphQL, and practical instructions for **updating**, **removing**, or **completely replacing** the homepage with a custom React storefront.

---

## Table of Contents
1. [Architecture: Default CMS Homepage vs Custom React Homepage](#1-architecture-default-cms-homepage-vs-custom-react-homepage)
2. [Default CMS Page Flow & PageBuilder](#2-default-cms-page-flow--pagebuilder)
3. [GraphQL Operations for Homepage Catalog Data](#3-graphql-operations-for-homepage-catalog-data)
4. [How to UPDATE Homepage Elements](#4-how-to-update-homepage-elements)
5. [How to REMOVE Default Sections](#5-how-to-remove-default-sections)
6. [How to REPLACE the Homepage (3 Methods)](#6-how-to-replace-the-homepage-3-methods)
7. [Full Hands-On Example: Building a 100% Custom E-Commerce Homepage](#7-full-hands-on-example-building-a-100-custom-e-commerce-homepage)

---

## 1. Architecture: Default CMS Homepage vs Custom React Homepage

In Magento PWA Studio, there are two distinct ways to build and serve the homepage:

```
                          ┌────────────────────────┐
                          │  Incoming Route: "/"   │
                          └───────────┬────────────┘
                                      │
            ┌─────────────────────────┴─────────────────────────┐
            │                                                   │
            ▼                                                   ▼
┌───────────────────────────────┐               ┌───────────────────────────────┐
│ METHOD A: Default Venia       │               │ METHOD B: Custom React        │
│ (CMS Page / PageBuilder)      │               │ (Dedicated Component)         │
├───────────────────────────────┤               ├───────────────────────────────┤
│ • Resolves URL via GraphQL    │               │ • Intercepts exact path="/"   │
│ • Fetches "home" CMS Page     │               │ • Fully dynamic React UI      │
│ • Renders PageBuilder HTML    │               │ • Direct Apollo GraphQL data  │
│ • Limited frontend control    │               │ • 100% custom styling & state │
└───────────────────────────────┘               └───────────────────────────────┘
```

---

## 2. Default CMS Page Flow & PageBuilder

### How Default Venia Loads the Homepage:
1. When navigating to `/`, `<MagentoRoute />` queries Magento with `resolveURL(url: "/")`.
2. Magento returns `{ type: "CMS_PAGE", identifier: "home" }`.
3. PWA Studio mounts `@magento/venia-ui/lib/RootComponents/CMS/cms.js`.
4. `cms.js` uses the talon `@magento/peregrine/lib/talons/RootComponents/CMS/useCmsPage.js` to execute:
   ```graphql
   query getCmsPage($identifier: String!) {
       cmsPage(identifier: $identifier) {
           title
           content
           content_heading
           meta_title
           meta_keywords
           meta_description
       }
   }
   ```
5. If PageBuilder content is detected, `@magento/pagebuilder` parses the HTML rows, columns, banners, and sliders.

---

## 3. GraphQL Operations for Homepage Catalog Data

When building a high-performance custom React homepage, you query Magento's catalog directly using Apollo Client.

### 3.1 Fetching Featured & Flash Sale Products
```javascript
import { gql } from '@apollo/client';

export const GET_HOMEPAGE_PRODUCTS = gql`
    query GetHomepageProducts($pageSize: Int!, $currentPage: Int!) {
        products(search: "", pageSize: $pageSize, currentPage: $currentPage) {
            total_count
            items {
                id
                uid
                name
                sku
                url_key
                stock_status
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
                            currency
                        }
                        discount {
                            amount_off
                            percent_off
                        }
                    }
                }
            }
        }
    }
`;
```

### 3.2 Fetching Category Tiles
```javascript
export const GET_HOMEPAGE_CATEGORIES = gql`
    query GetHomepageCategories {
        categories(filters: { parent_id: { in: ["2"] } }) {
            items {
                id
                uid
                name
                url_path
                image
            }
        }
    }
`;
```

---

## 4. How to UPDATE Homepage Elements

### Option 4.1: Updating via Magento 2 Admin (CMS / PageBuilder)
* Navigate to **Admin Panel > Content > Pages > Home Page**.
* Use the drag-and-drop PageBuilder to edit banners, text, buttons, and product sliders.
* Flush cache: `php bin/magento cache:clean`.

### Option 4.2: Updating a Custom React Homepage
* Edit your custom homepage component (e.g. `src/components/CustomHomePage/CustomHomePage.js`).
* Modify GraphQL query variables (e.g., `pageSize: 8` for top products), promotional banner copy, or seasonal countdown timers.

---

## 5. How to REMOVE Default Sections

If you are using default Venia and want to hide or remove default PageBuilder elements or footer/header attachments:

### Example 5.1: Removing CMS Heading via CSS
In `src/index.css`:
```css
/* Hide default CMS Page title on homepage */
.cms-home .page-title-wrapper {
  display: none !important;
}
```

### Example 5.2: Bypassing the CMS Engine Entirely
The cleanest way to remove all default Venia homepage blocks is to declare a custom route for `/` in your router (see Method C below).

---

## 6. How to REPLACE the Homepage (3 Methods)

### Method A: Route Hijack in Custom App Shell (Recommended)
Add a `<Route exact path="/">` inside your `<Switch>` before the fallback `<Routes />`:

```javascript
// src/components/CustomLayout/CustomApp.js
import React from 'react';
import { Switch, Route } from 'react-router-dom';
import CustomHomePage from '../CustomHomePage/CustomHomePage';
import Routes from '@magento/venia-ui/lib/components/Routes';

const CustomApp = () => (
    <Switch>
        {/* 1. Custom React Homepage */}
        <Route exact path="/" component={CustomHomePage} />

        {/* 2. Fallback to PDP, PLP, Search, Cart & Account */}
        <Route path="*">
            <Routes />
        </Route>
    </Switch>
);

export default CustomApp;
```

---

### Method B: Webpack Aliasing for CMS RootComponent
If you want all CMS home requests to resolve to your custom component:

In `webpack.config.js`:
```javascript
config.resolve.alias['@magento/venia-ui/lib/RootComponents/CMS'] = path.resolve(
    __dirname,
    'src/components/CustomHomePage'
);
```

---

### Method C: Targetables AST Interception
Inject a custom section into the default Venia CMS page:

In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const cmsComponent = targetables.reactComponent(
        '@magento/venia-ui/lib/RootComponents/CMS/cms.js'
    );

    cmsComponent.addImport("import PromoBanner from '../../../../src/components/PromoBanner'");
    cmsComponent.insertBeforeJSX('<RichContent', '<PromoBanner />');
};
```

---

## 7. Full Hands-On Example: Building a 100% Custom E-Commerce Homepage

Here is a complete, production-ready custom homepage featuring:
1. **Hero Banner**: High-impact promotional hero slider/banner with CTA buttons.
2. **Category Grid**: Visual category browse cards.
3. **Dynamic Flash Sale / Featured Product Grid**: Live GraphQL catalog query, price discount badges, rating stars, and one-click **Add to Cart**.
4. **Service & Value Proposition Badges**: Free Delivery, 24/7 Customer Service, and Money Back Guarantee.

### Step 1: Create `src/components/CustomHomePage/CustomHomePage.js`
```javascript
import React, { useState } from 'react';
import { useQuery, useMutation, gql } from '@apollo/client';
import { Link, useHistory } from 'react-router-dom';
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useToasts } from '@magento/peregrine';
import BrowserPersistence from '@magento/peregrine/lib/util/simplePersistence';
import classes from './customHomePage.module.css';

const GET_HOMEPAGE_PRODUCTS = gql`
    query GetHomepageProducts($pageSize: Int!) {
        products(search: "", pageSize: $pageSize, currentPage: 1) {
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
                            amount_off
                            percent_off
                        }
                    }
                }
            }
        }
    }
`;

const ADD_TO_CART_MUTATION = gql`
    mutation AddItemToCart($cartId: String!, $sku: String!) {
        addProductsToCart(cartId: $cartId, cartItems: [{ sku: $sku, quantity: 1 }]) {
            cart {
                id
                total_quantity
            }
        }
    }
`;

const CustomHomePage = () => {
    const history = useHistory();
    const [, { addToast }] = useToasts();
    const [cartState, cartApi] = useCartContext();
    const [addingSku, setAddingSku] = useState(null);

    const { data, loading, error } = useQuery(GET_HOMEPAGE_PRODUCTS, {
        variables: { pageSize: 8 },
        fetchPolicy: 'cache-and-network'
    });

    const [addToCart] = useMutation(ADD_TO_CART_MUTATION);

    const handleAddToCart = async (product) => {
        // If product has options (size/color), redirect to PDP
        if (product.__typename === 'ConfigurableProduct') {
            history.push(`/${product.url_key}.html`);
            return;
        }

        try {
            setAddingSku(product.sku);
            const storage = new BrowserPersistence();
            const cartId = cartState?.cartId || storage.getItem('cartId');

            await addToCart({ variables: { cartId, sku: product.sku } });
            await cartApi.getCartDetails({ cartId });

            addToast({
                type: 'info',
                message: `Added ${product.name} to your shopping bag!`,
                timeout: 4000
            });
        } catch (err) {
            addToast({
                type: 'error',
                message: err.message || 'Could not add item to cart.',
                timeout: 4000
            });
        } finally {
            setAddingSku(null);
        }
    };

    const products = data?.products?.items || [];

    return (
        <div className={classes.root}>
            {/* 1. Hero Section */}
            <section className={classes.hero}>
                <div className={classes.heroContent}>
                    <span className={classes.heroSubtitle}>iPhone 14 Series</span>
                    <h1 className={classes.heroTitle}>Up to 10% off Voucher</h1>
                    <Link to="/sale.html" className={classes.heroBtn}>
                        Shop Now →
                    </Link>
                </div>
            </section>

            {/* 2. Featured Flash Sale Products */}
            <section className={classes.section}>
                <div className={classes.sectionHeader}>
                    <div className={classes.badgeTag}>Today's</div>
                    <h2 className={classes.sectionTitle}>Flash Sales</h2>
                </div>

                {loading && !data && (
                    <div className={classes.grid}>
                        {[...Array(4)].map((_, i) => (
                            <div key={i} className={classes.skeleton} />
                        ))}
                    </div>
                )}

                {error && <p className={classes.error}>Failed to load catalog.</p>}

                <div className={classes.grid}>
                    {products.map(product => {
                        const minPrice = product.price_range?.minimum_price;
                        const regularPrice = minPrice?.regular_price?.value;
                        const finalPrice = minPrice?.final_price?.value;
                        const percentOff = minPrice?.discount?.percent_off;
                        const isAdding = addingSku === product.sku;

                        return (
                            <div key={product.id} className={classes.productCard}>
                                {percentOff > 0 && (
                                    <span className={classes.discountBadge}>-{Math.round(percentOff)}%</span>
                                )}

                                <Link to={`/${product.url_key}.html`} className={classes.imageWrapper}>
                                    <img
                                        src={product.small_image?.url}
                                        alt={product.name}
                                        className={classes.productImage}
                                        loading="lazy"
                                    />
                                </Link>

                                <div className={classes.cardBody}>
                                    <h3 className={classes.productName}>
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
                                        className={classes.addToCartBtn}
                                        disabled={isAdding}
                                    >
                                        {isAdding ? 'Adding...' : product.__typename === 'ConfigurableProduct' ? 'Select Options' : 'Add To Cart'}
                                    </button>
                                </div>
                            </div>
                        );
                    })}
                </div>
            </section>

            {/* 3. Value Proposition / Feature Highlights */}
            <section className={classes.featuresSection}>
                <div className={classes.featureCard}>
                    <div className={classes.featureIcon}>🚚</div>
                    <h4 className={classes.featureTitle}>FREE AND FAST DELIVERY</h4>
                    <p className={classes.featureDesc}>Free delivery on all orders over $140</p>
                </div>
                <div className={classes.featureCard}>
                    <div className={classes.featureIcon}>🎧</div>
                    <h4 className={classes.featureTitle}>24/7 CUSTOMER SERVICE</h4>
                    <p className={classes.featureDesc}>Friendly 24/7 customer support</p>
                </div>
                <div className={classes.featureCard}>
                    <div className={classes.featureIcon}>🛡️</div>
                    <h4 className={classes.featureTitle}>MONEY BACK GUARANTEE</h4>
                    <p className={classes.featureDesc}>We return money within 30 days</p>
                </div>
            </section>
        </div>
    );
};

export default CustomHomePage;
```

### Step 2: Create `src/components/CustomHomePage/customHomePage.module.css`
```css
.root {
  width: 100%;
  display: flex;
  flex-direction: column;
  gap: 60px;
  padding-bottom: 60px;
  font-family: var(--font-primary, sans-serif);
}

/* 1. Hero Section */
.hero {
  background: #000000;
  color: #ffffff;
  border-radius: 12px;
  padding: 60px 48px;
  display: flex;
  align-items: center;
  min-height: 320px;
}

.heroSubtitle {
  color: #db4444;
  font-size: 16px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.heroTitle {
  font-size: 42px;
  font-weight: 700;
  margin: 12px 0 24px;
  line-height: 1.2;
}

.heroBtn {
  display: inline-block;
  background: #db4444;
  color: #ffffff;
  padding: 12px 32px;
  border-radius: 6px;
  font-weight: 600;
  text-decoration: none;
  transition: opacity 0.2s;
}

.heroBtn:hover {
  opacity: 0.9;
}

/* 2. Section Header */
.section {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.sectionHeader {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.badgeTag {
  color: #db4444;
  font-weight: 700;
  font-size: 14px;
  border-left: 10px solid #db4444;
  padding-left: 10px;
}

.sectionTitle {
  font-size: 28px;
  font-weight: 700;
  color: #000000;
  margin: 0;
}

/* Product Grid */
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
  }
}

@media (max-width: 480px) {
  .grid {
    grid-template-columns: 1fr;
  }
}

.productCard {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
  position: relative;
  display: flex;
  flex-direction: column;
  transition: box-shadow 0.2s;
}

.productCard:hover {
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
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 24px;
  height: 220px;
}

.productImage {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
}

.cardBody {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  flex: 1;
}

.productName {
  font-size: 15px;
  font-weight: 600;
  margin: 0;
}

.productName a {
  color: #111827;
  text-decoration: none;
}

.productName a:hover {
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

.addToCartBtn {
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

.addToCartBtn:hover {
  background: #db4444;
}

.addToCartBtn:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

/* 3. Features Section */
.featuresSection {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 32px;
  padding: 40px 0;
}

@media (max-width: 768px) {
  .featuresSection {
    grid-template-columns: 1fr;
    gap: 24px;
  }
}

.featureCard {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 12px;
}

.featureIcon {
  font-size: 36px;
  background: #f3f4f6;
  width: 70px;
  height: 70px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.featureTitle {
  font-size: 16px;
  font-weight: 700;
  color: #000000;
  margin: 0;
}

.featureDesc {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
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
