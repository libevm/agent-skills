---
name: nano-banana
description: REQUIRED for image generation and conversational editing via the Gemini REST API. Handles text-to-image, image-to-image (editing), and multi-turn visual refinement. Use this skill when the user requests creation, modification, or design of any visual asset.
allowed-tools: Bash(curl, jq, base64)
---

# Nano Banana Image Skill (REST API)

Direct integration with Gemini's native image generation capabilities (Nano Banana).

## ⚠️ Environment & Dependency Check

Before execution, verify the environment. **If `curl` is missing, alert the user immediately.**

```bash
# 1. Check for curl (Critical)
if ! command -v curl &> /dev/null; then
    echo "ERROR: 'curl' is not installed. This skill requires curl for API communication."
    exit 1
fi

# 2. Check for API Key
[ -z "$GEMINI_API_KEY" ] && echo "ERROR: GEMINI_API_KEY is not set." && exit 1

```

## 🛠 Model Selection & Endpoints

| Model Name | API ID | Best For |
| --- | --- | --- |
| **Nano Banana** | `gemini-2.5-flash-image` | High-volume, low-latency, fast generation. |
| **Nano Banana Pro** | `gemini-3-pro-image-preview` | Professional assets, 4K, complex text rendering. |

**Base Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/{MODEL_ID}:generateContent?key=$GEMINI_API_KEY`

---

## 1. Image Generation (Text-to-Image)

Generates images based on a text description. The `responseModalities` MUST be set to `["IMAGE"]`.

### Example: Fancy Nano Banana Dish

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent?key=$GEMINI_API_KEY" \
-H "Content-Type: application/json" \
-d '{
  "contents": [{
    "parts": [{"text": "Create a picture of a nano banana dish in a fancy restaurant with a Gemini theme"}]
  }],
  "generationConfig": {
    "responseModalities": ["IMAGE"],
    "imageConfig": {
      "aspectRatio": "16:9",
      "imageSize": "1K"
    }
  }
}' | jq -r '.candidates[0].content.parts[0].inlineData.data' | base64 -d > nano_banana.png

```

---

## 2. Image Editing (Image-to-Image)

Modify an existing image by providing the original file and instructions.

### Example: Edit specific elements

```bash
# Encode original image
B64_IMAGE=$(base64 -w 0 input_image.png)

curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent?key=$GEMINI_API_KEY" \
-H "Content-Type: application/json" \
-d "{
  \"contents\": [{
    \"parts\": [
      { \"text\": \"Change the lighting to golden hour and add a cyberpunk neon sign in the background\" },
      { \"inlineData\": { \"mimeType\": \"image/png\", \"data\": \"$B64_IMAGE\" } }
    ]
  }],
  \"generationConfig\": { \"responseModalities\": [\"IMAGE\"] }
}" | jq -r '.candidates[0].content.parts[0].inlineData.data' | base64 -d > edited_banana.png

```

---

## 📑 API Reference: `generationConfig`

These parameters go inside the `generationConfig.imageConfig` block.

| Parameter | Options / Values | Description |
| --- | --- | --- |
| **aspectRatio** | `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9` | Default is `1:1`. Gemini 3 supports more variations. |
| **imageSize** | `1K`, `2K`, `4K` | Default `1K`. `4K` is available on Pro models only. |
| **personGeneration** | `dont_allow`, `allow_adult`, `allow_all` | `allow_adult` is default. `allow_all` includes children (restricted in some regions). |
| **candidateCount** | `1` to `4` (or `8`) | Number of variations to return. |
| **negativePrompt** | `string` | Describe what NOT to include in the image. |
| **outputMimeType** | `image/png`, `image/jpeg` | Format of the generated image. |

---

## 💡 Troubleshooting & Best Practices

* **Base64 extraction**: If `jq` is missing, use `sed 's/.*"data": "\(.*\)".*/\1/'` to extract the payload.
* **Request Size**: Inline data is limited to **20MB**. For larger images, use the Gemini Files API to upload first.
* **Prompting**: Be specific about medium (e.g., "oil painting", "DSLR photo"), lighting, and composition.
* **Refinement**: For multi-turn editing, always include the **most recent** generated image in the next request's `parts` array to maintain context.

