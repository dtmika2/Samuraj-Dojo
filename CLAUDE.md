# CLAUDE.md — Samurai Dojo (Figmin XR spatial app)

This file hands the project over from a claude.ai chat to Claude Code. Read it fully before changing anything.

## What this project is

**Samurai Dojo** is a mixed-reality physics game that runs inside **Figmin XR** (an XR app for Meta Quest, SteamVR, Windows, visionOS, iOS). It is a single HTML page (`index.html`) that uses the **Figmin XR Scripting API** (global `figmin` namespace) to create and control 3D objects in the player's real room.

Gameplay:
- The player grabs a **katana** (floats next to the app window) and swings it.
- Every 5 s a **shuriken** appears floating within arm's reach; the player grabs it with Figmin's grip (middle-finger) grab and throws it with a real arm swing. (The old trigger-throw was removed on user request, 2026-10-21: "shurikens still act weird".)
- **Cardboard figures** (3D models) pop up out of the floor in an arc in front of the player. Toppling one scores points: samurai = 1, colourful armoured "shogun" (axe warrior / spear samurai) = 3, innocent monk / geisha = −2 (score may go negative). Only falls the player caused count (see "hit credit" below).
- A round lasts **60 seconds** (3-2-1-"HAJIME!" countdown first).
- A **black bomb** appears now and then; hitting it (sword or shuriken) ends the game early ("BOOM!"). Can be disabled with a checkbox.
- Best score is persisted with `figmin.saveState` / `loadState`.
- Background music plays from the countdown until the round ends.
- **Spawn level** (1 Careful / 2 Busy / 3 Wild) picked in the menu, saved with the best score.
- NTT DATA logo (sponsor/branding) on the menu and in the room.

## The user

- Not a professional programmer. Explain in plain steps, avoid jargon, give exact click paths.
- Writes in English; their Discord/OS UI is in **Czech** (e.g. "Stáhnout" = Download, "Kopírovat odkaz" = Copy link).
- Timezone: Europe/Prague.
- Tests on a **Meta Quest 3** in Figmin XR. Cannot easily read logs unless told how (see Debugging).

## Figmin XR API — essentials

Official docs: https://www.figmin.com/api/documentation.html
AI skill file: https://www.figmin.com/api/SKILL.md — **fetch and read this before making API changes**; it is the most compact and accurate reference. A copy (2026-10-22) is in `docs/FIGMIN_SKILL.md`; re-fetch if it may be stale.

Rules that bite (all from the docs):
1. Page must include `<meta name="figmin">` and wait for the `FigminReady` DOM event before calling `figmin.*`.
2. Most features need the page **deployed as a Spatial App** (`figmin.isSpatialApp()` / `figmin.requireSpatialApp()`).
3. **Never mutate vectors in place.** `obj.transform.position.x = 1` does nothing. Always assign a full new `{x,y,z}` / `{x,y,z,w}` object. Same for physics `velocity`, `angularVelocity`, etc.
4. Vectors are plain objects — do the math manually (see `V` helpers in index.html).
5. Use `figmin.setUpdateFunction(dt)` or `obj.onUpdate(dt)` for per-frame logic, not `requestAnimationFrame`. Keep transform writes to roughly ≤20 objects/frame (the JS↔engine bridge is slow).
6. Objects are only safe to use after `obj.onStart(...)` (state READY). Add components there.
7. `SKETCH` objects **must** receive `brushStrokes` in `createObject`, or they stay in LOADING forever.
8. `TEXT3D` readable face is opposite `transform.forward`; copying the window/camera yaw makes it face the user.
9. Physics needs collider BOX / SPHERE / APPROXIMATE (not EXACT / NONE). Physics and parenting are mutually exclusive.
10. Controller buttons are only readable after `figmin.enableInputConnect(...)` and the player taps the connect icon on the window. `grabbableObjects` option restricts what can be grabbed while connected.
11. Coordinate system: +X right, +Y up, +Z forward (Unity style), meters, Euler degrees. World origin = room center at floor level (y = 0 is the floor).
12. `MODEL3D` supports **GLB only** (not .gltf). Model URL is fixed at creation.
13. Multiplayer: only the master client (app creator) can create/modify objects; the app early-outs for non-masters.
14. `app.loadHTML` / embedding: literal emoji may not survive; use HTML entities (index.html already does).

## How index.html is organised

