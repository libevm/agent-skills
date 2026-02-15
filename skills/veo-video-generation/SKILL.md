---
name: veo-video-generation
description: Generate videos via Google Veo 3.1 REST API using curl (text-to-video, image-to-video, frame interpolation, reference-image-guided generation, video extension, aspect ratio/resolution/duration control, async polling, audio prompting, negative prompts, base64 encode/decode workflows). Always write downloaded videos into ./veo-output/.
allowed-tools: Bash(curl:*), Bash(jq:*), Bash(base64:*), Bash(ls:*), Bash(file:*), Bash(mkdir:*), Bash(sleep:*), Bash(cat:*), Bash(echo:*)
---

# Veo 3.1 Video Generation (REST API)

Generate high-fidelity 8-second 720p, 1080p, or 4K videos with natively generated audio using
Google's Veo 3.1 model through the Gemini REST API.

## Prerequisites

- A valid Gemini API key (set as `GEMINI_API_KEY` environment variable)
- `curl` and `jq` installed (for shell scripts) or equivalent HTTP client in your language
- Base URL: `https://generativelanguage.googleapis.com/v1beta`

## Quick Reference — Model IDs

| Model              | ID                                | Notes                        |
|--------------------|-----------------------------------|------------------------------|
| Veo 3.1            | `veo-3.1-generate-preview`        | Highest quality, all features|
| Veo 3.1 Fast       | `veo-3.1-fast-generate-preview`   | Faster, optimized for speed  |
| Veo 3 (legacy)     | `veo-3.0-generate-preview`        | Stable, no extension support |
| Veo 2 (legacy)     | `veo-2.0-generate-001`            | Silent video only, no audio  |

**Default to `veo-3.1-generate-preview`** unless the user explicitly requests a different model
or needs faster generation (use `veo-3.1-fast-generate-preview`).

---

## Core Workflow (All Requests)

Every Veo generation follows the same async pattern:

1. **Submit** a `POST` to `{BASE_URL}/models/{MODEL}:predictLongRunning`
2. **Receive** an operation name (long-running job ID)
3. **Poll** `{BASE_URL}/{operation_name}` until `done == true`
4. **Download** the video from the URI in the response

### Polling Template (Bash)

```bash
BASE_URL="https://generativelanguage.googleapis.com/v1beta"

# Step 1: Submit and capture operation name
operation_name=$(curl -s "${BASE_URL}/models/${MODEL}:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d "${REQUEST_BODY}" | jq -r .name)

# Step 2: Poll until done
while true; do
  status_response=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${operation_name}")
  is_done=$(echo "${status_response}" | jq .done)

  if [ "${is_done}" = "true" ]; then
    # Step 3: Extract video URI and download
    video_uri=$(echo "${status_response}" | jq -r \
      '.response.generateVideoResponse.generatedSamples[0].video.uri')
    curl -L -o output.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
    echo "Video saved to output.mp4"
    break
  fi

  echo "Waiting for generation..."
  sleep 10
done
```

### Polling Template (Python — using `requests`)

```python
import requests
import time
import json

BASE_URL = "https://generativelanguage.googleapis.com/v1beta"
API_KEY = os.environ["GEMINI_API_KEY"]
MODEL = "veo-3.1-generate-preview"

# Step 1: Submit
headers = {"x-goog-api-key": API_KEY, "Content-Type": "application/json"}
resp = requests.post(
    f"{BASE_URL}/models/{MODEL}:predictLongRunning",
    headers=headers,
    json=request_body,  # See sections below for body shapes
)
operation_name = resp.json()["name"]

# Step 2: Poll
while True:
    status = requests.get(
        f"{BASE_URL}/{operation_name}",
        headers={"x-goog-api-key": API_KEY},
    ).json()
    if status.get("done"):
        video_uri = (
            status["response"]["generateVideoResponse"]
            ["generatedSamples"][0]["video"]["uri"]
        )
        # Step 3: Download
        video_bytes = requests.get(
            video_uri,
            headers={"x-goog-api-key": API_KEY},
            allow_redirects=True,
        ).content
        with open("output.mp4", "wb") as f:
            f.write(video_bytes)
        print("Video saved to output.mp4")
        break
    time.sleep(10)
```

