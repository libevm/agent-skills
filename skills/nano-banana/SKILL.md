---
name: nano-banana
description: Generate and edit images via the Gemini CLI Nano Banana extension (text-to-image, edit, restore, icons, patterns, stories, diagrams).
allowed-tools: Bash(gemini:*)
---

# Nano Banana (Gemini CLI) — Image Generation Skill

## Use this whenever the user asks to create, generate, make, draw, design, visualize, or edit any image/visual.

Supported jobs (via Nano Banana extension):

* Text-to-image (`/generate`) ([GitHub][1])
* Image editing (`/edit`) ([GitHub][1])
* Photo restoration (`/restore`) ([GitHub][1])
* Icons (`/icon`) ([GitHub][1])
* Seamless patterns/textures (`/pattern`) ([GitHub][1])
* Sequential/story images (`/story`) ([GitHub][1])
* Technical diagrams (`/diagram`) ([GitHub][1])
* Natural language interface (`/nanobanana`) ([GitHub][1])

---

## One-time setup (verify before first use)

```bash
# 1) Verify extension
gemini extensions list | grep -i nanobanana

# 2) Install if missing
gemini extensions install https://github.com/gemini-cli-extensions/nanobanana

# 3) Verify API key (any ONE works; NANOBANANA_* preferred by the extension)
[ -n "$NANOBANANA_GEMINI_API_KEY" ] && echo "NANOBANANA_GEMINI_API_KEY set"
[ -n "$NANOBANANA_GOOGLE_API_KEY" ] && echo "NANOBANANA_GOOGLE_API_KEY set"
[ -n "$GEMINI_API_KEY" ] && echo "GEMINI_API_KEY set (fallback)"
[ -n "$GOOGLE_API_KEY" ] && echo "GOOGLE_API_KEY set (fallback)"
```

Prereqs: Gemini CLI, Node.js 20+ (project prereq), npm. ([GitHub][1])

---

## Model selection

Default: `gemini-2.5-flash-image` ([GitHub][1])

Optional (Nano Banana Pro):

```bash
export NANOBANANA_MODEL=gemini-3-pro-image-preview
```

([GitHub][1])

---

## Execution rule

Always run Gemini CLI commands with `--yolo` to auto-approve actions.

---

## Commands (use the most specific command that matches the request)

### `/generate` — text to image

Core options: `--count=1..8`, `--styles="a,b"`, `--variations="lighting,angle,..."`, `--format=grid|separate`, `--seed=123`, `--preview` ([GitHub][1])

```bash
gemini --yolo '/generate "sunset over mountains" --count=3 --preview'
gemini --yolo '/generate "mountain landscape" --styles="watercolor,oil-painting" --count=4'
gemini --yolo '/generate "coffee shop interior" --variations="lighting,mood" --preview'
```

Styles (built-in list includes): `photorealistic, watercolor, oil-painting, sketch, pixel-art, anime, vintage, modern, abstract, minimalist` ([GitHub][1])
Variations include: `lighting, angle, color-palette, composition, mood, season, time-of-day` ([GitHub][1])

---

### `/edit` — modify an existing image

```bash
gemini --yolo '/edit my_photo.png "add sunglasses to the person"'
gemini --yolo '/edit portrait.jpg "change background to a beach scene" --preview'
```

([GitHub][1])

---

### `/restore` — repair/enhance an old/damaged photo

```bash
gemini --yolo '/restore old_family_photo.jpg "remove scratches and improve clarity"'
gemini --yolo '/restore damaged_photo.png "enhance colors and fix tears" --preview'
```

([GitHub][1])

---

### `/icon` — app icons, favicons, UI elements

Options:

* `--sizes="16,32,64,128,256,512,1024"`
* `--type="app-icon|favicon|ui-element"`
* `--style="flat|skeuomorphic|minimal|modern"`
* `--format="png|jpeg"`
* `--background="transparent|white|black|color"`
* `--corners="rounded|sharp"` ([GitHub][1])

