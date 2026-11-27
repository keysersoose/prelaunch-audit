---
name: ui-launch-check
description: Check a web app's user interface before launch - many screen sizes, horizontal overflow, clipped text and buttons, tap targets of 44px, axe-core accessibility (serious or critical = fail), visible keyboard focus, page titles and meta tags, search engine settings (robots.txt, noindex, canonical, sitemap), broken links, CSP errors and slow pages. Use when someone says "check my UI before launch", "is it mobile friendly", "accessibility check", "run axe", "check responsive layout", "broken links", "will Google index my site" or "Lighthouse".
---

# UI launch check

Goal: real visitors use small phones, sideways phones, tablets, scaled laptops and big
monitors. AI-made CSS is usually checked at one width. This module finds the broken sizes,
the unreadable or unreachable controls, and the basic accessibility failures.

Test the production build (`npm run build && npm start`, or the live or staging URL),
not the dev server; dev mode hides some problems and relaxes security policies.
Do not use real accounts or real payment flows. Mock the backend where you can.

Tools (ask before installing). Install them in their own temp folder so nothing in the
app repo changes, and keep the script and all outputs there:

```bash
T="$(mktemp -d)"; cd "$T" && npm init -y >/dev/null && npm i playwright @axe-core/playwright && npx playwright install chromium
```

## Step 1. Start a fresh server, not a stale one

An old server on the same port means you test yesterday's build. Free the port first:
macOS/Linux `lsof -ti:3000 | xargs kill`; Windows `netstat -ano | findstr :3000` then
`taskkill /F /T /PID <pid>`. Build, start, and wait for `curl -sf http://localhost:3000/`.

## Step 2. Screenshots at many sizes

List the key pages (home, sign-in, the main task page, pricing, settings, legal pages).
Sizes to cover: 320x568, 360x740, 390x844, 430x932, 844x390 (phone sideways), 768x1024,
1024x768, 1280x720, 1366x768, 1440x900, 1920x1080, 2560x1440. Use 3 to 4 of them on every
small change and the full set before a release.

Save this as `ui-check.mjs` in `$T` (Node finds `playwright` next to the script, so it must
live there) and run `cd "$T" && node ui-check.mjs http://localhost:3000`. Screenshots are
written to `$T` too, never into the app folder:

```js
import { chromium } from 'playwright';
import AxeBuilder from '@axe-core/playwright';
const base = process.argv[2], pages = ['/', '/login'];          // add your pages
const sizes = [[320,568],[390,844],[844,390],[768,1024],[1366,768],[1920,1080]];
const browser = await chromium.launch(); const fails = [];
for (const [w,h] of sizes) {
  const ctx = await browser.newContext({ viewport:{width:w,height:h}, hasTouch: w <= 900 });
  await ctx.addInitScript(() => { window.cspHits = []; document.addEventListener(
    'securitypolicyviolation', e => window.cspHits.push(e.effectiveDirective + ' ' + e.blockedURI)); });
  for (const p of pages) {
    const page = await ctx.newPage(); await page.goto(base + p, { waitUntil: 'networkidle' });
    await page.screenshot({ path: `shot${p.replace(/\W/g,'_')}-${w}x${h}.png`, fullPage: true });
    const r = await page.evaluate((w) => {
      const out = [], d = document.documentElement;
      const inScroller = el => { for (let a = el.parentElement; a && a !== document.body; a = a.parentElement)
        if (['hidden','clip','auto','scroll'].includes(getComputedStyle(a).overflowX)) return true; return false; };
      if (d.scrollWidth > d.clientWidth + 1) out.push('horizontal overflow');
      for (const el of document.querySelectorAll('body *')) {
        const b = el.getBoundingClientRect(), cs = getComputedStyle(el);
        if (b.width && (b.right > d.clientWidth + 1 || b.left < -1) &&
            !inScroller(el)) out.push('outside viewport: ' + el.tagName + '.' + el.className);
        if (!el.children.length && el.textContent.trim() && el.scrollWidth > el.clientWidth + 2 &&
            ['hidden','clip'].includes(cs.overflowX) && cs.textOverflow !== 'ellipsis') out.push('clipped text: ' + el.textContent.trim().slice(0,40));
        if (parseFloat(cs.fontSize) < 11 && el.textContent.trim() && !el.children.length) out.push('font under 11px: ' + el.tagName);
      }
      if (w <= 900) for (const el of document.querySelectorAll('a[href],button,summary,select,input:not([type=hidden])')) {
        const b = el.getBoundingClientRect(); const inText = el.tagName === 'A' && el.closest('p,li');
        if (b.width && !inText && (b.width < 44 || b.height < 44)) out.push(`tap target under 44px ${Math.round(b.width)}x${Math.round(b.height)}: ${el.textContent.trim().slice(0,30) || el.getAttribute('aria-label')}`);
      }
      return [...new Set(out)].slice(0, 40);
    }, w);
    const axe = await new AxeBuilder({ page }).withTags(['wcag2a','wcag2aa','wcag21a','wcag21aa','wcag22aa']).analyze();
    const bad = axe.violations.filter(v => ['serious','critical'].includes(v.impact)).map(v => `axe ${v.impact}: ${v.id} (${v.nodes.length})`);
    const csp = await page.evaluate(() => window.cspHits);
    for (const m of [...r, ...bad, ...csp.map(c => 'CSP blocked ' + c)]) fails.push(`${w}x${h} ${p} ${m}`);
    await page.close();
  }
  await ctx.close();
}
await browser.close(); console.log(fails.join('\n') || 'no automatic failures'); process.exit(fails.length ? 1 : 0);
```