### Polling Template (Node.js — using `fetch`)

```javascript
const BASE_URL = "https://generativelanguage.googleapis.com/v1beta";
const API_KEY = process.env.GEMINI_API_KEY;
const MODEL = "veo-3.1-generate-preview";

// Step 1: Submit
const submitResp = await fetch(
  `${BASE_URL}/models/${MODEL}:predictLongRunning`,
  {
    method: "POST",
    headers: { "x-goog-api-key": API_KEY, "Content-Type": "application/json" },
    body: JSON.stringify(requestBody), // See sections below
  }
);
const { name: operationName } = await submitResp.json();

// Step 2: Poll
let videoUri;
while (true) {
  const statusResp = await fetch(`${BASE_URL}/${operationName}`, {
    headers: { "x-goog-api-key": API_KEY },
  });
  const status = await statusResp.json();
  if (status.done) {
    videoUri =
      status.response.generateVideoResponse.generatedSamples[0].video.uri;
    break;
  }
  await new Promise((r) => setTimeout(r, 10000));
}

// Step 3: Download
const videoResp = await fetch(videoUri, {
  headers: { "x-goog-api-key": API_KEY },
  redirect: "follow",
});
const fs = await import("fs");
fs.writeFileSync("output.mp4", Buffer.from(await videoResp.arrayBuffer()));
```

---

## 1. Text-to-Video

Generate a video from a text prompt alone.

### Request Body

```json
{
  "instances": [
    {
      "prompt": "A close up of two people staring at a cryptic drawing on a wall, torchlight flickering. A man murmurs, 'This must be it.' The woman whispers, 'What did you find?'"
    }
  ]
}
```

### With Parameters

```json
{
  "instances": [
    {
      "prompt": "Your detailed prompt here"
    }
  ],
  "parameters": {
    "aspectRatio": "16:9",
    "resolution": "720p",
    "negativePrompt": "cartoon, drawing, low quality",
    "durationSeconds": 8
  }
}
```

---

## 2. Image-to-Video

Animate a starting image into a video. Pass the image as base64 inline data inside the instance.

### Request Body

```json
{
  "instances": [
    {
      "prompt": "Panning wide shot of a calico kitten sleeping in the sunshine",
      "image": {
        "inlineData": {
          "mimeType": "image/png",
          "data": "<BASE64_ENCODED_IMAGE>"
        }
      }
    }
  ]
}
```

**Tip:** To base64-encode an image file in bash: `base64 -w0 image.png`

---

## 3. Frame Interpolation (First + Last Frame)

*Veo 3.1 only.* Specify both the starting and ending frames. The model generates a video
that transitions between them.

- The **first frame** goes in `instances[0].image`
- The **last frame** goes in `parameters.lastFrame`

### Request Body

```json
{
  "instances": [
    {
      "prompt": "A cinematic haunting video of a ghostly woman fading from a swing in the mist.",
      "image": {
        "inlineData": {
          "mimeType": "image/png",
          "data": "<FIRST_FRAME_BASE64>"
        }
      }
    }
  ],
  "parameters": {
    "lastFrame": {
      "inlineData": {
        "mimeType": "image/png",
        "data": "<LAST_FRAME_BASE64>"
      }
    }
  }
}
```

---

## 4. Reference Images (Style/Content Guidance)

*Veo 3.1 only.* Provide up to 3 reference images to guide the video's visual content.
Each reference uses `referenceType: "asset"`.

### Request Body

```json
{
  "instances": [
    {
      "prompt": "A woman in a flamingo dress walks through turquoise lagoon water..."
    }
  ],
  "parameters": {
    "referenceImages": [
      {
        "image": {
          "inlineData": {
            "mimeType": "image/png",
            "data": "<DRESS_IMAGE_BASE64>"
          }
        },
        "referenceType": "asset"
      },
      {
        "image": {
          "inlineData": {
            "mimeType": "image/png",
            "data": "<GLASSES_IMAGE_BASE64>"
          }
        },
        "referenceType": "asset"
      },
      {
        "image": {
          "inlineData": {
            "mimeType": "image/png",
            "data": "<PERSON_IMAGE_BASE64>"
          }
        },
        "referenceType": "asset"
      }
    ]
  }
}
```

