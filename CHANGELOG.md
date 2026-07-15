# Changelog

## v1.0 — Editable numeric controls, filenames, and UI polish
- Every slider (plane positions, tilt angles, axial/radial/overall scale)
  now has a paired number input you can type an exact value into, clamped
  to whatever range the slider currently allows — including the plane
  sliders, whose range shifts dynamically with the object's bounds.
- Inputs never get overwritten mid-keystroke: the constant live-refresh
  loop skips whichever field currently has focus.
- Export filename is editable, defaults to `<uploaded-name>_stretched`,
  strips a typed extension so it can't double up, and sanitizes invalid
  filename characters.
- Renamed "Upload" to "Load" in the UI.

## v0.9 — Real mesh subdivision, and the bugs that came with it
- The core fix: deformation only ever moved *existing* vertices. A
  low-poly mesh where a single huge triangle spans an entire cutting band
  (a plain cylindrical wall with no intermediate edge loops) could have
  zero vertices in the band even while sitting squarely inside the
  object's solid volume — so scaling did nothing, even though the math
  was correct. Fixed with real plane-clipping subdivision (Sutherland-
  Hodgman triangle clipping, cascaded across the plane boundaries and the
  radial taper zones), cached per band position so it's a no-op except on
  the first tick after something actually changes.
- Found via that fix, in rough chronological order of discovery:
  - **Uncontrolled growth**: repeatedly moving a plane re-subdivided the
    already-subdivided mesh each time, compounding triangle count until
    the browser ran out of memory after a handful of drags. Capped
    subdivision per object once it's already fine enough.
  - **Indexed geometry silently corrupted**: the built-in default
    cylinder is the one shape in the whole tool that's indexed (shared
    vertices via an index buffer) — every import is already non-indexed.
    Subdivision assumed a flat triangle-soup layout and fed it garbage.
    Fixed by converting to non-indexed once, at load time, for every
    shape.
  - **Stale color-buffer crash**: `updateObjectColors()` reused the
    existing color attribute in place assuming it was always the right
    size, which broke the moment subdivision changed the vertex count.
  - **Mesh went invisible with no console error**: subdivision replaced
    the `position` attribute with a new, larger array but left `normal`/
    `uv` at the old vertex count — a WebGL-level attribute mismatch,
    which just fails to draw silently rather than throwing a JS error.
  - Added NaN/Infinity guard checks at every stage data could go bad
    (file load, before subdivision, after subdivision, before the
    transform), replacing a blocking `alert()` with a non-blocking
    on-screen banner so a numeric error can never freeze page load.
- Radial scale's edges get a smoothstep taper into a flat plateau instead
  of an instant jump from 1× to the target scale, which read as a visible
  hard step/flange on smooth round parts.
- Restoring smooth shading after the indexed→non-indexed conversion
  needed more than the default `computeVertexNormals()` (which only
  produces flat per-triangle normals on non-indexed geometry, with no
  shared-vertex concept left to average across). Replaced with a
  crease-aware smoothing pass — blends normals across triangles sharing a
  vertex position, but only within a 60° threshold, so curved surfaces
  stay smooth while genuine hard edges (like a cylinder's cap-to-wall
  seam) don't get incorrectly softened. Also scoped grouping to stay
  within each object, since two unrelated parts touching at a coincident,
  near-coplanar point would otherwise bleed shading across the boundary.

## v0.8 — Radial and whole-object scale
- New **radial scale** slider: stretches the band's cross-section
  (independent of axial length), pivoting around each selected object's
  own center rather than a single shared axis — verified a multi-part
  file's radial scale doesn't shift every part toward one shared point.
- New **overall size** slider: resizes selected objects entirely,
  independent of the band, respecting the same object-selection rules as
  axial/radial (initially implemented as a scene-level transform, which
  didn't respect per-object selection at all; reworked to bake per-vertex
  like the other two).
