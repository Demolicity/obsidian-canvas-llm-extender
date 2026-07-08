# Canvas LLM Extender Engineering Report

## 1. Purpose and Operating Model

Canvas LLM Extender is an Obsidian plugin that adds an LLM-powered action to the Canvas node context menu. When a user right-clicks a canvas node and chooses **LLM Extend**, the plugin reads the selected node, adjacent incoming and outgoing nodes, and sibling nodes that share an incoming parent. It turns that local graph neighborhood into a text prompt, sends the prompt to an OpenAI-compatible chat-completions endpoint, cleans common response prefixes, and creates a new outgoing canvas node containing the model response.

At runtime the control flow is:

1. Obsidian loads `src/main.ts` as the plugin entry point.
2. `CanvasLLMExtendPlugin.onload()` loads persisted settings.
3. `onload()` registers a `canvas:node-menu` workspace event handler.
4. The handler adds a `MenuItem` titled `LLM Extend` to the canvas node menu.
5. Clicking the menu item calls `CanvasLLMExtendPlugin.extendNode(node)`.
6. `extendNode()` validates that the event argument is a canvas node, builds a prompt from graph neighbors, calls `openai_get_reply()`, normalizes the text response, and calls `addNodeChild()`.
7. `addNodeChild()` creates a new Obsidian canvas text node, computes a size, searches for a non-overlapping location, creates an outgoing edge, renders the new elements, and requests a canvas save.

## 2. Repository Layout

| Path | Responsibility |
| --- | --- |
| `manifest.json` | Obsidian plugin metadata: plugin id, name, version, minimum app version, description, author, and desktop/mobile compatibility. |
| `package.json` | Node package metadata, dependency versions, and developer scripts. |
| `esbuild.config.mjs` | Bundles `src/main.ts` and its imports into `main.js`, marks Obsidian and Electron dependencies as external, and switches between watch/dev and one-shot production builds. |
| `tsconfig.json` | TypeScript compiler settings used by `npm run build`; enables strict null checks and `noImplicitAny`. |
| `src/main.ts` | Plugin lifecycle, canvas menu registration, settings loading/saving, and high-level node extension workflow. |
| `src/settings.ts` | Settings schema, defaults, and Obsidian settings tab UI. |
| `src/openai-utils.ts` | OpenAI-compatible API client creation and chat completion request. |
| `src/obsidian-canvas-utils.ts` | Canvas graph inspection, text-node creation, sizing, placement, edge construction, and child-node insertion. |
| `src/obsidian-helpers.ts` | Shared error reporting helper. |
| `src/obsidian.d.ts` | Local type augmentation for Obsidian workspace events, including the custom canvas node menu callback shape. |
| `styles.css` | Plugin stylesheet placeholder. |
| `README.md` | User-facing project summary, build notes, current limitations, and suggested contributions. |

## 3. Build, Packaging, and Runtime Boundaries

### 3.1 Build scripts

`package.json` defines these primary scripts:

- `npm run dev`: runs `node esbuild.config.mjs`, which starts an esbuild watch context and emits `main.js` with an inline sourcemap.
- `npm run build`: runs `tsc -noEmit -skipLibCheck` first, then runs `node esbuild.config.mjs production` to bundle once without a sourcemap.
- `npm run lint`: runs ESLint over TypeScript files, but the repository does not currently include a flat config or legacy `.eslintrc` file, so this command needs lint configuration before it can pass in modern ESLint.
- `npm run version`: delegates to an external script path that is not present in this repository snapshot, then stages version files.

### 3.2 Bundle configuration

The esbuild entry point is `src/main.ts`. The bundle format is CommonJS because Obsidian loads plugin `main.js` as a CommonJS module. The target is `es2018`, while TypeScript is configured for `ES6`; esbuild is the effective final transpiler for shipped JavaScript.

The esbuild `external` array excludes Obsidian, Electron, CodeMirror, Lezer, and Node built-ins from the bundle. This is correct for an Obsidian plugin because Obsidian supplies those host modules at runtime.

### 3.3 Obsidian manifest

The plugin id is `canvas-llm-extender`, the minimum Obsidian app version is `1.11.4`, and `isDesktopOnly` is `false`. Because `openai-utils.ts` creates an OpenAI client with `dangerouslyAllowBrowser: true`, API calls are made from the plugin JavaScript environment and therefore expose the configured API key to that runtime. This is normal for many client-side Obsidian plugins, but it is important when designing security-sensitive features.

## 4. Module-by-Module Code Structure

## 4.1 `src/main.ts`

### Imports

`src/main.ts` imports:

- `Plugin`, `Menu`, and `MenuItem` from `obsidian`.
- Canvas graph and mutation helpers from `./obsidian-canvas-utils`:
  - `addNodeChild(node, nodetext)`
  - `getNodeNeighbours(node)`
  - `isCanvasNodeData(node)`
- `notifyError(message)` from `./obsidian-helpers`.
- `openai_get_reply(prompt, model, temperature, apiKey, baseUrl?)` from `./openai-utils`.
- Settings defaults, settings interface, and settings tab class from `./settings`.

### Class: `CanvasLLMExtendPlugin extends Plugin`

This is the main plugin class exported as the default module export. Obsidian instantiates it.

#### Members

- `settings: CanvasLLMExtendPluginSettings`
  - Holds the merged effective settings object after `loadSettings()` completes.
  - Contains API key, model, temperature, default prompt, and optional OpenAI-compatible base URL.

