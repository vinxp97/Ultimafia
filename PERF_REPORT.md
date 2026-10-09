# Ultimafia performance pass: measurements and ranked fixes

Branch `perf/profiling-pass` off `master` @ `722f0d80` (2026-10-05). Analysis
first: five small proof-of-concept commits are on the branch, and everything
else below is a proposal waiting for approval.

## How this was measured

| What | Setup |
|---|---|
| Production checks | Read-only `curl` of public ultimafia.com static files and headers (no login, no games) |
| Frontend | `rsbuild build` (production) served by **nginx 1.26 using the repo's own `react_main/nginx.conf`** (TLS lines stripped). The baseline is master's config and the patched one is this branch's. Lighthouse 12 mobile preset (simulated Slow 4G, 4x CPU), median of 3 runs. Puppeteer with CDP throttling. |
| Bundle | `source-map-explorer` on the production source maps |
| API | Local stack on the box (Node 22.17.0, Mongo 7.0, Redis), www/games/chat on :3200/:3210/:3299. DB seeded with `scripts/perf/seed.js`: 20k users, 3k setups, 40k games over 30 days (≈1.3k/day, 7–16 players, 40 KB history each, activity skewed so 10% of users play 70% of games), 300k notifications, and 50 sessions. Sequential latency comes from `benchRoutes.js` (p50/p95 of 20), throughput from `autocannon -c20`, Mongo op counts from the DB profiler (`countQueries.js`), and plans from `explainQueries.js`. |
| Game server | `scripts/perf/simulateGame.js` plays a real `Mafia` game in-process: 20 players + 5 spectators, Cop/Doctor/4 Mafia/14 Villagers, N chat messages per day, then a full vote. Sockets serialize every frame exactly like `lib/sockets.js`. Run with `--cpu-prof`. |
| Chat rendering | The existing `npm run perf:chat` fixture (#3017), production build, 390×844 viewport, 4x CPU throttle, and a CDP CPU profile on an unminified build |

The Umbrel stacks were not touched. Their dev-server numbers below come from earlier in this workstream.

---

## Baseline numbers

### Frontend (first visit, logged in)

| | value |
|---|---|
| **Production nginx compression for JS/CSS** | **none.** `ultimafia.com` serves `9668.*.js` as 493,885 bytes and sends no `Content-Encoding` even for `Accept-Encoding: gzip, br`. The official nginx image ships with `#gzip on;` commented out. |
| Prod critical-path JS+CSS (7 files) | **1,045 KB on the wire**. Gzip would make it 304 KB. |
| Prod caching of hashed `/static/*` | ETag/Last-Modified only, with no `Cache-Control` (heuristic caching) |
| Initial JS (production build) | lib-react 129 KB, router, axios 35 KB, vendor `9668` 494 KB (**MUI 271 KB**, @mui/system 42 KB, **react-loading 27 KB**, popper 19 KB, iconify 18 KB, firebase/app+util 21 KB), `index` 199 KB (slangList 21 KB, Roles 20 KB, Popover 17 KB) |
| Game route chunk | `7365.js` 343 KB raw / 91 KB gz (`pages/Game` 212 KB) + 112 KB CSS |
| Fonts loaded on first view | Roboto **458 KB TTF**, RobotoSlab **244 KB TTF**, FA solid/brands 150 KB woff2. Nabla is 1.6 MB TTF but only loads where used. |
| Lighthouse mobile, `/` | perf score ~23, FCP 4.06 s, **LCP 12.1 s**, TBT 553 ms, **TTI 18.0 s**, 2,794 KB |
| Lighthouse mobile, `/play` | FCP 4.06 s, LCP 11.5 s, TTI 16.3 s, 2,540 KB |
| Lighthouse mobile, `/user/:id` | FCP 4.05 s, LCP 16.4 s, TTI 16.5 s, 2,563 KB |
| Test stacks (Umbrel :3001/:3101) | rsbuild **dev server**: ~5.6 MB compressed initial JS + ~2.3 MB game chunk, ~3 s fresh load (measured earlier in this workstream) |

### Chat rendering (`perf:chat` fixture, 4x CPU)

| messages in chat | load | **append → next frame (p50 / max)** | DOM nodes |
|---|---|---|---|
| 100 | 1.6 s | 47 / 61 ms | 855 |
| 1,000 | 3.0 s | **199 / 261 ms** | 7,305 |
| 5,000 | 8.7 s | **834 / 1,068 ms** | 35,969 |

After #3017, only **1** `MessageRow` re-renders per append (verified by instrumenting the bundle). The remaining cost is React reconciling the 1,000+ `Message` adapters and recreating the element array on every append, plus emotion/`sx` style serialization (`serializeStyles` + `styleFunctionSx` ≈ 35% of the click). In the trace, 197 of ~320 ms per append is the synchronous React dispatch, with layout and paint at ~60 ms.

### API (seeded DB, warm cache, one www process)

| route | p50 | p95 | Mongo ops | docs examined |
|---|---|---|---|---|
| `/api/user/:id/profile` (heavy user) | 82 ms | 104 ms | 20 | **40,258** |
| `/api/user/:id/profile` (light user) | 90 ms | 94 ms | 20 | 40,018 |
| `/api/user/:id/games?page=1` (light user) | **136 ms** | 143 ms | 7 | **63,986** |
| `/api/setup/:id` | **105 ms** | 124 ms | | 40,000 index keys |
| `/api/game/mostPlayedRecently` | 72–78 ms | 89 ms | aggregate | 9.3k games |
| `/api/notifs` (polled **every 10 s by every tab**) | 32 ms | 35 ms | 6 | 6,047 |
| `/api/user/info` | 5–7 ms | | 5 | |
| `/api/roles/all` | 2.4 ms | | 0 | 62 KB JSON, re-serialized and re-gzipped per request |

* Every request with a session cookie does **2 Mongo ops** on `sessions` (connect-mongo). On a trivial route (`/api/nextRestart`), that cuts throughput from **3,927 to 1,538 req/s** (−61%) and raises p50 latency from 4 to 12 ms.
* Missing indexes: `Game.users` has none, so the profile game count and the game-history page are COLLSCANs. `Game.setup` has none either, so the setup page `playedCount` walks the entire `endTime` index.

### Game server (20 players + 5 spectators, in-process)

| | 200 msgs/day | 1,000 msgs/day |
|---|---|---|
| server time per chat message (all recipients) | 0.08 ms | 0.05 ms |
| server time per vote (mean / p99) | 0.75 / 7.9 ms | 1.65 / 6.2 ms |
| websocket bytes for whole game | 10.4 MB | **46.9 MB** |
| of which `meeting` frames | 4.2 MB (40%) | **18.7 MB (40%), avg 70 KB per frame** |
| `JSON.stringify` (per-recipient frames) | | **26% of game CPU** (156 ms of ~600 ms active) |
| saved `history` JSON | 260 KB | 1.07 MB |

* The server is not CPU-bound per game. A busy 20-player game costs well under 1 ms per event, so lag inside a game is the **client** and the **bytes**, not server compute.
* `Player` `vote`/`unvote` handlers call `sendMeeting(meeting)`, which re-sends the **entire meeting including every chat message of the day** to the voter. Deaths, revivals and leaves call `game.sendMeetings()`, which does the same for **every player**. At 1,000 messages per day, each vote frame is ~70 KB, and one mid-day death/leave sends ~25 × 70 KB ≈ 1.7 MB at once.
* **Memory leak:** `lib/sockets.js` `Socket.received` (the replay buffer for late listeners) was only cleared on the client-side `open` event, which never fires for server-side sockets. Every game and chat connection kept every ping and message for its lifetime. 200 sockets × one simulated day measured **+148.9 MB of heap**.

---

## Proof-of-concept commits on this branch (with before/after)

| commit | change | before → after |
|---|---|---|
| `258be538` | **nginx gzip + immutable cache for `/static/`, `no-cache` for HTML** (`react_main/nginx.conf`) | Lighthouse mobile `/`: **LCP 12.1 → 7.5 s, TTI 18.0 → 10.0 s, 2,794 → 1,688 KB**. `/play`: LCP 11.5 → 7.3 s, TTI 16.3 → 9.7 s. `/user/:id`: LCP 16.4 → 10.6 s, TTI 16.5 → 10.6 s. Puppeteer Slow 4G `/play`: **FCP 3.04 → 1.04 s, DOMContentLoaded 7.7 → 2.7 s**. Vendor chunk 494 → 147 KB on the wire. |
| `304edc92` | **Mongo indexes** `games {users:1,endTime:-1}`, `{setup:1,endTime:-1}`, `{startTime:-1}` and `notifications {user:1,isChat:1,read:1}` (`db/schemas.js`) | profile **82 → 28 ms** (light user 90 → 22). Game history **136 → 7.6 ms** (heavy-user page 20: 92 → 7.6). Setup page **105 → 8.6 ms**. Docs examined 40,000 → 948 / 10 / 65. Unread-notif query 22 → 4 ms. |
| `687d0210` | **`/api/notifs/unreadCount`**: the 10-second poll only needed the count and restart time, so it now counts in Mongo instead of loading the full list (`routes/notifs.js`, `Main.jsx`). `/api/notifs` is unchanged. | per poll **56 KB → 0.04 KB, p50 15.5 → 5.2 ms, 104 → 605 req/s** (user with 416 unread). Typical users gain less on bytes, but every user still gets 3 fewer Mongo ops' worth of documents. |
| `401bb812` | **Socket replay buffer**: pings are no longer stored, and only the last 100 messages are kept (`lib/sockets.js`) | 200 sockets × 1 day: **+148.9 MB → +4.1 MB heap**. The simulated game still plays to completion. |
| `55cec29b` | `lazy()` route components moved to module scope (`Main.jsx`) | Correctness fix with no measured gain. The resize test still remounts because pages swap phone/desktop layouts themselves (see #12). |
| `dc36ed7f` | `scripts/perf/*` harness | |

The production frontend build succeeds. `test/sockets.test.js` passes. The `Game.test.js`, `History.test.js` and `redis.test.js` subset shows the same 2 passing and 9 failing tests on unmodified master in this box environment (redis-host timeouts), so those failures are not caused by these commits.

---

## Ranked fixes

Impact is for real users unless noted. Effort is S = hours, M = 1–3 days, L = a week or more.

### Quick wins

| # | Fix | Expected gain | Effort | Risk | Files |
|---|---|---|---|---|---|
| 1 | **Turn on gzip in prod nginx + immutable `/static/` cache** *(POC done)* | First load ~**40% faster to interactive on mobile** (TTI 18 → 10 s Slow 4G), −1.1 MB per first visit, **−740 KB on the critical path** (1,045 → 304 KB). No repeat-visit revalidation storms after deploys. | S | Low. Make sure `nginx.conf` reaches the image (it is both COPY'd and volume-mounted). Brotli would add another ~15% but needs the full nginx image or `gzip_static` pre-compression at build time. | `react_main/nginx.conf` |
| 2 | **Add the four Mongo indexes** *(POC done)* | Profile ~3x faster, game history and setup pages ~10–15x faster on the seed. In prod, a COLLSCAN of games with real histories (60 KB–1 MB each) is disk-bound, so the real gain is likely **seconds → ms**. | S | Low/Med. Mongoose `autoIndex` builds them on boot. Build them by hand first on prod (`createIndex`) during low traffic, and check `db.games.getIndexes()` for an existing manual `users` index. | `db/schemas.js` |
| 3 | **Unread-count endpoint for the 10-second poll** *(POC done)* | 5.8x more throughput on the most frequent authenticated request. Per-tab bandwidth drops from up to 56 KB per 10 s to ~0. | S | Low | `routes/notifs.js`, `react_main/src/Main.jsx` |
| 4 | **Fix the socket replay-buffer leak** *(POC done)* | −0.75 MB heap per connection-day on the chat and games processes, and less GC and restart pressure | S | Low | `lib/sockets.js` |
| 5 | **Fonts: TTF → WOFF2 + `font-display: swap`** | Roboto 457 → 204 KB, RobotoSlab 244 → 113 KB, RobotoMono 177 → 101 KB, Poppins 136 → 47 KB, Nabla 1,604 → 179 KB. That is **−384 KB on every first visit** for the two fonts on every page. `swap` removes up to 3 s of invisible text on slow links. A static 400/700 subset instead of the variable font would be ~−600 KB. | S | Low (visual check) | `react_main/src/fonts/*`, `src/css/main.css` `@font-face` |
| 6 | **Serve the test stacks (Umbrel) from a production build** instead of `rsbuild dev` | Fresh load ~3 s → <1 s. The 5.6 MB dev JS becomes ~0.3 MB gz. | S | Low, but redeploys need `rsbuild build` | stack compose/pm2 for :3001/:3101 |
| 7 | **Precompute static role JSON** (`/api/roles/all`, `/raw`, `modifiers`, `gamesettings`, `roletags`, `descriptions`): serialize and gzip once at boot, then send with `Cache-Control: max-age=300` + ETag | ~140 KB of JSON re-stringified and re-gzipped **per page load** today, at ~900 req/s per core. Precomputed this is ~10x+ cheaper, and browsers stop refetching on every navigation. | S | Low | `routes/roles.js` |
| 8 | **Cache `mostPlayedRecently` and the site-activity aggregates in Redis (5 min)** | 72 ms aggregate per lobby view → ~1 ms | S | Low | `routes/game.js`, `modules/redis.js` |
| 9 | Seasonal logos as WebP (`logo-halloween.png` 93 KB in the LCP area), drop `react-loading` (27 KB in vendor) for a CSS spinner | −80–100 KB first view | S | Low | `src/images/holiday/*`, components using `react-loading` |

### Bigger projects

| # | Fix | Expected gain | Effort | Risk | Files |
|---|---|---|---|---|---|
| 10 | **Chat list windowing** (react-virtuoso or similar), or as a cheaper first step **chunked memoization**: render messages in memoized blocks of ~100 so an append only reconciles the last block | Append → frame at 1,000 msgs **~200 ms → ~20–40 ms**, and at 5,000 msgs **~830 ms → ~40 ms** (O(1) instead of O(n)). DOM goes from 36k nodes to ~1k at 5,000 msgs, which fixes long-game jank on phones. | M (chunking) / L (virtualization, which touches scroll-follow, quotes and pins) | Med | `react_main/src/pages/Game/Game.jsx` (TextMeetingLayout, Message). Coordinate with the kudos branch, which also edits Game.jsx. |
| 11 | **Stop re-sending all meeting messages on vote/unvote/death/leave**: send a slim meeting update (votes, targets, members) without `messages`, since the client already has them, or send vote diffs | **−40% of all websocket bytes** in a busy game (18.7 of 46.9 MB in the sim), a 70 KB → ~2 KB vote frame, and no ~1.7 MB burst on a mid-day death or leave. That is a noticeable lag fix on mobile. | M | Med. The client reducer must merge the partial meeting, and review mode must still get full meetings. | `Games/core/Meeting.js getMeetingInfo`, `Games/core/Player.js` vote/unvote/kill/revive, `Games/core/Game.js sendMeetings`, `react_main/src/pages/Game/Game.jsx` meeting handler |
| 12 | **Move sessions to Redis** (connect-redis, already deployed) or cache the session read | **2.5x** throughput on trivial authenticated routes (1,538 → ~3,900 req/s), −7 ms p50 on every API call, and 2 fewer Mongo ops per request | M | Med: one-time logout of all users unless sessions are migrated, and `models.Session.deleteMany` in `routes/user.js` (force-logout/ban) must be ported | `modules/session.js`, `modules/mongoStore.js`, `routes/user.js`, `routes/mod.js` |
| 13 | **Serialize broadcast frames once**: `Game.broadcast` and `Message.send` stringify the same payload per recipient. Cache the frame per message version and send it raw. | ~20–25% of game-server CPU in chat-heavy games (stringify is 26% of the profile) | M | Low/Med | `lib/sockets.js` (add `sendRaw`), `Games/core/Game.js broadcast`, `Games/core/Message.js` / `Player.hear` |
| 14 | **pm2 cluster mode for `www`** (`instances: "max"`), with `modules/periodic.js` and `startup.js` guarded to run on instance 0 only | Linear API capacity with cores (www is single-threaded and does gzip + JSON on the main thread) | M | Med. Periodic jobs (expireGames, refreshHearts, stock evaluation) would otherwise run N times. The game load balancer is per-process already. | `pm2-prod.json`, `bin/www`, `modules/periodic.js` |
| 15 | **Reduce MUI `sx` usage in hot components** (message rows, player list, Main layout `Stack sx`) by switching to plain classNames or `styled()` | emotion `serializeStyles` + `styleFunctionSx` were ~35% of a chat-append profile, and they also dominated #3002's profile trace | M–L | Low | `pages/Game/*`, `Main.jsx` |
| 16 | **Phone/desktop layout swaps remount pages**: crossing the 900 px (`md`) breakpoint re-fires 6–14 API calls per page (42 on `/play` across 3 resizes) and reconnects the lobby chat. Render one tree and change styles, or keep the data-fetching components above the layout switch. | Avoids refetch storms on rotation and window resize on tablets and large phones | M | Low | `pages/Play/*`, `pages/User/Profile.jsx`, `Header` in `Main.jsx` |
| 17 | **websocket `perMessageDeflate`** (ws server, with `threshold: 1024` and `serverNoContextTakeover`) | Large frames (history, meetings) compress 5–10x. Only worth it after #11, and costs CPU and memory per socket. | S to try, M to tune | Med | `lib/sockets.js`, client `Socket.js` |
| 18 | Minor N+1s: `Game.js` game end does a sequential `User.findOne` per player and spectator, and `periodic.expireGames` does a per-game `ArchivedGame.findOne` plus a per-player `updateOne` every 10 min. Use one `$in` query and `bulkWrite`. | Tens of ms per game end, less DB churn | S | Low | `Games/core/Game.js` ~L3966, `modules/periodic.js` ~L80 |

### Suggested order

Do 1–4 first: they are on this branch and just need review. Then 5, 7 and 8, which are a day of work together. After that, 11 and 10 are the two items that change how a long game *feels* on a phone. 12 and 14 are capacity work for when traffic grows.

---

## Not measured, and caveats

* **Production data volumes and indexes.** I have no prod DB access, so the index wins are measured on a 40k-game seed that sits in RAM. Prod is likely bigger and partly cold, which makes the gain larger. Verify with `getIndexes()` and `explain()` there before building.
* **Real phones and network.** All frontend numbers use Lighthouse/CDP throttling on the box, not a device. Repeat-visit caching couldn't be separated in-session, because Chrome's heuristic cache masks it. The immutable-cache benefit mostly appears after a deploy or after heuristic expiry.
* **Game page Lighthouse.** This needs a live game, so it wasn't run. The game chunk sizes are reported instead.
* **Many concurrent games.** The game server was profiled with one in-process game, not 30+ simultaneous games over real sockets. ws framing and kernel cost aren't included.
* **`test/Games/Mafia.test.js` OOM.** I didn't reproduce it. Likely contributors: `TestSocket.clientMessages` keeps every frame for each test user (a 1,000-msg game is ~100k frames), and the test games are never torn down.
* The Umbrel stacks were deliberately not touched. Their numbers come from earlier in this workstream.

## Reproducing

```bash
# local stack (Mongo on :27027, Redis db 9, www :3200, games :3210, chat :3299)
node scripts/perf/seed.js 20000 40000 40          # writes /tmp/umperf-cookies.json
LIGHT_USER=<id> SETUP_ID=<id> node scripts/perf/benchRoutes.js 20
node scripts/perf/countQueries.js /api/user/perf1/profile /api/notifs
node scripts/perf/explainQueries.js
GAME_PORT=3290 node --cpu-prof scripts/perf/simulateGame.js 20 1000 5
cd react_main && npm run build && npm run perf:chat
```
