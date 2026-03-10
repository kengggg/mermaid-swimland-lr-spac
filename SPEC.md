# Swimlane-LR Diagram — Syntax Specification
> Reference document for implementing the `swimlane-lr` diagram type as a Mermaid fork.
> Covers: grammar, node shapes, lane declarations, edges, colors, and layout hints.
---
## 1. Diagram Declaration
Every diagram begins with `swimlane-lr`. This is the keyword the parser uses to identify the diagram type.

swimlane-lr

---
## 2. Lane Declaration
Lanes are horizontal bands. They are declared in top-to-bottom order. Lane label is a single word or quoted string. Background color is optional and defaults to `#ffffff`.

lane <LaneName> [#hexcolor]
lane <"Lane Name With Spaces"> [#hexcolor]

### Examples

lane Car
lane Fun #f0f4ff
lane Sales #fff7e6
lane Constr #f0fff4
lane "Legal Dept" #fff0f0

### Rules
- Lane order in the source = top-to-bottom render order.
- Lane height is **auto-sized** by the layout engine (ELK) to fit the tallest node in that lane.
- Lane label renders as rotated or left-aligned text on the left edge of the band.
- If no color is given, defaults to white (`#ffffff`).
---
## 3. Node Declaration
Nodes are declared **inside** a lane block. They are automatically assigned to the lane they are declared in.
### 3.1 Shapes
| Shape | Syntax | Use |
|---|---|---|
| Rectangle | `ID[label]` | Process step |
| Diamond | `ID{label}` | Decision (max 2 outgoing edges) |
| Circle | `ID((label))` | End / terminal event |
| Stadium | `ID([label])` | Start event (optional) |
### 3.2 Multi-line Labels
Use `\n` inside the label string to force a line break. The shape automatically expands to fit.

ID[First line\nSecond line\nThird line]
ID{Is condition met?\nReview required?}

### 3.3 Node Styling via classDef
Nodes can be styled using Mermaid's existing `classDef` and `:::` mechanism.

classDef <className> fill:<color>, stroke:<color>, color:<color>

Apply to a node:

ID[label]:::className

`classDef` declarations can appear anywhere at the top level of the diagram (before or after lane blocks).
---
## 4. Edge Declaration
Edges (arrows) can be declared **inside** a lane block or **at the top level**. Because many edges cross lanes, top-level edge declarations are recommended for cross-lane connections.
### 4.1 Edge Styles
| Style | Syntax | Renders as |
|---|---|---|
| Solid arrow | `A --> B` | Solid line, filled arrowhead |
| Dashed arrow | `A -.-> B` | Dashed line, filled arrowhead |
### 4.2 Edge Labels
Labels are placed in quotes between the arrow tokens.

A -- "label text" --> B
A -.- "label text" -> B

Multi-line edge labels: use `\n` inside the quoted string.

A -- "Yes but this is\na long label" --> B

### 4.3 Edge Colors
An optional `[#hexcolor]` token can be inserted to color the edge line and arrowhead.

%% Solid, colored
A --[#ff0000]--> B
%% Dashed, colored
A -.-[#0055ff]-> B
%% Solid, colored, with label
A --[#00aa00]"Yes path"--> B
%% Dashed, colored, with label
A -.-[#888888]"Async call"-> B

If `[#hexcolor]` is omitted, the edge falls back to the theme default color.
### 4.4 Chained Edges
Multiple nodes can be chained in one statement.

A --> B --> C --> D

---
## 5. Comments
Use `%%` for single-line comments. Comments are ignored by the parser.

%% This is a comment
lane Car
A[Start] %% inline comment on node

---
## 6. Full Syntax Grammar (EBNF sketch)
This is an informal sketch to guide the `.jison` / `.peggy` grammar implementation.

diagram := "swimlane-lr" NEWLINE (classDef | lane | edge | comment)*
classDef := "classDef" className styleList
lane := "lane" laneName [hexColor] NEWLINE INDENT (node | edge | comment)* DEDENT
node := nodeId shape [":::" className]
shape := "[" label "]" -- rectangle
| "{" label "}" -- diamond
| "((" label "))" -- circle
| "([" label "])" -- stadium
label := quoted_string | unquoted_text
edge := nodeId edgeToken [colorToken] [labelToken] edgeToken nodeId
edgeToken := "-->" | "-.->" | "--" | "-.-"
colorToken := "[" hexColor "]"
labelToken := quoted_string
hexColor := "#" [0-9a-fA-F]{3,6}
comment := "%%" text NEWLINE

---
## 7. Layout Engine Notes (ELK)
The renderer should pass the following hints to ELK:
- `org.eclipse.elk.partitioning.activate: true`
- Each node gets `partitionId` = its lane index (0-based, top to bottom)
- `rankdir: RIGHT` (left-to-right flow within partitions)
- Node sizes are **pre-measured** from label text before passing to ELK, so ELK receives exact `width` and `height` per node
- ELK returns `x`, `y` per node → renderer uses these plus lane Y-ranges for final SVG output
---
## 8. Complete Examples
### Example 1 — Simple Linear Flow

swimlane-lr
classDef highlight fill:#fffbe6, stroke:#f0c040
classDef danger fill:#ffe0e0, stroke:#cc0000
lane Start #f0f4ff
A([Begin])
lane Process #f9f9f9
B[Collect Data]
C[Validate Input]:::highlight
lane Review
D{Approved?}:::danger
lane Done #f0fff4
E((End))
%% edges
A --> B --> C --> D
D -- "Yes" --> E
D -- "No" --> B

---
### Example 2 — Cross-Lane Flow with Colors and Dashes

swimlane-lr
classDef critical fill:#ffcccc, stroke:#cc0000, color:#000
classDef async fill:#e8f4ff, stroke:#4488cc, color:#000
lane Car
A[Car Request]
lane Fun #fff7e6
F[Fun Activity]
G((Done))
lane Sales
B[Sales Check]
lane Constr #f0fff4
C[Construction Start]
D[Draft Plan]
H[Submit Plan]
E[Execute]
lane Legal #fff0f0
I{Legal Review?}:::critical
J[I am a very long label\nthat will wrap to next line\nand this is another line\nand one more line]
K[Notify Parties]
L[Final Sign-off]
M((Complete))
N[Archive]
%% main flow
A --> B --> C --> D --> H
H --> E
E --> F --> G
%% cross-lane legal flow
H --> I
I -- "Yes but with a long label\nthat wraps to next line" --> J
J --[#888888]"No" --> K --> L
L --> M
L --> N
%% loop-back (dashed = async notification)
J -.-> B

---
### Example 3 — Decision Tree with Styled Edges

swimlane-lr
classDef terminator fill:#333, stroke:#333, color:#fff
lane Input #f5f5ff
S([Start]):::terminator
Q{Has Account?}
lane New User #fff9e6
R[Register]
V[Verify Email]
lane Existing User #f0fff4
L[Login]
P[Reset Password?]
lane Core #ffffff
D[Dashboard]
E((End)):::terminator
%% edges
S --> Q
Q -- "No" --> R --> V --> D
Q --[#00aa00]"Yes" --> L
L --> D
L -.-[#ff8800]"Forgot password" -> P
P --> L
D --> E

---
### Example 4 — Minimal with End Circles

swimlane-lr
lane Request
A([Start])
B[Submit Form]
lane Approval
C{Manager Approval}
lane Outcome #f0fff4
D((Approved))
E((Rejected))
A --> B --> C
C -- "Approved" --> D
C --[#cc0000]"Rejected" --> E

---
## 9. Design Constraints & Validation Rules
These should be enforced at parse or semantic analysis time:
1. **Diamond nodes (decisions) may have at most 2 outgoing edges.** Error if 3+ edges leave a `{}` node.
2. **Circle nodes (terminals) may have 0 outgoing edges.** Warning if an edge leaves a `(())` node.
3. **Node IDs must be unique** across all lanes.
4. **Edge endpoints must reference declared node IDs.** Error if an edge references an unknown ID.
5. **Lane names must be unique.**
6. **`classDef` names must be unique.** Last definition wins if duplicated (or error — implementor's choice).
7. **`[#hexcolor]` must be valid 3 or 6 digit hex.** Error on invalid format.
---
## 10. SVG Rendering Order
To ensure correct visual layering:
1. Draw lane background rectangles (full width, auto height from ELK output)
2. Draw lane label text (left edge, vertically centered)
3. Draw lane separator lines (horizontal rules between bands)
4. Draw edges (lines + arrowheads), with color from edge declaration or theme default
5. Draw edge labels (small boxes with background fill, centered on edge midpoint)
6. Draw nodes on top (shapes + text), with fill/stroke from `classDef` or defaults
7. Draw node labels (centered text inside shape)
---
## 11. Theme Defaults (when no color is specified)
| Element | Default |
|---|---|
| Lane background | `#ffffff` |
| Lane label text | `#333333` |
| Lane separator | `#cccccc` |
| Node fill | `#ffffff` |
| Node border | `#333333` |
| Node text | `#333333` |
| Edge line | `#333333` |
| Edge label background | `#f5f5f5` |
| Edge label text | `#333333` |
| Arrowhead fill | `#333333` |