#### Method: `async onload()`

Lifecycle hook called by Obsidian when the plugin loads.

Parameters: none.

Behavior:

1. Calls `await this.loadSettings()` inside a `try` block.
2. If settings load fails, sends an Obsidian notice and console error through `notifyError()`.
3. Registers a workspace event through `this.registerEvent(...)` so Obsidian automatically cleans it up when the plugin unloads.
4. Listens to `this.app.workspace.on("canvas:node-menu", callback)`.
5. The callback receives:
   - `menu: Menu`: the context menu to modify.
   - `node: unknown`: the object supplied by Obsidian for the selected canvas node.
6. Adds a `MenuItem` to the menu:
   - Section: `canvasLLMExtend`
   - Title: `LLM Extend`
   - Click handler parameter: `_e: unknown`, currently unused.
   - Click behavior: calls `this.extendNode(node)`.
7. Adds the plugin settings tab: `new CanvasLLMExtendPluginSettingsTab(this.app, this)`.

Events registered:

- `canvas:node-menu`: custom/underdocumented Obsidian workspace event used to extend the canvas node context menu.

Extension considerations:

- Additional canvas menu commands should be registered in this method or extracted into a menu-registration helper.
- If the plugin grows multiple actions, the handler can add multiple `MenuItem`s under the same section.
- Long-running click handlers should surface progress or loading feedback because the current handler directly starts an async LLM call without disabling UI.

#### Method: `async extendNode(node: unknown)`

High-level feature workflow for generating and adding an LLM-proposed child node.

Parameters:

- `node: unknown`
  - The selected canvas node object as supplied by the `canvas:node-menu` event.
  - It is intentionally typed as `unknown` until validated with `isCanvasNodeData()`.

Returns: `Promise<void>`.

Behavior:

1. Validates the input using `isCanvasNodeData(node)`.
   - If validation fails, reports `Node is not Canvas Node` and returns.
2. Calls `getNodeNeighbours(node)` to obtain:
   - `incoming`: nodes with edges pointing into `node`.
   - `outgoing`: nodes reached by edges from `node`.
3. Initializes `prompt` with `this.settings.defaultPrompt` and a newline.
4. Adds sibling context:
   - For each incoming parent, it calls `getNodeNeighbours(incoming)`.
   - For each outgoing child of that incoming parent, it appends `Sibling: ${sibling.text}`.
   - This includes any node that shares an incoming parent with the selected node. Depending on graph shape, it may include the selected node itself.
5. Adds direct incoming node text as `Incoming: ...`.
6. Adds selected node text as `Main: ...`.
7. Adds direct outgoing node text as `Outgoing: ...`.
8. Calls `openai_get_reply(prompt, this.settings.model, this.settings.temperature, this.settings.apiKey, this.settings.baseUrl)`.
9. If the response is `null`, reports `Failed to get reply from OpenAI` and returns.
10. Removes known leading labels from the model response if present:
    - `outgoing:`
    - `new outgoing:`
    - `new outgoing node:`
    - `new node:`
    - `node:`
    - `additional item:`
    - `item:`
11. Trims the cleaned response.
12. Calls `addNodeChild(node, r.trim())` to mutate the canvas.

Important implementation details:

- The prompt is built as a plain text transcript rather than a structured JSON payload.
- Node text is read from `.text`, so non-text canvas node types are not truly supported even if `isCanvasNodeData()` passes.
- The LLM request is not wrapped in `try/catch`; network or API exceptions will currently propagate out of the click path.
- The prefix stripper lowercases only for checking, then uses the original string for slicing. It only strips one matching prefix because each banned phrase is checked against the response as mutated in sequence.
- There is no maximum output length enforcement in code; any token or response-length behavior depends on the configured model and prompt.

Extension considerations:

- Extract prompt construction into a pure function before adding alternate actions or tests.
- Add a typed return for `getNodeNeighbours()` to improve prompt-builder type safety.
- Add handling for markdown/file/group canvas nodes before claiming support for other node types.
- Add model output validation before creating a node.
- Wrap `openai_get_reply()` in `try/catch` and map API errors to user-friendly notices.
- Consider excluding the main node from siblings to avoid duplicate `Sibling` and `Main` context.

#### Method: `async loadSettings()`

Parameters: none.

Returns: `Promise<void>`.

Behavior:

- Calls `await this.loadData()` using the Obsidian plugin persistence API.
- Merges persisted data over `DEFAULT_SETTINGS` with `Object.assign({}, DEFAULT_SETTINGS, await this.loadData())`.
- Stores the result in `this.settings`.

Extension considerations:

- Add a migration layer if settings schema versions are introduced.
- Validate persisted values because malformed plugin data could assign invalid types.

#### Method: `async saveSettings()`

Parameters: none.

Returns: `Promise<void>`.

Behavior:

- Calls `await this.saveData(this.settings)` using Obsidian plugin persistence.

Extension considerations:

- If settings become complex, debounce high-frequency controls or batch saves.

## 4.2 `src/settings.ts`

### Imports

- `PluginSettingTab`, `App`, and `Setting` from `obsidian`.
- `CanvasLLMExtendPlugin` from `./main` with a `@ts-ignore` comment because of an import/type issue around the default class export.

### Interface: `CanvasLLMExtendPluginSettings`

Defines persisted plugin settings.

Members:

- `apiKey: string`
  - API key used by the OpenAI-compatible client.
