# Frau Pfeffer's Gallery Walk

Turn a folder of photos into a walkable 3D art gallery — no upload, no server, no account. Pick a folder, and the app procedurally builds a small maze of corridors sized to your collection, hangs each image on a wall in file order, and lets you explore it on foot.

## Features

- **Folder in, gallery out.** Select any local folder of images and the app builds a room for each one.
- **Real maze corridors**, not a single flat room — a randomized, fully-connected maze generated fresh from your image count.
- **Keyboard-first navigation.** Arrow keys or WASD to walk, click to look around with the mouse, minimap in the corner so the maze doesn't disorient you.
- **Runs entirely on your machine.** Your images never leave your computer — everything is read locally in the browser.
- **Lightweight by design.** Large photos are downscaled before becoming textures, there are no shadow maps, and collision uses simple 2D wall-segment math instead of a heavier physics engine.

## Getting started

1. Download or clone this repository.
2. Open `index.html` in a modern desktop browser (Chrome, Edge, or Firefox recommended).
3. Click **Select a folder of images** and choose a folder on your computer.
4. Walk through your gallery with the arrow keys or WASD; click the window to enable mouse-look.

> **Note:** the app loads the [Three.js](https://threejs.org/) 3D engine from a CDN the first time it runs, so it needs an internet connection to start. Your images themselves are never uploaded anywhere.

## Controls

| Input | Action |
|---|---|
| `↑` / `W` | Walk forward |
| `↓` / `S` | Walk backward |
| `←` `→` | Turn |
| `A` / `D` | Strafe |
| Mouse (after clicking) | Look around |
| `Esc` | Release the mouse |

## How it works

- A randomized depth-first "perfect maze" is carved into a grid, sized to the number of images selected.
- Each visited cell becomes a room; every wall the maze doesn't open into a neighboring room gets a wall, and one image is hung on an available wall per cell, in file order — so the tour roughly follows your photos' original ordering.
- Movement uses simple circle-vs-line-segment collision against the generated walls, smoothed with velocity lerping for a natural feel.

## Roadmap ideas

This is an MVP. Natural next steps if you want to take it further:
- A submission/upload workflow for multiple contributors, with moderation before an image goes live.
- Persisting a generated layout (so the same folder always produces the same maze) instead of regenerating on every load.
- Touch controls for mobile/tablet visitors.

## License

MIT — see [LICENSE](LICENSE).