The element checks are rough filters, so expect some noise (for example elements inside
an intentional scrolling carousel). Review each hit, then **open a handful of screenshots
by eye**, especially the 320px and sideways-phone ones. Automatic checks miss ugliness.

Run the key sizes in WebKit and Firefox too (`playwright.webkit`, `playwright.firefox`).

## Step 3. What each check means and the usual fix

- **Horizontal overflow**: one wide image, table, code block or fixed width. Fix with
  `img, video { max-width: 100%; height: auto }`, `min-width: 0` on flex and grid children,
  `overflow-wrap: anywhere` on long words, and a scrolling wrapper for tables.
- **Clipped text or buttons**: a label or button hidden by `overflow: hidden`. Also check
  buttons inside dialogs and panels stay inside their box at 320px.
- **Tap targets**: WCAG 2.2 asks for at least 24x24 CSS px (criterion 2.5.8); 44x44 is the
  common goal for main actions. Fix with padding, not a bigger icon:
  `min-height: 44px; min-width: 44px; display: inline-flex; align-items: center; justify-content: center`.
- **axe-core**: serious and critical violations fail. Moderate and minor are warnings.
  Automatic scans find only a part of real problems, so this is a floor, not a pass.
  If axe cannot measure contrast (for example over images or gradients), check those
  spots by hand.
- **CSP violations** (CSP: the browser rule for which scripts and hosts may load): a
  blocked script, font or image host silently breaks a feature in production. Any entry
  is a fail.

## Step 4. Keyboard focus walk

1. Press Tab through each key page (in Playwright, `page.keyboard.press('Tab')` up to about
   200 times) and check the focused element visibly changes (outline, box-shadow, border
   or background). `outline: none` without a replacement is a fail.
2. The focus ring must not be cut off by a parent with `overflow: hidden`.
3. Main controls (the main form fields, the main submit button, the menu toggle) are
   reachable by Tab.
4. Dialogs: focus moves inside on open, Tab stays inside, Esc closes it, and focus returns
   to the button that opened it.
5. Do one pass by hand with the mouse unplugged, and if possible one flow with a screen
   reader (VoiceOver or NVDA).

## Step 5. Titles, meta tags, search settings and links

```bash
S=https://your-site.example
for p in / /login /pricing; do curl -s "$S$p" | grep -oiE '<title>[^<]*</title>|<meta name="description"[^>]*>|<html[^>]*lang="[^"]*"' | head -3; echo "-- $p"; done
curl -s "$S/robots.txt" | grep -iE '^(user-agent|disallow|sitemap)'
curl -sI "$S/" | grep -i 'x-robots-tag'
curl -s "$S/" | grep -oiE '<meta[^>]+name="robots"[^>]*>|<link[^>]+rel="canonical"[^>]*>'
curl -s "$S/sitemap.xml" | grep -oE '<loc>[^<]+' | head -5
npx -y linkinator "$S" --recurse      # broken links; pin a version once you have checked it
```

Each page needs a unique `<title>`, a meta description, `lang` on `<html>`, Open Graph tags
for pages people share, and a favicon. No link returns 404 or 500.

Production fails if robots.txt has `Disallow: /` for all agents, if there is a `noindex`
meta tag or `X-Robots-Tag: noindex` header, if the canonical link points to a preview,
staging, `*.vercel.app`, `*.netlify.app` or `*.lovable.app` address, or if sitemap URLs use
`localhost` or a staging domain. For staging and preview it is the opposite: they should
be noindex.

## Step 6. Speed

```bash
# $T is the temp folder from the top (set it again if your shell was restarted)
npx -y lighthouse "$S/" --only-categories=performance,accessibility,seo,best-practices \
  --output=html --output-path="$T/lighthouse-home.html" --chrome-flags="--headless"
```

Run in a clean profile (not with extensions). Look at Largest Contentful Paint (LCP: how
long the main content takes to show), total JavaScript size and image sizes. Run once with
mobile settings (the default) and note the scores. Scores move between runs; look at the
big problems, not one number.

## Severity guide

| Finding | Severity |
|---------|----------|
| Main action (sign-up, pay, submit) clipped or unreachable at a common size | High |
| axe-core critical violation on a key page | High |
| CSP blocks a script or service on a key page | High |
| Production blocks search engines | High if the launch depends on search, else Medium |
| axe-core serious violation | Medium |
| Horizontal overflow on a phone size | Medium |
| No visible keyboard focus, or a dialog traps focus wrongly | Medium |
| Tap targets under 44px on main actions (under 24px is a WCAG fail) | Medium |
| Broken internal links | Medium |
| Very slow home page on mobile (LCP over about 4 s) | Medium |
| robots.txt or sitemap missing | Low |
| Missing titles, descriptions, `lang` or social tags | Low |
| Small fonts, small layout glitches | Low |

## What to report

- The pages and sizes tested, browsers used, and whether it was the production build.
- The automatic failures grouped by page and size, after you removed false hits.
- Five to ten screenshot file names (in the temp folder) that show real problems.
- axe-core serious and critical counts per page.
- Keyboard walk, search settings, link check results and Lighthouse numbers for the home page.
