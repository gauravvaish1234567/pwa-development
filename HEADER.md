# Magento 2 PWA Studio Header Developer Guide & Reference

This guide provides the complete developer reference for the **Header** component in Magento 2 PWA Studio. It details every sub-component, Peregrine talon, GraphQL query, and state context, along with practical examples for **updating**, **removing**, or **completely replacing** any part of the header.

---

## Table of Contents
1. [Architecture & Component Hierarchy](#1-architecture--component-hierarchy)
2. [Internal Sub-Components Breakdown](#2-internal-sub-components-breakdown)
3. [Peregrine Talons, State & GraphQL Operations](#3-peregrine-talons-state--graphql-operations)
4. [How to UPDATE Header Components (With Examples)](#4-how-to-update-header-components-with-examples)
5. [How to REMOVE Components from Header (With Examples)](#5-how-to-remove-components-from-header-with-examples)
6. [How to REPLACE Components (3 Architectural Approaches)](#6-how-to-replace-components-3-architectural-approaches)
7. [Full Hands-On Example: Building a 100% Custom Header](#7-full-hands-on-example-building-a-100-custom-header)

---

## 1. Architecture & Component Hierarchy

The default Venia header is located in `@magento/venia-ui/lib/components/Header/header.js`.

```
<Header> (header.js)
├── <div className={classes.switchersContainer}>
│   └── <div className={classes.switchers}>
│       ├── <StoreSwitcher />       (storeSwitcher.js)
│       └── <CurrencySwitcher />    (currencySwitcher.js)
├── <header className={rootClass}>
│   └── <div className={classes.toolbar}>
│       ├── <div className={classes.primaryActions}>
│       │   └── <NavTrigger />      (navTrigger.js - triggers side drawer menu)
│       ├── <Link to="/">
│       │   └── <Logo />            (../Logo/logo.js)
│       ├── <MegaMenu />            (../MegaMenu/megaMenu.js - desktop category nav)
│       └── <div className={classes.secondaryActions}>
│           ├── <SearchTrigger />   (searchTrigger.js - toggles search bar)
│           ├── <AccountTrigger />  (accountTrigger.js - auth & user dropdown)
│           └── <CartTrigger />     (cartTrigger.js - mini-cart drawer toggle)
├── {searchBar}                     (React.lazy -> ../SearchBar/searchBar.js)
├── <PageLoadingIndicator />        (../PageLoadingIndicator)
└── <OnlineIndicator />             (onlineIndicator.js - network connectivity pill)
```

---

## 2. Internal Sub-Components Breakdown

| Sub-Component | Source File Path | Responsibility |
| :--- | :--- | :--- |
| `<StoreSwitcher />` | `@magento/venia-ui/lib/components/Header/storeSwitcher.js` | Dropdown to switch between Magento Store Views (e.g. English, French, German). |
| `<CurrencySwitcher />` | `@magento/venia-ui/lib/components/Header/currencySwitcher.js` | Dropdown to switch display currencies (e.g. USD, EUR, GBP). |
| `<NavTrigger />` | `@magento/venia-ui/lib/components/Header/navTrigger.js` | Hamburger menu icon button that opens the mobile/sliding navigation drawer. |
| `<Logo />` | `@magento/venia-ui/lib/components/Logo/logo.js` | Brand logo (SVG/image) linking to the homepage (`/`). |
| `<MegaMenu />` | `@magento/venia-ui/lib/components/MegaMenu/megaMenu.js` | Desktop multi-level category navigation bar with hover dropdowns. |
| `<SearchTrigger />` | `@magento/venia-ui/lib/components/Header/searchTrigger.js` | Magnifying glass button that expands/opens `<SearchBar />`. |
| `<SearchBar />` | `@magento/venia-ui/lib/components/SearchBar/searchBar.js` | Lazy-loaded search input with live autocomplete suggestions and search history. |
| `<AccountTrigger />`| `@magento/venia-ui/lib/components/Header/accountTrigger.js`| User avatar icon button. Opens sign-in/register modal or account dropdown menu. |
| `<CartTrigger />` | `@magento/venia-ui/lib/components/Header/cartTrigger.js` | Shopping bag icon button with dynamic item quantity badge; toggles MiniCart. |
| `<OnlineIndicator />`| `@magento/venia-ui/lib/components/Header/onlineIndicator.js`| Displays notification banner when user goes offline or reconnects online. |
| `<PageLoadingIndicator />`| `@magento/venia-ui/lib/components/PageLoadingIndicator` | Top-edge progress bar indicating page route transitions. |

---

## 3. Peregrine Talons, State & GraphQL Operations

### 3.1 `useHeader` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Header/useHeader.js`
* **Returns**:
  ```javascript
  const {
      handleSearchTriggerClick, // Callback to toggle search state
      hasBeenOffline,           // Boolean: true if user experienced disconnection
      isOnline,                 // Boolean: current network state
      isSearchOpen,             // Boolean: whether search bar modal is open
      searchRef,                // Ref attached to SearchBar container
      searchTriggerRef          // Ref attached to SearchTrigger button
  } = useHeader();
  ```

### 3.2 `useCartTrigger` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Header/useCartTrigger.js`
* **Returns**:
  ```javascript
  const {
      itemCount,                // Number: total items in cart (sum of quantities)
      handleClick,              // Callback: toggles mini-cart drawer open/closed
      isMiniCartOpen            // Boolean: mini-cart drawer open state
  } = useCartTrigger({ queries: { getItemCountQuery: GET_ITEM_COUNT } });
  ```

### 3.3 `useAccountTrigger` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Header/useAccountTrigger.js`
* **Returns**:
  ```javascript
  const {
      accountMenuIsOpen,        // Boolean
      accountMenuRef,           // Ref
      accountMenuTriggerRef,    // Ref
      setAccountMenuIsOpen,     // State setter
      handleTriggerClick        // Callback
  } = useAccountTrigger();
  ```

### 3.4 Direct Global State Hooks
If building a custom header, you can bypass complex talons and access global state directly:
```javascript
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useUserContext } from '@magento/peregrine/lib/context/user';

// Cart State:
const [cartState, cartApi] = useCartContext();
const totalItems = cartState.derivedDetails.numItems;

// Auth / Customer State:
const [userState, userApi] = useUserContext();
const { isSignedIn, currentUser } = userState;
```

---

## 4. How to UPDATE Header Components (With Examples)

### Example 4.1: Customizing Header Styling & Colors via CSS
To change the header height, background, or font colors without touching JSX:

In `src/index.css`:
```css
/* Override Venia Header CSS Tokens */
:root {
  --venia-global-color-header-background: #ffffff;
  --venia-global-color-header-text: #1a1a1a;
}

/* Custom header bar styling */
header[data-cy="Header-root"] {
  background-color: #ffffff !important;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.06);
  border-bottom: 1px solid #eaeaea;
}
```

---

### Example 4.2: Updating the Brand Logo Image
Create a custom Logo component at `src/components/CustomLogo/customLogo.js`:

```javascript
import React from 'react';
import { Link } from 'react-router-dom';
import classes from './customLogo.module.css';

const CustomLogo = () => {
    return (
        <Link to="/" className={classes.logoLink}>
            <img
                src="/brand-logo.svg"
                alt="My Store Logo"
                className={classes.logoImage}
                width={150}
                height={40}
            />
        </Link>
    );
};

export default CustomLogo;
```

Then swap it into Venia Header via `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const header = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Header/header.js'
    );

    // Add import for our custom logo
    header.addImport("import CustomLogo from '../../../../src/components/CustomLogo/customLogo'");

    // Replace the default <Logo /> tag
    header.replaceJSX('<Logo classes={{ logo: classes.logo }} />', '<CustomLogo />');
};
```

---

### Example 4.3: Adding a Custom Top Promo Bar / Announcement
In `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const header = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Header/header.js'
    );

    header.insertBeforeJSX(
        '<header className={rootClass} data-cy="Header-root">',
        `
        <div style={{ background: '#000', color: '#fff', textAlign: 'center', padding: '8px', fontSize: '13px' }}>
            ⚡ Summer Flash Sale! Use code <strong>SUMMER20</strong> for 20% OFF!
        </div>
        `
    );
};
```

---

## 5. How to REMOVE Components from Header (With Examples)

### Example 5.1: Removing StoreSwitcher and CurrencySwitcher
If you operate a single store and single currency, remove the switcher bar using `local-intercept.js`:

```javascript
// local-intercept.js
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const header = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Header/header.js'
    );

    // Remove the switchers container entirely
    header.removeJSX('<div className={classes.switchersContainer} />');
};
```

---

### Example 5.2: Removing SearchTrigger or NavTrigger
In `local-intercept.js`:
```javascript
// Remove NavTrigger (hamburger button)
header.removeJSX('<NavTrigger />');

// Remove SearchTrigger
header.removeJSX('<SearchTrigger onClick={handleSearchTriggerClick} ref={searchTriggerRef} />');
```

---

## 6. How to REPLACE Components (3 Architectural Approaches)

### Method A: Targetables (AST Interception)
* **Best for**: Small to medium customizations of the existing Venia header.
* **File**: `local-intercept.js`

```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const header = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Header/header.js'
    );

    // 1. Add custom import
    header.addImport("import WishlistTrigger from '../../../../src/components/WishlistTrigger'");

    // 2. Insert custom component next to CartTrigger
    header.insertAfterJSX(
        '<CartTrigger />',
        '<WishlistTrigger />'
    );
};
```

---

### Method B: Webpack Aliasing (Global Replacement)
* **Best for**: Replacing `@magento/venia-ui/lib/components/Header` across all references with your custom component.
* **File**: `webpack.config.js`

```javascript
// webpack.config.js
const path = require('path');

module.exports = async env => {
    const config = await configureWebpack({
        context: __dirname,
        vendor: [ /* ... */ ],
        env
    });

    config.resolve.alias['@magento/venia-ui/lib/components/Header'] = path.resolve(
        __dirname,
        'src/components/MyCustomHeader'
    );

    return config;
};
```

---

### Method C: Root Layout Replacement (Cleanest for 100% Custom Storefronts)
* **Best for**: Total control over markup, styling, sticky behaviors, mobile navigation, and layout.
* **File**: `src/index.js` and `src/components/CustomLayout/CustomApp.js`

Mount your custom app inside `<Adapter>` in `src/index.js`, and render your own `<CustomHeader />` without touching Venia header at all.

---

## 7. Full Hands-On Example: Building a 100% Custom Header

Here is a complete, copy-pasteable custom header component with:
* Live reactive Cart Badge count (`useCartContext`)
* Customer Auth trigger / Name display (`useUserContext`)
* Integrated Search Input with submit handler
* Responsive Layout & Navigation Links

### Step 1: Create `src/components/CustomHeader/CustomHeader.js`
```javascript
import React, { useState } from 'react';
import { Link, useHistory } from 'react-router-dom';
import { useCartContext } from '@magento/peregrine/lib/context/cart';
import { useUserContext } from '@magento/peregrine/lib/context/user';
import classes from './customHeader.module.css';

const CustomHeader = () => {
    const history = useHistory();
    const [searchQuery, setSearchQuery] = useState('');

    // Access Peregrine global states
    const [cartState] = useCartContext();
    const [userState, userApi] = useUserContext();

    const cartCount = cartState?.derivedDetails?.numItems || 0;
    const { isSignedIn, currentUser } = userState;

    const handleSearchSubmit = e => {
        e.preventDefault();
        if (searchQuery.trim()) {
            history.push(`/search.html?query=${encodeURIComponent(searchQuery.trim())}`);
        }
    };

    const handleSignOut = async () => {
        await userApi.signOut();
        history.push('/');
    };

    return (
        <header className={classes.header}>
            {/* 1. Top Announcement Bar */}
            <div className={classes.topBar}>
                <div className={classes.container}>
                    <span>Summer Sale: 20% OFF all items with code <strong>SUMMER20</strong></span>
                    <div className={classes.topLinks}>
                        <Link to="/contact">Help & Contact</Link>
                        <Link to="/store-locator">Store Locator</Link>
                    </div>
                </div>
            </div>

            {/* 2. Main Navigation Bar */}
            <div className={classes.mainNav}>
                <div className={classes.container}>
                    {/* Brand Logo */}
                    <Link to="/" className={classes.logo}>
                        <span className={classes.logoHighlight}>MY</span>STORE
                    </Link>

                    {/* Navigation Links */}
                    <nav className={classes.navLinks}>
                        <Link to="/women.html" className={classes.navItem}>Women</Link>
                        <Link to="/men.html" className={classes.navItem}>Men</Link>
                        <Link to="/gear.html" className={classes.navItem}>Gear</Link>
                        <Link to="/sale.html" className={classes.navItemSale}>Sale</Link>
                    </nav>

                    {/* Integrated Search Bar */}
                    <form onSubmit={handleSearchSubmit} className={classes.searchForm}>
                        <input
                            type="search"
                            placeholder="Search products..."
                            value={searchQuery}
                            onChange={e => setSearchQuery(e.target.value)}
                            className={classes.searchInput}
                        />
                        <button type="submit" className={classes.searchBtn} aria-label="Search">
                            🔍
                        </button>
                    </form>

                    {/* Action Triggers (User & Cart) */}
                    <div className={classes.actions}>
                        {isSignedIn ? (
                            <div className={classes.userMenu}>
                                <Link to="/order-history" className={classes.userName}>
                                    Hi, {currentUser.firstname || 'Account'}
                                </Link>
                                <button onClick={handleSignOut} className={classes.logoutBtn}>
                                    Sign Out
                                </button>
                            </div>
                        ) : (
                            <Link to="/login" className={classes.actionBtn}>
                                Sign In
                            </Link>
                        )}

                        {/* Cart Trigger */}
                        <Link to="/cart" className={classes.cartBtn} aria-label="Cart">
                            <span className={classes.cartIcon}>🛒</span>
                            {cartCount > 0 && (
                                <span className={classes.cartBadge}>{cartCount}</span>
                            )}
                        </Link>
                    </div>
                </div>
            </div>
        </header>
    );
};

export default CustomHeader;
```

### Step 2: Create `src/components/CustomHeader/customHeader.module.css`
```css
.header {
  width: 100%;
  background: #ffffff;
  border-bottom: 1px solid #e5e7eb;
  position: sticky;
  top: 0;
  z-index: 100;
}

.topBar {
  background: #111827;
  color: #f9fafb;
  font-size: 12px;
  padding: 6px 0;
}

.container {
  max-width: 1240px;
  margin: 0 auto;
  padding: 0 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.topLinks {
  display: flex;
  gap: 16px;
}

.topLinks a {
  color: #d1d5db;
  text-decoration: none;
}

.topLinks a:hover {
  color: #ffffff;
}

.mainNav {
  padding: 14px 0;
}

.logo {
  font-size: 22px;
  font-weight: 800;
  letter-spacing: -0.5px;
  color: #111827;
  text-decoration: none;
}

.logoHighlight {
  color: #db4444;
}

.navLinks {
  display: flex;
  gap: 24px;
}

.navItem {
  color: #374151;
  font-weight: 500;
  text-decoration: none;
  transition: color 0.2s;
}

.navItem:hover {
  color: #db4444;
}

.navItemSale {
  color: #db4444;
  font-weight: 700;
  text-decoration: none;
}

.searchForm {
  display: flex;
  align-items: center;
  position: relative;
  width: 260px;
}

.searchInput {
  width: 100%;
  padding: 8px 36px 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  outline: none;
}

.searchInput:focus {
  border-color: #db4444;
}

.searchBtn {
  position: absolute;
  right: 8px;
  background: none;
  border: none;
  cursor: pointer;
}

.actions {
  display: flex;
  align-items: center;
  gap: 16px;
}

.actionBtn {
  color: #111827;
  font-weight: 500;
  text-decoration: none;
  font-size: 14px;
}

.userMenu {
  display: flex;
  align-items: center;
  gap: 8px;
}

.userName {
  font-weight: 600;
  font-size: 14px;
  color: #111827;
  text-decoration: none;
}

.logoutBtn {
  background: none;
  border: none;
  color: #6b7280;
  font-size: 12px;
  cursor: pointer;
  text-decoration: underline;
}

.cartBtn {
  position: relative;
  display: flex;
  align-items: center;
  text-decoration: none;
  color: #111827;
  font-size: 20px;
}

.cartBadge {
  position: absolute;
  top: -6px;
  right: -10px;
  background: #db4444;
  color: #ffffff;
  font-size: 11px;
  font-weight: 700;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

---

*Authored for Magento 2 PWA Studio Storefront Development.*
