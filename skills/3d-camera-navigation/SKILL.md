---
name: 3d-camera-navigation
description: "The house camera-navigation scheme for 3D viewport prototypes — middle-button, right-button and trackpad orbit/dolly/truck with pivot-under-cursor resolution, WASD flythrough, perspective/orthographic toggle and axis snap views — plus drop-in Three.js + TypeScript implementations that already handle pole flipping, ortho zoom-to-cursor, and OrbitControls arbitration. Use this whenever a 3D viewport needs camera controls: starting a new Three.js/WebGL/r3f prototype, adding orbit or pan or zoom, supporting trackpad gestures (two-finger drag, pinch to zoom) or telling a trackpad apart from a mouse wheel, wiring a fly/walk camera, adding Top/Front/Right views, or debugging complaints like 'the camera feels wrong', 'it flips upside down when I orbit', 'zoom doesn't go where my cursor is', 'the scroll wheel orbits instead of zooming', or 'orbiting spins around the wrong point'. Also use it when navigation has to share the mouse with selection or transform-gizmo dragging."
---

# 3D viewport camera navigation

This is a settled navigation scheme, not a menu of options. It was designed and
verified for CAD-adjacent 3D tools (architecture, transform manipulators) where
the user spends all day moving the camera and expects it to behave like the
tools they already know. Reproduce it as specified so every prototype feels
like the same application.

Camera feel is deceptively hard: the individual movements are trivial, and
almost all the difficulty is in the pivot, the pole crossing, and the ortho
special cases. Those are already solved in `assets/`. Copy the code rather than
re-deriving it — the failure modes are subtle enough that they usually ship.

## The input map

| Input | Action |
| --- | --- |
| **Middle drag** | Orbit |
| **Ctrl + middle drag** | Dolly (in/out along the view axis) |
| **Shift + middle drag** | Truck (slide across the screen plane) |
| **Right drag** | Orbit — the same three movements under the same modifiers |
| **Wheel** | Dolly, about the point under the cursor |
| Trackpad: **two-finger drag** | Orbit |
| Trackpad: **pinch / spread** | Dolly |
| Trackpad: **Shift + two-finger drag** | Truck |
| **Left button** | *Never navigation.* Reserved for selection and gizmo drags. |
| **Key above Tab** | Enter flythrough / commit and leave it |
| Flythrough: **WASD** | Move; **Q/E** down/up in world Z; **Shift** 5× speed; **mouse** looks |
| Flythrough: **Enter** / **left click** | Keep the new viewpoint |
| Flythrough: **Esc** / **right click** | Put the camera back exactly where it was |
| Toolbar | Perspective/orthographic toggle; Top / Front / Right ortho snaps |

Every camera movement lives on the middle button. That is the whole point of the
scheme: in a tool with manipulators, the left button belongs to the model, and a
camera that occasionally steals it makes precise dragging feel unsafe. Modifiers
select *which* movement, so the user's hand never leaves the middle button.

A two-button mouse has no middle button, so the right button is one: the same
three movements under the same modifiers, handed to the same code. The
unmodified right drag orbits, because orbit is what the unmodified drag is
everywhere else here, and a mouse with one button fewer should not have to learn
a different scheme to get the movement it reaches for most. Pan is not on the
right button; truck is the movement pan would be, and it is on Shift like
everywhere else.

The wheel dollies, about the point under the cursor rather than about the
controller's target. Navigation claims every wheel event over the canvas in the
capture phase, and the OrbitControls wrapper has its own zoom switched off, so
there is no second path to the camera.

The trackpad carries the same three movements, reached by gesture instead of by
modifier — a laptop has no middle button, and the left one still belongs to the
model. Both paths run the same movement and pivot code, so the two devices
navigate a scene identically rather than merely similarly.

Supporting both is harder than it looks, because a trackpad gesture and a mouse
wheel arrive as the same `wheel` event and the wheel is already spoken for by
zoom. Pitfalls 16 and 17 are that problem and its fix; do not reach for a
`userAgent` sniff, which cannot see which device a given event came from.

