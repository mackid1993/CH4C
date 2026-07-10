# Investigation: Does CH4C corrupt DirecTV account auth state?

**Date:** 2026-07-10
**Code examined:** `mackid1993/CH4C` current `main` (`8b17265`) and incident-era fork state
(`f078db610c1a142ec42a7b1411871b4d7fc27505`, last pushed 2026-05-18 — the code actually running
when the ~5:00 teardowns began around Jun 25). All incident-relevant mechanisms were verified to
exist **identically in both versions** (line numbers below are current `main`; f078db6 equivalents
noted where they differ). Static analysis only; no live DirecTV requests were made.

---

## (a) Verdict

**Yes — with an important precision.** CH4C does not corrupt *login credentials* or tokens on the
client side. What it provably does is **create full server-side DirecTV playback sessions and then
abandon every single one of them by killing Chrome mid-playback**, at a rate of one-or-more
abandoned sessions per tune, across three concurrent browser identities on the same account,
with no release, logout, or graceful stop anywhere in the codebase.

- **PROVEN (code, both versions):** the create-and-abandon pattern, its per-tune frequency, its
  3× concurrency, and its retry amplifiers. This is not in dispute — see §(b).
- **THEORY (backend effect):** that the resulting accumulation of orphaned playback
  sessions/leases in DirecTV's backend (Evergent-side session registry, concurrency leases,
  unreleased Widevine license sessions) is the "degraded account state" that made the Osprey
  boxes' ~5-minute media re-auth take the destructive Yospace-rotation path. The code cannot
  prove DirecTV's backend behavior. Confidence that CH4C is the origin: **moderate-to-high** —
  it is the only mechanism found that is simultaneously consistent with (1) the
  credential-reset fix, (2) the `error=null` capture, and (3) the timing correlation with
  CH4C's re-enable. No competing CH4C-side mechanism survived (§(d)).

## (b) The mechanism — exact code (PROVEN layer)

### 1. Session create: `playConsumable`, dispatched per tune

Every DirecTV tune navigates the pooled Chrome to `https://stream.directv.com/guide`, digs the
Redux store out of React fiber internals, extracts `__webpack_require__` via a synthetic chunk
push, locates the site's own `playConsumable` module, and invokes it with a **fabricated**
consumable payload (`duration: 3600`, `badges: ['OnNow']`, `consumableType: 'LINEAR'`,
constraints forged as `isPlayable: true`):

- `main.js:3001-3179` — `navigateDirectvStream()`; the dispatch is at `main.js:3146-3159`.
  (f078db6: `main.js:2940-3103`, dispatch ~3085 — **byte-identical**, verified by diff.)

Server-side, this starts a real playback context on the account: entitlement/session
registration, a concurrency lease, a Widevine license session, and a Yospace ad session —
exactly the objects seen being torn down and rebuilt in the 2026-07-10 box capture.

### 2. Session end: `browser.close()` / `kill -9`. Nothing else. Ever.

There is **no** `stopConsumable`, no logout, no `sendBeacon`, no `beforeunload` handling, no
navigation away from the player before shutdown — confirmed by exhaustive grep over both
versions. Every exit path ends the same way:

- Stream close (viewer stops, **every channel change**, recording end):
  `res.on('close')` → `cleanupManager.cleanup()` → `closeBrowser()` → `browser.close()`
  — `main.js:8352-8364`, `main.js:2765-2801`, `main.js:2681-2696`.
  (f078db6: `main.js:8215`, `2659` — identical.)
- Stream error / tune timeout / recording timeout: same `cleanup()` — `main.js:8366-8375,
  8884, 8928`.
- Recovery paths: `browser.close()` with 10s timeout, then **`taskkill /F` / `kill -9`** of
  every Chrome process on that profile — `error-handling.js:647-699`,
  `killChromeProcessesForUserData()` at `error-handling.js:19-160`.
- Process exit (SIGINT/SIGTERM/uncaught): `browser.close()` loop — `main.js:201-231, 2706-2722`.

