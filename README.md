# Samurai Dojo ⚔

A 60-second mixed-reality physics game for **Figmin XR**. Grab a katana, throw shurikens and topple as many cardboard samurai as you can — but don't hit the bomb.

## Play

1. Make sure GitHub Pages is on for this repo (**Settings → Pages → Branch: main, folder: / (root) → Save**).
2. In Figmin XR's browser, open `https://<your-github-name>.github.io/<repo-name>/`.
3. Accept the prompt to load it as a **spatial app**.
4. Tap the **controller connect icon** on the panel. (Until then the katana is locked: in Figmin, pulling the trigger while holding an object makes a copy of it.)
5. Grab the katana and press **Start Game**.

## Controls

| Action | Input |
|---|---|
| Swing sword | Grab the katana and swing it |
| Throw shuriken | Trigger (or A / X) on that hand |

## Scoring

- Cardboard samurai: **1 point** each
- Colourful armoured shogun (axe or spear): **3 points**
- Monk or geisha: **−2 points** — spare them!
- Only figures you actually hit count (a falling samurai knocking over another counts too)
- Black bomb: hitting it **ends the game**

## Files

- `index.html` — the whole game (single self-contained page)
- `models/game/` — the 3D models (GLB) the game loads (see `models/README.md`)
- `CLAUDE.md` — project notes for continuing development with Claude Code

Built with the [Figmin XR Scripting API](https://www.figmin.com/api/documentation.html).