Use `KeyboardEvent.code`, not `.key`, for the flythrough keys — `code` names
keys by physical position, so the WASD block stays a block on AZERTY (ZQSD) and
Dvorak. The key above Tab is `Backquote`, which is `²` on French layouts and `~`
on US.

## Pivot resolution — the part that makes it feel right

A camera that always orbits the scene origin is the single most common reason 3D
navigation feels wrong. People orbit around *what they are looking at*. Resolve
the pivot once, when a movement begins — on pointer-down for the middle button,
and for a trackpad on the first event after a pause in the wheel stream, since a
gesture has no press to hang it on. Resolve it in this priority order:

1. **Whatever is under the cursor.** Raycast scene content; use the hit point.
2. **Otherwise, the face they were just orbiting.** If the last orbit pivoted on
   a face, intersect the cursor ray with that face's *own* plane, so the cursor
   still names a point on the surface it has slipped off. This is what stops the
   camera from lurching to a different depth the moment the cursor leaves the
   model — which, mid-orbit, is constantly. Orbiting only: a dolly toward a
   point on an extended plane has the ground's problem, in that the step becomes
   however far off that patch of plane happens to be.
3. **Otherwise, the center of the content's bounds.**

Give the step-2 plane an edge. It is infinite, so a cursor out over the sky
meets it a street away and the view turns about a point nobody was looking at —
the same failure the ground has, arriving by a different road. Box the object
the face belongs to at the moment the plane is armed, and believe the plane only
within `faceReach` times that object's bounding radius; 1.5 is the default,
leaving the halo just off an edge in play, which is where a cursor that has just
slipped off one usually is. Past it, fall to step 3.

The step-2 plane is scratch state with deliberately short life: only an orbit
that lands on a face arms it, and a truck drops it, since sliding the view
sideways is leaving the place. It is a memory of the surface you were turning
about, so a drag begun over open ground turns about the face you were working on
or else about the middle of what has been built — never about a point read off a
surface nobody meant to name.

Step 3 is the bounds **center itself**, not a point under the cursor at the
center's depth. The alternative is tempting and was tried: the center is a point
in mid-air that can sit half a screen from the cursor, so a pivot marker travels
there the moment a drag misses, and jumps back as the cursor crosses onto the
model again. But a pivot under an off-axis cursor swings the content out of
frame, and keeping it in frame is the whole reason there is a last resort — it
is the answer for a drag that named nothing, and the scene as a whole is the
only thing such a drag can be about. If you draw a pivot marker, let it travel:
it is reporting honestly that the pivot has left the cursor.

Exclude gizmos, overlays, and helper chrome from the pivot raycast. Orbiting
around the handle you happen to be hovering instead of the model beneath it is
uniquely disorienting.

## Invariants

These are the things that break the feel when they are wrong, in rough order of
how often they get missed:

- **Rotate camera *and* target rigidly about the pivot.** Moving only the camera
  lets the pivot slide off screen. Keeping the target on the view axis is also
  what stops a per-frame `lookAt(target)` from undoing the orbit.
- **Let the view over the pole, and turn the up axis over with it.** Clamping
  short of the pole makes the top of the model a place you can get near and
  never over. Instead, read the camera's own up against the axis it is orbiting:
  when that flips sign the camera has crossed, so flip the up axis to match and
  tell the controller its up changed. Nothing moves when it happens — it only
  means the roll levelling agrees with the view it is levelling, and that a drag
  keeps taking the model the way the cursor goes. See `references/pitfalls.md`.
- **Ortho dolly is zoom plus a lateral correction.** Zoom alone scales about the
  screen center, so the pivot drifts out from under the cursor.
