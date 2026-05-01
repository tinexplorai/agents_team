# Design Input

> **You drop files here. The Designer Agent reads them.**

Place design references exported from your design tool (Claude design, Figma, Penpot, Sketch, etc.) into this folder. The Designer Agent reads everything here at the start of Phase 2.

## Supported formats

| Format | Status | Notes |
|--------|--------|-------|
| **PDF** | Preferred | Single file, multi-page; renders cleanly. |
| **PNG / JPG / WebP** | Supported | Use one image per screen/frame; name them descriptively (e.g. `01_login.png`, `02_dashboard.png`). |
| **PPT / PPTX** | **Not supported** — re-export as PDF | The agent cannot read PowerPoint directly. |
| **ZIP** | Unzip first | Drop the unzipped contents here, not the archive. |

## Naming tips (optional but helpful)

If you have multiple files, prefix with a number so the agent reads them in order:

```
docs/design_input/
├── 01_login.pdf
├── 02_dashboard.png
├── 03_settings.png
└── style_guide.pdf
```

## What the Designer Agent does with these

1. Reads every PDF / image in this folder.
2. Cross-references with `docs/user_stories.md` to map each screen to user stories.
3. Writes `docs/design_spec.md` — component list, layout, design tokens, interactions, accessibility notes.

If this folder is empty, the agent falls back to the Figma MCP (if configured) or writes a spec from user stories alone.
