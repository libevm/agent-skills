---
name: veo-video-generation
description: This skill teaches agents how to generate videos with **Veo 3.1** using the **Gemini API** with an emphasis on **REST** and **Python**. It covers: request/response shapes, long-running operations (polling), downloading results, and the major Veo controls (aspect ratio, resolution, duration, negative prompts, reference images, first/last frame interpolation, and video extension). :contentReference[oaicite:0]{index=0}
allowed-tools: Bash(curl:*), Bash(jq:*), Bash(base64:*), Bash(ls:*), Bash(file:*), Bash(mkdir:*), Bash(sleep:*), Bash(cat:*), Bash(echo:*)
---

## 1) What Veo 3.1 does

- Generates **short videos with native audio** via Gemini API. :contentReference[oaicite:1]{index=1}
- Supports:
  - Text → Video
  - Image → Video (input image becomes first frame)
  - First + last frame interpolation
  - Reference images (up to 3) to preserve a subject’s appearance
  - Extending a previously generated Veo video :contentReference[oaicite:2]{index=2}

### Model IDs (Gemini API)
- `veo-3.1-generate-preview`
- `veo-3.1-fast-generate-preview` :contentReference[oaicite:3]{index=3}

---

## 2) Authentication and base URL (REST)

### Base URL
- `https://generativelanguage.googleapis.com/v1beta` :contentReference[oaicite:4]{index=4}

### API Key header
Use header:
- `x-goog-api-key: $GEMINI_API_KEY` :contentReference[oaicite:5]{index=5}

Recommended environment variable:
- `GEMINI_API_KEY`

---

## 3) Core REST flow (long-running operation)

Veo video generation is asynchronous:
1. **POST** a generation request → receive an **operation** with a `name`.
2. **GET** the operation until `done: true`.
3. Extract the output video download URI from the final response.
4. **GET** the URI (include API key header) and follow redirects to download. :contentReference[oaicite:6]{index=6}

### Endpoint (start generation)
`POST /v1beta/models/{model}:predictLongRunning` :contentReference[oaicite:7]{index=7}

Example:
`POST https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning`

### Endpoint (poll operation)
`GET /v1beta/{operation_name}` :contentReference[oaicite:8]{index=8}

### Where the output URI appears (REST)
When done, the REST examples show the download URI at:
- `.response.generateVideoResponse.generatedSamples[0].video.uri` :contentReference[oaicite:9]{index=9}

---

## 4) REST request body schema (practical)

### Top-level shape
```json
{
  "instances": [
    {
      "prompt": "..."
      // optionally: "image": { ... } OR "video": { ... }
    }
  ],
  "parameters": {
    // optional controls (aspectRatio, resolution, durationSeconds, etc.)
  }
}
``` :contentReference[oaicite:10]{index=10}

### Media inputs (bytesBase64Encoded) — CRITICAL
**`inlineData` and `fileUri` do NOT work** with `veo-3.1-generate-preview` (returns 400).
Use `bytesBase64Encoded` format instead:
```json
{
  "bytesBase64Encoded": "<BASE64_BYTES>",
  "mimeType": "image/png"
}
```

For video extension input:
```json
{
  "bytesBase64Encoded": "<BASE64_BYTES>",
  "mimeType": "video/mp4"
}
```

> **Do NOT use** `{"inlineData": {"mimeType": "...", "data": "..."}}` — it is rejected by the API.
> **Do NOT use** `{"fileUri": "...", "mimeType": "..."}` — it is also rejected by the API.

### Parameter value types — CRITICAL
- `durationSeconds` MUST be a **number** (e.g. `4`), **NOT a string** (e.g. `"4"`). Passing a string returns 400.
- `aspectRatio`, `resolution`, `negativePrompt`, `personGeneration` are **strings**.

---

## 5) Supported Veo parameters (Gemini API)

In REST, these live under `"parameters": { ... }`. :contentReference[oaicite:12]{index=12}