- Object center is computed as a true vertex centroid (not a bounding-box
  midpoint), on request, so it's pulled toward wherever a shape actually
  has more geometry.
- A visible line through each selected object's own pivot axis, useful
  for telling whether the current band actually reaches a given object in
  a multi-part file spread across a wide area.

## v0.7 — Multi-object selection
- Click any part directly in the viewport to select/deselect it; only
  selected objects are affected by any scaling operation.
- Deselected objects dim in the viewport so selection state is visible at
  a glance.
- Selection changes now commit (bake) whatever deformation is currently
  showing before changing what's selected — otherwise a scale factor
  applied to one object would silently start applying to whatever got
  selected next.
- Real object names pulled from `Metadata/model_settings.config` when
  present (BambuStudio/OrcaSlicer convention), falling back to a generic
  label.
- Multi-step undo, including for selection changes.

## v0.6 — View cube, orthographic toggle, OBJ, Z-up handling
- Click-and-drag view-cube gizmo (click a face to snap to a standard
  view, drag to orbit) with its own always-on-top rendering so it's never
  occluded or z-fights with the model.
- Perspective/orthographic toggle, with the ortho camera kept
  continuously in sync with the perspective one.
- OBJ import/export, including named `o`/`g` groups as selectable
  objects, with a working plane-clipping-mesh-subdivision-safe exporter.
- Automatic Z-up → Y-up conversion on import (a proper rotation, not a
  naive axis swap, which would have mirrored the mesh and flipped its
  normals inside-out).
- Shift+drag constrains orbit/pan to a single screen axis.

## v0.5 — Tiltable planes
- Planes are no longer locked to X/Y/Z — two angle sliders rotate the shared
  plane normal away from the base axis (±85° each), labeled dynamically
  based on which axis is selected.
- Both planes always stay parallel (they share one normal), matching the
  original "cut with two parallel planes" spec.
- Core stretch math generalized from "offset along one coordinate axis" to
  "signed distance along an arbitrary unit normal" — verified to reduce
  exactly to the old axis-aligned behavior at zero tilt, and confirmed
  numerically that vertices only ever move along the normal (no
  perpendicular drift) at nonzero tilt.

## v0.4 — Commit-on-edit workflow
- Originally, moving a plane after stretching reset the shape back to the
  original geometry — surprising if you wanted to chain edits.
- Now: if you move a plane or change the tilt while a stretch is applied,
  the currently-displayed (already-stretched) shape is baked in as the new
  base geometry, and the stretch factor resets to 1×. The plane you aren't
  dragging snaps to its true current position rather than a stale
  pre-stretch value, so nothing visually jumps.
- "Reset to original" still fully discards all of this back to the
  pristine loaded/default geometry.

## v0.3 — 3MF import, including split-file objects
- Added a 3MF parser (unzip + walk `<object>`/`<components>`/`<build>`,
  resolving nested transforms).
- Discovered via a real user-uploaded file that BambuStudio/OrcaSlicer
  exports commonly use the 3MF *Production Extension*, splitting each
  object into its own part file under `3D/Objects/*.model` and referencing
  it via a `p:path` attribute instead of inlining the mesh. The parser now
  follows those cross-file references (with caching), verified against a
  real 864,010-triangle export.

## v0.2 — Bug fixes from real usage
- The upper cutting plane originally stayed at its pre-stretch position
  instead of tracking the deformed boundary — fixed so it visually follows
  the band as it scales.
- Verified the core per-vertex stretch math directly against real
  Three.js `CylinderGeometry` output (not just read through) after a
  report that stretching looked wrong, to rule out a logic bug before
  looking elsewhere.

## v0.1 — Initial tool
- 3D viewer (hand-rolled orbit controls, no external controls dependency)
- Two draggable cutting planes with live 3D visualization
- Stretch factor applied only to the band between the planes; outside
  regions translate without distorting
- STL import (ASCII + binary) and export