Chrome killed mid-playback means the DirecTV web player's own teardown JS (session-stop
beacons, EME/Widevine license close, Yospace session end, concurrency-lease release) never
runs — Puppeteer's `browser.close()` does not reliably fire page unload handlers, and the
`kill -9` recovery path guarantees nothing runs. The player's keepalive heartbeats simply
stop. **The server-side session is orphaned and can only die by backend TTL.**

### 3. Frequency and concurrency multipliers

- **Per-encoder Chrome profiles** (`encoder_0`, `encoder_1`, `encoder_2` —
  `main.js:2525, 4211`): three persistent, independent browser identities (separate cookie
  jars, separate `dcpmgw` JWTs, separate device fingerprints) all on the same DirecTV account.
- **~3 concurrent streams** = up to 3 live sessions + the trailing tail of not-yet-expired
  orphans from previous tunes. Channels DVR closes and reopens `/stream` on **every channel
  change**, so a normal evening of use creates dozens of orphaned sessions.
- **Retry amplification:** `safeStreamOperation` re-runs the entire tune up to 3× on
  network/timeout errors (`error-handling.js:1288-1330`), and a failed `playConsumable`
  falls through to the logo-click fallback which starts playback a second way
  (`main.js:3174, 3189`). A single flaky tune can open multiple sessions.
- **DirecTV is never "stopped" even between streams:** cleanup closes the whole browser and
  relaunches to `about:blank`, so the only DirecTV interactions are tune (create) and kill
  (abandon). There is no code path that ever tells DirecTV a session ended.

## (c) Reconciling with the capture (`error=null`, deterministic ~5:00) — THEORY, labeled

The naive "over the concurrency cap → hard denial" story predicts a `1013`-class error the
boxes never showed. The code evidence supports the *soft* variant instead:

- The boxes were never **denied** — nightly multi-hour streams worked. What the capture shows
  at 4:59 is a **renewal-boundary session migration**: player stop with `error=null` → old
  Yospace session shut down → `Initiating Media Authentication` → new ad-session context on a
  different endpoint → new Widevine `sid`. That is "your session context is no longer valid,
  rebuild it," not "you are rejected."
- 300 seconds is a canonical lease/heartbeat TTL. A per-account session registry polluted
  with CH4C's zombie leases (unreleased concurrency slots, unreleased DRM license sessions)
  plausibly fails **reconciliation** at the first 5-minute renewal — silently, below the
  error-code tier — forcing the box down the destructive re-auth path. When the registry is
  clean, the renewal is a silent no-op and one session survives for hours.
- This also fits the account math: 10 Osprey boxes + 3 CH4C browsers = 13 registered
  streaming identities before counting orphans — chronic pressure near whatever effective
  per-account session-table limit exists, without ever crossing into hard denial.

This backend model is **inference**. What is not inference is that CH4C manufactures exactly
the input (a steady drip of abandoned sessions on the shared account) that such a model needs.

## (d) Refuted / excluded CH4C-side mechanisms

- **Login churn / auto-relogin (`b41f3a0`) — excluded on two independent grounds.**
  (1) *Version:* `b41f3a0` ("Auto-login after TVE session expiry") is dated **2026-07-02**, a
  week after the incident began; f078db6 contains no `markLogoutDetected`/`pendingLogin` at
  all (verified by grep of the fetched incident-era source).
  (2) *Coverage:* even on current `main`, the auto-login trigger requires a site with
  `loggedOutIndicator` and no `checkLoginWaitMs` (`main.js:3534`); DirecTV's config has
  neither property in the right state (`login-manager.js:45-58` — `pollForUrlRedirect` +
  `checkLoginWaitMs: 12000`), so **DirecTV auto-relogin never fires from the stream path in
  any version**. Do not attribute the incident to this commit.
- **Repeated credential logins as fraud signal — mostly excluded.** DirecTV logins happen
  only via the manual Login Manager (`loginEncoders`, `login-manager.js:2324-2524`), which
  does perform a full password login **once per encoder profile** (3× same credentials,
  minutes apart, from stealth-patched automated Chrome with `navigator.webdriver` deletion —
  `login-manager.js:2400-2412`). The Jun-25 re-enable plausibly included one such burst.
  Keep as a secondary, one-time contributor to a soft risk flag at most; it cannot explain a
  recurring condition, and it was operator-triggered, not automatic.
