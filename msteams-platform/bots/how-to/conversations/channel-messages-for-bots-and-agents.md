---
title: Get All Channel and Chat Messages
description: Learn how to declare ChannelMessage.Read.Group and ChatMessage.Read.Chat RSC permissions so your agent receives all channel and chat messages without an @mention.
ms.topic: article
ms.localizationpriority: medium
ms.date: 09/24/2026
zone_pivot_groups: teams-sdk-languages
---

# Receive all channel and chat messages

By default, an agent installed in a team or a group chat receives a message only when it's @mentioned. To receive every message in the conversation, declare the following resource-specific consent (RSC) permissions in your app manifest:

* `ChannelMessage.Read.Group`: Receive all messages in the channels of the team where the app is installed.
* `ChatMessage.Read.Chat`: Receive all messages in the group chat where the app is installed.

A team owner or chat member grants consent to these permissions when they install or update the app in the conversation. For more information about RSC, see [permissions in Teams app](../../../graph-api/App-permissions/Teams-app-permissions.md) and [resource-specific consent for your Teams app](../../../graph-api/rsc/resource-specific-consent.md).

> [!NOTE]
> Along with the public cloud, this capability is also available in [Government Community Cloud (GCC), GCC High, and Department of Defense (DoD)](../../../concepts/cloud-overview.md#teams-app-capabilities) environments, and in [Teams operated by 21Vianet](../../../concepts/sovereign-cloud.md).

## Update the app manifest

Add the RSC permissions your agent needs to the `authorization.permissions.resourceSpecific` property of your app manifest:

```json
{
    "webApplicationInfo": {
        "id": "00000000-0000-0000-0000-000000000000",
        "resource": "https://AnyString"
    },
    "authorization": {
        "permissions": {
            "resourceSpecific": [
                {
                    "type": "Application",
                    "name": "ChannelMessage.Read.Group"
                },
                {
                    "type": "Application",
                    "name": "ChatMessage.Read.Chat"
                }
            ]
        }
    }
}
```

* `webApplicationInfo.id`: Microsoft Entra app ID for your app. Set this to your agent's app ID so Teams can map the granted permissions to your app.
* `webApplicationInfo.resource`: Placeholder resource string. Set this to any string value, such as `https://AnyString`, because RSC ignores the value but requires the property.
* `authorization.permissions.resourceSpecific`: RSC permissions granted on install. Set this to `ChannelMessage.Read.Group`, `ChatMessage.Read.Chat`, or both, depending on the conversation types your agent supports.

Declare only the permissions your agent uses. If your agent is installed in teams only, `ChannelMessage.Read.Group` is sufficient. For the full list of permission strings, see [supported RSC permissions](../../../graph-api/rsc/resource-specific-consent.md#supported-rsc-permissions).

> [!IMPORTANT]
> An existing installation doesn't pick up `ChatMessage.Read.Chat`. Your agent receives all chat messages only after the app is installed again or updated in the chat.

### Update permissions in Developer Portal

If you manage your app in Developer Portal instead of editing the manifest file directly:

1. Sign in to [Developer Portal](https://dev.teams.microsoft.com/) and select **Apps**.
1. Select your app, and then go to **Configure** > **Permissions**.
1. Select **ChannelMessage.Read.Group**, **ChatMessage.Read.Chat**, or both.
1. Select **Save**, and then download the updated app package.

For more information, see [manage RSC permissions in Developer Portal](../../../graph-api/App-permissions/Teams-app-permissions.md#resource-specific-consent).

## Filter @mention messages

After the RSC permissions are granted, your agent's message handler runs for every message in the conversation. If your agent is meant to respond only when it's addressed, check for a mention of your agent and return early for all other messages.

::: zone pivot="teams-sdk-csharp"

```csharp
app.OnMessage(async (context, cancellationToken) =>
{
    var isMentioned = context.Activity.Entities?
        .Any(entity => entity.Type == "mention"
            && entity.Properties["mentioned"]?["id"]?.ToString() == context.Activity.Recipient.Id) ?? false;

    // Remove this check to process every message in the conversation.
    if (!isMentioned)
    {
        return;
    }

    await context.Send("Using RSC, the agent receives messages across channels and chats without being @mentioned.", cancellationToken);
});
```

::: zone-end

::: zone pivot="teams-sdk-typescript"

```typescript
app.on('message', async ({ activity, send }) => {
  const isMentioned = activity.entities?.some(
    (entity) => entity.type === 'mention' && entity.mentioned?.id === activity.recipient.id
  );

  // Remove this check to process every message in the conversation.
  if (!isMentioned) {
    return;
  }

  await send('Using RSC, the agent receives messages across channels and chats without being @mentioned.');
});
```

::: zone-end

::: zone pivot="teams-sdk-python"

```python
@app.on_message
async def handle_message(ctx: ActivityContext[MessageActivity]):
    mentions = [e for e in (ctx.activity.entities or []) if e.type == "mention"]
    is_mentioned = any(m.mentioned.id == ctx.activity.recipient.id for m in mentions)

    # Remove this check to process every message in the conversation.
    if not is_mentioned:
        return

    await ctx.send("Using RSC, the agent receives messages across channels and chats without being @mentioned.")
```

::: zone-end

* Mention entities: Structured mention data in the activity payload. Compare each entity's `mentioned.id` to the activity recipient ID so the agent reacts only to mentions of itself, not of other users.
* Early return: Exit path for unaddressed messages. Remove it when your agent must process the full conversation, such as for summarization or moderation.

For more information about reading and stripping mentions, see [work with mentions](channel-and-group-conversations.md#work-with-mentions).

## Access message history with Microsoft Graph

RSC permissions deliver messages to your agent from the moment consent is granted. To read messages that were sent before installation, or to read messages in bulk, call the Microsoft Graph APIs for channel and chat messages with the same RSC permissions. For more information, see [list replies to messages in a channel](/graph/api/chatmessage-list-replies?view=graph-rest-1.0&tabs=http&preserve-view=true).

## Update your app description for Store approval

To pass Microsoft Teams Store validation, your app description must explain how your app uses the messages it reads:

* State that the app reads only the information required for its core functions and how that data supports the business need it addresses.
* Explain the user-facing value of reading all messages, such as summarization, moderation, or proactive support.
* Don't use `ChannelMessage.Read.Group` or `ChatMessage.Read.Chat` to extract large amounts of customer data.
* Note that `ChatMessage.Read.Chat` allows the app to read chat messages without a signed-in user. For more information, see [Microsoft Graph permissions reference](/graph/permissions-reference).

For more information, see [app descriptions](../../../concepts/deploy-and-publish/appsource/prepare/teams-store-validation-guidelines.md#app-descriptions).

## Code sample

| Sample name | Description | .NET | Node.js | Python | App manifest |
| --- | --- | --- | --- | --- | --- |
| Channel messages with RSC permissions | Shows how an agent receives all channel messages with RSC without being @mentioned. | [View](https://github.com/OfficeDev/Microsoft-Teams-Samples/tree/main/samples/TeamsSDK/Archived/bot-receive-channel-messages-withRSC/csharp) | [View](https://github.com/OfficeDev/Microsoft-Teams-Samples/tree/main/samples/TeamsSDK/Archived/bot-receive-channel-messages-withRSC/nodejs) | [View](https://github.com/OfficeDev/Microsoft-Teams-Samples/tree/main/samples/TeamsSDK/Archived/bot-receive-channel-messages-withRSC/python) | [View](https://github.com/OfficeDev/Microsoft-Teams-Samples/blob/main/samples/TeamsSDK/Archived/bot-receive-channel-messages-withRSC/csharp/demo-manifest/Bot-RSC.zip) |

## See also

* [Send and receive messages](../../build-conversational-capability.md)
* [Channel and group chat conversations for agents](channel-and-group-conversations.md)
* [Permissions in Teams app](../../../graph-api/App-permissions/Teams-app-permissions.md)
* [Resource-specific consent for your Teams app](../../../graph-api/rsc/resource-specific-consent.md)
* [Test resource-specific consent permissions in Teams](../../../graph-api/rsc/test-resource-specific-consent.md)
* [Upload your app in Teams](../../../concepts/deploy-and-publish/apps-upload.md)
