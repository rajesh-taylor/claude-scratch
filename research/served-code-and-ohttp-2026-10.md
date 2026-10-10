# Refueler Share: served-code integrity and OHTTP research brief

Compiled 2026-10-10. Research only.

**Confidence key:**
- **H**: primary source read in full this session.
- **M**: primary source seen only through a search-engine excerpt. This environment's network policy blocked freedom.press, securedrop.org, hacks.mozilla.org, blog.cloudflare.com, developers.cloudflare.com, rfc-editor.org, f-droid.org and zapstore.dev. Cloudflare docs and RFC text were read from their GitHub sources instead.
- **L**: secondary source, or our own inference. Inferences are marked *(inference)*.

Where a page shows no publication date, the date column reads "fetched 2026-10-10".

## Summary

- WEBCAT is a working **alpha** (since about March 2026). It runs as a Firefox extension only, and its spec is still marked WIP. It is the only system available today that blocks a page whose code doesn't match a signed manifest, but it protects nobody unless they have installed the extension. That excludes Vanadium users.
- WAICT is the effort to standardise the same idea inside browsers. It involves Mozilla, Cloudflare, FPF and Meta, has a prototype behind a pref in Firefox Nightly, and is a draft. That is the long-term route that needs no extension. WEBCAT-compatible discipline (static files, no inline JS, strict CSP, signed manifest) carries straight over to it.
- WEBCAT requires a fully static frontend, no inline JS, and a CSP whose `script-src` is limited to `'self'`/`'wasm-unsafe-eval'`. Three of our current pieces break that:
  - Turnstile, which needs `script-src https://challenges.cloudflare.com`.
  - Any Cloudflare-injected script.
  - Hosting at `/share/` on a shared origin, because enrollment applies to a whole domain, not a path.
- Cloudflare's injections can all be switched off. `Cache-Control: no-transform` blocks Web Analytics auto-injection and email obfuscation. Rocket Loader and challenge pages need zone settings.
- For GrapheneOS, only a WebView wrapper that bundles the same 27 files (or a native app) avoids loading fresh code on every visit. A TWA loads live code from the origin, so it gives no integrity benefit.
- OHTTP is standard (RFC 9458) and a Worker can technically act as the gateway: HPKE and OHTTP libraries in JS run on Workers, but none has been audited. The relay must not be Cloudflare:
  - Cloudflare would see IPs (its relay logs keep them about 124 days) and would also run the gateway.
  - Cloudflare's relay is enterprise-only, closed beta.
  - Fastly's relay is enterprise.
  - Payjoin relays only forward to gateways that advertise the BIP77 purpose.
- The biggest OHTTP catch is our own design *(inference)*. A 32 MiB part sent directly to a presigned URL reveals the client IP to our own storage, together with that upload's object key. Hiding the IP on "initiate" or "finalise" for the same upload is therefore close to worthless. OHTTP earns its keep only for calls that the server cannot tie to an upload, such as anonymous credential issuance.

## Decisions for Rajesh

1. **Move Share to its own origin (e.g. `share.refueler.io`).** WEBCAT enrolls a domain and the WAICT header applies to the whole origin. An integrity-checked `/share/` would drag the whole refueler.io site and POS into the same rules. **Recommend: yes, before any enrollment.**
2. **Turnstile on the enrolled page.** It cannot stay in the page: WEBCAT's CSP allows no third-party script, and frames only from other enrolled domains.
   - Option (a): run Turnstile on a small, non-enrolled "gate" page (separate origin). The gate exchanges the solve for blind-signed upload credentials, and the Share page then spends those. The gate never sees file keys.
   - Option (b): self-hosted proof-of-work (ALTCHA-style, served from `'self'`) plus rate limits.
   - Option (c): paid credentials (Cashu).
   - **Recommend (a) now, with (c) as an optional add-on.** This also feeds Topic 2.
3. **Canonical hash in our release manifest.** WEBCAT and WAICT v1 are both **SHA-256** only. **Recommend: SHA-256 (base64url) as canonical.** Add SHA-384 only if we also emit SRI attributes. Not worth it while the page is enrolled, so one hash is enough.
4. **Sigsum or Sigstore for WEBCAT signing.** Sigsum allows offline Ed25519 signing, needs no GitHub OIDC, and supports thresholds. Sigstore suits CI but FPF calls it "work in progress". **Recommend Sigsum**, signed by hand from the ship script.
5. **Signer set.**
   - Enroll two Ed25519 signers with `threshold: 1`: a primary hardware key, and a backup key kept offline in a separate place.
   - Changing the enrollment later is logged and subject to a cool-down, so get it right first time.
   - **Recommend 2 signers, threshold 1.** Move to 2-of-3 if a second person ever joins.
