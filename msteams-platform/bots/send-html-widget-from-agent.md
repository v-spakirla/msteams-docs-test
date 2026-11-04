---
title: Send an HTML widget from an agent
description: Learn how to send an interactive HTML widget from a Teams SDK agent, connect it to Teams, resize it, and handle tool calls.
ms.topic: how-to
ms.localizationpriority: medium
ms.date: 11/04/2026
---

# Send an HTML widget from an agent

An HTML widget is an HTML document that Teams renders in a sandboxed iframe inside a chat message. Use widgets to bring a custom interactive interface into a conversation, so users can work with your bot without leaving the chat. Your bot sends the HTML in an extended Markdown message, and the widget connects to Teams to resize its interface and call the bot's tools.

This article walks through a complete dice-roller widget using a TypeScript bot built with the [Teams SDK](https://github.com/microsoft/teams.ts), from sending the HTML to handling tool calls. If an MCP server provides the widget instead, see [Serve an MCP-generated HTML widget in Teams](serve-mcp-widget-in-teams.md).

> [!NOTE]
> HTML widgets are a preview feature. The Teams SDK marks the related APIs as experimental, and they may change.

## Prerequisites

- A working TypeScript bot built with the Teams SDK.
- A public HTTPS endpoint for your bot.

## User experience

The example is a dice roller. When you complete this walkthrough, your bot sends a widget with a **Roll a 6-sided die** button. Selecting the button asks the bot to roll the die, adds the result to a list, and grows the widget to fit the results.

The button is disabled while a roll is in progress. Errors appear in the widget and are announced to screen readers.

## What you write

You write two pieces of code:

| Piece | What it is | Responsibilities |
| --- | --- | --- |
| Widget | One HTML file with inline CSS and plain JavaScript. | Contains the interface, performs the handshake, reports its height, and calls tools. |
| Bot | TypeScript code in your existing Teams SDK bot. | Sends the widget in an extended Markdown message and answers tool calls. |

## Project layout

Add these files to your existing bot project:

```text
widgets/
  dice-widget.html     # HTML, CSS, and JavaScript in one file
src/
  bot/
    widget-markdown.ts # Wraps a widget payload in a fenced code block
    dice-widget.ts     # Builds the payload from dice-widget.html
    dice-tool.ts       # Handles tool calls from the widget
```

Follow this master procedure to send the widget:

1. [Create the widget file](#create-the-widget-file).
1. [Send the widget](#send-the-widget).
1. [Connect the widget to Teams](#connect-the-widget-to-teams).
1. [Report the widget's height](#report-the-widgets-height).
1. [Call a tool from the widget](#call-a-tool-from-the-widget).
1. [Handle tool calls in the bot](#handle-tool-calls-in-the-bot).
1. [Put it all together](#put-it-all-together).

## Implement sending a widget

### Create the widget file

Create `widgets/dice-widget.html`. This is the complete page that Teams displays. Leave the `<script>` element empty for now. Later in this walkthrough, you add the connection, height-reporting, and tool-calling code.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Dice roller</title>
    <style>
      /* Keep the root's height equal to the widget's full height. */
      html,
      body {
        margin: 0;
      }

      /* Include the first and last children's margins in measurements. */
      #widget-root {
        display: flow-root;
        font-family: system-ui, sans-serif;
      }
    </style>
  </head>
  <body>
    <!-- Keep all visible content inside the element measured for height. -->
    <div id="widget-root"></div>

    <script>
      // Add the connection, height-reporting, and tool-calling code here.
    </script>
  </body>
</html>
```

- `margin`: Removes space outside the measured root element. Set this to `0` on `html` and `body` so height reporting includes all visible content.
- `display`: Keeps child margins inside the measured root element. Set this to `flow-root` on `widget-root` so the widget can report its full height.

Everything is in one file: the CSS and JavaScript are inline, and the widget loads nothing from other origins. The example uses plain JavaScript without a framework.

### Send the widget

Widgets are sent in **extended Markdown messages**: message activities whose `textFormat` is `"extendedmarkdown"`. The widget goes in the message's `text`, as a fenced code block with the language `html-widget`. The content of the code block is the widget's payload, as JSON:

````text
```html-widget
{"type":"widget/mcp-ui","name":"Dice roller","domain":"https://mybot.example.com","html":"<!doctype html>...", ...}
```
````

Teams displays the code block as the widget. The message's `text` can also contain ordinary Markdown around the code block. For more information, see [Mix Markdown and widgets](#mix-markdown-and-widgets) and [Format your agent messages](how-to/format-your-bot-messages.md).

The payload has these fields:

| Field | Required | Description |
| --- | --- | --- |
| `type` | Yes | Always `"widget/mcp-ui"`. |
| `name` | Yes | Display name of the widget. |
| `description` | No | Short description of the widget. |
| `domain` | Yes | An HTTPS origin associated with the widget. Use your bot's origin. This is metadata, not a verified identity or the domain that hosts the widget. |
| `html` | Yes | The complete HTML document that Teams renders in the iframe. |
| `securityPolicy` | No | Origins the HTML may use. `resourceDomains` covers external scripts, styles, images, and fonts. `connectDomains` covers `fetch`, XHR, and WebSocket calls. Anything not listed is blocked. |

#### Build the widget's payload

Create `src/bot/dice-widget.ts`:

```typescript
import { readFileSync } from "node:fs";
import { resolve } from "node:path";
import type { IHtmlWidgetPayload } from "@microsoft/teams.api";

// Adjust the path to where the HTML file is deployed with your bot.
const DICE_WIDGET_HTML = readFileSync(
  resolve("widgets/dice-widget.html"),
  "utf8",
);

export function createDiceWidget(botOrigin: string): IHtmlWidgetPayload {
  return {
    type: "widget/mcp-ui",
    name: "Dice roller",
    description: "Rolls dice on the bot.",
    domain: botOrigin,
    html: DICE_WIDGET_HTML,
    securityPolicy: {
      connectDomains: [],
      resourceDomains: [],
      frameDomains: [],
      baseUriDomains: [],
    },
  };
}
```

- `botOrigin`: Supplies the HTTPS origin associated with your widget. Pass your bot's public origin, such as `https://mybot.example.com`, to set the payload's `domain`.
- `type`: Identifies the payload as a Teams HTML widget. Set this to `"widget/mcp-ui"` so Teams recognizes the widget.
- `html`: Supplies the complete document rendered inside the iframe. Set this to the contents of `dice-widget.html`, read once when the bot starts.
- `securityPolicy`: Limits external resources and network calls from the widget. Use empty arrays for this self-contained example so no external origins are allowed.

#### Put the widget in a fenced code block

Create `src/bot/widget-markdown.ts`. This helper works for any widget:

````typescript
import type { IHtmlWidgetPayload } from "@microsoft/teams.api";

export function widgetMarkdown(widget: IHtmlWidgetPayload): string {
  return "```html-widget\n" + JSON.stringify(widget) + "\n```";
}
````

- `widget`: Supplies the payload serialized into the message text. Pass the object returned by `createDiceWidget` to produce the `html-widget` fenced code block.

#### Send the message

Send an extended Markdown message whose `text` is the widget's fenced code block:

```typescript
import { createDiceWidget } from "./dice-widget";
import { widgetMarkdown } from "./widget-markdown";

// `send` comes from the context of the activity you are handling.
await send({
  type: "message",
  textFormat: "extendedmarkdown",
  text: widgetMarkdown(createDiceWidget("https://mybot.example.com")),
});
```

- `textFormat`: Enables widget rendering within the message's extended Markdown. Set this to `"extendedmarkdown"` so Teams renders the `html-widget` block as a widget.
- `text`: Contains the serialized widget and any surrounding Markdown. Set this to `widgetMarkdown(createDiceWidget(...))`, using your bot's public HTTPS origin.

> [!IMPORTANT]
> This example implements its own [handshake](#connect-the-widget-to-teams) and [height reporting](#report-the-widgets-height). Don't use `buildHtmlWidgetMessage` or `buildHtmlWidgetMarkdown` from `@microsoft/teams.apps` for this example. Their injected protocol code can conflict with the widget's own implementation.

#### Mix Markdown and widgets

An extended Markdown message can contain ordinary Markdown around a widget. Build the message's `text` from an array of Markdown strings and widgets, joined with blank lines:

```typescript
import { createDiceWidget } from "./dice-widget";
import { widgetMarkdown } from "./widget-markdown";

await send({
  type: "message",
  textFormat: "extendedmarkdown",
  text: [
    "**Feeling lucky?** Roll the die below.",
    widgetMarkdown(createDiceWidget("https://mybot.example.com")),
    "_The bot picks every number._",
  ].join("\n\n"),
});
```

- `text`: Combines explanatory text and widget payloads into one message. Join the entries with `"\n\n"` so blank lines separate Markdown and widgets.

The message's `text` is:

````text
**Feeling lucky?** Roll the die below.

```html-widget
{"type":"widget/mcp-ui","name":"Dice roller", ...}
```

_The bot picks every number._
````

To show several widgets in one message, add a `widgetMarkdown(...)` entry to the array for each of them.

Each message contains the complete widget. Messages that were already sent keep the HTML they were sent with. Changes to `dice-widget.html` apply only to messages you send afterward.

#### Validate external origins

If your widget loads anything from another origin, list that origin in the security policy. Use `validateSecurityPolicy` from `@microsoft/teams.apps` in your unit tests to find origins that the HTML uses but the policy doesn't list:

```typescript
import { validateSecurityPolicy } from "@microsoft/teams.apps";
import { createDiceWidget } from "./dice-widget";

const widget = createDiceWidget("https://mybot.example.com");
const warnings = validateSecurityPolicy(
  widget.html,
  widget.securityPolicy ?? {},
);
// Expect: warnings is an empty array.
```

- `widget.securityPolicy`: Declares which external origins the widget may use. Pass it with `widget.html` so the validator can compare references against the allowed origins.

The validator uses static analysis. It doesn't detect every dynamically constructed URL.

#### Where widgets run

Teams hosts every widget on a dedicated domain, in this format:

```text
https://<hash>.widget-renderer.usercontent.microsoft
```

For the agent's own widgets, the hash is derived from the bot's ID. To find the domain, enter your bot's ID in the [widget domain tool](https://widgettool.z19.web.core.windows.net/).

To let a widget call one of your services, such as an API listed in `connectDomains`, configure that service to accept calls from the widget's domain, for example in its CORS settings.

MCP-generated widgets require a domain scoped to both the bot's ID and the MCP server URL. For that scenario and its isolation requirements, see [Serve an MCP-generated HTML widget in Teams](serve-mcp-widget-in-teams.md#call-the-tool-and-create-the-widget).

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
- Remove margins around the root. The [HTML shell](#create-the-widget-file) sets `html, body { margin: 0; }` and gives the root `display: flow-root` so that its first and last children's margins are measured.

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

For optional per-widget state and MCP routing, see [Identify which widget is calling](serve-mcp-widget-in-teams.md#identify-which-widget-is-calling).

### Put it all together

Your existing bot needs two additions:

1. A `send` call for the extended Markdown message containing the widget, as shown in [Send the message](#send-the-message).
1. The [`widget.callTool` route](#handle-tool-calls-in-the-bot).

The `<script>` element in `widgets/dice-widget.html` contains the code from these sections, in this order:

1. [Connect the widget to Teams](#connect-the-widget-to-teams).
1. [Report the widget's height](#report-the-widgets-height).
1. [Call a tool from the widget](#call-a-tool-from-the-widget), including the dice widget interface.

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
| An unexpected bot error occurs | Log the details on the bot and return the generic error result from the route's `catch` block. |

The security-policy validator returns warnings for external origins that aren't allowed by the declared policy. Add each required origin to the appropriate policy field before you send the widget.

## Code sample

| Sample name | Description | TypeScript |
| --- | --- | --- |
| HTML widgets example | Demonstrates widget messages, payload validation, and tool calls with the Teams SDK. | [View](https://github.com/microsoft/teams.ts/tree/main/examples/html-widgets) |

## Design guidelines and best practices

- Keep the example's HTML, CSS, and JavaScript in one file.
- Include only the external origins your widget needs in its security policy.
- Use surrounding Markdown to explain the action users can take.
- Complete the handshake before sending size notifications or calling tools.
- Let content determine height, rather than sizing content to the iframe.
- Disable actions while requests are in progress and show errors in the widget.
- Render returned values with `textContent` rather than `innerHTML`.
- Validate browser input on the bot. Don't use `activity.from` as proof of user identity.

> [!WARNING]
> Anyone who can see the message can read the widget's HTML. Never put secrets, tokens, or private data in it.

<!-- TODO: Add direct links to the HTML widget UX and accessibility guidance when their publication paths are confirmed. -->

## Limitations and known issues

> [!NOTE]
> HTML widget content can't currently be exported through export APIs. Consider this limitation when developing bots for organizations with content export requirements. Organizations can disable bots that can send HTML widgets if this limitation doesn't meet their requirements.

- HTML widgets and the related Teams SDK APIs are in preview.
- Sent messages retain their original HTML. Updating the file doesn't update widgets in existing messages.
- This walkthrough doesn't implement user authentication for tools that require a verified Microsoft Entra identity.
- This walkthrough provides TypeScript bot code and browser JavaScript. It doesn't provide .NET or Python implementations.

<!-- TODO: Confirm preview eligibility, enablement, supported clients, and rollout limits with engineering before publication. -->

## Checklist

### Sending the widget

- [ ] The widget is one HTML file with inline CSS and JavaScript.
- [ ] The widget is sent in an extended Markdown message (`textFormat: "extendedmarkdown"`), as an `html-widget` fenced code block containing the payload.
- [ ] The message isn't built with `buildHtmlWidgetMessage` or `buildHtmlWidgetMarkdown` when using this walkthrough's own protocol implementation.
- [ ] Any origin the HTML loads from is in `resourceDomains`; any origin it calls is in `connectDomains`.
- [ ] The HTML contains no secrets.
- [ ] Services that the widget calls accept calls from `https://<hash>.widget-renderer.usercontent.microsoft`.

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

## Reference: messages

| Code | Message | Kind | Payload | Reply |
| --- | --- | --- | --- | --- |
| Bot | Message activity with `textFormat: "extendedmarkdown"` | Message | `text` contains an `html-widget` fenced code block with the JSON widget payload. | Not a widget protocol request. |
| Widget | `ui/initialize` | Request | `{ protocolVersion, appInfo: { name, version }, appCapabilities }` | Object that can include `hostContext`, such as the current theme. |
| Widget | `ui/notifications/initialized` | Notification | `{}` | None. |
| Widget | `ui/notifications/size-changed` | Notification | `{ height }` in CSS pixels. | None. |
| Widget | `tools/call` | Request | `{ name, arguments }` | The bot's `callToolResult`. |
| Bot | `widget.callTool` route (`htmlwidget/calltool`) | Handler | `activity.value`: `{ name, arguments }` | `{ responseType: "htmlwidget/calltoolresult", callToolResult }`. |

Requests wait for a reply. Notifications don't.

## Next step

> [!div class="nextstepaction"]
> [Serve an MCP-generated HTML widget in Teams](serve-mcp-widget-in-teams.md)

## See also

- [Format your agent messages](how-to/format-your-bot-messages.md)
- [Serve an MCP-generated HTML widget in Teams](serve-mcp-widget-in-teams.md)
- [Teams SDK](https://github.com/microsoft/teams.ts)
