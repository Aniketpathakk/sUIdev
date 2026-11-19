# Swarm Rebuild — reverse-engineering notes

## What it is

Swarm Rebuild is a Manifest V3 Chrome extension (version 1.1.0). Pressing its toolbar button or **Alt+Shift+S** runs a local visual reconstruction of the active page. It briefly hides the live page, builds a visual blueprint from its rendered DOM, and animates simulated cursors placing the page back on a canvas. It does not send the page to a server, and the agents are simulated in the tab rather than connected to other people or machines.

## Source files

- [`manifest.json`](rebuilt-swarm-source/manifest.json) — extension identity, icon, command, and permissions.
- [`background.js`](rebuilt-swarm-source/background.js) — toolbar entry point, special-media screenshot capture, script injection, and failure cleanup.
- [`swarm.js`](rebuilt-swarm-source/swarm.js) — page capture, canvas-like reconstruction view, cursor/worker simulation, controls, and generated sound.
- [`icons/`](rebuilt-swarm-source/icons/) — toolbar icons.

The installed copy was found in Chrome profile **Default**, at extension ID `diaeaocnhdhbknkbfbnkjikfcgokainm`, version `1.1.0`. I copied its readable source into the output folder; I did not change or reinstall the installed extension.

## Runtime sequence

1. `background.js` receives the toolbar click (the shortcut invokes the same action). It checks whether a swarm is already running; clicking again toggles it off.
2. If the page contains visible canvas, video, or embedded content, the worker hides ordinary DOM elements for one frame, captures the visible tab, and passes that image to the content script. This supplies pixels that cannot be read from the DOM, such as cross-origin iframes and WebGL canvases.
3. `swarm.js` scans the document and records positioned boxes, computed styles, text, images, SVG, form controls, pseudo-elements, and layout relationships. It tries to find the page’s real scroll container and captures long pages in world coordinates.
4. It mounts a fixed overlay, restores visibility of any temporarily hidden media, and animates colored cursor agents as they place the captured elements. The original page is represented by visual clones while the animation runs.
5. On completion, **Go live** fades the overlay away to reveal the real page. **Replay** starts the reconstruction again. **Escape** or the close button exits the overlay. **Tour** auto-scrolls, the speed button cycles 1×/2×/3×, and **M** or the sound button toggles generated audio.

## How the “swarm” behaves

The agents are generated locally by the `Engine` class. Names and colors are chosen from built-in lists; cursor count changes with workload and motion, between 8 and 48 agents. There is no multiplayer service or network synchronization in the page scripts. The “multiplayer” effect is a visual simulation.

The overlay includes a blueprint/canvas view, rulers and alignment guides, text and shape placement, cursor movement, a progress bar, agent counters, and synthesized sound effects. The source also contains interactions for drawing, editing, and duplicating reconstructed elements.

## Data access and privacy

The manifest requests only `activeTab` and `scripting`. The extension reads the active page’s rendered DOM and computed styles when invoked. Password inputs are rendered as bullets in the reconstruction. The inspected `background.js` and `swarm.js` contain no `fetch`, `WebSocket`, `XMLHttpRequest`, `sendBeacon`, or storage calls; the captured page model and optional screenshot are passed in memory to the injected script. The manifest does include Chrome’s standard Web Store `update_url` for extension updates.

## Practical limits

- Pages with browser-protected URLs (such as `chrome://` and the Chrome Web Store) cannot be injected; the extension shows a temporary error badge.
- The capture has hard limits of 4,000 items and 40,000 CSS pixels of page height.
- Cross-origin and GPU-rendered media depend on a one-frame screenshot and can fall back to placeholders.
- This is a visual reconstruction, not a working copy of the original site's application. Links, forms, and other original page behavior are not the purpose of the animation.

## Main implementation map

- [`background.js`](rebuilt-swarm-source/background.js): `rebuild()` coordinates the tab and screenshot, then injects `swarm.js`.
- [`swarm.js`](rebuilt-swarm-source/swarm.js): `captureModel()` walks the page; `detectEnv()` chooses the scrolling surface; `View` draws the reconstruction; `Engine` schedules tasks and moves agents; `Sound` synthesizes the audio; `App` coordinates startup, controls, replay, and cleanup.
