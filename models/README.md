# models/

## `game/` — the models the game actually loads

| File | Used as | Triangles |
|---|---|---|
| `katana.glb` | the sword (straightened: blade along +Y, tip up, centered) | ~10.7k |
| `samurai.glb` | normal samurai (1 point) | ~30k |
| `monk.glb` | innocent (−2 points) | ~30k |
| `geisha.glb` | innocent (−2 points) | ~30k |
| `samurai-warrior.glb` | shogun (3 points) | ~30k |
| `samurai-spear.glb` | shogun (3 points) | ~30k |

The game loads them from GitHub Pages:
`https://dtmika2.github.io/Samuraj-Dojo/models/game/<name>.glb` (see `MODEL_BASE` in `index.html`).
They must be committed and pushed before the game can use them.

How they were made from the Meshy originals (no Draco / WebP — Figmin support for those is unknown):

```
npx gltf-transform weld     original.glb  1.glb
npx gltf-transform simplify 1.glb 2.glb --ratio <30000 / original triangles> --error 0.05
npx gltf-transform resize   2.glb 3.glb --width 1024 --height 1024
npx gltf-transform jpeg     3.glb out.glb --quality 85
```

The katana was additionally rotated so the blade points along +Y and re-centered.

## Other files

- `Meshy_AI_*.glb` — full-detail originals from Meshy (1.5–1.9 million triangles, 50–64 MB). Too heavy for the headset; keep only as sources.
- `Models Optimized/` — Draco + WebP compressed versions (still ~300k+ triangles). Not used by the game.

Figmin XR only loads **.glb** (not .gltf).