- `model: string`
  - Chat-completions model name.
- `temperature: number`
  - Sampling temperature passed to the chat completion call.
- `defaultPrompt: string`
  - Prefix/instruction prompt placed before graph context.
- `baseUrl: string`
  - Optional OpenAI-compatible base URL. Empty string means use the OpenAI package default endpoint.

### Constant: `DEFAULT_SETTINGS: CanvasLLMExtendPluginSettings`

Default values:

- `apiKey`: placeholder-looking key string.
- `model`: `gpt-3.5-turbo`.
- `temperature`: `1.0`.
- `defaultPrompt`: instruction asking the model to suggest one new outgoing node and start with `new outgoing node:`.
- `baseUrl`: empty string.

Security note:

- The default API key value is not a valid production secret pattern for this repository, but UI descriptions should still make clear that users must provide their own API key or local-compatible service.

### Class: `CanvasLLMExtendPluginSettingsTab extends PluginSettingTab`

Settings UI class rendered inside Obsidian's plugin settings area.

#### Members

- `plugin: CanvasLLMExtendPlugin`
  - Reference to the main plugin instance so UI controls can read and persist settings.

#### Constructor: `constructor(app: App, plugin: CanvasLLMExtendPlugin)`

Parameters:

- `app: App`: Obsidian application instance passed to the parent `PluginSettingTab`.
- `plugin: CanvasLLMExtendPlugin`: main plugin instance.

Behavior:

- Calls `super(app, plugin)`.
- Assigns `this.plugin = plugin`.

#### Method: `display(): void`

Renders all settings controls.

Parameters: none.

Returns: `void`.

Behavior:

1. Gets `containerEl` from the settings tab instance.
2. Clears it with `containerEl.empty()`.
3. Adds an **OpenAI API key** text setting:
   - Placeholder: `DEFAULT_SETTINGS.apiKey`
   - Value: `this.plugin.settings.apiKey`
   - `onChange(value)`: assigns `value` to `plugin.settings.apiKey` and saves.
4. Adds an **OpenAI model** text setting:
   - Placeholder: `DEFAULT_SETTINGS.model`
   - Value: `this.plugin.settings.model`
   - `onChange(value)`: assigns `value` to `plugin.settings.model` and saves.
5. Adds an **OpenAI temperature** slider:
   - Initial value: `this.plugin.settings.temperature`
   - Limits: min `0`, max `2`, step `0.1`
   - Dynamic tooltip enabled.
   - `onChange(value)`: assigns `value` to `plugin.settings.temperature` and saves.
6. Adds an **OpenAI base URL** text setting:
   - Placeholder: `http://localhost:8080/v1`
   - Value: `this.plugin.settings.baseUrl`
   - `onChange(value)`: trims whitespace, assigns to `plugin.settings.baseUrl`, and saves.
7. Adds a **Prompt** text area:
   - Placeholder: `DEFAULT_SETTINGS.defaultPrompt`
   - Value: `this.plugin.settings.defaultPrompt`
   - `onChange(value)`: assigns `value` to `plugin.settings.defaultPrompt` and saves.

Events/callbacks:

- Each setting control registers an asynchronous `onChange` callback that persists immediately with `saveSettings()`.

Extension considerations:

- Additional model parameters such as `max_tokens`, `top_p`, or system prompt can be added by extending the interface, defaults, UI, and OpenAI request.
- For secret handling, replace plain text API key entry with a UI pattern that reduces accidental display where possible.
- For multiple actions, introduce an array of prompt profiles rather than one `defaultPrompt` string.

## 4.3 `src/openai-utils.ts`

### Import

- Default `OpenAI` client from the `openai` package.

### Function: `openai_get_reply(prompt, model, temperature, apiKey, baseUrl?)`

Signature:

```ts
export async function openai_get_reply(
    prompt: string,
    model: string,
    temperature: number,
    apiKey: string,
    baseUrl?: string
): Promise<string | null>
```

Parameters:

- `prompt: string`
  - User message content sent to the chat completion endpoint.
- `model: string`
  - Model identifier passed directly to `chat.completions.create()`.
- `temperature: number`
  - Sampling temperature passed directly to the model.
- `apiKey: string`
  - API key passed to the OpenAI client constructor.
- `baseUrl?: string`
  - Optional OpenAI-compatible endpoint. If supplied and non-empty after trimming, it becomes the client's `baseURL`.

Returns:

- `Promise<string | null>` containing `chatCompletion.choices[0].message.content`.

Behavior:

1. Builds `openaiOptions` with:
   - `apiKey`
   - `dangerouslyAllowBrowser: true`
2. If `baseUrl` has non-whitespace content, sets `openaiOptions.baseURL` to `baseUrl.trim()`.
3. Constructs `const openai = new OpenAI(openaiOptions)`.
4. Calls `openai.chat.completions.create({ messages, model, temperature })` with a single user message.
5. Returns the first choice message content.

Important implementation details:

- A new OpenAI client is created for each LLM request.
- `openaiOptions` is typed as `any`, which bypasses OpenAI constructor option type checking.
- There is no system message; all instructions are sent as a user message.
- There is no streaming support.
- There is no explicit timeout, cancellation, retry, rate-limit handling, or error mapping.
- The return type allows `null` because OpenAI message content can be nullable.

Extension considerations:

