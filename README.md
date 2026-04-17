# ComfyUI-PoseTracks

Audio-reactive and motion-capture-driven pose generation for LTX IC-LoRA Union Control video generation.

## What it does

Generates 3D skeleton pose renders without requiring input video tracking. Rendered frames feed directly into `LTXAddVideoICLoRAGuide` as pose control for the LTX-2.3 IC-LoRA Union Control model.

Motion sources:

1. **Beat detection** — Poses hit on musical beats
2. **Multi-Character Sync** — Orchestrate groups with Unison, Mirror, or Random interactions
3. **Physics Simulation** — Momentum, drag, and "Sticky Feet" constraints for grounded motion
4. **Auto-Rigging** — Automatically corrects "non-neutral" reference poses (e.g., character starting with foot up)
5. **Dynamic Scaling** — Adjusts movement amplitude based on character size/distance to prevent collisions
6. **AIST++ Dance Library** — Real motion capture dance data from professional dancers across 10 genres
7. **CMU Motion Library** — 2500+ general motion capture sequences (walk, run, jump, sports, interactions, and more)

## Basic workflow

```
[PTBeatDetector] ──► beat_info ──►
[PTAudioFeatureExtractor] ──► audio_features ──► [PTBeatDrivenPose] ──► pose_sequence ──► [PTPoseRenderer] ──► images ──► [LTXAddVideoICLoRAGuide]
[PTPoseFromDWPose] ──► base_pose ──►
```

Or with motion capture:

```
[PTCMULibraryLoader] ──► [PTCMUMotion] ──► pose_sequence ──► [PTPoseRenderer] ──► images ──► [LTXAddVideoICLoRAGuide]
```

Or with FBX / Mixamo:

```
[PTMixamoFBX] ──► pose_sequence ──► [PTPoseRenderer] ──► images ──► [LTXAddVideoICLoRAGuide]
```

---

## Nodes

### Audio Analysis

- **PTAudioFeatureExtractor** — Extracts per-frame RMS, bass, mid, treble, and onset features.
- **PTBeatDetector** — Detects beats, downbeats, and tempo from audio using librosa.

### Base Pose Generation

- **PTBasePoseGenerator** — Procedurally create 1 to 5 skeletons (T-pose, Active Idle) with adjustable spacing.
- **PTPoseFromDWPose** — Extract base poses from reference images. Supports **1:1 fidelity** for multiple characters (detects everyone in the image, no cloning).

### Animation

- **PTBeatDrivenPose** — The Choreographer. Generates physics-based dance sequences.
  - Supports **Interaction Modes** (Mirror, Unison, Random).
  - Handles **Anti-Jelly Bone** constraints to keep limbs rigid.
  - Includes **80+ Dance Poses** (Hip Hop, Rock, Pop, etc.)
- **PTAlignPoseToReference** — Align generated pose sequence to match reference position/scale.

### AIST++ Dance Library

Real motion capture data from the AIST++ dataset — professional dancers performing choreographed routines across 10 genres. Auto-downloads from HuggingFace on first use.

**Chunk-Based (Reactive):**
- **PTAISTLibraryLoader** — Loads the chunked dance library (~220MB, auto-downloads).
- **PTAISTBeatDance** — Beat-triggered chunk selection based on audio energy.
  - Mix multiple genres (break, pop, lock, etc.)
  - Energy-aware: low/mid/high energy chunks matched to audio
  - Chunks chain for longer sequences
  - **Note:** Can be janky at transitions since chunks are cut at velocity minimums, not perfect phrase boundaries. Good for variety and reactivity, less smooth than full sequences.
- **PTAISTChunkPreview** — Preview individual chunks by genre/energy.

**Full Sequence (Smooth):**
- **PTAISTFullLoader** — Loads full dance sequences (~200MB, auto-downloads).
- **PTAISTFullSequence** — Plays complete choreographed dances.
  - Single genre dropdown (no mixing)
  - Smooth, continuous motion from real performances
  - **Note:** Can feel repetitive if you recognize the source choreography. Best for shorter generations or when you want authentic, uncut dance motion.

**Available Genres:** break, pop, lock, waack, krump, house, street_jazz, ballet_jazz, la_hip_hop, middle_hip_hop

### CMU Motion Capture Library

General motion capture from Carnegie Mellon University — 2548 motions across 16 categories including walking, running, jumping, dancing, sports, climbing, and two-person interactions. Auto-downloads from HuggingFace on first use (~660MB).

- **PTCMULibraryLoader** — Loads the CMU motion library.
  - Source: HuggingFace or local path
  - Auto-downloads on first use
