# Samurai Dojo ⚔

A 60-second mixed-reality physics game for **Figmin XR**. Grab a katana, throw shurikens and topple as many cardboard samurai as you can — but don't hit the bomb.

## Play

1. Make sure GitHub Pages is on for this repo (**Settings → Pages → Branch: main, folder: / (root) → Save**).
2. In Figmin XR's browser, open `https://<your-github-name>.github.io/<repo-name>/`.
3. Accept the prompt to load it as a **spatial app**.
4. Tap the **controller connect icon** on the panel. (Until then the katana is locked: in Figmin, pulling the trigger while holding an object makes a copy of it.)
5. Pick a **spawn level** (1 Careful = only clear floor well inside the room, 2 Busy, 3 Wild = more samurai, faster, right up to and slightly past the scanned walls).
6. Grab the katana and press **Start Game**.

## Controls

| Action | Input |
|---|---|
| Swing sword | Grab the katana and swing it |
| Throw shuriken | A shuriken floats up near you every 5 seconds: grab it with the **grip button** (middle finger), swing your arm and let go |

## Scoring

- Cardboard samurai: **1 point** each
- Colourful armoured shogun (axe or spear): **3 points**
- Monk or geisha: **−2 points** — spare them!
- Only figures you actually hit count (a falling samurai knocking over another counts too)
- Black bomb: hitting it **ends the game**

## Files

- `index.html` — the whole game (single self-contained page)
- `models/game/` — the 3D models (GLB) the game loads (see `models/README.md`)
- `images/logo-scroll.png` — the NTT DATA logo on a paper scroll, shown at the top of the menu and floating in the background during a round (`images/Logo_Global_NTT_DATA_Future_Blue_RGB.png` is the original)
- `audio/final-taiko-clash.mp3` — background music, plays from the countdown until the round ends
- `CLAUDE.md` — project notes for continuing development with Claude Code

Built with the [Figmin XR Scripting API](https://www.figmin.com/api/documentation.html).
