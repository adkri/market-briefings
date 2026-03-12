# Programmatic Excalidraw: Generate Interactive Diagrams from Code

> Generate fully interactive Excalidraw diagrams programmatically — from any language — and embed them in a single HTML file with zero build tools.

## The Pipeline

```
Python/Node/Any Language
        ↓
   .excalidraw JSON (just a dict/object)
        ↓
   Embed in HTML with CDN-loaded React + Excalidraw
        ↓
   Fully interactive canvas in any browser
```

No npm install. No build step. No server. One HTML file.

---

## Why This Matters

Most diagramming tools force a choice: static images (dead on arrival) or manual whiteboard drawing. This pipeline gives you **code-generated diagrams that are fully editable**. Generate architecture diagrams from your codebase, database schemas from migrations, or API flows from specs — all as interactive Excalidraw canvases that humans can then annotate and evolve.

---

## Part 1: The .excalidraw Format

An `.excalidraw` file is just JSON. Any language can generate it.

### Top-Level Structure

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [ ... ],
  "appState": {
    "gridSize": null,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
```

### Element Types

| Type | Use Case |
|------|----------|
| `rectangle` | Boxes, containers, process nodes |
| `ellipse` | Start/end nodes, circles |
| `diamond` | Decision nodes |
| `text` | Labels (standalone or bound to containers) |
| `arrow` | Connectors between shapes |
| `line` | Non-arrow connectors |

### Base Fields (Every Element Needs These)

```json
{
  "id": "unique-string",
  "type": "rectangle",
  "x": 100,
  "y": 200,
  "width": 200,
  "height": 60,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "#a5d8ff",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "roundness": { "type": 3 },
  "seed": 123456789,
  "version": 1,
  "isDeleted": false,
  "boundElements": null,
  "updated": 1,
  "link": null,
  "locked": false
}
```

### ⚠️ Forbidden Fields (cause rendering bugs)

Do NOT include these — they'll break things on excalidraw.com:
- `frameId`
- `index`
- `versionNonce`
- `rawText` (on text elements)

### Text Elements

Additional fields beyond the base:

```json
{
  "type": "text",
  "text": "Hello World",
  "originalText": "Hello World",
  "fontSize": 20,
  "fontFamily": 4,
  "textAlign": "center",
  "verticalAlign": "middle",
  "containerId": null,
  "autoResize": true,
  "lineHeight": 1.25
}
```

**Font families:** `1` = Virgil (hand-drawn), `4` = Assistant (clean sans-serif), `5` = Excalifont

### Putting Text Inside a Shape (Binding)

Container gets:
```json
"boundElements": [{ "id": "my-text-id", "type": "text" }]
```

Text gets:
```json
"containerId": "my-container-id"
```

### Arrow Elements

```json
{
  "type": "arrow",
  "points": [[0, 0], [200, 100]],
  "startArrowhead": null,
  "endArrowhead": "arrow",
  "startBinding": {
    "elementId": "source-shape-id",
    "focus": 0,
    "gap": 5
  },
  "endBinding": {
    "elementId": "target-shape-id",
    "focus": 0,
    "gap": 5
  }
}
```

Arrow `points` are relative to the arrow's `x, y` position.

### Style Reference

**Fill styles:** `"solid"`, `"hachure"` (hatched), `"cross-hatch"`, `"dots"`

**Stroke styles:** `"solid"`, `"dashed"`, `"dotted"`

**Roundness:** `{ "type": 1 }` sharp, `{ "type": 2 }` slight, `{ "type": 3 }` full (recommended)

**Excalidraw's built-in colors:**

| Color | Hex | Good For |
|-------|-----|----------|
| Light blue | `#a5d8ff` | State, data |
| Light green | `#b2f2bb` | Effects, success |
| Light pink | `#fcc2d7` | Refs, errors |
| Light orange | `#ffd8a8` | Performance, warnings |
| Light purple | `#d0bfff` | Render cycle, process |
| Light yellow | `#fff3bf` | Entry points, highlights |
| Light gray | `#e9ecef` | Internals, infrastructure |
| Light red | `#ffc9c9` | Danger, critical path |

---

## Part 2: Generating from Python

```python
import json
import random

def make_id(prefix="el"):
    return f"{prefix}_{random.randint(100000, 999999)}"

def rect(id, x, y, w, h, label, bg_color="#a5d8ff", font_size=16):
    """Create a rectangle with centered text label."""
    text_id = make_id("txt")
    return [
        {
            "id": id, "type": "rectangle",
            "x": x, "y": y, "width": w, "height": h,
            "angle": 0, "strokeColor": "#1e1e1e",
            "backgroundColor": bg_color, "fillStyle": "solid",
            "strokeWidth": 2, "strokeStyle": "solid",
            "roughness": 1, "opacity": 100,
            "groupIds": [], "roundness": {"type": 3},
            "seed": random.randint(1, 99999), "version": 1,
            "isDeleted": False, "boundElements": [{"id": text_id, "type": "text"}],
            "updated": 1, "link": None, "locked": False
        },
        {
            "id": text_id, "type": "text",
            "x": x, "y": y, "width": w, "height": h,
            "angle": 0, "strokeColor": "#1e1e1e",
            "backgroundColor": "transparent", "fillStyle": "solid",
            "strokeWidth": 1, "strokeStyle": "solid",
            "roughness": 1, "opacity": 100,
            "groupIds": [], "roundness": None,
            "seed": random.randint(1, 99999), "version": 1,
            "isDeleted": False, "boundElements": None,
            "updated": 1, "link": None, "locked": False,
            "text": label, "originalText": label,
            "fontSize": font_size, "fontFamily": 4,
            "textAlign": "center", "verticalAlign": "middle",
            "containerId": id, "autoResize": True, "lineHeight": 1.25
        }
    ]

def arrow(from_id, to_id, x, y, dx, dy, color="#1e1e1e", style="solid"):
    """Create an arrow between two elements."""
    return {
        "id": make_id("arr"), "type": "arrow",
        "x": x, "y": y, "width": abs(dx), "height": abs(dy),
        "angle": 0, "strokeColor": color,
        "backgroundColor": "transparent", "fillStyle": "solid",
        "strokeWidth": 2, "strokeStyle": style,
        "roughness": 1, "opacity": 100,
        "groupIds": [], "roundness": {"type": 2},
        "seed": random.randint(1, 99999), "version": 1,
        "isDeleted": False, "boundElements": [],
        "updated": 1, "link": None, "locked": False,
        "points": [[0, 0], [dx, dy]],
        "startArrowhead": None, "endArrowhead": "arrow",
        "startBinding": {"elementId": from_id, "focus": 0, "gap": 5},
        "endBinding": {"elementId": to_id, "focus": 0, "gap": 5}
    }

# Build a simple diagram
elements = []
elements.extend(rect("api", 100, 100, 200, 60, "API Gateway", "#a5d8ff"))
elements.extend(rect("auth", 100, 250, 200, 60, "Auth Service", "#b2f2bb"))
elements.extend(rect("db", 100, 400, 200, 60, "Database", "#ffd8a8"))
elements.append(arrow("api", "auth", 200, 160, 0, 90))
elements.append(arrow("auth", "db", 200, 310, 0, 90))

scene = {
    "type": "excalidraw",
    "version": 2,
    "source": "https://excalidraw.com",
    "elements": elements,
    "appState": {"gridSize": None, "viewBackgroundColor": "#ffffff"},
    "files": {}
}

with open("diagram.excalidraw", "w") as f:
    json.dump(scene, f, indent=2)
```

---

## Part 3: Embedding as Interactive HTML (The Breakthrough)

This is the key discovery. You can load the full Excalidraw editor from CDN and feed it your scene data — no build tools, no npm, just one HTML file.

Based on the [official Excalidraw browser integration docs](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/integration#browser).

### The Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>My Excalidraw Diagram</title>

<!-- Excalidraw CSS -->
<link rel="stylesheet"
  href="https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.css" />

<!-- Tell Excalidraw where to find fonts/assets -->
<script>
  window.EXCALIDRAW_ASSET_PATH =
    "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/prod/";
</script>

<!-- Import map for React + ReactDOM -->
<script type="importmap">
{
  "imports": {
    "react": "https://esm.sh/react@19.0.0",
    "react/jsx-runtime": "https://esm.sh/react@19.0.0/jsx-runtime",
    "react-dom": "https://esm.sh/react-dom@19.0.0",
    "react-dom/client": "https://esm.sh/react-dom@19.0.0/client"
  }
}
</script>

<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body { width: 100%; height: 100%; overflow: hidden; }
  #app { width: 100vw; height: 100vh; }
</style>
</head>
<body>
<div id="app"></div>

<script type="module">
import React from "react";
import { createRoot } from "react-dom/client";

// Load Excalidraw library
const ExcalidrawLib = await import(
  "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.js?external=react,react-dom"
);

// YOUR SCENE DATA — paste the .excalidraw JSON here
const sceneData = {
  "type": "excalidraw",
  "version": 2,
  "elements": [
    // ... your elements array ...
  ],
  "appState": {
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
};

const App = () => {
  return React.createElement(
    "div",
    { style: { width: "100vw", height: "100vh" } },
    React.createElement(ExcalidrawLib.Excalidraw, {
      initialData: {
        elements: sceneData.elements || [],
        appState: sceneData.appState || {},
        files: sceneData.files || {},
      },
    })
  );
};

const root = createRoot(document.getElementById("app"));
root.render(React.createElement(App));
</script>
</body>
</html>
```

### Key Details

1. **`react-dom/client`** — `createRoot` lives here, NOT in `react-dom`. This will bite you.
2. **`?external=react,react-dom`** — tells esm.sh to use YOUR React imports instead of bundling its own. Without this, you get duplicate React instances and everything breaks.
3. **`EXCALIDRAW_ASSET_PATH`** — must be set BEFORE the module loads. Excalidraw uses this to find its font files (Virgil, Excalifont, etc.).
4. **Light theme for pastel colors** — if you use Excalidraw's built-in pastel fills (`#a5d8ff`, `#b2f2bb`, etc.), use light theme. Dark theme turns pastels into mud.
5. **Import map** — required for the ESM imports to resolve correctly. `react/jsx-runtime` is needed by Excalidraw's internals.

### Python Generator Pattern

For diagrams with lots of data, write a Python script that builds the HTML:

```python
import json

# Generate your elements
elements = [...]  # your element-building code

scene = {
    "type": "excalidraw", "version": 2,
    "elements": elements,
    "appState": {"viewBackgroundColor": "#ffffff"},
    "files": {}
}

html = f'''<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>My Diagram</title>
<link rel="stylesheet"
  href="https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.css" />
<script>
  window.EXCALIDRAW_ASSET_PATH =
    "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/prod/";
</script>
<script type="importmap">
{{
  "imports": {{
    "react": "https://esm.sh/react@19.0.0",
    "react/jsx-runtime": "https://esm.sh/react@19.0.0/jsx-runtime",
    "react-dom": "https://esm.sh/react-dom@19.0.0",
    "react-dom/client": "https://esm.sh/react-dom@19.0.0/client"
  }}
}}
</script>
<style>
  * {{ margin: 0; padding: 0; box-sizing: border-box; }}
  html, body {{ width: 100%; height: 100%; overflow: hidden; }}
  #app {{ width: 100vw; height: 100vh; }}
</style>
</head>
<body>
<div id="app"></div>
<script type="module">
import React from "react";
import {{ createRoot }} from "react-dom/client";

const ExcalidrawLib = await import(
  "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.js?external=react,react-dom"
);

const sceneData = {json.dumps(scene)};

const App = () => {{
  return React.createElement(
    "div",
    {{ style: {{ width: "100vw", height: "100vh" }} }},
    React.createElement(ExcalidrawLib.Excalidraw, {{
      initialData: {{
        elements: sceneData.elements || [],
        appState: sceneData.appState || {{}},
        files: sceneData.files || {{}},
      }},
    }})
  );
}};

const root = createRoot(document.getElementById("app"));
root.render(React.createElement(App));
</script>
</body>
</html>'''

with open("interactive-diagram.html", "w") as f:
    f.write(html)
```

---

## Part 4: Alternative Approaches

### excalidraw-cli (npm) — Best for Flowcharts

Auto-layout via ELK.js. You describe the graph, it figures out positioning.

```bash
npm install -g excalidraw-cli

# Simple DSL
excalidraw-cli generate \
  -d '(Start) -> [Process] -> {Decision} -> (End)' \
  -o flow.excalidraw
```

DSL syntax:
- `(Label)` → ellipse
- `[Label]` → rectangle
- `{Label}` → diamond
- `->` solid arrow, `-->` dashed arrow
- `"label"` between arrows for edge labels

### Mermaid → Excalidraw

Official package from the Excalidraw team:

```bash
npm install @excalidraw/mermaid-to-excalidraw
```

Note: pulls in React as a dependency (heavy).

### Export to SVG/PNG

No pure server-side renderer exists. Options:

| Tool | Method | Weight |
|------|--------|--------|
| `excalidraw-brute-export-cli` | Playwright + Firefox | ~500MB, pixel-perfect |
| `excalidraw-to-svg` | jsdom | Lightweight, some rendering differences |
| `excalidraw_export` | jsdom + embedded fonts | Better font handling |

```bash
# Pixel-perfect export
npx excalidraw-brute-export-cli -i diagram.excalidraw --format svg -o output.svg

# Lightweight export
npx excalidraw-to-svg ./diagram.excalidraw ./output/
```

---

## Part 5: Gotchas & Tips

1. **Forbidden fields** — never include `frameId`, `index`, `versionNonce`, or `rawText`. They cause silent rendering bugs.

2. **Text sizing** — Excalidraw auto-sizes text containers. Set text `width`/`height` to match the container, and use `autoResize: true`.

3. **Arrow positioning** — arrow `x, y` is the start point. `points` are relative offsets from there. `points: [[0,0], [200, 100]]` draws from (x,y) to (x+200, y+100).

4. **Bindings are bidirectional** — if an arrow binds to a shape, the shape should list that arrow in `boundElements`. Missing either side = broken connections when editing.

5. **Seed values** — Excalidraw uses seeds for its hand-drawn rendering randomness. Different seeds = different wobble patterns. Use consistent seeds for consistent-looking diagrams.

6. **`roughness`** — `0` = clean lines, `1` = slight hand-drawn wobble (default), `2` = very sketchy. Use `0` for technical diagrams.

7. **Large diagrams** — for 50+ elements, always use the Python generator pattern. Don't try to hand-write the JSON.

8. **Dark mode** — if using dark theme, avoid pastel fills. Use darker, saturated colors instead. Or just use light theme where pastels shine.

---

## Quick Reference: Minimal Working Example

Save this as `test.excalidraw` and open at excalidraw.com:

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [
    {
      "id": "box1", "type": "rectangle",
      "x": 100, "y": 100, "width": 200, "height": 60,
      "angle": 0, "strokeColor": "#1e1e1e",
      "backgroundColor": "#a5d8ff", "fillStyle": "solid",
      "strokeWidth": 2, "strokeStyle": "solid",
      "roughness": 1, "opacity": 100, "groupIds": [],
      "roundness": {"type": 3}, "seed": 1, "version": 1,
      "isDeleted": false,
      "boundElements": [{"id": "label1", "type": "text"}],
      "updated": 1, "link": null, "locked": false
    },
    {
      "id": "label1", "type": "text",
      "x": 100, "y": 100, "width": 200, "height": 60,
      "angle": 0, "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent", "fillStyle": "solid",
      "strokeWidth": 1, "strokeStyle": "solid",
      "roughness": 1, "opacity": 100, "groupIds": [],
      "roundness": null, "seed": 2, "version": 1,
      "isDeleted": false, "boundElements": null,
      "updated": 1, "link": null, "locked": false,
      "text": "Hello Excalidraw",
      "originalText": "Hello Excalidraw",
      "fontSize": 20, "fontFamily": 4,
      "textAlign": "center", "verticalAlign": "middle",
      "containerId": "box1", "autoResize": true,
      "lineHeight": 1.25
    }
  ],
  "appState": {"gridSize": null, "viewBackgroundColor": "#ffffff"},
  "files": {}
}
```

---

*Generated from hands-on experimentation. Tested with Excalidraw 0.18.0, React 19, esm.sh CDN. March 2026.*
