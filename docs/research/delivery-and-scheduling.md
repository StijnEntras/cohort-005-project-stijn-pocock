# Research: delivering Notifications and scheduling Nudges

_Researched 2026-11-16 against the installed packages (react-router 8.3.0, better-sqlite3 12.11.1, drizzle-orm 0.45.2, compression 1.8.2) and the live docs. reactrouter.com currently serves 8.5.1 docs; where the docs that ship in `node_modules/react-router/docs` (8.3.0) cover the same point, they agree._

## Question

How can this app (a) show new in-app **Notifications** without real-time push, and (b) run time-based **Nudge** work? We compare:

- **(a) Delivery:** fetching at load time in a layout loader; polling with `useRevalidator` or `useFetcher`; Server-Sent Events (SSE) from a resource route.
- **(b) Scheduling:** an external cron hitting an endpoint; an in-process scheduler (node-cron, croner, `setInterval`); lazy evaluation on request.

The user set these constraints:

- This is not real-time chat. Showing up on the next page load or after a short poll is fine.
- Notifications are the core feature.
- Nudges are separate, and they must never block Notifications.

## Stack facts (from the repo)

- **Pinned versions.** `package.json` pins `react-router`, `@react-router/node`, `@react-router/serve` and `@react-router/dev` at **8.3.0**. It also has `better-sqlite3 ^12.8.0` (12.11.1 installed), `drizzle-orm ^0.45.2`, `react ^19.2.7` and `vite ^7`.
- **One process.** The `start` script is `react-router-serve ./build/server/index.js`, a single long-lived Node process. `dev` is `react-router dev`, a Vite dev server. No deployment config exists for the app (no Dockerfile, Fly or Render files).
- **SSR on.** `react-router.config.ts` sets `ssr: true` (framework mode with server rendering).
- **One SQLite file.** `docs/adr/0006-one-sqlite-file-through-drizzle.md` (uncommitted, in the main checkout) says:
  - Cadence runs as one server process.
  - Data lives in one `data.db` file, reached through synchronous better-sqlite3 and Drizzle.
  - There is one writer at a time.
  - The app cannot run on more than one machine against the same data.
- **Connection setup.** `app/db/index.ts` opens one connection lazily and sets `journal_mode = WAL` and `foreign_keys = ON`.
- **The app shell loader.** `app/routes/layout.app.tsx` wraps every authenticated route (see `app/routes.ts`). On every request, its loader:
  - reads the session user (`getCurrentUserId`, a cookie session in `app/lib/session.ts`);
  - loads **all users** for the DevUI;
  - runs several queries per recent course to build sidebar progress.

  So it is not a cheap loader. The `api/*` resource routes (e.g. `app/routes/api.video-tracking.ts`) sit outside this layout.
- **Existing patterns.** `app/root.tsx` has no loader. `useFetcher` is used widely (`app/components/comment-thread.tsx`, admin and instructor routes). Nothing uses `useRevalidator`, `shouldRevalidate` or `EventSource` today. The only `setInterval` is client-side, in `app/components/youtube-player.tsx`.
- **No user timezone.** `app/db/schema.ts` has no timezone column on `users`.

### React Router v8 points that matter here

