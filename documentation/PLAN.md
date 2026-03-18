# Meshborn Development Plan

## Context

Meshborn is a Kokokino spoke app that generates TalkingHead-ready 3D avatars from text prompts or images. The output is a GLB file with split lips, teeth, tongue, eyes, 52 ARKit blend shapes, 15 Oculus visemes, a Mixamo-compatible rig, and animations. The spoke skeleton (SSO, subscriptions, chat) is already built. This plan covers the avatar pipeline.

The key insight from the brainstorming phase: **MPFB2 (MakeHuman Plugin For Blender 2)** eliminates the hardest problems. Mika Suominen (met4citizen), who created TalkingHead, also contributed the exact MPFB2 asset packs needed for TalkingHead compatibility. The blend shape problem is already solved.

---

## Key Architectural Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Template system | **MPFB2** from makehumancommunity.org | Parametric humanoids with ARKit + viseme shape keys built in. Eliminates need for manual templates, ICT FaceKit, deformation transfer |
| 3D generation (post-MVP) | **Trellis v2** (MIT license) | Cleaner topology, more permissive license than Hunyuan3D |
| Image generation | **HiDream-I1** (MIT license) | Best prompt adherence, fully commercial-friendly |
| Blender version | **Blender 5 always** | Runs in cloud containers only. No local Blender |
| Container runtime | **Podman** | Rootless, daemonless, drop-in Docker replacement |
| Container registry | **Quay.io** | Red Hat's OCI registry, pairs naturally with Podman |
| GPU compute | **RunPod Serverless** | Pay-per-second, scales to zero, shared by dev and prod |
| File storage | **Backblaze B2** | S3-compatible, follows backlog-beacon pattern with AWS SDK |
| 3D viewer | **Babylon.js v8** | Kokokino standard. GLB morph target support built in |
| LLM layer | **OpenRouter** | Prompt decomposition, HiDream prompt optimization, refinement parsing. Cheap models (GPT-4o-mini), no cold start, vision models for post-MVP. Follows false-colors (a spoke app) pattern |
| Dev philosophy | **Cloud services everywhere** | Same RunPod endpoints for dev and prod. No crippled local models |

---

## System Architecture

```
                YOUR DEV MACHINE
                Code + Meteor local dev + Git
                NO local Blender, NO local ML models
                        │
          ┌─────────────┴──────────────┐
          │ deploy                     │ podman push
          ▼                            ▼
  ┌────────────────────┐    ┌──────────────────────────────┐
  │  METEOR GALAXY     │    │  RUNPOD SERVERLESS           │
  │  meshborn.koko...  │◄──►│  (shared by dev + prod)      │
  │                    │    │                              │
  │  Mithril.js UI     │    │  EP1: MPFB2 + Blender 5     │
  │  Babylon.js viewer │    │   Character generation       │
  │  Pipeline orch.    │    │                              │
  │  MongoDB           │    │  EP2: HiDream-I1             │
  │  SSO via Hub       │    │   Concept image generation   │
  │                    │    │                              │
  └────────┬───────────┘    │  EP3: Trellis v2 (post-MVP)  │
           │                │   Image → 3D mesh            │
           ▼                │                              │
  ┌────────────────────┐    │  EP4: UniRig (post-MVP)      │
  │  BACKBLAZE B2      │    │   Auto-rigging               │
  │  Avatar storage    │    └──────────────────────────────┘
  │  Intermediate files│
  │  Params for remix  │
  └────────────────────┘
```

Both dev (localhost:3050) and production (meshborn.kokokino.com) call the same RunPod endpoints and write to the same B2 bucket (with environment-prefixed paths).

---

## The MPFB2 Template Pipeline (MVP)

This is the core pipeline. MPFB2 + Mika Suominen's asset packs handle the entire chain from parametric humanoid to TalkingHead-ready GLB.

### Pipeline Stages