Top of the `<script>`: a **TUNING** block of constants (round length, pool sizes, spawn distances, shuriken speed, fall threshold, bomb timing, `SPAWN_LEVELS`, text sizes / `TEXT_MATERIAL`, logo sizes and placement). Change gameplay feel there. It also holds the model setup: `MODEL_BASE` (full Pages URL — Figmin may embed the page, so relative URLs aren't safe), `SAMURAI_HEIGHT`, `KATANA_LENGTH`, `NORMAL_MODELS` / `SHOGUN_MODELS` (which model fills each pool slot), `MODEL_WIDTH_FIX` (targetSize scales the *largest* side; the spear model is wider than tall), and the blade/hit-volume fractions.

Sections, in order:
- Vector helpers `V`, `clamp`, `rand`, `segSegDist` (segment–segment distance, used for sword hit tests).
- DOM refs (`ui`) and `game` state object (`phase`: loading → ready → countdown → playing → over).
- `init()` on `FigminReady`: spatial-app + master checks, load best score, build everything, set update loop.
- `trackLoad(obj, onReady, label, makeBackup?)`: counts objects to READY; per-group report; tolerant of setup errors (try/catch) and load errors; after 15 s of loading the status box lists which group is stuck. If a MODEL3D fails (`onError`) and `makeBackup` is given, it creates the sketch version instead and the report says "using backup shape". `onReady` must use the object it receives (it may be the backup).
- Geometry builders (procedural sketch strokes): `katanaStrokes`, `samuraiStrokes(isShogun)` — only used as backups for failed models — plus `shurikenStrokes`, `bombStrokes` (always used).
- Object pools created once at startup (hidden at y = -20, `active = false`): katana (1, MODEL3D, APPROXIMATE collider — was EXACT, but the ~10.7k-triangle mesh collider caused heavy lag while the blade was inside a samurai (user, 2026-10-22: "heavy lag on collision… when I keep hitting the same target"); BOX was ~10 cm thick; don't go back to EXACT), samurai records (6 normal + 2 shogun + 2 innocent, MODEL3D, APPROXIMATE (convex) collider — BOX was far too big around arms/spears/banners; sketch backups use BOX; `rec.kind` / `rec.points`), shurikens (8, scaled ×`SHURIKEN_SCALE`, Light2 trail only while thrown), bomb (1), scoreboard text, banner text, "+1"/"+3"/"-2" popup texts (4 + 2 + 2), 3 logo images. Total 33 objects.
- Samurai records carry `halfH` / `footR` read from `transform.size` after load (models are centered on their bounding box). Spawn spacing uses `footR` so wide colliders (axe, spear) don't overlap.
- Spawning is room-aware and async: `measureFloor()` at Start probes the real floor under the player (`game.floorY`; a freshly mapped Figmin scene isn't always at y = 0). `spawnSamurai` picks a direction/distance (`rec.pending`); `checkSpawnSpot` probes (`raycastSphere`, `Layers.SURFACE`) from the Start position at waist height along that direction — if a wall/furniture is closer, the samurai is placed between `SPAWN_ROOM_MIN` and the wall (minus its footprint) instead of being rejected; then probes down for the floor there (> floorY + 0.2 = table/bed = blocked). Rejections (`spawnBlocked`) retry after `SPAWN_RETRY`, widen the arc, and after 12 in a row with an empty field show a "not much free floor" hint. Then `placeSamurai` rises it out of the floor and turns physics on (`rec.physicsAt`).
- History: v3 rejected any spot with a wall closer than the wanted distance and required floor |y| < 0.2; after the user switched Meta to roomscale and mapped a new Figmin scene, almost every spot was rejected ("maybe 2 per minute"). Without a room scan there are no SURFACE colliders, so the checks always pass.
- **Hit credit** (`rec.hitAt`): set by `knock()` (shuriken / katana), by the katana geometric check even at low speed, and by a samurai being bumped by another samurai hit within `HIT_CREDIT_SECONDS` (chain reaction; looked up by name in `samuraiByName` — samurai are named `DojoSamuraiN`, bomb `DojoBomb`). Falls without credit score nothing (console warn). A samurai faster than `EJECT_SPEED` within 0.6 s of physics turning on, without credit, was pushed out by room geometry → hidden, console warn.
- Fallen when `transform.up.y < FALL_UP_Y` or dropped low → score (if credited) + popup → hidden after 1.6 s.
- Shuriken lifecycle `rec.state`: idle → floating (`spawnFloatingShuriken`, bob + spin, kinematic, ENABLED if connected, max `SHURIKEN_MAX_FLOAT`) → held (`bodypart.onGrabChanged` on both hands → `onShurikenGrab`; positions sampled each frame) → thrown (`throwHeldShuriken`: velocity = displacement over the last ~0.1 s × `THROW_BOOST`, clamped; below `THROW_MIN_SPEED` it's a drop and `live` stays false so it can't score) → idle after `SHURIKEN_LIFETIME`. Nothing here is verified on device: if `onGrabChanged` doesn't fire for these objects, or Figmin applies its own release velocity, revisit.
- `enableInputConnect` is called from `onAllLoaded` (`setupInputConnect`) so `grabbableObjects` can list the katana and all shurikens. Trigger/A/X are still requested only so Figmin doesn't act on them.
- Controller connection gates the game: katana is `LOCKED` until `enableInputConnect` reports connected (then `ENABLED`), Start button disabled until connected (`updateStartButton`). Reason: unconnected, Figmin owns the trigger, and trigger-while-holding duplicates the object (Tilt Brush heritage) — the user saw the katana "clone itself and stay in place". Grip inputs are not requested.
- Katana model is pre-straightened: blade along local +Y, tip up, centered; `katanaSegment()` uses `BLADE_*_FRAC × transform.size.y`.
- **Spawn levels** (`SPAWN_LEVELS`, added 2026-10-22 on user request: "how forgiving the rules about spawning should be together with frequency"): level 1 = the earlier fixed values. Each level sets spawn `interval`, `startAlive`, `rampSeconds`, `arcDeg`, `roomMin`, `wallSlack` (added to the room left before a wall, so samurai may stand closer to / slightly past it; the margin `footR + 0.15` is conservative because `footR` is the *widest* side while the depth toward the wall is the thin side) and `floorTol` (floor bumps accepted by the down probe). `game.level` is read from the radio buttons at Start (`lvl()`), saved in the `saveState` JSON as `level`. Spots that end up inside room geometry are still removed by the ejection check.
- **Logos**: `images/logo-scroll.png` (generated from the original NTT DATA PNG: logo colours unchanged, on a paper scroll with wooden rollers). Three IMAGE objects (`logos.menu` above the scoreboard, `logos.banner` shown/hidden with the countdown/result banner, `logos.bg` hung `LOGO_BG_DIST` ahead at `LOGO_BG_HEIGHT` during a round, closer if a wall raycast hits, slow bob), collider NONE, tracked as load group "logo". Image facing is unverified: `LOGO_TURN = 180` flips them all. The HTML menu shows the same image at the top.
- **Floating texts** use `TEXT_MATERIAL = "unlit"` and larger sizes (2026-10-22: user said they were too small and the "fire" / "diamond" materials looked see-through). The result banner uses the smaller `RESULT_SIZE` because the text is long.
- Hits: `knock(rec, dir, strength)` sets velocity + angular velocity to tip the samurai in the push direction.
  - Collision callbacks (`rec.onHit`) request only `objectName` and are registered by `listenForHits` only while needed: samurai from the end of their rise until they fall / hide, shurikens while thrown and live. Figmin only computes collision events while a callback is registered, and each event crosses the slow JS bridge (part of the 2026-10-22 lag fix).
  - Shuriken → its `onHit`, target looked up via `samuraiByName`.
  - Katana → both a collision callback on samurai (by object name `DojoKatana`) **and** a geometric check every frame (blade segment vs samurai axis within `rec.hitR` = 60 % of half the body's slimmer side, clamped 0.10–0.22 m — was a fixed 0.30 m, user: "the katana has a too big hitbox"; knocks only when blade-tip speed > `KATANA_MIN_SPEED`). The geometric check exists because a grabbed non-physics object may not generate collisions.
- Bomb: static, bobbing, fuse trail (Embers) + pulsing light. Shuriken collision or katana geometric check → `endGame("bomb")`.
- `tick(dt)`: katana motion → shurikens → samurai → popups → countdown → playing (timer, spawn ramp, floating-shuriken timer, bomb, HUD, end on timer).

## Current status (as of 2026-10-21)

- **Not yet confirmed working on a device.** Nothing has been playtested past loading.
- v1 got stuck at "Preparing the dojo… 18/26". 26 − 18 = 8 = the samurai pool, so the samurai `onStart` setup most likely threw. Suspected cause: setting `o.physicsMaterial.*` right after adding physics, before that component existed on the JS side.
- v2 (this `index.html`) fixes that by adding `PHYSICS_MATERIAL` (with params) **before** `PHYSICS`, and makes loading error-tolerant with a per-group report. **This fix is unverified** — first thing to do is have the user load it and report the status text.
- Physics tuning (knock strength, masses, fall threshold, spawn distances) is untested guesswork; expect to iterate with the user.
- Music: `audio/final-taiko-clash.mp3` (user's track, 3:48, MP3) on an AUDIO component of the scoreboard (`setupMusic`), non-spatial, no loop, volume `MUSIC_VOLUME`; `startMusic()` (stop + play = from the start) in `startGame`, `stopMusic()` in `endGame` (time or bomb). Loaded from `MUSIC_URL` on Pages; the game works if it fails (console error). No sound effects yet.
- **3D models are wired in (v3, 2026-10-21), untested on device.** Meshy models simplified to ~30k triangles (katana ~10.7k), plain JPEG textures, no Draco/WebP; see `models/README.md`. They load from `MODEL_BASE` on GitHub Pages, so they only work after a push with Pages enabled. Unknowns to check on the headset: models actually load (else status shows "using backup shape"), model front faces the player (assumed +Z, verified only in offline renders), BOX colliders of wide models topple sensibly, overall frame rate with 8 models.
- The script was smoke-tested in Node against a fake `figmin` mock (load, backup path, round, hits, bomb, not-connected, blocked spawn spot, ejection, self-fall) — this catches JS/logic errors only, not engine behaviour.
- User report on v2 (before these fixes): "some samurais were flying out on their own" and the katana "cloned itself and stayed in place" when grabbed. Clone = Figmin trigger-duplicate while not connected (fixed by locking). Flying: root cause not confirmed; most likely spawning inside room geometry (furniture/walls/floor above y=0) → physics depenetration. v3 adds room probes, ejection removal, and hit credit. If it still happens, ask for Console lines containing "pushed out" / "fell over without being hit" / "Spawn spot blocked".

- 2026-10-22 lag fix (convex katana collider, fewer collision callbacks) confirmed on the Quest 3: "better now… still a bit of lags, but acceptable". If it needs more work: lower-poly samurai models (simpler convex colliders, ~30k tris each to render), fewer per-frame transform writes, smaller pools on level 3.

## Next tasks

1. ~~Host on GitHub Pages~~ — done 2026-10-21: Pages serves `main` / root at **https://dtmika2.github.io/Samuraj-Dojo/** (models at `.../models/game/`). Every push to `main` redeploys in ~1 min. The user opens that URL in Figmin's browser and accepts the spatial-app prompt. The full-detail Meshy originals are git-ignored (local only; a local branch `backup/start-of-project-with-originals` still holds the first commit that contained them — never push it).
2. **Verify the loading fix** with the user (status box should reach "Start Game"). If it reports failures, get Console errors (see Debugging).
3. **Verify the 3D models on the headset** (done in code, see Current status). If a model faces away, add 180 to `rec.yaw` for models; if toppling feels wrong for wide models, try `APPROXIMATE` colliders or a smaller `SAMURAI_HEIGHT`. New models: simplify the same way (`models/README.md`), inspect bounding box / facing / pivot before coding. None of the current models have animations.
4. Playtest and tune constants in the TUNING block.
5. Optional: sound (audio component, `.ogg` recommended — files need permanent hosting too, e.g. this repo), difficulty levels, high-score table.

## Debugging with the user

- In Figmin, the app panel's settings have a **Console** toggle; turn it on, press reload, then use the copy button and paste errors to Claude. `figmin.showLogConsole()` opens it programmatically.
- The status box in the app UI shows loading progress and, after 15 s, a per-group report (e.g. `samurai 0/8 (8 waiting)`).
- Faster iteration option: Figmin XR on Steam (2D windowed mode, SteamVR off) lets the user copy console errors easily.
- Alternative to Pages during development: `python -m http.server 8000` on the PC and open `http://<pc-lan-ip>:8000/` in Figmin (same Wi-Fi; never publish apps pointing at LAN IPs).

## Lessons from getting the file into Figmin

- Links to files in a claude.ai chat don't work in Figmin's browser (not logged in) → "Image not found".
- LimeWire share links are landing pages with client-side decryption → likely don't work.
- Discord works only with the **attachment CDN link** (`https://cdn.discordapp.com/attachments/...samurai-dojo.html?ex=...&is=...&hm=...`, copied from the browser's downloads list via Ctrl+J), not the message link. These expire after ~a day.
- GitHub Pages is the long-term answer.

## Conventions

- Keep the game a **single self-contained HTML file** (inline CSS/JS) unless there's a good reason; Figmin can embed it directly.
- UI must be XR-friendly: big buttons (≥64 px), ≥16 px text, generous spacing — the user may use hand or eye tracking.
- Keep tunable numbers in the TUNING block with a comment each.
- After any change: syntax-check the script (extract `<script>` contents and run `node --check`), commit, and tell the user to reload the Pages URL in Figmin.