- Add a typed options object instead of `any`.
- Reuse a client when settings do not change, or construct a service class that owns the client.
- Add `max_tokens` or equivalent output controls.
- Add support for streaming if future UI should show progressive generation.
- Add provider abstraction for non-OpenAI APIs with non-chat-completions semantics.
- Add cancellation through `AbortController` if the OpenAI SDK and Obsidian UI path support it.

## 4.4 `src/obsidian-canvas-utils.ts`

This module contains the low-level canvas graph and geometry logic. It uses internal Obsidian Canvas runtime objects more directly than the rest of the plugin, including APIs not fully represented by public typings.

### Imports

- `CanvasNodeData` and `CanvasData` from `obsidian/canvas`.
- `notifyError(message)` from `./obsidian-helpers`.

### Function: `isCanvasNodeData(node: unknown): node is CanvasNodeData`

Parameters:

- `node: unknown`: arbitrary value, usually from a workspace event.

Returns:

- Type predicate indicating whether TypeScript may treat `node` as `CanvasNodeData`.

Behavior:

- Returns true when `node` is non-null, is an object, and has a `canvas` property.

Important implementation details:

- This is a structural guard, not a complete runtime validation.
- It does not prove that `.id`, `.text`, `.x`, `.width`, `.canvas.nodes`, or `.canvas.edges` exist.

Extension considerations:

- Strengthen the guard if supporting multiple canvas event payloads.
- Add guards for text-node-specific behavior before reading `.text`.

### Interface: `Box`

Internal geometry shape.

Members:

- `x: number`
- `y: number`
- `width: number`
- `height: number`

### Function: `isBox(node: unknown): node is Box`

Parameters:

- `node: unknown`: arbitrary value.

Returns:

- Type predicate indicating whether the value has `x`, `y`, `width`, and `height` properties.

Behavior:

- Checks for non-null object and property presence.

Important implementation details:

- It does not verify that properties are numbers.

Extension considerations:

- Validate numeric values before geometry operations to avoid `NaN` placement bugs.

### Function: `getNodeNeighbours(node: CanvasNodeData)`

Parameters:

- `node: CanvasNodeData`: selected canvas node.

Returns:

- Object with:
  - `incoming`: array of nodes that have edges into `node`.
  - `outgoing`: array of nodes that `node` points to.

Behavior:

1. Initializes `incoming` and `outgoing` arrays.
2. Iterates over `node.canvas.edges.values()`.
3. For each edge:
   - If `edge.from.node.id == node.id`, pushes `edge.to.node` to `outgoing`.
   - Else if `edge.to.node.id == node.id`, pushes `edge.from.node` to `incoming`.
4. Returns both arrays.

Important implementation details:

- Uses loose equality (`==`) for id comparison.
- The arrays are currently inferred as `any[]` or broad types unless TypeScript can infer from Obsidian canvas edge typings.
- It considers directed edges only. Undirected or non-arrow semantics are not modeled separately.

Extension considerations:

- Add explicit return type such as `{ incoming: CanvasNodeData[]; outgoing: CanvasNodeData[] }`.
- Add a helper for siblings to avoid repeated graph traversals in prompt construction.
- Add filtering by node type when supporting file/group nodes.

### Function: `generate_id(): string`

Private helper.

Parameters: none.

Returns:

- Random 16-character hexadecimal string.

Behavior:

- Runs a 16-iteration loop.
- Each iteration uses `(16 * Math.random() | 0).toString(16)` to produce one hex character.
- Joins all characters into the id.

Important implementation details:

- It is not cryptographically secure.
- Collision probability is low for casual use, but not impossible.
- It is used only for manually constructed edge ids, not text node ids created by Obsidian.

Extension considerations:

- Prefer an Obsidian-provided id generator if one exists.
- Track collisions if adding batch creation.

### Function: `overlaps(node1: unknown, node2: unknown): boolean`

Private helper.

Parameters:

- `node1: unknown`: expected to be a `Box`.
- `node2: unknown`: expected to be a `Box`.

Returns:

- `true` if rectangular boxes overlap; otherwise `false`.

Behavior:

1. Validates each argument with `isBox()`.
2. On invalid input, reports an error and returns `false`.
3. Performs axis-aligned rectangle separation checks:
   - `node1` entirely right of `node2` means no overlap.
   - `node1` entirely left of `node2` means no overlap.
   - `node1` entirely below `node2` means no overlap.
   - `node1` entirely above `node2` means no overlap.
4. If none of those no-overlap cases apply, returns `true`.

Important implementation details:

- Touching edges are treated as overlapping because comparisons use `>` and `<`, not `>=` and `<=`.
- Error strings use regular quotes with `${node1}` and `${node2}`, so interpolation does not happen.

Extension considerations:

- Decide whether edge-touching should count as overlap.
- Use template literals for diagnostics if the object identity matters.
- Consider canvas zoom/transform only if coordinates are not already canvas-space.

### Function: `findEmptySpace(neighbor_node, node_to_fit, distance_between, updown, leftright)`

Private helper.

Parameters:

- `neighbor_node: CanvasNodeData`
  - Anchor node around which candidate positions are searched.
- `node_to_fit: CanvasNodeData`
  - Newly created node that needs a location.
- `distance_between: number`
  - Gap to place between anchor and new node.
- `updown: boolean`
  - Whether to consider below and above positions.
- `leftright: boolean`
  - Whether to consider right and left positions.

Returns:

- `true` if a non-overlapping location was found and applied with `moveTo()`.
- `false` otherwise.