**Constraints when using reference images:**
- Duration must be `8` seconds
- Maximum 3 reference images
- `personGeneration` is `"allow_adult"` only

---

## 5. Video Extension

*Veo 3.1 only.* Extend a previously Veo-generated video by ~7 seconds (up to 20 times,
max 148 seconds total). The input video must be from a previous Veo generation.

### Request Body

```json
{
  "instances": [
    {
      "prompt": "The butterfly lands on an orange origami flower. A puppy runs up and pats it.",
      "video": {
        "inlineData": {
          "mimeType": "video/mp4",
          "data": "<PREVIOUS_VIDEO_BASE64>"
        }
      }
    }
  ],
  "parameters": {
    "numberOfVideos": 1,
    "resolution": "720p"
  }
}
```

**Extension constraints:**
- Input must be a Veo-generated video (≤141 seconds)
- Resolution locked to **720p**
- Duration must be **8 seconds**
- Aspect ratio: `9:16` or `16:9`
- Videos expire after 2 days (timer resets on extension)
- You can also pass the video URI directly instead of base64 if you have it from
  the previous generation response

---

## All Parameters Reference

| Parameter          | Type     | Values                                                      | Default   |
|--------------------|----------|-------------------------------------------------------------|-----------|
| `prompt`           | string   | Text description (supports dialogue in quotes, SFX cues)    | required  |
| `negativePrompt`   | string   | What to exclude (use nouns, not "don't")                     | —         |
| `aspectRatio`      | string   | `"16:9"`, `"9:16"`                                          | `"16:9"`  |
| `resolution`       | string   | `"720p"`, `"1080p"`, `"4k"`                                 | `"720p"`  |
| `durationSeconds`  | int      | `4`, `6`, `8`                                                | `8`       |
| `personGeneration` | string   | `"allow_all"` (text-to-video), `"allow_adult"` (image-based)| varies    |
| `numberOfVideos`   | int      | `1`                                                          | `1`       |
| `seed`             | int      | Any integer (improves but doesn't guarantee determinism)     | —         |
| `referenceImages`  | array    | Up to 3 `VideoGenerationReferenceImage` objects              | —         |
| `lastFrame`        | object   | Image object for the ending frame (interpolation)            | —         |

### Resolution Constraints

- `1080p` and `4k` — only available with 8-second duration
- `4k` — higher latency and higher cost
- Video extension — locked to `720p`
- Reference images — duration must be `8` seconds

### Duration Constraints

| Feature                       | Allowed durations |
|-------------------------------|-------------------|
| Standard text-to-video        | 4, 6, 8 seconds   |
| 1080p or 4k resolution        | 8 seconds only    |
| Reference images              | 8 seconds only    |
| Video extension               | 8 seconds only    |

---

## Prompt Writing Guide

Write prompts that include these elements for best results:

1. **Subject** — The main focus (person, object, animal, scenery)
2. **Action** — What the subject does (walking, flying, melting)
3. **Style** — Visual aesthetic (cinematic, film noir, 3D cartoon, sci-fi)
4. **Camera** — Position and movement (aerial view, dolly shot, close-up, POV)
5. **Composition** — Shot framing (wide shot, two-shot, extreme close-up)
6. **Lens effects** — Depth of field (shallow focus, macro lens, wide-angle)
7. **Ambiance** — Mood via color and light (warm tones, blue tones, golden hour)

### Audio Prompting (Veo 3.1 generates audio natively)

- **Dialogue:** Use quotes — `A man murmurs, "This must be the key."`
- **Sound effects:** Describe explicitly — `tires screeching loudly, engine roaring`
- **Ambient noise:** Describe the soundscape — `a faint eerie hum in the background`

### Negative Prompts

- ❌ Don't use instructive language: ~~"No walls"~~
- ✅ Do list unwanted elements as nouns/adjectives: `"wall, frame, cartoon, blurry"`

---

## Response Structure

### Submit Response

```json
{
  "name": "operations/generate-video-abc123..."
}
```

### Poll Response (in progress)

```json
{
  "name": "operations/generate-video-abc123...",
  "done": false
}
```

### Poll Response (complete)

```json
{
  "name": "operations/generate-video-abc123...",
  "done": true,
  "response": {
    "generateVideoResponse": {
      "generatedSamples": [
        {
          "video": {
            "uri": "https://generativelanguage.googleapis.com/v1beta/files/...",
            "encoding": "video/mp4"
          }
        }
      ]
    }
  }
}
```

Download the video from the `uri` field. You **must** include the API key header:
```bash
curl -L -o video.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
```

---

## Error Handling

- **Safety filter blocks:** Veo may block generation due to safety filters or audio
  processing issues. You will not be charged. Retry with a revised prompt.
- **Polling timeout:** Latency ranges from ~11 seconds to ~6 minutes during peak hours.
  Set a reasonable timeout (e.g., 10 minutes) and implement exponential backoff if desired.
- **Video expiry:** Generated videos are stored for **2 days only**. Download promptly.
- **Region restrictions:** In EU/UK/CH/MENA, `personGeneration` is restricted:
  - Veo 3.1: `allow_adult` only
  - Veo 2: `dont_allow` (default) or `allow_adult`

---

## Watermarking

All Veo-generated videos are watermarked with **SynthID** for AI content identification.
This is automatic and cannot be disabled.

---

## Complete Examples

### Example: Portrait Video with Negative Prompt (Bash)

```bash
BASE_URL="https://generativelanguage.googleapis.com/v1beta"
MODEL="veo-3.1-generate-preview"

operation_name=$(curl -s "${BASE_URL}/models/${MODEL}:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "instances": [{
      "prompt": "A montage of pizza making: chef tossing dough, ladling sauce, sprinkling cheese, final bubbling pizza. Upbeat electronic music."
    }],
    "parameters": {
      "aspectRatio": "9:16",
      "negativePrompt": "blurry, low quality, distorted hands"
    }
  }' | jq -r .name)

while true; do
  status_response=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${operation_name}")
  is_done=$(echo "${status_response}" | jq .done)
  if [ "${is_done}" = "true" ]; then
    video_uri=$(echo "${status_response}" | jq -r \
      '.response.generateVideoResponse.generatedSamples[0].video.uri')
    curl -L -o pizza_portrait.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
    echo "Saved: pizza_portrait.mp4"
    break
  fi
  sleep 10
done
```

### Example: 4K Resolution Video (Bash)

```bash
BASE_URL="https://generativelanguage.googleapis.com/v1beta"
MODEL="veo-3.1-generate-preview"

operation_name=$(curl -s "${BASE_URL}/models/${MODEL}:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "instances": [{
      "prompt": "A stunning drone view of the Grand Canyon during sunset. The drone slowly flies towards the sun then dives into the canyon."
    }],
    "parameters": {
      "resolution": "4k"
    }
  }' | jq -r .name)

while true; do
  status_response=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${operation_name}")
  is_done=$(echo "${status_response}" | jq .done)
  if [ "${is_done}" = "true" ]; then
    video_uri=$(echo "${status_response}" | jq -r \
      '.response.generateVideoResponse.generatedSamples[0].video.uri')
    curl -L -o grand_canyon_4k.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
    echo "Saved: grand_canyon_4k.mp4"
    break
  fi
  sleep 10
done
```

### Example: Image-to-Video (Python)

```python
import requests, time, base64, os

BASE_URL = "https://generativelanguage.googleapis.com/v1beta"
API_KEY = os.environ["GEMINI_API_KEY"]
MODEL = "veo-3.1-generate-preview"
headers = {"x-goog-api-key": API_KEY, "Content-Type": "application/json"}

# Encode image
with open("start_frame.png", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

body = {
    "instances": [{
        "prompt": "Panning wide shot of a calico kitten waking up and stretching in sunshine",
        "image": {
            "inlineData": {"mimeType": "image/png", "data": img_b64}
        }
    }]
}

resp = requests.post(f"{BASE_URL}/models/{MODEL}:predictLongRunning", headers=headers, json=body)
op_name = resp.json()["name"]

while True:
    status = requests.get(f"{BASE_URL}/{op_name}", headers={"x-goog-api-key": API_KEY}).json()
    if status.get("done"):
        uri = status["response"]["generateVideoResponse"]["generatedSamples"][0]["video"]["uri"]
        video = requests.get(uri, headers={"x-goog-api-key": API_KEY}, allow_redirects=True).content
        with open("kitten_video.mp4", "wb") as f:
            f.write(video)
        print("Saved: kitten_video.mp4")
        break
    time.sleep(10)
```

### Example: Frame Interpolation (Python)

```python
import requests, time, base64, os

BASE_URL = "https://generativelanguage.googleapis.com/v1beta"
API_KEY = os.environ["GEMINI_API_KEY"]
MODEL = "veo-3.1-generate-preview"
headers = {"x-goog-api-key": API_KEY, "Content-Type": "application/json"}

with open("first_frame.png", "rb") as f:
    first_b64 = base64.b64encode(f.read()).decode()
with open("last_frame.png", "rb") as f:
    last_b64 = base64.b64encode(f.read()).decode()

body = {
    "instances": [{
        "prompt": "A smooth cinematic transition from the first scene to the last",
        "image": {"inlineData": {"mimeType": "image/png", "data": first_b64}}
    }],
    "parameters": {
        "lastFrame": {"inlineData": {"mimeType": "image/png", "data": last_b64}}
    }
}

resp = requests.post(f"{BASE_URL}/models/{MODEL}:predictLongRunning", headers=headers, json=body)
op_name = resp.json()["name"]

while True:
    status = requests.get(f"{BASE_URL}/{op_name}", headers={"x-goog-api-key": API_KEY}).json()
    if status.get("done"):
        uri = status["response"]["generateVideoResponse"]["generatedSamples"][0]["video"]["uri"]
        video = requests.get(uri, headers={"x-goog-api-key": API_KEY}, allow_redirects=True).content
        with open("interpolated.mp4", "wb") as f:
            f.write(video)
        print("Saved: interpolated.mp4")
        break
    time.sleep(10)
```

### Example: Video Extension (Bash)

```bash
BASE_URL="https://generativelanguage.googleapis.com/v1beta"
MODEL="veo-3.1-generate-preview"

# Assume previous_video_base64 is set from a prior generation
operation_name=$(curl -s "${BASE_URL}/models/${MODEL}:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "instances": [{
      "prompt": "Continue the scene as the camera slowly pulls back to reveal the full landscape.",
      "video": {"inlineData": {"mimeType": "video/mp4", "data": "'"$previous_video_base64"'"}}
    }],
    "parameters": {
      "numberOfVideos": 1,
      "resolution": "720p"
    }
  }' | jq -r .name)

while true; do
  status_response=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${operation_name}")
  is_done=$(echo "${status_response}" | jq .done)
  if [ "${is_done}" = "true" ]; then
    video_uri=$(echo "${status_response}" | jq -r \
      '.response.generateVideoResponse.generatedSamples[0].video.uri')
    curl -L -o extended_video.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
    echo "Saved: extended_video.mp4"
    break
  fi
  sleep 10
done
```

---

## Checklist Before Generating

1. ✅ `GEMINI_API_KEY` environment variable is set
2. ✅ Model ID is correct (`veo-3.1-generate-preview` for full features)
3. ✅ Prompt is descriptive (subject + action + style + camera at minimum)
4. ✅ If using images: base64-encoded with correct `mimeType`
5. ✅ If using 1080p/4k: `durationSeconds` is `8`
6. ✅ If extending video: resolution is `720p`, input is Veo-generated, ≤141s
7. ✅ If using reference images: max 3, duration is `8`
8. ✅ Polling loop has a sleep interval (10s recommended) and a timeout
9. ✅ Download uses `-L` flag (curl) or `allow_redirects=True` (requests) for redirects
10. ✅ Download includes the API key header
