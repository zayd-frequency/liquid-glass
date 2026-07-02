# Path to 100 — everything to close the render pipeline to a proven end-to-end result

Current: ~76/100 (v2 spec). **100 = the spec is complete AND one real unattended run proves every
stage on the actual box, landing a verified gorgeous frame + clip + delivery in S3 with a full proof
packet.** A spec is never 100 on paper; the last 24 points are *proof*, not prose.

Three companion docs stay authoritative: `BLACKWELL_8K_MASTER_SPEC` (pipeline), `VOLUME_LOOK_LAW`
(the look), this doc (the closure plan). Node names are the add-on's.

Each item below is: **Gap → Deliverable → Acceptance (the proof that closes it).**

---

## Part 1 — Close the three §10.8 runner glue gaps (fast; unblocks "unattended")

**1.1 Smoke stage must gate, not just render.**
- Gap: §10.8 `smoke` renders a 1080p frame but never checks it, then proceeds to the expensive 8K hero.
- Deliverable: after the smoke render, bake the AgX PNG (§7 step 1) and run the §10.5 `oiiotool` gate
  (`avg>0.001 && stddev>0.005 && not Constant`); `exit 1` if it fails.
- Acceptance: inject a deliberately black smoke → runner aborts at `STAGE=smoke`, never reaches `hero`.
- **Status: implemented** in the master spec §10.8 (`gate_png` + exit 65 at `STAGE=smoke`).
  Acceptance test still to be run on the box.

**1.2 Validate stage must run the real gate.**
- Gap: `validate` comment claims "count + per-frame numeric + NaN," code does count only.
- Deliverable: loop every `frame_*.exr` → `oiiotool --stats` → assert `NanCount==0` and the numeric
  gate on each; `exit 1` on the first bad frame, naming it.
- Acceptance: drop one zero-byte/NaN frame into `out/` → validate exits nonzero naming that frame.
- **Status: implemented** in §10.8 (count → zero-byte → per-EXR NaN → AgX proof bake → per-PNG
  numeric gate, each failure named). Acceptance test still to be run on the box.

**1.3 Derived dead-man ceiling vs the runner ordering.**
- Gap: `deadman` arms `shutdown -h +<CEILING_MIN>` at the top, but `<CEILING_MIN>` is smoke-derived and
  the smoke runs later — circular.
- Deliverable: arm a generous default (e.g. +180) first; after `smoke` measures minutes/frame, compute
  `CEILING = ceil(min_per_frame × frames_on_box × 1.5)`, then `shutdown -c; shutdown -h +$CEILING`.
- Acceptance: run log shows the ceiling re-armed to a derived value after the smoke stage.
- **Status: implemented** in §10.8 (`STAGE=deadman` arms +180 default; new `STAGE=ceiling` re-arms
  with `SMOKE_SEC × SCALE × FRAMES × 1.5`, where `SCALE=16` covers the 1080p→8K pixel ratio).
  Acceptance test still to be run on the box.

*Closes: orchestration + QA + "does a run finish" to full mechanism.*

---

## Part 2 — Build the two missing layers (the biggest points: stages 2 & 3)

### 2.1 Preprocessing layer (raw → clean volume) — currently out of scope
- Gap: the spec starts at a render-ready volume; nothing cleans the source.
- Deliverable: `preprocess.py` (SimpleITK/NumPy, runs in the add-on's bundled Python or a pinned env):
  1. Load with **physical spacing + affine preserved**.
  2. Reorient to a canonical axis convention (record the transform, never silently flip).
  3. Intensity: clamp to the real scalar window, record min/max; optional denoise (recorded params).
  4. Optional resample to isotropic (record factor) — off by default (preserves fidelity).
  5. Optional mask/segment + **keep-largest-component / remove-islands** (record every param).
  6. Write **preview level** (fast) + **full-res level**, and `preprocess_manifest.json`
     (source URI, spacing, affine, orientation transform, intensity window, every op+param, checksums).
- Acceptance: a 2D triptych (axial/coronal/sagittal) overlaying cleaned-on-source shows scale and
  L/R orientation preserved; manifest has an entry for every operation; re-running is deterministic.

### 2.2 Headless data→cache build — the single biggest hole
- Gap: the spec punts cache creation to a manual GUI step (§1.5), so the "unattended farm" isn't
  actually end-to-end.
- Deliverable: `build_cache.py` — run headless in Blender, proving the add-on's lower-level API path
  (these functions were confirmed present in the 2.0 source this session; **verify the exact module
  and argument spellings against the installed add-on source when implementing — the draft's
  spellings varied**):
  ```python
  from oxelnodes.oxel.data import read
  data = read(cleaned_volume_path)                    # → Data (spacing/affine/min-max preserved)
  layers = data.to_layers(kind="scalar", oxel_size=<vox>, smooth=0)
  from oxelnodes.layer import save_layers_to_json
  ids = save_layers_to_json(layers, cache_dir=<project>/cache/layers)   # writes data.vdb + snapshot + histogram + meta
  ```
  Then build the object + Geometry Nodes tree and add `O Layer` pointed at the saved cache id.
- Acceptance (must all pass on ONE box, zero GUI):
  1. `data.vdb` (or `data.0001.vdb…`), `snapshot.npy`, `histogram.png`, and `oxel_layers` meta exist.
  2. A subsequent **headless** render reads the cache and produces a **non-empty** frame.
  3. The cache survives `.blend` reopen on a *fresh* box (path enumeration → 0 missing).
- If the API path can't be made to work headless, the honest fallback stands (author cache locally once,
  farm render-only) — but 100 requires **proving** the headless path or explicitly accepting the manual
  pre-step as permanent. Prove it; don't assume it.