```
1. USER INPUT
   Text prompt or image
        │
        ▼
2. PROMPT DECOMPOSITION (Meteor server, Claude API)
   Parse into MPFB2 parameters + appearance description
   Output: { bodyParams, faceParams, ethnicity, age, style, ... }
        │
        ▼
3. HERO IMAGE (RunPod: HiDream-I1)
   Generate full-body character concept on white bg
   Generate face close-up for texture detail
   User approves or requests changes
        │
        ▼
4. CHARACTER GENERATION (RunPod: MPFB2 + Blender 5)
   a. MPFB2 generates parametric humanoid via HumanService
      - Body shape, face shape, ethnicity, age via API
   b. MPFB2 adds system assets via AssetService
      - Eyes, teeth, tongue, eyelashes, eyebrows (separate mesh objects)
   c. MPFB2 adds Mixamo rig via RigService
   d. MPFB2 loads Faceunits 01
      - 52 ARKit blend shape keys on basemesh
   e. MPFB2 loads Visemes 02 (Meta/Oculus)
      - 15 Oculus viseme shape keys: viseme_aa, viseme_CH, viseme_sil, etc.
   f. MPFB2 selects hair from asset library
      - Based on prompt decomposition output
   g. MPFB2 selects clothing from asset library
      - Based on prompt decomposition output
   h. Save params.json for future remixing
        │
        ▼
5. TEXTURE PROJECTION (RunPod: Blender 5)
   Project HiDream face close-up onto MPFB2 mesh UV
   Project body appearance onto body UV islands
   Bake to texture map
        │
        ▼
6. EXPORT (RunPod: Blender 5)
   Export as GLB with all morph targets
   Generate thumbnail (256x256)
   Upload GLB + assets to Backblaze B2
        │
        ▼
7. DELIVERY
   Babylon.js loads GLB in browser
   User can preview, download, or use in TalkingHead
```

### MPFB2 Asset Packs

Install **all available MakeHuman/MPFB2 asset packs** in the container from the start. More variety from day one; the container image will be larger but this avoids repeated rebuilds as we discover needs.

**Critical packs (TalkingHead compatibility)**:

| Pack | Creator | License | Purpose |
|------|---------|---------|---------|
| **Faceunits 01 (ARKit)** | Mika Suominen (met4citizen) | CC0 | 52 ARKit facial expression shape keys |
| **Visemes 02 (Meta/Oculus)** | Mika Suominen (met4citizen) | CC0 | 15 Oculus viseme shape keys for lip-sync |
| **Visemes 01 (Microsoft)** | Mika Suominen (met4citizen) | CC0 | Microsoft visemes for broader compatibility |
| System assets (eyes, teeth, tongue, eyelashes) | MPFB2 core | CC0 | Separate mesh objects |

**Additional packs (all available)**: clothing, hair, body morphs, skins, targets, etc. These expand character variety and give users more customization options. Bake all packs into the Blender container image so they're available without runtime downloads.

### MPFB2 API Reference

MPFB2 uses a service-oriented architecture with stateless singletons. Key services for our pipeline:

```python
from mpfb.services.humanservice import HumanService    # Create base human, apply targets
from mpfb.services.rigservice import RigService        # Add Mixamo rig, weight painting
from mpfb.services.assetservice import AssetService    # Load eyes, teeth, clothing, hair
```

Compatible with Blender 4.2 through 5.1. Version 2.0.14+ recommended.

**Example MPFB2 target paths** (used in params.json and prompt decomposition):
```json
{
  "macrodetails/Gender": 0.1,
  "macrodetails/Age": 0.4,
  "macrodetails/Weight": 0.3,
  "macrodetails/Muscle": 0.7,
  "macrodetails/Proportions": 0.5,
  "head/head-square": 0.6,
  "jaw/jaw-width-max": 0.7,
  "eyes/eyes-size-small": 0.6,
  "forehead/forehead-height-min": 0.3
}
```

These target names map to shape keys on the basemesh. The prompt decomposer must build a dictionary between natural language descriptors and MPFB2's target system.

### Why This Works

The previous brainstorming docs treated blend shape creation as "the hardest unsolved problem." MPFB2 makes it a configuration step:

- **No ICT FaceKit** needed (MPFB2 has its own shape key system)
- **No deformation transfer** needed (shape keys are applied directly to the basemesh)
- **No manual template creation** (MPFB2 generates parametric humanoids procedurally)
- **No separate rigging step** (MPFB2 adds Mixamo rig natively)
- **Separate mesh objects** come free (eyes, teeth, tongue are separate by design in MPFB2)

---

## AI Mesh Path (Post-MVP — Trellis v2)

Deferred to after the MPFB2 template pipeline is working. This path generates truly novel body shapes that MPFB2 templates can't cover.

