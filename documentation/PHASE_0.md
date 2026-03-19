# Phase 0: Foundation, Cloud Setup & Texturing Spike

## Context

Phase 0 establishes cloud infrastructure, proves MPFB2 works headless in Blender 5, and de-risks texture projection. No real pipeline code is written here — this phase validates the architecture before committing to it.

The MPFB2 smoke test is the **critical gate**. If MPFB2 cannot generate a humanoid headlessly in Blender's `--background` mode, the entire architecture must change. The plan front-loads this gate so it's known within the first day.

**Goal**: Call RunPod endpoint from localhost:3010, receive a GLB with correct morph targets, store it in B2, load it in Babylon.js. Have clear texturing strategy chosen.

---

## Dependency Graph

```
MANUAL: Create accounts (Quay.io, Backblaze B2, RunPod)
    |
    +-- Workstream A: Container (critical gate)
    |   Step 1 -> Step 2 -> Step 4 -> Step 5
    |
    +-- Workstream B: Meteor infrastructure (parallel with A)
    |   Step 3 -> Step 5
    |
    +-- Workstream C: Texturing spike (after gate passes)
    |   Step 6 (after Step 4)
    |
    +-- Workstream D: Viewer + verification
        Step 7 (after Step 5) -> Step 8
```

Steps 2 and 3 can run in parallel. Steps 6 and 7 can run in parallel.

---

## Step 1: Account Setup & Tooling (Manual)

No code. These must happen first.

### 1.1 Install Podman

```bash
brew install podman
podman machine init
podman machine start
podman info  # verify
```

### 1.2 Create Quay.io Repository

- Create account at quay.io (or use existing `kokokino` org)
- Create repository: `kokokino/meshborn-blender`
- Set visibility to public (all Kokokino code is open source)
- Full image path: `quay.io/kokokino/meshborn-blender`

### 1.3 Create Backblaze B2 Bucket

- Log in to backblaze.com
- Create bucket: `meshborn-avatars`
- Region: `us-east-005` (or closest to RunPod GPU region)
- Bucket type: Private
- Create application key scoped to this bucket (read/write)
- Record: `applicationKeyId`, `applicationKey`, `bucketName`, `region`

### 1.4 Create RunPod Account

- Sign up at runpod.io, add payment method
- Navigate to Serverless section
- Do NOT create an endpoint yet — that happens after the container is built and pushed (Step 4)

---

## Step 2: MPFB2 Smoke Test Container (Critical Gate)

This is the single most important task in Phase 0.

### 2.1 Create Container Directory Structure

```
containers/
  blender/
    Containerfile
    scripts/
      smoke_test.py
```

### 2.2 Write the Containerfile

**File**: `containers/blender/Containerfile`

The container must include:

| Component | Source | Notes |
|-----------|--------|-------|
| Ubuntu 24.04 | Base image | |
| Blender 5.0 headless | blender.org Linux tarball | Self-contained with bundled Python |
| MPFB2 addon | makehumancommunity.org release | Install to `~/.config/blender/5.0/scripts/addons/` |
| ALL MPFB2 asset packs | makehumancommunity.org downloads | Install to MPFB2's data directory |
| Python deps | Blender's pip | `runpod`, `numpy`, `scipy`, `boto3` |

Key details:
- Install Python packages into **Blender's bundled Python**, not the system Python: `/opt/blender/5.0/python/bin/pip install runpod numpy scipy boto3`
- System deps needed for headless Blender: `libgl1 libxi6 libxrender1 libxxf86vm1 libxfixes3 libxkbcommon0`
- MPFB2 asset packs include: Faceunits 01 (ARKit), Visemes 02 (Oculus), system assets (eyes, teeth, tongue, eyelashes), clothing, hair, skins, targets
- Check makehumancommunity.org for exact download URLs and data directory paths at implementation time

Skeleton:

```dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV BLENDER_VERSION=5.0.0
ENV BLENDER_PATH=/opt/blender

# System dependencies for headless Blender
RUN apt-get update && apt-get install -y \
    curl wget unzip xz-utils libgl1 libxi6 libxrender1 \
    libxxf86vm1 libxfixes3 libxkbcommon0 && \
    rm -rf /var/lib/apt/lists/*

# Install Blender 5
RUN wget -q https://download.blender.org/release/Blender5.0/blender-${BLENDER_VERSION}-linux-x64.tar.xz && \
    tar xf blender-${BLENDER_VERSION}-linux-x64.tar.xz && \
    mv blender-${BLENDER_VERSION}-linux-x64 ${BLENDER_PATH} && \
    rm blender-${BLENDER_VERSION}-linux-x64.tar.xz

# Install MPFB2 addon (check makehumancommunity.org for current URL)
RUN wget -q <MPFB2_RELEASE_URL> -O mpfb2.zip && \
    mkdir -p /root/.config/blender/5.0/scripts/addons/ && \
    unzip mpfb2.zip -d /root/.config/blender/5.0/scripts/addons/ && \
    rm mpfb2.zip

# Install ALL MPFB2 asset packs (check makehumancommunity.org for URLs)
RUN mkdir -p /root/.mpfb2/data && \
    # Download and extract each pack...
    echo "Asset packs installed"

# Install Python deps into Blender's Python
RUN ${BLENDER_PATH}/5.0/python/bin/pip install runpod numpy scipy boto3

COPY containers/blender/scripts/ /app/scripts/
COPY containers/blender/handler.py /app/handler.py

WORKDIR /app
CMD ["python", "/app/handler.py"]
```

The exact Blender download URL and MPFB2 asset pack URLs must be verified at implementation time — check blender.org/download and makehumancommunity.org respectively.

### 2.3 Write the Smoke Test Script

**File**: `containers/blender/scripts/smoke_test.py`

Run via: `blender --background --python scripts/smoke_test.py`

The script validates every critical capability in sequence:

| Test | What It Proves |
|------|---------------|
| A. Import MPFB2 services | MPFB2 loads in `--background` mode |
| B. `HumanService.create_human()` | Headless human creation works |
| C. Apply body/face targets | Shape key values can be set |
| D. Load system assets | Eyes, teeth, tongue, eyelashes, eyebrows |
| E. Add Mixamo rig | `RigService` works headlessly |
| F. Load Faceunits 01 | All 52 ARKit shape keys present |
| G. Load Visemes 02 | All 15 Oculus visemes present |
| H. Export GLB | `bpy.ops.export_scene.gltf()` with `export_morph=True` |
| I. Validate export | Correct morph target count in output |

Expected ARKit shape keys (52): browDownLeft, browDownRight, browInnerUp, browOuterUpLeft, browOuterUpRight, cheekPuff, cheekSquintLeft, cheekSquintRight, eyeBlinkLeft, eyeBlinkRight, eyeLookDownLeft, eyeLookDownRight, eyeLookInLeft, eyeLookInRight, eyeLookOutLeft, eyeLookOutRight, eyeLookUpLeft, eyeLookUpRight, eyeSquintLeft, eyeSquintRight, eyeWideLeft, eyeWideRight, jawForward, jawLeft, jawOpen, jawRight, mouthClose, mouthDimpleLeft, mouthDimpleRight, mouthFrownLeft, mouthFrownRight, mouthFunnel, mouthLeft, mouthLowerDownLeft, mouthLowerDownRight, mouthPressLeft, mouthPressRight, mouthPucker, mouthRight, mouthRollLower, mouthRollUpper, mouthShrugLower, mouthShrugUpper, mouthSmileLeft, mouthSmileRight, mouthStretchLeft, mouthStretchRight, mouthUpperUpLeft, mouthUpperUpRight, noseSneerLeft, noseSneerRight, tongueOut.

Expected Oculus visemes (15): viseme_aa, viseme_CH, viseme_DD, viseme_E, viseme_FF, viseme_I, viseme_kk, viseme_nn, viseme_O, viseme_PP, viseme_RR, viseme_sil, viseme_SS, viseme_TH, viseme_U.

The script must print clear PASS/FAIL for each test and exit with code 0 on full success, non-zero on any failure.