*Closes: preprocessing (stage 2) and cache build (stage 3) — the two weakest, and the precondition for
a genuinely unattended data→frame pipeline.*

---

## Part 3 — Wire the look (stages 6 & 8: from "clean" to "gorgeous")

- Gap: the spec renders clean-but-flat; the look lives in `VOLUME_LOOK_LAW`, not yet wired into code.
- Deliverable: `look.py` that
  1. **Measures** the six numbers off the cache (`histogram.png` + `oxel_layers` min/max + a numpy
     pass): `smin/smax, air_floor, P2/P50/P90/P98, g_med, vox_mm, depth_vox`.
  2. **Derives** every look value by the law's §2 formulas (threshold, ramp stops, base opacity,
     boundary-boost scale, density, emission).
  3. **Builds the O-graph** with those values: `O Layer` (Center on, Resample OFF) → `O Cutout by
     Threshold` (derived) → `O Scalar Layer to Color` (derived ramp, one project hue family) +
     `O Scalar Layer to Factor` (opacity + gradient boundary boost) → `O Realize Structure`
     (Density derived, Surface on).
  4. Applies the **material** (volume scatter + small density-tied emission), **lighting rig**
     (black world, warm key + cool rim, low ambient), **camera** (85 mm hero, thirds, f/4), and
     **compositor** (low Fog-Glow glare + vignette).
  5. Selects the **hero tier** (biased path: volume bounces 16, step 0.15) per the law's override.
- Acceptance: a **contact sheet** (front/side/top + one cutaway) that a human passes against the
  reject list (not plastic / fog / rainbow / gray blob / crushed / blown-out / depthless). This is the
  one stage that needs a human eye + 3–5 look-dev loops — budget for it.

---

## Part 4 — Automate the proof harness (make every claim proven, not asserted)

- **4.1 GPU-actually-used, automated.** Gap: §5 says run `nvidia-smi pmon` in a *second shell* by hand.
  Deliverable: the runner backgrounds a `nvidia-smi pmon` sampler during the render, greps for the
  `blender` process with non-zero `sm%`, writes `gpu_telemetry.txt`; **fail the run if it never appears**
  (that is the true CPU-fallback catch). Acceptance: telemetry shows blender on the GPU for the render's
  duration; a forced CPU render fails this gate.
  **Status: implemented** in §10.8 (`STAGE=hero` sampler + gate; telemetry uploaded to
  `postmortem/` on every exit). Acceptance test still to be run on the box.
- **4.2 Cache-portability proof.** Reopen the staged `.blend` on a fresh box, enumerate linked
  library/image/cache paths, assert 0 missing before render. Acceptance: a deliberately un-shipped cache
  fails this (exit 66) — already partly in §10.8, make it enumerate, not just `ls`.
- **4.3 Everything checksummed + manifested.** `sha256sum` on frames/passes/.blend+cache/videos; a
  `run_manifest.json` listing every artifact (path, size, checksum, stage, versions). Acceptance: the
  manifest reproduces byte-for-byte on a re-run from the same input.
- **4.4 The seven+3 proofs run inside the runner** (v2 has proofs 2–10 in `STAGE=proofs`; confirm proof
  10 renders + `libnvoptix` + driver ≥575 all abort the run on failure before any GPU-hour).

---

## Part 5 — Cost, AMI, reproducibility actually built (stages 10 & 12)

- **5.1 Bake the golden AMI** from the certified box (driver + pinned Blender + add-on + all four
  scripts: preprocess/build_cache/look/render). Pin its AMI ID. This is the **precondition** that makes
  `terminate`-on-shutdown safe for workers. Until it exists, BW-C stays **seed = stop + fail-closed**
  (v2 §10.3a — correct).
