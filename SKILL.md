---
name: error-discovery
description: Run error analysis on a dataset. Build a review UI, select diverse samples, monitor annotations, and organize failure modes.
---

# Error Discovery Skill

You are running an interactive error analysis session. The user has a dataset (JSONL, CSV, JSON, etc.) of LLM outputs or traces and wants to discover failure modes by reviewing samples.

## Phase 1: Understand the domain and data

Before building anything, study the dataset thoroughly.

### 1a: Read and inventory the data

Load the file. Examine 5-10 records across the distribution. For each record, identify:
- All fields, their types, and what they represent
- Which fields are the **primary content** the human needs to judge
- Which fields are **metadata** (context for understanding but not the thing being judged)
- Which fields **vary across records** vs which are constant

### 1b: Identify the content structure

The data could take many forms. Determine which pattern fits:

- **Single text field**: article, summary, email, essay, translation
- **Input/output pair**: prompt + completion, question + answer, instruction + result
- **Multi-turn trace**: a sequence of messages with different roles (system, human, assistant, tool calls, tool results, thinking/reasoning blocks). This is common for agent traces, chatbot logs, and agentic workflows.
- **Code**: source files, patches, diffs, with or without surrounding context
- **Structured output**: JSON, function calls, extracted entities, classifications
- **Composite**: multiple of the above combined (e.g., a task description + an agent trace + a final output)

For multi-turn traces specifically, identify:
- What roles/authors exist (system, user, assistant, tool, thinking, etc.)
- Whether there are tool call / tool result pairs
- Whether there are thinking/reasoning blocks
- What the logical grouping of turns is (e.g., one "step" = thinking + tool call + tool result)

### 1c: Identify dimensions of variation

Think about what differs **between** data points and **within** each data point. These dimensions drive the visual design.

**Between data points** (vary across records):
- Metadata: topic, model, difficulty, source, label, task type, etc.
- Structural: length, number of turns, number of tool calls, etc.
- Outcome: success/failure, score, etc.

**Within each data point** (vary across parts of one record):
- Author/role of each segment (human vs agent vs system vs tool)
- Content type of each segment (natural language vs code vs JSON vs thinking)
- Importance/relevance of each segment (boilerplate system prompt vs the actual response)
- Anomaly level (how unusual is this word/phrase/pattern compared to the dataset distribution)

### 1d: Think about what "bad" means

What are plausible failure categories in this domain? Seed a few mentally but expect the human to discover most of them. This informs what to pre-compute for visual emphasis.

## Phase 2: Design the visual encoding

**Before writing any code**, design how every dimension of variation maps to a visual property. Use Gestalt principles and information visualization fundamentals.

### Core Gestalt principles to apply

- **Similarity** (color, shape): things that share a visual property are perceived as related. Use this for categorical dimensions — different message roles get different colors, different labels get different badge colors.
- **Proximity** (spacing): things close together are perceived as grouped. Use this for logical units — turns in a conversation step are close together, steps are separated by more space.
- **Common region** (containers, backgrounds): things inside a shared boundary are grouped. Use this for multi-part content — a tool call and its result share a container.
- **Figure/ground** (opacity, contrast): important content should be high-contrast (figure), less important content should recede (ground). Use this for emphasis — boilerplate at low opacity, anomalous content highlighted.

### Visual encoding rules

Map each dimension of variation to exactly one visual channel. Don't use the same channel for two different things.

**Color hue**: categorical dimension with the most important distinction.
- For multi-turn traces: message role. Each role (human, assistant, system, tool, thinking) gets its own distinct hue at full saturation. Pick 3-5 easily distinguishable hues.
- For single-content: label or source category.
- Never use color hue for quantitative data.

**Opacity / saturation**: whether the human needs to read this content carefully.
- Reduce opacity only for content the human genuinely does not need to read: repeated boilerplate that is identical across records, verbose tool schemas, auto-generated headers. Think: "if I removed this, would the human miss anything?"
- Do NOT mute content by role. System messages, tool results, and thinking blocks can all contain the actual bug. Mute only specific content that is redundant or mechanical, regardless of which role produced it.
- Normal content: full opacity.
- Anomalous or noteworthy content: highlighted background (not just higher opacity — use a background tint or underline to make it pop).

