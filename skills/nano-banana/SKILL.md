---
name: nano-banana
description: Generate and edit images via Gemini “Nano Banana” native image models over the REST API using curl (text-to-image, image editing, multi-turn image editing, high-res up to 4K, reference images, grounding, prompting guides, base64 decode workflows). Always write decoded images into ./nanobanana-output/.
allowed-tools: Bash(curl:*), Bash(jq:*), Bash(base64:*), Bash(ls:*), Bash(file:*), Bash(sed:*), Bash(awk:*), Bash(mkdir:*)
--------------------------------------------------------------------------------------------------------------------------

# Nano Banana (Gemini REST API via `curl`) — Image Generation & Editing Skill

Use this whenever the user asks to create, generate, make, draw, design, visualize, or edit any image/visual.

Nano Banana = Gemini’s native image generation models:

* **gemini-2.5-flash-image** (fast, default)
* **gemini-3-pro-image-preview** (pro, supports **1K/2K/4K**, best for production)

All generated images include a **SynthID watermark**.

---

## One-time setup

```bash
export GEMINI_API_KEY="YOUR_KEY_HERE"
```

Base endpoint:

* `https://generativelanguage.googleapis.com/v1beta/models/<MODEL>:generateContent`

---

# Output directory rule (MANDATORY)

**All decoded images must be saved into:**

```bash
./nanobanana-output/
```

Always ensure it exists before decoding:

```bash
mkdir -p ./nanobanana-output
```

---

# Decoding images from `response.json` (REQUIRED)

Gemini returns images as base64 in:

* `.candidates[0].content.parts[].inline_data.data`

Example part:

```json
{
  "inline_data": {
    "mime_type": "image/png",
    "data": "BASE64..."
  }
}
```

You must decode into real files inside `./nanobanana-output/`.

---

## A) Decode the FIRST returned image → nanobanana-output

```bash
mkdir -p ./nanobanana-output

jq -r '
  .candidates[0].content.parts[]
  | select(.inline_data != null)
  | .inline_data.data
' response.json | head -n 1 | base64 -d > ./nanobanana-output/output.png
```

Verify:

```bash
file ./nanobanana-output/output.png
```

---

## B) Decode ALL returned images → nanobanana-output

```bash
mkdir -p ./nanobanana-output

jq -r '
  .candidates[0].content.parts[]
  | select(.inline_data != null)
  | .inline_data.data
' response.json \
| awk '{print NR ":" $0}' \
| while IFS=":" read -r idx b64; do
    echo "$b64" | base64 -d > "./nanobanana-output/image_${idx}.png"
  done
```

Outputs:

* `./nanobanana-output/image_1.png`
* `./nanobanana-output/image_2.png`
* …

---

## C) Preserve MIME type (PNG vs JPEG)

Check MIME type:

```bash
jq -r '
  .candidates[0].content.parts[]
  | select(.inline_data != null)
  | .inline_data.mime_type
' response.json
```

If JPEG:

```bash
jq -r '
  .candidates[0].content.parts[]
  | select(.inline_data != null)
  | .inline_data.data
' response.json | head -n 1 | base64 -d > ./nanobanana-output/output.jpg
```

---

## D) Save text + image outputs separately

Print any text parts:

```bash
jq -r '
  .candidates[0].content.parts[]
  | select(.text != null)
  | .text
' response.json
```

Decode images as above into `./nanobanana-output/`.

---

## E) One-liner: generate → decode directly into nanobanana-output

```bash
mkdir -p ./nanobanana-output

curl -s -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{"text": "A photorealistic nano banana dessert on a marble table"}]
    }],
    "generationConfig": {
      "responseModalities": ["TEXT","IMAGE"]
    }
  }' \
| tee response.json \
| jq -r '.candidates[0].content.parts[] | select(.inline_data!=null) | .inline_data.data' \
| head -n 1 \
| base64 -d > ./nanobanana-output/output.png
```

---

# Image Generation & Editing

## 1) Text-to-image

```bash
curl -s -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{"text": "Create a nano banana dish in a futuristic restaurant"}]
    }],
    "generationConfig": {
      "responseModalities": ["TEXT","IMAGE"],
      "imageConfig": { "aspectRatio": "1:1" }
    }
  }' > response.json
```

Decode:

```bash
mkdir -p ./nanobanana-output

jq -r '.candidates[0].content.parts[] | select(.inline_data!=null) | .inline_data.data' \
  response.json | base64 -d > ./nanobanana-output/generated.png
```

---

## 2) Image editing (text + image → image)

```bash
BASE64_IMG="$(base64 -w 0 < cat.jpg)"

curl -s -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"contents\": [{
      \"parts\": [
        {\"text\": \"Add sunglasses to the cat. Keep lighting realistic.\"},
        {\"inline_data\": {\"mime_type\": \"image/jpeg\", \"data\": \"$BASE64_IMG\"}}
      ]
    }],
    \"generationConfig\": {
      \"responseModalities\": [\"TEXT\",\"IMAGE\"]
    }
  }" > response.json
```

Decode into output directory:

```bash
mkdir -p ./nanobanana-output

jq -r '.candidates[0].content.parts[] | select(.inline_data!=null) | .inline_data.data' \
  response.json | base64 -d > ./nanobanana-output/edited.jpg
```

---

## 3) Multi-turn editing

Always replay prior turns and decode intermediate outputs into:

* `./nanobanana-output/step1.png`
* `./nanobanana-output/step2.png`

---

## 4) High-res up to 4K (Gemini 3 Pro)

```bash
curl -s -X POST \
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{"text": "A premium hero shot of a smart speaker on white background"}]
    }],
    "generationConfig": {
      "responseModalities": ["TEXT","IMAGE"],
      "imageConfig": {
        "aspectRatio": "16:9",
        "imageSize": "4K"
      }
    }
  }' > response.json
```

Decode:

```bash
mkdir -p ./nanobanana-output

jq -r '.candidates[0].content.parts[] | select(.inline_data!=null) | .inline_data.data' \
  response.json | base64 -d > ./nanobanana-output/hero_4k.png
```

---

## Result handoff (after each run)

List newest outputs:

```bash
ls -lt ./nanobanana-output | head
```

---

## Troubleshooting decode issues

* Empty output → no inline_data returned; check `responseModalities`.
* Invalid base64 → ensure `jq -r` raw output.
* Wrong file type → inspect `.inline_data.mime_type`.