```
HiDream concept image
    → Trellis v2 (image → 3D mesh, MIT license)
    → Retopology (Instant Meshes CLI + Blender cleanup)
    → Face component separation (MediaPipe landmarks)
    → UniRig (auto-skeleton + skinning, MIT license)
    → Deformation transfer (ARKit blend shapes from reference)
    → GLB export
```

This path may revisit ICT FaceKit and deformation transfer. Higher risk, higher novelty. The template pipeline fallback ensures users always get a working avatar even if the AI path fails quality checks.

---

## Cloud Infrastructure (RunPod Serverless)

All GPU compute runs on RunPod Serverless. Same endpoints for development and production.

### Containers (built with Podman, pushed to Quay.io)

#### Container 1: MPFB2 + Blender 5 (workhorse)
```
Base: ubuntu:24.04
+ Blender 5.0 headless
+ MPFB2 addon installed
+ ALL MakeHuman/MPFB2 asset packs (faceunits, visemes, clothing, hair, skins, targets, etc.)
+ Python deps: numpy, scipy, runpod
+ Pipeline scripts
Registry: quay.io/kokokino/meshborn-blender:latest
GPU: CPU-only endpoint (~$0.20/hr) or GPU for rendering
```

Handles: character generation, texture projection, GLB export, thumbnail rendering.

#### Container 2: HiDream-I1
```
Base: pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime
+ HiDream-I1-Dev (baked weights)
+ diffusers, transformers, accelerate, runpod, rembg
Registry: quay.io/kokokino/meshborn-hidream:latest
GPU: RTX 4090 24GB (~$0.69/hr)
```

Handles: concept image generation, face close-up generation.

#### Container 3: Trellis v2 (post-MVP)
```
Base: pytorch CUDA image
+ Trellis v2 model weights
Registry: quay.io/kokokino/meshborn-trellis:latest
GPU: RTX 4090/5090 (~$0.69-0.89/hr)
```

#### Container 4: UniRig (post-MVP)
```
Base: pytorch CUDA image
+ UniRig checkpoints
Registry: quay.io/kokokino/meshborn-unirig:latest
GPU: RTX 4090 (~$0.69/hr)
```

### Podman Build & Push

```bash
# Build
podman build -t quay.io/kokokino/meshborn-blender:latest -f containers/blender/Containerfile .
podman build -t quay.io/kokokino/meshborn-hidream:latest -f containers/hidream/Containerfile .

# Push to Quay.io
podman push quay.io/kokokino/meshborn-blender:latest
podman push quay.io/kokokino/meshborn-hidream:latest
```

RunPod Serverless pulls from Quay.io just like Docker Hub — any OCI-compatible registry works.

---

## Storage (Backblaze B2)

Following backlog-beacon's pattern: AWS SDK with S3-compatible API, not ostrio:files for B2 mode.

### NPM Dependencies (same as backlog-beacon)
```
@aws-sdk/client-s3
@aws-sdk/lib-storage
```

### Bucket Structure

```
b2://meshborn-avatars/
  avatars/
    {userId}/
      {avatarId}/
        hero.png              # Approved HiDream concept image
        face_closeup.png      # High-res face for texture
        variants/             # Rejected concept alternatives
          variant_01.png
          variant_02.png
        params.json           # MPFB2 body/face parameters (remix key)
        mesh_pretexture.glb   # Mesh before texturing
        avatar.glb            # Final TalkingHead-ready GLB
        avatar_lowpoly.glb    # Low-poly variant if generated
        thumbnail.jpg         # 256x256 gallery thumbnail
        metadata.json         # Prompt, settings, timestamps
```

### Settings Structure
```json
{
  "private": {
    "storage": {
      "type": "b2",
      "b2": {
        "applicationKeyId": "...",
        "applicationKey": "...",
        "bucketName": "meshborn-avatars",
        "region": "us-east-005"
      }
    }
  }
}
```

### Remix Support

`params.json` stores the exact MPFB2 slider values that generated the character. For remixing:
- Same body, different face → reload params, modify face values, re-enter at stage 4
- Same face, different appearance → reload params, re-enter at stage 3 (new HiDream image)
- All intermediate files stored so users can branch from any pipeline stage

### MongoDB Collections (metadata only — binary files in B2)

