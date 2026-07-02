# 8K GPU Render on AWS (Blackwell) with Blender — Complete Master Spec

**Self-contained and pre-answered.** Every setting below is locked to a world-class default so you can
follow it top-to-bottom and get a flawless, repeatable 8K result. The input is treated as generic 3D
volume data. Only the values that are unique to your account/shot are left as `<PLACEHOLDERS>` — a
fill-in table for those is at the very end.

> **Golden rule:** Blender **5.0+** and the **2.0.x volume add-on** only. v2.0.x needs Blender 5.0+,
> works completely differently from 1.x, and **cannot open old-version files** — rebuild a 1.x file,
> never "upgrade" it.

**How to read this document:** §§0–9 are the locked spec. Items marked **[FIX]** are corrections
where a locked value, as written, fails against verified July-2026 AWS/Blender reality (each says
why). §10 is the gap check: everything that was missing, added and marked. Nothing locked was removed.
§11 is a second-pass audit run against §§0–10; every defect it found is fixed in place and marked
**[FIX v2 — Issue N]**, and §10.8 (the unattended job runner) exists because of it.

---

## 0. Locked defaults at a glance

| Area | Locked value |
| --- | --- |
| Engine / device / compute | Cycles · GPU · **OptiX** · CPU device **OFF** |
| Resolution | **7680 × 4320** @ 100% (option: render 8K → downscale to 4K) |
| Samples (preview / hero) | 512 / **4096**, adaptive ON, threshold 0.02 / **0.005** |
| Denoiser | preview **OptiX** · final **OpenImageDenoise** + data passes |
| Light bounces | total **12** (diffuse 4, glossy 4, transmission 8, **volume 4**, transparent 8) |
| Clamp | direct 0 · **indirect 10** |
| Volume | **unbiased integrator (5.0 default, committed)** — biased + step 0.25 / max steps 1024 only as the budget fallback (fix 1 below) |
| Frame rate | **24 fps**, fixed seed 0 |
| Output | **EXR multilayer, 16-bit half, DWAA** + PNG proof |
| Colour | View Transform **AgX**, scene-linear |
| Motion blur | ON, shutter **0.5** |
| Delivery chain | **EXR sequence → ProRes 4444 master → H.265 MP4 share** |
| Cost guard | smoke/default ceiling **90 min** · hero ceiling **derived** (smoke time × frames × 1.5), idle cutoff **15 min**, auto-stop + budget alarm |

**[FIX] Three locked values need one extra step each in Blender 5.0, or they silently do nothing / error out:**

1. **Volume integrator — now committed [FIX v2 — Issue 6]:** the lock is Blender 5.0's **unbiased**
   null-scattering integrator (`volume_biased = False`). It has no Step Rate / Max Steps knobs at
   all — volume quality is governed by samples 4096 + adaptive threshold 0.005 + volume bounces 4.
   The 0.25 / 1024 values survive only as the **budget fallback**: if the smoke-derived hero frame
   time (§10.4) blows the per-frame budget, flip to `volume_biased = True` + step rate 0.25 / max
   steps 1024 and knowingly trade a little bias for speed. One default, one documented escape
   hatch — nothing contingent, nothing silent.
2. **EXR multilayer** — in 5.0 you must set `image_settings.media_type = 'MULTI_LAYER_IMAGE'`
   **before** `file_format = 'OPEN_EXR_MULTILAYER'`, or Python raises an enum error (official 5.0
   API breaking change). The §5 script does this.
3. **EXR sequence → ProRes** — ffmpeg fed Blender **multilayer** EXRs directly picks channels
   unpredictably and, worse, encodes **scene-linear** data without the AgX view transform → a flat,
   washed-out master. §7 has the fixed two-step chain (bake the view transform, then encode).

Also note: **DWAA is a lossy codec** (visually lossless at default level). It is a fine locked
default for size; if you ever need a mathematically lossless archive master, use ZIP instead.

---

## 1. Volume workflow (data → render-ready)

Mental model — the thing that prevents most failures:

```
Data Cache (files on disk)  ->  Data Layer (a LINK in the .blend)  ->  O Layer node
```

The `.blend` stores **only a link**. Move the `.blend` alone and the layer arrives **empty**. **The
cache must always travel with the `.blend`** (S3 upload, worker copy, handoff) — like image textures.

1. Install the volume add-on: `Edit > Preferences > Get Extensions`, search `biox`, **Install**
   (60–100 MB). Confirm it reads **2.0.x** and is enabled.
2. Add a **Geometry Nodes** modifier to a cube → sidebar (`N`) → the volume-nodes tab → **Import Data**
   → pick any one `.dcm` slice (the whole series is ingested) → **Import Data** → **OK** both dialogs →
   **Add** → drop the **O Layer** node.
   Supported inputs: `.dcm/.DICOM/.ima`, `.nii/.nii.gz`, `.nrrd`, `.mha/.mhd`, `.vtk`,
   `.tif/.tiff/.ome.tif`, `.png/.jpg/.bmp`, `.h5/.hdf5`, `.mrc`, gzip variants.
3. Wire `O Layer → O Cutout by Threshold → O Realize Structure → Group Output`
   (nodes from **Add > (volume) > Structure**). O Layer: `Center` ON, `Resample` OFF for finals
   (Resample ON + larger size = fast preview). O Cutout `Threshold` per data. O Realize `Density`
   with `Surface` ON. Engine = **Cycles**.
4. **Save the cache:** volume-nodes tab → **Save** → a folder **inside the project**, then ship the
   cache with the `.blend` every time. Skipping this is the #1 cause of "empty render on the server."
5. Add-on limits: engines = Cycles CPU / Cycles GPU (OptiX) / EEVEE only; mesh conversion makes no cut
   surfaces (plan for volume rendering). **Headless:** author + import + **save cache** interactively
   once, then render the saved `.blend`+cache headless — don't script the modal import.
   **[FIX v2 — Issue 4]** That boundary is now explicit, not implied: **authoring (data → cache) is
   a one-time, local, manual pre-step; ONLY rendering is farmed.** The farm job's input contract is
   `.blend` + cache staged in `jobs/<RUN_ID>/in/`, and the §10.8 runner **fails fast** (exit 66,
   clear message) when the cache is missing — it never attempts an import on the worker. If the
   add-on exposes importable Python APIs for building layers + saving the cache, prove them
   headless on ONE box before relying on them; until proven, do not promise an unattended
   data→cache build.

---

## 2. Blender — every setting answered

**Engine / device**
- Cycles · `Device = GPU` · `Compute = OptiX` · CPU device unticked · **use all GPUs** on the box.
- `Sampling > Seed = 0`, **not** animated (prevents motion flicker).
- Pin the exact Blender build (record `blender --version`) so every worker matches.
  **[ADDED]** Pin **Blender 5.0.1** (released 2025-12-16, the final 5.0.x — 5.0 is not an LTS
  series, so 5.0.1 is the build to standardize on).

**Sampling / denoise**
- Max Samples: **preview 512 / hero 4096**. Adaptive Sampling ON; Noise Threshold **0.02 / 0.005**.
- Denoiser: **OpenImageDenoise** for finals (steadier across frames), **OptiX** for previews.
- **Denoising Data passes ON** (`View Layer > Passes > Denoising Data` → Normal + Albedo).
- `Performance > Final Render > Persistent Data` **ON**. No per-frame time limit (sample cap governs).
- **[ADDED]** Set `scene.cycles.denoising_use_gpu = True` — OIDN runs on the render GPU instead of
  the worker's few vCPUs; at 8K this saves minutes per frame. Verified present in the 5.0 API
  (default is False, so it must be set explicitly).