### Common controls
- `prompt` (text description; can include audio cues) :contentReference[oaicite:13]{index=13}
- `negativePrompt` (what to avoid) :contentReference[oaicite:14]{index=14}
- `aspectRatio`: `"16:9"` (default) or `"9:16"` :contentReference[oaicite:15]{index=15}
- `resolution`: `"720p"` (default), `"1080p"` (8s only), `"4k"` (8s only)
  - Extension limited to **720p** :contentReference[oaicite:16]{index=16}
- `durationSeconds`: `4`, `6`, `8` — **MUST be a number, NOT a string** (passing `"4"` returns 400: `"The value type for durationSeconds needs to be a number"`)
  - Must be `8` when using extension, reference images, or 1080p/4k :contentReference[oaicite:17]{index=17}
- `personGeneration` (region-restricted behavior described in docs)
  - Veo 3.1: text-to-video & extension `"allow_all"` only; image-to-video & interpolation & reference images `"allow_adult"` only :contentReference[oaicite:18]{index=18}
- `seed` exists for Veo 3 models (does not guarantee determinism) :contentReference[oaicite:19]{index=19}

### Image / interpolation / reference / extension inputs
- `image`: initial image to animate (also used as first frame) :contentReference[oaicite:20]{index=20}
- `lastFrame`: final image (must be combined with `image`) :contentReference[oaicite:21]{index=21}
- `referenceImages`: **NOT supported by `veo-3.1-generate-preview`** (returns 400 error despite docs suggesting otherwise). Use image-to-video instead to preserve a subject's appearance. :contentReference[oaicite:22]{index=22}
- `video`: input video for extension (must be from a previous Veo generation) :contentReference[oaicite:23]{index=23}

---

## 6) Feature-specific REST patterns

### A) Text → Video (minimal)
`instances[0].prompt` only. :contentReference[oaicite:24]{index=24}

### B) Aspect ratio
Set `parameters.aspectRatio` to `"9:16"` for portrait. :contentReference[oaicite:25]{index=25}

### C) Resolution
Set `parameters.resolution` to `"4k"` (or `"1080p"` / `"720p"`). :contentReference[oaicite:26]{index=26}

### D) Image → Video
Provide `instances[0].image` using `bytesBase64Encoded` format; image becomes the starting frame.
**Do NOT use `inlineData` or `fileUri`** — they return 400 errors.
```json
"image": {
  "bytesBase64Encoded": "<BASE64_BYTES>",
  "mimeType": "image/png"
}
```
:contentReference[oaicite:27]{index=27}

### E) First + last frame interpolation
Provide:
- `instances[0].image` (first frame)
- `parameters.lastFrame` (last frame) :contentReference[oaicite:28]{index=28}

### F) Reference images — NOT WORKING
**`referenceImages` is NOT supported by `veo-3.1-generate-preview`** (returns 400:
`"referenceImages isn't supported by this model"`).
As a workaround, use **image-to-video** (Section 6D) to provide the subject as the first frame
and describe the desired animation in the prompt. :contentReference[oaicite:29]{index=29}

### G) Video extension (Veo 3.1 only)
- Input must be a **Veo-generated video**
- Extension can add **7 seconds** and can be repeated up to **20 times** (per docs)
- Input limits include: 720p, 9:16 or 16:9, length ≤ 141s, and “recent within 2 days” storage window noted in docs :contentReference[oaicite:30]{index=30}

REST: include `instances[0].video` using `bytesBase64Encoded` format and typically set `parameters.resolution` to `"720p"`. :contentReference[oaicite:31]{index=31}

---

## 7) Python (REST-first) reference implementation

This implementation:
- Starts a video job via REST
- Polls the long-running operation
- Downloads the resulting MP4

### Dependencies
- Python 3.9+
- `requests`