```javascript
// AvatarsCollection
{
  _id: "abc123",
  userId: "user456",
  prompt: "A stocky middle-aged man...",
  style: "realistic",
  params: { /* MPFB2 target values */ },
  assets: { skin: "skins02/middleage_caucasian", hair: "hair01/short_receding", ... },
  status: "complete",
  files: {
    hero: "https://f003.backblazeb2.com/.../hero.png",
    glb: "https://f003.backblazeb2.com/.../avatar.glb",
    thumbnail: "https://f003.backblazeb2.com/.../thumbnail.jpg"
  },
  pipelineState: {
    conceptGenerated: true,
    conceptApproved: true,
    meshGenerated: true,
    textured: true,
    blendshapesApplied: true,
    exported: true
  },
  createdAt: ISODate("..."),
  generationTimeMs: 85000
}

// JobsCollection (drives real-time UI progress via DDP)
{
  _id: "job789",
  userId: "user456",
  avatarId: "abc123",
  status: "generating_mesh",  // reactive, drives progress bar
  progress: 55,               // 0-100
  currentStep: "Adding Mixamo rig",
  runpodJobIds: {
    hidream: "rp_xxx",
    blender: "rp_yyy"
  },
  createdAt: ISODate("..."),
  updatedAt: ISODate("...")
}
```

The `pipelineState` object enables remix re-entry — load an existing avatar, check which stages are complete, and re-enter at the right point.

---

## LLM Layer (OpenRouter)

Following the false-colors spoke app pattern: server-side only, native `fetch()`, tiered model fallback, per-user call caps.

### Why OpenRouter over self-hosted LLMs on RunPod

Self-hosting an LLM (e.g., Llama 3.1 on RunPod) has a cold start problem: 30-60 seconds to load the model after idle. Keeping a warm worker costs ~$500/month. OpenRouter costs ~$0.0005 per avatar for prompt decomposition — effectively a rounding error compared to HiDream and Blender compute costs. OpenRouter also provides vision models for the post-MVP image-to-parameters path through the same API.

### What needs an LLM

1. **Prompt decomposition** (every avatar): User text → structured JSON with MPFB2 target values + asset selections + HiDream prompt
2. **HiDream prompt optimization** (every avatar): Rewrite user text into image-gen-optimized prompt (T-pose, white bg, studio lighting)
3. **Iterative refinement** (occasional): "Make hair longer" → which params to change and by how much
4. **Image → parameters** (post-MVP): Vision model extracts proportions from uploaded photo → MPFB2 targets

### Model Fallback Chain

```
1. GPT-4o-mini          — $0.15/$0.60 per M tokens, excellent structured JSON,
                          supports response_format: { type: "json_object" }
2. Gemini Flash 2.0     — comparably cheap, good structured output
3. Claude Haiku 3.5     — more expensive but very reliable if cheaper models struggle
```

All three models fail → show user an outage message ("We're experiencing a temporary issue, please try again shortly"). The prompt decomposition is essential to the pipeline — without it, we cannot map user intent to MPFB2 parameters, so there is no degraded mode.

### Settings Structure

```json
{
  "private": {
    "ai": {
      "openRouterApiKey": "sk-or-v1-...",
      "openRouterBaseUrl": "https://openrouter.ai/api/v1",
      "openRouterModels": [
        "openai/gpt-4o-mini",
        "google/gemini-2.0-flash-001",
        "anthropic/claude-3.5-haiku"
      ],
      "maxCallsPerUserPerDay": 50,
      "maxTokensPerCall": 1000,
      "temperature": 0.3
    }
  }
}
```

Lower temperature (0.3) than false-colors (0.8) because we need consistent, predictable parameter mapping rather than creative variation.

### Implementation Pattern (from false-colors)

- Native `fetch()` to OpenRouter API (no SDK dependency)
- Server-side Meteor methods only — API key never reaches client
- Tiered fallback: try each model in order, catch errors, continue to next
- Per-user daily call cap to prevent cost runaway
- All calls include `response_format: { type: "json_object" }` where supported

### Cost Estimate

| Scale | Avatars/day | LLM cost/month |
|-------|------------|---------------|
| Demo | 5 | ~$0.08 |
| Small | 50 | ~$0.75 |
| Medium | 500 | ~$7.50 |

---

## Spoke App Structure (New Files)

Built on the existing spoke skeleton. New additions:

