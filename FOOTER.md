# Magento 2 PWA Studio Footer Developer Guide & Reference

This guide provides the complete developer reference for the **Footer** component in Magento 2 PWA Studio. It covers the component architecture, sub-component breakdown, Peregrine talons, GraphQL data fetching, and practical instructions for **updating**, **removing**, or **completely replacing** any part of the footer.

---

## Table of Contents
1. [Architecture & Component Hierarchy](#1-architecture--component-hierarchy)
2. [Internal Sub-Components Breakdown](#2-internal-sub-components-breakdown)
3. [Peregrine Talons, State & GraphQL Operations](#3-peregrine-talons-state--graphql-operations)
4. [How to UPDATE Footer Components (With Examples)](#4-how-to-update-footer-components-with-examples)
5. [How to REMOVE Components from Footer (With Examples)](#5-how-to-remove-components-from-footer-with-examples)
6. [How to REPLACE Components (3 Architectural Approaches)](#6-how-to-replace-components-3-architectural-approaches)
7. [Full Hands-On Example: Building a 100% Custom Footer](#7-full-hands-on-example-building-a-100-custom-footer)

---

## 1. Architecture & Component Hierarchy

The default Venia footer is located in `@magento/venia-ui/lib/components/Footer/footer.js`.

```
<Footer> (footer.js)
├── <div className={classes.links}>
│   ├── {linkGroups}                    (Rendered from props.links / sampleData.js)
│   │   ├── <ul className={classes.linkGroup}> (e.g. "About Us", "Stories")
│   │   └── <ul className={classes.linkGroup}> (e.g. "Customer Care", "Help")
│   ├── <div className={classes.callout}>
│   │   ├── <span className={classes.calloutHeading}> "Follow Us!"
│   │   ├── <p className={classes.calloutBody}>       (Lorem Ipsum / brand tagline)
│   │   └── <ul className={classes.socialLinks}>
│   │       ├── <Instagram size={20} />               (from react-feather)
│   │       ├── <Facebook size={20} />                (from react-feather)
│   │       └── <Twitter size={20} />                 (from react-feather)
│   └── <Newsletter />                  (../Newsletter/newsletter.js)
└── <div className={classes.branding}>
    ├── <ul className={classes.legal}>
    │   ├── <li className={classes.terms}>   "Terms of Use"
    │   └── <li className={classes.privacy}> "Privacy Policy"
    ├── <p className={classes.copyright}>    {copyrightText} (From Magento Backend)
    └── <Link to="/">
        └── <Logo />                    (../Logo/logo.js)
```

---

## 2. Internal Sub-Components Breakdown

| Element / Sub-Component | Source File Path | Responsibility |
| :--- | :--- | :--- |
| `linkGroups` | `@magento/venia-ui/lib/components/Footer/sampleData.js` | 2-column menu map for default store navigation links (e.g. About, Customer Service). |
| `callout` & `socialLinks` | Inline in `footer.js` (using `react-feather` icons) | Displays the "Follow Us!" header, brand bio text, and Instagram/Facebook/Twitter icons. |
| `<Newsletter />` | `@magento/venia-ui/lib/components/Newsletter/newsletter.js` | Email input form that submits a customer newsletter subscription mutation to Magento. |
| `legal` (`terms` & `privacy`)| Inline in `footer.js` | Footer links for Terms of Service and Privacy Policy. |
| `copyright` | Dynamically provided by `useFooter` | Displays copyright string configured in Magento Admin (`Content > Design > Configuration`). |
| `<Logo />` | `@magento/venia-ui/lib/components/Logo/logo.js` | Bottom brand logo linking to homepage. |

---

## 3. Peregrine Talons, State & GraphQL Operations

### 3.1 `useFooter` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Footer/useFooter.js`
* **GraphQL Query**: `storeConfigData` (fetches `copyright` and `store_name` from backend configuration)
* **Returns**:
  ```javascript
  const talonProps = useFooter();
  const { copyrightText } = talonProps;
  ```

### 3.2 `useNewsletter` Talon
* **Module Path**: `@magento/peregrine/lib/talons/Newsletter/useNewsletter.js`
* **GraphQL Mutation**: `subscribeEmailToNewsletter(email: String!)`
* **Usage**:
  ```javascript
  import { useNewsletter } from '@magento/peregrine/lib/talons/Newsletter/useNewsletter';

  const {
      formErrors,       // Array of validation errors (e.g. invalid email format)
      handleSubmit,     // Form submit handler
      isBusy,           // Boolean: true while mutation request is in-flight
      setFormApi        // Callback to register react-hook-form / inform form state
  } = useNewsletter({
      mutations: {
          subscribeMutation: SUBSCRIBE_TO_NEWSLETTER
      }
  });
  ```

---

## 4. How to UPDATE Footer Components (With Examples)

### Example 4.1: Updating Footer Links via `sampleData.js`
By default, Venia imports links from `sampleData.js`. You can override the default links passed to the Footer component:

Create your custom link map:
```javascript
// src/components/CustomFooter/myFooterLinks.js
export const CUSTOM_FOOTER_LINKS = new Map([
    [
        'company',
        new Map([
            ['About Us', '/about-us'],
            ['Careers', '/careers'],
            ['Sustainability', '/sustainability'],
            ['Press', '/press']
        ])
    ],
    [
        'help',
        new Map([
            ['Track Order', '/order-status'],
            ['Shipping & Returns', '/shipping-returns'],
            ['Help Center / FAQ', '/faq'],
            ['Contact Us', '/contact-us']
        ])
    ]
]);
```

Inject your custom link structure via Targetables in `local-intercept.js`:
```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const footer = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Footer/footer.js'
    );

    // 1. Add import for custom link map
    footer.addImport(
        "import { CUSTOM_FOOTER_LINKS } from '../../../../src/components/CustomFooter/myFooterLinks'"
    );

    // 2. Override defaultProps.links
    footer.insertBeforeSource(
        'Footer.defaultProps = {',
        '// Override default links\n'
    );

    footer.replaceJSX(
        'const { links } = props;',
        'const links = props.links && props.links.size > 0 ? props.links : CUSTOM_FOOTER_LINKS;'
    );
};
```

---

### Example 4.2: Styling & Theme Colors via CSS
In `src/index.css`:
```css
/* Custom Footer Styling */
footer[data-cy="Footer-root"] {
  background-color: #111827 !important;
  color: #f3f4f6 !important;
  padding: 60px 0 30px !important;
}

footer[data-cy="Footer-root"] a {
  color: #9ca3af !important;
  text-decoration: none;
  transition: color 0.2s ease;
}

footer[data-cy="Footer-root"] a:hover {
  color: #ffffff !important;
}
```

---

### Example 4.3: Customizing Social Media Links & Icons
Replace static placeholders in `footer.js` with active external links using `local-intercept.js`:

```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const footer = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Footer/footer.js'
    );

    // Replace social links list
    footer.replaceJSX(
        '<ul className={classes.socialLinks}>',
        `
        <ul className={classes.socialLinks}>
            <li>
                <a href="https://instagram.com/yourbrand" target="_blank" rel="noreferrer" aria-label="Instagram">
                    <Instagram size={20} />
                </a>
            </li>
            <li>
                <a href="https://facebook.com/yourbrand" target="_blank" rel="noreferrer" aria-label="Facebook">
                    <Facebook size={20} />
                </a>
            </li>
            <li>
                <a href="https://twitter.com/yourbrand" target="_blank" rel="noreferrer" aria-label="Twitter">
                    <Twitter size={20} />
                </a>
            </li>
        </ul>
        `
    );
};
```

---

## 5. How to REMOVE Components from Footer (With Examples)

### Example 5.1: Removing the "Follow Us" Callout Box
If you don't need the default callout text and social block:

```javascript
// local-intercept.js
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const footer = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Footer/footer.js'
    );

    // Remove the callout container
    footer.removeJSX('<div className={classes.callout} />');
};
```

---

### Example 5.2: Removing the Newsletter Subscription Box
If you are using a third-party newsletter (Klaviyo/Mailchimp) or wish to remove the default newsletter form:

```javascript
// local-intercept.js
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const footer = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Footer/footer.js'
    );

    // Remove the Newsletter component
    footer.removeJSX('<Newsletter />');
};
```

---

## 6. How to REPLACE Components (3 Architectural Approaches)

### Method A: Targetables (AST Interception)
* **Best for**: Small to medium tweaks to default footer sections.
* **File**: `local-intercept.js`

```javascript
const { Targetables } = require('@magento/pwa-buildpack');

module.exports = function localIntercept(targets) {
    const targetables = Targetables.using(targets);

    const footer = targetables.reactComponent(
        '@magento/venia-ui/lib/components/Footer/footer.js'
    );

    // Add Payment Methods Badge row at the bottom
    footer.addImport("import PaymentBadges from '../../../../src/components/PaymentBadges'");

    footer.insertAfterJSX(
        '<p className={classes.copyright}>{copyrightText || null}</p>',
        '<PaymentBadges />'
    );
};
```

---

### Method B: Webpack Aliasing (Global Replacement)
* **Best for**: Swapping `@magento/venia-ui/lib/components/Footer` across the entire codebase with your custom footer.
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

    config.resolve.alias['@magento/venia-ui/lib/components/Footer'] = path.resolve(
        __dirname,
        'src/components/MyCustomFooter'
    );

    return config;
};
```

---

### Method C: Root Layout Replacement (Cleanest for 100% Custom Themes)
* **Best for**: Full creative control over footer layout columns, responsive grids, accordion mobile menus, and design system tokens.
* **File**: `src/components/CustomLayout/CustomApp.js`

Mount your `<CustomFooter />` directly at the bottom of your root app layout.

---

## 7. Full Hands-On Example: Building a 100% Custom Footer

Here is a complete, copy-pasteable 5-column responsive custom footer featuring:
* Working Newsletter subscription with Apollo GraphQL mutation (`useMutation`) and Peregrine Toast alerts (`useToasts`)
* Organized Navigation Columns (Company, Quick Links, Customer Service, Account)
* Social Media links with hover micro-interactions
* Payment Badges & Dynamic Copyright year

### Step 1: Create `src/components/CustomFooter/CustomFooter.js`
```javascript
import React, { useState } from 'react';
import { Link } from 'react-router-dom';
import { useMutation, gql } from '@apollo/client';
import { useToasts } from '@magento/peregrine';
import classes from './customFooter.module.css';

const SUBSCRIBE_MUTATION = gql`
    mutation SubscribeNewsletter($email: String!) {
        subscribeEmailToNewsletter(email: $email) {
            status
        }
    }
`;

const CustomFooter = () => {
    const [, { addToast }] = useToasts();
    const [email, setEmail] = useState('');
    const [subscribe, { loading }] = useMutation(SUBSCRIBE_MUTATION);

    const handleSubscribe = async e => {
        e.preventDefault();
        if (!email.trim()) return;

        try {
            await subscribe({ variables: { email: email.trim() } });
            addToast({
                type: 'info',
                message: 'Thank you for subscribing to our newsletter!',
                timeout: 5000
            });
            setEmail('');
        } catch (err) {
            addToast({
                type: 'error',
                message: err.message || 'Subscription failed. Please try again.',
                timeout: 5000
            });
        }
    };

    return (
        <footer className={classes.footer}>
            <div className={classes.container}>
                {/* 5-Column Grid */}
                <div className={classes.grid}>
                    {/* Col 1: Brand & Newsletter */}
                    <div className={classes.col}>
                        <h3 className={classes.brandTitle}>
                            <span className={classes.highlight}>MY</span>STORE
                        </h3>
                        <p className={classes.brandDesc}>
                            Get 10% off your first order when you join our insider newsletter.
                        </p>
                        <form onSubmit={handleSubscribe} className={classes.newsletterForm}>
                            <input
                                type="email"
                                placeholder="Enter your email"
                                value={email}
                                onChange={e => setEmail(e.target.value)}
                                className={classes.newsletterInput}
                                required
                            />
                            <button
                                type="submit"
                                className={classes.newsletterBtn}
                                disabled={loading}
                            >
                                {loading ? '...' : '→'}
                            </button>
                        </form>
                    </div>

                    {/* Col 2: Support & Contact */}
                    <div className={classes.col}>
                        <h4 className={classes.heading}>Support</h4>
                        <ul className={classes.linkList}>
                            <li>123 Commerce St, New York, NY</li>
                            <li><a href="mailto:support@mystore.com">support@mystore.com</a></li>
                            <li><a href="tel:+18005550199">+1 (800) 555-0199</a></li>
                        </ul>
                    </div>

                    {/* Col 3: Customer Account */}
                    <div className={classes.col}>
                        <h4 className={classes.heading}>Account</h4>
                        <ul className={classes.linkList}>
                            <li><Link to="/login">My Account</Link></li>
                            <li><Link to="/login">Sign In / Register</Link></li>
                            <li><Link to="/cart">View Cart</Link></li>
                            <li><Link to="/order-history">Order History</Link></li>
                        </ul>
                    </div>

                    {/* Col 4: Quick Links */}
                    <div className={classes.col}>
                        <h4 className={classes.heading}>Quick Links</h4>
                        <ul className={classes.linkList}>
                            <li><Link to="/privacy-policy">Privacy Policy</Link></li>
                            <li><Link to="/terms-of-use">Terms of Use</Link></li>
                            <li><Link to="/faq">FAQ & Returns</Link></li>
                            <li><Link to="/contact">Contact</Link></li>
                        </ul>
                    </div>

                    {/* Col 5: Social & App */}
                    <div className={classes.col}>
                        <h4 className={classes.heading}>Follow Us</h4>
                        <div className={classes.socialIcons}>
                            <a href="https://facebook.com" target="_blank" rel="noreferrer" aria-label="Facebook">Facebook</a>
                            <a href="https://instagram.com" target="_blank" rel="noreferrer" aria-label="Instagram">Instagram</a>
                            <a href="https://twitter.com" target="_blank" rel="noreferrer" aria-label="Twitter">Twitter</a>
                        </div>
                    </div>
                </div>

                {/* Bottom Bar: Copyright & Payment Badges */}
                <div className={classes.bottomBar}>
                    <p className={classes.copyright}>
                        © {new Date().getFullYear()} MyStore Inc. All rights reserved.
                    </p>
                    <div className={classes.paymentBadges}>
                        <span className={classes.badge}>Visa</span>
                        <span className={classes.badge}>MasterCard</span>
                        <span className={classes.badge}>PayPal</span>
                        <span className={classes.badge}>Apple Pay</span>
                    </div>
                </div>
            </div>
        </footer>
    );
};

export default CustomFooter;
```

### Step 2: Create `src/components/CustomFooter/customFooter.module.css`
```css
.footer {
  background: #000000;
  color: #ffffff;
  padding: 64px 0 24px;
  font-family: var(--font-primary, sans-serif);
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 16px;
}

.grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr 1fr;
  gap: 32px;
  margin-bottom: 48px;
}

@media (max-width: 900px) {
  .grid {
    grid-template-columns: 1fr 1fr;
    gap: 24px;
  }
}

@media (max-width: 600px) {
  .grid {
    grid-template-columns: 1fr;
    gap: 28px;
  }
}

.col {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.brandTitle {
  font-size: 24px;
  font-weight: 800;
  letter-spacing: -0.5px;
  color: #ffffff;
  margin: 0;
}

.highlight {
  color: #db4444;
}

.brandDesc {
  font-size: 14px;
  color: #9ca3af;
  line-height: 1.5;
  margin: 0;
}

.newsletterForm {
  display: flex;
  position: relative;
  max-width: 240px;
}

.newsletterInput {
  width: 100%;
  background: transparent;
  border: 1.5px solid #ffffff;
  border-radius: 4px;
  padding: 10px 40px 10px 12px;
  color: #ffffff;
  font-size: 14px;
  outline: none;
}

.newsletterInput::placeholder {
  color: #6b7280;
}

.newsletterBtn {
  position: absolute;
  right: 8px;
  top: 50%;
  transform: translateY(-50%);
  background: transparent;
  border: none;
  color: #ffffff;
  font-size: 18px;
  cursor: pointer;
}

.heading {
  font-size: 18px;
  font-weight: 600;
  color: #ffffff;
  margin: 0;
}

.linkList {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: 10px;
  font-size: 14px;
  color: #9ca3af;
}

.linkList a {
  color: #9ca3af;
  text-decoration: none;
  transition: color 0.2s;
}

.linkList a:hover {
  color: #ffffff;
}

.socialIcons {
  display: flex;
  flex-direction: column;
  gap: 8px;
  font-size: 14px;
}

.socialIcons a {
  color: #9ca3af;
  text-decoration: none;
  transition: color 0.2s;
}

.socialIcons a:hover {
  color: #db4444;
}

.bottomBar {
  border-top: 1px solid #1f2937;
  padding-top: 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 16px;
}

.copyright {
  font-size: 13px;
  color: #6b7280;
  margin: 0;
}

.paymentBadges {
  display: flex;
  gap: 8px;
}

.badge {
  background: #1f2937;
  color: #d1d5db;
  font-size: 11px;
  font-weight: 600;
  padding: 4px 8px;
  border-radius: 4px;
}
```

---

*Authored for Magento 2 PWA Studio Storefront Development.*
