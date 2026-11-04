---
title: Send the widget
description: Learn how to create an HTML widget and send it in an extended Markdown message from a Teams SDK bot.
ms.topic: how-to
ms.localizationpriority: medium
ms.date: 11/04/2026
---

# Send the widget

An HTML widget is an HTML document that Teams renders in a sandboxed iframe inside a chat message. Use widgets to bring a custom interactive interface into a conversation, so users can work with your bot without leaving the chat. Your bot sends the HTML in an extended Markdown message, and the widget connects to Teams to resize its interface and call the bot's tools.

This article shows how to create the widget file and send its payload from a TypeScript bot built with the [Teams SDK](https://github.com/microsoft/teams.ts). Then follow [Share the widget with Teams](share-the-widget-with-teams.md) to add the handshake, height reporting, and tool calls that make the widget interactive. In that article, *share* means connecting the embedded widget to its Teams host, not forwarding a message or publishing an app.

> [!NOTE]
> HTML widgets are a preview feature. The Teams SDK marks the related APIs as experimental, and they may change.

## Prerequisites

- A working TypeScript bot built with the Teams SDK.
- A public HTTPS endpoint for your bot.

## User experience

The example is a dice roller. When you complete both articles, your bot sends a widget with a **Roll a 6-sided die** button. Selecting the button asks the bot to roll the die, adds the result to a list, and grows the widget to fit the results.

This article creates the HTML shell and sends it. The shell's root element is empty until you add the interface in [Share the widget with Teams](share-the-widget-with-teams.md#call-a-tool-from-the-widget).

## What you write

You write two pieces of code:

| Piece | What it is | Responsibilities |
| --- | --- | --- |
| Widget | One HTML file with inline CSS and plain JavaScript. | Contains the interface. In the next article, it performs the handshake, reports its height, and calls tools. |
| Bot | TypeScript code in your existing Teams SDK bot. | Sends the widget in an extended Markdown message. In the next article, it answers tool calls. |

## Project layout

Add these files to your existing bot project:

```text
widgets/
  dice-widget.html     # HTML, CSS, and JavaScript in one file
src/
  bot/
    widget-markdown.ts # Wraps a widget payload in a fenced code block
    dice-widget.ts     # Builds the payload from dice-widget.html
```

Follow this master procedure to send the widget:

1. [Create the widget file](#create-the-widget-file).
1. [Send the widget](#send-the-widget).

## Implement sending a widget

### Create the widget file

Create `widgets/dice-widget.html`. This is the complete page that Teams displays. Leave the `<script>` element empty for now. In [Share the widget with Teams](share-the-widget-with-teams.md), you add the connection, height-reporting, and tool-calling code.

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
| `mcpUrl` | For MCP widgets | The URL of the MCP server whose tool produced the widget. Always set it for those widgets. Teams uses it to run the widget on a dedicated domain scoped to the bot's ID and the MCP URL. See [Widgets from MCP servers](share-the-widget-with-teams.md#widgets-from-mcp-servers). |

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
> This example implements its own handshake and height reporting in [Share the widget with Teams](share-the-widget-with-teams.md). Don't use `buildHtmlWidgetMessage` or `buildHtmlWidgetMarkdown` from `@microsoft/teams.apps` for this example. Their injected protocol code can conflict with the widget's own implementation.

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

`<hash>` depends on where the widget comes from:

| Widget | What to set in the payload | Domain scoped to | Text to hash |
| --- | --- | --- | --- |
| Your bot's own widgets, such as the dice roller | No `mcpUrl` is needed. | Your bot's ID | `<bot id>` |
| A widget produced by an MCP tool | Set `mcpUrl` to the MCP server's URL. | Your bot's ID and the MCP URL | `<bot id>-<mcp URL>` |

When you send a widget produced by an MCP tool, pass the MCP server's URL in the `mcpUrl` field of the widget's payload. Teams generates the hash from your bot's ID and that URL, so the widget gets its own domain.

To let a widget call one of your services, such as an API listed in `connectDomains`, configure that service to accept calls from the widget's domain, for example in its CORS settings.

To find the domain, generate the hash with the [widget domain tool](https://widgettool.z19.web.core.windows.net/). Enter either your bot's ID or a string in the format `<bot id>-<mcp URL>`.

> [!WARNING]
> If you omit `mcpUrl` from a widget produced by an MCP tool, the widget runs on the domain scoped to your bot's ID only. It can then call any service that accepts calls from that domain, including services meant for your bot's own widgets. Always pass `mcpUrl` for widgets produced by MCP tools.

## Handle errors

<!-- TODO: Confirm service error codes and developer actions with engineering. The source doesn't provide status or error codes for sending widgets. -->

The security-policy validator returns warnings for external origins that aren't allowed by the declared policy. Add each required origin to the appropriate policy field before you send the widget.

## Code sample

| Sample name | Description | TypeScript |
| --- | --- | --- |
| HTML widgets example | Demonstrates widget messages, payload validation, and tool calls with the Teams SDK. | [View](https://github.com/microsoft/teams.ts/tree/main/examples/html-widgets) |

## Design guidelines and best practices

- Keep the example's HTML, CSS, and JavaScript in one file so each message contains the complete widget.
- Include only the external origins your widget needs in its security policy.
- Use surrounding Markdown to explain the action users can take in the widget.
- Complete the [Teams handshake and height reporting](share-the-widget-with-teams.md) before expecting the dice widget to be interactive.

> [!WARNING]
> Anyone who can see the message can read the widget's HTML. Never put secrets, tokens, or private data in it.

## Limitations and known issues

- HTML widgets and the related Teams SDK APIs are in preview.
- Sent messages retain their original HTML. Updating the file doesn't update widgets in existing messages.
- This walkthrough provides TypeScript bot code and browser JavaScript. It doesn't provide .NET or Python implementations.
- The `mcpUrl` field described in this preview isn't part of the published `IHtmlWidgetPayload` type. The [MCP example](share-the-widget-with-teams.md#call-the-tool-and-create-the-widget) extends the type to include it.

## Checklist

- [ ] The widget is one HTML file with inline CSS and JavaScript.
- [ ] The widget is sent in an extended Markdown message (`textFormat: "extendedmarkdown"`), as an `html-widget` fenced code block containing the payload.
- [ ] The message isn't built with `buildHtmlWidgetMessage` or `buildHtmlWidgetMarkdown` when using this walkthrough's own protocol implementation.
- [ ] Any origin the HTML loads from is in `resourceDomains`; any origin it calls is in `connectDomains`.
- [ ] The HTML contains no secrets.
- [ ] Services that the widget calls accept calls from `https://<hash>.widget-renderer.usercontent.microsoft`.
- [ ] Each widget produced by an MCP tool includes `mcpUrl` so it uses the domain scoped to that MCP server and your bot.

## Reference: messages

| Code | Message | Kind | Payload | Reply |
| --- | --- | --- | --- | --- |
| Bot | Message activity with `textFormat: "extendedmarkdown"` | Message | `text` contains an `html-widget` fenced code block with the JSON widget payload. | Not a widget protocol request. |

For the handshake, height notifications, and tool-call messages, see [Reference: messages](share-the-widget-with-teams.md#reference-messages) in the next article.

## Next step

> [!div class="nextstepaction"]
> [Share the widget with Teams](share-the-widget-with-teams.md)

## See also

- [Format your agent messages](how-to/format-your-bot-messages.md)
- [Share the widget with Teams](share-the-widget-with-teams.md)
- [Teams SDK](https://github.com/microsoft/teams.ts)