- **5.2 Wire the cost guards for real** (not just described): AWS Budgets alarm at 80% of `<CEILING>`,
  an EventBridge one-off `StopInstances`/`TerminateInstances` at T+Nh as the outside-the-box backstop,
  the derived dead-man switch (Part 1.3), the idle watchdog. Acceptance: budget alarm exists
  (`describe-budgets`); EventBridge rule exists; a wedged box is stopped by the backstop in a test.
- **5.3 Reproducibility proof.** A fresh worker from the AMI, given the same input, produces a
  **checksum-identical** frame (seed 0 + persistent data make this achievable). Acceptance: two boxes,
  same frame → same sha256.

---

## Part 6 — Delivery + license (stages 1 & 15)

- **6.1 Delivery chain automated.** EXR → **bake AgX PNG first** → ProRes 4444 master → H.265 share,
  run post-frames on one box, synced to `jobs/<RUN_ID>/delivery/`, checksummed. Acceptance: `master.mov`
  + `share.mp4` in S3, color-correct (not washed out — proves the bake-first fix), counts match.
  (Note: §10.8's validate stage now bakes the AgX proof PNGs for every frame — 6.1's encode consumes
  them directly instead of re-baking.)
- **6.2 Re-add the source-license matrix** the spec dropped: per-source `cleared_for_private_proof` vs
  `cleared_for_public_interactive_publication`, attribution/share-alike flags. Acceptance: no public
  export path runs unless `cleared_for_public = yes`; private proof only needs private clearance.

---

## Part 7 — The one real end-to-end run (this is what actually scores 100)

Everything above is *capability*. 100 is *demonstrated* capability. Execute once, unattended:

```
stage a source volume + preprocess (2.1) → headless cache (2.2) → look-wired graph (3) →
proofs (4.4) → 1080p smoke that GATES (1.1) → derive ceiling (1.3) → 8K hero + clip →
GPU-used telemetry (4.1) → per-frame validate + NaN (1.2) → checksums/manifest (4.3) →
delivery encode (6.1) → upload → self-stop → postmortem/status in S3
```

Acceptance for **100**: a single **proof packet** in which all 16 scorecard stages read `verified`,
each backed by an artifact in S3 (frame, telemetry, manifest, contact sheet, delivery, postmortem) that
another agent can independently re-check — and the hero contact sheet is human-graded **gorgeous**, not
merely non-black.

---

## Scorecard: 76 → 100 (what each part closes)

| Stage | Now | To 100 via |
|---|---:|---|
| 1 Source & licensing | 6 | Part 6.2 (license matrix back) → 9 |
| 2 Preprocessing | 5 | **Part 2.1** → 9 |
| 3 Cache build | 5 | **Part 2.2 (headless, proven)** → 9 |
| 4 Blender/Python setup | 8 | Part 5.1 (AMI pins it) → 10 |
| 5 GPU/OptiX | 9 | Part 4.1 (auto GPU-used proof) → 10 |
| 6 Render settings | 8 | Part 3 (look tier wired) → 10 |
| 7 Output/color | 8 | Part 6.1 (delivery proves color) → 10 |
| 8 Scene/look | 7 | **Part 3 (gorgeous contact sheet)** → 10 |
| 9 Instance | 8 | 5.1 (certified box in AMI) → 9 |
| 10 Driver/AMI/repro | 8 | Part 5.1 + 5.3 → 10 |
| 11 Proofs | 9 | Part 4.4 → 10 |
| 12 Cost/shutdown | 8 | Part 1.3 + 5.2 → 10 |
| 13 Orchestration | 8 | Part 1 (runner gates) → 10 |
| 14 Output QA | 8 | Part 1.2 + 4.1 → 10 |
| 15 Delivery | 8 | Part 6.1 → 10 |
| 16 Does a run finish | 8 | **Part 7 (proven once)** → 10 |

Target: **~155/160 (~97) on the spec, 100 the moment Part 7's proof packet is green.** The last 3
points are inherently "prove it," not "write it."

---

## Execution order (do it in this sequence)

1. **Part 1** (3 runner fixes) — cheap, unblocks unattended. [~1 pass] **← done in spec §10.8; test on box**
2. **Part 2.1 preprocess** then **2.2 headless cache** — the hard, high-value layer; prove headless on
   ONE box before trusting it. [the real engineering]
3. **Part 3 look** — wire the law; then 3–5 look-dev loops to gorgeous (human eye needed).
4. **Part 4 proofs** + **Part 6.1 delivery** — automate the evidence. **← 4.1 done in spec §10.8; test on box**
5. **Part 5.1 golden AMI** + **5.2 guards** + **5.3 repro** — makes it reproducible and cost-safe.
6. **Part 6.2 license** matrix.
7. **Part 7** — run it once, unattended, produce the proof packet. That green packet is 100.

Reality note: Parts 2.2 (headless cache) and 3 (gorgeous look-dev) are where real effort and iteration
live — everything else is mechanical. Budget accordingly; don't expect gorgeous on run one.
