# Taobao Login Flow Reference
# v1.0 - Login detection, QR code handling, session persistence

## Login Detection (Multi-Factor)

Taobao login status must be verified using BOTH DOM elements and cookies. Checking only one is unreliable.

### JavaScript Detection Function

```js
() => {
  const getCookie = (name) => {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);
    if (parts.length === 2) return parts.pop().split(';').shift();
    return null;
  };

  const nickElement = document.querySelector('.site-nav-login-info-nick');
  const dnk = getCookie('dnk');           // Display Name cookie
  const tbToken = getCookie('_tb_token_'); // Taobao session token

  // ALL THREE must be present for confirmed login
  const isLoggedIn = !!nickElement && !!dnk && !!tbToken;

  return {
    isLoggedIn,
    username: nickElement?.textContent?.trim() || null,
    dnk: dnk ? decodeURIComponent(dnk) : null,
    hasTbToken: !!tbToken,
    hasNickElement: !!nickElement
  };
}
```

### Why Multi-Factor?

| Factor | Alone is Unreliable Because... |
|--------|-------------------------------|
| `.site-nav-login-info-nick` element | May not render on all page types |
| `dnk` cookie | Can persist after session expires |
| `_tb_token_` cookie | Can exist without full authentication |

Only when all three are present is the user truly logged in.

---

## Login Flow Steps

### Step 1: Navigate to Taobao Homepage

```
mcp__chrome-devtools__navigate_page → https://www.taobao.com
```

### Step 2: Check if Redirected to Login Page

After navigation, check the current URL. If it contains `login.taobao.com` or `login.tmall.com`, login is required.

### Step 3: Handle "Quick Entry" Button

If redirected to login page but the user was previously logged in, Taobao may show a "快速进入" (Quick Entry) confirmation button.

1. Use `take_snapshot` to look for a button containing "快速进入" text
2. If found, `click` it
3. Wait 3 seconds for redirect
4. Re-check login status

Known selectors for the button (may change):
- `#login > div.login-content.nc-outer-box > div > div.fm-btn > button`
- `button.fm-submit`
- Any button with text "快速进入"

### Step 4: QR Code Login

If no quick entry available:

1. Navigate to `https://login.taobao.com`
2. Tell the user: **"Please scan the QR code in Chrome to log in to Taobao."**
3. Poll login status every 5 seconds using the detection function above
4. Maximum wait: 3 minutes (36 polls)
5. Once `isLoggedIn === true`, proceed with scraping

### Step 5: Verify After Login

After login is detected:
1. Navigate to the target product page
2. If redirected to login again, the session may have issues — ask user to try again

---

## Session Persistence

### Chrome Profile Directory

Launch Chrome with a persistent user data directory:

```bash
open -na "Google Chrome" --args \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/taobao-chrome-profile \
  --no-first-run
```

The `--user-data-dir=/tmp/taobao-chrome-profile` flag ensures:
- Cookies persist across Chrome restarts
- Login state is maintained
- No need to re-scan QR code every session

### Key Advantage Over Playwright

With Chrome DevTools MCP:
- Chrome stays open between skill invocations
- Login state persists because browser is NOT closed
- Next `/taobao-scrape` call immediately detects existing login
- Only need QR scan once until cookies expire (typically days/weeks)

With old Playwright approach:
- Each Python process created a new browser context
- Session could be lost between runs
- Frequent re-authentication required

---

## Common Login Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `dnk` cookie exists but `nickElement` is null | Session partially expired | Navigate to taobao.com homepage first, then re-check |
| Redirected to login on product page but not on homepage | Product page requires higher auth level | Complete full QR login |
| "快速进入" button not found | Button selector changed | Use `take_snapshot` to find any button in the login form |
| Login detected but immediately redirected again | Cookies expired mid-session | Clear profile and login fresh |