- **A trackpad gesture begins at a gap, not a press.** Re-resolving the pivot on
  every wheel event makes it jump under the cursor for the length of the
  gesture; resolving it once and never again leaves it stale for the rest of the
  session. A pause of ~140 ms is the boundary, and it is also what ends the
  device-detection latch.
- **Trackpad rates are the gesture's own pixels.** A two-finger drag orbits and
  trucks exactly as far as a middle-drag of the same distance, so the two
  devices agree by construction rather than by tuning. Pinch is the one
  exception: its deltas are a zoom quantity rather than pixels, so it needs a
  gain — the only number in the scheme set by judgement, and the only one worth
  re-tuning by feel.
- **Rates are screen-relative.** A full viewport height sweeps 2π of orbit.
  Dolly is exponential in drag distance (e-fold per ~220 px) so it never crawls
  when close or lurches when far. Truck is exact world-units-per-pixel, so
  content tracks the cursor 1:1.
- **Z-up world.** Yaw is about world Z; Q/E move along world Z.
- **Flythrough commits by re-planting the orbit target** ahead of the camera at
  the distance it had on entry, so orbiting still works afterwards. Without
  this, the user flies somewhere, then orbits around a point behind them.

## Using the bundled implementation

`assets/` holds the verified Three.js + TypeScript implementation. Copy the
files into the project's viewport module and wire them up.

| File | What it owns |
| --- | --- |
| `navigation.ts` | Middle-button, right-button, wheel and trackpad orbit/dolly/truck, and pivot resolution |
| `cameras.ts` | `CameraRig`: both cameras, view-preserving projection toggle, ortho axis snaps, screen-space `worldPerPixel` |
| `controls.ts` | Thin OrbitControls wrapper — no mouse button and no wheel; it keeps `target` normalized, handles touch, and eats the context menu |
| `flythrough.ts` | WASD/mouse-look fly camera with commit and revert |

Requires `three` and `three/addons/controls/OrbitControls.js`.

Wiring:

1. Set `Object3D.DEFAULT_UP` to Z-up **before** anything is constructed. Each
   object copies `up` at construction time, so this also means no module may
   construct an `Object3D` at module top level — a scratch `new Vector3()` is
   fine, a top-level mesh or camera silently keeps Y-up.
2. `const rig = new CameraRig(aspect)`.
3. `const controls = new ViewControls(rig, canvas)`.
4. Implement `NavigationHost` (declared at the top of `navigation.ts`) — two
   members: `solidObjects` for the pivot raycast and `contentCenter(out)` for
   the fallback pivot. Geometry only: not the ground plane, which runs to the
   horizon so a cursor a few pixels further out names a point half a scene away,
   and not chrome.
5. `new Navigation(canvas, rig, controls, host)`.
6. `new Flythrough(canvas, rig, controls, viewportEl, delegate)`, where the
   delegate disables gizmo picking, `navigation.enabled` and `controls.enabled`
   while flying, and re-runs `controls.onCameraToggled()` on projection change.
7. Per frame: if the flythrough is active call `flythrough.update(dt)` and
   **skip** `controls.update()` — OrbitControls re-aims at its target every
   update and would undo the steering. Otherwise call `controls.update()`.
8. Call `controls.onCameraToggled()` after any projection change or ortho snap;
   OrbitControls binds spherical state to one camera and must be recreated.

## Porting to something other than Three.js

The scheme is engine-independent; only the code is not. Read
`references/pitfalls.md` first — the pole crossing, the ortho pivot
correction, and the pointer-lock/Escape contract are bugs any implementation
will hit, and they are described there in terms of the maths rather than the
API. Keep the input map and the pivot priority exactly as specified so
prototypes stay interchangeable.

For react-three-fiber, keep these as imperative classes driven from a
`useEffect` and `useFrame`; rewriting the drag maths as React state costs
precision and gains nothing.

## Reference

- `references/pitfalls.md` — the non-obvious maths and the specific bugs this
  implementation already fixes. Read it before modifying the movement maths,
  porting to another engine, or debugging a feel complaint.
