# Stretch

A browser-based tool for selectively stretching a 3D mesh: place two
parallel (optionally tilted) cutting planes, and scale only the band
between them — axially (length), radially (cross-section), or resize
whole selected objects independently. Everything outside the band shifts
to stay attached, but is never distorted.

No install, no build step — it's a single self-contained HTML file. Open
it in a browser, or host it anywhere static files are served.

**[Open the tool](https://vibe-coded-goodies.github.io/stretch-tool/)**

## Features

**Viewing**
- Interactive 3D viewer with custom orbit/pan/zoom controls
- Perspective/orthographic toggle
- Click-and-drag view-cube gizmo for snapping to standard views
- Shift+drag constrains rotation or panning to a single axis

**Cutting & deforming**
- Two draggable, tiltable cutting planes that stay parallel to each other
- **Axial scale** — stretches the band's length
- **Radial scale** — stretches the band's cross-section, with a smooth
  crease-aware taper at the edges instead of a hard step
- **Overall size** — resizes selected objects entirely, independent of
  the band
- Every numeric value (plane positions, tilt angles, scale factors) can be
  typed directly, not just dragged
- Real mesh subdivision at the cut boundaries, so deformation works even
  on very low-poly source meshes — not just wherever the original mesh
  happened to have vertices
- Commit-on-edit workflow: moving a plane, changing tilt, or switching
  axis bakes the current shape in as the new baseline, so you can chain
  multiple stretches
- Multi-step undo

**Multi-part files**
- Click any part in the viewport to select/deselect it — only selected
  objects are affected by scaling
- Each object gets its own radial pivot axis, not one shared axis for the
  whole file
- Real object names pulled from BambuStudio/OrcaSlicer metadata when
  available

**Import / export**
- STL (ASCII + binary), 3MF (including the Production Extension used by
  BambuStudio/OrcaSlicer, where object geometry is split across multiple
  part files inside the archive, and multi-plate layouts), and OBJ
  (including named `o`/`g` groups as selectable objects)
- Exports back to whichever format you imported, with an editable output
  filename
- Z-up-to-Y-up conversion applied automatically on import (STL/3MF/OBJ
  commonly author Z-up; this viewer is Y-up)

## How it works

For a chosen normal vector `n` (shared by both planes) and two offsets
`d1 < d2` along it, every vertex's signed distance `t = dot(vertex, n)`
falls into one of three regions:

- `t <= d1` — untouched
- `t >= d2` — translated (not scaled) by the same amount the band grew,
  so it stays attached without changing shape
- `d1 < t < d2` — scaled from `d1` (axially) and from each object's own
  pivot (radially), with a smoothstep taper near the edges

Deformation is combined with real mesh subdivision: before scaling,
triangles crossing the plane boundaries (and the radial taper zones) are
clipped and re-triangulated, so there are always real vertices for the
transform to act on — not just wherever the source mesh happened to
already have geometry. Subdivision is capped per object to prevent
runaway growth across many repeated edits.

See [`CHANGELOG.md`](./CHANGELOG.md) for the full history — this went
through a lot of real bug fixes against real-world files, and most of the
design decisions above exist because of something specific that broke
first.

## Local development

There's nothing to build. Just open `index.html` in a browser. It loads
Three.js and JSZip from cdnjs at runtime, so an internet connection is
needed the first time (or whenever those aren't cached).

## Known limitations

- 3MF export is minimal — geometry only, no color/material data, no
  production-extension metadata
- Very large selected objects (hundreds of thousands of triangles after
  subdivision) can feel sluggish while dragging, since normals and the
  bounding box recompute on every tick
- The two tilt sliders rotate around fixed world axes in sequence, not a
  true independent spherical (azimuth/elevation) control

## License

MIT — see [`LICENSE`](./LICENSE).
