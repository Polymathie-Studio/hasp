> **Moved.** This standard now lives in the consolidated DS4AI suite at [Polymathie-Studio/ds4ai/standards/hasp](https://github.com/Polymathie-Studio/ds4ai/tree/main/standards/hasp). This repository is archived and read-only.

# HASP

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/hasp-overview-dark.svg">
  <img alt="HASP overview: why it exists (a free AI tool must pay for every user or let each bring their own key, and the bring-your-own-key plumbing gets rebuilt every time), what it does (holds an AI API key in the browser, never on a server, auto-detecting the provider), its privacy model (no server touches the key), and how it differs from a shared key, a proxy, rebuilding it, and nothing." src="assets/hasp-overview-light.svg" width="1200">
</picture>

HASP is the client-surface hardening posture of DS4AI, the Design Suite for AI: the security a shipped page declares to the browser for the browser to enforce. It spans three altitudes, the response headers (a Content-Security-Policy, clickjacking protection, HSTS, `nosniff`, and the rest), markup integrity (Subresource Integrity, `rel=noopener`, no mixed content), and secrets (no keys in the shipped bundle, and a user's key held in the browser). It audits *declared* hardening, judged for effect and not presence alone, and never calls a site "secure."

This repository is the home of all HASP code: **HASP-KEY** (the `hasp-key` package, the secrets part), **HASP-GUARD** (`hasp-guard`, the response-header, policy, and integrity generator), and **`hardened.js`**, the auditor that checks all three altitudes and is composed into [MISSING](https://github.com/Polymathie-Studio/missing)'s whole-surface conformance auditor.

## HASP-KEY, the keys instrument

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/hasp-render-dark.png">
  <img alt="HASP-KEY rendering its bring-your-own-key modal over an app: a key entered, the provider auto-detected as Claude, and links to get a key, with the key held only in the browser." src="assets/hasp-render-light.png" width="820">
</picture>

**Hold Any-provider Secrets Privately.** Bring your own key.

HASP-KEY is a small, dependency-free holder for AI API keys. A tool that uses it runs on the user's own key: the key is kept in the user's browser, never on a server, and the charges land on the user's own account. It auto-detects the provider from the key, so one entry field handles Anthropic, OpenAI, Google, xAI, and Mistral, and any other key is held as `other` rather than mislabeled, so the name is literally true: it holds any provider's secret. The user, or the app, can also set the provider explicitly.

It exists so apps stop rebuilding the same "paste your key" plumbing, and so the privacy posture (the key never leaves the browser) is the same everywhere.

## Why

Most small AI tools face the same fork: pay for every user's model calls out of a shared server key, or make each user bring their own. Bring-your-own-key is the honest default for a free tool, but the plumbing (a modal, storage, provider detection, the privacy promise) gets rewritten every time. HASP-KEY is that plumbing, once.

## Install

```
npm install hasp-key
```

The package name is `hasp-key`; it is planned for npm but not yet published. Until then, use it from the repository.

React is an optional peer dependency, needed only for the modal and the hook.

## Core (no framework)

```ts
import { setKey, getKey, clearKey, detectProvider } from 'hasp-key'

setKey('sk-ant-...')          // stored in this browser only; provider auto-detected
const key = getKey()           // read it at call time
detectProvider('sk-ant-...')   // 'anthropic'
clearKey()                     // forget it
```

Keys live in `localStorage` by default (persists in the browser). Pass `'session'` as the `kind` argument for `sessionStorage`, which clears when the tab closes, when you want the stricter posture.

## React

```tsx
import { useHaspKey, KeyModal } from 'hasp-key/react'

function AssistButton() {
  const { present, read } = useHaspKey()
  const [open, setOpen] = useState(false)
  // send read() with your own request; never persist it elsewhere
  return present
    ? <button onClick={runWithKey}>Ask</button>
    : <button onClick={() => setOpen(true)}>Add your key</button>
    // ...render <KeyModal onClose={() => setOpen(false)} /> when open
}
```

`KeyModal` is lightly styled and takes `className` and `inputClassName` so it inherits your app's theme. Its default styling reads TEMPER's semantic tokens with fallbacks, so where TEMPER is present the modal follows the active theme, and where it is absent it renders a clean neutral panel.

## The privacy model

The key is held only in the browser. HASP-KEY never sends it anywhere. Your app decides how the key reaches the model: either the browser calls the provider directly (the key never touches any server), or the browser sends the key with a request to your own route which forwards it and does not store or log it. HASP-KEY takes no position beyond keeping the key local; it is your app's job to honor the promise on the wire.

## HASP-GUARD, the headers and integrity instrument

**The security a page declares to the browser, generated right.** `hasp-guard` writes the client-surface hardening posture as server and build configuration: an effective Content-Security-Policy, clickjacking protection, HSTS, `nosniff`, Referrer- and Permissions-Policy, plus the safe-markup helpers (Subresource Integrity, `rel=noopener`). It is zero-dependency, and its counterpart auditor `hardened.js` (in this repo) checks the same posture, so a run of `hardened.js` over `hasp-guard`'s output clears the header tiers.

The defaults are effective, not merely present. There is no `unsafe-inline` or `unsafe-eval`: 91 percent of real-world CSPs undermine themselves with `unsafe-inline` (Web Almanac 2024), so the default here refuses it and serves inline code by nonce or hash. Framing is closed with `frame-ancestors 'none'`, `object-src 'none'` is set, and the HSTS max-age is two years.

`hasp-guard` is planned for npm but not yet published; until then, use it from this repository.

### Generate the headers

```js
import { headerConfig, securityHeaders, csp } from 'hasp-guard'

headerConfig('vercel')    // a vercel.json headers block
headerConfig('netlify')   // a _headers file
headerConfig('nginx')     // add_header lines

securityHeaders()         // { 'Content-Security-Policy': ..., 'Strict-Transport-Security': ..., ... }
csp({ nonce: 'r4nd0m' })  // a strict CSP with a script nonce
```

Widen one directive for a real dependency without reopening the rest:

```js
csp({ connectSrc: ["'self'", 'https://api.example.com'] })
```

### The markup helpers

```js
import { safeLinkRel, integrityAttr, sriHash } from 'hasp-guard'

safeLinkRel('external')          // 'external noopener noreferrer'
integrityAttr('sha384-...')      // 'integrity="sha384-..." crossorigin="anonymous"'
await sriHash('console.log(1)')  // 'sha384-<digest>', via Web Crypto
```

### The auditor

`hardened.js` audits a served response for the same posture across the three altitudes (response headers, markup integrity, secrets), judging effectiveness rather than presence: a wildcard or `unsafe-inline` CSP, a short HSTS, a `frame-ancestors` that allows any origin are each flagged as present but ineffective. It is composed into MISSING's whole-surface conformance auditor, and it is what `hasp-guard`'s output is tested against.

Boundary: `hasp-guard` writes declared client-surface hardening. It never makes a site secure. Server-side security, authentication, TLS, and denial-of-service defense stay the operator's.

## Part of DS4AI, the Design Suite for AI

HASP is one of the standards in **DS4AI, the Design Suite for AI, from [Polymathie-Studio](https://github.com/Polymathie-Studio)**: small, dependency-free pieces that each close one axis of the *invisible-correctness layer*, the part of a shipped surface a look-at-it review cannot see and that fast, AI-assisted building drops.

- **[TEMPER](https://github.com/Polymathie-Studio/temper)**: perceivable, color and design tokens
- **[GRASP](https://github.com/Polymathie-Studio/grasp)**: operable, interaction components
- **[LUCID](https://github.com/Polymathie-Studio/lucid)** + **[GRACE](https://github.com/Polymathie-Studio/grace)**: honest off the happy path, disclosure and state components
- **[HASP](https://github.com/Polymathie-Studio/hasp)**: hardened, client-surface security posture
- **[BEACON](https://github.com/Polymathie-Studio/beacon)**: findable, head metadata and site files
- **[FLEET](https://github.com/Polymathie-Studio/fleet)**: fast and stable, delivery

**[MISSING](https://github.com/Polymathie-Studio/missing)** is the standard at the center of DS4AI: it names the axes, routes each to its instrument, and ships a machine-readable manifest and a conformance auditor. Adopt one and the others compose with it.

## License

Apache-2.0. Copyright 2026 Regis Lloyd Chapman. See `LICENSE` and `NOTICE`.
