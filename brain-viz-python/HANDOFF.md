# Session Handoff — Brain Gene Expression Visualizer

## What this is

An interactive 3D brain visualization prototype for the NYU Neuroinformatics Lab. Maps gene expression data from GTEx (measured tissue samples) and AHBA (Allen Human Brain Atlas) onto a custom 156-parcel brain atlas, rendered in the browser via a Python server.

**Run it:**
```bash
cd brain-viz-python
python app.py
# Open http://localhost:1234
```

---

## Stack (important — this is not React/Dash/Streamlit)

| Layer | Library | Role |
|---|---|---|
| 3D renderer | PyVista (VTK) | Brain meshes, colourmaps, camera |
| Web bridge | trame | Python websocket → browser WebGL canvas |
| UI components | Vuetify 3 / Vue 3 | Toolbar, dropdowns, buttons — all defined in Python |
| Data | data_loader.py | NPZ loading, parcel alignment, value dicts |

No separate JS build. Trame ships Vue + Vuetify as bundled assets. Everything is in `app.py`.

---

## File map

```
brain-viz-python/
├── app.py              # Everything: rendering, UI, state, callbacks
├── data_loader.py      # Data access only (no rendering)
├── build_atlas.py      # One-time setup — do not re-run unless rebuilding atlas
├── atlas_cache/
│   ├── subcortical/    # 54 .vtk meshes — BUILT, ready to use
│   └── cortical/       # EMPTY — needs wb_command (see below)
└── HANDOFF.md          # This file

subject_bundle/         # (sibling directory, not inside brain-viz-python)
├── atlas_info/
│   ├── atlas-4S156Parcels_dseg_reformatted.csv   # 156-parcel metadata + MNI centroids
│   ├── atlas-4S156Parcels_space-MNI152NLin6Asym_dseg.nii.gz  # original 4D NIfTI
│   └── atlas-4S156Parcels_3d.nii.gz              # squeezed 3D copy (use this one)
├── cache/{naive,dlam,plam}/GTEX-1117F.npz        # expression data per subject/model
├── gene_lists/ahba_100hvg.txt                     # 100-gene dropdown preset
└── parcel_order.json                              # canonical 150-parcel row order

subject_bundle_GTEX-13OW8/  # same structure, second subject
└── cache/{naive,dlam,plam}/GTEX-13OW8.npz
```

---

## Atlas status

**Subcortical** (`atlas_cache/subcortical/`): **Built.** 54 VTK meshes extracted via marching cubes from the NIfTI. LH-MN and RH-HN were too small to form a mesh and are correctly dropped — neither carries GTEx data.

**Cortical** (`atlas_cache/cortical/`): **Not built.** Needs `wb_command` (Connectome Workbench). The installer `workbench-macub-v2.1.0.dmg` is in `~/Downloads` but has not been installed. Until the cortical atlas is built, the cortex renders as a solid grey shell (no parcel-level colour). To build:
```bash
# 1. Install workbench from ~/Downloads/workbench-macub-v2.1.0.dmg
# 2. Add to PATH: export PATH="/Applications/workbench/bin_macosx64:$PATH"
# 3. python build_atlas.py
```

---

## Data schema (NPZ files)

Each `.npz` has these arrays, all with 150 rows aligned to `parcel_order.json` (alphabetical AHBA labels):

| Key | Shape | Used in | Model-dependent? |
|---|---|---|---|
| `gene_names` | (13618,) | — | No |
| `gtex_mask` | (150,) | V2 mask | No |
| `loro_truth_subject_raw` | (150, G) | **V1 GTEx Values** | No |
| `loro_eval_mask` | (150,) | V1 mask | No |
| `loro_fused_subject_h` | (150, G) | **V2 Reconstruction** | **Yes** |
| `loro_truth_subject_h` | (150, G) | (unused in current UI) | No |
| `fullfit_subject_h` | (150, G) | **V3 Whole-brain Fit** | **Yes** |
| `imputed_mask` | (150,) | V3 mask (union with gtex_mask) | No |

