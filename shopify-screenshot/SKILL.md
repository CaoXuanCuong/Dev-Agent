---
name: shopify-screenshot
description: Screenshot Shopify embedded app bằng Playwright MCP. Tự động login Shopify admin và chụp các page trong embedded app (iframe). Dùng cho UI review, QA, so sánh với Figma.
trigger: explicit
---

# Shopify Screenshot — Playwright MCP

## Known project config (do not ask user)

| Key | Value |
|-----|-------|
| Login URL | `https://caocuongxuan.myshopify.com/admin` |
| App base URL | `https://admin.shopify.com/store/caocuongxuan/apps/bloy` |
| Credentials file | ./.auth.json` |
| Output dir | `features/plans/screenshots/` |
| Persistent Chrome profile | `/home/pc/.cache/bloy-playwright-profile` |
| Known working dashboard screenshot | `features/plans/screenshots/dashboard.png` |

**Important learned context (2026-06-02):**
- Shopify can block Playwright MCP when the browser user agent is `HeadlessChrome`, even if `navigator.webdriver` is false. Symptom: page title `Just a moment...` and text `Your connection needs to be verified before you can proceed`.
- Do **not** waste time retrying headless. Use a headed Chrome profile and keep it persistent: `/home/pc/.cache/bloy-playwright-profile`.
- The merchant session and app grant are now stored in that profile. Reuse it for future screenshots.
- BLOY may redirect through `/apps/bloy/exitiframe` and then `/app/grant?...` before the app renders. Do not screenshot these intermediate pages.
- If Shopify shows `BLOY needs access to:` with buttons `Cancel` and `Update`, click `Update`, wait for redirect back to `/apps/bloy`, then screenshot.

---

## Step 0 — Load credentials

Read `./.auth.json`.

Format:
```json
{ "email": "...", "password": "..." }
```

- **File exists** → use the credentials silently, do not ask user.
- **File missing** → ask user for email + password, then offer to save:
  > "Save credentials to `./.auth.json` so you don't need to enter them again? (file is gitignored)"
  If user agrees: write the file. Either way, proceed.

**Never** print password to chat after reading it.

---

## Step 1 — Check session

Call `browser_snapshot` and check the current URL:
- Contains `admin.shopify.com/store/caocuongxuan` → **skip login**, go to Step 3.
- Contains title `Just a moment...` or text `Your connection needs to be verified before you can proceed` → MCP/headless is blocked. Use the headed persistent Chrome fallback below.
- Otherwise → Step 2.

Login happens **at most once per conversation**.

---

## Step 2 — Login

Shopify uses a two-step flow (email first, then password on next screen):

```
1. browser_navigate   → https://caocuongxuan.myshopify.com/admin
2. browser_snapshot   → locate Email textbox ref
3. browser_fill_form  → Email field only
4. browser_click      → "Continue with email" button
5. browser_snapshot   → Password field now appears
6. browser_fill_form  → Password field
7. browser_click      → "Log in" button
8. browser_snapshot   → check result
9. IF "Not now" button exists (passkey enrollment) → browser_click it
10. Confirm URL is admin.shopify.com/store/caocuongxuan ✓
```

**2FA:** If a verification/OTP field appears at step 8 → ask user for code → fill → submit.

**Login failed** (URL still `/login`): if credentials came from `.auth.json`, ask user to re-enter and optionally update the file. Otherwise ask to re-enter password.

### Headed Persistent Chrome Fallback

Use this fallback immediately when MCP is blocked by Shopify verification or repeatedly returns to the login method screen.

Requirements already validated on this machine:
- Chrome binary: `/usr/bin/google-chrome-stable`
- Display exists: `DISPLAY=:0`
- Playwright package exists in npx cache, e.g. `/home/pc/.npm/_npx/705bc6b22212b352/node_modules/playwright`

Use persistent profile:
```text
/home/pc/.cache/bloy-playwright-profile
```

Launch headed Chrome through Playwright with:
```js
chromium.launchPersistentContext('/home/pc/.cache/bloy-playwright-profile', {
  channel: 'chrome',
  headless: false,
  viewport: { width: 1440, height: 1600 },
  args: ['--disable-blink-features=AutomationControlled'],
});
```

If manual login is required, keep Chrome open and let the user complete Shopify login/verification in that browser. Do not close the browser until the app URL is reached. Once login succeeds, the session persists in the profile and future screenshots should be much faster.

---

## Step 3 — Screenshot each route

Shopify App Bridge stretches the iframe to fill the viewport height. Set a tall viewport **before** navigating so the app renders all content without internal scrolling.

For each route:

```
1. browser_resize    → width: 1440, height: 1600
2. browser_navigate  → <APP_BASE_URL><route>
                       dashboard: no suffix, others: /analytics, /customers, etc.