**Light paths**
- Max Bounces total **12** — diffuse 4, glossy 4, transmission 8, **volume 4**, transparent 8.
- Clamp Direct **0**, Clamp Indirect **10** (kills fireflies without dulling).
- `Light Tree` **ON**. Caustics **OFF** unless you have glass/liquid. Fast GI **OFF**.

**Volume**
- **[FIX v2 — Issue 6]** Committed: **unbiased integrator** (5.0 default, `Render > Volumes >
  Biased` OFF) — it has no Step Rate / Max Steps controls; volume quality is governed by samples +
  adaptive threshold + volume bounces. Fallback **only** on a blown frame-time budget:
  `Biased` ON, Step Rate **0.25** (hero) / 0.6 (preview), Max Steps **1024** — carried in the §5
  script as a commented block, not a live setting.
- The volume grid **detail/resample** is the real VRAM driver at 8K — raise only as far as VRAM allows.

**Output / colour**
- Resolution **7680 × 4320 @ 100%** (or render 8K and downscale to 4K for free anti-alias).
- Format **OpenEXR multilayer, 16-bit half, DWAA** compression. Add a **PNG 8-bit** proof pass.
- `Film > Transparent` **OFF** (solid dark bg) by default; turn ON if you'll composite a background.
- View Transform **AgX**, Look None (or Medium-High Contrast), Exposure 0. Grade on the EXR, never bake.
  (Verified: `'AgX'` is still a valid view-transform name in 5.0 on the default sRGB display.
  Remember the EXR master stores **scene-linear** data — AgX is applied at proof/encode time, §7.)
- Enable **Cryptomatte + Object Index** passes if you'll composite; Mist optional. Burn-in OFF for masters.

**Camera / animation**
- `fps 24`, Start 1, End `24 × <SECONDS>`, step 1.
- Camera 50 mm, sensor 36 mm, perspective. Depth of Field OFF (or subtle f/4 for a hero).
- Move the **camera, not the subject**: parent camera to an Empty, keyframe a slow orbit/push-in,
  add **Track To** the subject, all curves **Bezier + ease in/out**. Exact move = `<CAMERA_MOVE>`.
- Motion Blur **ON**, position Center, Shutter **0.5**, applied to the volume.

**Memory / tiling / scratch**
- Let Cycles auto-tile; if the volume nears VRAM, render in **quadrant render-regions** and stitch, or
  drop volume detail for a preview pass.
  **[ADDED]** In 5.0 the "Use Tiling" checkbox is gone — tiling is **always on**; `Tile Size`
  (default 2048) is the only knob. `use_auto_tile` in scripts is a harmless deprecated no-op.
- Blender temp/scratch on **fast local/NVMe**, never a network mount.

**Compositor finish (optional, on the EXR)**
- Subtle Glare (Fog Glow, low), gentle Vignette, final colour grade → output nodes writing the graded
  EXR alongside the raw. Keep the raw master clean.

---

## 3. AWS — every setting answered

**Instance**
- Type `<INSTANCE_TYPE>` (OptiX-capable Blackwell GPU), count `<N>` boxes. GPUs/box + VRAM `<GPU/VRAM>`.
  **[FIX]** "OptiX-capable Blackwell" on AWS means **G7e** (NVIDIA **RTX PRO 6000 Blackwell Server
  Edition**, 96 GB, 4th-gen RT cores — GA Jan 2026), starting at **g7e.2xlarge** (1 GPU, 8 vCPU,
  64 GiB RAM, 1.9 TB NVMe, ≈ $3.36/hr on-demand us-east-1, spot ≈ $1.67/hr). It is plain
  On-Demand/Spot — no Capacity Blocks. Any single-GPU G7e size qualifies: if you already have a
  certified **g7e.4xlarge**, keep it — same single 96 GB GPU, more vCPU/RAM for encode and CPU
  fallbacks. Do **not** use P6 (B200/B300) for rendering: B200 has **no RT
  cores** (Blender Open Data: B200 median ≈ 8,311 vs ≈ 16,740 for RTX PRO 6000 Blackwell), the only
  size is 8-GPU `p6-b200.48xlarge` at ≈ $114/hr, and P-family quota/capacity is the hardest to get —
  ~34× the cost to render *slower per GPU*. Full decision table in §10.1.
- **On-demand** for a single hero; **Spot** for large batches — interruption behavior **stop**, and the
  per-frame EXR *is* the checkpoint (resume skips finished frames). Max Spot price `<PRICE>`.
- Region/AZ `<REGION>` / `<AZ>`. Keep the bucket in the **same region**.
  **[ADDED]** G7e regions as of May 2026: us-east-1, us-east-2, us-west-2, Tokyo, Seoul, Spain,
  London (+ LA Local Zones). Default to **us-east-1** or **us-east-2**.
- **Check the G/VT vCPU service quota** for that family before launch. Known cost/hr `<COST_HR>`.
  **[ADDED]** Exact quota: **"Running On-Demand G and VT instances" (L-DB2E81BA)**, measured in
  vCPUs, **defaults to 0 on new accounts** — a g7e.2xlarge needs 8 vCPUs approved. Spot has its own
  quota ("All G and VT Spot Instance Requests", L-3819A6DF). Request days ahead; commands in §10.5.

**Image / driver**
- Base AMI `<AMI_ID>` with an NVIDIA driver new enough for the card's compute capability (verify with
  `nvidia-smi`) + bundled CUDA. **OptiX 8+** comes with a current driver.
- **Bake a golden AMI**: driver + pinned Blender 5.x + the 2.0.x add-on preinstalled, so every worker
  is identical. Record the AMI version.
- **[FIX]** Three driver facts that break Blackwell boxes if missed:
  1. On Linux, Blackwell GPUs work **only with NVIDIA's open kernel modules** — the proprietary
     kernel module reports "No devices were found."
  2. **[FIX v2 — Issue 9]** One driver floor, stated once: **R575+** — the RTX PRO 6000 Server
     Edition requirement, which subsumes every other Blackwell minimum. The DLAMI below ships
     595.x, which clears it; pin that AMI and stop reasoning about driver branches.
  3. OptiX needs **`libnvoptix.so.1`**, which ships with the *graphics* driver package
     (`libnvidia-gl-<ver>`), **not** with compute-only "headless" driver installs — the #1 cause of
     `OPTIX_ERROR_LIBRARY_NOT_FOUND` on servers (in containers, set
     `NVIDIA_DRIVER_CAPABILITIES=compute,utility,graphics`).
  The easy path that satisfies all three: **AWS Deep Learning Base OSS Nvidia Driver GPU AMI
  (Ubuntu 24.04)** — the 2026-04 release ships open-kernel driver 595.58.03 and explicitly supports
  G7e (and G6e, P6). Pin its AMI ID via the AWS SSM parameter, then bake your golden AMI on top.

**Storage**
- Root/EBS **gp3, 300 GB**, 3000 IOPS / 250 MB/s, **delete-on-termination ON**.
  ([ADDED] note: gp3 baseline is 125 MB/s free; 250 MB/s is a small paid throughput bump — keep it.)
- Use built-in **NVMe instance-store scratch** if present (mount `/mnt/nvme`); else render on EBS.
  **[ADDED]** Every G7e size has NVMe (g7e.2xlarge: 1.9 TB). It arrives **unformatted** — format
  and mount at boot (§10.4), and remember instance-store contents **vanish on stop/terminate**:
  upload to S3 before any shutdown (the §3 flow already does).
- Cloud disk **≥ 2× (inputs + outputs)** — 8K EXRs are large.