### Code
```python
import base64
import os
import time
from typing import Any, Dict, Optional

import requests


BASE_URL = "https://generativelanguage.googleapis.com/v1beta"


def b64_file(path: str) -> str:
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")


def gemini_headers() -> Dict[str, str]:
    api_key = os.environ.get("GEMINI_API_KEY")
    if not api_key:
        raise RuntimeError("Missing GEMINI_API_KEY env var")
    return {
        "x-goog-api-key": api_key,
        "Content-Type": "application/json",
    }


def start_predict_long_running(model: str, body: Dict[str, Any]) -> str:
    url = f"{BASE_URL}/models/{model}:predictLongRunning"
    r = requests.post(url, headers=gemini_headers(), json=body, timeout=60)
    r.raise_for_status()
    op = r.json()
    # Operation name is used for polling:
    # GET {BASE_URL}/{op['name']}
    return op["name"]


def poll_operation(op_name: str, poll_seconds: int = 10, timeout_seconds: int = 900) -> Dict[str, Any]:
    url = f"{BASE_URL}/{op_name}"
    start = time.time()
    while True:
        r = requests.get(url, headers={"x-goog-api-key": gemini_headers()["x-goog-api-key"]}, timeout=60)
        r.raise_for_status()
        op = r.json()
        if op.get("done") is True:
            return op
        if time.time() - start > timeout_seconds:
            raise TimeoutError(f"Operation not done after {timeout_seconds}s: {op_name}")
        time.sleep(poll_seconds)


def download_video_uri(video_uri: str, out_path: str) -> None:
    # The docs show downloading the uri with x-goog-api-key and following redirects. :contentReference[oaicite:32]{index=32}
    r = requests.get(
        video_uri,
        headers={"x-goog-api-key": gemini_headers()["x-goog-api-key"]},
        stream=True,
        allow_redirects=True,
        timeout=120,
    )
    r.raise_for_status()
    with open(out_path, "wb") as f:
        for chunk in r.iter_content(chunk_size=1024 * 1024):
            if chunk:
                f.write(chunk)


def extract_video_uri(operation_json: Dict[str, Any]) -> str:
    # REST examples extract:
    # .response.generateVideoResponse.generatedSamples[0].video.uri :contentReference[oaicite:33]{index=33}
    return operation_json["response"]["generateVideoResponse"]["generatedSamples"][0]["video"]["uri"]


def generate_text_to_video(
    prompt: str,
    model: str = "veo-3.1-generate-preview",
    *,
    aspect_ratio: Optional[str] = None,
    resolution: Optional[str] = None,
    duration_seconds: Optional[int] = None,
    negative_prompt: Optional[str] = None,
) -> str:
    parameters: Dict[str, Any] = {}
    if aspect_ratio:
        parameters["aspectRatio"] = aspect_ratio
    if resolution:
        parameters["resolution"] = resolution
    if duration_seconds:
        parameters["durationSeconds"] = duration_seconds
    if negative_prompt:
        parameters["negativePrompt"] = negative_prompt

    body: Dict[str, Any] = {"instances": [{"prompt": prompt}]}
    if parameters:
        body["parameters"] = parameters

    op_name = start_predict_long_running(model, body)
    op = poll_operation(op_name)
    video_uri = extract_video_uri(op)
    return video_uri


def generate_image_to_video(
    prompt: str,
    image_path: str,
    model: str = "veo-3.1-generate-preview",
    *,
    aspect_ratio: Optional[str] = None,
    negative_prompt: Optional[str] = None,
) -> str:
    image_b64 = b64_file(image_path)
    # IMPORTANT: use bytesBase64Encoded, NOT inlineData (which returns 400)
    body: Dict[str, Any] = {
        "instances": [
            {
                "prompt": prompt,
                "image": {"bytesBase64Encoded": image_b64, "mimeType": "image/png"},
            }
        ],
        "parameters": {
            "personGeneration": "allow_adult",
        },
    }
    if aspect_ratio:
        body["parameters"]["aspectRatio"] = aspect_ratio
    if negative_prompt:
        body["parameters"]["negativePrompt"] = negative_prompt
    op_name = start_predict_long_running(model, body)
    op = poll_operation(op_name)
    return extract_video_uri(op)


if __name__ == "__main__":
    # Example: portrait 9:16 + 4k (note: 4k implies 8s in docs). :contentReference[oaicite:34]{index=34}
    prompt = (
        "A montage of pizza making: a chef tossing dough, ladling sauce, sprinkling cheese, "
        "final shot of bubbling pizza; upbeat electronic music; high-energy professional video."
    )
    uri = generate_text_to_video(prompt, aspect_ratio="9:16", resolution="4k", negative_prompt="cartoon, drawing, low quality")
    print("Video URI:", uri)

    download_video_uri(uri, "output.mp4")
    print("Saved output.mp4")
````

---

## 8) Canonical curl snippets (REST)

### Start generation (example with aspect ratio)

````bash
BASE_URL="https://generativelanguage.googleapis.com/v1beta"

operation_name=$(
  curl -s "${BASE_URL}/models/veo-3.1-generate-preview:predictLongRunning" \
    -H "x-goog-api-key: $GEMINI_API_KEY" \
    -H "Content-Type: application/json" \
    -X POST \
    -d '{
      "instances": [{"prompt": "A cinematic shot of a majestic lion in the savannah."}],
      "parameters": {"aspectRatio": "9:16"}
    }' | jq -r .name
)
echo "Operation: $operation_name"
``` :contentReference[oaicite:35]{index=35}

