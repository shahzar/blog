---
title: "When the standard mobile OIDC flow doesn't fit"
description: 'Integrating biometric-required national ID providers in mobile apps — UAE Pass, BankID, Singpass, and others that break the assumptions of ASWebAuthenticationSession and Chrome Custom Tabs.'
pubDate: 2026-05-18
tags: [mobile, oauth, kotlin, android]
categories: [Mobile]
toc: true
---

*Integrating biometric-required national ID providers in mobile apps*

Most "Sign-in with X" tutorials on mobile look the same: open the authorize URL in a system-managed browser, let the OS route the redirect back to your app, exchange the code for tokens. Libraries like AppAuth, kalinjul-oidc, and the platform-native APIs (`ASWebAuthenticationSession` on iOS, Chrome Custom Tabs on Android) all assume the same shape, and when the IdP cooperates, you get a clean ~50-line integration.

Then you get assigned to integrate a national digital identity system. UAE Pass. BankID. Singpass. ID.me. NemID/MitID. Various eIDAS-tied or eKYC providers. Many of them look OIDC-shaped on the outside but break the assumptions of the standard mobile flow on the inside — and your "afternoon of work" estimate balloons into a sprint.

This post is about what to do then. The concrete example is UAE Pass (we just shipped a native integration for it in a Kotlin Multiplatform app), but the constraints and the choices generalize.

## The standard mobile OIDC flow (a reminder)

What `ASWebAuthenticationSession` / Custom Tabs assume:

1. App opens the authorize URL in a sandboxed system browser
2. User authenticates (often single-sign-on via shared cookies)
3. IdP 302s to your `redirect_uri` — a custom scheme or universal link your app owns
4. OS routes that URL back to your app
5. App exchanges the auth code for tokens

This works because the OS handles the return leg natively (URL scheme registration, Universal Links / App Links), the sandboxed browser shares cookies with Safari/Chrome for SSO, and nothing in the flow crosses out of the browser.

It implicitly assumes:
- The IdP completes the entire authentication inside the browser
- The final redirect can be a custom scheme or universal link your app owns
- Nothing in the flow hands off to another native app

When all three hold, the standard pattern is excellent. When they don't, you have a project on your hands.

## The shape of a "non-compliant" IdP

The pattern that breaks standard tools: the IdP requires biometric authentication, which it implements by handing off to a separate native authenticator app it owns. That handoff is where the cracks appear.

The IdP's authorize page detects the authenticator app is installed and tries to invoke it via a custom-scheme deep link (`uaepass://...`, `bankid://...`, etc.). The authenticator app does biometric, then needs to return control. *How* it returns control is what separates well-behaved IdPs from the rest.