```
meshborn/
├── imports/
│   ├── api/
│   │   ├── avatars/
│   │   │   ├── collection.js         # Avatars MongoDB collection
│   │   │   ├── methods.js            # generate, refine, remix, delete
│   │   │   └── publications.js       # userAvatars, avatarDetail
│   │   └── jobs/
│   │       ├── collection.js         # Generation jobs collection
│   │       ├── methods.js            # createJob, cancelJob
│   │       └── publications.js       # jobStatus (reactive progress)
│   ├── lib/
│   │   ├── runpodClient.js           # RunPod Serverless API client
│   │   ├── b2Storage.js              # Backblaze B2 upload/download/delete
│   │   ├── storageClient.js          # S3Client initialization
│   │   ├── promptDecomposer.js       # LLM prompt → MPFB2 params
│   │   └── llmProxy.js              # OpenRouter client (fetch-based, model fallback chain)
│   └── ui/
│       ├── pages/
│       │   ├── GeneratorPage.js      # Main generation flow
│       │   ├── GalleryPage.js        # User's saved avatars
│       │   └── AvatarDetailPage.js   # Single avatar view + remix
│       └── components/
│           ├── PromptInput.js        # Text prompt + style selector
│           ├── ConceptPreview.js     # HiDream image approval UI
│           ├── AvatarViewer.js       # Babylon.js GLB viewer
│           ├── ProgressTracker.js    # Real-time generation progress
│           ├── ParamSliders.js       # Body/face customization
│           └── AvatarCard.js         # Gallery thumbnail card
├── server/
│   ├── avatarPipeline.js             # Multi-step pipeline orchestrator
│   ├── runpodWebhooks.js             # Receive RunPod completion callbacks
│   └── migrations/
│       └── 2_create_avatars_indexes.js
├── containers/
│   ├── blender/
│   │   ├── Containerfile             # Podman/OCI container definition
│   │   ├── handler.py                # RunPod serverless handler
│   │   └── scripts/
│   │       ├── generate_character.py # MPFB2 API: create humanoid
│   │       ├── project_textures.py   # Texture projection
│   │       ├── export_glb.py         # Final GLB export
│   │       └── validate.py           # Morph target validation
│   └── hidream/
│       ├── Containerfile
│       └── handler.py
└── public/
    └── (static assets served by Meteor)
```

---

## Development Phases

### Phase 0: Foundation, Cloud Setup & Texturing Spike

**Goal**: Establish cloud infrastructure. Prove MPFB2 works headless. De-risk texture projection early.

**Tasks**:

*Cloud infrastructure:*
- Set up Quay.io account and repository
- Install Podman locally
- Build Blender 5 + MPFB2 container with ALL asset packs
- Deploy to RunPod Serverless as first endpoint
- Test calling the endpoint from local Meteor dev server
- Set up Backblaze B2 bucket (meshborn-avatars)
- Implement `b2Storage.js` and `storageClient.js` (following backlog-beacon pattern)
- Add RunPod API client module
- Add Jobs collection + reactive publication for progress tracking

*MPFB2 smoke test (critical gate — proves or disproves the architecture):*
- Import MPFB2 services in `--background` mode:
  `from mpfb.services.humanservice import HumanService`
- Call `HumanService.create_human()` headlessly
- **Risk**: Some MPFB2 code uses `bpy.ops` calls that require a UI context. If these crash in `--background` mode, wrap with context overrides or patch the specific service functions (MPFB2 is open source GPLv3). This is the single biggest technical risk — but the architecture strongly suggests it will work, and it will be known within the first day of Phase 0.
- Apply body/face targets via shape key values
- Load system assets (eyes, teeth, tongue, eyelashes, eyebrows)
- Add Mixamo rig
- Load Faceunits 01 (ARKit shape keys) — verify all 52 present and correctly named
- Load Visemes 02 (Oculus visemes) — verify all 15 present
- Export as GLB, load in Babylon.js viewer
- Load in TalkingHead test page to verify lip-sync works

*Texturing spike (test both approaches before committing):*
- **Approach A — MPFB2 built-in materials**: Use MPFB2's procedural skin system (skin tone, eye color, ethnicity parameters) to generate a textured character. Evaluate quality, flexibility, and how "unique" characters can look.
- **Approach B — Blender UV projection**: Generate a test image with HiDream, project it onto the MPFB2 mesh's UV layout using Blender's texture baking. Evaluate seam quality, face alignment, and overall fidelity.
- Compare results, document findings. The winner becomes the Phase 2 texturing strategy. Both approaches may end up complementary (MPFB2 materials for body, HiDream projection for face).

