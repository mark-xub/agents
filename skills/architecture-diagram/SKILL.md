---
name: architecture-diagram
description: Create clear conceptual, business, and system architecture diagrams as editable draw.io files, applying layered structure, semantic color encoding, a narrative reading spine, and feedback loops. Use when the user asks to draw or design an architecture diagram, concept map, business architecture, capability map, system topology, operating-model diagram, or mentions 架构图 / 概念架构 / 业务架构 / 分层架构 / drawio.
---

# Architecture Diagram

Produce professional, readable architecture diagrams as native `.drawio` files, then export to PNG (with embedded XML so it stays editable) and present to the user.

This skill encodes a reusable design method so every diagram is clear by construction — not just decorated. Apply the method first, then draw.

## Design method: four elements + four rules

Every good architecture / concept / business diagram is built from four elements. Decide each before drawing.

**Layer (层) — how many abstraction levels.** Slice the system into horizontal bands top-to-bottom, each band carrying one kind of responsibility. Enforce "homogeneous within a layer, orthogonal between layers." This is the foundation of clarity.

**Block (块) — what roles/capabilities fill each layer.** Each block is one nameable responsibility unit. Keep blocks at the "same altitude" — never mix a whole subsystem with a tiny function at equal size. Mismatched granularity is the most common failure.

**Relation (线) — how blocks/layers connect.** Let direction carry meaning: horizontal = process flow or parallelism; vertical = dependency, invocation, or support. Within one diagram, one direction expresses only one kind of relation.

**Boundary (框) — the scope and goal.** Use an outer frame, grouping, and edge anchors (goal top-right, reading axis on the left, external systems on the right) to make "what is inside vs outside" obvious at a glance.

Then apply three global rules:

1. **Narrative spine.** Give the reader an entry point and path — a one-line flow (`阶段一 → 阶段二 → …`) or a clear top-down / left-right direction. Avoid diagrams you can start reading anywhere.
2. **Consistent encoding.** Color/shape/size map to categories consistently, so the reader builds the mental map once. Always include a small legend.
3. **Show the loop.** Draw feedback / reflow, not just one-way pipelines. A feedback arrow is what makes a diagram express a living system instead of a dead flowchart.
4. **Balance blocks and connectors.** Blocks must not crowd out the arrows between them. Leave a visible gap of roughly 20–30px between adjacent blocks so each connector renders as a clear segment with a readable arrowhead — never let boxes sit only a few pixels apart, which swallows the line. When a layer gets crowded, shrink block widths to open up the gaps (raising information density) rather than widening the band. For sequential blocks in a row, pin connectors to fixed side anchors (`exitX=1;exitY=0.5;entryX=0;entryY=0.5`) so arrows stay horizontal and visible.

One-line summary: **layers set structure, direction sets meaning, the spine sets order, the loop sets vitality, and spacing keeps the connectors visible.**

## Color palette

Color must carry information, not decoration. Rules: same category = same color; light fill + dark stroke + dark text; keep to ≤4 hues + 1 neutral; cool = stable/system/foundation, warm = people/driver/risk; never rely on color alone (pair with shape and label).

Use this palette (fill / stroke / text):

| Role | Fill | Stroke | Text |
|------|------|--------|------|
| Blue · core / execution | `#E8F1FB` | `#2B7DE0` | `#15497E` |
| Green · foundation / steady state | `#E9F5EA` | `#4A9A4A` | `#2E6B2E` |
| Orange · driver / attention | `#FFF2E6` | `#E8833A` | `#B35A16` |
| Purple · platform / support | `#EFEAF7` | `#7E57C2` | `#4A2E80` |
| Gray · external / secondary | `#F2F4F7` | `#98A2B3` | `#475467` |

## Workflow