### 2.4 Build and Run Locally

```bash
cd /Users/recurve/Documents/programming/kokokino/meshborn
podman build -t meshborn-blender:test -f containers/blender/Containerfile .
podman run --rm meshborn-blender:test \
  /opt/blender/blender --background --python /app/scripts/smoke_test.py
```

### 2.5 Critical Gate Decision

**If all tests PASS**: Architecture validated. Proceed to Step 4.

**If imports or `bpy.ops` calls fail** (most likely failure mode), try mitigations in order:

1. **Context overrides**: Wrap failing `bpy.ops` calls with `bpy.context.temp_override(...)` (Blender 4.0+ syntax). Many ops that previously required a UI context work with temp overrides.

2. **MPFB2 non-ops API**: Check if the MPFB2 service layer has direct mesh manipulation functions that bypass `bpy.ops`. `HumanService` may have lower-level methods.

3. **Patch MPFB2 source**: MPFB2 is GPLv3 open source. Fork the specific service functions that fail and replace `bpy.ops` calls with direct mesh manipulation.

4. **Xvfb fallback**: Install virtual framebuffer in the container and run Blender with a virtual display instead of `--background`. Adds container size but solves all context issues:
   ```dockerfile
   RUN apt-get install -y xvfb
   # Run via: xvfb-run blender --python script.py
   ```

5. **Architecture pivot** (extremely unlikely): If MPFB2 fundamentally cannot work headless, fall back to MakeHuman CLI. This requires a new planning session.

---

## Step 3: Meteor Server Infrastructure (Parallel with Step 2)

These tasks can begin immediately alongside the smoke test. They only become useful if the gate passes, but starting in parallel saves time.

### 3.1 Add npm Dependencies

**File to modify**: `package.json`

```bash
meteor npm install @aws-sdk/client-s3@^3.974.0 @aws-sdk/lib-storage@^3.980.0
```

Same versions as backlog-beacon.

### 3.2 Add Avatars and Jobs Collections

**File to modify**: `imports/api/collections.js`

Add after existing collection exports:

```javascript
export const Avatars = new Mongo.Collection('avatars');
export const Jobs = new Mongo.Collection('jobs');
```

Follows the existing single-file pattern where all collections are defined in `collections.js`.

**Avatars document shape** (from PLAN.md):
```javascript
{
  _id: "abc123",
  userId: "user456",
  prompt: "A stocky middle-aged man...",
  status: "complete",  // queued | generating | complete | failed
  files: {
    glb: "https://...backblazeb2.com/.../avatar.glb",
    thumbnail: "https://...backblazeb2.com/.../thumbnail.jpg"
  },
  params: { /* MPFB2 target values */ },
  createdAt: new Date(),
  generationTimeMs: 85000
}
```

**Jobs document shape** (drives real-time UI progress via DDP):
```javascript
{
  _id: "job789",
  userId: "user456",
  avatarId: "abc123",
  status: "generating_mesh",  // queued | submitted | generating_mesh | complete | failed
  progress: 55,               // 0-100
  currentStep: "Adding Mixamo rig",
  runpodJobId: "rp_xxx",
  createdAt: new Date(),
  updatedAt: new Date()
}
```

### 3.3 Create storageClient.js

**File to create**: `imports/lib/storageClient.js`

Follow the backlog-beacon pattern from `backlog_beacon/server/covers/storageClient.js`:

```javascript
import { Meteor } from 'meteor/meteor';
import { S3Client } from '@aws-sdk/client-s3';

let s3Client = null;
let storageConfig = null;

export function initStorageClient() {
  const settings = Meteor.settings.private?.storage;
  storageConfig = settings || { type: 'local' };

  if (storageConfig.type === 'b2' && storageConfig.b2) {
    const { applicationKeyId, applicationKey, region } = storageConfig.b2;

    if (!applicationKeyId || !applicationKey || !region) {
      console.error('Storage: B2 configuration incomplete');
      storageConfig = { type: 'local' };
      return;
    }

    s3Client = new S3Client({
      endpoint: `https://s3.${region}.backblazeb2.com`,
      region: region,
      credentials: {
        accessKeyId: applicationKeyId,
        secretAccessKey: applicationKey
      },
      forcePathStyle: false
    });
    console.log('Storage: Initialized B2 client');
  } else {
    console.log('Storage: B2 not configured');
  }
}