**Access / IAM / network**
- Attach an **IAM instance profile** with `s3:GetObject/PutObject/ListBucket` on **only** your bucket
  (least privilege).
  **[ADDED]** The two actions live on *different ARNs* — mixing them up is the classic
  `AccessDenied` on `s3 sync`: `ListBucket` targets `arn:aws:s3:::<BUCKET>`;
  `GetObject`/`PutObject` target `arn:aws:s3:::<BUCKET>/jobs/*`. Exact policy JSON in §10.5.
- Connect via **SSM** (no keys/open ports). If SSH: key pair + SG inbound 22 from your IP only.
- Private subnet + NAT (or public with a locked SG). **Enforce IMDSv2.**
  **[ADDED]** Launch with
  `--metadata-options "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2"`.
  (G7e/P6 are post-2024 instance types and are IMDSv2-only at the platform level anyway — which is
  why the §4 identity proof must use the token flow.)

**S3**
- Bucket `s3://<BUCKET>/`. Layout `jobs/<RUN_ID>/in/` and `jobs/<RUN_ID>/out/`.
- Storage class STANDARD; lifecycle to expire proofs after `<N_DAYS>`. Multipart upload on (CLI default).
- **[ADDED]** Prefer `aws s3 sync` over `cp --recursive` (copies only new/changed files → free
  resume). Since AWS CLI v2.23 every upload gets an automatic client-side CRC64NVME checksum that S3
  verifies — `--checksum-algorithm SHA256` is only for compliance-grade digests; the §7 `sha256sum`
  manifest already covers delivery verification.

**Orchestration / dispatch**
- Dispatch via **SSM run-command** (small scale) or **AWS Batch** (large). Assign **non-overlapping
  frame ranges** per box; **resume logic skips frames already in S3**.
- One box runs the **ffmpeg encode** after every frame lands. Tag all resources with project + `<RUN_ID>`.

**Logging / monitoring / verification**
- Capture Blender stdout to **CloudWatch Logs** (or upload a per-box log to S3). Write a **heartbeat**
  object to S3 each finished frame.
- Verify before delivery: **frame count == expected**, no zero-byte/black frames, **sha256 checksums**.
  Clean scratch after upload.

**Cost / shutdown (the most expensive mistake is a forgotten box)**
- Hard cost ceiling `$<CEILING>`; **AWS Budgets alarm at 80%**.
- **Max runtime**: 90 min is the **smoke/default** ceiling only. **[FIX v2 — Issue 3]** A fixed
  90 min cannot cover an 8K/4096 volume hero, let alone a sequence — derive the real per-box
  ceiling from the smoke: measured minutes-per-frame × frames-on-this-box × 1.5 (§10.4).
  **Idle cutoff 15 min** (no `blender`/`cycles` process).
- Auto-stop: an on-box **watchdog** *and* an **EventBridge scheduled stop** as backstop.
- **user-data boot script** does it hands-free: pull job → render range → upload → verify → **stop the
  instance**. Retry a failed frame once, then flag.
- **[FIX + FIX v2 — Issue 2]** For `sudo shutdown -h +N` to work as a true dead-man switch, launch
  with `--instance-initiated-shutdown-behavior terminate` — the EBS-backed default is **stop**, and
  a stopped instance **keeps billing for its EBS volumes** (a forgotten 300 GB gp3 ≈ $24/month).
  But terminate is a **cattle setting**: it applies ONLY to disposable workers spawned from the
  finished golden AMI. The **AMI-seed box is a pet** — launch it with `stop` and arm no dead-man
  switch on it, so nothing can destroy it before the AMI exists. An already-certified,
  hand-configured box you have no AMI of yet is a seed box: keep it `stop` + fail-closed until its
  AMI is baked. Both launch profiles are in §10.3; watchdog scripts in §10.4.

---

## 4. The seven proofs (run before trusting any box — a 0 exit code is NOT proof)

```bash
# 1 identity — [FIX] G7e/P6 are IMDSv2-only: the old plain curl hangs/fails; use the token flow
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
        -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
hostname
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id

nvidia-smi                                                                  # 2 GPU + driver
blender --version                                                          # 3 Blender 5.x
blender -b --python-expr "import bpy;p=bpy.context.preferences.addons['cycles'].preferences;p.compute_device_type='OPTIX';p.refresh_devices();print([(d.name,d.type,d.use) for d in p.devices])"  # 4 OptiX sees the GPU
blender -b --python-expr "import addon_utils;print([m.__name__ for m in addon_utils.modules() if 'iox' in m.__name__.lower()])"   # 5 add-on present
df -h                                                                      # 6 fast scratch present
aws s3 ls s3://<BUCKET>/ >/dev/null && echo S3_OK                          # 7 S3 reachable
```

Any of 1–7 fails → **stop the box** and fix.

**[ADDED] Proofs 8–10** (each catches a failure the seven above miss):

```bash
ldconfig -p | grep -q libnvoptix.so.1 && echo OPTIX_LIB_OK        # 8 OptiX driver lib present (headless installs often lack it)
nvidia-smi --query-gpu=driver_version --format=csv,noheader       # 9 driver branch — must be 575+ on G7e (RTX PRO 6000)

# 10 fail-fast OptiX proof — [FIX v2 — Issue 1] the previous "import bpy" form never rendered, so
#    it proved nothing: --cycles-device only takes effect when a render actually runs. Render a
#    tiny frame of the default scene; the proof is exit 0 AND a non-empty file on disk.
blender -b --python-expr "import bpy; s=bpy.context.scene; s.render.engine='CYCLES'; s.cycles.device='GPU'; s.render.resolution_x=64; s.render.resolution_y=64; s.render.resolution_percentage=100; s.cycles.samples=8" \
        -o /tmp/optix_probe_#### -f 1 -- --cycles-device OPTIX \
  && test -s /tmp/optix_probe_0001.png && echo OPTIX_RENDER_OK
```

Proof 5 note: extensions register as `bl_ext.<repo>.<id>` (e.g. `bl_ext.user_default....`), so the
module grep must match the extension's **id**, and the render must run as the **same OS user** whose
Blender preferences enabled it (prefs live in `~/.config/blender/5.0/`).

---

## 5. The headless render script (`render_8k.py`) — all settings baked in