6. **Android app form.** **Recommend:** a minimal hand-written Android WebView wrapper that serves the 27 bundled files through `WebViewAssetLoader`. Its CSP should match the web build. On GrapheneOS the WebView is Vanadium's. Reasons:
   - A TWA would load live code from the server.
   - Capacitor works but adds a large dependency tree that would then need reproducible builds.
7. **APK signing key.** Use a separate key from the manifest key, generated offline, with its SHA-256 certificate fingerprint published in at least three places: the GitHub README, a refueler.io page, and the Zapstore/Nostr profile. **Recommend: yes.** Treat the key as permanent, because Android only accepts updates signed by the same key.
8. **WEBCAT timing.** **Recommend:**
   - Make Share WEBCAT-compliant now. That covers CSP, no inline JS, `no-transform`, and the manifest built in the ship script.
   - Enroll a *staging* subdomain during the alpha.
   - Don't market "verified" until WEBCAT reaches beta or WAICT ships in a stable browser.
9. **OHTTP go/no-go.** Only worth doing with a relay run by someone other than Cloudflare (and other than us). **Recommend:** defer until a partner relay exists. When it does, scope it to credential issuance and any reads not tied to an upload. Don't claim IP privacy for uploads or downloads.
10. **Gateway key configuration delivery.** RFC 9458 requires the gateway key configuration to be authenticated and the same for all clients. **Recommend:** pin it inside the signed release (in the manifest and the app bundle) and rotate it only with a release.
11. **Cloudflare zone settings for the Share host.** Turn off Web Analytics auto-injection, Email Obfuscation, Rocket Loader, Bot Fight Mode and any challenge or Under-Attack rules. Send `Cache-Control: no-transform` on HTML and JS. **Recommend: yes.** These settings are cheap and are needed for any integrity scheme.

## Topic 1 findings

