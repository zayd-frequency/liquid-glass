# Volume Look Law — deterministic settings from measured data (the answer, every time)

Purpose: never guess look-dev again. You **measure** the volume, plug into **rules**, and every setting
resolves. The only per-shot inputs are numbers you read off the cache; everything else is fixed. This
layers on top of the technical render spec (8K / OptiX / OIDN / AgX / encode) — it governs the *look*,
which the technical spec doesn't.

Node names are the add-on's (`O Layer`, `O Cutout by Threshold`, `O Scalar Layer to Color/Factor`,
`O Realize Structure`).

**[ADDED — how this composes with `BLACKWELL_8K_MASTER_SPEC.md`]** This law is the look authority;
the master spec is the pipeline authority. Where the two overlap they agree, except two deliberate
overrides stated in place below (§3): the volume integrator mode and the hero lens. Measurement (§1)
is part of the **local authoring pre-step** (master spec §1.5) — it happens where the cache is
built, never on the farm.

---

## 1. MEASURE (read these off every dataset's cache — one numpy pass on the VDB + histogram.png)

| Symbol | What | How |
| --- | --- | --- |
| `smin,smax` | scalar min/max | from `oxel_layers` meta (min/max) |
| `air_floor` | valley between the air peak and first tissue peak | first histogram trough / Otsu on the low tail |
| `P2,P50,P75,P90,P98` | percentiles of voxels **above** `air_floor` | numpy `percentile` on non-air voxels |
| `g_med` | median gradient magnitude `|∇scalar|` over non-air | `np.gradient` → magnitude → median |
| `vox_mm` | mm per voxel | oxel size from meta |
| `depth_vox` | volume depth in voxels (view axis) | axis length, or `voxelcount**(1/3)` |

That's the whole per-shot input. Six numbers. Everything below is derived or fixed.

**[ADDED — ship the numbers with the job]** Write the six measurements plus every §2-derived value
to `jobs/<RUN_ID>/in/look.json` next to the `.blend`+cache. The farm then renders a look that is
fully determined by committed inputs, the §10.8 postmortem can echo *which* look inputs produced a
bad frame, and re-rendering a dataset months later reproduces the identical look from the file
instead of memory.

## 2. DERIVED (formulas — plug the measured numbers in)

| Setting | Rule | Why |
| --- | --- | --- |
| Air cutoff (fully transparent below) | `= air_floor` | kills background haze at the real air/tissue edge, not a guess |
| `O Cutout by Threshold` | `= P2` (raise toward `P50` to show only the dense core) | shows everything above the noise floor by default |
| Ramp stop **positions** (`O Scalar Layer to Color`) | `[air_floor, P50, P90, P98]` | stops track the data's own distribution → consistent contrast across datasets |
| Base opacity (`O Scalar Layer to Factor`) | `clamp(0.22 * sqrt(256/depth_vox), 0.08, 0.35)` | thicker volume → lower base so it stays **translucent**, not solid |
| Boundary boost (gradient→alpha) | `scale = 3.0 / g_med` | edges/membranes pop; normalized by median so it behaves the same on any data |
| `O Realize Structure` Density | `clamp(40 * (1.0/vox_mm), 20, 80)` | finer voxels → scale so the surface reads consistently |
| Emission (internal glow) | `= 0.12 * (base_opacity/0.2)` | small, tied to density → lit-from-within, never a light bulb |

Ramp **value** is always monotonic dark→light. Ramp **hue family** is chosen ONCE per project (§3),
never per shot.

## 3. FIXED (identical every time — set once, reuse forever)

