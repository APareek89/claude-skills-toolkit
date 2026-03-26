# Prediction Models Reference

This file is populated by the `/media onboard` command. Below is the template structure.

## Common Models

| Model Name | Plugin ID | Operation | Input | Output |
|------------|-----------|-----------|-------|--------|
| Image Generation | `nanoBanana2` | `generate` | prompt, images, aspect_ratio, output_resolution | Array of image URLs |
| Super Resolution | `superResolution` | `upscale` | image URL, scale factor | Upscaled image URL |
| Background Removal | `eraseBg` | `bg` | image URL | Image with transparent BG |
| Watermark Removal | `wmRemover` | `detect` | image URL | Clean image URL |

## Input Schema Reference

### Image Generation (`nanoBanana2_generate`)
```json
{
  "name": "nanoBanana2_generate",
  "input": {
    "prompt": "string (required, non-empty)",
    "images": ["string[] (optional, max 14 URLs)"],
    "aspect_ratio": "enum: 1:1|2:3|3:2|3:4|4:3|4:5|5:4|9:16|16:9|21:9|4:1|1:4|1:8|8:1",
    "output_resolution": "enum: 0.5K|1K|2K|4K"
  }
}
```

## After Onboarding

Run `/media onboard` to auto-discover your available models and populate this file with your specific account's capabilities.