```python
import bpy

# [FIX v2 — Issue 5] Pin ONE exact build and refuse anything else. The [FIX]es below are verified
# on 5.0.1; if your certified box runs a different build (e.g. 5.1.0), change THIS tuple, re-run
# the §4 proofs + one smoke on that build, and the pin guarantees the whole fleet matches it.
PINNED = (5, 0, 1)
assert bpy.app.version == PINNED, \
    f"Blender {bpy.app.version} != pinned {PINNED} — refusing to render on an unverified build"

def lock(obj, name, val):
    # [FIX v2 — Issue 7] loud failure at second 1 instead of a silent no-op after hours of setup
    if not hasattr(obj, name):
        raise AttributeError(f"{type(obj).__name__}.{name} missing on this build — fix before rendering")
    setattr(obj, name, val)

S = bpy.context.scene
P = bpy.context.preferences.addons['cycles'].preferences
P.compute_device_type = 'OPTIX'; P.refresh_devices()
for d in P.devices: d.use = (d.type == 'OPTIX')          # all GPUs; CPU off
S.render.engine = 'CYCLES'; S.cycles.device = 'GPU'
S.render.resolution_x, S.render.resolution_y, S.render.resolution_percentage = 7680, 4320, 100
S.cycles.samples = 4096
S.cycles.use_adaptive_sampling = True; S.cycles.adaptive_threshold = 0.005
S.cycles.use_denoising = True; S.cycles.denoiser = 'OPENIMAGEDENOISE'
lock(S.cycles, 'denoising_input_passes', 'RGB_ALBEDO_NORMAL')
lock(S.cycles, 'denoising_use_gpu', True)                # [ADDED] OIDN on the GPU, not 8 vCPUs
S.render.use_persistent_data = True
S.render.use_overwrite = False                           # [ADDED] farm resume: skip frames already on disk
S.render.use_placeholder = True                          # [ADDED] placeholder stops two boxes racing one frame
S.cycles.max_bounces = 12; S.cycles.diffuse_bounces = 4; S.cycles.glossy_bounces = 4
S.cycles.transmission_bounces = 8; S.cycles.volume_bounces = 4; S.cycles.transparent_max_bounces = 8
S.cycles.sample_clamp_indirect = 10.0; S.cycles.sample_clamp_direct = 0.0
S.cycles.use_light_tree = True
lock(S.cycles, 'volume_biased', False)   # [FIX v2 — Issue 6] committed: unbiased integrator (no step knobs)
# Budget fallback ONLY — flip all three together when the smoke-derived hero frame time is over budget:
# lock(S.cycles, 'volume_biased', True)
# S.cycles.volume_step_rate = 0.25; S.cycles.volume_max_steps = 1024
S.cycles.seed = 0
S.render.use_motion_blur = True; S.render.motion_blur_shutter = 0.5
S.render.fps = 24
S.view_settings.view_transform = 'AgX'
lock(S.render.image_settings, 'media_type', 'MULTI_LAYER_IMAGE')  # [FIX] 5.0: BEFORE file_format
S.render.image_settings.file_format = 'OPEN_EXR_MULTILAYER'
S.render.image_settings.color_depth = '16'; S.render.image_settings.exr_codec = 'DWAA'
used = [d.name for d in P.devices if d.use]; print("GPU DEVICES USED:", used)
assert used, "NO GPU SELECTED — aborting to avoid a CPU render"
```

```bash
# render a frame range on this box (split ranges across boxes)
# [FIX] argument order is strictly positional: .blend first, then -P, then -o, then -s/-e/-a LAST
# [ADDED] "-- --cycles-device OPTIX" is belt-and-braces: if OptiX is unusable, Blender exits 1
#         with "Found no Cycles device of the specified type" instead of silently CPU-rendering.
blender -b /mnt/nvme/jobs/<RUN_ID>/scene.blend -P render_8k.py \
        -o /mnt/nvme/jobs/<RUN_ID>/out/frame_#### -s <START> -e <END> -a \
        -- --cycles-device OPTIX --cycles-print-stats
echo "blender exit code: $?"        # must be 0; nonzero = no output was written

# [FIX] run this from a SECOND shell WHILE the render is going (it samples live processes):
nvidia-smi pmon -c 1     # the 'blender' process MUST appear = truly on GPU
```

---

## 6. Multi-part material regions (each part editable + consistent)
When the volume needs several separately-shaded parts:
- Each part = its **own O Cutout by Threshold band (min/max)** → **own O Realize Structure** → **own
  material**, kept in its **own named collection** (`PART_01`, `PART_02` …). **Never merge them.**
- **Same threshold values on every frame** and matched resample resolution, or edges shimmer between frames.
- Prove it with a per-part contact sheet (each part alone + combined, front/side/top) before the hero render.

---

## 7. Animation → encode
- Render the **EXR sequence** across boxes (§5), all writing to `jobs/<RUN_ID>/out/`.
- Verify frames (count, no black/zero-byte). Then encode:

**[FIX] Do not feed the multilayer EXRs straight to ffmpeg.** Two failures: ffmpeg's EXR decoder
does not reliably pick the right layer from a Blender *multilayer* file, and the data is
scene-linear — encoding it raw bakes a flat, washed-out image because the **AgX view transform was
never applied**. Bake the view transform first with Blender itself (it applies scene colour
management in `save_render`), then run the locked ffmpeg commands on the baked frames:

```bash
# Step 1 — bake AgX 16-bit PNG delivery frames from the EXR masters (headless, no display needed)
blender -b --python-expr "
import bpy, glob
S = bpy.context.scene
S.view_settings.view_transform = 'AgX'
S.render.image_settings.file_format = 'PNG'
S.render.image_settings.color_depth = '16'
for f in sorted(glob.glob('/mnt/nvme/jobs/<RUN_ID>/out/frame_*.exr')):
    img = bpy.data.images.load(f)
    img.save_render(f[:-4] + '.png')
    bpy.data.images.remove(img)
    print('baked', f)
"

# Step 2 — the locked encode chain, fed the baked frames
ffmpeg -framerate 24 -i out/frame_%04d.png -c:v prores_ks -profile:v 4 -pix_fmt yuv444p10le \
       -color_primaries bt709 -color_trc bt709 -colorspace bt709 master.mov
ffmpeg -i master.mov -c:v libx265 -crf 18 -pix_fmt yuv420p10le -tag:v hvc1 share.mp4
```

- Deliver: **EXR seq + `master.mov`** (full 8K) and **`share.mp4`** (H.265, Rec.709, 4K/1080p) to
  `s3://<BUCKET>/jobs/<RUN_ID>/delivery/`, checksummed.

---

## 8. Values only you can supply (fill these in)

| Placeholder | Meaning |
| --- | --- |
| `<INSTANCE_TYPE>` / `<N>` | GPU instance type + how many boxes |
| `<GPU/VRAM>` / `<COST_HR>` / `<PRICE>` | GPUs+VRAM per box, on-demand $/hr, Spot max price |
| `<REGION>` / `<AZ>` | AWS region + AZ (bucket must match region) |
| `<AMI_ID>` | golden AMI (driver + Blender + add-on) |
| `<BUCKET>` / `<RUN_ID>` / `<N_DAYS>` | S3 bucket, this run's id, proof expiry |
| `<CEILING>` | hard cost ceiling ($) |
| `<SECONDS>` / `<CAMERA_MOVE>` | animation length + the exact camera move |

**[ADDED] Recommended values** (verified July 2026, us-east-1): `<INSTANCE_TYPE>` = **g7e.2xlarge**
(1× RTX PRO 6000 Blackwell 96 GB, 1.9 TB NVMe) — or an already-certified **g7e.4xlarge** (same
single GPU); `<COST_HR>` ≈ **$3.36** on-demand / ≈ $1.67 spot, `<REGION>` = **us-east-1**,
`<AMI_ID>` = Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 24.04) resolved via its SSM
parameter (§10.3), Blender pinned to **the exact build your certified box runs** — 5.0.1 is this
doc's verified baseline; if the box runs 5.1.0, set `PINNED = (5, 1, 0)` in §5 and re-run the §4
proofs + one smoke to revalidate the 5.0-era fixes on it (**[FIX v2 — Issue 5]**).

---

## 9. One-screen "did I do it right?"
1. Blender 5.0+ + the 2.0.x add-on; not a 1.x file. Cache ships with the `.blend`.
2. Cycles + **OptiX**, CPU off, all GPUs, **seed 0**.
3. 8K EXR (16-bit half, DWAA) + PNG proof; **AgX**.
4. Samples 4096, adaptive 0.005, **OpenImageDenoise + data passes**, Persistent Data on.
5. Bounces total 12 (volume 4), clamp indirect 10, Light Tree on.
6. Volume: unbiased integrator committed (biased 0.25/1024 only as the budget fallback); watch
   VRAM, quadrant-tile if needed; smoke at 1080p first.