Behavior:

1. Builds a candidate `positions` array.
2. If `leftright` is true, pushes:
   - Right of neighbor: `x = neighbor.x + neighbor.width + distance_between`, `y = neighbor.y`.
   - Left of neighbor: `x = neighbor.x - distance_between - neighbor.width`, `y = neighbor.y`.
3. If `updown` is true, pushes:
   - Below neighbor: `x = neighbor.x`, `y = neighbor.y + neighbor.height + distance_between`.
   - Above neighbor: `x = neighbor.x`, `y = neighbor.y - neighbor.height - distance_between`.
4. For each candidate position:
   - Builds `nodetemp` with candidate `x`, `y`, and the new node's width/height.
   - Iterates over all nodes in `neighbor_node.canvas.nodes.values()`.
   - Skips the new node itself.
   - If any existing node overlaps `nodetemp`, tries the next candidate.
   - If none overlap, calls `node_to_fit.moveTo(...)` and returns `true`.
5. Returns `false` if no candidate is open.

Important implementation details:

- The left placement uses `neighbor_node.width` instead of `node_to_fit.width`, so left spacing is accurate only when widths match. Current callers create the child with the selected node's width, so this often works for the selected node but can be wrong around differently sized siblings.
- Candidate order is controlled by `addNodeChild()` through `updown` and `leftright` flags.
- Search is local and finite; it does not spiral or expand if nearby slots are occupied.

Extension considerations:

- Replace with a more general layout algorithm for dense canvases.
- Include viewport-aware placement if new nodes should appear near visible area.
- Parameterize `distance_between` as a setting.

### Function: `fitToText(node, width, textsizenode)`

Private helper.

Parameters:

- `node: CanvasNodeData`
  - Newly created text node to resize.
- `width: number`
  - Target node width.
- `textsizenode: CanvasNodeData`
  - Existing canvas node whose DOM style is used for text measurement.

Returns: `void`.

Behavior:

1. Reads computed style from `textsizenode.nodeEl`.
2. Derives `lineHeight` from `compStyle.lineHeight`; falls back to `19`.
3. Reuses or creates a hidden `canvas` element stored on `fitToText.htmlCanvas` via a `@ts-ignore` dynamic property.
4. Gets a 2D drawing context.
5. Sets `context.font` to `compStyle.font`.
6. Measures `node.text` line width.
7. Estimates height as `((lineWidth / width) + 2) * lineHeight`.
8. Calls `node.resize({ width, height })`.

Important implementation details:

- The algorithm estimates wrapping by dividing single-line pixel width by target width.
- It does not account for explicit newlines, markdown rendering, padding, lists, code blocks, or Obsidian canvas text rendering details.
- The code assumes `getContext('2d')` returns a non-null context.
- The dynamic static property on a function requires `@ts-ignore`.

Extension considerations:

- Add a typed module-level `let htmlCanvas: HTMLCanvasElement | undefined` instead of mutating the function object.
- Account for multiline text and markdown formatting.
- Consider using Obsidian's own resize behavior if a public API becomes available.

### Function: `createNode(text, width, canvas, textsizenode)`

Parameters:

- `text: string`
  - Text content for the new canvas node.
- `width: number`
  - Target width, usually inherited from the selected node.
- `canvas: CanvasData`
  - Canvas data/controller object from the selected node.
- `textsizenode: CanvasNodeData`
  - Existing node used as style reference for sizing.

Returns:

- `CanvasNodeData` for the new text node.

Behavior:

1. Calls `canvas.createTextNode()` with:
   - `text`
   - initial position `{ x: 0, y: 0 }`
   - initial size `{ width: 1, height: 1 }`
   - `save: true`
   - `focus: false`
2. Calls `canvas.addNode(cn)`.
3. Calls `fitToText(cn, width, textsizenode)`.
4. Calls `cn.attach()`.
5. Returns the created node.

Important implementation details:

- Initial creation happens at `(0, 0)` and placement occurs later.
- The node is added before being moved to its final location.
- The function only creates text nodes.

Extension considerations:

- Add a typed options object when adding node type, dimensions, color, or focus behavior.
- If placement can fail, consider delaying permanent add/save until after a location is found.

### Function: `createEdge(from, to, related, canvas)`

Parameters:

- `from: CanvasNodeData`
  - Source node for the edge.
- `to: CanvasNodeData`
  - Target node for the edge.
- `related: CanvasNodeData`
  - Existing node used to find a reference edge from `from` to `related`.
- `canvas: CanvasData`
  - Canvas to mutate.

Returns: `void`.

Behavior:

1. Iterates over `canvas.edges.values()`.
2. If `related` is falsy, breaks immediately.
3. Otherwise, searches for an edge where:
   - `edge.from.node.id == from.id`
   - `edge.to.node.id == related.id`
4. If no reference edge is found, reports `Please add at least one arrow to your canvas.` and returns.
5. Constructs a new edge using `new edge.constructor(...)` with:
   - The canvas object.
   - A random id from `generate_id()`.
   - A `from` endpoint using `edge.from.side`, `node: from`, and `end: "none"`.
   - A `to` endpoint using `edge.to.side`, `node: to`, and `end: "arrow"`.
6. Calls `canvas.addEdge(e)`.
7. Calls `e.attach()`.
8. Calls `e.render()`.

Important implementation details:

