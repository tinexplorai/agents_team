# PO Input

> **You drop reference docs here. The PO Agent reads them.**

Place anything that should inform user stories: PRDs, briefs, customer interviews, market research, competitor analysis, feature wishlists, support tickets — whatever you have.

The PO Agent reads everything in this folder + [`_input/1_project_description.md`](../1_project_description.md) before writing user stories.

## Supported formats

| Format | Status | Notes |
|--------|--------|-------|
| **Markdown (.md)** | Preferred | Easiest to read and reference. |
| **PDF** | Supported | Read tool handles up to 20 pages per call. |
| **Plain text (.txt)** | Supported | — |
| **PNG / JPG screenshots** | Supported | Useful for sketches, whiteboard photos, support-ticket screenshots. |
| **DOC / DOCX** | **Not supported** — convert to Markdown or PDF first. |
| **PPT / PPTX** | **Not supported** — re-export as PDF. |

## Suggested file naming

```
project_setup/step_2_requirements/
├── 01_brief.md              # the original product brief
├── 02_customer_interviews.md
├── 03_competitor_analysis.pdf
└── 04_support_pain_points.md
```

Number-prefix optional, but it gives the agent a reading order if there's a natural sequence.

## What the PO Agent does with these

1. Reads `_input/1_project_description.md` for the high-level "what + why".
2. Reads everything in this folder for context, tone, evidence, and edge cases.
3. Writes `project_code/documentation/user_stories.md` — user stories with acceptance criteria, citing this folder where relevant.

If this folder is empty, the agent writes user stories from `_input/1_project_description.md` alone.
