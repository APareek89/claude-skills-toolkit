# Known Failure Patterns

## Failure Categories

| Category | Code | Description | Retryable |
|----------|------|-------------|-----------|
| **Validation** | V | Input validation failed (bad params, missing fields) | No — fix input |
| **Billing** | B | Insufficient credits or auth failure | No — top up / fix token |
| **Generation** | G | Backend generation failed (model error, timeout) | Yes — retry with backoff |
| **Network** | N | Network timeout, 5xx errors | Yes — retry |

## Error → Category Mapping

| Error / Symptom | Category | Fix |
|-----------------|----------|-----|
| 400 Bad Request | V | Check `name` format (`pluginId_operationId`), validate prompt, check enums |
| 401 Unauthorized | B | Invalid or expired `PIXELBIN_API_TOKEN` |
| 429 Too Many Requests | N | Rate limited — wait `retry-after` seconds |
| 500 Internal Server Error | G | Backend issue — retry after 10s |
| 503 Service Unavailable | N | Service overloaded — retry after 30s |
| `status: "FAILURE"` | G | Check `result.error` — model-specific failure |
| "Insufficient credits" | B | No credits remaining — check billing |
| "imageValidation" in meta | V | Image URL inaccessible, wrong format, or exceeds limits |
| Timeout (no response) | N | Request took too long — try lower resolution or simpler prompt |
| "Cannot read property" | V | Missing `await` before async call |

## Validation Checklist

Before submitting a prediction, verify:

1. [ ] `name` is in `{pluginId}_{operationId}` format
2. [ ] `input.prompt` is a non-empty string
3. [ ] `input.images` is an array (not a string), max 14 items
4. [ ] `input.aspect_ratio` is a valid enum value (colon format, not slash)
5. [ ] `input.output_resolution` is one of: `0.5K`, `1K`, `2K`, `4K`
6. [ ] API token is set via environment variable (not hardcoded)
7. [ ] Response handling checks `result.status` before using `result.output`
8. [ ] Error handling covers: 429, 401, FAILURE status, timeout
9. [ ] `await` is used on all async calls

## Retry Strategy

```javascript
async function predictWithRetry(params, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const result = await pixelbin.predictions.createAndWait(params);
      if (result.status === "SUCCESS") return result;
      if (result.status === "FAILURE") {
        // Don't retry validation failures
        if (result.error?.includes("validation")) throw new Error(result.error);
        // Retry generation failures
        console.log(`Attempt ${i + 1} failed: ${result.error}. Retrying...`);
        await new Promise(r => setTimeout(r, (i + 1) * 5000));
        continue;
      }
    } catch (error) {
      if (error.response?.status === 429) {
        const wait = parseInt(error.response.headers["retry-after"] || "60");
        await new Promise(r => setTimeout(r, wait * 1000));
        continue;
      }
      throw error;
    }
  }
  throw new Error(`Failed after ${maxRetries} attempts`);
}
```
