# Fixing Supabase Email Confirmation & Password Reset Deep Links (Expo React Native)

## Symptoms

- Custom scheme deep link (`projectName://?token_hash=...&type=signup`) worked in development but broke on the Android **release** build.
- Copying the "Confirm Your Email" link from **Gmail** showed a mangled URL:
  ```
  https://mail.google.com/mail/u/1/#m_7049260036663267569_ZgotmplZ?token_hash=...&type=signup
  ```
- Copying the same link from **Outlook** appeared correct:
  ```
  projectName://?token_hash=ahsbjhabsjhbjshdbjahs%26type%3Dsignup
  ```

---

## Root Cause #1 — Go template sanitization (`#ZgotmplZ`)

Supabase renders email templates using Go's `html/template` package. That package **auto-sanitizes any URL placed in an `href` attribute**, and only allows `http:`, `https:`, and `mailto:` schemes.

Your custom scheme (`projectName://`) isn't on that allow-list, so Go silently replaces the link with a placeholder: `#ZgotmplZ`.

- **Gmail** renders the actual (broken) `href` as sent — `#ZgotmplZ?token_hash=...` — and resolves it relative to `mail.google.com`, producing the odd URL you saw.
- **Outlook** appears to show a fallback/plain-text portion of the email rather than the actual button `href`, which is why it looked "correct" — but that's not what real mail clients (or Android/iOS) will actually open.

**This is not a Gmail-vs-Outlook quirk.** The underlying email HTML is broken for every recipient; Outlook just happened to display a version of the link that dodges the bug.

## Root Cause #2 — Doubled encoding (`%26` / `%3D`)

`%26type%3Dsignup` means the literal characters `&type=signup` got URL-encoded *into the value of* `token_hash`, instead of remaining a separate query parameter. If this is really what's stored in the template (not just an editor display artifact), your app receives one giant `token_hash` value with `&type=signup` glued onto the end — `type` never arrives as its own param, and `verifyOtp` fails even if the link opens correctly.

## Root Cause #3 — Android release build

- Changing `"scheme"` in `app.json` only takes effect after a **native rebuild** (`expo prebuild` / `eas build`). Testing in Expo Go during development does not prove the intent-filter exists in the compiled `AndroidManifest.xml`.
- On API 31+, intent-filters need `android:exported="true"` or the link silently fails to resolve, even if it worked in a dev client.

---

## The Fix — Email → Website → App pattern (Supabase's own recommendation)

Don't put the custom scheme directly in the email template. Route through a real `https://` URL first.

### 1. Supabase Dashboard → Authentication → URL Configuration

| Setting | Value |
|---|---|
| `Site URL` | `https://yourapp.com` (a real https URL) |
| `Additional Redirect URLs` | `projectName://*` |

### 2. Email templates — use plain `&`, route through your https domain

**Confirm signup:**
```html
<a href="https://yourapp.com/auth/confirm?token_hash={{ .TokenHash }}&type=signup&redirect_to=projectName://">
  Confirm Your Email
</a>
```

**Forgot password (recovery):**
```html
<a href="https://yourapp.com/auth/confirm?token_hash={{ .TokenHash }}&type=recovery&redirect_to=projectName://">
  Reset Your Password
</a>
```

No manual `%26` / `%3D` — just a plain `&`. Since the `href` is now `https://`, Go's sanitizer won't touch it.

### 3. Build a small web callback page (`/auth/confirm` on yourapp.com)

- Read `token_hash`, `type`, `redirect_to` from the query string.
- Show a **"Continue" button** (require an explicit user tap — do not auto-redirect).
- On tap: `window.location.href = redirect_to + "?token_hash=" + token_hash + "&type=" + type`

The tap step matters for a second reason: Gmail/Outlook Safe Links scanners "pre-click" links to check for malware, which can silently consume one-time tokens before the real user clicks. Requiring a tap on your own page (not the raw deep link) avoids that.

---

## App side (Expo React Native)

```ts
import * as Linking from 'expo-linking';
import { supabase } from './supabaseClient';

Linking.addEventListener('url', async ({ url }) => {
  const { queryParams } = Linking.parse(url);
  const { token_hash, type } = queryParams as {
    token_hash: string;
    type: 'signup' | 'recovery';
  };

  const { data, error } = await supabase.auth.verifyOtp({
    token_hash,
    type, // 'signup' for email confirm, 'recovery' for password reset
  });

  if (error) {
    console.error(error);
  } else if (type === 'recovery') {
    // navigate to your "set new password" screen — session is active now
  } else {
    // navigate to home / logged-in screen
  }
});
```

---

## Verifying the Android release build

1. Confirm the intent-filter exists in the compiled manifest:
   ```bash
   npx expo prebuild
   cat android/app/src/main/AndroidManifest.xml | grep -A5 "projectName"
   ```
   You should see `<data android:scheme="projectName"/>` with categories `DEFAULT` and `BROWSABLE`.

2. Test the deep link directly, bypassing email entirely:
   ```bash
   adb shell am start -W -a android.intent.action.VIEW \
     -d "projectName://auth/confirm?token_hash=test&type=signup" \
     com.yourpackage.name
   ```
   If this fails to open your app on the release APK, the manifest/build config is the problem — independent of Gmail/Outlook.

## Verifying on iOS

`expo-linking` reads the scheme from `app.json` automatically, but confirm `CFBundleURLTypes` is present after prebuild, then test:
```bash
xcrun simctl openurl booted "projectName://auth/confirm?token_hash=test&type=signup"
```

---

## Longer-term option: Universal / App Links

For the most robust setup (works even if the app isn't installed yet, and survives more email-client quirks), consider migrating from a custom scheme to **Universal Links (iOS)** / **App Links (Android)** using your `https://yourapp.com` domain instead of the callback-page approach above. This requires hosting `apple-app-site-association` and `assetlinks.json` on your domain, plus `associatedDomains` in `app.json` — Expo supports this natively.