- This relies on an existing edge's constructor because the public Obsidian Canvas API does not provide an explicit edge constructor in the available typings.
- The `related` parameter is typed as required but the function checks `if (!related)`, suggesting it was originally intended to be optional.
- If the selected node has no existing outgoing edge to the related placement anchor, edge creation can fail even after the new node is created and placed.
- The edge sides are copied from the reference edge, which helps preserve visual directionality for sibling-based placement.

Extension considerations:

- Add an explicit `related?: CanvasNodeData` type if no-reference behavior is desired.
- Implement side calculation based on relative positions to avoid requiring an existing arrow.
- Remove the newly created node if edge creation fails.

### Function: `addNodeChild(node, nodetext)`

Parameters:

- `node: CanvasNodeData`
  - Existing selected canvas node that will become the parent/source.
- `nodetext: string`
  - Text content for the new child node.

Returns: `void`.

Behavior:

1. Calls `createNode(nodetext, node.width, node.canvas, node)` to create a new text node sized to match the selected node width.
2. Calls `getNodeNeighbours(node)` to get existing outgoing siblings.
3. Initializes `altorder` with one placement strategy:
   - Try above/below the selected node.
4. For each outgoing sibling:
   - Adds an above/below strategy for that sibling to the beginning of the list.
   - Adds a left/right strategy for that sibling to the end of the list.
5. Adds a final left/right strategy for the selected node.
6. Iterates over `altorder`.
7. For each strategy, calls `findEmptySpace(strategy.node, n, 15, strategy.updown, strategy.leftright)`.
8. On first successful placement:
   - Calls `createEdge(node, n, strategy.node, node.canvas)`.
   - Calls `n.render()`.
   - Calls `node.canvas.requestSave(true)`.
   - Returns.
9. If no placement succeeds, reports `No place for child node`.

Important implementation details:

- The hard-coded spacing between nodes is `15`.
- The new node is created before placement is known. If placement fails, the function reports an error but does not remove the orphan node.
- The placement strategy tries to keep the new node near existing outgoing nodes and the selected node.
- `createEdge()` may report an error and return without creating an edge, but `addNodeChild()` still renders and saves after calling it because it does not receive a success/failure value.

Extension considerations:

- Make placement and edge creation transactional: create, place, edge, save; roll back on failure.
- Return the created node or a result object to support tests and downstream features.
- Convert magic number `15` into a named constant or user setting.
- Add batch creation support by repeating placement with progressively expanded search radius.

## 4.5 `src/obsidian-helpers.ts`

### Function: `notifyError(message: string)`

Parameters:

- `message: string`: error text for both user-visible and developer-visible output.

Returns: `void`.

Behavior:

- Creates a new Obsidian `Notice(message, 5000)` visible for 5 seconds.
- Writes `llm-canvas-utils: ${message}` to `console.error()`.

Extension considerations:

- Add `notifyInfo()` or `notifySuccess()` helpers for non-error status.
- Add an error-code or logging-level convention if debugging grows.
- Consider including the original exception object where available.

## 4.6 `src/obsidian.d.ts`

This file augments Obsidian's TypeScript declarations.

### Module augmentation: `declare module "obsidian"`

Adds a `Workspace.on()` overload with:

- `name: string`
- `callback: (menu: Menu, arg: unknown) => void`
- `ctx?: unknown`
- return type: `EventRef`

Purpose:

- Lets `src/main.ts` register `canvas:node-menu` without TypeScript rejecting the callback signature.

Extension considerations:

- Narrow `name` to the known event string if possible: `"canvas:node-menu"`.
- Add a better event argument type if Obsidian Canvas typings expose one in future versions.

## 5. End-to-End Runtime Sequence

## 5.1 Plugin startup

1. Obsidian reads `manifest.json` and loads `main.js` generated from `src/main.ts`.
2. Obsidian constructs `CanvasLLMExtendPlugin`.
3. Obsidian calls `onload()`.
4. `onload()` loads settings from plugin data storage and merges them with defaults.
5. `onload()` registers the canvas context menu event.
6. `onload()` registers the settings tab.

Failure mode:

- If settings load fails, the plugin reports an error but continues to register the event and settings tab. If `this.settings` is not assigned because `loadSettings()` threw, later settings access can fail.

## 5.2 User invokes LLM Extend

1. User right-clicks a canvas node.
2. Obsidian emits `canvas:node-menu` with a menu and node-like payload.
3. Plugin adds `LLM Extend` to that menu.
4. User clicks `LLM Extend`.
5. Click handler calls `extendNode(node)`.

Failure mode:

- If the payload is not structurally recognized as a canvas node, the plugin reports `Node is not Canvas Node`.

## 5.3 Prompt construction

Given selected node `N`, the plugin builds:

```text
{defaultPrompt}
Sibling: {text of outgoing child of each incoming parent}
Incoming: {text of direct incoming node}
Main: {text of N}
Outgoing: {text of direct outgoing node}
```

Graph semantics:

- Incoming nodes are direct predecessors of `N`.
- Outgoing nodes are direct successors of `N`.
- Siblings are successors of each predecessor of `N`.

Failure mode:

- If a node lacks `.text`, prompt lines can include `undefined`.

## 5.4 LLM request

1. `openai_get_reply()` creates an OpenAI client with configured key and optional base URL.
2. It sends one chat user message containing the full prompt.
3. It passes through configured model and temperature.
4. It returns the first choice's message content.

Failure modes:

