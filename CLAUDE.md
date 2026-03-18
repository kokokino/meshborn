# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Meshborn** is a Kokokino spoke app that generates TalkingHead-ready 3D avatars from text prompts or images. Built on the spoke skeleton (SSO, subscriptions, chat already working), the avatar pipeline uses MPFB2 (MakeHuman Plugin For Blender 2) for parametric humanoid generation with ARKit blend shapes and Oculus visemes.

The Hub (kokokino.com) handles authentication and billing via Lemon Squeezy; this spoke validates SSO tokens and checks subscriptions via Hub API.

See `documentation/PLAN.md` for the full development plan and `documentation/PHASE_0.md` for current implementation phase.

## Commands

```bash
# Development (runs on port 3050)
meteor --port 3050 --settings settings.development.json

# Or use npm script (note: script may reference old port, use command above)
npm run dev

# Run tests once
npm test

# Run tests in watch mode
npm run test-app

# Analyze bundle size
npm run visualize

# Deploy to Meteor Galaxy
meteor deploy meshborn.kokokino.com --settings settings.production.json
```

## Tech Stack

| Technology | Purpose |
|------------|---------|
| **Meteor 3.4** | Real-time framework with MongoDB integration (requires Node.js 22.x) |
| **Mithril.js 2.3** | UI framework - uses JavaScript to generate HTML (no JSX) |
| **Pico CSS** | Classless CSS framework for minimal styling |
| **Babylon.js v8** | 3D GLB viewer with morph target support |
| **jsonwebtoken** | JWT validation for SSO tokens |
| **Podman** | Rootless container builds (drop-in Docker replacement) |
| **RunPod Serverless** | GPU/CPU compute for Blender + ML pipelines |
| **Backblaze B2** | S3-compatible file storage (via @aws-sdk/client-s3) |
| **MPFB2** | Parametric humanoid generation in Blender (server-side only, GPLv3) |
| **OpenRouter** | LLM routing for prompt decomposition |

## Architecture

### Avatar Pipeline (planned)
```
User prompt → LLM decomposition → MPFB2 params + HiDream prompt
    → RunPod: HiDream concept image → user approval
    → RunPod: MPFB2 character generation (Blender 5 headless)
    → RunPod: Texture projection → GLB export
    → Backblaze B2 storage → Babylon.js viewer
```

### SSO Flow
1. User clicks "Launch" in Hub → Hub generates RS256-signed JWT
2. User redirected to `/sso?token=<jwt>` on spoke
3. Spoke validates JWT signature with Hub's public key
4. Spoke calls Hub API for fresh user data
5. Spoke creates local Meteor session via custom `Accounts.registerLoginHandler`

### Key Directories
- `imports/hub/` - Hub integration (SSO handler, API client, subscription checking)
- `imports/ui/components/` - Mithril components including `RequireAuth` and `RequireSubscription` HOCs
- `imports/ui/pages/` - Route pages including `SsoCallback` for SSO handling
- `imports/lib/` - Shared server-side libraries (storage, RunPod client)
- `imports/api/collections.js` - All MongoDB collections in one file
- `server/accounts.js` - Custom login handler for SSO
- `server/methods.js` - Meteor methods
- `server/publications.js` - Data publications
- `containers/blender/` - Blender 5 + MPFB2 container and Python scripts
- `documentation/` - PLAN.md, PHASE_0.md, CONVENTIONS.md, HUB_SPOKE_STRATEGY.md

### Sibling Spoke Apps (reference patterns)
- `../backlog_beacon` - B2 storage pattern (`server/covers/storageClient.js`, `b2Storage.js`)
- `../false-colors` - OpenRouter LLM pattern (`imports/ai/llmProxy.js`)
- `../talon-and-lance` - Another spoke app