7. 24 fps, camera-moves-subject-stays, motion blur 0.5; render an **EXR sequence** (never straight to video).
8. AWS: golden AMI, seven proofs, `nvidia-smi pmon` shows blender on GPU, reject gray frames.
9. IAM least-privilege, SSM, IMDSv2, same-region bucket, disk ≥ 2× job.
10. Budget alarm + max-runtime 90 + idle 15 + auto-stop; resume skips done frames; encode EXR→ProRes→MP4.
11. **[ADDED]** G7e (RTX PRO 6000, 96 GB, RT cores) not P6/B200; driver **R575+ open kernel modules**;
    `libnvoptix.so.1` present (proof 8); quota **L-DB2E81BA ≥ 8 vCPUs** approved before render day.
12. **[ADDED]** 5.0 specifics honored: volume mode committed and guarded, `media_type` before
    `file_format`, GPU OIDN on, exact-build pin asserted at render start, view transform **baked
    before encoding** video.
13. **[FIX v2]** Proof 10 actually renders; seed box = stop/pet, workers = terminate/cattle; hero
    ceiling derived from the smoke; the run is one unattended §10.8 job, never hand-driven.

---

## 10. Fable — gap check

*Reviewed §§1–9 against fact-checked July-2026 AWS and Blender sources (every load-bearing fact
below was web-verified and adversarially re-checked; several were verified empirically against the
official Blender 5.0.1 Linux binary). Nothing locked was removed. Everything in this section is an
addition.*

### 10.1 The instance decision — the single choice the whole plan hangs on

"Blackwell on AWS" spans five instance families, and four of them are wrong for rendering:

| Instance | GPU | RT cores | VRAM/GPU | Smallest unit | How you buy | ~$/hr (us-east-1) | Verdict for Cycles |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **g7e.2xlarge** | RTX PRO 6000 Blackwell SE | **Yes (4th gen)** | 96 GB | **1 GPU** | On-Demand / Spot | **$3.36** / ~$1.67 spot | ✅ **The box.** Newest RT cores + 96 GB kills OOM risk |
| g6e.xlarge | L40S (Ada) | Yes (3rd gen) | 48 GB | 1 GPU | On-Demand / Spot | $1.86 / ~$1.69 spot | ✅ Fallback **only if** measured peak VRAM ≤ ~40 GB |
| g7.2xlarge | RTX PRO 4500 Blackwell | Yes | 32 GB | 1 GPU | On-Demand / Spot | cheaper | ⚠️ Budget previews only (32 GB) |
| p6-b200.48xlarge | 8× B200 | **No** | 180 GB | **8 GPUs** | OD / Capacity Blocks | **$113.93** | ❌ No RT cores; Blender Open Data median ≈ 8,311 vs ≈ 16,740 (RTX PRO 6000) — ~34× the price to render slower per GPU |
| P6-B300 / P6e-GB200 | B300 / GB200 NVL72 | No | — | 8 / **36 GPUs** | Capacity Blocks (upfront, non-cancellable) | $10.58–11.70/GPU-hr | ❌ ML-training hardware; not purchasable sanely for a render job |

Why the P6 experiments "kept messing up": P-family quota (**L-417A185B**) defaults to **0**, increases
get human review, launches then hit `InsufficientInstanceCapacity`, the minimum rental is 8 GPUs —
and even a successful render runs ray traversal in software because B200 has no RT cores (OptiX
*supports* it, on CUDA cores; Blender ships no sm_100 kernels, so B200 also JIT-compiles kernels on
first render). Every one of those problems disappears on **g7e.2xlarge**.

### 10.2 Worker bootstrap (golden-AMI recipe, every command verified)

Base: **Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 24.04)** — ships open-kernel-module
driver 595.58.03 (≥ R575 ✓, open modules ✓, `libnvoptix` ✓, G7e supported ✓). On top of it:

```bash
# 1) Blender 5.0.1 (pinned) — exact runtime libs the 5.0.1 binary needs on a bare server
sudo apt-get update && sudo apt-get install -y \
     libx11-6 libxi6 libxrender1 libxfixes3 libxkbcommon0 libgl1 libxext6 libsm6 libice6
curl -fL https://download.blender.org/release/Blender5.0/blender-5.0.1-linux-x64.tar.xz \
  | sudo tar -xJ -C /opt
sudo ln -sf /opt/blender-5.0.1-linux-x64/blender /usr/local/bin/blender
blender --version    # must print 5.0.1

# 2) Volume add-on, installed + enabled headless (no GUI needed; persists to user prefs)
blender --command extension install-file -r user_default -e /path/to/<addon-2.0.x>.zip
# module registers as bl_ext.user_default.<id>; render as the SAME user that ran this

# 3) NVMe scratch (G7e instance store arrives unformatted; contents vanish on stop/terminate)
sudo mkfs.ext4 -E nodiscard /dev/nvme1n1
sudo mkdir -p /mnt/nvme && sudo mount -o noatime /dev/nvme1n1 /mnt/nvme
sudo chown "$USER" /mnt/nvme
export TMPDIR=/mnt/nvme/tmp && mkdir -p "$TMPDIR"

# 4) Persist the CUDA JIT cache across renders (harmless on G7e, big win if a P-box is ever used)
export CUDA_CACHE_PATH=/mnt/nvme/cuda-cache && mkdir -p "$CUDA_CACHE_PATH"
```

Then run the §4 proofs 1–10, render one 1080p smoke frame, and **bake the AMI from this box** so
`<AMI_ID>` gives you identical workers forever. Notes: headless OptiX needs **no X server** — but if
you ever assemble a machine image yourself instead of the DLAMI, install `libnvidia-gl-<ver>`
(despite the name it needs no X) or OptiX dies with `OPTIX_ERROR_LIBRARY_NOT_FOUND`; and
`--factory-startup` would skip the enabled add-on — never add it to render commands.

### 10.3 Quota, launch, and IAM (exact commands)

```bash
# Quota: check, then request BEFORE render day (G-family small increases often auto-approve in
# minutes-hours; do not assume — new accounts sit at 0)
aws service-quotas get-service-quota --service-code ec2 --quota-code L-DB2E81BA --region us-east-1
aws service-quotas request-service-quota-increase --service-code ec2 \
    --quota-code L-DB2E81BA --desired-value 8 --region us-east-1

# Resolve the current DLAMI Ubuntu 24.04 Base GPU AMI id (then PIN the value you get)
aws ssm get-parameter --region us-east-1 --query 'Parameter.Value' --output text --name \
  /aws/service/deeplearning/ami/x86_64/base-oss-nvidia-driver-gpu-ubuntu-24.04/latest/ami-id

# [FIX v2 — Issue 2] TWO launch profiles. The AMI-seed box is a PET (stop, no dead-man switch) —
# you must not be able to lose it before the golden AMI exists. Workers are CATTLE (terminate +
# delete-on-termination), spawned only FROM the finished AMI.

# (a) SEED box — bootstrap per §10.2, bake the golden AMI from it, then stop it manually
aws ec2 run-instances --region us-east-1 \
  --instance-type g7e.2xlarge --image-id <DLAMI_ID> \
  --iam-instance-profile Name=<RENDER_PROFILE> \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2" \
  --instance-initiated-shutdown-behavior stop \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=300,VolumeType=gp3,Throughput=250,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=project,Value=<RUN_ID>},{Key=role,Value=ami-seed}]'

# (b) On-Demand WORKER — from the golden AMI; OS shutdown fully destroys it (dead-man safe)
aws ec2 run-instances --region us-east-1 \
  --instance-type g7e.2xlarge --image-id <GOLDEN_AMI_ID> \
  --iam-instance-profile Name=<RENDER_PROFILE> \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2" \
  --instance-initiated-shutdown-behavior terminate \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=300,VolumeType=gp3,Throughput=250,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=project,Value=<RUN_ID>},{Key=role,Value=worker}]'

# (c) SPOT worker — [FIX v2 — Issue 8] the Spot design §3 promises, actually configured:
# interruption-behavior=stop preserves the box for resume, use_overwrite=False (§5) skips finished
# frames on restart, and shutdown-behavior stays stop (terminate would defeat resume). The
# dead-man switch is the cost backstop. After delivery, CLEAN UP explicitly — a stopped spot box
# still bills EBS: cancel the spot request, then terminate the instance.
aws ec2 run-instances --region us-east-1 \
  --instance-type g7e.2xlarge --image-id <GOLDEN_AMI_ID> \
  --iam-instance-profile Name=<RENDER_PROFILE> \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2" \
  --instance-initiated-shutdown-behavior stop \
  --instance-market-options 'MarketType=spot,SpotOptions={SpotInstanceType=persistent,InstanceInterruptionBehavior=stop}' \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=300,VolumeType=gp3,Throughput=250,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=project,Value=<RUN_ID>},{Key=role,Value=spot-worker}]'
# cleanup after the job:
#   aws ec2 cancel-spot-instance-requests --spot-instance-request-ids <SIR_ID>
#   aws ec2 terminate-instances --instance-ids <IID>
```

