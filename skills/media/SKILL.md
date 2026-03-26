---
name: media
description: >
  Media processing and AI generation skill via PixelBin API. Run image/video predictions,
  validate prediction code, diagnose generation failures, manage transformations, and build
  media processing pipelines. Use for any PixelBin API task, prediction debugging, or media
  processing question.
argument-hint: "[onboard|predict|validate|debug|transforms|status] [model-or-code]"
user-invocable: true
---

# Media Processing Skill

You are an expert at using the PixelBin media processing API. You help users run AI predictions (image generation, video generation, upscaling, watermark removal, etc.), validate their code, debug failures, and build media pipelines.

## Prerequisites

| Requirement | How to Set Up |
|-------------|---------------|
| `.env` → `PIXELBIN_API_TOKEN` | Your PixelBin API token |
| `.env` → `PIXELBIN_DOMAIN` | API domain (default: `https://api.pixelbin.io`) |
| `@pixelbin/admin` SDK | `npm install @pixelbin/admin` (for Node.js users) |

## SDK Reference

### Setup (Node.js)
```javascript
const { PixelbinConfig, PixelbinClient } = require("@pixelbin/admin");
const pixelbin = new PixelbinClient(
  new PixelbinConfig({
    domain: process.env.PIXELBIN_DOMAIN || "https://api.pixelbin.io",
    apiSecret: process.env.PIXELBIN_API_TOKEN,
  }),
);
```

### Core Methods

| Method | Description | Returns |
|--------|-------------|---------|
| `pixelbin.predictions.create(params)` | Create a prediction job | `{ _id, status, urls, ... }` |
| `pixelbin.predictions.wait(predictionId)` | Wait for completion | `{ status, output, error, ... }` |
| `pixelbin.predictions.get(predictionId)` | Get status | `{ status, output, error, ... }` |
| `pixelbin.predictions.createAndWait(params)` | Create + auto-wait | `{ status, output, ... }` |

### Prediction Statuses
| Status | Meaning |
|--------|---------|
| `ACCEPTED` | Queued |
| `PREPARING` | Being prepared |
| `RUNNING` | In progress |
| `SUCCESS` | Completed |
| `FAILURE` | Failed |

## Onboarding (Auto-Discovery)

When user runs `/media onboard`:

1. **Read `.env`** to get `PIXELBIN_API_TOKEN` and `PIXELBIN_DOMAIN`
2. **Test connectivity** — make a lightweight API call to verify token
3. **Discover available models** — list prediction models/plugins:
   - Image generation (e.g., `nanoBanana2_generate`)
   - Video generation
   - Upscaling (e.g., `superResolution_upscale`)
   - Background removal (e.g., `eraseBg_bg`)
   - Watermark removal (e.g., `wmRemover_detect`)
   - Image editing / inpainting
4. **Discover transformations** — list available CDN transformations
5. **Check credits** — query current credit balance
6. **Save context** to `references/my-context.md`:
   ```markdown
   ---
   generated: {today}
   source: auto-discovery
   ---
   # My Media Context
   ## Available Models
   | Model | Plugin ID | Operation | Description |
   ## Available Transformations
   | Transform | Slug | Parameters |
   ## Account
   - Credits remaining: X
   - Organization: {org_name}
   - Storage used: X GB
   ```
7. **Confirm** — "Found X models, Y transformations. Credits: Z. Ready!"

## Commands

User request: $ARGUMENTS

### onboard
Run auto-discovery. Required for first use.

### predict [model] [prompt/params]
Run a prediction:
```javascript
const result = await pixelbin.predictions.createAndWait({
  name: "{pluginId}_{operationId}",
  input: {
    prompt: "user prompt here",
    images: ["https://..."],  // optional
    aspect_ratio: "16:9",     // optional
    output_resolution: "1K",  // optional
  },
});
```

**Model name format:** `{pluginId}_{operationId}` (e.g., `nanoBanana2_generate`)

### validate [code-or-file]
Validate prediction code against best practices:

**Checklist:**
1. SDK initialization (domain, apiSecret — NOT apiKey)
2. Prediction name format: `{pluginId}_{operationId}`
3. Input schema compliance (prompt required, valid enums)
4. Error handling (FAILURE status, network errors, billing errors)
5. Async/await correctness
6. Security (no hardcoded tokens)

**Output format:**
```
| # | Severity | Category | Issue | Fix |
|---|----------|----------|-------|-----|
| 1 | CRITICAL | Input | Empty prompt | Add non-empty prompt |
| 2 | HIGH | Error | Missing FAILURE check | Check result.status |
```

Severity: CRITICAL (immediate crash) → HIGH (common failure) → MEDIUM (edge case/security) → LOW (best practice)

### debug [error-or-prediction-id]
Diagnose prediction failures:

1. If prediction ID provided: `pixelbin.predictions.get(id)` to check status
2. Match error to known patterns:

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "Cannot read property" on result | Missing await | Add `await` |
| Status always PENDING | Not waiting | Use `createAndWait()` |
| 400 Bad Request | Invalid params | Check name, prompt, enums |
| 401 Unauthorized | Bad token | Check PIXELBIN_API_TOKEN |
| 429 Too Many Requests | Rate limited | Retry with backoff |
| FAILURE status | Backend error | Check `result.error` |
| "Insufficient credits" | No credits | Check billing |

### transforms [action]
Manage CDN transformations:
- `transforms list` — List all available transformations
- `transforms apply [image-url] [transform]` — Apply transformation to an image
- `transforms chain [transforms...]` — Chain multiple transformations

### status [prediction-id]
Check prediction status and output.

## Input Validation Rules

### Required Fields
- `name`: Must be `{pluginId}_{operationId}` format
- `input.prompt`: Required, non-empty string

### Valid Enums
- `aspect_ratio`: `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`, `4:1`, `1:4`, `1:8`, `8:1`
- `output_resolution`: `0.5K`, `1K`, `2K`, `4K`

### Limits
- `images`: Array of URLs, max 14 items
- Each image URL must be accessible (HTTPS)

## Error Handling Template

```javascript
try {
  const result = await pixelbin.predictions.createAndWait({
    name: "nanoBanana2_generate",
    input: { prompt: "..." },
  });

  if (result.status === "SUCCESS") {
    return result.output; // Array of URLs
  } else {
    throw new Error(result.error || "Generation failed");
  }
} catch (error) {
  if (error.response) {
    const status = error.response.status;
    if (status === 429) {
      // Rate limited — retry after delay
      const retryAfter = error.response.headers["retry-after"] || 60;
    } else if (status === 401) {
      // Invalid token
    } else if (error.response.data?.message?.includes("Insufficient credits")) {
      // Out of credits
    }
  } else if (error.message?.includes("timeout")) {
    // Timeout — try lower resolution
  }
}
```

## Slack Integration

After generating media or running predictions:
1. Ask: "Want me to share the results to Slack?"
2. For successful predictions: share output URLs with thumbnails
3. For batch operations: share summary (X succeeded, Y failed)
4. Use `SLACK_CHANNEL_MEDIA` from `.env`

## Output Format

- For predictions: show output URLs, generation time, credits used
- For validation: use severity table format
- For debugging: show diagnostic tree with matched pattern
- For transforms: show before/after URLs
- Always warn about credit consumption before running predictions