**Spacing**: hierarchical structure.
- Tight spacing (4-8px) between items within a logical group (e.g., consecutive messages in one "step" of an agent trace).
- Medium spacing (16-24px) between logical groups (e.g., between steps).
- Large spacing (32-48px) between major sections.

**Typography**: content type.
- Natural language prose: proportional font, normal size.
- Code: monospace font, slightly smaller.
- Metadata: small, muted, compact.
- Thinking/reasoning: could be italic or a distinct font treatment to signal "internal."

**Border / container**: grouping of related parts.
- A tool call + its tool result: share a container with a subtle border.
- An input + output pair: separated by a clear divider or in side-by-side containers.

**Structural outlier flags**: header-only, not inline.
- Pre-compute dataset-level averages for key structural features (length, heading count, paragraph count, etc.).
- For each record, flag dimensions where it is a clear statistical outlier (e.g., top/bottom 10%).
- Show these as small, compact badges in the header (e.g., "6 headings (more than 89%)", "shorter than 95%").
- Keep it minimal. Most records should have zero or one flag. Don't annotate inline content.

## Phase 3: Build the review interface

### Architecture

- **Python HTTP server** (stdlib `http.server`, no dependencies):
  - `GET /` — serves the HTML app
  - `GET /api/samples` — returns current sample set
  - `POST /api/samples` — agent pushes new samples
  - `GET /api/annotations` — returns current annotations
  - `POST /api/annotations` — app saves annotations on every change
  - `GET /api/graph` — returns 2D projection of all records for the cluster map
  - `GET /api/patterns` — returns the agent's current failure mode taxonomy
  - `POST /api/patterns` — agent pushes updated taxonomy
- **On-disk files** in an `error_discovery_data/` directory:
  - `samples.json`, `annotations.json`, `graph.json`
- The HTML app auto-saves to the server on every annotation, and polls for new samples.

### HTML app structure

**Three views**, toggled from the top bar:

1. **Article/content view** — the main review interface where the human reads and annotates.
2. **Map view** — a 2D scatter plot (PCA or UMAP projection) of all records, showing clusters, which items are in the sample, and which have been annotated. Clicking a sample node navigates to its content view.
3. **Patterns view** — a treemap of failure modes the agent has categorized so far. Each block is a failure mode, sized by how many annotations it contains. Inside each block, list the individual notes. Clicking a note navigates to that annotation in the content view. This view is the agent's live taxonomy — it updates as the agent categorizes new annotations.

**Content view design — apply the visual encoding from Phase 2:**

- **Header**: title + label + topic. Keep it minimal. Add structural outlier flags (from Phase 2) only when the record is a genuine outlier. Don't clutter with cluster IDs, sampling method, percentile bars, or other pipeline metadata the reviewer doesn't need.
- **Body**: render the primary content using the appropriate treatment per content type:

  **For multi-turn traces (agent logs, conversations, chat):**
  - Each message/turn is a block. Left-align all but use a colored left-border or background tint per role.
  - Role label (small, bold) at the top of each block: "System", "User", "Assistant", "Tool Call", "Tool Result", "Thinking".
  - Color coding: assign a distinct hue to each role. Be consistent across all records. All roles at full opacity by default.
  - Thinking/reasoning blocks: render in a visually distinct way (e.g., lighter background, italic, or slightly indented) to signal "internal monologue" while keeping full readability.
  - Tool calls: show the function name prominently, parameters in collapsible formatted JSON.
  - Tool results: collapsible by default if long, with a summary line visible.
  - Only reduce opacity for content that is literally identical across records (e.g., the same system prompt repeated verbatim in every trace). If system messages vary, keep them fully visible.
  - Group related turns: a thinking block + the tool call it leads to + the tool result, visually grouped with tight spacing and a shared container.

  **For single text content (articles, summaries, etc.):**
  - Render markdown as formatted HTML (use marked.js or similar).
  - Render plain text with paragraph breaks.

  **For code / diffs:**
  - Syntax highlighting (highlight.js or Prism via CDN).
  - Diffs: green background for additions, red for deletions.
  - Line numbers.

  **For input/output pairs:**
  - Stacked with a clear divider, or side-by-side if both are short.
  - Each section labeled.