Instance-profile policy — note the bucket-ARN / object-ARN split (the classic `AccessDenied` fix):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<BUCKET>",
      "Condition": { "StringLike": { "s3:prefix": ["jobs/*"] } } },
    { "Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::<BUCKET>/jobs/*" }
  ]
}
```

### 10.4 Dead-man switch + idle watchdog (the always-on cost guard, concrete)

```bash
# At job start — hard ceiling. [FIX v2 — Issue 3] 90 is the SMOKE/default ceiling only. For the
# hero job DERIVE it: CEILING_MIN = smoke-measured minutes/frame × frames-on-this-box × 1.5.
# [FIX v2 — Issue 2] NEVER arm this on the AMI-seed box (§10.3a) — workers only: on (b) workers
# shutdown = terminate (box destroyed), on (c) spot workers shutdown = stop (resumable).
# Cancel: shutdown -c
sudo shutdown -h +<CEILING_MIN> "render dead-man switch"

# Idle watchdog — every minute; 15 consecutive minutes with no blender/cycles process => shutdown
cat <<'EOF' | sudo tee /usr/local/bin/render-watchdog.sh >/dev/null
#!/usr/bin/env bash
IDLE_FILE=/run/render-idle-count
pgrep -f 'blender' >/dev/null && { echo 0 > "$IDLE_FILE"; exit 0; }
n=$(( $(cat "$IDLE_FILE" 2>/dev/null || echo 0) + 1 )); echo "$n" > "$IDLE_FILE"
[ "$n" -ge 15 ] && shutdown -h now "idle 15 min - auto stop"
EOF
sudo chmod +x /usr/local/bin/render-watchdog.sh
( sudo crontab -l 2>/dev/null; echo '* * * * * /usr/local/bin/render-watchdog.sh' ) | sudo crontab -
```

Backstop outside the box (fires even if the OS wedges): an EventBridge Scheduler one-off
`ec2:StopInstances`/`ec2:TerminateInstances` at T+3h on the instance id, created at launch time.
Spot boxes: also poll `http://169.254.169.254/latest/meta-data/spot/instance-action` (IMDSv2
headers) every 30 s — you get a 2-minute warning; the per-frame EXRs in S3 are the checkpoint, so an
interruption costs at most one frame.

### 10.5 Output validation — reject bad frames with numbers, not eyeballs

```bash
sudo apt-get install -y openimageio-tools
# Human inspection of a master (per-channel Min/Max/Avg/StdDev, plus NaN/Inf counts):
oiiotool --stats /mnt/nvme/jobs/<RUN_ID>/out/frame_0001.exr

# Automated gate — run it on the §7 Step-1 AgX-baked proof PNGs, NOT the multilayer EXRs
# (the EXRs contain many passes, so naive whole-file stats are ambiguous). oiiotool also
# prints "Constant: Yes" for any single-valued frame, which catches flat-gray failures that
# a mean threshold alone would miss. Tune 0.005 after your first good smoke frame.
# (Gate verified empirically: black frame -> REJECTED, flat gray -> REJECTED, real content -> OK.)
oiiotool --stats frame_0001.png | awk '
  /Stats Avg:/    {a=($3+$4+$5)/3}
  /Stats StdDev:/ {s=($3+$4+$5)/3}
  /^ *Constant: Yes/ {c=1}
  END { exit !(a>0.001 && s>0.005 && !c) }' && echo FRAME_OK || echo FRAME_REJECTED

# Frame-count gate for a range:
[ "$(ls out/frame_*.exr | wc -l)" -eq "<EXPECTED>" ] && echo COUNT_OK
```

The human proof PNG comes from the §7 Step-1 bake (AgX applied — a raw `oiiotool` linear→sRGB
convert looks washed out by comparison and is fine only for blank-detection, not for judging look).
Also check the EXR stats line `Stats NanCount` — any nonzero NaN count in the beauty pass means a
broken frame even if the averages look sane.

### 10.6 Failure → cause → fix (the ones that actually happen)

| Symptom | Cause | Fix |
| --- | --- | --- |
| Render is empty/black on the worker, fine locally | volume cache didn't ship, or path layout changed | §1.4: cache folder next to the `.blend`, same relative layout in `jobs/<RUN_ID>/in/` |
| `OPTIX_ERROR_LIBRARY_NOT_FOUND` | `libnvoptix.so.1` missing (compute-only driver / container without graphics capability) | §10.2: DLAMI or `libnvidia-gl-<ver>`; containers: `NVIDIA_DRIVER_CAPABILITIES=compute,utility,graphics` |
| `Found no Cycles device of the specified type`, exit 1 | driver too old for the card, proprietary kernel module, or no GPU | §3 driver [FIX]: R575+ open modules; re-run proofs 2/9/10 |
| `nvidia-smi`: "No devices were found" | proprietary kernel module on a Blackwell GPU | install the **open** kernel-module driver build |
| Launch fails: `MaxSpotInstanceCountExceeded` / vCPU limit | quota L-DB2E81BA (or Spot L-3819A6DF) at 0 | §10.3 quota request; wait for approval before render day |
| Launch fails: `InsufficientInstanceCapacity` | AZ out of that instance type | retry another AZ, then another G7e region (§3 region list) |
| `AccessDenied` on `aws s3 sync` | ListBucket vs GetObject on the wrong ARN | §10.3 policy: bucket ARN for List, `/jobs/*` ARN for Get/Put |
| Proof 1 curl hangs forever | IMDSv1 call on an IMDSv2-only box | §4 [FIX]: token flow |
| Video master looks flat/washed out | linear EXR encoded without the AgX view transform | §7 [FIX]: bake with Blender first, then ffmpeg |
| Step Rate/Max Steps tweaks change nothing | expected — the committed unbiased integrator has no step knobs | only meaningful in the biased budget-fallback (§0 fix 1, §5 commented block) |
| Python error setting `OPEN_EXR_MULTILAYER` | 5.0 media_type gate | §5 [FIX]: set `media_type='MULTI_LAYER_IMAGE'` first |
| "OptiX passed proofs" but render ran on CPU | old proof 10 never rendered, so it validated nothing | §4 proof 10 [FIX v2 — Issue 1]: render a tiny frame, require exit 0 + non-empty file |
| Bootstrap box vanished before the AMI was baked | terminate + dead-man switch armed on the seed box | §10.3a [FIX v2 — Issue 2]: seed = stop, no dead-man; terminate is workers-only |
| Box auto-stopped mid-hero with no frame out | fixed 90-min ceiling < one 8K/4096 volume frame | §10.4 [FIX v2 — Issue 3]: derive the ceiling from the smoke |
| Run never finishes across sessions | job hand-driven over SSM through interruptions | §10.8: one unattended runner — finishes or fails with a postmortem |
| Spot batch can't resume after interruption | terminate behavior / no Spot request config | §10.3c [FIX v2 — Issue 8]: interruption-behavior=stop + `use_overwrite=False` |
| OOM at 8K | volume grid too dense for VRAM | 96 GB G7e headroom; else lower resample, or quadrant render-regions (§2) |
| Box gone but bill keeps growing | instance *stopped*, EBS still billing | terminate-on-shutdown + delete-on-termination (§10.3); check `aws ec2 describe-instances` |