**Verification**: Call RunPod endpoint from localhost:3050, receive a GLB with correct morph targets, store it in B2, load it in Babylon.js. Have clear texturing strategy chosen.

### Phase 1: MPFB2 Character Pipeline

**Goal**: Generate parametric characters from structured parameters.

**Tasks**:
- Write `generate_character.py` (MPFB2 Python API):
  - Accept body/face parameters as JSON
  - Generate humanoid with MPFB2
  - Add system assets (eyes, teeth, tongue, eyelashes)
  - Add Mixamo rig
  - Load Faceunits 01 (ARKit shape keys)
  - Load Visemes 02 (Oculus viseme shape keys)
  - Export as GLB
- Write `validate.py`:
  - Check all 67+ morph targets present with correct names
  - Check skeleton bone count and naming
  - Check separate mesh objects (eyes, teeth, tongue)
- Write `llmProxy.js` (OpenRouter client, following false-colors pattern):
  - Native fetch, model fallback chain, per-user call caps
  - All models fail → outage message, no degraded mode
- Write `promptDecomposer.js`:
  - Call OpenRouter via llmProxy to parse user prompt into MPFB2 parameters
  - Map natural language ("tall, athletic woman") to slider values + asset selections + HiDream prompt
  - System prompt includes curated MPFB2 target list with ranges and descriptions
- Build ParamSliders component for manual parameter adjustment
- Build ProgressTracker component (reactive via DDP)
- Store params.json in B2 for each generation

**Verification**: Type "a tall athletic woman with angular features" → get a GLB with correct morph targets and rig. Load in Babylon.js viewer.

### Phase 2: HiDream Appearance Pipeline

**Goal**: Generate unique character appearances and project onto MPFB2 meshes.

**Tasks**:
- Build HiDream-I1 container, deploy to RunPod
- Write hero image generation handler:
  - Full-body T-pose on white background
  - Face close-up for texture detail
  - Background removal with rembg
- Build ConceptPreview component (image approval flow)
- Write `project_textures.py` (Blender 5 headless):
  - Project face close-up onto mesh face UV
  - Project body image onto body UV islands
  - Bake to texture map (2048x2048)
- Build end-to-end pipeline orchestrator (`avatarPipeline.js`):
  - Decompose prompt → generate images → user approval → generate mesh → project textures → export → upload to B2
- Implement webhook handler for RunPod completion callbacks

**Verification**: Full flow from text prompt to textured, TalkingHead-ready GLB in Babylon.js viewer with working morph targets.

### Phase 3: Gallery, Remix & Polish

**Goal**: Complete user experience.