export function getS3Client() { return s3Client; }
export function getStorageConfig() { return storageConfig; }
export function isUsingB2() { return storageConfig?.type === 'b2'; }
```

### 3.4 Create b2Storage.js

**File to create**: `imports/lib/b2Storage.js`

Follow the backlog-beacon pattern from `backlog_beacon/server/covers/b2Storage.js`:

Exports:
- `uploadToB2(buffer, key, contentType)` — PutObjectCommand, returns public URL
- `uploadStreamToB2(stream, key, contentType)` — Upload from @aws-sdk/lib-storage, for large files like GLBs
- `deleteFromB2(key)` — DeleteObjectCommand
- `checkB2FileExists(key)` — HeadObjectCommand, handles 404 gracefully
- `extractKeyFromB2Url(url)` — regex to extract key from public URL
- `getDownloadUrl(key)` — constructs public URL without uploading (new, not in backlog-beacon)

URL format: `https://${bucketName}.s3.${region}.backblazeb2.com/${key}`

Key path convention:
```
avatars/{userId}/{avatarId}/avatar.glb
avatars/{userId}/{avatarId}/thumbnail.jpg
avatars/{userId}/{avatarId}/params.json
avatars/{userId}/{avatarId}/hero.png
```

### 3.5 Create runpodClient.js

**File to create**: `imports/lib/runpodClient.js`

New module — no existing kokokino pattern to follow. Uses native `fetch()` (Node.js 22.x), similar approach to `false-colors/imports/ai/llmProxy.js`.

Exports:
- `submitJob(endpointId, input)` — POST to `https://api.runpod.ai/v2/{endpointId}/run`, returns `{ id, status }`
- `checkJobStatus(endpointId, jobId)` — GET from `https://api.runpod.ai/v2/{endpointId}/status/{jobId}`
- `cancelJob(endpointId, jobId)` — POST to cancel endpoint
- `pollJobUntilComplete(endpointId, jobId, intervalMs, timeoutMs)` — polls at intervals, resolves on completion, rejects on timeout/failure

Authentication: `Authorization: Bearer ${Meteor.settings.private.runpod.apiKey}`

Endpoint IDs from: `Meteor.settings.private.runpod.endpoints.blender`

Include an AbortController timeout on each fetch call (30 seconds), matching the false-colors pattern.

### 3.6 Add Jobs Publication

**File to modify**: `server/publications.js`

Add after existing publications:

```javascript
import { Jobs } from '../imports/api/collections.js';

Meteor.publish('jobStatus', function(avatarId) {
  check(avatarId, String);
  if (!this.userId) {
    return this.ready();
  }
  return Jobs.find(
    { avatarId, userId: this.userId },
    { fields: { status: 1, progress: 1, currentStep: 1, updatedAt: 1, avatarId: 1 } }
  );
});
```

This enables real-time progress tracking. When the server updates the Jobs document while polling RunPod, Meteor's oplog tailing pushes changes to subscribed clients.

### 3.7 Add Test Meteor Method

**File to modify**: `server/methods.js`

Add a `pipeline.smokeTest` method that ties together all infrastructure:
1. Check user is authenticated
2. Create an Avatars document (status: `queued`)
3. Create a Jobs document (status: `queued`)
4. Call `submitJob()` with the blender endpoint and smoke test action
5. Update Jobs to `submitted` with the RunPod job ID
6. Poll for completion, updating Jobs progress as it goes
7. On completion: update Avatars with the B2 URL, set Jobs to `complete`
8. Return the avatar ID

This method is temporary — it exists only to verify the end-to-end flow in Step 5.

### 3.8 Update settings.example.json

**File to modify**: `settings.example.json`

