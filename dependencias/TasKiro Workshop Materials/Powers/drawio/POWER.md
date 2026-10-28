---
name: "drawio"
displayName: "Draw.io Diagrams"
description: "Generate draw.io diagrams from draw.io XML and save them as editable .drawio files. Search 10,000+ shapes (AWS, Azure, GCP, Cisco, Kubernetes, UML, BPMN) for exact style strings."
keywords: ["drawio", "diagram", "mxgraph", "flowchart", "architecture-diagram"]
author: "jgraph"
---

# Draw.io Diagrams

## Overview

This power generates [draw.io](https://www.draw.io) diagrams from draw.io XML
(`mxGraphModel` format) and connects to the official draw.io MCP App Server
(`https://mcp.draw.io/mcp`, a hosted remote server — no installation, no CLI) for
shape search.

**Important — Kiro does not render diagrams inline in chat.** The MCP App Server's
`create_diagram` tool is designed to draw the diagram inside the chat response, but
that requires an MCP Apps-capable host; Kiro is not one, so no image appears. The
**reliable artifact in Kiro is a `.drawio` file saved to the workspace** — you can
open and edit it with the Draw.io Integration VS Code extension (`hediet.vscode-drawio`)
or draw.io Desktop. Therefore the primary workflow of this power is: generate the XML,
**always save it as a `.drawio` file**, and tell the user where it is.

## Available Steering Files

- **xml-reference** — The complete draw.io XML generation reference: styles, edge
  routing and layout passes, containers/swimlanes, cross-functional tables, layers,
  tags, metadata/placeholders, dark mode, XML well-formedness, and shape search usage.
  Read this before hand-writing any non-trivial draw.io XML.

  Access it with: `readSteering` → powerName `drawio`, steeringFile `xml-reference.md`.

- **aws-architecture** — Senior-level guide for AWS architecture diagrams using the
  `mxgraph.aws4` icon set: correct nested containment (Cloud ⊃ Region ⊃ VPC ⊃ Subnet ⊃
  Security Group ⊃ resource), Security Groups drawn as dashed boundaries around
  resources, route tables, connection semantics + legend, the AWS category color
  palette, official `resIcon`/`grIcon`/`shape` style strings, layout/spacing tips, and a
  reusable skeleton + review checklist. Read this whenever the user asks for an AWS
  architecture diagram.

  Access it with: `readSteering` → powerName `drawio`, steeringFile `aws-architecture.md`.

## Primary Workflow: Always Save a .drawio File

When the user asks for any diagram, follow these steps:

1. **Decide whether to search shapes.** Skip `search_shapes` for standard diagrams
   (flowcharts, UML, ERD, org charts, mind maps) that only need rectangles, diamonds,
   circles, and arrows. Use `search_shapes` for cloud (AWS/Azure/GCP), network
   (Cisco/racks), P&ID, electrical, or Kubernetes diagrams to get the real icon styles.

2. **Generate the diagram XML** (`mxGraphModel` content) following the rules in the
   `xml-reference` steering file. No XML comments; escape `& < > "`; every edge needs a
   `<mxGeometry relative="1" as="geometry"/>` child.

3. **ALWAYS write the diagram to a `.drawio` file** in the workspace using a file-write
   tool. Wrap the `mxGraphModel` in the `.drawio` container so draw.io opens it
   directly:

   ```xml
   <mxfile host="Kiro">
     <diagram name="Page-1" id="page-1">
       <mxGraphModel dx="800" dy="600" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="850" pageHeight="1100" math="0" shadow="0" adaptiveColors="auto">
         <root>
           <mxCell id="0"/>
           <mxCell id="1" parent="0"/>
           <mxCell id="2" value="Example" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="1">
             <mxGeometry x="40" y="40" width="140" height="60" as="geometry"/>
           </mxCell>
         </root>
       </mxGraphModel>
     </diagram>
   </mxfile>
   ```

   Your vertices and edges are children of `parent="1"`, after cells `0` and `1`. Save
   the file with a descriptive name, e.g. `docs/<diagram-name>.drawio` or next to the
   relevant code. Never put an actual XML comment in the output.

4. **Tell the user the file path** and how to open it (Draw.io Integration VS Code
   extension or draw.io Desktop). This saved file is the source of truth.

5. **(Optional) Call `create_diagram`** with the inner `mxGraphModel` XML. In Kiro this
   returns the XML as text rather than an inline image, so it is not required — the
   `.drawio` file is the deliverable. It is still useful if you want the server's
   `routing: "libavoid"` / `postLayout: "elk"` passes to compute clean wiring; copy the
   resulting routed XML back into the `.drawio` file.

## Available MCP Server and Tools

**Server name:** `drawio` (remote HTTP, `https://mcp.draw.io/mcp`)

### `search_shapes`

Searches 10,000+ shapes across all draw.io libraries and returns exact `style` strings
you can drop straight into `mxCell` attributes.

- **Input:** `query` (string, required) — space-separated keywords (AND logic, exact +
  Soundex matching); `limit` (number, optional, default 10, max 50).
- **Output:** array of `{ style, w, h, title }`. Use `style` on a vertex and `w`/`h` in
  its `mxGeometry`.

### `create_diagram`

Renders draw.io XML. Intended for inline display in MCP Apps hosts; in Kiro it returns
the XML as text.

- **Input:** `xml` (string, required) — `mxGraphModel` XML. Optional `routing:
  "libavoid"` (reroute connectors around shapes, keep positions) OR `postLayout: "elk"`
  (full hierarchical re-layout) plus `direction: "vertical"|"horizontal"`. Pick one, not
  both.

## Example: Simple flowchart saved as a .drawio file

1. Generate the cells (standard shapes, so no `search_shapes` needed).
2. Write this file to e.g. `docs/login-flow.drawio`:

```xml
<mxfile host="Kiro">
  <diagram name="Login Flow" id="login-flow">
    <mxGraphModel dx="800" dy="600" grid="1" gridSize="10" page="1" pageWidth="850" pageHeight="1100" adaptiveColors="auto">
      <root>
        <mxCell id="0"/>
        <mxCell id="1" parent="0"/>
        <mxCell id="2" value="Start" style="rounded=1;whiteSpace=wrap;html=1;" vertex="1" parent="1">
          <mxGeometry x="40" y="40" width="140" height="60" as="geometry"/>
        </mxCell>
        <mxCell id="3" value="Valid?" style="rhombus;whiteSpace=wrap;html=1;" vertex="1" parent="1">
          <mxGeometry x="40" y="160" width="140" height="80" as="geometry"/>
        </mxCell>
        <mxCell id="e1" edge="1" parent="1" source="2" target="3" style="edgeStyle=orthogonalEdgeStyle;rounded=1;html=1;">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

3. Tell the user: "Saved `docs/login-flow.drawio` — open it with the Draw.io Integration
   extension or draw.io Desktop to view/edit."

## Example: Cloud diagram (search first)

For AWS/Azure/GCP/Cisco/Kubernetes diagrams, call `search_shapes` to get the icon
style, substitute it into a vertex, then save the `.drawio` file as above.

1. `search_shapes` with `{ "query": "aws lambda", "limit": 5 }`.
2. Use the returned `style`, `w`, `h` on a vertex:

```xml
<mxCell id="fn" value="OrderFn" style="STYLE_FROM_SEARCH" vertex="1" parent="1">
  <mxGeometry x="40" y="40" width="78" height="78" as="geometry"/>
</mxCell>
```

3. Wrap in `mxfile`/`diagram` and save as `.drawio`. For nested cloud layouts (VPC → AZ
   → instance), use nested `swimlane` containers — see the `xml-reference` steering file.

> For **AWS** architecture specifically, read the **`aws-architecture`** steering file
> first — it has the correct `aws4` icon styles, nested containment rules, Security
> Group boundaries, route tables, color palette, and a reusable skeleton. For AWS,
> position shapes by hand and use `routing: "libavoid"` (never `postLayout: "elk"`,
> which would discard the meaningful spatial layout).

## Best Practices

- **Always produce a `.drawio` file.** That is the reliable output in Kiro — never rely
  on inline chat rendering.
- **Wrap XML in `<mxfile><diagram>…`** for the saved file; pass only the inner
  `mxGraphModel` to `create_diagram` if you call it.
- **Decide whether to search.** Skip `search_shapes` for standard diagrams; use it only
  for industry/branded icons.
- **Let the router do layout.** Declare `source`/`target` on edges; don't hand-compute
  waypoints or coordinates. Use `routing: "libavoid"` or `postLayout: "elk"` when wires
  would cross shapes.
- **Every edge needs a geometry child**; use the expanded `<mxCell …>…</mxCell>` form.
- **Always set `html=1`**, XML-escape HTML in `value` (`&lt; &gt; &amp; &quot;`), use
  `&#xa;` or `&lt;br&gt;` for line breaks (never `\n`).
- **No XML comments** (`<!-- -->`) in generated diagram output.
- **Match the user's language** for labels and titles.

## Troubleshooting

### No diagram image appears in chat

**Cause:** Kiro does not support MCP Apps inline rendering, so `create_diagram` cannot
draw inside the chat.
**Solution:** This is expected. Use the saved `.drawio` file (the primary workflow) and
open it in the Draw.io Integration VS Code extension or draw.io Desktop.

### `.drawio` file won't open or is blank

**Cause:** malformed XML — missing `<mxfile>`/`<diagram>` wrapper, XML comments,
unescaped `& < > "`, or a self-closing edge cell.
**Solution:** ensure the `mxGraphModel` is wrapped in `<mxfile><diagram>…`, remove all
`<!-- -->` comments, escape special characters, and give every edge a `<mxGeometry
relative="1" as="geometry"/>` child. See the `xml-reference` steering file.

### `search_shapes` / MCP server not connecting

**Symptoms:** the `drawio` tools don't appear or calls time out.
**Solutions:**
1. Confirm the power is enabled and `mcp.json` points to `https://mcp.draw.io/mcp`.
2. Allow outbound HTTPS to `mcp.draw.io` (hosted remote server).
3. Reconnect the server from Kiro's MCP Server view (no restart needed).

### "Session not found" (404)

**Cause:** stale session ID after the server's idle timeout (~5 min) or a redeploy.
**Solution:** reconnect the MCP server and retry.

## Configuration

**No additional configuration required.** Shape search uses the hosted endpoint as-is —
no API key, no install, no CLI. Saving `.drawio` files uses Kiro's file tools and needs
nothing extra. To view files, install the Draw.io Integration VS Code extension
(`hediet.vscode-drawio`) or draw.io Desktop.

**Privacy note:** when you call `create_diagram` against the hosted endpoint, the XML is
sent to the draw.io server. Saving `.drawio` files is fully local. If you need shape
search to stay local too, the upstream project supports self-hosting the App Server;
point the `url` in `mcp.json` at your own instance.

---

**MCP Server:** `drawio` (remote HTTP)
**Endpoint:** `https://mcp.draw.io/mcp`
**Upstream:** https://github.com/jgraph/drawio-mcp
