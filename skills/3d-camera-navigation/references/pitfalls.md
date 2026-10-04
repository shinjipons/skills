# Camera navigation pitfalls

Every item here is a bug that was actually hit and fixed in the bundled
implementation. They are written as maths and contracts rather than API calls,
so they carry over to any engine. If a prototype's camera "feels wrong", the
cause is almost always on this list.

## Contents

1. [Orbit flips over at the poles](#1-orbit-flips-over-at-the-poles)
2. [Ortho zoom drifts out from under the cursor](#2-ortho-zoom-drifts-out-from-under-the-cursor)
3. [The orbit library fights your orbit](#3-the-orbit-library-fights-your-orbit)
4. [Swapping projections leaves stale controller state](#4-swapping-projections-leaves-stale-controller-state)
5. [Truck rate differs by projection](#5-truck-rate-differs-by-projection)
6. [The remembered face plane reaches to the horizon](#6-the-remembered-face-plane-reaches-to-the-horizon)
7. [Middle-click opens browser autoscroll](#7-middle-click-opens-browser-autoscroll)
8. [Pointer capture throws on synthetic events](#8-pointer-capture-throws-on-synthetic-events)
9. [Escape out of pointer lock is invisible](#9-escape-out-of-pointer-lock-is-invisible)
10. [Keys stick after alt-tab](#10-keys-stick-after-alt-tab)
11. [Panel fields eat the movement keys](#11-panel-fields-eat-the-movement-keys)
12. [Keyboard layout breaks WASD](#12-keyboard-layout-breaks-wasd)
13. [Flying an orthographic camera does nothing](#13-flying-an-orthographic-camera-does-nothing)
14. [Axis snap views are degenerate or rolled](#14-axis-snap-views-are-degenerate-or-rolled)
15. [Headless testing stalls in a hidden tab](#15-headless-testing-stalls-in-a-hidden-tab)
16. [A trackpad gesture and a mouse wheel are the same event](#16-a-trackpad-gesture-and-a-mouse-wheel-are-the-same-event)
17. [The zoom listener eats the trackpad gesture](#17-the-zoom-listener-eats-the-trackpad-gesture)

---

## 1. Orbit flips over at the poles

**Symptom.** Orbiting up past vertical snaps the view upside down, and the next
pointer event snaps it back. Looks like jitter; is actually two flips.

**Cause.** Crossing a pole leaves the camera upside down with respect to the
axis it is orbiting, and two things read that axis on the next frame. The
controller levels the roll against it, which stands the view back up — a 180°
snap that bounces you off the pole you had just crossed. And yaw turns about it,
so dragging sideways suddenly sends the model the other way.

**Fix.** Follow the camera over: after the pitch, compare the camera's own up
against the axis being orbited, and when that dot goes negative, negate the axis
and tell the controller its up changed.

```
extract camera_up from the camera's world matrix
if (camera_up · orbit_axis < 0) {
    orbit_axis = −orbit_axis
    controls.onUpChanged()               // rebuild whatever cached it
}
```

Read it off the camera's own up rather than off the polar angle. Halfway over,
that vector is square to the axis and its sign *is* what the crossing is; an
angle would have to be compared against a pole it has already passed, which is
the ambiguity that made the older clamp necessary.

Nothing moves when the flip happens. It says nothing about where the camera is
or how it is turned — it only means the levelling now agrees with the view it is
levelling, and that a drag keeps taking the model the way the cursor goes.

**Why not clamp instead.** Stopping a thousandth of a radian short of the pole
is the obvious fix and was the earlier one. It makes the top of the model a
place you can get near and never over, and getting over it means letting go,
yawing half a turn, and coming at it from the other side. The flythrough camera
is the exception that still clamps, with a larger ε (~0.02 rad, about 1.1°):
stopping short of the pole is what keeps a mouse-look camera from tipping into a
roll.

## 2. Ortho zoom drifts out from under the cursor

**Symptom.** In orthographic, dollying toward a point slowly slides it away from
the cursor. Perspective is fine.

**Cause.** A perspective dolly moves the camera, so scaling camera and target
about the pivot keeps the pivot fixed on screen for free. An orthographic camera
does not dolly — it changes `zoom`, which scales about the *screen center*. Any
pivot that is not dead center drifts.

**Fix.** Orthographic projection is linear, so the drift can be measured and
undone exactly:

1. Project the pivot to normalized device coordinates.
2. Apply the zoom change.
3. Project the pivot again.
4. Convert the NDC delta to world space using the frustum half-extents divided
   by the new zoom, along the camera's right and up basis vectors.
5. Translate camera **and** target by that vector.

No iteration, no approximation. The same idea fixes wheel zoom-to-cursor.

## 3. The orbit library fights your orbit

**Symptom.** Orbit works during the drag and snaps back on the next frame. Or
flythrough steering visibly fights an invisible force.

**Cause.** Orbit controllers re-aim the camera at their target on every update.
Two distinct consequences:

- Rotating the camera about an arbitrary pivot without also rotating the target
  gets undone the next frame.
- Any camera you drive yourself (flythrough) gets re-aimed every frame.

**Fix.** Rotate camera and target rigidly about the pivot, which keeps the
target on the view axis and makes the per-frame `lookAt` a no-op. While a
self-driven camera is active, do not call the controller's update at all.

**Also:** to free the left and middle buttons, omit them from the button map
entirely. A missing entry reads as "no action"; assigning them a dummy action
does not work.

## 4. Swapping projections leaves stale controller state

**Symptom.** After toggling perspective/orthographic or hitting a snap view, pan
and zoom behave as if the camera were still the old one.

**Cause.** Orbit controllers cache spherical coordinates bound to one camera
instance.

**Fix.** Dispose and recreate the controller on every projection change,
carrying over the target position and the enabled flag. Cheap, and the only
reliable option.

When toggling, preserve framing rather than position: going to ortho, set
`zoom = base_half_height / (tan(fov/2) · distance_to_target)`; coming back,
derive the distance that reproduces the current ortho half-height. Round trips
then land where they started.

## 5. Truck rate differs by projection

**Symptom.** Panning tracks the cursor perfectly in one projection and drifts in
the other.

**Cause.** World units per pixel is computed differently:

```
perspective:   2 · distance_to_target · tan(fov/2) / viewport_height
orthographic:  (frustum_top − frustum_bottom) / (zoom · viewport_height)
```

**Fix.** Branch on projection. In perspective the same rate applies to both
axes; in orthographic compute x and y separately, since the frustum is not
square.

## 6. The remembered face plane reaches to the horizon

**Symptom.** A drag begun over empty sky orbits about a point far off in space,
sometimes behind the model, sometimes a street away. It only happens after
orbiting a face, and the further the cursor is from that face the worse it is.

**Cause.** Step 2 of pivot resolution remembers the plane of the face the last
orbit turned about, and a plane is infinite. Once the cursor is off the object,
the ray still meets that plane — just at a point with no relation to anything
anyone was looking at. Near the plane's horizon the point runs away to infinity.

**Fix.** Give the memory an edge. Box the object the face belongs to at the
moment the plane is armed, take its bounding sphere, and accept the plane's
point only while it lies within `faceReach` times that radius of the sphere's
center. Past that, fall through to the content-bounds fallback.

```
arming (an orbit that lands on a face):
    plane  = plane of the hit face, in world space
    extent = bounding sphere of the hit object

using it (a later orbit that hits nothing):
    p = ray ∩ plane
    accept p only if |p − extent.center| ≤ faceReach · extent.radius
```

The object's bounds rather than the face's, because a face is remembered as a
place on a *thing*, and the thing is what the cursor is either still near or has
left. `faceReach` a little over 1 — 1.5 works — so the halo just off an edge
still counts, which is where a cursor that has slipped off one usually is.

An earlier version of this scheme instead pivoted on the *centroid* of the
coplanar run of triangles and built a camera-facing plane through it. That
solved a different problem (a quad face is two triangles, so the hit triangle's
centroid is off-center) but it answers the wrong question: the memory is of a
surface, not of a depth, and the cursor should still be able to name a point on
that surface after it has slipped off the edge.

## 7. Middle-click opens browser autoscroll

**Symptom.** On Chrome/Windows, middle-dragging opens the autoscroll cursor
instead of orbiting.

**Cause.** Autoscroll is triggered from `mousedown`. Calling `preventDefault` on
`pointerdown` is too late.

**Fix.** Add a separate `mousedown` listener that calls `preventDefault()` when
`button === 1`, alongside the pointer handlers.

## 8. Pointer capture throws on synthetic events

**Symptom.** Driving the viewport from a test or automation script throws
`NotFoundError` on `setPointerCapture`.

**Cause.** Synthetic pointer events have no capturable pointer.

**Fix.** Wrap capture and release in `try`/`catch` and continue. Capture is an
enhancement — dragging past the canvas edge — not a requirement.

## 9. Escape out of pointer lock is invisible

**Symptom.** Pressing Escape to cancel a flythrough exits pointer lock but
leaves the mode active, with the camera stranded.

**Cause.** Browsers swallow the Escape keydown that releases pointer lock. The
handler never fires.

**Fix.** Treat *any* unexpected loss of pointer lock as cancel — it is the
recoverable outcome, and Escape is the overwhelmingly likely reason. Distinguish
your own releases with a flag set immediately before calling exit, so committing
does not get misread as cancelling.

Also do not treat a refused lock request as fatal. Mouse-look still works
unlocked; it just stops at the edge of the screen.

## 10. Keys stick after alt-tab

**Symptom.** Alt-tab away mid-flight, come back, and the camera drifts forever.

**Cause.** The keyup arrives at the other window, so the held-keys set never
clears.

**Fix.** Clear the held set on window `blur`.

## 11. Panel fields eat the movement keys

**Symptom.** Entering flythrough right after typing in a settings panel types
`wasd` into the field instead of moving.

**Cause.** Focus is still in the input.

**Fix.** Blur the active element when entering the mode, and guard the toggle
key itself with a check for `INPUT`, `TEXTAREA`, or `isContentEditable` targets
so it cannot be triggered mid-typing.

## 12. Keyboard layout breaks WASD

**Symptom.** WASD does nothing on a French or Dvorak keyboard.

**Cause.** Reading `KeyboardEvent.key`, which reports the *character*.

**Fix.** Read `KeyboardEvent.code`, which names keys by physical US position —
so `KeyW` is one row above and one column right of Tab whatever it is labelled,
and the WASD block stays a block on AZERTY (ZQSD) and Dvorak (`,AOE`). Same for
the toggle key: `Backquote` is always the key above Tab.

For on-screen hints, ask the browser what those physical keys are labelled via
`navigator.keyboard.getLayoutMap()` (Chromium only) and fall back to US labels
elsewhere. Render the fallback immediately and re-render if the map resolves —
the lookup is async and the HUD should never wait on it.

## 13. Flying an orthographic camera does nothing

**Symptom.** Entering flythrough while in an ortho view leaves W and S inert.

**Cause.** Moving an orthographic camera along its view axis changes nothing —
there is no perspective divide, so the image is identical.

**Fix.** Silently take over with the perspective camera on entry, preserving
framing, and restore the original projection if the user cancels. Record the
projection on entry so cancel is exact.

## 14. Axis snap views are degenerate or rolled

**Symptom.** The Top view is blank, spins randomly, or arrives rotated 45°.

**Cause.** In a Z-up world, looking straight down Z with a Z up-vector makes
`lookAt` degenerate — the up vector is parallel to the view direction.

**Fix.** Nudge the top view direction off-axis by ~1e-4, **along −Y only**.
That is what puts world +Y up and +X right on screen. Nudging X as well rolls
the view 45°, which looks like a bug and is one.

Keep the "is this a standard view" test loose enough to accommodate the nudge:
treat a view as axis-aligned when the dominant direction component is ≥ 0.999.

## 15. Headless testing stalls in a hidden tab

**Symptom.** Automated camera checks hang when the tab is not visible.

**Cause.** `requestAnimationFrame` is throttled or paused in hidden tabs, so the
render loop never advances.

**Fix.** Expose a `renderFrame(dt)` method that performs exactly one
update-and-render with no scheduling, and drive it directly from tests via a
window hook. Keep the rAF loop as a thin wrapper that computes `dt` and calls
the same method.

## 16. A trackpad gesture and a mouse wheel are the same event

**Symptom.** Adding two-finger-drag orbit makes the mouse wheel orbit too, so
zoom is gone. Mapping it the other way round leaves the trackpad only able to
zoom.

**Cause.** The browser reports a two-finger drag as a `wheel` event, identical
in kind to a wheel notch. There is no device field, and a `userAgent` sniff
cannot help: the same machine has both, often at once.

**Fix.** Read the shape of the numbers, not the device. A wheel notch is coarse,
whole, and vertical-only; a trackpad reports fine deltas, frequently fractional,
usually with a non-zero horizontal component. In order:

- `deltaMode !== 0` (lines or pages) → wheel.
- `deltaX !== 0` → trackpad; a wheel has no horizontal axis.
- `deltaY` not an integer → trackpad; sub-pixel deltas are gesture deltas.
- otherwise `|deltaY|` under ~40 → trackpad, over it → wheel.

Then **latch the verdict for the length of the gesture**, using the same idle
gap that starts a gesture. This is not an optimisation: a fast two-finger flick
does reach notch-sized whole numbers partway through, and without the latch the
back half of a flick is misread as a wheel and the orbit stutters into a zoom.
The opening events of a flick are small, so the gesture is already identified by
the time the big numbers arrive.

A pinch needs no heuristic at all — every browser reports it as `ctrlKey` +
`wheel`. Test that arm first. It also catches a real Ctrl+wheel from a mouse,
which is correct: Ctrl already means dolly on the middle button.

**Verify it** by synthesising events rather than by hand — real hardware cannot
produce a clean test matrix. Feed the classifier a notch (`deltaY: 120`), an
accelerated notch (`360`), a line-mode notch, a fractional delta, a diagonal
one, and a ramping flick (`1.5, 3.25, 8, 21, 54, 96, 120, …`) and assert the
whole flick classifies as trackpad.

## 17. The zoom listener eats the trackpad gesture

**Symptom.** A two-finger drag orbits *and* zooms at the same time, or the
gesture is swallowed entirely by the orbit library.

**Cause.** The orbit library's wheel-zoom listener is on the canvas, and so is
yours. `stopPropagation` does not help between two listeners on the same
element — they both run, in registration order, and the library's was registered
first.

**Fix.** Listen on the canvas's **parent** with `capture: true`. The capture
phase runs over ancestors before any listener on the target itself, so the
gesture handler gets first refusal: it calls `preventDefault` and
`stopPropagation` on the events it claims, and returns without touching the
ones it does not, which then reach the zoom untouched.

Guard on `event.target === canvas` — a capture listener on the parent also sees
wheel events over any toolbar or overlay inside it.

`passive: false` is required or `preventDefault` is ignored, and without it the
page itself scrolls, and a pinch zooms the whole document.
