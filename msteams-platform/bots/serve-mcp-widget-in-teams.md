---
title: Serve an MCP-generated HTML widget in Teams
description: Learn how to read an HTML widget from an MCP server, send it in a Teams agent message, and forward widget tool calls to its server.
ms.topic: how-to
ms.localizationpriority: medium
ms.date: 11/04/2026
---

# Serve an MCP-generated HTML widget in Teams

An MCP-generated HTML widget is an interactive interface that an MCP server provides alongside a tool result. Your Teams agent reads the widget's HTML, sends it in an extended Markdown message, and forwards user-initiated tool calls to the originating MCP server. Users can interact with the tool's results directly in the conversation.

Use this walkthrough when an MCP server supplies the widget rather than your agent owning the HTML. To build your own widget, including its handshake, sizing, and bot-side tool handler, see [Send an HTML widget from an agent](send-html-widget-from-agent.md).

> [!NOTE]
> HTML widgets are a preview feature. The Teams SDK marks the related APIs as experimental, and they may change.

## Prerequisites

- A working TypeScript Teams SDK bot available on a public HTTPS endpoint.
- An MCP client connected to a server that provides MCP App tools and widget resources, using `Client` from `@modelcontextprotocol/sdk`.
- The [widget Markdown helper](send-html-widget-from-agent.md#put-the-widget-in-a-fenced-code-block) and [tool-call route](send-html-widget-from-agent.md#handle-tool-calls-in-the-bot) from the agent-owned widget walkthrough. The combined route below reuses `handleWidgetTool` for calls that aren't from MCP widgets.

## User experience

The agent displays the interface supplied by the MCP tool in a chat message. When a user performs an action in that interface, the widget calls a tool, the agent forwards the request to the originating MCP server, and the widget receives the result.

The MCP widget's HTML already performs the Teams handshake and reports its height. You don't add the dice-roller interface or protocol script to it.

## Project layout

Add these files to your existing bot project. Reuse `widget-markdown.ts` and `dice-tool.ts` from [Send an HTML widget from an agent](send-html-widget-from-agent.md#project-layout).

```text
src/
  bot/
    correlation.ts     # Identifies which widget made a tool call
    mcp-widgets.ts     # Reads MCP widgets and forwards their tool calls
```

Follow this master procedure to serve an MCP-generated widget:

1. [Identify which widget is calling](#identify-which-widget-is-calling).
1. [Declare widget support](#declare-widget-support).
1. [Call the tool and create the widget](#call-the-tool-and-create-the-widget).
1. [Send the MCP widget](#send-the-mcp-widget).
1. [Forward the widget's tool calls](#forward-the-widgets-tool-calls).
1. [Put it all together](#put-it-all-together).

## Implement MCP-generated widgets

### Identify which widget is calling

> [!NOTE]
> The `correlationId` behavior is preview behavior and isn't part of the published `IHtmlWidgetPayload` type.

To route MCP widget calls, add a top-level `correlationId` string to the widget payload. Teams passes it back with every tool call from that widget in `arguments.__msteams__.correlationId`. Use it to find server-side state for that widget. For example, generate a random UUID, store it with your data, and look it up when a tool is called.

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

### Declare widget support

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

### Call the tool and create the widget

An MCP App tool declares its widget with a `ui://` URI in `_meta.ui.resourceUri`. The widget's HTML is a resource on the MCP server, which the bot reads with `readResource`. The server can also declare the widget's domain, security policy, and permissions.

> [!IMPORTANT]
> Always set `mcpUrl` on widgets produced by MCP tools. Teams uses it to run the widget on a dedicated domain scoped to your bot's ID and that MCP server's URL. Teams derives the widget domain from `<bot id>-<mcp URL>`, isolating it from other MCP servers and your agent's own widgets. Without `mcpUrl`, the widget runs on the bot-only domain and can call services that accept calls from that domain. Enter `<bot id>-<mcp URL>` in the [widget domain tool](https://widgettool.z19.web.core.windows.net/) to find the domain. For CORS configuration, see [Where widgets run](send-html-widget-from-agent.md#where-widgets-run).

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

### Send the MCP widget

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

An MCP widget's HTML already performs the handshake and reports its height. Send it unchanged rather than adding the dice widget's script. To add surrounding Markdown, use a `text` array as shown in [Mix Markdown and widgets](send-html-widget-from-agent.md#mix-markdown-and-widgets).

### Forward the widget's tool calls

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

Update the `widget.callTool` route from [Handle tool calls in the bot](send-html-widget-from-agent.md#handle-tool-calls-in-the-bot) so it sends MCP widget calls to `handleMcpWidgetTool`:

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

Use the same [tool-handler rules](send-html-widget-from-agent.md#handle-tool-calls-in-the-bot), plus these checks:

- Only forward calls from widgets you sent, with a known correlation ID belonging to the same conversation.
- Only forward calls to tools that widgets may call.
- Remove `__msteams__` before forwarding arguments.
- Return the server's result, normalizing omitted fields and removing unsupported content items.

### Put it all together

1. Create your MCP client with the `io.modelcontextprotocol/ui` extension.
1. Add `correlationIdOf`, `callMcpTool`, and `handleMcpWidgetTool` to the bot.
1. Call a widget-producing tool through `callMcpTool`, passing the MCP server URL and the current conversation ID.
1. Send the returned widget unchanged with `widgetMarkdown` in an extended Markdown message.
1. Register the combined `widget.callTool` route so MCP widget calls reach their originating server.
1. In Teams, invoke the tool and interact with its widget. Confirm that the tool call reaches the MCP server and its result returns to the widget.

## Handle errors

<!-- TODO: Confirm service error codes and developer actions with engineering. The source describes failures but doesn't provide status or error codes. -->

| Failure | Developer action |
| --- | --- |
| The initial MCP tool returns `isError: true` | Use the tool result to report the failure; the example doesn't create a widget for a failed tool call. |
| The MCP resource is missing or has the wrong MIME type | Handle the error thrown by `callMcpTool` and verify the tool's resource URI and returned MIME type. |
| A widget's route is missing or belongs to another conversation | Return **This widget is no longer available.** Don't forward the call. |
| A tool doesn't exist or isn't available to widgets | Return an error result rather than forwarding the call. |
| An unexpected MCP error occurs during forwarding | Log the details on the bot and return the generic error result from the route's `catch` block. |

For rejected widget requests and `isError` handling in the interface, see [Handle errors](send-html-widget-from-agent.md#handle-errors).

## Code sample

| Sample name | Description | TypeScript |
| --- | --- | --- |
| HTML widgets example | Demonstrates widget tool-call handlers and the Teams SDK invoke response contract. | [View](https://github.com/microsoft/teams.ts/tree/main/examples/html-widgets) |

## Design guidelines and best practices

- Send MCP widget HTML unchanged. Don't inject a second handshake or height-reporting implementation.
- Always set `mcpUrl` to isolate the widget from other MCP servers and the agent's own widgets.
- Treat correlation IDs and payload domains as metadata, not proof of user identity.
- Check that the widget's correlation ID belongs to the current conversation before forwarding calls.
- Forward only tools available to widgets, and remove Teams metadata from forwarded arguments.
- Follow the shared [tool-handler rules](send-html-widget-from-agent.md#handle-tool-calls-in-the-bot).

<!-- TODO: Add direct links to the HTML widget UX and accessibility guidance when their publication paths are confirmed. -->

## Limitations and known issues

- HTML widgets and the related Teams SDK APIs are in preview.
- The `correlationId` and `mcpUrl` fields aren't part of the published `IHtmlWidgetPayload` type. The examples extend that type for preview use.
- This walkthrough doesn't implement user authentication for tools that require a verified Microsoft Entra identity.
- The example uses an in-memory routing map. Persist routing information for production.
- The forwarding snippet lists one page of tools. Follow `nextCursor` when the server returns more pages.
- Teams doesn't support `resource_link` items in tool-call results.
- This walkthrough provides TypeScript bot code, not .NET or Python implementations.

<!-- TODO: Confirm preview eligibility, enablement, supported clients, rollout limits, and the MCP integration with engineering before publication. -->

## Checklist

- [ ] The MCP client declares the `io.modelcontextprotocol/ui` extension.
- [ ] Each MCP widget includes `mcpUrl`, the URL of the server whose tool produced it.
- [ ] Each MCP widget has a unique `correlationId`, stored with its MCP server and conversation.
- [ ] The bot sends the server-provided HTML unchanged.
- [ ] Tool calls are forwarded only for known widgets in the same conversation and only for tools that widgets may call.
- [ ] `__msteams__` is removed from the arguments before they're forwarded.
- [ ] The result always includes `content`, `structuredContent`, and `isError`, with unsupported `resource_link` items removed.
- [ ] The shared [sending requirements](send-html-widget-from-agent.md#sending-the-widget) are met.

## Reference: messages

The widget uses the same [handshake, size notifications, and tool-call protocol](send-html-widget-from-agent.md#reference-messages) as an agent-owned widget. The MCP integration adds this server-side flow:

| Operation | Bot API | Purpose |
| --- | --- | --- |
| Initial tool call | `client.callTool({ name, arguments })` | Get a tool result and inspect the tool's `_meta.ui.resourceUri`. |
| Read widget resource | `client.readResource({ uri })` | Retrieve the widget HTML and settings. |
| Send widget | `send({ type: "message", textFormat: "extendedmarkdown", text })` | Display the payload produced by `callMcpTool`. |
| Forward widget tool call | `widget.callTool` route to `handleMcpWidgetTool`, then `client.callTool` | Route the request to the originating server and return an `htmlwidget/calltoolresult` response. |

## See also

- [Send an HTML widget from an agent](send-html-widget-from-agent.md)
- [Format your agent messages](how-to/format-your-bot-messages.md)
- [Teams SDK](https://github.com/microsoft/teams.ts)