- **v8 changes are not relevant.** From v7 to v8, middleware became always-on and `context` became a `RouterContextProvider`. The raw `request` is now always passed to loaders, with a separate `url` arg for the normalized URL. Node ≥ 22.22 is required. ([CHANGELOG v8.0.0](https://github.com/remix-run/react-router/blob/main/packages/react-router/CHANGELOG.md)) Nothing changed in revalidation, fetchers or resource routes that affects these options. v8.2.0 only adds "Preserve navigation blocker state through a revalidation".
- **Framework mode with SSR revalidates on every navigation.** "In framework mode with SSR, route loaders are automatically revalidated after all navigations and form submissions" ([route-module#shouldRevalidate](https://reactrouter.com/start/framework/route-module)). Active loaders are revalidated on `Link`, `Form` and `fetcher.submit`. Failed 4xx/5xx submissions do not revalidate ([optimize-revalidation](https://reactrouter.com/how-to/optimize-revalidation)). **Consequence:** an unread count in `layout.app`'s loader refreshes on every client navigation and after every mutation, with no extra code.
- **Each route decides for itself.** Matched routes are handled independently. A child that opts out does not stop its ancestors from revalidating ([optimize-revalidation](https://reactrouter.com/how-to/optimize-revalidation)).

## Part A — surfacing Notifications

| Option | Freshness | Server cost per open tab | Complexity | Fit with one process + sync SQLite |
| --- | --- | --- | --- | --- |
| **A1. Unread count in `layout.app` loader** | Next navigation or mutation | Zero when idle; one indexed `COUNT` per navigation | Trivial | Excellent: sub-millisecond synchronous read |
| **A2. A1 + `useRevalidator` on an interval or visibility change** | Interval (e.g. 30–60 s) | Re-runs **all matched loaders** per tick, including the heavy layout loader and the page loader (inference, see below) | Low | OK, but wasteful given this layout loader |
| **A3. A1 + `useFetcher().load("/api/notifications/unread")` on an interval or visibility change** | Interval | One tiny resource-route request per tick | Low | Excellent |
| **A4. SSE resource route** | Seconds | One **held-open** connection per tab, plus a per-connection timer or event listener | Medium–high | Works, but has many sharp edges (below) |

### A1. Load-time fetch in the layout loader

- **How it updates.** Revalidation (above) means the count updates on every navigation and after every `fetcher.submit` or `Form` post. Marking a Notification read through a fetcher also refreshes the badge.
- **Cost.** One synchronous indexed query on an `(recipient_id, read_at)` index. Blocking time is negligible next to the queries this loader already runs. That is inference, but better-sqlite3 positions its synchronous API as giving "better concurrency than an asynchronous API" ([README](https://github.com/WiseLibs/better-sqlite3)).
- **Downside.** A user who sits on one page (e.g. watching a lesson video) sees nothing new until they navigate. The user said that is acceptable.

### A2. Polling with `useRevalidator`

- **What the docs say.** The docs pitch it exactly for this: "Revalidate the data on the page for reasons outside of normal data mutations like Window focus or polling on an interval" ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)).
- **Cost is the problem.** It revalidates "the data on the page", meaning every active loader. In this app that includes `getAllUsers()` and the per-course progress queries in `layout.app.tsx`, plus whatever the leaf route loads. Polling every 30 s per tab multiplies all of that.
  - Narrowing it with `shouldRevalidate` is awkward. A route that always returns `false` also opts out of explicit `useRevalidator` calls ([optimize-revalidation](https://reactrouter.com/how-to/optimize-revalidation)).
  - _Inference:_ the docs do not explicitly say "all loaders including layouts". "Data on the page" plus the independent-routes rule implies it.
- **Pause when hidden.** MDN: `visibilitychange` to `hidden` "is also a good point at which pages can stop making UI updates and stop any tasks that the user doesn't want to have running in the background" ([MDN visibilitychange](https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event)). Gate the interval on `document.visibilityState`, and revalidate once when the tab becomes visible.

### A3. Polling a small resource route with `useFetcher`

- **Resource route.** A module that exports a `loader` and no default component is a resource route. When it is called from fetchers, it should return `data()` ([resource-routes](https://reactrouter.com/how-to/resource-routes)). Put it outside `layout.app` next to the existing `api/*` routes (`app/routes.ts`) so only its own loader runs.
- **Revalidates for free.** `fetcher.load` "only revalidates by default after action submissions and explicit useRevalidator calls" ([optimize-revalidation](https://reactrouter.com/how-to/optimize-revalidation)). The badge therefore also refreshes after any mutation, at no extra cost.
- **Shared state.** `useFetcher({ key })` lets the sidebar badge and a dropdown share one fetcher's state ([useFetcher](https://reactrouter.com/api/hooks/useFetcher)).
- **Avoid plain `fetch()`.** A plain `fetch()` to a resource route "does not go through the router, so it does not revalidate loaders" ([optimize-revalidation](https://reactrouter.com/how-to/optimize-revalidation)). Prefer the fetcher.
- **Seeding.** Seed the first render from the A1 loader value. Then show `fetcher.data ?? loaderData.unreadCount`.

### A4. Server-Sent Events from a resource route

- **It does work through `react-router-serve`.** Checked in the installed source:
  - `@react-router/serve` always installs `compression()` for non-RSC builds (`node_modules/@react-router/serve/dist/cli.js`).
  - `text/event-stream` matches compression's default `^text\/` compressible rule (`node_modules/compressible/index.js`). The compression README warns that it "does not work out of the box with server-sent events" unless `res.flush()` is called ([compression README](https://github.com/expressjs/compression#server-sent-events)).
  - However, `@react-router/express` calls `res.flushHeaders()` for `text/event-stream`. `@react-router/node`'s `writeReadableStreamToWritable` calls `writable.flush()` after every chunk when it exists (`node_modules/@react-router/express/dist/index.js`, `node_modules/@react-router/node/dist/index.js`).
  - So events are flushed. Sending `Cache-Control: no-cache, no-transform` makes compression skip the response entirely ([compression README](https://github.com/expressjs/compression#compressionoptions)).
- **Connection limit.** Without HTTP/2, SSE "suffers from a limitation to the maximum number of open connections … per browser and is set to a very low number (6)" ([MDN Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)). `react-router-serve` calls `app.listen` (plain HTTP/1.1; `cli.js`). Seven tabs of Cadence would starve the browser's connections to the site. _Inference:_ a TLS-terminating HTTP/2 proxy in front would lift this, but no deployment config exists.
- **Proxies and timeouts.** MDN's example sets `X-Accel-Buffering: no` (an nginx buffering opt-out) and recommends periodic `:` comment lines "to prevent connections from timing out". The browser reconnects automatically, honouring a `retry:` field ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)).
- **Where events would come from.** With synchronous SQLite and no change-feed, each open stream must either:
  - poll the DB on its own timer (polling moved server-side, now per tab, with connection state to clean up on `request.signal` abort); or
  - subscribe to an in-process `EventEmitter` fired by the code that writes Notifications.

  The second only works because there is exactly one process (ADR 0006), and it adds cleanup work: leaked listeners, morgan logging requests that never finish.
- **Dev mode.** Module-level emitters or timers survive or duplicate across Vite HMR unless disposed. Vite asks modules to "use `hot.dispose` to clean-up any persistent side effects" ([Vite HMR API](https://vite.dev/guide/api-hmr)).
- **Verdict.** It buys seconds-level freshness the user explicitly does not need, and costs a held connection per tab.

## Part B — running Nudge work

| Option | Survives restarts / downtime | Duplicates risk | Dev ergonomics | Works if nobody visits |
| --- | --- | --- | --- | --- |
| **B1. Lazy, on request** | Yes: state is derived from DB timestamps on each visit | Needs a unique key | Nothing extra to run | No Nudge row is created, but nobody would see an in-app Nudge anyway |
| **B2. In-process scheduler** (croner / node-cron / `setInterval`) | Runs missed while down are not replayed by default | Two processes (dev + start, HMR reloads) double-fire unless keyed | Must guard against HMR and test imports | Yes |
| **B3. External cron → authenticated endpoint** | Depends on the cron host's retry | Retries double-fire unless keyed | Needs a second system; none in repo | Yes |

### B1. Lazy, on-request evaluation

- **How it works.** When an authenticated page loads, compute which Nudges are due for *this* user and insert them. Examples: "no lesson progress in 7 days", "50% through course X".
- **Idempotency.** Make it idempotent with a unique index on `(user_id, nudge_kind, period_key)`, e.g. `period_key = '2026-W47'` or `'course:12:50pct'`, and insert with conflict-ignore:
  - SQLite supports `INSERT … ON CONFLICT DO NOTHING` ([SQLite UPSERT](https://sqlite.org/lang_upsert.html)).
  - Drizzle's SQLite insert builder exposes `.onConflictDoNothing({ target })` (`node_modules/drizzle-orm/sqlite-core/query-builders/insert.d.ts`; [Drizzle insert](https://orm.drizzle.team/docs/insert)).
- **Writes during GET.** Writing in a loader is fine for SQLite in WAL mode: "writers do not block readers and readers do not block writers", though "there can only be one writer at a time" ([sqlite.org/wal](https://sqlite.org/wal.html)). In one process with one synchronous connection, writes are serialized anyway. To limit write volume:
  - throttle evaluation per user (e.g. store `nudges_evaluated_at` and skip if under 1 h old); and
  - only write when something is actually due.
- **Must not block Notifications.**
  - Evaluate Nudges in their own function, wrapped in `try/catch`, *after* the Notification count is computed. Never let its failure fail the layout loader.
  - Or move it entirely off the critical path: a fire-and-forget `fetcher.submit` to an `api/nudges/evaluate` action on mount. That fetcher's action would then revalidate loaders, which is useful so new Nudges appear.
- **"Nobody visits" is a non-issue for in-app-only Nudges.** A Nudge is only seen in-app, so creating it at the visit is the same for the user as creating it at 09:00 (inference). It stops being equivalent if Nudges later go out by email or push, or if "created at" must reflect the scheduled time.
- **Timezone.** Lazy evaluation uses the request time. There is no user timezone column, so periods default to UTC (or server-local) unless a timezone is added.

### B2. In-process scheduler

- **Fits the deployment.** One long-lived `react-router-serve` process fits ADR 0006.
- **Nothing persists or catches up.**
  - croner "Operates in-memory, with no need for a database or configuration files" and supports `timezone` and `protect` (overrun protection) ([croner](https://github.com/Hexagon/croner)).
  - node-cron v4 "coordinates but does not persist state to a database", has `timezone` and `noOverlap`, and lists an `execution:missed` event without documenting catch-up ([node-cron](https://github.com/node-cron/node-cron)).
  - Neither README promises to replay runs missed while the process was down. Plan as if they don't, so jobs must be idempotent and should scan "everything due since last run", not "exactly now".
- **Plain `setInterval`.** Node makes "no guarantees about the exact timing" ([Node timers](https://nodejs.org/api/timers.html)). `timeout.unref()` lets the process exit without waiting on it.
- **Event-loop blocking.** Because better-sqlite3 is synchronous, a batch over all users blocks every request until it finishes (inference from the synchronous API). Chunk the work or keep it small. better-sqlite3 offers worker threads "for large/slow queries" ([README](https://github.com/WiseLibs/better-sqlite3)).
- **Where to start it.** The app has no custom server and no `entry.server.tsx`. Starting a scheduler from a route module or `app/db` would:
  - start it lazily on first request;
  - re-register it on every HMR update in `react-router dev` unless guarded (`globalThis` flag or `import.meta.hot.dispose`, per [Vite HMR API](https://vite.dev/guide/api-hmr));
  - start it inside vitest if a test imports that module.

  Running `npm run dev` and `npm start` against the same `data.db` also gives two schedulers. Unique keys make that harmless.

### B3. External cron hitting an endpoint

- **What it needs.** An `api/cron/nudges` action protected by a shared secret header, and a host that runs cron (systemd timer, platform cron, GitHub Actions schedule). The repo has no deployment config, so this is speculative today.
- **Strength.** It survives app restarts, and timing lives outside the app.
- **Same constraints as B2.** It still runs inside the one process against the one SQLite file. It needs the same idempotency keys, because cron hosts retry. It adds a moving part that has to be deployed and monitored.

## Recommendation

**Notifications.**

- Store Notifications as rows. Read the unread count in `app/routes/layout.app.tsx`'s loader (A1). Framework-mode revalidation keeps it fresh on every navigation and mutation.
- Add a short poll (A3): `useFetcher({ key: "notifications" }).load("/api/notifications/unread")` every 30–60 s, only while `document.visibilityState === "visible"`, plus one load on `visibilitychange` to visible.
- Prefer this over `useRevalidator` (A2), because this layout loader is heavy and a revalidate re-runs it.
- Skip SSE (A4). The 6-connection HTTP/1.1 limit, held connections, and HMR and cleanup work buy freshness the user said they don't need.

**Nudges.**

- Evaluate them lazily per user on request (B1). Make it idempotent with a unique `(user_id, nudge_kind, period_key)` index and `onConflictDoNothing`, and throttle it per user.
- Keep it isolated from the Notification path: a separate `try/catch`, or a separate fetcher action after mount.
- Keep evaluation logic in a pure function `dueNudges(user, now)` so an in-process croner job (B2) or an external cron (B3) can call the same code later. Revisit only if Nudges must be delivered off-site (email or push) or at fixed local times.

## Open questions for the spec

1. **Unread vs list.** Is a badge count enough on every page, or does the dropdown list need preloading too? (This decides layout loader vs fetcher-on-open.)
2. **Poll interval.** 30 s? 60 s? Only on visible tabs? Is "next navigation only" (no poll at all) acceptable for v1?
3. **Read semantics.** Per-item read, "mark all read", or auto-read on open? Do Nudges share the same inbox and read state as Notifications, or sit in a separate surface? (The glossary keeps them separate.)
4. **Nudge periods.** Which kinds exist, and what is the dedupe period for each (once per course milestone, once per ISO week of inactivity)? This defines `period_key`.
5. **Timezone.** Is UTC acceptable for "this week" and "today", or do we add a `users.timezone`?
6. **Off-app delivery.** Will Nudges or Notifications ever go out by email? If so, B1 stops being enough and B2 or B3 is needed.
7. **Retention.** Do old read Notifications or Nudges get pruned, and by what (lazy, or a scheduled job)?

## Sources

- Repo:
  - `package.json`, `react-router.config.ts`, `app/routes.ts`
  - `app/routes/layout.app.tsx`, `app/root.tsx`, `app/db/index.ts`, `app/db/schema.ts`, `app/lib/session.ts`, `app/routes/api.video-tracking.ts`
  - `docs/adr/0006-one-sqlite-file-through-drizzle.md` (main checkout, uncommitted)
- Installed source:
  - `node_modules/@react-router/serve/dist/cli.js` (compression, express, `app.listen`)
  - `node_modules/@react-router/express/dist/index.js` (`flushHeaders` for event-stream)
  - `node_modules/@react-router/node/dist/index.js` (per-chunk `flush`)
  - `node_modules/compressible/index.js`
  - `node_modules/drizzle-orm/sqlite-core/query-builders/insert.d.ts`
  - `node_modules/react-router/CHANGELOG.md`
- React Router:
  - https://github.com/remix-run/react-router/blob/main/packages/react-router/CHANGELOG.md
  - https://reactrouter.com/start/framework/route-module
  - https://reactrouter.com/how-to/optimize-revalidation
  - https://reactrouter.com/api/hooks/useRevalidator
  - https://reactrouter.com/api/hooks/useFetcher
  - https://reactrouter.com/how-to/resource-routes
  - https://reactrouter.com/start/framework/data-loading
- MDN:
  - https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
  - https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event
- compression: https://github.com/expressjs/compression#server-sent-events
- better-sqlite3:
  - https://github.com/WiseLibs/better-sqlite3
  - https://github.com/WiseLibs/better-sqlite3/blob/master/docs/performance.md
- SQLite:
  - https://sqlite.org/wal.html
  - https://sqlite.org/lang_upsert.html
- Drizzle: https://orm.drizzle.team/docs/insert
- Schedulers:
  - croner: https://github.com/Hexagon/croner
  - node-cron: https://github.com/node-cron/node-cron
- Node timers: https://nodejs.org/api/timers.html
- Vite HMR API: https://vite.dev/guide/api-hmr
