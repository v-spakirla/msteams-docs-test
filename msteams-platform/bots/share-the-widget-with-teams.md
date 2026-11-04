---
title: Share the widget with Teams
description: Learn how to connect an HTML widget to Teams, report its height, call bot tools, and display widgets from MCP servers.
ms.topic: how-to
ms.localizationpriority: medium
ms.date: 11/04/2026
---

# Share the widget with Teams

An HTML widget is an HTML document embedded in a Teams chat message. To make it interactive, connect the widget to its Teams host, report its content height, and send tool requests to the bot. This lets users complete actions and see results inside the conversation without leaving the chat.

In this article, *share* means connecting the embedded widget to Teams, not forwarding a message or publishing an app. Continue the dice-roller example from [Send the widget](send-the-widget.md) to add a button that calls the bot and a results list that grows with each roll.

> [!NOTE]
> HTML widgets are a preview feature. The Teams SDK marks the related APIs as experimental, and they may change.

## Prerequisites

- Complete [Send the widget](send-the-widget.md), including the HTML shell, payload builder, and Markdown helper.
- Use a working TypeScript Teams SDK bot available on a public HTTPS endpoint.
- For the optional MCP integration, use a bot that already calls MCP servers through a `Client` from `@modelcontextprotocol/sdk`.

## User experience

The completed widget displays a **Roll a 6-sided die** button. When a user selects it, the widget asks the bot to roll the die and adds the result to a list. For example, three rolls might display **You rolled 4.**, **You rolled 2.**, and **You rolled 6.** The widget grows to fit the list without a scroll bar.

The button is disabled while a roll is in progress. Errors appear in the widget and are announced to screen readers.

## Project layout