```json
{
  "public": {
    "appName": "Meshborn",
    "appId": "meshborn",
    "hubUrl": "https://kokokino.com",
    "requiredProducts": ["base-monthly"]
  },
  "private": {
    "hubApiKey": "YOUR_SPOKE_API_KEY_HERE",
    "hubApiUrl": "https://kokokino.com/api/spoke",
    "hubPublicKey": "-----BEGIN PUBLIC KEY-----\nYOUR_HUB_PUBLIC_KEY_HERE\n-----END PUBLIC KEY-----",
    "storage": {
      "type": "b2",
      "b2": {
        "applicationKeyId": "YOUR_B2_KEY_ID",
        "applicationKey": "YOUR_B2_APPLICATION_KEY",
        "bucketName": "meshborn-avatars",
        "region": "us-east-005"
      }
    },
    "runpod": {
      "apiKey": "YOUR_RUNPOD_API_KEY",
      "endpoints": {
        "blender": "YOUR_BLENDER_ENDPOINT_ID"
      }
    }
  }
}
```

### 3.9 Add Database Indexes

**File to modify**: `server/indexes.js`

Add indexes for new collections:

```javascript
// Avatars: lookup by userId, sorted by creation
await Avatars.createIndexAsync({ userId: 1, createdAt: -1 });

// Jobs: lookup by avatarId + userId
await Jobs.createIndexAsync({ avatarId: 1, userId: 1 });

// Jobs: TTL index to auto-clean completed jobs after 24 hours
await Jobs.createIndexAsync(
  { updatedAt: 1 },
  { expireAfterSeconds: 86400, partialFilterExpression: { status: 'complete' } }
);
```

### 3.10 Add Migration

**File to create**: `server/migrations/2_create_avatars_jobs_indexes.js`

Follow the pattern from `server/migrations/1_create_used_nonces_ttl_index.js`. Register as version 2. Creates the indexes from 3.9 via `rawCollection().createIndex()`.

**File to modify**: `server/migrations/0_steps.js`

Add import for the new migration file:
```javascript
import './2_create_avatars_jobs_indexes.js';
```

### 3.11 Update server/main.js

**File to modify**: `server/main.js`

Add:
1. Import `initStorageClient` from `../imports/lib/storageClient.js`
2. Call `initStorageClient()` in `Meteor.startup()`
3. Add validation warnings for missing `storage` and `runpod` config

---

## Step 4: RunPod Handler & Character Generation Scripts

Depends on Step 2 passing (critical gate).

### 4.1 Write handler.py

**File to create**: `containers/blender/handler.py`

RunPod serverless handler. Receives JSON input, dispatches to Blender scripts via subprocess, returns results.

```python
import runpod
import subprocess
import json
import os

def handler(event):
    input_data = event.get("input", {})
    action = input_data.get("action", "smoke_test")

    scripts = {
        "smoke_test": "/app/scripts/smoke_test.py",
        "generate_character": "/app/scripts/generate_character.py",
        "validate": "/app/scripts/validate.py",
        "texture_spike_a": "/app/scripts/texture_spike_a.py",
        "texture_spike_b": "/app/scripts/texture_spike_b.py",
    }

    script = scripts.get(action)
    if not script:
        return {"error": f"Unknown action: {action}"}

    # Write input params for Blender script to read
    with open("/tmp/input_params.json", "w") as f:
        json.dump(input_data, f)

    # Run Blender headlessly
    result = subprocess.run(
        ["/opt/blender/blender", "--background", "--python", script],
        capture_output=True, text=True, timeout=300
    )

    # Read output from temp file
    output_path = "/tmp/output_result.json"
    if os.path.exists(output_path):
        with open(output_path, "r") as f:
            return json.load(f)

    return {
        "stdout": result.stdout[-2000:],
        "stderr": result.stderr[-2000:],
        "returncode": result.returncode
    }

runpod.serverless.start({"handler": handler})
```