- API/network errors are not caught locally.
- Empty choices or missing first choice are not guarded.
- `message.content` may be `null`, which `extendNode()` handles by reporting failure.

## 5.5 Canvas mutation

1. The model response is stripped of known leading labels and trimmed.
2. `addNodeChild()` creates a text node at temporary position `(0, 0)`.
3. `fitToText()` estimates and applies a node height.
4. The plugin searches candidate positions around outgoing siblings and the selected node.
5. When a clear slot is found, the new node is moved there.
6. `createEdge()` copies edge-side information from a related existing edge and constructs a new arrow from selected node to new node.
7. The new node is rendered and the canvas is saved.

Failure modes:

- Dense canvases can exhaust the finite candidate list.
- Placement failure leaves the created node in the canvas.
- Edge creation failure can leave a node without an edge.
- If the selected node has no outgoing arrows, `createEdge()` can fail with `Please add at least one arrow to your canvas.`

## 6. Current Design Strengths

- Small, understandable module boundaries.
- Main workflow is isolated in `CanvasLLMExtendPlugin.extendNode()`.
- Canvas mutation details are isolated in `obsidian-canvas-utils.ts`.
- Settings UI persists changes immediately.
- OpenAI-compatible `baseUrl` already enables local servers and compatible hosted providers.
- Build tooling is minimal and appropriate for an Obsidian plugin.

## 7. Technical Debt and Risk Register

| Area | Risk | Impact | Recommended remediation |
| --- | --- | --- | --- |
| Error handling | LLM API exceptions are not caught in `extendNode()` or `openai_get_reply()`. | User can see uncaught errors and no Obsidian notice. | Wrap request and canvas mutation steps with targeted `try/catch`; preserve console details. |
| Canvas API usage | `createEdge()` uses `edge.constructor` and `@ts-ignore`. | Can break on Obsidian internal changes. | Encapsulate edge creation behind one adapter and add version notes/tests. |
| Placement rollback | Failed placement or edge creation can leave orphan nodes. | Canvas can be mutated despite operation failure. | Return success/failure from placement and edge functions; remove new node on failure. |
| Typing | Several helpers rely on loose structural checks or implicit array types. | More difficult to safely extend. | Add explicit return types and stronger guards. |
| Prompt construction | Prompt builder is embedded in `extendNode()`. | Hard to test and hard to add new actions. | Extract `buildExtendPrompt(node, settings)` or graph-context service. |
| Node type support | Code assumes text-bearing nodes. | File/group/media nodes may produce bad prompt text. | Add canvas-node type detection and text extraction strategy. |
| OpenAI client lifecycle | New client created per request. | Simpler but inefficient and harder to configure globally. | Introduce an `LlmClient` wrapper keyed by settings. |
| Settings validation | Persisted values are merged without validation. | Bad plugin data can cause runtime errors. | Validate and normalize after `loadData()`. |
| Lint command | ESLint config appears absent. | `npm run lint` cannot be used reliably. | Add ESLint configuration or remove script. |

## 8. Feature Extension Pathways

## 8.1 Add support for multiple prompt-based actions

Goal: Let users choose actions such as "extend", "summarize branch", "generate counterargument", or "create questions".

Recommended architecture:

1. Introduce an action model:

```ts
interface CanvasLlmAction {
    id: string;
    title: string;
    promptTemplate: string;
    outputMode: "single-child" | "multiple-children" | "replace-node" | "clipboard";
}
```

2. Add `actions: CanvasLlmAction[]` and `defaultActionId` to settings.
3. Extract prompt construction:

```ts
function buildPrompt(action: CanvasLlmAction, context: CanvasGraphContext): string
```

4. Extract graph collection:

```ts
function collectGraphContext(node: CanvasNodeData): CanvasGraphContext
```

5. In `onload()`, add one menu item per action or add a submenu.
6. Dispatch by `outputMode` after receiving the LLM response.

Files likely touched:

- `src/main.ts`
- `src/settings.ts`
- New `src/prompt-utils.ts` or `src/actions.ts`
- Potentially `src/obsidian-canvas-utils.ts` for output modes

Key engineering concern:

- Avoid hard-coding one prompt transcript format into every action.

## 8.2 Support non-text canvas nodes

Goal: Include file nodes, links, groups, or other canvas node variants in context.

Recommended architecture:

1. Create a node text extraction function:

```ts
function getCanvasNodeText(node: CanvasNodeData): string | null
```

2. Detect node kind from Obsidian Canvas properties.
3. For file nodes, decide whether to include:
   - File path only.
   - File title/alias.
   - File contents from the vault.
   - A bounded excerpt.
4. For group nodes, include label text if available.
5. Modify prompt builder to skip unsupported nodes or include typed placeholders.

Files likely touched:

- `src/obsidian-canvas-utils.ts`
- `src/main.ts`
- Possibly new `src/canvas-node-text.ts`

Key engineering concern:

- Reading file contents introduces asynchronous vault I/O and token-budget management.

## 8.3 Add response length and token controls

Goal: Prevent excessively large generated nodes.

Recommended architecture:

1. Extend settings:

```ts
maxTokens: number;
```

2. Add a settings slider or text input.
3. Pass `max_tokens` or provider-appropriate output limit in `openai_get_reply()`.
4. Add client-side truncation as a final guard before `addNodeChild()`.

Files likely touched:

- `src/settings.ts`
- `src/openai-utils.ts`
- `src/main.ts`