- **Inline annotation**: select text: floating popover with text input: Enter to save: span highlighted. Hovering shows note. Click to edit/delete.
- **No quality labels, no dropdowns, no structured forms.** Free-text notes only.
- **Auto-save** to server on every change. localStorage backup.
- **Poll for new samples** every ~10s. Show a banner when the agent adds new ones.

**Map view:**
- 2D scatter of all records from PCA/UMAP projection.
- Nodes colored by primary categorical dimension (label, or role distribution, or outcome).
- Sample items: larger nodes with dark border.
- Annotated items: distinct color (e.g., orange).
- Cluster hulls or convex boundaries as subtle background shapes.
- Hover: tooltip with title, metadata, annotation count.
- Click (on sample nodes): navigate to content view for that item.

## Phase 4: Cluster and select initial samples

1. **Extract features** appropriate to the content type (see Phase 1c for the dimensions).
2. **Cluster** using KMeans or similar on normalized features. Target 6-10 clusters.
3. **Build the initial sample** (15-25 items for datasets >50):
   - **Cluster representatives (~60-70%)**: 1-2 items closest to each centroid, mixing categories/labels.
   - **Random samples (~30-40%)**: from the full dataset regardless of cluster. Clustering may miss important dimensions — random picks guard against blind spots.
   - Deduplicate overlaps.
4. **Prioritize diversity** — the goal is discovering failure modes, not estimating prevalence.

## Phase 5: Run the interactive review loop

This phase is **ongoing and interactive**. The agent is an active participant.

### 5a: Launch and monitor

1. Start the Python server in the background.
2. Open the app in the browser.
3. Tell the user the app is ready. Explain the interaction briefly.
4. **Monitor the annotations file** using the Monitor tool. React when new annotations come in.

### 5b: Process annotations as they arrive

When new annotations appear:
1. Read all annotations.
2. Categorize each into a failure mode — match to existing or create new.
3. Maintain a running taxonomy: `{mode_name: {description, count, example_ids, example_quotes[]}}`.
4. Track which records have been reviewed and which clusters/dimensions are covered.

### 5c: Propose new samples to increase coverage

After the human reviews a batch:
1. **Find more instances of known failure modes**: search remaining records for matching patterns.
2. **Cover gaps**: sample from unreviewed clusters, topics, or feature regions.
3. **Random exploration**: always include a few random picks.
4. Push new samples to the server. The app shows a banner.
5. Tell the human what was added and why.

### 5d: Report and converge

Periodically:
- Report the failure mode taxonomy with counts and examples.
- Report coverage: records reviewed, clusters/dimensions covered, what remains.
- Track convergence: are new records revealing new modes, or mostly repeats?
- When discovery rate drops, suggest stopping or narrowing focus.

## Key principles

1. **The human notices, the agent organizes.** Free-text notes: structured taxonomy. Don't make the human categorize.
2. **The agent drives coverage.** The agent proposes new samples based on what's been found and what's missing.
3. **Live feedback loop.** The agent monitors and reacts as annotations come in.
4. **Guard against blind spots.** Always include random samples alongside cluster-based ones.
5. **Adapt to the data.** The rendering, features, clustering, and visual encoding all derive from what the data actually is.
6. **Encode dimensions visually.** Map key dimensions to visual channels (color, spacing, typography). Use Gestalt principles so perception does the work.
7. **Keep the UI minimal.** Only surface information the reviewer needs to judge the content. Structural outlier flags in the header, not inline. No pipeline metadata (cluster IDs, sampling method, percentile bars).