### 1.1 WEBCAT: status and architecture

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| Status | Alpha. The repo says "experimental software and currently released as an alpha". There is no tagged release. | [github.com/freedomofpress/webcat](https://github.com/freedomofpress/webcat) | fetched 2026-10-10 | H |
| Alpha launch | Entered alpha testing with a Firefox extension and a decentralised enrollment system open to testers. | [freedom.press: Help SecureDrop test WEBCAT alpha](https://freedom.press/tech/news/help-securedrop-test-webcat-alpha/) | ~Mar 2026 | M |
| Funding | Ethereum Foundation "Trillion Dollar Security" grant for wallet and dApp frontends. Grant covers research into Chrome/Chromium support. | [blog.ethereum.org 1TS grant](https://blog.ethereum.org/en/2026/08/05/1ts-grant) | 2026-08-05 | M |
| Browsers | Firefox (MV2 extension on AMO) only. No Chromium, no mobile, no Tor Browser build found. | [webcat repo](https://github.com/freedomofpress/webcat) | fetched 2026-10-10 | H (Firefox) / L (rest) |
| Components | Enrollment chain (`webcat-infra-chain`, CometBFT), spec, extension, `webcat-cli`, sigsum-ts, cometbft-ts, sigstore-browser. | [webcat repo](https://github.com/freedomofpress/webcat) | fetched 2026-10-10 | H |
| Enrollment policy | Served at `/.well-known/webcat/enrollment.json`. It contains `type` (sigsum or sigstore), `signers` (Ed25519, base64url), `threshold`, compiled Sigsum `policy`, `max_age`, `cas_url` and `logs`. | [webcat-spec/server.md](https://github.com/freedomofpress/webcat-spec/blob/main/server.md) | fetched 2026-10-10 | H |
| Enrollment process | Oracles each fetch the policy file and post signed `Observe` transactions to a permissioned CometBFT chain. Validators are "trusted partner orgs". A quorum moves the change to pending, and it applies after a **cool-down** (`voting_config.delay`). To unenroll, serve 404 or 410. | [webcat-spec/enrollment.md](https://github.com/freedomofpress/webcat-spec/blob/main/enrollment.md) | fetched 2026-10-10 | H |
| Policy change | The new policy must stay byte-stable through the observation period. The old one SHOULD stay at `enrollment-prev.json` until clients converge. | [server.md §2](https://github.com/freedomofpress/webcat-spec/blob/main/server.md) | fetched 2026-10-10 | H |
| Manifest | Fields: `app`, `version`, `default_csp`, `files` (path → SHA-256 base64url), `default_index`, `default_fallback`, Sigsum `timestamp`, optional `wasm` hashes and per-path `extra_csp`. `signatures` maps signer key → Sigsum proof and must meet `threshold`. | [webcat-spec/manifest.md](https://github.com/freedomofpress/webcat-spec/blob/main/manifest.md) | fetched 2026-10-10 | H |
| Transparency | Each signature carries a Sigsum inclusion proof (witnessed log). The chain's signed block headers stop manifests being future-dated, so `max_age` expiry is enforceable. Snapshots go to a CDN hourly. | [enrollment.md](https://github.com/freedomofpress/webcat-spec/blob/main/enrollment.md) | fetched 2026-10-10 | H |
| Monitoring | Monitors pull manifests and all referenced files from the site's CAS (`cas_url`), so third parties can audit what was served. There is an open issue on monitoring and auditing. | [server.md](https://github.com/freedomofpress/webcat-spec/blob/main/server.md), [issue #222](https://github.com/freedomofpress/webcat/issues/222) | issue 2026-08-15 | H / M |
| Site requirements | "Frontend **must be fully static**", "**No inline JavaScript**", CSP delivered by HTTP header. Eleventy output plus plain ES modules fits. | [webcat-cli README](https://github.com/freedomofpress/webcat-cli) | fetched 2026-10-10 | H |
| CSP rules: scripts | `script-src` may only be `'none'`, `'self'` or `'wasm-unsafe-eval'`. Hashes, nonces and `unsafe-eval` are banned, so no third-party scripts. | [webcat-spec/csp.md](https://github.com/freedomofpress/webcat-spec/blob/main/csp.md) | fetched 2026-10-10 | H |
| CSP rules: other directives | If `default-src` isn't `'none'`, `object-src 'none'`, `worker-src` and `frame-src`/`child-src` are all required. Frames may point to external URLs **only if that domain is also enrolled**. `connect-src`, `img-src`, `font-src` and `media-src` are unrestricted. | [csp.md](https://github.com/freedomofpress/webcat-spec/blob/main/csp.md) | fetched 2026-10-10 | H |
| Domain scope | Enrollment is per (sub)domain under a registered zone. The number of subdomains per zone is capped (`max_enrolled_subdomains`). It is not per-path. | [enrollment.md](https://github.com/freedomofpress/webcat-spec/blob/main/enrollment.md) | fetched 2026-10-10 | H |
| Onion sites | Only clearnet TLS sites can enroll today. Private .onion enrollment is planned. | search excerpt from FPF/SecureDrop pages | 2026 | M |
| Deployments | Proof-of-concept ports: Jitsi, GlobaLeaks, Element, CryptPad, Standard Notes, Bitwarden. No public list of production enrolled sites found. | [IACR ePrint 2025/797](https://eprint.iacr.org/2025/797), [webcat-cli porting guides](https://github.com/freedomofpress/webcat-cli) | 2025; guides marked "outdated" | M |
| Known limits | Extension "might not yet provide the intended security guarantees". Protects only users who installed it. First visit before enrollment data loads is a trust-on-first-use window *(inference)*. A server that drops enrollment needs the cool-down to take effect, which is the anti-downgrade point. | [webcat repo](https://github.com/freedomofpress/webcat), [enrollment.md](https://github.com/freedomofpress/webcat-spec/blob/main/enrollment.md) | fetched 2026-10-10 | H / L |

### 1.2 Alternatives to WEBCAT

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| WAICT | Cross-vendor spec (Mozilla, Cloudflare, FPF, Meta) to build whole-app integrity and transparency into browsers. Prototype behind a pref in Firefox Nightly. | [Mozilla Hacks: Trustworthy JavaScript for the Open Web](https://hacks.mozilla.org/2026/05/trustworthy-javascript-for-the-open-web/) | 2026-05 | M |
| WAICT: mechanism | Opt-in with the `Integrity-Policy-WAICT-v1` header (`max-age`, `manifest`, mode `enforce`/`warn`/`report`). Manifest fields: `url_hashes`, `wasm_hashes`, `inline_js_hashes`, `eval_hashes`, `emergency_opt_out`. SHA-256 only in v1. External URLs may be listed. Cross-origin iframes are covered only if that origin opts in itself. | [waict-integrity-spec](https://github.com/waict-wg/waict-integrity-spec) | fetched 2026-10-10 (draft) | H |
| WAICT: Cloudflare | Cloudflare announced its part in the effort. | [Cloudflare blog: Improving the trustworthiness of JavaScript](https://blog.cloudflare.com/improving-the-trustworthiness-of-javascript-on-the-web/) | ~Oct 2025 | M |
| WAICT: open issues | An open issue covers possible downgrade protection. | [waict-integrity-spec #74](https://github.com/waict-wg/waict-integrity-spec/issues/74) | fetched 2026-10-10 | M |
| Meta Code Verify | Extension for Chrome, Firefox, Edge and Safari. Checks Facebook, Messenger, Instagram and WhatsApp Web JS/CSS against a published hash manifest. Shows a warning rather than blocking (WEBCAT blocks). The hash source of truth is hosted by Cloudflare. Works only for Meta's own sites. | [meta-code-verify](https://github.com/facebookincubator/meta-code-verify); press coverage via search | repo fetched 2026-10-10 | H / M |
| Chrome Isolated Web Apps | App packaged as a Signed Web Bundle (Ed25519 or ECDSA P-256), so no code is fetched live. Install only by enterprise policy on managed ChromeOS. Since Chrome 143, a Google-managed allowlist gates installs. Managed Windows support is listed for a future Chrome version (150 or 161; sources conflict). Nothing for unmanaged users or Android. | [developer.chrome.com IWA](https://developer.chrome.com/docs/iwa/introduction), [blink-dev allowlist PSA](https://groups.google.com/a/chromium.org/g/blink-dev/c/iTCPaBw6HxU/m/xSwr3FDWAgAJ), [Chrome Enterprise notes](https://support.google.com/chrome/a/answer/10314655) | 2024–2026 | M |
| SRI + CSP alone | Protects subresources only if the HTML is trusted. A compromised host that serves the HTML can rewrite the hashes. Fine for third-party CDN drift, useless against our threat *(inference from how SRI works)*. | — | — | L |
| Sigstore / Rekor | Transparency log for signatures, not an enforcement point in the browser. WEBCAT can use it as a signer type, but Sigstore enrollment is "a work in progress" and relies on CI OIDC identity. | [webcat-cli README](https://github.com/freedomofpress/webcat-cli) | fetched 2026-10-10 | H |
| Academic baseline | "Trust on Reload" (WWW 2026, CISPA): true E2EE in web apps is impossible without a way to verify client code before it runs. Five existing tools found to address the threat only partly. | [CISPA publication page](https://cispa.de/en/research/publications/213253-trust-on-reload-securing-browser-based-end-to-end-encryption) | 2026 | M |

### 1.3 Obstacles specific to our stack

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| Turnstile CSP | Needs `script-src https://challenges.cloudflare.com` and `frame-src https://challenges.cloudflare.com`, or nonce/`strict-dynamic`. All three are banned by WEBCAT. Cloudflare's challenge domain isn't WEBCAT-enrolled, so the frame route fails too. | [cloudflare-docs: turnstile/reference/content-security-policy](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/turnstile/reference/content-security-policy.mdx) | fetched 2026-10-10 | H |
| Turnstile pre-clearance | Pre-clearance mode fetches `/cdn-cgi/` on our domain to set `cf_clearance`. That response is Cloudflare-generated, not in our manifest *(inference: avoid on the enrolled host)*. | same | fetched 2026-10-10 | H / L |
| Captchas on WEBCAT sites | No WEBCAT doc or port addresses captchas. The ported apps have none in the verified frontend. | — | — | L (gap) |
| Web Analytics | Automatic setup injects `beacon.min.js` into every proxied page and subdomain in the zone, on by default for zones that used Browser Insights. It is blocked by `Cache-Control: public, no-transform` and can be disabled per site. | [cloudflare-docs: web-analytics/get-started](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/web-analytics/get-started/index.mdx), [faq](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/web-analytics/faq.mdx) | fetched 2026-10-10 | H |
| Email obfuscation | On by default at signup. Rewrites HTML and injects `email-decode.min.js`. Skipped when `Cache-Control: no-transform` is set. Can be disabled per zone or with a Configuration Rule. | [cloudflare-docs: email-address-obfuscation](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/waf/tools/scrape-shield/email-address-obfuscation.mdx) | fetched 2026-10-10 | H |
| Rocket Loader | Rewrites inline and external scripts to defer them and needs CSP changes. Available on all plans. Must be off on the Share host. | [cloudflare-docs: rocket-loader](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/speed/optimization/content/rocket-loader/index.mdx) | fetched 2026-10-10 | H |
| Challenge pages | Cloudflare challenge pages are served instead of our HTML, and their CSP cannot be overridden. On an enrolled host they would fail verification and the page would be blocked. Disable Bot Fight Mode, Under Attack mode and challenge rules on that host *(inference)*. | [Turnstile CSP note](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/turnstile/reference/content-security-policy.mdx) | fetched 2026-10-10 | H / L |

### 1.4 Installable app for GrapheneOS

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| TWA | Content is "rendered by the user's browser", fetched from the website, with app↔site ownership checked by Digital Asset Links. It is **fresh code per visit**, so no integrity gain. Vanadium's TWA support is unconfirmed. | [Android: Trusted Web Activities overview](https://developer.android.com/develop/ui/views/layout/webapps/trusted-web-activities) | fetched 2026-10-10 | M |
| WebView + bundled assets | `WebViewAssetLoader` serves files packaged in the APK over `https://appassets.androidplatform.net/…`, keeping same-origin rules. The code ships inside the signed APK, so no fresh code per visit. | [androidx WebViewAssetLoader](https://developer.android.com/reference/kotlin/androidx/webkit/WebViewAssetLoader) | fetched 2026-10-10 | M |
| GrapheneOS WebView | Vanadium provides the system WebView, so wrapper apps inherit part of its hardening. The WebView cannot be swapped. | [GrapheneOS forum](https://discuss.grapheneos.org/d/8656-can-we-change-the-default-webview), [Vanadium repo](https://github.com/GrapheneOS/Vanadium) | forum post | L |
| Capacitor / native | Capacitor also bundles assets locally (same property as above) but adds a large npm and Gradle dependency surface to make reproducible. Native Kotlin means rewriting the crypto *(inference)*. | — | — | L |
| F-Droid reproducible standard | Verifies by copying the developer's signature onto F-Droid's own unsigned build and checking it validates (v2/v3 signatures cover every byte). If it matches, F-Droid can **publish the developer-signed APK**. | [f-droid.org Reproducible Builds](https://f-droid.org/en/docs/Reproducible_Builds/) | fetched via excerpt 2026-10-10 | M |
| Fingerprint publishing format | Package name plus SHA-256 of the signing certificate, as colon-hex. Verify with `apksigner verify --print-certs`. | [AppVerifier README](https://github.com/soupslurpr/AppVerifier), [Obtainium README](https://github.com/ImranR98/Obtainium) | fetched 2026-10-10 | H |
| Obtainium verification | Obtainium shows the signing certificate. The usual GrapheneOS flow is to check it against the developer's published fingerprint with AppVerifier. After the first install, Android itself refuses updates signed by a different key. No built-in pinning documented in the Obtainium README. | [Obtainium](https://github.com/ImranR98/Obtainium), [GrapheneOS forum](https://discuss.grapheneos.org/d/24381-wireguard-verify-apk-certificate) | fetched 2026-10-10 | H / L |
| Zapstore publishing | `zsp` CLI plus a `zapstore.yaml` in the repo; the relay checks the pubkey in it before whitelisting. Signing: nsec, NIP-46 bunker or NIP-07. Free, no registration. Pulls arm64-v8a APKs from GitHub releases. Extracts the certificate fingerprint. `zsp identity --link-key` ties the APK signing certificate to a Nostr identity. `--commit` flag for reproducible builds. | [zapstore/zsp README](https://github.com/zapstore/zsp) | fetched 2026-10-10 | H |
| Zapstore FAQ | Publishing is free and needs only a Nostr keypair. | [zapstore.dev FAQ](https://zapstore.dev/docs/faq) | excerpt | M |

### 1.5 Release discipline

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| Minimal manifest | Use WEBCAT's format as our release manifest: `app` (repo URL), `version`, `files` (path → SHA-256 base64url), `default_csp`, `default_index`, `default_fallback`, Sigsum timestamp and signatures. Add the git commit and the APK cert fingerprint as our own fields. Unknown fields may break WEBCAT canonical verification, so keep them in a sidecar file *(inference)*. | [manifest.md](https://github.com/freedomofpress/webcat-spec/blob/main/manifest.md) | fetched 2026-10-10 | H / L |
| Signing tooling | `webcat-cli` (Node 20 or later) with Go `sigsum-key` and `sigsum-submit`. Sigsum is "better if you want offline, manual signing" but less convenient for automation. | [webcat-cli README](https://github.com/freedomofpress/webcat-cli) | fetched 2026-10-10 | H |
| Hardware key | Sigsum signs SSH-format Ed25519. Sigsum's key-mgmt repo documents a YubiHSM ssh-agent. Whether a YubiKey works is unverified: OpenPGP-card Ed25519 through gpg-agent's SSH support is plausible, but `ed25519-sk` (FIDO) produces a different signature format and is likely incompatible. | [sigsum.org/key-mgmt](https://beta.pkg.go.dev/sigsum.org/key-mgmt) | fetched via excerpt | M / L |
| Deploy consistency | During a deploy, a cached old file next to a new manifest (or the reverse) gives a mismatch, which shows as a false failure. Mitigations *(inference)*: (1) version the file paths per release (`/v1.4.0/…`) so old and new never mix; (2) `Cache-Control: no-cache` on the manifest and HTML; (3) keep the previous release's files served for at least `max_age`. Cloudflare Pages deploys are atomic per deployment *(not verified this session)*. | — | — | L |
| Ship-script gate | Refuse to deploy unless: hashes of built files = manifest; Sigsum proof present; CSP header in `_headers` = `default_csp`; `no-transform` present; no `<script>` without `src` in the HTML *(inference)*. | — | — | L |

### 1.6 Competitors

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| WhatsApp / Meta | **Verifiable web.** Code Verify extension with a Cloudflare-held hash manifest. Warns rather than blocks. | [meta-code-verify](https://github.com/facebookincubator/meta-code-verify) | 2022 launch; repo fetched 2026-10-10 | H |
| Proton | Web clients unverified. Proton engineers said at FOSDEM 2020 that open source is "necessary, but… not sufficient"; secure delivery is the hard part. Proton publishes **APK signing-certificate SHA-256 fingerprints**. | [FOSDEM 2020 talk](https://archive.fosdem.org/2020/schedule/event/dip_securing_protonmail/), [proton.me/support/verify-apks](https://proton.me/support/verify-apks) | 2020; fetched 2026-10-10 | M |
| Bitwarden | Web vault is served code. A WEBCAT proof-of-concept port exists, done by the WEBCAT researchers, not Bitwarden. No Bitwarden-run integrity mechanism found. | [IACR 2025/797](https://eprint.iacr.org/2025/797) | 2025 | M |
| Signal | **Native only** (desktop is a linked Electron app). No official statement found giving served-code risk as the reason. | [Computerworld](https://www.computerworld.com/article/1639074/encrypted-messaging-app-signal-now-available-for-desktops.html) | ~2016 | L |
| wormhole.app | Browser-based, 128-bit AES-GCM, key in the fragment. No code-integrity mechanism found. Exactly our threat model, unsolved. | [wormhole.app/security](https://wormhole.app/security) | fetched via excerpt | M |
| crypt.fyi | Apache-2.0, browser AES-256-GCM (README also mentions ML-KEM), strict CSP, npm CLI, self-hostable. No verification beyond CSP found. | [osbytes/crypt.fyi](https://github.com/osbytes/crypt.fyi), [Railway template](https://railway.com/deploy/Pmkrsc) | fetched via excerpt | M / L |
| Tresorit Send | Client-side encryption in the browser. No served-code integrity mechanism found. | [Tresorit blog](https://tresorit.com/blog/tresorit-opens-its-end-to-end-encrypted-file-sharing-service-to-the-public) | — | L |
| SwissTransfer | Appears to be server-side encryption only, not end-to-end. Unconfirmed against Infomaniak docs. | [europeanpurpose.com](https://europeanpurpose.com/alternative-to/wetransfer) | — | L |
| WEBCAT ports | CryptPad, Element, Jitsi, GlobaLeaks: porting guides exist, so these are the closest peers to copy from. | [webcat-cli](https://github.com/freedomofpress/webcat-cli) | fetched 2026-10-10 | H |
| Pattern | Nobody in file transfer ships enforced verifiable web code. Messaging and password managers either go native-first (Signal) or use extension-based checks (Meta). Getting into WEBCAT early would be a real differentiator *(inference)*. | — | — | L |

## Topic 2 findings

| Item | Finding | Source | Date | Conf. |
|---|---|---|---|---|
| Non-collusion | "The Oblivious Relay Resource cannot be operated by the same entity as the Oblivious Gateway Resource." Gateway and target can be co-located. | RFC 9458 §Security Considerations (read from [WG source](https://github.com/ietf-wg-ohai/oblivious-http)) | RFC Jan 2024 | H |
| Why not both Cloudflare | Our Worker is the gateway, so Cloudflare already terminates TLS and sees request content. Cloudflare's relay logs client IP, target, user-agent-type metadata and timestamp, kept "~124 days". Both halves would sit with one company and one legal order *(inference from the legal page)*. | [cloudflare-docs: ohttp-relay/reference/legal](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/ohttp-relay/reference/legal.mdx) | fetched 2026-10-10 | H |
| Cloudflare OHTTP Relay | Renamed from Privacy Gateway. **Enterprise plan, closed beta** for "select privacy-oriented companies". Fixed URL `privacy-relay.cloudflare.com/<gateway>`. No public price. | [cloudflare-docs: ohttp-relay index](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/ohttp-relay/index.mdx), [get-started](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/ohttp-relay/get-started.mdx) | fetched 2026-10-10 | H |
| Cloudflare OHTTP Gateway | New self-serve gateway, closed beta and waitlist, planned as a paid zone add-on. Announced Birthday Week 2026. | [Cloudflare blog](https://blog.cloudflare.com/announcing-cloudflare-ohttp-gateway/) | ~late Sep 2026 | M |
| Fastly | Runs an OHTTP relay for Google's Privacy Sandbox. Strips headers not required by the spec. No public self-serve price. | [Fastly press release](https://www.fastly.com/press/press-releases/google-selects-fastly-oblivious-http-relay-for-privacy-sandbox) | 2023-03-15 | M |
| Payjoin (BIP77) relays | Relays forward only to gateways advertising `/.well-known/ohttp-gateway?allowed_purposes` with the magic string `BIP77 454403bb-…`. That is how they avoid acting as open proxies. Example relays: `pj.benalleng.com`, `pj.bobspacebkk.com`, `payjoin.achow101.com`; directory `payjo.in`. Using them for Refueler would mean faking the BIP77 purpose, which is not acceptable. Only an explicit agreement with an operator would make it legitimate. | [BIP 77](https://github.com/bitcoin/bips/blob/master/bip-0077.md) (Draft, v0.2.0), [payjoin-cli README](https://github.com/payjoin/rust-payjoin/blob/master/payjoin-cli/README.md) | fetched 2026-10-10 | H |
| BIP77 uniform size | Encapsulated requests padded to a fixed **8192 bytes** so the relay and directory can't tell them apart. Worth copying for our control calls. | [BIP 77](https://github.com/bitcoin/bips/blob/master/bip-0077.md) | fetched 2026-10-10 | H |
| Worker as gateway | Feasible. `hpke-js` lists Cloudflare Workers as a supported runtime and passes RFC 9180 and Wycheproof vectors, but is "**not been formally audited**". | [dajiaji/hpke-js](https://github.com/dajiaji/hpke-js) | fetched 2026-10-10 | H |
| JS OHTTP libraries | `ohttp-js` (chris-wood) targets **draft-06**, has no releases and no audit. `ohttp-ts` claims browser and Workers support (v0.5.2). The Rust `ohttp` crate (Martin Thomson) is the reference Cloudflare points to; compiling it to WASM is an option. No audited JS/WASM OHTTP library found. | [ohttp-js](https://github.com/chris-wood/ohttp-js), [ohttp-ts README](https://cdn.jsdelivr.net/npm/ohttp-ts@0.5.2/README.md), [martinthomson/ohttp](https://github.com/martinthomson/ohttp) | fetched 2026-10-10 | H / M |
| Reference gateway | `privacy-gateway-server-go` is a "reference implementation", based on draft-02, with "no key rotation yet". | [cloudflare/privacy-gateway-server-go](https://github.com/cloudflare/privacy-gateway-server-go) | fetched 2026-10-10 | H |
| Key configuration | Clients MUST use a key configuration that is integrity-protected and attributable to the gateway, and applications MUST stop per-client configurations being used for tracking. The `application/ohttp-keys` format exists; RFC 9540 adds DNS SVCB discovery. Pinning the configuration in our signed manifest or APK meets both requirements *(inference)*. | RFC 9458 §Client Responsibilities and §Privacy; [RFC 9540](https://www.rfc-editor.org/rfc/rfc9540) | Jan 2024 / Feb 2024 | H |
| Fresh HPKE context | Clients MUST generate a new HPKE context for every request. Reuse links requests and can expose content to the relay. | RFC 9458 §Client Responsibilities | Jan 2024 | H |
| Timing and size | The relay sees message boundaries and timing. Pad messages (binary HTTP padding) and consider delays. A large-volume relay gives a bigger anonymity set. | RFC 9458 §Traffic Analysis | Jan 2024 | H |
| Direct bulk transfers | Presigned PUT/GET to our storage reveals the **client IP + object key** to the storage operator (Cloudflare R2 = same company). The gateway issued that key, so "initiate/finalise via OHTTP, parts direct" links IP to upload with **no timing analysis needed** *(inference)*. | — | — | L (strong) |
| Abuse without IPs | The gateway sees only the relay IP. RFC: relays MAY rate-limit, but differential treatment shrinks the anonymity set. Gateway-to-relay feedback can de-anonymise. Servers may need an agreement with the relay. Move rate limits to **anonymous tokens** (blind-signed credentials, Privacy Pass-style, or Cashu ecash spent per upload) *(design inference)*. | RFC 9458 §Differential Treatment, §Denial of Service | Jan 2024 | H / L |
| Turnstile and OHTTP | The Turnstile widget calls Cloudflare directly from the client IP at solve time. To stay unlinkable, the solve must only *issue* blind credentials, and those credentials are later *redeemed* over OHTTP. Never send the Turnstile token itself over OHTTP alongside the request *(inference)*. | [Turnstile CSP doc](https://github.com/cloudflare/cloudflare-docs/blob/production/src/content/docs/turnstile/reference/content-security-policy.mdx) | — | L |
| VPN / Tor users | OHTTP adds little for Tor users but does no harm. VPN users gain separation from the VPN provider only for control calls. Neither changes the bulk-transfer leak. | — | — | L |
| Applicability | OHTTP "removes linkage at the transport layer, which is only useful for an application that does not carry state between requests". Upload sessions carry state, so it is a weak fit for initiate/finalise. Credential issuance is a good fit. | RFC 9458 §Applicability | Jan 2024 | H |

**Honest claim wording** (draft, if shipped as scoped in Decision 9): "Credential requests travel through an independent relay run by [operator]. The relay sees your IP address but not the request; Refueler sees the request but not your IP address. File uploads and downloads connect directly to storage, so your IP address is visible to our storage provider (Cloudflare) for those transfers. Use Tor or a VPN if that matters to you."

**What still leaks** even with OHTTP:
- Your IP address for bulk transfers.
- The relay operator learns that you use Refueler, and when.
- Request sizes and timing, unless padded.
- Cloudflare (as our host) sees all request content.
- The relay operator could collude with us.

## Cross-repo notes

For BRIDGE.md. These apply to refueler.io site/POS, Relay, Refill and NutPub as well as Share.

- **One origin per key-handling app.** WEBCAT and WAICT apply integrity to a whole (sub)domain. Any Refueler product that holds keys or tokens in the browser should live on its own subdomain with a static frontend, no inline JS and a CSP of `script-src 'self'`. Mixing marketing pages with key-handling pages on one origin blocks enrollment.
- **WEBCAT subdomain cap.** WEBCAT caps enrolled subdomains per registered domain (`max_enrolled_subdomains`, value not published). Check the cap before planning `share.`, `pos.`, `refill.` and so on.
- **Cloudflare zone hygiene.** Set `Cache-Control: no-transform` and turn off Web Analytics auto-inject, Email Obfuscation, Rocket Loader and challenge rules on every key-handling host. Today Web Analytics auto-inject covers *every subdomain in the zone* by default.
- **Shared release tooling.** Build one ship-script module (WEBCAT manifest + Sigsum signing + fingerprint publishing) and one key-custody policy for all repos. Use separate signer keys per product, so a compromise of one product's key leaves the others intact *(recommendation)*.
- **Shared Android pattern.** Any product that needs a GrapheneOS app can reuse the WebView wrapper with bundled assets and the Obtainium/Zapstore publishing flow (`zsp`, `zapstore.yaml`, `--link-key`).
- **Cashu as anonymous abuse control.** Blind-signed ecash spent per action is the natural replacement for IP rate limits behind an OHTTP gateway, and Refueler already works with Cashu. A shared credential-issuer design could serve Share, Refill and NutPub *(inference; product internals not reviewed)*.
- **OHTTP gateway Worker could be shared** by all products' small control calls. One gateway key configuration means a larger anonymity set. It still needs a non-Cloudflare relay partner, and Payjoin relays are BIP77-only by design.
- **Turnstile in key-handling pages** blocks integrity enrollment in every product, not just Share. Use the same gate-origin pattern everywhere.

## Open questions / unverified

1. Does the WEBCAT extension tolerate deploy races (old cached file against a new manifest)? Is there a grace mechanism beyond `enrollment-prev.json`? Not documented in the spec. Ask FPF.
2. Can the zone apex (`refueler.io`) be enrolled, or only strict subdomains? The spec says the domain "must be a strict subdomain of `zone`". The meaning of `zone` is ambiguous.
3. The actual values of `voting_config.delay` (cool-down) and `max_enrolled_subdomains` on the production chain.
4. Is a WEBCAT Chromium extension or Android support on the roadmap with dates? The EF grant mentions only research.
5. When will WAICT ship enabled by default in Firefox or Chromium, and will Vanadium follow Chromium? Unknown.
6. Can a YubiKey (OpenPGP-card Ed25519 through gpg-agent's SSH support) sign for `sigsum-submit`? Not tested.
7. Does Vanadium support TWAs? Moot for us, but unverified.
8. Do Cloudflare Pages deployments switch atomically, and does `_headers` reliably set `no-transform` on every asset? Read the Pages docs (blocked here).
9. Is Turnstile's `api.js` allowed to be self-hosted under Cloudflare's terms? Assumed no.
10. Is OHTTP draft-06 (ohttp-js) wire-compatible with final RFC 9458? Verify before use, or use `ohttp-ts` or WASM `ohttp`.
11. Does any privacy organisation run a general-purpose, non-Cloudflare OHTTP relay open to small projects? None found.
12. Fastly OHTTP relay terms, logging and price: not public.
13. Were these pages read only through search excerpts? Yes for: WEBCAT alpha date (~Mar 2026), Cloudflare OHTTP Gateway date, Mozilla WAICT post contents, the F-Droid RB page, the Zapstore FAQ and the Proton APK page. Re-read them in a browser before quoting externally.
14. Is there an independent audit of WEBCAT? Search excerpts mention an "independent evaluation" post ([securedrop.org](https://securedrop.org/news/webcat-update-independent-evaluation-waict-and-a-growing-team/)). Contents not read.
