# Project Memory — Frau Pfeffer's Gallery Walk

This file exists so a future AI session (or a human) can pick this project up without
re-deriving decisions that were already made. It covers: why this project exists, what
was researched before building, how the current code works, bugs that were already found
and fixed (read this before "fixing" the same thing again), what's deliberately not built
yet, and environment quirks that cost real time to discover.

Last updated: 2026-09-19 by Claude Opus 5 in Claude Code (touch controls in §3, git and
GitHub notes in §6, Google Drive research in §8). Originally written the same day by
Claude Sonnet 5 in Anthropic's Cowork mode. Both sessions worked with the repository
owner.

**If you are picking this project up cold: read §8 first.** It holds unimplemented
research with two open decisions, and it is where work stopped.

---

## 1. What this is

A browser-based, keyboard-navigated 3D art gallery, generated from a local folder of
images. Point it at a folder of photos/artwork; it procedurally builds a maze of
corridors sized to the number of images, hangs one image per wall in file order, and
lets you walk through it with arrow keys / WASD. No upload, no backend, no build step —
`index.html` is the entire app.

Original motivating idea: a virtual gallery for **student art submissions** (a "digital
degree show"), navigated like the old Windows maze screensaver. The current repo is the
MVP proving out the core mechanic (folder → walkable maze); it is *not yet* the
multi-student submission platform that was the original end goal — see §5.

## 2. Research that came before this code

Before writing any code, a research pass was done across existing open-source projects
to avoid reinventing solved problems (originally published as an artifact titled
"Corridor & Canvas" — not saved as a file, so the key findings are captured here
instead). Condensed findings:

**Closest existing fork target found:** [`gecapistrano/museum-engine`](https://github.com/gecapistrano/museum-engine)
(Next.js + React Three Fiber, MIT). Drop images in a folder, it generates a room, hangs
and lights the prints, includes a visual layout editor and an admin upload panel. Floor
plan is declared as data (`SPACES[]` + doorways), walls double as collision geometry.
Its main gap for our use case: ships as a single enclosed room, not a corridor maze.

**Most mature no-code option:** [`lbartworks/openvgal`](https://github.com/lbartworks/openvgal)
("OpenVGAL", Babylon.js, MIT, ~50 stars, active since 2022). Browser-based generator at
openvgal.com/create turns image folders into a downloadable, self-contained gallery ZIP.
Built around rectangular halls with panel walls, not winding corridors — good reference
for artwork placement/lighting-bake technique, wrong shape for a maze.

**Best maze-generation reference:** [`majidmanzarpour/threejs-procedural-dungeon`](https://github.com/majidmanzarpour/threejs-procedural-dungeon)
("Dungeon Forge", MIT). Deterministic pipeline: scatter rooms → Delaunay triangulate →
reduce to a minimum-spanning-tree (guarantees connectivity) → add back some loops →
carve into a tile grid. The MST-plus-loops graph is the right algorithm if this project
ever needs *rooms of varying size* connected by corridors, rather than the uniform-grid
maze currently implemented (see §3 — current code uses a simpler DFS "perfect maze"
directly on a uniform grid, which was sufficient for the MVP and much less code).

**Other repos worth knowing about, roughly in relevance order:**
- [`khushishahxr/art-gallery-webgl`](https://github.com/khushishahxr/art-gallery-webgl) — A-Frame/WebXR, free-roam nav + collision + proximity audio narration. Small but the feature list matches this brief almost exactly.
- [`meir-schindler/virtual-gallery`](https://github.com/meir-schindler/virtual-gallery) — walkable photo gallery with a drag-and-drop wall-hanging tool; good pattern for letting a non-technical curator place their own work.
- [`rahel-yab/Virtual-art-gallery`](https://github.com/rahel-yab/Virtual-art-gallery) — small student project, cleanly split into `CameraController` / `LightingSystem` / `ArtworkManager` / `InteractionManager` modules.
- [`ptrgags/virtual-museum`](https://github.com/ptrgags/virtual-museum) — independent reference implementation of WASD + `PointerLockControls`.
- Three.js official FPS example (`games_fps.html`) — canonical `PointerLockControls` + `Octree` collision pattern. This project uses a *simpler* 2D circle-vs-segment collision instead of a full 3D Octree, since the maze floor plan is a flat grid (see §3).
- **Adjacent (non-OSS) platforms**, useful as a fallback/benchmark, not as code to reuse: Artsteps (free, best-for-education), Kunstmatrix (freemium, professional, 10-artwork free cap), Spatial.io (event-focused), Mozilla Hubs (Mozilla discontinued official hosting in 2024; third-party self-hosted forks exist).

**Gap identified across every project reviewed, including this one:** none solve
student-submission intake, moderation, per-student accounts/attribution, image
rights/licensing, or analytics. Rendering/navigation/lighting are solved problems;
content operations are not. If this project grows beyond a single-curator MVP, that's
the part actually worth engineering custom (see §5).

## 3. Current architecture (`index.html`)

Single self-contained HTML file. Three.js is loaded as an ES module directly from a CDN
at runtime (`https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js`) — **this
app needs an internet connection to start**, even though everything else (image
reading, rendering, all computation) is fully local. This was a deliberate tradeoff: the
sandbox this was built in has no general internet egress (see §6), so vendoring
Three.js as a local file wasn't possible from that environment. If full offline capability
matters later, download `three.module.js` once and reference it with a relative
`<script type="module" src="./vendor/three.module.js">` import instead of the CDN URL —
no other code changes needed.

**Folder selection:** a plain `<input type="file" webkitdirectory multiple>` — not the
File System Access API (`showDirectoryPicker`). Chosen deliberately for compatibility:
`webkitdirectory` works over `file://` (double-click the HTML file, no server needed) in
Chrome/Edge/Firefox/Safari, whereas `showDirectoryPicker` has secure-context quirks. It
filters to `/\.(jpe?g|png|webp|gif|bmp)$/i` and sorts by `webkitRelativePath` for a
stable, deterministic ordering.

**Maze generation (`generateMaze(n)`):** a randomized depth-first "perfect maze" (a
spanning tree — fully connected, no loops, has dead ends) carved into a grid, sized to
`n` = number of images. Grid bound is `ceil(sqrt(n)) + 3` in each direction. Each cell
records which of its 4 compass sides (N/E/S/W) connect to a visited neighbor
(`openSides` map); every non-open side becomes a wall. This was verified with a
standalone Node test (connectivity + no duplicate cells) at n = 1, 2, 5, 20, 100, 220 —
all passed. This is intentionally simpler than the Delaunay/MST approach used by
`threejs-procedural-dungeon` (see §2): a uniform-grid DFS maze was enough to deliver the
"winding corridors" feel for the MVP, at a fraction of the code.

**Geometry:** one cell = one `CELL x CELL` (4m x 4m) room with a floor, ceiling, and a
wall on every closed side. No shared/merged geometry or instancing — for the realistic
size of a folder-based gallery (tens to low hundreds of images) plain per-wall meshes
with shared materials/geometries are fast enough; this was a deliberate choice to keep
the code simple, not an oversight. Revisit with `InstancedMesh` only if profiling shows
it's actually needed at a larger scale.

**Artwork placement:** walks the maze cells in **DFS visit order** (the order they were
carved, which reads as a natural tour), and for each cell hangs the next image on the
first available closed wall. This means the gallery's walking order roughly follows the
original file order — a deliberate, load-bearing detail, not a side effect.

**Textures:** `createImageBitmap(file)` (off-main-thread decode) → drawn onto a `<canvas>`
downscaled to a max dimension of 1600px → `THREE.CanvasTexture`. This keeps large phone
photos from tanking performance. **Mipmaps are explicitly disabled**
(`generateMipmaps = false`, `LinearFilter` for both min/mag, `ClampToEdgeWrapping`) —
downscaled photos are almost never power-of-two dimensions, and some WebGL contexts
render an NPOT texture with mipmapping enabled as solid black. This was a real, deliberate
fix (see §4) — don't re-enable mipmaps without re-testing on the same class of hardware.

**Collision:** 2D circle-vs-line-segment math against a flat list of wall segments
(`wallSegments`), not a 3D `Octree`. This is correct and sufficient *because* the maze is
a single-story flat grid — there is no vertical geometry to collide with. If the project
ever adds multiple floors, ramps, or non-grid-aligned geometry, this collision approach
will need to be replaced with something 3D-aware (e.g. the `Octree` pattern from the
official Three.js FPS example, referenced in §2).

**Controls:** Arrow keys turn + move (classic maze-screensaver feel); WASD adds strafing;
click the canvas for pointer-lock mouse-look (optional, layers on top of the above,
doesn't replace it). Movement uses velocity lerping for smooth acceleration/deceleration
rather than instant on/off movement.

**Touch controls (added 2026-09-19).** Phones have no keyboard and no pointer-lock, so
before this the app rendered fine on mobile and was then completely unnavigable. Two
additions, both deliberately feeding the *existing* input seams rather than adding a
parallel movement system:

- An on-screen 4-arrow D-pad (`#touch-controls`) writes into a `touchInput {fwd, turn}`
  object, which `updatePlayer()` sums into the same `turnInput` / `fwdInput` scalars the
  keyboard produces (clamped via `clamp1`). Collision, velocity lerping and rendering are
  untouched — which is *why* touch movement feels identical to keyboard movement. Don't
  "improve" this by giving touch its own movement path.
- Drag anywhere on the canvas mutates `yaw`/`pitch` directly, exactly like the existing
  `mousemove` handler, reusing the same ±1.3 pitch clamp.

Decisions worth not re-litigating:

- **On-screen ←/→ turn, they do not strafe.** Deliberate: a visitor who never discovers
  drag-to-look can still navigate the entire maze with the four buttons alone. Strafing
  is desktop-only. Making them strafe would turn the D-pad into a dead end for anyone who
  doesn't realize the screen is draggable.
- **Pointer Events, not Touch Events.** One code path for mouse and touch, and `pointerId`
  is what lets a thumb hold an arrow while a second finger drags to look.
- **Buttons release on `pointerup`, `pointercancel` AND `lostpointercapture`**, plus a
  `visibilitychange` handler that clears all held state. Dropping any of these is the
  classic cause of a stuck, permanently-walking player. If someone reports "it won't stop
  walking," look here first.
- **`setPointerCapture` on each button** so a thumb sliding off the button keeps the input
  held rather than sticking on.
- **Visibility is pure CSS:** `@media (hover:none) and (pointer:coarse)` — no UA sniffing,
  and desktop rendering is byte-identical to before. The canvas `pointerdown` handler
  bails on `pointerType === 'mouse'` so the desktop pointer-lock path is untouched.
- `touch-action:none` on the canvas and buttons is what actually stops iOS Safari
  hijacking drags as scroll/pinch-zoom — **not** `user-scalable=no`, which iOS has ignored
  since Safari 10. Both are set; only the former does the work.
- Mobile layout shuffles two existing elements out of the D-pad's way: `#reload-btn` moves
  to the top-left (it was at `left:18px; bottom:18px`, directly under the pad) and the
  minimap shrinks to 96px and moves top-right.

**Not yet verified on a real device** as of this writing — the implementation was
syntax-checked and served over LAN, but nobody had confirmed the feel of it on hardware.
`TOUCH_LOOK_SENS` (0.0045 rad/px, ~2× the mouse's 0.0022) is an educated guess and is the
first thing to tune if looking feels sluggish or twitchy.

**Minimap:** bottom-right canvas, redrawn every frame from the same `order` array the
maze generator produced — no separate data structure to keep in sync.

**Performance decisions, summarized:** no shadow maps, no mipmaps on artwork textures,
image downscaling to 1600px, a hard cap of `MAX_IMAGES = 220` files (extras are silently
dropped — worth surfacing this to the user in the UI if it ever becomes a real limit in
practice, it currently fails silently).

**Key tunable constants** (top of the `<script>` block): `CELL` (4m), `WALL_H` (3.2m),
`WALL_T` (0.15m), `EYE_H` (1.65m), `PLAYER_R` (0.35m), `MOVE_SPEED` (3.0 m/s),
`TURN_SPEED` (2.0 rad/s), `TOUCH_LOOK_SENS` (0.0045 rad/px), `MAX_IMG_DIM` (1600px),
`MAX_IMAGES` (220).

## 4. Bugs already found and fixed — read before debugging the same symptom

**Symptom: artwork renders as solid black rectangles.** This happened twice in testing
and had **two different contributing fixes**, in this order:

1. First hypothesis (real fix, but not the actual cause of the reported bug): non-power-
   of-two canvas textures with mipmapping enabled can render solid black on some WebGL
   contexts. Fixed by disabling mipmaps and using `LinearFilter` (see §3). This shipped
   but **did not** resolve the user's reported black-image bug — worth knowing so this
   isn't "fixed" a second time chasing the wrong lead.
2. **Actual root cause:** the picture-frame mesh (a solid, slightly-larger-than-the-photo
   box, meant to look like a border) was positioned *in front of* the photo plane
   (closer to the camera) instead of *behind* it. Since the frame is both opaque and
   larger than the photo in every direction, it fully occluded the photo from every
   angle — what looked like "the texture isn't loading" was actually "you're looking at
   the frame, and the photo is hidden directly behind it." Fixed by flipping the sign of
   the frame's offset along the wall normal (`+= dx * 0.02` / `+= dz * 0.02` instead of
   `-=`), confirmed geometrically for all 4 wall directions with a standalone check
   before shipping.
   
   **Lesson for next time:** when a *textured* object appears solid black, check
   z-ordering / occlusion by nearby opaque geometry before assuming it's a texture
   loading, color-space, or WebGL capability problem. The texture-loading fix in step 1
   was reasonable defensive practice and stayed in the code, but it was not the bug.

## 5. What's deliberately NOT built yet (roadmap)

In rough priority order if this becomes more than a single-curator MVP. **Note:** a
curated Google Drive folder as the image source was researched on 2026-09-19 and is
written up in §8 — it is the most likely next feature, and it partially addresses items
1 and 5 below.

1. **Student submission workflow.** Nothing today lets multiple people contribute; it's
   "pick a folder you already have images in." A real version needs per-student upload,
   metadata (name/title/program/year), and a rights/consent checkbox.
2. **Moderation / review queue.** Nothing gatekeeps what gets hung before it's visible.
3. **Accounts & multi-tenancy.** No auth model at all currently.
4. **Deterministic/persisted layout.** The maze regenerates fresh (with a new random
   seed) on every page load — the *same* folder produces a *different* maze each time.
   Fine for a demo, probably wrong for a "send someone a link to walk through last
   week's show" use case. Fix: seed the RNG in `generateMaze` from something stable
   (e.g. a hash of the sorted file list) and/or persist the generated layout.
   **Gets more urgent alongside §8** — one shared canonical gallery means visitors will
   compare notes on "the room with the blue painting", which a reshuffling maze breaks.
5. ~~**Touch/mobile controls.**~~ **Done 2026-09-19** — see §3. What's still missing on
   mobile: strafing, a virtual joystick (the MVP is a 4-button D-pad), landscape-specific
   layout, and — the real blocker for phone visitors — a way to *load* images at all,
   since `webkitdirectory` folder-picking is poorly supported on mobile browsers. Touch
   navigation works; touch *ingestion* does not — **§8 would close that gap**, because
   images arriving from Drive means phone visitors never touch a folder picker.
6. **Analytics** (which pieces got looked at, dwell time) — not present in any reference
   project either, flagged as a general gap in §2.
7. Consider whether forking `museum-engine` (§2) becomes worthwhile once requirements
   grow past what this from-scratch single-file approach can comfortably hold — it
   already has an admin upload panel and an optional Postgres/Supabase + Prisma schema,
   which cover a meaningful chunk of items 1–3 above.

## 6. Repo, environment, and tooling notes

- **GitHub repo:** https://github.com/PfefferUndSalz/Frau-Pfeffers-Gallery-Walk — **public,
  and pushed as of 2026-09-19.** `main` is the default branch.
- **Local project path (canonical, as of 2026-09-19):** `~/Frau-Pfeffers-Gallery-Walk`.
  An earlier copy also exists in the Cowork session's temp/outputs folder from before
  this path was set up as the project home — that copy is stale and can be deleted.
- **License:** MIT (`LICENSE` file), copyright attributed to `PfefferUndSalz`
  (placeholder — amend if a different author/entity should be credited).
- **Git setup is done** (resolved 2026-09-19, natively in Terminal via Claude Code). The
  earlier sandbox failures are history, but the cause is worth remembering: running
  `git init`/`git commit` from inside Cowork's mounted filesystem failed with
  `Operation not permitted` on `.git/index.lock` and temp objects, because the mount
  blocks the unlink/rename operations Git needs internally. **If you're an AI in a
  sandboxed/mounted environment: don't retry `git` here, it will fail the same way — ask
  for it to be run natively.** A leftover 0-byte `.git/index.lock` from those failed
  attempts had to be deleted before the first real commit would go through.
- **GitHub auth gotcha.** The `gh` CLI on this Mac is logged in as
  `arthurallainfreitasdacosta`, but the repo is owned by `PfefferUndSalz`. That account
  was added as a *collaborator* with **write** access, which is enough to push but **not**
  enough to change repo settings — anything admin-level (enabling GitHub Pages, branch
  protection) returns a bare `404 Not Found` from the API rather than a clear permissions
  error. `git config user.name` is set to `PfefferUndSalz` locally, so commit authorship
  is correct regardless.
- **GitHub Pages is NOT enabled yet** (as of 2026-09-19). It was attempted and blocked by
  exactly the admin-permission gap above. The repo needs nothing done to it first —
  `index.html` is already at the root and no file is underscore-prefixed, so Jekyll won't
  skip anything and no `.nojekyll` is required. Enabling it is just
  Settings → Pages → Deploy from a branch → `main` / `/ (root)`, which would publish to
  `https://pfefferundsalz.github.io/Frau-Pfeffers-Gallery-Walk/`.
- **No package manager / build step / dependencies.** Everything is inline in
  `index.html`. Resist adding a bundler unless the project outgrows a single file —
  most of its value (zero-install, double-click to run) depends on staying this simple.
- **The AI sandbox this was built in had no general internet egress** (`curl`/`npm`/
  `pip` to npmjs.org, pypi.org, jsdelivr, unpkg, cdnjs all returned
  `403 blocked-by-allowlist`). That's why Three.js is loaded from a CDN at runtime by
  the *user's* browser rather than vendored into the repo at build time — the AI
  environment itself couldn't fetch the file to include it.

## 7. Files in this repo

- `index.html` — the entire application.
- `README.md` — user-facing description, controls, how to run.
- `LICENSE` — MIT.
- `.gitignore` — OS junk + a `vendor/` entry (reserved in case Three.js is vendored
  locally later, per §3/§6).
- `MEMORY.md` — this file.

## 8. Google Drive integration — research, 2026-09-19 (NOT yet implemented)

Everything in this section was **empirically tested on 2026-09-19** against the real
folder, not recalled from documentation. Two architectural decisions were still open when
the session ended, so no code was written. Resume here rather than re-running the probes.

### The goal

Students open a URL and are already standing in a curated gallery — no folder picker, no
clicks. A teacher curates by adding or removing files in one public Drive folder. Keeping
the folder URL in a repo config file (editable through the GitHub web UI) was a stated
second priority, behind "make it load from Drive at all".

Test folder used throughout: `1CV4Kye6PTvj0S8gOvELFJaOqKa5hYL2S` — 27 PNG screenshots,
shared "anyone with the link".

### The one finding that shapes everything

**A browser can freely download Drive images, but cannot list a Drive folder.**

- Image bytes: CORS-open, no credentials needed.
- Folder listing: no CORS-open, credential-free endpoint exists.

So the listing has to happen **somewhere other than the visitor's browser** — i.e. a build
step — *unless* an API key is embedded in the page. That constraint, not taste, is what
forces the architecture choice below.

### Measured CORS matrix

Probed with `Origin: https://pfefferundsalz.github.io`:

| Endpoint | Status | `access-control-allow-origin` | Usable from browser? |
|---|---|---|---|
| `lh3.googleusercontent.com/d/<ID>=w1600` | 200 `image/png` | `*` | **yes** — use this for images |
| `drive.google.com/thumbnail?id=<ID>&sz=w1600` | 302 → 200 `image/png` | `*` | yes, redirects first |
| `www.googleapis.com/drive/v3/files?…&key=` | 400 (bad key) | echoes the origin | yes, but **needs a key** |
| `drive.google.com/embeddedfolderview?id=<ID>` | 200, 23,536 bytes HTML | **absent** | **no** — server-side only |
| `drive.google.com/uc?export=view&id=<ID>` | **403** | — | **no** — dead |

`uc?export=view` is all over older tutorials and **no longer works**. Don't spend time on
it. Calling `files.list` with no key at all returns a clear 403: *"Method doesn't allow
unregistered callers."*

### Why the existing texture pipeline needs no changes

`loadDownscaledTexture(file)` (§3, `index.html:385`) opens with `createImageBitmap(file)`,
which accepts **any Blob** — not just a `File` from the picker. So
`fetch(url) → .blob() → loadDownscaledTexture(blob)` reuses the whole existing path:
downscaling, `CanvasTexture`, the `generateMipmaps = false` fix from §4, all of it. The
Drive work is therefore an *ingestion* change, not a rendering change.

This also **sidesteps canvas tainting**, which would otherwise be the blocker: drawing a
cross-origin `<img>` onto a canvas taints it, and WebGL then refuses that canvas as a
texture — which would surface as the same solid-black artwork symptom documented in §4,
sending a future debugger down entirely the wrong path. Bytes pulled through `fetch()`
carry no taint. **Do not later "simplify" this into `THREE.TextureLoader().load(url)` or
an `<img>` tag** — that reintroduces exactly the problem the Blob route avoids.

### `embeddedfolderview` scraping — what actually comes back

Parseable HTML, one entry per file:
- File ID from `https://drive.google.com/file/d/<ID>` links — 27 unique IDs recovered
  cleanly from the test folder.
- Filename from `class="flip-entry-title"` — e.g. `Screenshot 2026-09-19 at 14.18.50.png`.

Caveats: **undocumented**, so Google can change the markup without warning; direct
children only, no recursion; behaviour on folders large enough to paginate was not tested.

### Drive API v3 with an API key — confirmed viable

Works against "anyone with the link" folders without OAuth:
`GET https://www.googleapis.com/drive/v3/files?q='<FOLDER_ID>'+in+parents&key=<KEY>`

Gotchas worth pre-empting:
- Default projection is sparse — pass `fields=nextPageToken,files(id,name,mimeType)`.
- Default `pageSize` is 100; use `pageSize=1000` plus a `pageToken` loop.
- Non-recursive; subfolders show up as `mimeType = application/vnd.google-apps.folder`.
- Shared Drives need `supportsAllDrives=true&includeItemsFromAllDrives=true`.
- Google-native files (Docs/Sheets) reject `alt=media` — irrelevant for photos.
- Any key must be restricted to **Drive API only + an HTTP-referrer restriction**.

### Image sizing — measured, and it matters on phones

`lh3.googleusercontent.com/d/<ID>=w<N>` caps width and never upscales:

| Request | Result | Bytes | × 27 images |
|---|---|---|---|
| `=w800` | 800 × 968 | 628 KB | **~17 MB** |
| `=w1600` | 1274 × 1542 (native) | 1.93 MB | **~52 MB** |
| `=w2048` | identical to `=w1600` | 1.93 MB | ~52 MB |

52 MB is not acceptable over mobile data, and the app already downscales to
`MAX_IMG_DIM = 1600` internally — so requesting full size spends bandwidth on pixels that
are immediately discarded. Request `=w1600` as the ceiling, drop to `=w800` if load time
disappoints, and make it a named constant next to `MAX_IMG_DIM`.

### The three candidate architectures

In all three, **images always stream live from Drive**; only the freshness of the
*filename list* differs.

1. **GitHub Action → `gallery.json`, scraping `embeddedfolderview`.** No credentials
   anywhere, no Google Cloud setup, same-origin fetch at runtime, survives a Drive outage
   on the last good list, and the committed JSON *is* the "config in the repo" that was
   asked for. Cost: new photos appear only after the next run; leans on undocumented HTML.
2. **GitHub Action → `gallery.json`, via the Drive API with the key in GitHub Secrets.**
   Same shape, official API, key never in the repo. Cost: one-time GCP setup.
3. **Runtime Drive API with a referrer-restricted key in `index.html`.** Instantly live,
   no build step. Cost: a credential visible in a public repo, plus an extra round-trip
   before the gallery can start.

### Open decisions — settle these before writing code

1. **Which architecture.** Leaning (1): keeps a credential out of a public repo and
   matches the "config file in the repo, editable via GitHub" framing.
2. **Fate of the local folder picker.** Leaning: Drive auto-loads on open, picker demoted
   to a secondary "or browse a local folder" link — preserves offline testing and deletes
   no working code.
3. **Surfaced by this work, not yet decided:** roadmap item 4 (seeding the maze RNG from
   the file list) gets materially more important once there is a single canonical shared
   gallery. Students will compare notes on "the room with the blue painting", and today
   every reload builds a different maze.

Worth noting: this change would also resolve the mobile ingestion gap in roadmap item 5 —
students on phones never touch `webkitdirectory` if the images arrive from Drive.

### Sources

- [Using the Google Drive API for public folders — Nick Felker](https://fleker.medium.com/using-the-google-drive-api-for-public-folders-f1f7308385ad)
- [Use Google Drive public folder without authentication using API](https://medium.com/@patrabiswajit133/use-google-drive-public-folder-without-authentication-using-api-8ea71ad90dcd)
- [Accessing public Google Drive files via API without login](https://community.latenode.com/t/accessing-public-google-drive-files-via-api-without-login/32858)