- **Volume path:** `volume_biased = True` (required on Blender 5.x or step rate is ignored).
  **[RECONCILED with the master spec]** The master spec's committed default is the *unbiased*
  integrator (its §11 Issue 6). This law **deliberately overrides that for the shots it governs**,
  because step rate is one of its tier knobs (§4) and step rate only exists on the biased path.
  That is a second committed choice, not a contradiction: shots under this law flip the three
  settings together in `render_8k.py` (the commented fallback block goes live —
  `volume_biased=True` + the tier's step rate + max steps 1024), and they stay identical across
  every frame of a clip. Scenes *not* under this law keep the master default.
- **World:** black (`0.002`), `film_transparent = False`.
- **Lighting rig:** warm key area light 45°/above (soft); **cool rim** behind/opposite to catch the
  silhouette against black; low/zero ambient so black stays black and glow reads.
- **Camera:** 85 mm hero / 50 mm context, subject on thirds; hero DoF f/4; clip = slow orbit/push,
  camera parented to an Empty + `Track To` subject, curves Bezier ease-in/out.
  **[RECONCILED]** The master spec's 50 mm lock is this law's *context* lens; heroes shoot 85 mm.
- **Color:** AgX view transform; compositor **low Fog-Glow glare** + gentle vignette + slight contrast.
  Grade on the EXR, **never** baked into the volume data.
- **Fidelity:** `O Layer` Resample **OFF** for final; samples 4096 / adaptive 0.005; OIDN + Normal+Albedo
  data passes.
- **Hue family (pick one, project-wide):** `neutral-cool` (blue-grey→white), `tissue-warm`
  (maroon→cream), or `ice` (deep teal→white). Value ramp is identical across all three.

## 4. QUALITY TIERS (selected by PHASE, not per shot)

| Tier | Volume bounces | Step rate | Samples | Adaptive | Use |
| --- | ---: | ---: | ---: | ---: | --- |
| preview | 4 | 0.6 | 512 | 0.02 | iterate framing/threshold |
| balance | 8 | 0.3 | 1024 | 0.01 | look-dev loop |
| **hero** | **16** | **0.15** | **4096** | **0.005** | final |

(The technical spec's generic `volume_bounces=4` is the *preview* tier — never ship a hero on it.)

**[ADDED — the ceiling must be re-derived per tier]** Hero tier (16 bounces, step 0.15) is several
times heavier per frame than the master spec's baseline. The §10.4 dead-man ceiling formula
(`smoke min/frame × frames × 1.5`) only protects you if the smoke runs the **same tier** as the
final — a preview-tier smoke under-predicts a hero-tier 8K frame badly. Smoke at 1080p on **hero
tier**, then derive `<CEILING_MIN>` from that number.

## 5. DETERMINISTIC ITERATION (symptom → the ONE knob — don't flail)

| It looks like… | Turn this knob |
| --- | --- |
| fog / no form | ↑ boundary boost `scale`, then ↑ threshold toward `P50` |
| solid / opaque brick | ↓ base opacity |
| flat / no depth | ↑ volume bounces (tier up); strengthen the rim; ↑ emission slightly |
| washed out / milky | confirm AgX is applied at output; ↓ emission |
| rainbow / garish | collapse the hue spread — make the ramp value-only |
| gray blob | `O Scalar Layer to Color` isn't driving — check the graph is wired |
| crushed blacks | ease the vignette; tiny fill light |
| noisy / grainy | ↑ samples; confirm OIDN + Normal/Albedo passes are on |
| edges shimmer across frames | **same** threshold + **same** resample res every frame (never per-frame) |

3–5 loops from `preview` to a clean contact sheet, then one `hero` render. Reject a contact sheet that
is plastic, plain fog, rainbow, gray blob, crushed, blown-out, or depthless.

## 6. One-screen check
1. Measured the six numbers (§1) off the cache.
2. Derived threshold / ramp stops / opacity / boundary boost / density / emission (§2).
3. Fixed rig applied (§3): black world, key+rim, 85 mm, AgX+glare, Resample OFF.
4. Tier = hero for final (§4); previewed at lower tier first.
5. Iterated by the symptom→knob table (§5), not by guessing.
6. Same threshold + resample res on every frame of a clip.
7. **[ADDED]** `look.json` (measurements + derived values) staged in `jobs/<RUN_ID>/in/` with the
   `.blend`+cache; hero smoke ran at hero tier before the ceiling was derived.

If all six hold, the look is repeatable on the *next* dataset with zero re-guessing — you just re-measure
and the rules resolve again.