- **PTCMUMotion** — Plays a single motion sequence. Chain multiple for choreographed sequences.
  - Category selection (walk, run, jump, dance, fight, sports, etc.)
  - Random or specific motion selection
  - Loop modes: none, loop, ping_pong
  - Proper locomotion drift (character moves across frame)
  - Chainable: connect `pose_sequence_out` → `pose_sequence_in` for multi-motion sequences
  - Frame count auto-calculated from motion length

**Available Categories:** walk (955), run (114), jump (165), dance (150), fight (70), sports (114), climb (31), interact (47), sit (32), crouch (7), gesture (17), swim (24), getup (18), fall (3), balance (2), misc (799)

**Two-Person Motions:** 55 paired interactions (handshakes, partner dances, etc.) with matching motion files.

### FBX / Mixamo

Load any FBX animation file and play it as a pose sequence. Designed for Mixamo exports but works with any FBX using standard humanoid bone naming. Requires Blender installed — auto-detected on Windows, Linux, and macOS.

- **PTMixamoFBX** — Loads an FBX file and converts it to a pose sequence.
  - Place FBX files in `ComfyUI/input/fbx_animations/`
  - Auto-detects Blender installations (newest version selected by default)
  - Cascading scale detection: full height → torso → shoulder width → fallback
  - Proper Y-axis flip (Blender Z-up → screen Y-down)
  - Floor anchor keeps character grounded
  - Chainable: connect `pose_sequence_out` → `pose_sequence_in`
  - Reference pose for positioning and scale matching