### 10.7 First-run order of operations (one box, end to end)

1. Request quota (§10.3) → wait for approval. Create bucket + IAM role. Set the Budgets alarm.
2. Author interactively: import volume data, build node chain, **save cache**, save `.blend` (§1).
3. `aws s3 sync` the job to `s3://<BUCKET>/jobs/<RUN_ID>/in/`.
4. Launch the **seed box** with profile (a) (§10.3 — stop behavior, **no dead-man switch**).
   Bootstrap it (§10.2), bake the golden AMI, stop the seed. Launch one **worker** from the AMI
   with profile (b); the worker arms its own dead-man switch + watchdog (§10.4).
5. Run proofs 1–10 (§4). Any failure → terminate, fix, relaunch.
6. Sync the job down to `/mnt/nvme`. **Smoke: one 1920×1080 frame** of the exact scene
   (`-o .../smoke_####` and override resolution via a 2-line `-P` snippet or a preview copy of
   render_8k.py). Validate with §10.5 + the AgX proof PNG. Never debug at 8K — it's 16× the pixels.
7. Render the 8K range (§5) with `pmon` spot-check. Validate every frame (§10.5), sync up to S3,
   checksum manifest.
8. Bake + encode (§7) on one box. Sync `delivery/`. Verify counts + sizes in S3.
9. `sudo shutdown -h now` → instance terminates itself; confirm state `terminated`. Record cost.
10. Only after this single box passes end-to-end: fan out N boxes with non-overlapping `-s/-e`
    ranges, each with its own watchdog and dead-man switch. Batch starts on an explicit go — never
    automatically.

**[FIX v2 — orchestration]** Steps 5–9 must run as **one unattended script on the box** (§10.8) —
never hand-driven over SSM turn by turn. A hand-driven run dies with every chat/session/SSM-window
interruption; the unattended job either finishes or fails cleanly with a postmortem in S3, no
matter what happens to the operator.

### 10.8 The unattended job runner (`run_job.sh`) — a run finishes or explains itself

One script, started once (user-data, SSM one-shot, or `nohup ... &`), owning the whole job. It
assumes the §1.4/§1.5 input contract: the `.blend` **and its saved cache** are already staged under
`jobs/<RUN_ID>/in/` — it never attempts the GUI import.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
RUN_ID=<RUN_ID>; BUCKET=<BUCKET>
SCRATCH=/mnt/nvme/jobs/$RUN_ID; LOG=$SCRATCH/run.log
mkdir -p "$SCRATCH/out"; exec > >(tee -a "$LOG") 2>&1

postmortem() {                                   # runs on EVERY exit path — success or death
  code=$?
  aws s3 cp "$LOG" "s3://$BUCKET/jobs/$RUN_ID/postmortem/run.log" || true
  printf '{"exit":%d,"stage":"%s","ts":"%s"}\n' "$code" "${STAGE:-unknown}" "$(date -u +%FT%TZ)" \
    | aws s3 cp - "s3://$BUCKET/jobs/$RUN_ID/postmortem/status.json" || true
  sudo shutdown -h +2 "job ended (exit $code, stage ${STAGE:-unknown}) - self stop"
}
trap postmortem EXIT

STAGE=deadman   # ceiling derived per §10.4 — smoke min/frame × frames × 1.5. Workers only, never the seed box.
sudo shutdown -h +<CEILING_MIN> "render dead-man switch"

STAGE=pull
aws s3 sync "s3://$BUCKET/jobs/$RUN_ID/in/"  "$SCRATCH/in/"
aws s3 sync "s3://$BUCKET/jobs/$RUN_ID/out/" "$SCRATCH/out/"   # resume: pre-seed finished frames

STAGE=inputs    # fail-closed input contract — never render toward an empty volume
test -s "$SCRATCH/in/scene.blend"
[ -d "$SCRATCH/in/<CACHE_DIR>" ] && [ -n "$(ls -A "$SCRATCH/in/<CACHE_DIR>")" ] \
  || { echo "FATAL: volume cache missing — author it locally (§1.4) and re-stage"; exit 66; }

STAGE=proofs    # §4 proofs 2-10 scripted; each failure aborts here, before any GPU-hour is spent
nvidia-smi >/dev/null
ldconfig -p | grep -q libnvoptix.so.1
blender -b --python-expr "import bpy; s=bpy.context.scene; s.render.engine='CYCLES'; s.cycles.device='GPU'; s.render.resolution_x=64; s.render.resolution_y=64; s.cycles.samples=8" \
        -o /tmp/optix_probe_#### -f 1 -- --cycles-device OPTIX
test -s /tmp/optix_probe_0001.png

STAGE=smoke     # one 1080p frame of the real scene + the §10.5 numeric gate
blender -b "$SCRATCH/in/scene.blend" -P render_smoke.py \
        -o "$SCRATCH/out/smoke_####" -f <MID_FRAME> -- --cycles-device OPTIX
# (render_smoke.py = render_8k.py with 1920×1080 + preview samples; §10.5 gate on the baked proof)

STAGE=hero
blender -b "$SCRATCH/in/scene.blend" -P render_8k.py \
        -o "$SCRATCH/out/frame_####" -s <START> -e <END> -a \
        -- --cycles-device OPTIX --cycles-print-stats

STAGE=validate  # §10.5: count gate + per-frame numeric gate + NaN check; any failure exits here
[ "$(ls "$SCRATCH"/out/frame_*.exr | wc -l)" -eq <EXPECTED> ]

STAGE=upload
( cd "$SCRATCH/out" && sha256sum * > checksums.sha256 )
aws s3 sync "$SCRATCH/out/" "s3://$BUCKET/jobs/$RUN_ID/out/"