**Tasks**:
- Build GalleryPage (user's saved avatars with thumbnails)
- Build AvatarDetailPage (full viewer + download + remix options)
- Implement remix flow:
  - Load params.json from B2
  - Re-enter pipeline at appropriate stage
  - "Same body, different look" / "Same face, tweak proportions"
- Build AvatarViewer with Babylon.js:
  - GLB loading with morph target animation
  - Morph target slider controls for testing
  - Animation playback (idle, etc.)
- Add TalkingHead demo (iframe embed or native Babylon.js lip-sync)
- Download options (GLB, separate params.json)
- Add subscription gating (avatar generation requires subscription)
- Polish UI with Pico CSS

**Verification**: Full user journey from login → generate → approve → view → remix → download.

### Phase 4: Production Deploy

**Goal**: Live service on Meteor Galaxy.

**Tasks**:
- Configure production settings (RunPod API keys, B2 credentials, Hub SSO)
- Set subscription tiers in Kokokino Hub:
  - Free: 3 avatars/month
  - Pro: 50 avatars/month + remix
- Deploy to Galaxy: `meteor deploy meshborn.kokokino.com`
- Set up monitoring (RunPod dashboard, Galaxy APM)
- Error handling: automatic fallback/retry
- Rate limiting on generation endpoints

### Phase 5: AI Mesh Path (Post-MVP)

**Goal**: Support fully AI-generated novel meshes via Trellis v2.

**Tasks**:
- Build Trellis v2 container, deploy to RunPod
- Build UniRig container, deploy to RunPod
- Implement retopology pipeline (Instant Meshes + Blender cleanup)
- Implement face component separation (MediaPipe)
- Implement deformation transfer for non-template meshes
- Quality scoring with automatic fallback to template path

---

## Tech Stack Summary

### Core (All Commercially Clean)

| Component | License | Role |
|-----------|---------|------|
| Meteor 3.4 | MIT | Spoke app framework |
| Mithril.js 2.3 | MIT | UI components |
| Babylon.js v8 | Apache 2.0 | 3D viewer, morph target animation |
| MPFB2 | GPLv3 (code), CC0 (assets + output) | Parametric humanoid generation (server-side only) |
| MPFB2 asset packs | CC0 | ARKit shape keys, Oculus visemes |
| HiDream-I1 | MIT | Concept image generation |
| Trellis v2 | MIT | Image → 3D mesh (post-MVP) |
| UniRig | MIT | Auto-rigging (post-MVP) |
| Blender 5.0 | GPL | Headless mesh processing (server-side only) |
| Pico CSS | MIT | Classless styling |
| OpenRouter | N/A (API service) | LLM routing — prompt decomposition, prompt optimization |

### Infrastructure

| Service | Role |
|---------|------|
| Meteor Galaxy | Web app hosting |
| RunPod Serverless | GPU/CPU compute (scales to zero) |
| Quay.io | Container registry |
| Backblaze B2 | File storage (S3-compatible) |
| Kokokino Hub | SSO + billing |
| Podman | Container build/run |

### Per-Avatar Cost Estimate (Template Path)

| Step | Time | Cost |
|------|------|------|
| HiDream hero + face | ~20s on RTX 4090 | ~$0.004 |
| MPFB2 character gen | ~30s on CPU | ~$0.002 |
| Texture projection | ~30s on CPU | ~$0.002 |
| GLB export | ~15s on CPU | ~$0.001 |
| **Total** | **~2 min** | **~$0.01** |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| MPFB2 fails to run headlessly (`bpy.ops` context issues) | Low-Medium | Critical | Phase 0 validates this first. Wrap failing calls with context overrides. Worst case: patch specific MPFB2 service functions (open source GPLv3). |
| Faceunits 01 doesn't contain all 52 ARKit shapes | Low | High | Verify in Phase 0. Same author created both TalkingHead and the packs — full coverage is near-certain. If subset, supplement missing shapes. |
| MPFB2 Mixamo rig weight painting quality | Medium | Medium | Known to be "crude in places" per MPFB2 release notes. Works for common body types. Extreme proportions may need weight cleanup scripts. |
| HiDream concept images poor for texture projection | Medium | Medium | Phase 0 texturing spike de-risks this. Fallback to MPFB2's procedural skin materials. Can improve with prompt engineering. |
| RunPod cold start latency | Low | Low | FlashBoot reduces to sub-200ms for pre-warmed workers. First request after long idle may take 30-60s for model loading. |
| Viseme naming mismatch with TalkingHead | Very Low | Low | Same author created both. If somehow different, simple rename script in export. |

---

## Resolved Decisions

1. **MakeHuman asset packs**: Install ALL available packs in the container from the start. More variety, fewer rebuilds.

2. **Texturing approach**: Spike both MPFB2 built-in materials AND Blender UV projection in Phase 0 before committing to either. They may be complementary.

3. **3D generation tool**: Trellis v2 only (MIT license). Hunyuan3D deferred/excluded for now.

4. **Containers**: Podman + Quay.io (not Docker + Docker Hub).

5. **Blender version**: Blender 5 in cloud containers only. No local Blender.

## Notes

- **MPFB2 license (GPLv3)**: Code is GPLv3, runs server-side only in containers, never distributed to users. All assets (meshes, textures, targets, rigs) are CC0. Generated characters have no restrictions — the MakeHuman team explicitly states that output contains no trace of program logic and can be sold, modified, and sublicensed freely.
- **Deformation transfer / ICT FaceKit**: Not needed for MVP (MPFB2 handles blend shapes natively). May revisit for the post-MVP AI mesh path.
- **TalkingHead compatibility**: Confirmed via Mika Suominen's own asset packs — same person built both TalkingHead and the MPFB2 packs that bridge to it.