### Poll until done and download
```bash
while true; do
  status_response=$(curl -s -H "x-goog-api-key: $GEMINI_API_KEY" "${BASE_URL}/${operation_name}")
  is_done=$(echo "${status_response}" | jq .done)
  if [ "${is_done}" = "true" ]; then
    video_uri=$(echo "${status_response}" | jq -r '.response.generateVideoResponse.generatedSamples[0].video.uri')
    curl -L -o output.mp4 -H "x-goog-api-key: $GEMINI_API_KEY" "${video_uri}"
    break
  fi
  sleep 10
done
``` :contentReference[oaicite:36]{index=36}

---

## 9) Operational guidance (what Codex should do)

### A) Always treat video generation as async
- Capture the operation `name`, poll `GET /v1beta/{name}` until `done: true`. :contentReference[oaicite:37]{index=37}

### B) Download via URI with API key header
- The returned `video.uri` is downloaded with `x-goog-api-key` and redirects enabled. :contentReference[oaicite:38]{index=38}

### C) Respect constraints
- 1080p/4k imply 8s videos; extension is 720p only. :contentReference[oaicite:39]{index=39}
- `referenceImages` does NOT work with `veo-3.1-generate-preview` (400 error). Use image-to-video instead.
- `inlineData` and `fileUri` do NOT work for media inputs. Always use `bytesBase64Encoded` format.
- Image-to-video requires `"personGeneration": "allow_adult"` in parameters. :contentReference[oaicite:40]{index=40}

### D) Extension constraints (strict)
- Only Veo-generated videos; extension window and length/format limits described in docs (720p, ≤141s, etc.). :contentReference[oaicite:41]{index=41}

---

## 10) Safety and blocked generations

- Generated videos are passed through safety filters and checks; prompts that violate terms/guidelines are blocked. :contentReference[oaicite:42]{index=42}
- Docs note videos may be blocked due to safety filters or audio processing issues, and you are not charged if blocked. :contentReference[oaicite:43]{index=43}

---

## 11) Related docs (for continued maintenance)

- Veo 3.1 video generation docs (this guide): :contentReference[oaicite:44]{index=44}
- Gemini API rate limits: :contentReference[oaicite:45]{index=45}
- Gemini API pricing: :contentReference[oaicite:46]{index=46}
- Gemini model/version semantics (stable vs preview): :contentReference[oaicite:47]{index=47}
````

