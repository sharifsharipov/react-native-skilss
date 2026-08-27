# Security — Secrets, Tokens, Storage, Transport, Auth

Read this when handling credentials, tokens, API keys, secure storage, network
transport, deep links, or authentication/authorization flows. Review every code
change against these — a leaked token or hardcoded key is a production incident.

## 1. Secrets & API keys — never hardcode

- No secrets in source, ever. Not in a `.ts` file, not in a committed `.env`, not
  in `app.json`, not in version control history.
- Inject build-time config through the project's mechanism (`react-native-config`,
  Expo's `extra` + EAS secrets, or a generated config module), and keep the
  values file out of git.
- **Anything shipped in the app bundle is extractable.** The JS bundle is not
  compiled away — a `.ipa`/`.apk` can be unzipped and the bundle read in minutes,
  and Hermes bytecode can be decompiled. `EXPO_PUBLIC_*` / any bundled env value
  is public by definition.
- Truly sensitive operations (signing, third-party secret keys, payment
  capture) belong on your backend, with the client calling your API. Client keys
  should be scoped, restricted by bundle id / SHA-1, and rotatable.
- Scan for accidental leaks: no keys in log output, error messages, analytics
  events, crash reports, or a screenshot in a bug report.

## 2. Token storage — use the Keychain/Keystore

- Store access/refresh tokens, PINs, and credentials in
  `react-native-keychain` or `expo-secure-store` (iOS Keychain / Android
  Keystore).
- **Never** in `AsyncStorage`, MMKV without encryption, Redux persist, or a plain
  file — those are readable on a rooted/jailbroken device and in some backup
  extractions.
- Keep tokens in memory only as long as needed; don't stash them in module-level
  mutable singletons that outlive the session.
- On logout, wipe secure storage, clear the query cache, reset stores, and drop
  in-memory copies. A "logged out" app still holding the previous user's cached
  data in a query cache is a data-leak bug.
- Biometric gating (`react-native-keychain`'s access control /
  `expo-local-authentication`) for high-value tokens where the product allows it.

## 3. Token lifecycle in the network layer

- Attach the access token via one interceptor, not by hand at each call site.
- Handle `401` centrally: attempt a single refresh, **queue** the in-flight
  requests, retry them once, and force logout if refresh fails. Guard against
  refresh storms — five parallel 401s must trigger one refresh, not five.
- Never log the `Authorization` header, request bodies with credentials, or
  responses containing PII.

```ts
let refreshPromise: Promise<string> | null = null;

http.interceptors.response.use(undefined, async (error) => {
  if (error.response?.status !== 401 || error.config._retried) throw error;
  error.config._retried = true;
  refreshPromise ??= refreshTokens().finally(() => { refreshPromise = null; });
  const token = await refreshPromise;      // all callers await the same refresh
  error.config.headers.Authorization = `Bearer ${token}`;
  return http(error.config);
});
```

## 4. Transport security & certificate pinning

- HTTPS only. Reject cleartext: Android `android:usesCleartextTraffic="false"`
  (and a network security config), iOS keep ATS enabled. Any localhost exception
  for development must not exist in the release build.
- For high-value apps, add **certificate/public-key pinning**
  (`react-native-ssl-pinning`, a native network security config, or the platform
  APIs) so a compromised or rogue CA cannot MITM traffic. Pin the SPKI hash, keep
  a backup pin, and have a rotation plan — an expired pin with no backup bricks
  the app.
- Never disable certificate verification, and never ship a debug flag that does.
- Remember the JS bundle is fetched over the network in dev — never point a
  release build at a dev server.

## 5. Authentication & authorization

- **Authentication** (who you are) and **authorization** (what you may do) are
  distinct. Enforce both on the **server**. Client-side checks are UX only — a
  hidden button is not a security control, and the screen is one navigation call
  away regardless.
- Use OAuth2/OIDC with **PKCE** for third-party login, through a system browser
  (`expo-auth-session` / `react-native-app-auth`) — never an embedded WebView,
  which can read the user's credentials and breaks the provider's trust model.
- Never embed a client secret in the app for an auth flow.
- Implement session timeout and re-authentication for sensitive actions
  (payments, profile changes).

## 6. Deep links & input handling

- **Validate every deep-link parameter before acting on it.** A link is
  attacker-controlled input: `myapp://transfer?to=…&amount=…` must not execute
  anything without the same authorization checks a normal flow gets.
- Register **App Links / Universal Links** with verified domains where possible;
  a custom scheme can be claimed by another app on Android.
- Treat all server and user input as untrusted. Validate at the boundary (zod
  schemas — `patterns.md` §4).
- Parameterize any local SQL (SQLite/WatermelonDB/op-sqlite) — never
  string-concatenate user input into a query.
- **WebView is the highest-risk component in a React Native app.** Restrict
  `originWhitelist`, disable JS if the content doesn't need it, never inject
  unescaped user content, never expose a broad native bridge to remote content,
  and validate `onMessage` payloads as untrusted.

## 7. Logging & data exposure

- No sensitive data in logs (tokens, passwords, full card/PAN, precise location,
  national IDs). Strip or mask before logging.
- Strip network logging in release. `console.log` statements survive into the
  production bundle unless the transform removes them — configure the babel
  transform, and gate anything deliberate behind `__DEV__`.
- Configure crash/analytics tooling to scrub PII; don't attach raw request
  bodies.
- Mask the app-switcher preview and block screenshots on screens showing secrets
  (`FLAG_SECURE` on Android; a privacy overlay on iOS backgrounding).
- Clipboard: don't copy secrets automatically; on Android 13+ mark sensitive
  clipboard content, and clear OTP-style copies after use.

## 8. Platform & release hardening

- Enable Hermes bytecode and JS minification in release; consider an obfuscation
  step for high-value apps, understanding it raises the bar rather than closing
  the door.
- Consider root/jailbreak and emulator detection for fraud-sensitive flows — as
  *defense in depth*, not a guarantee.
- **OTA updates are a code-delivery channel**: sign/verify them, restrict who can
  publish, and never let an OTA channel be settable from user-controllable input.
  An unprotected update channel is remote code execution.
- Keep dependencies patched: `npm audit` / Snyk in CI, and pay attention to
  native dependencies too — a vulnerable native SDK is your vulnerability.
- Keep the source maps for each release so production crash traces are readable.

## Review checklist

```
[ ] No hardcoded secrets / keys / tokens anywhere (JS bundle is readable)
[ ] Tokens & credentials only in Keychain/Keystore; wiped on logout
[ ] Query cache + stores cleared on logout
[ ] 401/refresh handled centrally, single-flight, no refresh storms
[ ] HTTPS enforced; cleartext disabled in release; pinning where warranted
[ ] Auth via system browser + PKCE; no client secret in the app
[ ] Authn & authz enforced server-side, not just hidden in the UI
[ ] Deep-link params validated before any privileged action
[ ] WebView locked down (origin whitelist, no unescaped injection)
[ ] No sensitive data in logs / crash reports / analytics / clipboard
[ ] console.* stripped in release; OTA channel signed and access-controlled
```