**Key gotcha:** 156 atlas parcels, 150 NPZ rows. The 6 missing (no AHBA coverage): `Cerebellar_Region9`, `LH-MN`, `RH-EXA`, `RH-STH`, `RH-VeP`, `RH_SomMot_2`. Always align via `parcel_order.json`, never by atlas CSV index order.

---

## Hemisphere handling

The upstream pipeline forces all MNI x-coordinates negative (left hemisphere) before model fitting. As a result, some GTEx samples end up matched to an RH-labelled AHBA parcel (the mirrored RH copy was nearest), even though the model treats that parcel as LH.

**Display strategy (implemented in `data_loader.py`):**
- **V1 / V2** — `remap_rh_to_lh_nearest()`: for each RH-labelled parcel that carries data, flip its centroid x → −x, find the nearest same-structure LH parcel, and display the value there. Everything renders in the left hemisphere.
- **V3** — `mirror_lh_to_rh()`: discard the raw RH predictions from `fullfit_subject_h` (which are unreliable because the model only ever saw LH-space data) and replace each RH parcel's displayed value with its nearest LH counterpart's value. This gives a symmetric bilateral view backed by the actual LH predictions. Raw RH predictions vary by model: `naive` produces spatially inconsistent RH values; `dlam` collapses most RH parcels to a near-constant — both are artefacts of predicting at locations never seen during training.

`data_loader.build_rh_to_lh_map(atlas_aligned)` pre-computes the RH→LH nearest-neighbour mapping once at startup (69 entries for this atlas).

---

## Current UI

Three side-by-side viewports share a single camera — rotating on any panel rotates all three simultaneously. Mouse drag to rotate, scroll wheel to zoom.

| Panel | NPZ array | Mask | Notes |
|---|---|---|---|
| **V1 GTEx Values** | `loro_truth_subject_raw` | `loro_eval_mask` | Sparse (~9 parcels); LH only after remap |
| **V2 Reconstruction** | `loro_fused_subject_h` | `gtex_mask` | LORO out-of-sample predictions; LH only after remap |
| **V3 Whole-brain Fit** | `fullfit_subject_h` | `imputed_mask` ∪ `gtex_mask` | All 150 parcels; bilateral, no remap |

| Control | State key | Effect |
|---|---|---|
| Gene dropdown | `state.gene` | Re-slices the in-memory array — no I/O |
| Subject dropdown | `state.subject` | Reloads the NPZ for the selected subject |
| Model dropdown | `state.model` | Reloads the NPZ (only changes V2 and V3 output) |
| Full/Half Brain toggle | `state.half_brain` | `'full'` = no clip; `'half'` = sagittal clip at x=0, camera moves to side view |
| Cortex slider | `state.cortex_alpha` | Cortex surface opacity (0–1); lower to expose subcortical regions |
| Rotate buttons ← → ↑ ↓ | `state.rot_left/right/up/down` | Counter-increment pattern fires one discrete rotation step per click |
| Step size selector | `state.rot_step` | Degrees per rotation button click (5/10/15/30/45) |

The V1 panel also shows a bottom legend strip: one coloured swatch per active GTEx parcel, sorted by expression value, matching the panel's colourmap.

---

## Coordinate system

The fsLR32k surfaces and MNI subcortical meshes use RAS orientation:
- **Left hemisphere: x < 0**
- **Right hemisphere: x > 0**

**Half-brain clip** (sagittal cut at x = 0, keeping left):
```python
mapper.AddClippingPlane(_clip_plane)   # _clip_plane normal = (-1, 0, 0)
```
The half-brain view also repositions the camera to a lateral side view (`_CAM_HALF_BASE`).

---

## Known trame/PyVista quirks — read before debugging

**1. Always call `plotter.render()` before `ctrl.view_update()`.**
When actors are removed and re-added (every `redraw()` call), PyVista's internal render state must be committed before trame flushes the frame to the client. Without `plotter.render()`, the client receives the previous scene. This was the root cause of the half-brain clip appearing to do nothing.