Key engineering concern:

- OpenAI-compatible providers may differ in parameter naming or behavior.

## 8.4 Add robust layout and batch node creation

Goal: Support generating multiple child nodes and placing them cleanly.

Recommended architecture:

1. Change the LLM output parser to return an array of node texts.
2. Create all nodes only after a placement plan is found, or create them with rollback support.
3. Replace finite candidate search with a layout planner:
   - Ring/spiral around selected node.
   - Directional branch layout.
   - Grid placement below outgoing siblings.
4. Return placement results from helpers.

Potential API shape:

```ts
interface PlacementResult {
    ok: boolean;
    position?: { x: number; y: number };
    reason?: string;
}
```

Files likely touched:

- `src/obsidian-canvas-utils.ts`
- `src/main.ts`
- New `src/canvas-layout.ts`

Key engineering concern:

- Canvas mutation should become transactional to avoid partial writes.

## 8.5 Add streaming or progress UI

Goal: Improve user feedback during slow LLM calls.

Recommended architecture:

1. Add a notice when generation starts.
2. Optionally create a temporary placeholder node such as `Generating...`.
3. Use a streaming API in `openai-utils.ts` if supported.
4. Update the placeholder node as chunks arrive.
5. Replace or remove placeholder on error.

Files likely touched:

- `src/openai-utils.ts`
- `src/main.ts`
- `src/obsidian-canvas-utils.ts`
- `src/obsidian-helpers.ts`

Key engineering concern:

- Streaming updates must not call expensive canvas render/save operations too frequently.

## 8.6 Add provider abstraction

Goal: Support OpenAI, local OpenAI-compatible servers, and future providers with clean boundaries.

Recommended architecture:

```ts
interface LlmRequest {
    prompt: string;
    model: string;
    temperature: number;
    maxTokens?: number;
}

interface LlmProvider {
    complete(request: LlmRequest): Promise<string | null>;
}
```

Then implement:

- `OpenAIChatCompletionsProvider`
- `OpenAICompatibleProvider`
- Future providers as needed

Files likely touched:

- Replace or expand `src/openai-utils.ts`
- `src/settings.ts`
- `src/main.ts`

Key engineering concern:

- Keep provider-specific settings isolated instead of growing one flat settings interface indefinitely.

## 8.7 Add automated tests

Goal: Make feature work safer before refactors.

Recommended first tests:

- Pure prompt builder tests.
- Prefix stripping tests.
- Graph neighbor extraction tests using small mocked canvas objects.
- Rectangle overlap tests.
- Placement candidate tests with mocked nodes.
- OpenAI utility tests with mocked SDK client or provider interface.

Suggested refactor-first steps:

1. Extract pure functions from `main.ts` and `obsidian-canvas-utils.ts`.
2. Add a test runner such as Vitest.
3. Keep Obsidian runtime integration behind thin adapters.

Files likely touched:

- `package.json`
- `tsconfig.json`
- New test files under `test/` or `src/**/*.test.ts`

Key engineering concern:

- Obsidian's runtime APIs are not naturally available in Node test environments, so isolate pure logic from host calls.

## 9. Recommended Near-Term Refactor Plan

### Phase 1: Safety and typing

1. Add explicit return types for `getNodeNeighbours()`, `createEdge()`, `addNodeChild()`, and placement helpers.
2. Replace `openaiOptions: any` with the OpenAI SDK option type.
3. Add `try/catch` around the LLM request in `extendNode()`.
4. Make `createEdge()` return `boolean` and have `addNodeChild()` only save after success.
5. Roll back newly created nodes on placement or edge failure.

### Phase 2: Extract pure feature logic

1. Move prompt construction into a pure helper.
2. Move response prefix stripping into a pure helper.
3. Add tests for both helpers.
4. Add a graph-context data structure that separates Obsidian nodes from prompt-ready text.

### Phase 3: Product extensibility

1. Add multiple actions/prompt profiles.
2. Add max token/output length control.
3. Add richer node type extraction.
4. Improve layout for multiple generated nodes.

## 10. Practical Development Notes

- Start development with `npm install` if dependencies are absent.
- Use `npm run build` as the primary correctness check because it runs TypeScript and production bundling.
- Do not rely on `npm run lint` until ESLint configuration is added.
- Most new feature work should avoid touching `createEdge()` unless necessary; it depends on Obsidian internals and should be changed carefully.
- Any feature that reads vault file contents should be asynchronous all the way from click handler to prompt builder.
- Any feature that changes node placement should add rollback behavior first, because current mutation order can leave orphan nodes.

## 11. Suggested First Issues for Contributors

1. **Extract and test prompt building**
   - Difficulty: low.
   - Value: high.
   - Risk: low.

2. **Catch and report OpenAI API errors**
   - Difficulty: low.
   - Value: high.
   - Risk: low.

3. **Make edge creation return success/failure**
   - Difficulty: medium.
   - Value: high.
   - Risk: medium because it touches canvas mutation.

4. **Add max token setting**
   - Difficulty: medium.
   - Value: medium.
   - Risk: low to medium depending on provider compatibility.

5. **Support multiple prompt actions**
   - Difficulty: medium to high.
   - Value: high.
   - Risk: medium because it changes settings shape and menu behavior.

6. **Add non-text node support**
   - Difficulty: high.
   - Value: high.
   - Risk: high because it touches Obsidian vault I/O and token budgeting.
