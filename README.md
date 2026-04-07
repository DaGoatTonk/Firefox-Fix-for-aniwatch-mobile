# Firefox/Brave/Kiwi/Extension supporting browsers-Fix-for-aniwatch-mobile
A script for Firefox/Kiwi/Brave/Extension supporting mobile browsers with the Tampermonkey extension, its primary use is to intercept and redirect the requests from and to megacloud.blog to megacloud.tv


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

### Step 1 - Install Firefox/Kiwi/Brave/etc. for Android

Download from the Google Play Store if you don't have it already.

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
// @name         MegaCloud Domain Redirect + Fullscreen Fix
// @namespace    http://tampermonkey.net/
// @version      2.0
// @description  Fixes megacloud.blog → megacloud.tv and restores fullscreen
// @match        *://*.aniwatchtv.to/*
// @match        *://*.megacloud.blog/*
// @grant        none
// @run-at       document-start
// ==/UserScript==

(function () {
  'use strict';

  const OLD = 'megacloud.blog';
  const NEW = 'megacloud.tv';
  const re = /megacloud\.blog/g;

  const rewrite = str => typeof str === 'string' ? str.replace(re, NEW) : str;

  // ─────────────────────────────
  // HARD REDIRECT IF ON OLD DOMAIN
  // ─────────────────────────────
  if (location.hostname.includes(OLD)) {
    location.replace(location.href.replace(OLD, NEW));
    return;
  }

  // ─────────────────────────────
  // NETWORK LAYER PATCHES
  // ─────────────────────────────

  const _fetch = window.fetch;
  window.fetch = function (input, init) {
    if (typeof input === 'string') input = rewrite(input);
    else if (input instanceof Request) input = new Request(rewrite(input.url), input);
    return _fetch.call(this, input, init);
  };

  const _open = XMLHttpRequest.prototype.open;
  XMLHttpRequest.prototype.open = function (method, url, ...rest) {
    return _open.call(this, method, rewrite(url), ...rest);
  };

  const _WebSocket = window.WebSocket;
  window.WebSocket = function (url, protocols) {
    return new _WebSocket(rewrite(url), protocols);
  };

  const _Worker = window.Worker;
  window.Worker = function (url, options) {
    return new _Worker(rewrite(url), options);
  };

  const _sendBeacon = navigator.sendBeacon;
  navigator.sendBeacon = function (url, data) {
    return _sendBeacon.call(this, rewrite(url), data);
  };

  // ─────────────────────────────
  // DOM CREATION / ATTRIBUTE PATCHES
  // ─────────────────────────────

  const _createElement = document.createElement.bind(document);
  document.createElement = function (tag, ...rest) {
    const el = _createElement(tag, ...rest);

    if (tag.toLowerCase() === 'script') {
      const desc = Object.getOwnPropertyDescriptor(HTMLScriptElement.prototype, 'src');
      Object.defineProperty(el, 'src', {
        set(val) { desc.set.call(this, rewrite(val)); },
        get() { return desc.get.call(this); },
        configurable: true
      });
    }

    return el;
  };

  const _setAttribute = Element.prototype.setAttribute;
  Element.prototype.setAttribute = function (name, value) {
    return _setAttribute.call(this, name, rewrite(value));
  };

  // ─────────────────────────────
  // STRING INJECTION PATCHES
  // ─────────────────────────────

  const _eval = window.eval;
  window.eval = function (code) {
    return _eval.call(this, rewrite(code));
  };

  const _Function = window.Function;
  window.Function = function (...args) {
    return _Function(...args.map(rewrite));
  };
  Object.setPrototypeOf(window.Function, _Function);
  window.Function.prototype = _Function.prototype;

  const innerDesc = Object.getOwnPropertyDescriptor(Element.prototype, 'innerHTML');
  Object.defineProperty(Element.prototype, 'innerHTML', {
    set(val) { innerDesc.set.call(this, rewrite(val)); },
    get() { return innerDesc.get.call(this); },
    configurable: true
  });

  const _insertAdjacentHTML = Element.prototype.insertAdjacentHTML;
  Element.prototype.insertAdjacentHTML = function (pos, html) {
    return _insertAdjacentHTML.call(this, pos, rewrite(html));
  };

  // ─────────────────────────────
  // MUTATION OBSERVER (LAST LINE DEFENSE)
  // ─────────────────────────────

  function rewriteNode(node) {
    if (!node || !node.getAttribute) return;

    ['src', 'href', 'data-src'].forEach(attr => {
      const val = node.getAttribute(attr);
      if (val && val.includes(OLD)) {
        node.setAttribute(attr, rewrite(val));
      }
    });
  }

  new MutationObserver(mutations => {
    for (const m of mutations) {
      for (const n of m.addedNodes) {
        rewriteNode(n);
        if (n.querySelectorAll) {
          n.querySelectorAll(`[src*="${OLD}"],[href*="${OLD}"],[data-src*="${OLD}"]`)
            .forEach(rewriteNode);
        }
      }
    }
  }).observe(document.documentElement, { childList: true, subtree: true });

  // ─────────────────────────────
  // FULLSCREEN — HARD OVERRIDE FIX
  // ─────────────────────────────

  function forceFullscreen(video) {
    if (!video) return;

    if (video.requestFullscreen) return video.requestFullscreen();
    if (video.webkitRequestFullscreen) return video.webkitRequestFullscreen();
    if (video.webkitEnterFullscreen) return video.webkitEnterFullscreen(); // mobile fallback
  }

  document.addEventListener('click', function (e) {
    const btn = e.target.closest('[class*="full"], [id*="full"]');
    if (!btn) return;

    const video = document.querySelector('video');
    if (!video) return;

    // Kill site handler
    e.stopImmediatePropagation();
    e.preventDefault();

    forceFullscreen(video);
  }, true);

  // ─────────────────────────────
  // DEBUG HOOK (optional but useful)
  // ─────────────────────────────

  ['requestFullscreen', 'webkitRequestFullscreen', 'webkitEnterFullscreen'].forEach(fn => {
    const proto = Element.prototype;
    if (proto[fn]) {
      const orig = proto[fn];
      proto[fn] = function (...args) {
        console.log('[FS CALL]', fn);
        return orig.apply(this, args);
      };
    }
  });

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