3. handle redirects  → wait through /exitiframe; if /app/grant appears, click Update
4. browser_wait_for  → wait until actual app content is visible
5. browser_take_screenshot
                       filename: features/plans/screenshots/<name>.png
                       fullPage: true
```

No need to locate the iframe — screenshot the full page. Shopify admin chrome in the frame is fine for UI review.

**Do not screenshot until all are true:**
- URL contains `admin.shopify.com/store/caocuongxuan/apps/bloy`
- URL does **not** contain `/exitiframe`
- URL does **not** contain `/app/grant`
- Title is not `Log in` and not `Just a moment...`
- Body text contains route-specific app content, for dashboard any of: `Metrics`, `App status`, `Support channels`, `Loyalty`, `Rewards`

**Timeout:** wait up to 30s for normal routes. If the URL is still `/exitiframe` or `/app/grant`, handle those states rather than taking a blank screenshot.

### Grant Page Handling

If body text contains:
```text
BLOY needs access to:
```

Then click the `Update` button. This is expected after app permission changes. After clicking:
1. Wait 8-12s for redirect.
2. If still on `/app/grant`, click `Update` again only if the button is visible.
3. Wait until URL returns to `/apps/bloy`.
4. Only then screenshot.

### Known Console Noise

These messages were observed during successful dashboard render and should not block screenshot:
- CORS errors from `dev-cuongcxlenovo-bloy-api.dev-bsscommerce.com`
- React warnings about `fill-rule` / `clip-rule`
- React Router future flag warnings
- Polaris accessibility warnings

---

## Default routes (Bloy Loyalty)

| Name | Route suffix |
|------|-------------|
| Dashboard | *(none)* |
| Analytics | `/analytics` |
| Customers | `/customers` |
| Rewards Program | `/rewards_program` |
| Branding | `/branding` |
| Settings | `/settings` |
| Pricing | `/pricing_plans` |
| Onboarding | `/onboarding` |

---

## Output

Files saved to `features/plans/screenshots/`. Create directory if missing.

```
✓ Screenshots done: N/M pages
  - features/plans/screenshots/dashboard.png
  - features/plans/screenshots/analytics.png
✗ Failed: [route] — [reason]
```

---

## Error handling

| Situation | Action |
|-----------|--------|
| `.auth.json` missing | Ask user once, offer to save |
| Wrong password (from file) | Ask user to re-enter, offer to update file |
| 2FA prompt | Ask user for code, fill and submit |
| Session expired mid-flow | Re-run login (Step 2), then continue |
| Page load timeout | Retry with 6s; skip if still fails |
| Route 404 in app | Skip, note in report |
| `Just a moment...` / connection verification | Stop MCP retries; use headed persistent Chrome fallback |
| URL contains `/exitiframe` | Wait for redirect; do not screenshot |
| URL contains `/app/grant` or text `BLOY needs access to:` | Click `Update`, wait for `/apps/bloy`, then screenshot |
| Screenshot is Shopify shell blank/white | It was captured too early; wait for dashboard text like `Metrics` or `App status` |