**2. `VBtnToggle` values must be strings, not booleans.**
Vue/trame treats Python `False` as falsy/unset; the `@state.change` callback never fires reliably. Use `'full'`/`'half'` (or any non-empty strings) and compare with `==` in Python.

**3. `VBtnToggle` children must use context manager syntax.**
```python
# WRONG — causes double rendering (text nodes leak as siblings)
v3.VBtnToggle(..., children=[v3.VBtn(text='X', ...)])

# CORRECT
with v3.VBtnToggle(...):
    v3.VBtn('X', value='x', ...)
```

**4. PyVista background colour: use `'white'`, not `'#ffffff'`.**
Hex strings are silently ignored in some PyVista/trame version combinations. Set it three ways:
```python
pv.global_theme.background = 'white'
plotter.set_background('white')
plotter.background_color = 'white'
```
Also set `background:white` on both the `VContainer` and `plotter_ui` HTML elements.

**5. Pylance import warnings are false positives.**
The warnings for numpy/pyvista/trame/yabplot in the IDE are because Pylance points at a different Python interpreter than conda base. The app runs correctly.

**6. Programmatic camera changes via `plotter.camera_position =` don't survive trame's `view_update()` reliably.**
In trame's server-side rendering mode, `view_update()` serialises the VTK render state from the widget pipeline, which can restore the previous camera position on top of any programmatic change you just made. Mouse interactions work because VTK's own interactor writes directly to the VTK camera object before serialisation. The fix is to use the VTK camera API directly on the shared camera object:
```python
_shared_cam.SetPosition(*pos)
_shared_cam.SetFocalPoint(*foc)
_shared_cam.SetViewUp(*up)
for renderer in plotter.renderers:
    renderer.ResetCameraClippingRange()
```
This is why a Zoom slider was removed: mouse scroll wheel zoom already works via the interactor, and a programmatic slider could not reliably override it. Transparency control (cortex opacity) is easier because it goes through actor properties, not camera state.

---

## Open issues

1. **Cortical atlas** — Cortex is grey until `wb_command` is installed and `build_atlas.py` is run. Low-hanging fruit once Workbench is installed.

2. **Model selector UX** — Model dropdown affects only V2 and V3. Consider greying it out or hiding it when V1 is the focus.

---

## Footnotes — deferred for later

### V2 vs V3 value discrepancy at GTEx parcels

When comparing V2 (Reconstruction) and V3 (Whole-brain Fit) at parcels where the subject donated GTEx tissue, the values are not identical even though both use model predictions:

- **V2 uses `loro_fused_subject_h`**: LORO (Leave-One-Region-Out) cross-validation. When predicting a GTEx parcel, that parcel's measured data was held out during training. This is a true out-of-sample prediction.
- **V3 uses `fullfit_subject_h`**: Full model fit with all available data included. The GTEx parcels' measured values were part of training, so predictions at those parcels are in-sample fits — pulled toward the actual measurement.

For **imputed parcels** (in `imputed_mask`, no GTEx tissue): `loro_fused == fullfit` exactly, because LORO never held these out (nothing to hold out).

Verified for `GTEX-13OW8 / DPM1`:

| Parcel | V2 `loro_fused` | V3 `fullfit` | Δ |
|---|---|---|---|
| Cerebellar_Region1 | 5.384 | 5.087 | 0.30 |
| Cerebellar_Region5 | 5.161 | 5.151 | 0.01 |

The visual difference is most obvious in the cerebellum because: (a) V3 shows all 9 cerebellar parcels while V2 only shows the 2 with GTEx tissue, and (b) `Cerebellar_Region1` has a notable value gap between the two arrays.

**Possible follow-up:** Decide whether to align V2 and V3 to a shared colour scale for direct comparison, or keep per-panel auto-scaling. Currently each panel auto-scales independently.