Continue with these files in your existing bot project. Keep `dice-widget.ts` and `widget-markdown.ts` from [Send the widget](send-the-widget.md#project-layout).

```text
widgets/
  dice-widget.html     # Add the connection, sizing, and tool-calling code
src/
  bot/
    dice-tool.ts       # Handles tool calls from the widget
    correlation.ts     # Optional: identifies which widget made a tool call
    mcp-widgets.ts     # Optional: shows widgets from MCP servers
```

Follow this master procedure to connect the widget to Teams:

1. [Connect the widget to Teams](#connect-the-widget-to-teams).
1. [Report the widget's height](#report-the-widgets-height).
1. [Call a tool from the widget](#call-a-tool-from-the-widget).
1. [Handle tool calls in the bot](#handle-tool-calls-in-the-bot).
1. Optionally, [display widgets from MCP servers](#widgets-from-mcp-servers).
1. [Put it all together](#put-it-all-together).

## Implement widget interaction with Teams

### Connect the widget to Teams

Before Teams accepts anything else from the widget, the widget must complete a handshake:

1. Send the `ui/initialize` request.
1. Wait for its response.
1. Send the `ui/notifications/initialized` notification.

Add the following code to the `<script>` element in `dice-widget.html`. It defines a helper that sends requests and waits for their responses, and the `connect` function that performs the handshake.

```javascript
const REQUEST_TIMEOUT_MS = 15000;
let requestCount = 0;
// Set by connect() and shared by later tool calls.
let connection;

/**
 * Sends a message to Teams.
 * @param {object} message The message, without the "jsonrpc" field.
 * @returns {void}
 */
function postToHost(message) {
  window.parent.postMessage({ jsonrpc: "2.0", ...message }, "*");
}

/**
 * Normalizes responses that arrive as objects or JSON strings.
 * @param {unknown} data The data of a "message" event.
 * @returns {object | undefined} The message, or undefined if it is not an object.
 */
function readMessage(data) {
  if (typeof data === "string") {
    try {
      return JSON.parse(data);
    } catch {
      return undefined;
    }
  }

  return typeof data === "object" && data !== null ? data : undefined;
}

/**
 * Sends a request to Teams and waits for its response.
 * @param {string} method The request's method name.
 * @param {object} params The request's parameters.
 * @returns {Promise<object>} The response's result.
 */
function sendRequest(method, params) {
  requestCount += 1;
  const id = "request-" + requestCount;

  return new Promise((resolve, reject) => {
    const timer = setTimeout(
      () => finish(new Error(method + " timed out.")),
      REQUEST_TIMEOUT_MS,
    );

    function finish(error, result) {
      clearTimeout(timer);
      window.removeEventListener("message", onMessage);

      if (error) {
        reject(error);
      } else {
        resolve(result ?? {});
      }
    }

    function onMessage(event) {
      // Only trust responses from the Teams host.
      if (event.source !== window.parent) {
        return;
      }

      const message = readMessage(event.data);

      if (message?.jsonrpc !== "2.0" || message.id !== id) {
        return;
      }

      if (message.error) {
        finish(
          new Error(method + " failed (code " + message.error.code + ")."),
        );
      } else {
        finish(undefined, message.result);
      }
    }

    // Listen before sending so a response cannot be missed.
    window.addEventListener("message", onMessage);
    postToHost({ id, method, params });
  });
}

/**
 * Performs the handshake once. A failed handshake can be retried.
 * @param {string} appName The widget's name, sent to Teams.
 * @returns {Promise<void>} Completes when the handshake is done.
 */
function connect(appName) {
  connection ??= sendRequest("ui/initialize", {
    protocolVersion: "2026-01-26",
    appInfo: { name: appName, version: "1.0.0" },
    appCapabilities: {},
  }).then(
    () => {
      postToHost({ method: "ui/notifications/initialized", params: {} });
    },
    (error) => {
      connection = undefined;
      throw error;
    },
  );

  return connection;
}
```

- `REQUEST_TIMEOUT_MS`: Limits how long requests wait for a host response. Set this to `15000` for the example so a missing response rejects the promise.
- `protocolVersion`: Identifies the protocol implemented by this widget's handshake. Set this to `"2026-01-26"` for the protocol used in this example.
- `appInfo`: Identifies the widget during the connection to Teams. Set `name` to the widget's name and `version` to `"1.0.0"` for this example.
- `appCapabilities`: Declares optional capabilities that the widget supports to Teams. Set this to `{}` because the dice widget declares none.

Requests use unique IDs so the widget can match responses. Calling `connect` again returns the same promise unless the handshake failed, in which case it starts a new attempt.

### Report the widget's height

**A widget's height must be driven by its content.**

Teams sizes the widget's iframe to the height reported with the `ui/notifications/size-changed` notification. Report the height after the handshake and every time the content height changes.

Add `reportHeight` to the `<script>` element:

```javascript
/**
 * Reports the height of `root` now and whenever it changes.
 * @param {HTMLElement} root The element containing all widget content.
 * @returns {() => void} A function that stops reporting.
 */
function reportHeight(root) {
  let lastHeight;

  function report() {
    const height = root.getBoundingClientRect().height;

    // Skip duplicates unless the iframe is still too short.
    if (height === lastHeight && window.innerHeight >= height) {
      return;
    }

    lastHeight = height;
    postToHost({
      method: "ui/notifications/size-changed",
      params: { height },
    });
  }

  const observer = new ResizeObserver(report);

  observer.observe(root);
  report();

  return () => observer.disconnect();
}
```

- `root`: Identifies the element containing all visible widget content. Pass the `widget-root` element so every content change is measured.
- `height`: Supplies the exact rendered content height in CSS pixels. Send `root.getBoundingClientRect().height` without rounding so Teams doesn't clip content or leave extra space.

Because the iframe is sized from the widget, the widget can't be sized from the iframe. To keep the widget's height driven by its content:

- Put all content inside one root element, and measure that element with `getBoundingClientRect().height`. Don't measure `document.documentElement.scrollHeight` or `document.body.scrollHeight`: they are never smaller than the iframe, so the widget could grow but never shrink.
- Report the exact height without rounding. Rounding up leaves a strip below the widget, and rounding down clips it.
- Don't use `100vh`, or `height: 100%` on `html`, `body`, or the root element.
- Keep all content inside the root in the normal page flow. Content placed outside the root with negative margins or `position: fixed` isn't measured and can be clipped.
- Remove margins around the root. The [HTML shell](send-the-widget.md#create-the-widget-file) sets `html, body { margin: 0; }` and gives the root `display: flow-root` so that its first and last children's margins are measured.

The observer reports changes caused by new content, loaded fonts or images, or text that wraps differently. The dice widget's height is driven by its button, list of rolls, and error message.

### Call a tool from the widget

A widget asks its bot to do something by calling a tool with a `name` and an `arguments` object. The bot that sent the message handles the call as described in [Handle tool calls in the bot](#handle-tool-calls-in-the-bot).

#### Add the callTool function

Add `callTool` to the `<script>` element:

```javascript
/**
 * Calls a tool on the bot and returns the bot's result.
 * @param {string} name The tool name that the bot recognizes.
 * @param {object} args The arguments received as activity.value.arguments.
 * @returns {Promise<{content?: Array<{type: string, text?: string}>,
 *   structuredContent?: object, isError?: boolean}>} The bot's result.
 */
async function callTool(name, args) {
  if (!connection) {
    throw new Error("Call connect() before calling tools.");
  }

  await connection;

  return sendRequest("tools/call", { name, arguments: args });
}
```

- `name`: Selects the bot tool that handles the requested action. Pass `"roll_dice"` for the dice example so the bot dispatches to its dice handler.
- `args`: Supplies the input validated by the bot's tool handler. Pass `{ sides: 6 }` to request a six-sided die.

The result is the `callToolResult` object that your bot returns:

| Field | Description |
| --- | --- |
| `content` | Array of content items. Use `{ type: "text", text }` for a human-readable message. |
| `structuredContent` | An object with data for the widget to render. |
| `isError` | `true` when the tool failed. The text in `content` explains why. |

Handle both kinds of failure:

- The promise rejects when the call can't be completed, such as a timeout or a failed bot handler.
- The promise resolves with `isError: true` when your bot ran the tool and reported a failure.

#### Build the dice widget interface

Add the widget's own code at the end of the `<script>` element. It builds the interface, connects to Teams, reports its height, and calls `roll_dice` when the button is selected:

```javascript
const SIDES = 6;
const root = document.getElementById("widget-root");

const button = document.createElement("button");
const rolls = document.createElement("ul");
const errorMessage = document.createElement("p");

button.type = "button";
button.textContent = "Roll a " + SIDES + "-sided die";
errorMessage.setAttribute("role", "alert");
root.append(button, rolls, errorMessage);

button.addEventListener("click", async () => {
  button.disabled = true;
  errorMessage.textContent = "";

  try {
    const result = await callTool("roll_dice", { sides: SIDES });
    const text = result.content?.find((item) => item.type === "text")?.text;

    if (result.isError) {
      throw new Error(text ?? "The dice could not be rolled.");
    }

    const roll = document.createElement("li");

    // Render returned data as text, not HTML.
    roll.textContent = "You rolled " + result.structuredContent?.value + ".";
    rolls.append(roll);
  } catch (error) {
    errorMessage.textContent =
      error instanceof Error ? error.message : String(error);
  } finally {
    button.disabled = false;
  }
});

connect("Dice roller").then(
  () => reportHeight(root),
  () => {
    errorMessage.textContent = "This widget could not connect to Teams.";
  },
);
```

- `SIDES`: Sets the number of sides requested for each roll. Set this to `6` for the button label and the tool's `sides` argument.
- `role`: Makes the error message available to assistive technology immediately. Set this to `"alert"` so screen readers announce errors when they appear.
- `textContent`: Renders the bot's returned value as plain text. Assign the result string here instead of using `innerHTML` so returned data isn't parsed as HTML.

The button is disabled while a call is in progress, so a quick double click doesn't roll twice. Each new roll or error message changes the root's height, and `ResizeObserver` reports that change.

### Handle tool calls in the bot

When the widget calls a tool, your bot receives an `htmlwidget/calltool` invoke activity. In the Teams SDK, handle it with the `widget.callTool` route:

- `activity.value.name` is the tool name.
- `activity.value.arguments` is the arguments object, typed as `unknown`.

The handler returns an `IHtmlWidgetCallToolResponse` with `responseType: "htmlwidget/calltoolresult"` and a `callToolResult` object.

Create `src/bot/dice-tool.ts`:

```typescript
import { randomInt } from "node:crypto";
import type {
  IHtmlWidgetCallToolInvokeActivity,
  IHtmlWidgetCallToolResponse,
} from "@microsoft/teams.api";

function toolResult(
  text: string,
  structuredContent: Record<string, unknown> = {},
  isError = false,
): IHtmlWidgetCallToolResponse {
  return {
    responseType: "htmlwidget/calltoolresult",
    callToolResult: {
      content: [{ type: "text", text }],
      structuredContent,
      isError,
    },
  };
}

function rollDice(args: unknown): IHtmlWidgetCallToolResponse {
  const sides =
    typeof args === "object" && args !== null && "sides" in args
      ? args.sides
      : undefined;

  if (
    typeof sides !== "number" ||
    !Number.isInteger(sides) ||
    sides < 2 ||
    sides > 100
  ) {
    return toolResult("sides must be a whole number from 2 to 100.", {}, true);
  }

  const value = randomInt(1, sides + 1);

  return toolResult("You rolled " + value + ".", { value, sides });
}

export function handleWidgetTool(
  activity: IHtmlWidgetCallToolInvokeActivity,
): IHtmlWidgetCallToolResponse {
  const { name, arguments: args } = activity.value;

  switch (name) {
    case "roll_dice":
      return rollDice(args);
    default:
      return toolResult("Unknown tool: " + name, {}, true);
  }
}
```

- `sides`: Supplies the die size from the widget's untrusted arguments. Accept only integers from `2` to `100` so invalid input returns an error result.
- `structuredContent`: Carries data that the widget uses to render results. Set this to `{ value, sides }` on success and `{}` when there's no data.
- `isError`: Distinguishes a tool failure from a successful tool result. Set this to `true` for invalid input or an unknown tool, and `false` on success.

Register the route on your bot's `App`:

```typescript
import type { IHtmlWidgetCallToolResponse } from "@microsoft/teams.api";
import { handleWidgetTool } from "./dice-tool";

// `app` is your bot's App.
app.on(
  "widget.callTool",
  async ({ activity }): Promise<IHtmlWidgetCallToolResponse> => {
    try {
      return handleWidgetTool(activity);
    } catch (error) {
      console.error("Widget tool failed.", error);

      return {
        responseType: "htmlwidget/calltoolresult",
        callToolResult: {
          content: [{ type: "text", text: "Something went wrong." }],
          structuredContent: {},
          isError: true,
        },
      };
    }
  },
);
```

- `responseType`: Identifies the invoke response as a widget tool result. Set this to `"htmlwidget/calltoolresult"` so Teams returns `callToolResult` to the widget.
- `content`: Provides a human-readable explanation of the tool call outcome. Return a text item for errors while logging unexpected failure details on the bot.

Follow these rules in every tool handler:

1. Always return a result. Don't let the handler throw. Report failures with `isError: true` and a readable message in `content`. If the handler throws, the invoke fails and the widget receives only a generic error.
1. Always set all three fields: `content`, `structuredContent` (use `{}` when there's no data), and `isError`. Fields you omit can reach the widget as `null`, which widgets built with MCP Apps SDKs reject.
1. Validate `arguments`. They come from the browser, so treat them as untrusted input.
1. Ignore properties you don't recognize. Teams can add a reserved `__msteams__` property to `arguments`, so don't reject unexpected properties.
1. Don't authorize based on `activity.from`. The invoke identifies the conversation but doesn't reliably carry the user's Microsoft Entra identity. If a tool needs a verified user, the widget must send a token that the bot validates. That authentication flow is outside this walkthrough.
1. Respond quickly. The user is waiting for the result in the widget.

#### Identify which widget is calling

> [!NOTE]
> The `correlationId` behavior is preview behavior and isn't part of the published `IHtmlWidgetPayload` type.

Optionally, add a top-level `correlationId` string to the widget payload. Teams passes it back with every tool call from that widget in `arguments.__msteams__.correlationId`. Use it to find server-side state for that widget. For example, generate a random UUID, store it with your data, and look it up when a tool is called.

Create `src/bot/correlation.ts`:

```typescript
import type {
  IHtmlWidgetCallToolInvokeActivity,
  IHtmlWidgetPayload,
} from "@microsoft/teams.api";

type HtmlWidgetPayloadWithCorrelation = IHtmlWidgetPayload & {
  correlationId: string;
};

export function correlationIdOf(
  activity: IHtmlWidgetCallToolInvokeActivity,
): string | undefined {
  const args = activity.value.arguments;

  if (typeof args !== "object" || args === null || !("__msteams__" in args)) {
    return undefined;
  }

  const metadata = args.__msteams__;

  return typeof metadata === "object" &&
    metadata !== null &&
    "correlationId" in metadata &&
    typeof metadata.correlationId === "string"
    ? metadata.correlationId
    : undefined;
}
```

- `correlationId`: Associates a widget's requests with its server-side state. Set a unique string on the payload and read it from `arguments.__msteams__.correlationId` when handling calls.

A correlation ID identifies a widget. It doesn't prove who the user is. Check that the ID exists in your store and belongs to the conversation in `activity.conversation.id` before you use it.

### Widgets from MCP servers

Some MCP servers return a widget along with a tool's result. These tools are MCP App tools. Your bot shows those widgets in the same way as its own widgets: it sends each widget in an extended Markdown message and forwards the widget's tool calls to the MCP server that provided it.

This optional section assumes your bot already calls MCP tools through a `Client` from `@modelcontextprotocol/sdk`. It uses the [correlation ID](#identify-which-widget-is-calling) to route each widget's calls to the right server.

#### Declare widget support

MCP servers only return widgets to clients that support them. Declare that support when you create the MCP client:

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";

const client = new Client(
  { name: "My bot", version: "1.0.0" },
  {
    capabilities: {
      extensions: {
        "io.modelcontextprotocol/ui": {
          mimeTypes: ["text/html;profile=mcp-app"],
        },
      },
    },
  },
);
```

- `extensions`: Declares that your MCP client can display app widgets. Add `"io.modelcontextprotocol/ui"` with `mimeTypes: ["text/html;profile=mcp-app"]` so servers can return widgets.

#### Call the tool and create the widget

An MCP App tool declares its widget with a `ui://` URI in `_meta.ui.resourceUri`. The widget's HTML is a resource on the MCP server, which the bot reads with `readResource`. The server can also declare the widget's domain, security policy, and permissions.

> [!IMPORTANT]
> Always set `mcpUrl` on widgets produced by MCP tools. Teams uses it to run the widget on a dedicated domain scoped to your bot's ID and that MCP server's URL. Without it, an MCP widget can call services that accept calls from the domain meant for your bot's own widgets. See [Where widgets run](send-the-widget.md#where-widgets-run).

Create `src/bot/mcp-widgets.ts`. Its imports cover this code and [Forward the widget's tool calls](#forward-the-widgets-tool-calls):

```typescript
import { randomUUID } from "node:crypto";
import type { Client } from "@modelcontextprotocol/sdk/client/index.js";
import {
  CallToolResultSchema,
  type CallToolResult,
  type Tool,
} from "@modelcontextprotocol/sdk/types.js";
import type {
  IHtmlWidgetCallToolInvokeActivity,
  IHtmlWidgetCallToolResponse,
  IHtmlWidgetPayload,
  IHtmlWidgetPermissions,
  IHtmlWidgetSecurityPolicy,
} from "@microsoft/teams.api";
import { correlationIdOf } from "./correlation";

const MCP_APP_MIME_TYPE = "text/html;profile=mcp-app";

// These fields aren't part of the published IHtmlWidgetPayload type yet.
export type McpWidgetPayload = IHtmlWidgetPayload & {
  correlationId: string;
  mcpUrl: string;
};

interface McpWidgetSettings {
  domain?: string;
  csp?: IHtmlWidgetSecurityPolicy;
  permissions?: IHtmlWidgetPermissions;
}

interface McpWidgetRoute {
  client: Client;
  conversationId: string;
}

// Use persistent storage in production so older widgets survive bot restarts.
const routes = new Map<string, McpWidgetRoute>();

function uiMetadata(
  meta: Record<string, unknown> | undefined,
): object | undefined {
  const ui = meta?.ui;

  return typeof ui === "object" && ui !== null ? ui : undefined;
}

function widgetUri(tool: Tool): string | undefined {
  const ui = uiMetadata(tool._meta);

  return ui && "resourceUri" in ui && typeof ui.resourceUri === "string"
    ? ui.resourceUri
    : undefined;
}

export async function callMcpTool(
  client: Client,
  serverUrl: string,
  conversationId: string,
  tool: Tool,
  args: Record<string, unknown>,
): Promise<{ result: CallToolResult; widget?: McpWidgetPayload }> {
  const result = CallToolResultSchema.parse(
    await client.callTool({ name: tool.name, arguments: args }),
  );
  const uri = widgetUri(tool);

  if (!uri || result.isError) {
    return { result };
  }

  const resource = await client.readResource({ uri });
  const content = resource.contents.find((item) => item.uri === uri);

  if (!content || content.mimeType !== MCP_APP_MIME_TYPE) {
    throw new Error("The MCP server did not return the tool's widget.");
  }

  const html =
    "text" in content
      ? content.text
      : Buffer.from(content.blob, "base64").toString("utf8");
  const settings: McpWidgetSettings = uiMetadata(content._meta) ?? {};
  const correlationId = randomUUID();

  routes.set(correlationId, { client, conversationId });

  return {
    result,
    widget: {
      type: "widget/mcp-ui",
      name: tool.title ?? tool.name,
      description: tool.description,
      domain: settings.domain ?? new URL(serverUrl).origin,
      html,
      securityPolicy: settings.csp,
      permissions: settings.permissions,
      toolInput: args,
      toolOutput: result,
      correlationId,
      mcpUrl: serverUrl,
    },
  };
}
```

- `serverUrl`: Identifies the MCP server that produced this widget's HTML. Pass the server URL used by the client so the payload's `mcpUrl` isolates the widget.
- `conversationId`: Associates the widget with the conversation receiving its message. Set this to `activity.conversation.id` so later calls can be checked against the same conversation.
- `toolInput` and `toolOutput`: Supply the tool arguments and result to the widget. Set these to `args` and `result` so Teams can pass the data to its interface.
- `correlationId`: Routes future tool calls back to the originating MCP server. Set this to a generated UUID and store it with the client and conversation.

The example's routes are held in memory. In production, persist the routing information and restore the corresponding MCP connections so widgets in older messages can still call tools after the bot restarts.

#### Send the MCP widget

Call the MCP tool with `callMcpTool` instead of `client.callTool`. If the tool returned a widget, send it:

```typescript
import { callMcpTool } from "./mcp-widgets";
import { widgetMarkdown } from "./widget-markdown";

// `client` is your MCP client, `tool` comes from client.listTools(), and
// `args` contains the tool's arguments.
const { result, widget } = await callMcpTool(
  client,
  "https://mcp.example.com/mcp",
  activity.conversation.id,
  tool,
  args,
);

if (widget) {
  await send({
    type: "message",
    textFormat: "extendedmarkdown",
    text: widgetMarkdown(widget),
  });
}

// Use `result` as you would for any MCP tool, for example in model context.
```

- `text`: Contains the widget payload returned by the MCP integration. Set this to `widgetMarkdown(widget)` and `textFormat` to `"extendedmarkdown"` so Teams displays the widget.

An MCP widget's HTML already performs the handshake and reports its height. Send it unchanged rather than adding the dice widget's script. To add surrounding Markdown, use a `text` array as shown in [Mix Markdown and widgets](send-the-widget.md#mix-markdown-and-widgets).

#### Forward the widget's tool calls

When an MCP widget calls a tool, the bot forwards the call to the MCP server that provided the widget and returns the server's result.

Add this code to `src/bot/mcp-widgets.ts`:

```typescript
function isAvailableToWidgets(tool: Tool): boolean {
  const ui = uiMetadata(tool._meta);
  const visibility =
    ui && "visibility" in ui && Array.isArray(ui.visibility)
      ? ui.visibility
      : ["model", "app"];

  return visibility.includes("app");
}

function errorResult(text: string): IHtmlWidgetCallToolResponse {
  return {
    responseType: "htmlwidget/calltoolresult",
    callToolResult: {
      content: [{ type: "text", text }],
      structuredContent: {},
      isError: true,
    },
  };
}

export async function handleMcpWidgetTool(
  activity: IHtmlWidgetCallToolInvokeActivity,
): Promise<IHtmlWidgetCallToolResponse | undefined> {
  const correlationId = correlationIdOf(activity);

  if (!correlationId) {
    return undefined;
  }

  const route = routes.get(correlationId);

  if (!route || route.conversationId !== activity.conversation.id) {
    return errorResult("This widget is no longer available.");
  }

  // Follow nextCursor when the server has more than one page of tools.
  const { tools } = await route.client.listTools();
  const tool = tools.find((entry) => entry.name === activity.value.name);

  if (!tool || !isAvailableToWidgets(tool)) {
    return errorResult("Unknown tool: " + activity.value.name);
  }

  const { __msteams__, ...args } = activity.value.arguments as Record<
    string,
    unknown
  >;
  const result = CallToolResultSchema.parse(
    await route.client.callTool({ name: tool.name, arguments: args }),
  );

  return {
    responseType: "htmlwidget/calltoolresult",
    callToolResult: {
      content: result.content.filter((item) => item.type !== "resource_link"),
      structuredContent: result.structuredContent ?? {},
      isError: result.isError ?? false,
    },
  };
}
```

- `visibility`: Controls whether a tool is available to widget callers. Require `"app"` in `_meta.ui.visibility`; tools with only `["model"]` are reserved for the model.
- `__msteams__`: Contains Teams metadata intended for the bot's routing logic. Remove this property before forwarding the remaining arguments to the MCP server.
- `callToolResult`: Returns the MCP result using the Teams invoke response shape. Set `structuredContent` and `isError` even when omitted by the server, and remove unsupported `resource_link` items.

Update the `widget.callTool` route from [Handle tool calls in the bot](#handle-tool-calls-in-the-bot) so it sends MCP widget calls to `handleMcpWidgetTool`:

```typescript
import type { IHtmlWidgetCallToolResponse } from "@microsoft/teams.api";
import { handleWidgetTool } from "./dice-tool";
import { handleMcpWidgetTool } from "./mcp-widgets";

app.on(
  "widget.callTool",
  async ({ activity }): Promise<IHtmlWidgetCallToolResponse> => {
    try {
      return (
        (await handleMcpWidgetTool(activity)) ?? handleWidgetTool(activity)
      );
    } catch (error) {
      console.error("Widget tool failed.", error);

      return {
        responseType: "htmlwidget/calltoolresult",
        callToolResult: {
          content: [{ type: "text", text: "Something went wrong." }],
          structuredContent: {},
          isError: true,
        },
      };
    }
  },
);
```

- `handleMcpWidgetTool`: Routes calls from known MCP widgets to their server. Call it first and use `handleWidgetTool` only when it returns `undefined`.

Use the same [tool-handler rules](#handle-tool-calls-in-the-bot), plus these checks:

- Only forward calls from widgets you sent, with a known correlation ID belonging to the same conversation.
- Only forward calls to tools that widgets may call.
- Remove `__msteams__` before forwarding arguments.
- Return the server's result, normalizing omitted fields and removing unsupported content items.

### Put it all together

Your existing bot needs two additions:

1. A `send` call for the extended Markdown message containing the widget, as shown in [Send the widget](send-the-widget.md#send-the-message).
1. The [`widget.callTool` route](#handle-tool-calls-in-the-bot).

The `<script>` element in `widgets/dice-widget.html` contains the code from these sections, in this order:

1. [Connect the widget to Teams](#connect-the-widget-to-teams).
1. [Report the widget's height](#report-the-widgets-height).
1. [Call a tool from the widget](#call-a-tool-from-the-widget), including the dice widget interface.

To show widgets from MCP servers, also declare widget support on your MCP client, call MCP tools with `callMcpTool`, and send MCP widget tool calls to `handleMcpWidgetTool`.

To try the dice widget:

1. Run your bot and make it send the widget in a Teams chat.
1. Select **Roll a 6-sided die** a few times.
1. Confirm that each roll appears in the list and the widget grows to fit it without a scroll bar.

## Handle errors

<!-- TODO: Confirm service error codes and developer actions with engineering. The source describes failures but doesn't provide status or error codes. -->

| Failure | Developer action |
| --- | --- |
| Handshake or tool request times out | Catch the rejected promise and show a readable message. The example waits 15 seconds for each request. |
| Tool returns `isError: true` | Display the text from `content`; don't treat a resolved promise as proof of success. |
| Tool arguments are invalid | Return a result with `isError: true`, a readable text item, and `structuredContent: {}`. |
| An unexpected bot or MCP error occurs | Log the details on the bot and return the generic error result from the route's `catch` block. |
| An MCP widget's route is missing or belongs to another conversation | Return **This widget is no longer available.** Don't forward the call. |

## Code sample

| Sample name | Description | TypeScript |
| --- | --- | --- |
| HTML widgets example | Demonstrates widget tool-call handlers and the Teams SDK invoke response contract. | [View](https://github.com/microsoft/teams.ts/tree/main/examples/html-widgets) |

## Design guidelines and best practices

- Complete the handshake before sending size notifications or calling tools.
- Let content determine height, rather than sizing content to the iframe.
- Disable actions while requests are in progress and show errors in the widget.
- Render returned values with `textContent` rather than `innerHTML`.
- Validate browser input on the bot and check correlation IDs against the conversation.
- Treat correlation IDs and payload domains as metadata, not proof of user identity.
- Preserve MCP widget HTML without adding a second protocol implementation.

## Limitations and known issues

- HTML widgets and the related Teams SDK APIs are in preview.
- The `correlationId` and `mcpUrl` fields aren't part of the published `IHtmlWidgetPayload` type. The examples extend that type for preview use.
- This walkthrough doesn't implement user authentication for tools that require a verified Microsoft Entra identity.
- The MCP example uses an in-memory routing map. Persist routing information for production.
- The MCP forwarding snippet lists one page of tools. Follow `nextCursor` when the server returns more pages.
- Teams doesn't support `resource_link` items in tool-call results.
- This walkthrough provides TypeScript bot code and browser JavaScript, not .NET or Python implementations.

<!-- TODO: Confirm preview eligibility, enablement, supported clients, rollout limits, and the MCP integration with engineering before publication. -->

## Checklist

### Handshake and size

- [ ] The handshake (`ui/initialize`, then `ui/notifications/initialized`) completes before anything else.
- [ ] The widget measures one root element with `getBoundingClientRect().height`, never `scrollHeight`.
- [ ] The widget isn't sized to fit the iframe with `100vh` or `height: 100%` on `html`, `body`, or the root.
- [ ] Heights are reported exactly, without rounding, from a `ResizeObserver`.
- [ ] `html` and `body` have no margin, the root uses `display: flow-root`, and all content stays inside the root in the normal page flow.

### Tool calls

- [ ] The widget calls tools in response to user actions.
- [ ] The widget handles both a rejected `callTool` promise and a result with `isError: true`.
- [ ] The bot's handler always returns `content`, `structuredContent`, and `isError`, and never throws.
- [ ] The bot validates `arguments`, ignores unknown properties such as `__msteams__`, and doesn't authorize based on `activity.from`.

### MCP widget integration

- [ ] The MCP client declares the `io.modelcontextprotocol/ui` extension.
- [ ] Each MCP widget uses `mcpUrl` as described in the [sending checklist](send-the-widget.md#checklist).
- [ ] Each MCP widget has a unique `correlationId`, stored with its MCP server and conversation.
- [ ] Tool calls are forwarded only for known widgets in the same conversation and only for tools that widgets may call.
- [ ] `__msteams__` is removed from the arguments before they're forwarded.

## Reference: messages

| Code | Message | Kind | Payload | Reply |
| --- | --- | --- | --- | --- |
| Widget | `ui/initialize` | Request | `{ protocolVersion, appInfo: { name, version }, appCapabilities }` | Object that can include `hostContext`, such as the current theme. |
| Widget | `ui/notifications/initialized` | Notification | `{}` | None. |
| Widget | `ui/notifications/size-changed` | Notification | `{ height }` in CSS pixels. | None. |
| Widget | `tools/call` | Request | `{ name, arguments }` | The bot's `callToolResult`. |
| Bot | `widget.callTool` route (`htmlwidget/calltool`) | Handler | `activity.value`: `{ name, arguments }` | `{ responseType: "htmlwidget/calltoolresult", callToolResult }`. |

Requests wait for a reply. Notifications don't. For the message that contains the widget payload, see [Reference: messages](send-the-widget.md#reference-messages) in the sending article.

## See also

- [Send the widget](send-the-widget.md)
- [Format your agent messages](how-to/format-your-bot-messages.md)
- [Teams SDK](https://github.com/microsoft/teams.ts)