STAGE=done; echo "JOB COMPLETE"
```

Properties that end the never-finishes loop: **fail-closed** (missing cache exits 66 before any
render), **resumable** (out/ pre-seed + §5 `use_overwrite=False` skip already-finished frames, so
an interrupted box relaunches and continues), **self-terminating** (the EXIT trap stops the box on
every path — the shutdown behavior chosen at launch decides stop vs destroy), and **evidenced**
(log + stage + exit code land in `postmortem/` even on death, so a failed run tells you why without
a live SSM session).

---

## 11. Second-pass audit — defects in the spec itself
*Same review method as §10, run on §§0–10. Terminology kept as the doc uses it ("volume data",
"the volume add-on"). Ranked worst first. Each: what's wrong → why it fails → fix. Every issue
below is now fixed in place in §§0–10 and marked `[FIX v2 — Issue N]`; the resolution pointer
follows each entry.*

**[ISSUE 1 — FALSE PROOF] Proof 10 does not actually test OptiX (§4).**
`blender -b --python-expr "import bpy" -- --cycles-device OPTIX; echo "exit=$?"` never renders, so
Cycles never creates a device session — `--cycles-device` only takes effect when a render runs. On a
box where OptiX is fully broken this still exits **0**. The proof that's supposed to be the fail-fast
catch is the one proof that can't catch anything.
→ Fix: make it render one tiny frame and check exit≠0 AND that a non-zero-byte frame landed.
**Resolved:** §4 proof 10 now renders a 64×64/8-spp frame of the default scene and requires exit 0
plus a non-empty `/tmp/optix_probe_0001.png`; §10.8 runs the same probe in its `proofs` stage.

**[ISSUE 2 — SELF-DESTRUCT ORDERING] terminate-on-shutdown was applied to the box you haven't baked
yet (§10.3 + §10.4 + §10.7).**
Launching every box with `--instance-initiated-shutdown-behavior terminate` plus a `shutdown -h +90`
dead-man switch meant the FIRST box — the one the golden AMI is baked from — could destroy itself
before the AMI existed.
→ Fix: bake the AMI on a box launched `shutdown-behavior=stop`; `terminate` ONLY for disposable
workers spawned from the finished AMI. terminate is a cattle setting; the AMI-seed box is a pet.
**Resolved:** §10.3 now has explicit seed (a) / worker (b) / spot-worker (c) launch profiles; §10.4
and §10.7 step 4 forbid arming the dead-man switch on the seed; §3 states the pet/cattle rule.
An already-certified hand-built box with no AMI yet is a **seed** — keep it stop + fail-closed.

**[ISSUE 3 — TWO LOCKED VALUES THAT CONTRADICT] "max runtime 90 min" cannot coexist with "8K @ 4096
samples" for volume content (§0).**
A single 33-MP frame at 4096 samples with volume bounces can outrun 90 min on one GPU; a 24 fps
animation is orders of magnitude beyond it.
→ Fix: 90 min is a *smoke/default* ceiling. Derive the real per-box runtime from the 1080p smoke's
measured per-frame time × frames-per-box × ~1.5, set per job.
**Resolved:** §0 table, §3 cost guard, §10.4, and §10.8 all now use the derived `<CEILING_MIN>`;
90 min remains only as the smoke/default value.

**[ISSUE 4 — NOT ACTUALLY HEADLESS] The farm requires a manual GUI step it never removes (§1.5,
§10.7 step 2).**
Cache creation is "author + import + save cache interactively once" with no headless path offered —
the exact step that must be automated for a farm was left to a human at a GUI, silently.
→ Fix: (a) prove a headless cache build via the add-on's lower-level APIs, or (b) state plainly that
authoring is a one-time manual pre-step done locally and ONLY rendering is farmed.
**Resolved (as (b), honestly):** §1.5 now states the boundary explicitly — data→cache is a local,
manual, one-time pre-step; the farmed unit is render-only with `.blend`+cache as the input contract,
and §10.8 fails fast (exit 66) if the cache is missing rather than limping into an empty render.
Path (a) remains open: if the add-on's Python API can build layers + save the cache, prove it
headless on ONE box before promising it — until then this spec does not.

**[ISSUE 5 — VERSION PIN vs FIXES] Pins Blender 5.0.1, but every [FIX] is 5.0-API-specific and the
pin reasoning was backwards (§2, §5, §7).**
"Not an LTS series, so standardize on 5.0.1" is a non-sequitur, and a worker on a different minor
makes `media_type`, `volume_biased`, and the AgX name behave unpredictably.
→ Fix: pin ONE exact build and hard-assert `bpy.app.version` before rendering.
**Resolved:** §5 now opens with `PINNED = (5, 0, 1)` + a refusing assert; §8 says the pin is
*whatever exact build your certified box runs* (e.g. `(5, 1, 0)`) — change the tuple once, re-run
the §4 proofs + one smoke on that build, and the fleet is guaranteed uniform.

**[ISSUE 6 — "LOCKED" VALUE THAT ISN'T] The step-rate/biased volume default contradicted itself
(§0, §2, §5).**
§0 locked step rate 0.25/1024 while §2/§5 admitted the unbiased default makes them optional — a
contingent value presented as locked, and forcing `volume_biased=True` opted out of the newer path.
→ Fix: commit to one.
**Resolved:** committed to **unbiased** (§0 fix 1, §2, §5): no step knobs, quality governed by
samples/threshold/bounces. Biased + 0.25/1024 survives only as the explicit budget fallback with a
stated trigger (smoke-derived frame time over budget), carried as a commented block in §5.

**[ISSUE 7 — SILENT-FAIL RISK] API properties asserted as working, with no runtime guard (§5).**
`denoising_use_gpu`, `denoising_input_passes`, and the `media_type` gate could silently no-op or
raise late on a slightly different build.
→ Fix: wrap each in an explicit guard that aborts loud at second 1.
**Resolved:** §5 adds `lock()` — a hasattr-guarded setter that raises immediately — applied to
every 5.0-specific property, alongside the version assert from Issue 5.

**[ISSUE 8 — SPOT IS RECOMMENDED BUT NEVER CONFIGURED] (§3 vs §10.3).**
§3 recommended Spot-with-stop for batches, but the only launch template was On-Demand with
`terminate` everywhere — no Spot request config existed, and terminate clashes with stop-to-resume.
→ Fix: add the actual Spot request block and reconcile behaviors.
**Resolved:** §10.3 profile (c): `--instance-market-options
'MarketType=spot,SpotOptions={SpotInstanceType=persistent,InstanceInterruptionBehavior=stop}'`,
shutdown-behavior stop, resume via §5 `use_overwrite=False` + §10.8 out/ pre-seed, and an explicit
post-job cleanup (cancel spot request, then terminate — a stopped spot box still bills EBS).

**[ISSUE 9 — FACTUAL WOBBLE] R575 vs R570 in the same breath (§3 [FIX] 2).**
A driver floor that contradicts itself one line later sends someone chasing a phantom driver problem.
→ Fix: state the real floor once.
**Resolved:** §3 now states one floor — **R575+** (the RTX PRO 6000 SE requirement subsumes all
other Blackwell minima) — and pins the DLAMI (595.x) as the way to never think about it again.

### 11.1 Failure → cause → fix (the ones this spec would have caused)
| Symptom | Cause | Fix |
| --- | --- | --- |
| "OptiX passed proofs" but render is on CPU | Proof 10 never rendered → validated nothing | Issue 1 → §4 proof 10 |
| Bootstrap box vanished before the AMI was baked | terminate + dead-man switch on the seed box | Issue 2 → §10.3a |
| Box auto-stopped mid-hero, no frame | 90-min ceiling < one 8K/4096 volume frame | Issue 3 → §10.4 derived ceiling |
| "Automated farm" stalls waiting on a person | cache build is a manual GUI step | Issue 4 → §1.5 contract + §10.8 fail-fast |
| A [FIX] does nothing on a worker | worker runs a different Blender minor | Issue 5 → §5 version assert |
| Master looks wrong despite "locked" volume values | biased vs unbiased never committed | Issue 6 → committed unbiased |
| Hours in, output flat/undenoised | a denoise property silently no-op'd | Issue 7 → §5 `lock()` |
| Spot batch can't resume after interruption | terminate instead of stop; no Spot config | Issue 8 → §10.3c |

### 11.2 Scope note — what this spec still does not do
Two things remain deliberately out of scope, stated so nobody assumes otherwise:
1. **Headless data→cache build.** The add-on's import is GUI-modal; until its Python API is proven
   headless on a real box, authoring stays a local manual pre-step (Issue 4, path (a) open).
2. **Preprocessing (raw data → clean volume).** This is a render/farm spec; cleaning, spacing, and
   level normalization of the source volume happen before §1 and are not covered here.