- **Synthetic consumable payload** (fabricated `duration`/`badges`/constraints in the
  `playConsumable` call): a real oddity — session records are created from forged metadata —
  but playback proceeds through the site's own entitlement flow afterward. Plausible minor
  irritant; no evidence it matters. Low confidence, low priority.
- **Mid-stream watchdog restarts:** `StreamMonitor` only logs inactivity
  (`error-handling.js:1259-1278`); it never restarts streams. Not a churn source.

## (e) What the credential reset actually cleared (reasoning backward — THEORY)

A password + security-question + password reset on an Evergent-backed identity invalidates,
account-wide: all refresh tokens, all issued session JWTs (`dcpmgw`), and — critically — the
standing session/device registry entries derived from them. Combined with `pm clear` on each
box, this is a **full flush of the account's server-side session table**, orphans included.
That the fix worked fleet-wide without any client code changing is the strongest evidence
that the poisoned state lived in exactly the layer CH4C's abandon pattern writes to. (It also
logged the CH4C browsers out — note the teardowns stopped while CH4C's sessions were dead.)

## (f) Minimal fix (specified, NOT applied — awaiting operator OK)

**Give the DirecTV player one navigation's worth of time to tear itself down before the
browser dies.** A normal page navigation (unlike process kill) reliably fires
`pagehide`/`visibilitychange` and flushes `sendBeacon`, which is how SPAs like DirecTV's
release sessions, close EME/Widevine sessions, and end ad sessions. The repo already uses
this exact idiom: `directv-service.js:257` parks the page at `about:blank` after channel
discovery.

In `createCleanupManager.cleanup()` (`main.js` ~2788), before `closeBrowser(encoderUrl)`:

```js
// Let streaming SPAs run their pagehide teardown (session release, DRM close,
// ad-session end) before the browser process dies. Process kill skips all of it.
try {
  const browser = browsers.get(encoderUrl);
  if (browser && browser.isConnected()) {
    for (const p of await browser.pages()) {
      const u = p.url();
      if (u && u !== 'about:blank' && !u.startsWith('chrome')) {
        await p.goto('about:blank', { waitUntil: 'domcontentloaded', timeout: 5000 }).catch(() => {});
      }
    }
    await delay(1000); // let teardown beacons flush
  }
} catch (_) {}
```

Notes:
- This is generic (helps every site, not just DirecTV), ~12 lines, and cannot make teardown
  worse — every failure path still falls through to the existing `browser.close()`.
- The stronger variant — locating and invoking the site's own stop action the same way the
  tune locates `playConsumable` (scan webpack factories for `stopConsumable`/`stopAsset`) —
  cannot be verified statically and should only be attempted with a live (throwaway) account.
- Independent hygiene option: reduce DirecTV to fewer concurrent encoder profiles. PrismCast's
  1-stream-per-account cap is the same discipline; no empirical claim it suffices alone.
- Upstream has no existing hook for pre-close page teardown; this would be new (small) code.

## (g) Safe confirmation for the operator (no DirecTV probes)

1. **Log tripwire (no code change needed):** every abandoned session is bracketed by the
   existing lines `DirecTV: successfully tuned to "<ch>"` … `response stream closed for <enc>`
   → `Browser closed for encoder <enc>` with no intervening stop. Counting tune-lines per day
   in CH4C's log = the orphaned-session creation rate while the fix is off.
2. **With the fix:** add one line after the about:blank navigation (e.g.
   `Released streaming page for <enc> before close`) and A/B it: run CH4C with the fix for a
   few days while a box streams; the health signal is the proven one — boxes crossing 5:00
   and holding single DRM+ad sessions for hours.
3. **Re-degradation tripwire (from prior work):** if `1013/1014` ever appears in box logs
   live, the account layer has re-soured; also watch for the 4:59 `VSTB Player stop
   (error=null)` signature returning.
4. If it re-sours **with the fix active**, the abandon mechanism is falsified as the sole
   cause — next suspects in rank order: the 3-profile login bursts (§d), then the synthetic
   consumable payload (§d).