1. **Clarify intent.** If the diagram type/scope is unclear, ask briefly: is it conceptual/business (roles & collaboration) or technical (components & data flow)? How many layers? What is the goal and boundary? Do not over-ask when the request is already specific.
2. **Map to the method.** Decide the layers, the blocks per layer (same altitude), the relation directions, and the boundary anchors. Draft the narrative spine sentence and the feedback loop.
3. **Compose the `.drawio`.** Start from `template.drawio` in this skill directory — it already contains 4 layer bands, same-altitude blocks, an external-systems column, vertical relation arrows, a dashed feedback loop, a narrative spine, and a legend/palette. Replace the `【placeholder】` text, add/remove layers and blocks, and recolor per the palette. For a horizontal/swimlane process diagram, rotate the layers into columns.
4. **Export and present.** Export to PNG with embedded XML, then present both the PNG and the `.drawio` source.

## Building blocks (drawio XML)

Layer band (container, title at top):

```xml
<mxCell id="L2" value="② 核心层 · 谁在执行" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#e8f1fb;strokeColor=#2b7de0;fontColor=#15497e;fontStyle=1;fontSize=13;verticalAlign=top;spacingTop=6;arcSize=6;" vertex="1" parent="1">
  <mxGeometry x="60" y="180" width="640" height="100" as="geometry"/>
</mxCell>
```

Same-altitude block:

```xml
<mxCell id="L2a" value="【模块】" style="rounded=1;whiteSpace=wrap;html=1;fillColor=#ffffff;strokeColor=#2b7de0;fontColor=#15497e;fontSize=12;arcSize=18;" vertex="1" parent="1">
  <mxGeometry x="80" y="215" width="150" height="52" as="geometry"/>
</mxCell>
```

Openness marker (avoid exhaustive enumeration): a text cell with value `…`.

Vertical dependency edge (every edge needs a child `mxGeometry`):

```xml
<mxCell id="e23" value="依赖 / 调用" style="edgeStyle=orthogonalEdgeStyle;html=1;endArrow=classic;startArrow=classic;strokeColor=#98a2b3;fontColor=#667085;fontSize=10;" edge="1" parent="1" source="L2" target="L3">
  <mxGeometry relative="1" as="geometry"/>
</mxCell>
```

Feedback loop (dashed, single direction):

```xml
<mxCell id="loop" value="反馈 / 回流" style="html=1;endArrow=classic;dashed=1;strokeColor=#4a9a4a;fontColor=#2e6b2e;fontSize=10;" edge="1" parent="1" source="R" target="L4">
  <mxGeometry relative="1" as="geometry"/>
</mxCell>
```

Database/store shape: `shape=cylinder3;whiteSpace=wrap;html=1;boundedLbl=1;backgroundOutline=1;`. People/actor: `shape=actor;`. Vertical axis label: `text;html=1;horizontal=0;`.

## XML rules

- Wrap in `<mxGraphModel adaptiveColors="auto"><root><mxCell id="0"/><mxCell id="1" parent="0"/> … </root></mxGraphModel>`; all elements use `parent="1"`.
- Never include XML comments. Escape `&amp; &lt; &gt; &quot;`. Unique `id` per cell. Every edge has a child `<mxGeometry relative="1" as="geometry"/>`.

## Export and present

Locate the draw.io CLI, then export to PNG with embedded XML (`-e`) and delete the intermediate file only if you exported to a double-extension file.

CLI paths: macOS `/Applications/draw.io.app/Contents/MacOS/draw.io`; Linux `drawio`; Windows `"C:\Program Files\draw.io\draw.io.exe"`; WSL2 `` `/mnt/c/Program Files/draw.io/draw.io.exe` ``.

```bash
DRAWIO=/Applications/draw.io.app/Contents/MacOS/draw.io
"$DRAWIO" -x -f png -e -b 12 -s 2 -o outputs/diagram.drawio.png diagram.drawio
```

If the CLI is not found, keep the `.drawio` file and tell the user to install the draw.io desktop app or open the file directly.

Save deliverables to the output folder, then call the `present` tool with the PNG and the `.drawio` source. Share direct `file://` links.

## Additional resources

- Blank starting point: `template.drawio` in this skill directory — copy it into the working folder and edit.