Communication between handler.py (regular Python) and Blender scripts (Blender's Python) happens via temp files since they run in separate Python interpreters.

### 4.2 Write generate_character.py

**File to create**: `containers/blender/scripts/generate_character.py`

Accepts parameters via `/tmp/input_params.json`:

```json
{
  "action": "generate_character",
  "targets": {
    "macrodetails/Gender": 0.1,
    "macrodetails/Age": 0.4,
    "macrodetails/Weight": 0.3,
    "macrodetails/Muscle": 0.7
  },
  "assets": { "eyes": true, "teeth": true, "tongue": true, "eyelashes": true, "eyebrows": true },
  "rig": "mixamo",
  "shapeKeys": { "faceunits01": true, "visemes02": true },
  "b2": {
    "applicationKeyId": "...",
    "applicationKey": "...",
    "bucketName": "meshborn-avatars",
    "region": "us-east-005",
    "key": "avatars/{userId}/{avatarId}/avatar.glb"
  }
}
```

The script:
1. Creates a human via MPFB2
2. Applies body/face targets
3. Loads system assets
4. Adds Mixamo rig
5. Loads Faceunits 01 + Visemes 02
6. Exports GLB to `/tmp/output.glb`
7. Uploads GLB to B2 using boto3
8. Writes result to `/tmp/output_result.json`

Output:
```json
{
  "status": "success",
  "glbUrl": "https://meshborn-avatars.s3.us-east-005.backblazeb2.com/avatars/.../avatar.glb",
  "glbSizeBytes": 4521000,
  "morphTargets": 67,
  "morphTargetNames": ["viseme_aa", "..."],
  "boneCount": 65,
  "meshObjects": ["basemesh", "eyes", "teeth", "tongue", "eyelashes"]
}
```

### 4.3 Write validate.py

**File to create**: `containers/blender/scripts/validate.py`

Standalone validation that checks a GLB file:
- All 52 ARKit morph targets present with correct names
- All 15 Oculus visemes present with correct names
- Skeleton bone count and naming matches Mixamo convention
- Separate mesh objects exist (eyes, teeth, tongue)
- No degenerate geometry

Can be called independently via `action: "validate"` with a GLB path or URL as input.

### 4.4 Push to Quay.io and Deploy to RunPod

```bash
# Login
podman login quay.io

# Build final image
podman build -t quay.io/kokokino/meshborn-blender:latest \
  -f containers/blender/Containerfile .

# Push
podman push quay.io/kokokino/meshborn-blender:latest
```

In RunPod dashboard:
1. Serverless > Endpoints > Create
2. Container image: `quay.io/kokokino/meshborn-blender:latest`
3. GPU type: CPU-only (character generation is CPU-bound)
4. Workers: Min 0, Max 1 (scale to zero)
5. Note the endpoint ID — add to `settings.development.json` at `private.runpod.endpoints.blender`

---

## Step 5: End-to-End Integration Test

Depends on Steps 2, 3, and 4 all complete.

### 5.1 Create settings.development.json

Based on `settings.example.json` with real credentials for B2, RunPod, and Hub. This file is `.gitignore`d.

### 5.2 Start Meteor and Test

```bash
meteor npm install
meteor --port 3010 --settings settings.development.json
```

### 5.3 Run the Pipeline Smoke Test

Call the `pipeline.smokeTest` method from the browser console or a temporary UI button. Verify:

- [ ] Jobs document is created with status `queued`
- [ ] RunPod job is submitted, Jobs updates to `submitted`
- [ ] Jobs publication pushes real-time status updates to the client
- [ ] RunPod worker completes, GLB is uploaded to B2
- [ ] Avatars document is created with the B2 URL
- [ ] GLB is accessible at its B2 URL
- [ ] GLB can be downloaded and opened in a desktop 3D viewer

---

## Step 6: Texturing Spike

Depends on Step 4 (character generation working). Can run in parallel with Step 7.

### 6.1 Approach A: MPFB2 Procedural Materials

**File to create**: `containers/blender/scripts/texture_spike_a.py`

1. Generate a character (same as generate_character.py)
2. Use MPFB2's built-in material/skin system for procedural textures
3. Set skin tone, eye color, lip color via MPFB2 parameters
4. Render preview images (front + side, 512x512)
5. Export textured GLB
6. Upload preview images + GLB to B2

### 6.2 Approach B: Blender UV Projection

**File to create**: `containers/blender/scripts/texture_spike_b.py`

1. Generate a character
2. Load a test face image (bundled in container for simplicity)
3. Set up UV projection: camera facing mesh face, project image onto UV map
4. Bake projection to texture (2048x2048)
5. Render preview images
6. Export textured GLB
7. Upload to B2

### 6.3 Compare and Document

Run both spikes via RunPod (or locally via Podman). Compare:

| Criteria | Approach A | Approach B |
|----------|-----------|-----------|
| Visual quality | | |
| Character uniqueness | | |
| UV seam quality | | |
| Complexity | | |
| Complementary potential | | |

Document findings in `documentation/TEXTURE_SPIKE_RESULTS.md` with side-by-side screenshots, pros/cons, and the recommended strategy for Phase 2. Both approaches may end up complementary (MPFB2 materials for body, HiDream projection for face).

---

## Step 7: Minimal Babylon.js Viewer

Depends on Step 5 (GLB in B2). Can run in parallel with Step 6.

### 7.1 Add Babylon.js via CDN

**File to modify**: `client/main.html`

Add in `<head>`:
```html
<script src="https://cdn.babylonjs.com/babylon.js"></script>
<script src="https://cdn.babylonjs.com/loaders/babylonjs.loaders.min.js"></script>
```

CDN approach for Phase 0 keeps the Meteor bundle small. Switch to npm in Phase 3 if needed.

### 7.2 Write AvatarViewer Component

**File to create**: `imports/ui/components/AvatarViewer.js`

Mithril component following existing patterns (ChatRoom.js):
- Creates a `<canvas>` element
- Initializes Babylon.js scene in `oncreate` (needs DOM element)
- Loads GLB from `vnode.attrs.glbUrl`
- Sets up basic lighting and orbit camera controls
- Shows loading state while GLB loads, error state on failure
- Cleans up engine in `onremove`
- Calls `m.redraw()` after async loads

### 7.3 Add Temporary Viewer Route

**File to modify**: `client/main.js`

Add a `/viewer` route that renders AvatarViewer with a hardcoded B2 URL from the smoke test. Temporary — replaced by proper routes in Phase 3.

---

## Step 8: TalkingHead Verification

Depends on Step 5 (GLB produced). Manual step, no code to write.

### 8.1 Test in TalkingHead

1. Download the GLB from B2
2. Load it into TalkingHead's demo page (by Mika Suominen)
3. Verify:
   - 52 ARKit blend shapes are recognized
   - 15 Oculus visemes work for lip-sync
   - Mixamo rig drives body animations
   - Eyes, teeth, tongue are separate and controllable

### 8.2 If Lip-Sync Fails

Check:
- Morph target naming matches TalkingHead expectations exactly
- Viseme shape keys are on the correct mesh object (basemesh)
- Shape key ranges are 0-1
- If names differ slightly, add a rename mapping in `generate_character.py`'s export step

---

## File Manifest

### Files to Create (13)

| File | Purpose |
|------|---------|
| `containers/blender/Containerfile` | Blender 5 + MPFB2 + all asset packs |
| `containers/blender/handler.py` | RunPod serverless handler |
| `containers/blender/scripts/smoke_test.py` | MPFB2 headless validation (critical gate) |
| `containers/blender/scripts/generate_character.py` | Parametric character generation |
| `containers/blender/scripts/validate.py` | GLB morph target validation |
| `containers/blender/scripts/texture_spike_a.py` | MPFB2 procedural materials test |
| `containers/blender/scripts/texture_spike_b.py` | UV projection test |
| `imports/lib/storageClient.js` | S3Client init for B2 (backlog-beacon pattern) |
| `imports/lib/b2Storage.js` | B2 upload/download/delete helpers (backlog-beacon pattern) |
| `imports/lib/runpodClient.js` | RunPod Serverless API client (native fetch) |
| `server/migrations/2_create_avatars_jobs_indexes.js` | Database migration |
| `imports/ui/components/AvatarViewer.js` | Minimal Babylon.js GLB viewer |
| `documentation/TEXTURE_SPIKE_RESULTS.md` | Texturing spike findings (written after Step 6) |

### Files to Modify (8)

| File | Changes |
|------|---------|
| `imports/api/collections.js` | Add `Avatars` and `Jobs` exports |
| `server/main.js` | Import/call `initStorageClient()`, validate new settings |
| `server/indexes.js` | Add indexes for Avatars, Jobs |
| `server/publications.js` | Add `jobStatus` publication |
| `server/methods.js` | Add `pipeline.smokeTest` method |
| `server/migrations/0_steps.js` | Import migration 2 |
| `settings.example.json` | Add storage, runpod config sections |
| `package.json` | Add `@aws-sdk/client-s3`, `@aws-sdk/lib-storage` |
| `client/main.html` | Add Babylon.js CDN scripts |
| `client/main.js` | Add `/viewer` route |

### Reference Files (copy patterns from these)

| File | Pattern |
|------|---------|
| `backlog_beacon/server/covers/storageClient.js` | S3Client init for B2 |
| `backlog_beacon/server/covers/b2Storage.js` | B2 CRUD operations |
| `false-colors/imports/ai/llmProxy.js` | Native fetch with AbortController timeout |

---

## Verification Checklist

At the end of Phase 0, all of the following must be true:

**Infrastructure:**
- [ ] Podman installed and working locally
- [ ] Quay.io repository exists with `meshborn-blender:latest` image
- [ ] Backblaze B2 bucket `meshborn-avatars` exists and is accessible
- [ ] RunPod Serverless endpoint is live and responding

**MPFB2 Critical Gate:**
- [ ] MPFB2 creates a human headlessly in Blender 5 `--background` mode
- [ ] Body/face targets applied via shape key values
- [ ] System assets load (eyes, teeth, tongue, eyelashes, eyebrows)
- [ ] Mixamo rig applied
- [ ] All 52 ARKit shape keys (Faceunits 01) present on basemesh
- [ ] All 15 Oculus visemes (Visemes 02) present on basemesh
- [ ] GLB export preserves all morph targets

**End-to-End:**
- [ ] Calling RunPod endpoint from `localhost:3010` succeeds
- [ ] Generated GLB stored in B2 and accessible via URL
- [ ] Jobs publication provides real-time progress updates
- [ ] GLB loads in Babylon.js AvatarViewer in the browser
- [ ] GLB loads in TalkingHead and lip-sync works

**Texturing:**
- [ ] Spike A (MPFB2 materials) produces a textured character
- [ ] Spike B (UV projection) produces a textured character
- [ ] Clear texturing strategy documented for Phase 2

**Configuration:**
- [ ] All new settings documented in `settings.example.json`

---

## Risk Register

| Risk | Likelihood | Impact | Step | Mitigation |
|------|-----------|--------|------|------------|
| MPFB2 `bpy.ops` fails in `--background` | Low-Medium | Critical | 2.4 | Context overrides -> service bypass -> source patch -> Xvfb |
| Blender 5 tarball URL changed | Low | Low | 2.2 | Check blender.org downloads; adjust version string |
| MPFB2 asset pack URLs changed | Low | Low | 2.2 | Check makehumancommunity.org; may need git clone |
| MPFB2 data directory path differs | Medium | Low | 2.2 | Check MPFB2 docs for env var or config path |
| Faceunits 01 has <52 ARKit shapes | Very Low | High | 2.4 | Same author built TalkingHead; supplement missing if needed |
| Viseme naming mismatch with TalkingHead | Very Low | Low | 8.1 | Add rename step in export |
| RunPod cold start timeout | Low | Low | 5.3 | Increase polling timeout; configure FlashBoot |
| B2 upload from RunPod worker fails | Low | Medium | 4.2 | Fall back to returning GLB as base64 in RunPod output |
| Container image too large for RunPod | Low | Low | 4.4 | RunPod supports large images; longer first cold start |

---

*Last updated: 2026-03-17*