```bash
gemini --yolo '/icon "coffee cup logo" --sizes="64,128,256,512" --type="app-icon" --corners="rounded" --preview'
gemini --yolo '/icon "company logo" --type="favicon" --sizes="16,32,64"'
gemini --yolo '/icon "settings gear" --type="ui-element" --style="minimal" --background="transparent"'
```

---

### `/pattern` — seamless patterns & textures

Options: `--size="128x128|256x256|512x512"`, `--type="seamless|texture|wallpaper"`, `--style="geometric|organic|abstract|floral|tech"`, `--density="sparse|medium|dense"`, `--colors="mono|duotone|colorful"`, `--repeat="tile|mirror"` ([GitHub][1])

```bash
gemini --yolo '/pattern "subtle geometric hexagons" --type="seamless" --colors="duotone" --density="sparse" --preview'
gemini --yolo '/pattern "brushed metal surface" --type="texture" --style="tech" --colors="mono"'
gemini --yolo '/pattern "art deco design" --type="wallpaper" --style="geometric" --size="512x512"'
```

---

### `/story` — sequential images (process/tutorial/timeline/story)

Options: `--steps=2..8`, `--type="story|process|tutorial|timeline"`, `--style="consistent|evolving"`, `--layout="separate|grid|comic"`, `--transition="smooth|dramatic|fade"`, `--format="storyboard|individual"` ([GitHub][1])

```bash
gemini --yolo '/story "a seed growing into a tree" --steps=4 --type="process" --preview'
gemini --yolo '/story "git workflow tutorial" --steps=6 --type="tutorial" --layout="comic"'
gemini --yolo '/story "company logo evolution" --steps=4 --type="timeline" --transition="smooth"'
```

---

### `/diagram` — flowcharts, architecture, network, DB schemas, wireframes, mindmaps, sequence diagrams

Options:

* `--type="flowchart|architecture|network|database|wireframe|mindmap|sequence"`
* `--style="professional|clean|hand-drawn|technical"`
* `--layout="horizontal|vertical|hierarchical|circular"`
* `--complexity="simple|detailed|comprehensive"`
* `--colors="mono|accent|categorical"`
* `--annotations="minimal|detailed"` ([GitHub][1])

```bash
gemini --yolo '/diagram "CI/CD pipeline with testing stages" --type="flowchart" --complexity="detailed" --preview'
gemini --yolo '/diagram "microservices architecture for chat app" --type="architecture" --style="technical"'
gemini --yolo '/diagram "REST API authentication flow" --type="sequence" --layout="vertical"'
gemini --yolo '/diagram "e-commerce database schema" --type="database" --annotations="detailed"'
```

---

### `/nanobanana` — natural language interface (fallback)

Use when the user request doesn’t map cleanly to a single command. ([GitHub][1])

```bash
gemini --yolo '/nanobanana create a logo for my tech startup'
gemini --yolo '/nanobanana I need 5 different versions of a cat illustration in various art styles'
gemini --yolo '/nanobanana fix the lighting in sunset.jpg and make it more vibrant'
```

---

## Files: inputs, outputs, naming

* Output directory: `./nanobanana-output/` (auto-created). ([GitHub][1])
* Smart filenames derived from prompt; duplicates get `_1`, `_2`, etc. ([GitHub][1])
* When editing/restoring, input file search paths include:

  1. current dir, 2) `./images/`, 3) `./input/`, 4) `./nanobanana-output/`, 5) `~/Downloads/`, 6) `~/Desktop/` ([GitHub][1])

---

## Result handoff

After each run:

1. List outputs

```bash
ls -lt ./nanobanana-output/ | head
```

2. Present the newest file(s) to the user.
3. If they want “options,” rerun with `--count=3` (or more) and/or `--styles=`.

---

## Troubleshooting quick hits

* “Command not recognized”: restart Gemini CLI; ensure extension installed under `~/.gemini/extensions/`. ([GitHub][1])
* “No API key found”: set one of the supported env vars (prefer `NANOBANANA_GEMINI_API_KEY` / `NANOBANANA_GOOGLE_API_KEY`). ([GitHub][1])
* “Image not found”: ensure the file is in one of the search locations above. ([GitHub][1])

