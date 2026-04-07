# Firefox/Brave/Kiwi/Extension supporting browsers-Fix-for-aniwatch-mobile
A script for Firefox mobile with the Tampermonkey extension, its primary use is to intercept and redirect the requests from and to megacloud.blog to megacloud.tv


A super easy fix with 5 steps.
tested and it works perfectly!


# MegaCloud Domain Redirect - Userscript

Automatically redirects all `megacloud.blog` requests to `megacloud.tv` on aniwatchtv.to, fixing broken video playback after the streaming CDN moved domains.

Works at the network level - intercepts `fetch()`, `XMLHttpRequest`, and DOM-injected elements before they fire, so videos load seamlessly without any manual action.

---

## Requirements

- Android phone or tablet
- Firefox for Android
- Tampermonkey extension, installed via Firefox Add-ons

iOS users: Use Orion Browser by Kagi instead of Firefox. It supports Firefox extensions natively. The rest of the steps are identical.

---

## Installation

### Step 1 - Install Firefox for Android

Download and install Firefox from the Google Play Store if you don't have it already.

### Step 2 - Install Tampermonkey

1. Open Firefox and navigate to `https://addons.mozilla.org/en-US/firefox/addon/tampermonkey/`
2. Tap Add to Firefox
3. Tap Allow when prompted for permissions

### Step 3 - Create the userscript

1. Tap the Tampermonkey icon in the Firefox toolbar. If you don't see it, tap the three-dot menu, then Add-ons, then Tampermonkey.
2. Tap Dashboard
3. Tap the + button to create a new script
4. Select all the placeholder code and delete it
5. Paste the full script below:

```javascript
// ==UserScript==
// @name         MegaCloud Domain Redirect
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Redirects megacloud.blog to megacloud.tv on aniwatchtv.to
// @author       You
// @match        *://*.aniwatchtv.to/*
// @match        *://*.megacloud.blog/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function () {
  'use strict';

  // If we landed directly on the old domain, bounce immediately
  if (location.hostname.includes('megacloud.blog')) {
    location.replace(location.href.replace('megacloud.blog', 'megacloud.tv'));
    return;
  }

  // Intercept fetch() calls made by the page
  const originalFetch = window.fetch;
  window.fetch = function (input, init) {
    if (typeof input === 'string') {
      input = input.replace(/megacloud\.blog/g, 'megacloud.tv');
    } else if (input instanceof Request) {
      input = new Request(input.url.replace(/megacloud\.blog/g, 'megacloud.tv'), input);
    }
    return originalFetch.call(this, input, init);
  };

  // Intercept XMLHttpRequest calls
  const originalOpen = XMLHttpRequest.prototype.open;
  XMLHttpRequest.prototype.open = function (method, url, ...rest) {
    if (typeof url === 'string') {
      url = url.replace(/megacloud\.blog/g, 'megacloud.tv');
    }
    return originalOpen.call(this, method, url, ...rest);
  };

  // Rewrite any <script src>, <iframe src>, <source src> already in the DOM
  // and watch for new ones injected later
  function rewriteNode(node) {
    if (!node || !node.getAttribute) return;
    ['src', 'href', 'data-src'].forEach(attr => {
      const val = node.getAttribute(attr);
      if (val && val.includes('megacloud.blog')) {
        node.setAttribute(attr, val.replace(/megacloud\.blog/g, 'megacloud.tv'));
      }
    });
  }

  const observer = new MutationObserver(mutations => {
    mutations.forEach(m => m.addedNodes.forEach(n => {
      rewriteNode(n);
      if (n.querySelectorAll) {
        n.querySelectorAll('[src*="megacloud.blog"],[href*="megacloud.blog"],[data-src*="megacloud.blog"]')
          .forEach(rewriteNode);
      }
    }));
  });

  observer.observe(document.documentElement, { childList: true, subtree: true });

})();
```

6. Tap Save

### Step 4 - Verify it's enabled

1. Go back to the Tampermonkey Dashboard
2. You should see MegaCloud Domain Redirect listed with a green toggle
3. Make sure the toggle is on

### Step 5 - Test it

1. Open Firefox and navigate to aniwatchtv.to
2. Pick any anime and start an episode
3. The video should load normally - the redirect happens silently in the background

---

## How it works

The script runs at `document-start`, before the page has begun rendering, and patches four layers of network access:

- `location.replace` catches direct navigations to `megacloud.blog` URLs
- `fetch()` override catches API and media requests made by the video player
- `XMLHttpRequest` override catches older-style AJAX calls
- `MutationObserver` catches `<script>`, `<iframe>`, and `<source>` tags injected into the DOM after load

The regex `megacloud\.blog` matches any subdomain automatically, so it won't break if the site uses multiple subdomains on the old domain.

---

## Other browsers

- Firefox on Android - this guide
- Kiwi Browser on Android - Chrome Web Store, install Tampermonkey, use the same script
- Orion Browser on iOS - Firefox Add-ons, install Tampermonkey, use the same script
- Safari on iOS - Userscripts app, use the same script

---

## Troubleshooting

Video still won't load after installing: make sure the script toggle is green in the Tampermonkey dashboard, then hard-refresh the page by closing the tab and reopening it. Also check that you're on aniwatchtv.to - the match rule won't fire on mirror domains.

Tampermonkey icon doesn't appear in toolbar: tap the three-dot menu, then Add-ons. Tampermonkey will be listed there. Tap it and then tap Dashboard.

The site updated and it broke again: open the script in Tampermonkey and update the replacement string from `megacloud.tv` to whatever the new domain is.

---

## License

MIT
