# Face Squishy Maker — handoff

Last updated: 2026-10-13

## Current goal
Turn a photo into a squishy face. Flow: auto background removal → **the user fixes the selection by hand** → 3D squishy.

## Status
- Production: https://face-malangi-maker.vercel.app (deployed 2026-10-13, latest)
- Repo: not a git repo yet (the first `/ship` will run `git init` and make the first commit). No remote.
- Everything is in a single file, `index.html`. No build step.

## Structure (`index.html`)
- **First `<script>` (3D squishy):** `window.startMalangi(MAKER)`. The shape comes from the face outline (`MAKER.radii`), the front/back thickness constant `C`, and `FACE_S`.
  - The thickness is set in the "face shape" branch of `roundCube()`: `C = 0.36`, front/back profile `pow(|uz|, 0.7)`.
  - Before 2026-10-13 it was 0.26 / 0.6. It was made thicker after the user said the vertical cross-section was too thin.
- **Second `<script type="module">` (maker):**
  1. `preparePhoto`: opens the photo (HEIC is converted with heic-to only when the browser can't open it), scales it down to at most 1024px, then runs `autoMask`.
  2. `autoMask`: MediaPipe `selfie_multiclass_256x256`. Categories 1 (hair) and 3 (face skin) → `cleanMask` (keeps the largest blob, fills holes). Returns null if it fails, finds less than 1% of the image, or takes over 20 seconds.
  3. `openEditor`:
     - If auto succeeded, the erase brush is the default and the "자동 선택으로" (back to auto selection) button is shown.
     - If it failed, it starts with an empty selection and the lasso tool, and shows "couldn't find a face automatically".
     - Tools: lasso, erase, restore, undo, clear all.
  4. `buildMalangi`: mask → 512px face texture + 180 outline radii + hair color (average of the darker half of the top 20% of the selection).

## Commands that worked
- Local preview: the `face-malangi-maker` entry in `.claude/launch.json` (python http.server, port 8940).
- Deploy: `npx -y vercel@latest deploy --prod --yes --scope magic3ightballs-projects`
  - Without `--scope` it fails with "Not authorized". The `vercel` binary isn't on PATH, so use npx.
  - After deploying, delete `.env.local` (OIDC token) if it was created.
  - Alternatively use the user-level `/ship` skill (commit → deploy → verify).
- Live check: `curl -s https://face-malangi-maker.vercel.app | grep -c mkAuto`

## Verified (built-in browser, local)
- A real face photo composited onto a test background: auto-selection picked hair + face and left out the neck and clothes. Clear all → back to auto selection → make squishy all worked.
- A photo with no person: fallback message shown, switches to the lasso tool.
- After the thickness change, the side looked thicker from an angle. An exact 90° side view was not captured.
- No console errors.

## Not verified / risks
- Real phone photos (whole person, several people), iPhone Safari touch, HEIC rotation.
- The first visit downloads the MediaPipe model, so the first photo can take a few seconds. On a slow connection, after 20 seconds it falls back to manual selection.
- In the hidden built-in browser pane, `Image.decode()` hangs during tests → use `fetch` + `createImageBitmap` to composite the image, then put it into `#mkFile` with a DataTransfer.
- Test images placed in the project folder get deployed along with it, so delete them before deploying.

## Next recommended step
Have the user open it on their phone and try a real photo → take feedback on auto-selection quality and the squishy's thickness.