**Requirements:** [Blender](https://www.blender.org/download/) installed on the machine running ComfyUI.

### Rendering

- **PTPoseRenderer** — Renders pose sequence to 3D cylinder skeleton images. Output feeds `LTXAddVideoICLoRAGuide`.
- **PTAISTChunkPreview** — Visualize AIST++ chunks by genre/energy.

---

## Installation

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/ckinpdx/ComfyUI-PoseTracks
pip install -r ComfyUI-PoseTracks/requirements.txt
```

Motion libraries download automatically on first use:
- AIST++ chunks: ~220MB
- AIST++ full sequences: ~200MB
- CMU motions: ~660MB

---

## LTX compatibility notes

- **`PTPoseRenderer` output → `LTXAddVideoICLoRAGuide.image`** — wire directly, no conversion needed
- Use `LTXICLoRALoaderModelOnly` to load `ltx-2.3-22b-ic-lora-union-control-ref0.5.safetensors` and extract `latent_downscale_factor` (2.0)
- Pass `latent_downscale_factor` from the loader into `LTXAddVideoICLoRAGuide`
- Frame count must satisfy `8n+1` for LTX: 9, 17, 25 … 97, 105 … Use `LTXFrameCalculator` from [ComfyUI-LTXAVTools](https://github.com/ckinpdx/ComfyUI-LTXAVTools) to snap automatically
- Set `PTPoseRenderer` width/height to match your generation resolution

---

## Parameters

### PTBasePoseGenerator
- `character_count` — Generate 1 to 5 procedural skeletons side-by-side.
- `spacing` — Distance between characters in world units.

### PTBeatDrivenPose

**Interaction & Style:**
- `dance_style` — auto / hip_hop / rock / disco / etc. `auto` selects moves based on audio energy.
- `interaction_mode` — unison / mirror / random.
- `energy_style` — auto / low / medium / high.
- `motion_smoothness` — Higher = floatier/smoother. Lower = snappier/robotic.
- `anticipation` — Frames to start moving *before* the beat hits.

**Audio Modulation:**
- `groove_amount` — Intensity of the continuous hip sway/figure-8 loop.
- `bass_intensity` — Bass → vertical bounce (scaled to body size).
- `treble_intensity` — Treble → arm/hand jitter.

### PTAISTBeatDance (Chunk Mode)
- `sync_dancers` — All dancers do same moves (true) or independent (false).
- `genre_*` toggles — Enable/disable each of the 10 dance genres.
- `transition_frames` — Frames to blend between chunks.
- `chunks_per_beat` — Chain consecutive chunks (1=0.5s, 2=1s, etc.).
- `energy_sensitivity` — How strongly audio energy affects chunk selection.

### PTAISTFullSequence (Full Sequence Mode)
- `genre` — Single genre dropdown.
- `transition_frames` — Frames to blend from reference pose to dance start.
- `start_beat` — Which beat in the source to start from (0=beginning).

### PTCMUMotion
- `category` — Motion category (walk, run, jump, dance, fight, sports, etc.).
- `selection_mode` — Random from category or specific motion ID.
- `motion_id` — Specific motion ID when using "specific" mode (e.g., "02_01").
- `transition_frames` — Frames to blend from previous pose/sequence.
- `loop_mode` — none / loop / ping_pong.
- `target_fps` — Output frame rate (source is 120fps, scaled to target).
- `pose_ref` — Optional reference pose for positioning/scaling.
- `pose_sequence_in` — Optional input from previous motion node for chaining.

**Outputs:**
- `pose_sequence` — Connect to next CMU Motion node or directly to `PTPoseRenderer`.
- `frame_count_out` — Total frames (useful for matching LTX frame count).
- `motion_info` — Description of selected motion.

### PTPoseFromDWPose
- **Note:** Extracts what is detected in your reference image only — no character cloning. To animate 3 people, use a reference image with 3 people.

---

## Multi-Character Tips

If using a reference image with multiple people, ensure your upstream DWPose/OpenPose detector finds them all.

1. **Resolution:** Set detector resolution to **1024** or higher for group shots.
2. **Model:** Use `dw-ll_ucoco.onnx` (not the 384 variant) for the pose model, and `yolo_nas_l_fp16.onnx` or `yolox_x.onnx` for BBox detection. Standard `yolox_l` often misses people in complex poses.
3. **Max People:** Ensure upstream node allows `max_people > 1` if using OpenPose.

---

## Motion Library Comparison

| Feature | Procedural | AIST++ Chunks | AIST++ Full | CMU Motion | FBX / Mixamo |
|---------|------------|---------------|-------------|------------|--------------|
| Motion Quality | Synthetic | Real mocap | Real mocap | Real mocap | Real mocap |
| Variety | 80+ poses | ~44k chunks | ~1400 sequences | 2548 motions | Unlimited |
| Categories | Dance only | 10 dance genres | 10 dance genres | 16 categories | Any |
| Audio Reactivity | High | Medium | Low | None | None |
| Transitions | Smooth | Can be janky | Smooth | Smooth | Smooth |
| Locomotion | Stationary | Stationary | Stationary | **Full drift** | **Full drift** |
| Two-Person | No | No | No | **55 pairs** | No |
| Requires | — | Auto-download | Auto-download | Auto-download | Blender + FBX file |
| Best For | Rhythmic loops | Reactive dance | Cinematic dance | Walking, actions, interactions | Custom animations, Mixamo library |

---

## Technical Details

### "Sticky Feet" Physics
The `MotionDynamics` engine uses variable drag coefficients. Ankles have 0 drag and 2x stiffness, forcing them to snap to the floor position unless a specific dance move lifts them. Prevents the floating effect common in procedural animation.

### Neutral Structure Initialization (Auto-Rigging)
If your reference image has a character mid-stride, the system calculates a mathematical "Neutral Standing" skeleton from their bone lengths. The physics engine interpolates Reference → Neutral → Dance Move, preventing the character from getting stuck in their initial pose.

### Scale-Aware Movement
Movements are normalized by the character's torso length. A small background character will perform smaller absolute movements than a foreground character, maintaining correct perspective and preventing collisions.

### CMU Locomotion Drift
Unlike AIST++ which keeps dancers stationary, CMU motions preserve the original movement trajectory. A walking motion moves the character across the frame. Drift accumulates through chained sequences.

### AIST++ Data Processing
AIST++ dance data is converted from COCO 17-keypoint format to OpenPose 18-keypoint format. Chunks are cut at velocity minimums rather than fixed intervals to preserve move integrity. Energy tagging uses velocity + acceleration metrics, calculated per-genre.

### CMU Data Processing
CMU BVH files are converted to OpenPose 18-keypoint NPY format at 120fps. The first 5 frames (T-pose calibration) are automatically skipped. Joint mapping preserves left/right orientation for correct rendering.

---

## Credits

- Original SCAIL system: [zai-org/SCAIL](https://github.com/zai-org/SCAIL)
- Taichi renderer from [zai-org/SCAIL-Pose](https://github.com/zai-org/SCAIL-Pose)
- Beat detection: [librosa](https://librosa.org/)
- Expanded Pose Library: Discord user **NebSH**

---

## Legal & Licensing

This software downloads and processes third-party datasets. Users are responsible for complying with attribution requirements when publishing generated content.

### CMU Motion Capture Database

- **Source:** [mocap.cs.cmu.edu](http://mocap.cs.cmu.edu)
- **License:** Free for use (including commercial), requires citation.
- **Required Citation:**
  > "The data used in this project was obtained from mocap.cs.cmu.edu. The database was created with funding from NSF EIA-0196217."

### AIST++ Dataset

- **Source:** [Google Research AIST++](https://google.github.io/aistplusplus_dataset/)
- **License:** CC-BY 4.0 International — commercial use and redistribution allowed with attribution.
- **Required Attribution:** Credit Li et al., ICCV 2021 in derived works.

### Disclaimer

The authors of PoseTracks are not responsible for end-user license violations. This tool is provided "AS IS" without warranty of any kind.

---

## License

MIT