Symptoms that you're dealing with one of "the rest":
- The integration guide explicitly tells you to use a WebView, or ships a vendor SDK that wraps one
- There's no authorize-endpoint parameter to set `successURL` — the IdP's page emits it server-side as HTTPS, so customizing it means intercepting and rewriting the deep link from a WebView (sandboxed system browsers can't see navigation events)
- Deep-link schemes are identical across environments — there's no `authn-test://` for staging
- The vendor SDK ships its own HTTP stack (Retrofit, OkHttp, Alamofire) that conflicts with yours
- Multiplatform support is uneven: the Android AAR is mature, the iOS pod is a thin afterthought, the docs assume you're on one stack

UAE Pass exhibits all five. So do several other national IdPs.

## The three return-path patterns

Once the authenticator app finishes biometric, it calls `openURL(...)` with some URL. That URL determines how cleanly control returns to your app — and the IdP controls the shape of that URL unless you intervene.

**Pattern A — HTTPS resume URL on the IdP's domain (the default for most "non-compliant" IdPs)**

- Authenticator → `openURL("https://idp.com/resume?session=XYZ")`
- OS opens it in Safari/Chrome (not your sandboxed session — that's already dead)
- IdP's resume page runs in Safari, eventually 302s to your `redirect_uri`
- OS routes the custom scheme back to your app
- **UX:** brief Safari/Chrome chrome flash on return
- **Setup needed:** nothing, this is the IdP's default

**Pattern B — `successURL` is a custom scheme pointing at your app**

- The IdP lets you specify `successURL=myapp://resume?broker_state=XYZ`
- Authenticator → `openURL("myapp://resume?...")`
- OS routes directly to your app
- Your app re-opens a browser to hit the IdP's HTTPS resume URL with the broker state, eventually lands at `redirect_uri`
- **UX:** no flash
- **Setup needed:** IdP support for custom-scheme `successURL`

**Pattern C — `successURL` is a Universal Link / App Link on a domain you own**

- `successURL = https://yourdomain.com/resume?...` with `apple-app-site-association` / Digital Asset Links files served on that domain claiming the path
- OS sees "this HTTPS URL belongs to App X" → routes directly to your app
- Same continuation as Pattern B
- **UX:** cleanest
- **Setup needed:** domain ownership + association files + IdP lets you supply the URL

Pecking order: **C > B > A** for UX. C is what Sign-in with Apple and polished bank apps use. Most national IdPs default to A.

## What you actually get out of the box

If you just open the authorize URL in `ASWebAuthenticationSession` / Custom Tabs and accept whatever the IdP does, you get Pattern A. It works. Auth completes. Tokens are issued. Users are logged in.

What you pay for it:
- A visible Safari/Chrome chrome flash returning from the authenticator
- A second of "where am I?" for the user
- Whatever custom UI you wanted on the auth screens gets browser chrome instead

For many products, this is fine. Enterprise-internal apps, government-facing apps, B2B tools — ship Pattern A and move on. The "Safari flash" reads worse on paper than it does in practice.

For consumer-facing products where the auth screen is part of the first impression, or where the flash generates support tickets, you'll want Pattern B or C.

## When you'd actually leave the standard flow

Reasons to abandon `ASWebAuthenticationSession` / Custom Tabs:

1. **UX**: you can't accept the browser chrome flash
2. **Bugs in the IdP**: e.g., the authorize endpoint emits the production deep-link scheme even from the staging server, routing your staging build to the production authenticator
3. **UI control**: you want to brand the auth screens or suppress browser chrome entirely
4. **Embedded intermediate steps**: term acceptance, OTP, etc. that should feel native

Item 2 is the killer. We hit it on UAE Pass: staging URLs emitted `uaepass://` (the prod authenticator's scheme) instead of `uaepassstg://`. Inside a sandboxed system browser, you cannot intercept that — the navigation event is invisible to your app. The only way to rewrite the scheme at runtime is from inside a WebView you control.

If your reason for leaving the standard flow is purely UX polish, do the cost-benefit math first. If it's an IdP bug that makes the standard flow unusable, you have no choice.

## The escape hatch: WebView interception

Once you're hosting the auth flow in your own WebView, you get:
- Visibility into every navigation request the page tries to make
- The ability to cancel a navigation before it fires
- Control over the UI surrounding the WebView (loading states, error screens, branding)

You pay for it with:
- Hand-rolled lifecycle: back press, dismissal, errors, timeouts
- Two implementations (Android `WebView`, iOS `WKWebView`) with subtly different APIs
- Re-implementing what the system gives you for free: cookie management, security boundaries, redirect handling
- Audit/compliance scrutiny — some regulators dislike custom WebViews for sensitive flows

Most teams underestimate the second column. We ended up with ~600 lines of platform code for what's conceptually a 50-line integration. Almost all of it is boilerplate around state machines for "user came back from the authenticator," "user cancelled mid-auth," "WebView errored," "lifecycle observer fires while a continuation is pending," etc. None of it is core to auth itself.

## The interception pattern in detail

Here's the trick that turns a Pattern-A IdP into something closer to Pattern B at runtime.

The IdP's page tries to invoke `authn://digitalid?successURL=https://idp.com/resume?...`. Before that fires, your WebView's navigation delegate sees it. You:

1. Cancel the navigation
2. Extract the `successURL` value
3. Build a new deep link with `successURL` rewritten to your app's scheme: `myapp://resume?status=success&url=<encoded-original>`
4. Hand the rewritten URL to the OS to launch the authenticator app
5. Stash the original `successURL` somewhere
6. When the authenticator app returns control via `myapp://resume?...`, pull the original URL out and load it in *the same WebView*
7. The IdP's resume page runs inside your WebView, completes the broker flow, eventually navigates to your `redirect_uri`
8. Your WebView intercepts the `redirect_uri` navigation, extracts the auth code, dismisses the WebView, exchanges the code for tokens

The user sees: your app → authenticator app → your app. No browser flash, no context switch.

**iOS interception (WKNavigationDelegate):**

```kotlin
override fun webView(
    webView: WKWebView,
    decidePolicyForNavigationAction: WKNavigationAction,
    decisionHandler: (WKNavigationActionPolicy) -> Unit,
) {
    val url = decidePolicyForNavigationAction.request.URL ?: return
    val scheme = url.scheme

    if (scheme == "authn") {
        decisionHandler(WKNavigationActionPolicyCancel)
        handleAuthenticatorDeepLink(url)
        return
    }
    if (url.absoluteString!!.startsWith(redirectUri)) {
        decisionHandler(WKNavigationActionPolicyCancel)
        extractCodeAndFinish(url)
        return
    }
    decisionHandler(WKNavigationActionPolicyAllow)
}
```

**Android interception (WebViewClient):**

```kotlin
override fun shouldOverrideUrlLoading(
    view: WebView,
    request: WebResourceRequest,
): Boolean {
    val url = request.url
    if (url.scheme == "authn") {
        handleAuthenticatorDeepLink(url)
        return true
    }
    if (url.toString().startsWith(redirectUri)) {
        handleRedirectUri(url)
        return true
    }
    return false
}
```

Both snippets omit edge cases that consume real lines in production:

- **Android** fires `ERR_UNKNOWN_URL_SCHEME` on `onReceivedError` for custom schemes the system can't route. You need to ignore those for non-fatal cases and treat them as the redirect URI for your own scheme.
- **iOS** sometimes routes failed custom-scheme loads through `didFailProvisionalNavigation` instead of `decidePolicyForNavigationAction`, so you need a fallback path there too.
- **Both** need an app-resume hook (`onResume` on Android, `UIApplicationDidBecomeActiveNotification` on iOS) to know when control returned from the authenticator app and load the stashed `successURL`.

These gotchas are where the bulk of the implementation lines go.

## What to ask your IdP for

If you have any leverage with the IdP's product team — you're integrating a national system in a tier-1 market, or you represent a large user base — these requests would let you skip the WebView entirely:

1. **Expose `successURL` as an authorize-endpoint parameter.** Let the calling SP supply where to return. Standard practice for OIDC brokers.
2. **Accept custom-scheme `successURL` values.** Gets you to Pattern B without owning a domain.
3. **Use per-environment deep-link schemes** (`authn://` for prod, `authn-test://` for staging). Lets staging builds coexist with prod.
4. **Document a Universal Link / App Link path.** If they insist on HTTPS, let it be a domain *you* control.

Realistically, IdPs that pre-date these conventions are slow to retrofit them — they have to coordinate with every SP that depends on the current behavior. But asking is free, and if enough integrators ask, eventually it lands on a roadmap.

## Trade-off summary

| Approach | Setup | UX | Reliability | When to pick |
|---|---|---|---|---|
| Pattern A (system browser defaults) | Minimal | Browser flash | High (OS owns edge cases) | Most products, enterprise apps |
| WebView rewrite to Pattern B | High | Seamless | Medium (you own edge cases) | Consumer UX, broken-IdP staging |
| Pattern C (Universal Links) | Medium (domain + assoc. files) | Seamless | High | When IdP lets you supply the URL |
| Vendor SDK | Low initially | Varies | Medium (version coupling) | High-quality SDK, single platform |

One KMP-specific note: the vendor SDK path often disqualifies itself on multiplatform projects. The SDKs are typically platform-native (Swift pod for iOS, Java AAR for Android) and don't share well with `commonMain` code. The hand-rolled approach, while more code, lets you push most logic into `commonMain` and isolate just the WebView/lifecycle bits in platform-specific modules.

## Closing

The standard mobile OIDC flow is excellent — when your IdP cooperates. When it doesn't, you have real choices to make, and none of them are free.

Two things I'd tell my past self before starting this:

1. **Don't reach for the WebView reflexively.** Try Pattern A first. The Safari flash is worse on paper than in person, and the system browser handles a long tail of edge cases for free.
2. **If you do need the WebView, budget for it properly.** It's not "an afternoon of work." It's a week of building a state machine that handles all the lifecycle weirdness of two apps tag-teaming a user's attention — and you'll find a new edge case on every device family for months.

The IdP you're integrating may be specific to one country, but the architectural question — "how do I cleanly integrate a third party that doesn't fit the standard tools?" — is universal.