### Settings Structure
```json
{
  "public": {
    "appId": "meshborn",
    "hubUrl": "https://kokokino.com",
    "requiredProducts": ["base-monthly"]
  },
  "private": {
    "hubApiKey": "api-key-from-hub",
    "hubApiUrl": "https://kokokino.com/api/spoke",
    "hubPublicKey": "-----BEGIN PUBLIC KEY-----...",
    "storage": {
      "type": "b2",
      "b2": {
        "applicationKeyId": "...",
        "applicationKey": "...",
        "bucketName": "meshborn-avatars",
        "region": "us-east-005"
      }
    },
    "runpod": {
      "apiKey": "...",
      "endpoints": { "blender": "endpoint-id" }
    }
  }
}
```

## Code Conventions

### Meteor v3
- Use async/await patterns (no fibers) - e.g., `Meteor.users.findOneAsync()`, `insertAsync()`, `updateAsync()`
- Do not use `autopublish` or `insecure` packages
- When recommending Atmosphere packages, ensure Meteor v3 compatibility

### JavaScript Style
- Use `const` by default, `let` when needed, avoid `var`
- Always use curly braces with `if` blocks
- Avoid early returns - prefer single return statement at end
- Each variable declaration on its own line (no comma syntax)
- Use readable variable names (`document` not `doc`)

### UI Style
- Leverage Pico CSS patterns - avoid inline styles
- Use semantic CSS class names (`warning` not `yellow`)
- Use Mithril for UI; Blaze integration is acceptable for packages like accounts-ui
- Avoid React unless specifically instructed

### Security
- Validate all user input using `check()` from `meteor/check`
- Implement rate limiting on sensitive endpoints
- Never store Hub's private key in spoke code
- Sanitize user content before display to prevent XSS

## Patterns

### Mithril Components
Components are plain objects with lifecycle hooks:
- `oninit(vnode)` - Initialize state, start async operations
- `oncreate(vnode)` - Set up subscriptions, Tracker computations
- `onupdate(vnode)` - React to prop/state changes
- `onremove(vnode)` - Cleanup (stop computations, unsubscribe)
- `view(vnode)` - Return virtual DOM

State lives on `vnode.state`. Call `m.redraw()` after async operations complete.

### Meteor-Mithril Reactivity
The `MeteorWrapper` component in `client/main.js` bridges Meteor's reactivity with Mithril:
```javascript
Tracker.autorun(() => {
  Meteor.user(); Meteor.userId(); Meteor.loggingIn();
  m.redraw();
});
```

### Publications
- Always check `this.userId` before publishing sensitive data
- Return `this.ready()` for unauthenticated users
- Use field projections to limit exposed data

### Methods
- Use `check()` for input validation at method start
- Throw `Meteor.Error('error-code', 'message')` for client-handleable errors
- Common error codes: `not-authorized`, `not-found`, `invalid-message`, `subscription-required`

### Migrations
Uses `quave:migrations` package. Migrations in `server/migrations/` with `up()` and `down()` methods. Auto-run on startup via `Migrations.migrateTo('latest')`.

### Rate Limiting
Configure in `server/rateLimiting.js` using `DDPRateLimiter.addRule()`:
```javascript
DDPRateLimiter.addRule({ type: 'method', name: 'chat.send' }, 10, 10000);
```

### Testing
Run with `meteor test --driver-package meteortesting:mocha`. Tests use Mocha with Node.js assert. Server-only tests wrap in `if (Meteor.isServer)`.

### Database Indexes
Created in `server/indexes.js` during `Meteor.startup()`. Uses TTL indexes for automatic cleanup:
```javascript
collection.createIndexAsync({ createdAt: 1 }, { expireAfterSeconds: 600 });
```
Used for SSO nonces (replay attack prevention) and subscription cache.

### B2 Storage (follows backlog-beacon)
- `imports/lib/storageClient.js` - S3Client init with B2 endpoint
- `imports/lib/b2Storage.js` - upload/download/delete via @aws-sdk/client-s3
- Initialized in `server/main.js` at startup

### RunPod Integration
- `imports/lib/runpodClient.js` - Native `fetch()` to RunPod Serverless API
- Submit jobs, poll status, cancel — same endpoints for dev and prod
- Container scripts in `containers/blender/scripts/` run inside Blender's Python via subprocess
