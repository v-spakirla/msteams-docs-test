---
title: Teams for Government Clouds
author: v-preethah
description: Learn about the Teams features and capabilities available for Government Community Cloud (GCC), GCC High, and Department of Defense (DoD) environments. Get an overview on how to deploy Teams in government clouds.
ms.topic: article
ms.localizationpriority: high
ms.date: 08/23/2026
---

# Plan for government clouds

Microsoft Teams plays a key role to facilitate secure and efficient communication across various government sectors and agencies. Government entities handle sensitive and confidential data, and there's a crucial need for cloud platforms that are secure and compliant. To meet the stringent security requirements, the specialized government cloud services offered are:

:::row:::
   :::column:::
      :::image type="icon" source="../assets/images/app-fundamentals/gcc.png" link="/microsoftteams/plan-for-government-gcc" border="false":::
   :::column-end:::
   :::column:::
      :::image type="icon" source="../assets/images/app-fundamentals/gcc-high.png" link="/microsoftteams/plan-for-government-gcc-high" border="false":::
   :::column-end:::
   :::column:::
      :::image type="icon" source="../assets/images/app-fundamentals/dod.png" link="/microsoftteams/plan-for-government-dod" border="false":::
   :::column-end:::
:::row-end:::

Government Community Cloud (GCC), GCC High, Department of Defense (DoD) are limited for use within the United States.

## Teams app capabilities

Teams is compatible with all Microsoft 365 Government environments, but the capabilities and features vary by tenant. Due to security and compliance restrictions, all the features available in commercial tenants might not be accessible in government tenants while other features take time to roll out in government tenant. We recommend that you have the understanding of the Teams apps and its capabilities supported in government tenants to ensure effective utilization.

<!--[!INCLUDE [deprecation-note](~/includes/deprecation-note.md)]-->

The following table details the Teams apps and its capabilities for GCC, GCC High, and DoD:

| &nbsp; | GCC | GCC High | DoD |
|-------------|---------|---|---|
| **Apps** | &nbsp; | &nbsp; | &nbsp; |
| Apps built by Microsoft | ✔️ | ✔️ | ✔️ |
| Third-party apps | ✔️ | ❌ | ❌ |
| Custom apps built for your org (LOB apps) distributed and used in specific organization | ✔️ | ✔️ | ✔️ |
| Upload a custom app | ✔️ | ❌ | ❌ |
| **App capabilities** | &nbsp; | &nbsp; | &nbsp; |
| Tabs | ✔️ | ✔️ | ✔️ |
| Bots | ✔️ | ✔️ | ✔️ |
| Message extensions | ✔️ | ✔️ | ✔️ |
| Message actions | ✔️ | ✔️ | ✔️ |
| Cards: Adaptive, Hero, Thumbnail, Microsoft 365 connector, Receipt, Sign in, and OAuth cards | ✔️ | ✔️ | ✔️ |
| Dialogs (referred as task modules in TeamsJS v1.x) | ✔️ | ✔️ | ✔️ |
| Link unfurling | ✔️ | ✔️ | ✔️ |
| Meeting extensions | ✔️ | ✔️ | ✔️ |
| Webhooks and connectors | ✔️ | ✔️ | ❌ |
| Workflows| ✔️ | ❌ | ❌ |
| **Experiences** | &nbsp; | &nbsp; | &nbsp; |
| Teams Store | ✔️ | ✔️ | ✔️ |
| In-context Teams Store or app flyouts | ✔️ | ✔️ | ✔️ |
| Manage apps in Teams | ✔️ | ✔️ | ✔️ |
| Manage apps in Teams Admin Center | ✔️ | ✔️ | ✔️ |
| Graph APIs | ✔️ | ✔️ | ✔️ |
| Developer Portal for Teams | ✔️ | ❌ | ❌ |

For more information on Graph API, see [Graph API for Government clouds](/graph/teamwork-national-cloud-differences).

> [!NOTE]
>
> * Third-party apps are turned off by default for GCC and aren't available for GCC High and DoD. To turn on third-party apps for GCC, see [manage org-wide app settings for Microsoft 365 Government](/microsoftteams/manage-apps).
> * GCC High supports Incoming Webhooks only.

### Images in agent messages and cards

In GCC High and DoD environments, an agent must embed the images that it sends directly in the message as base64 encoded data. Images that are referenced by a URL, such as an image hosted on a web server or a content delivery network (CDN), aren't supported in these environments due to compliance requirements. This requirement applies to the images that an agent sends in messages, in message attachments, and in cards.

To send an image from an agent in GCC High or DoD:

1. Read the image bytes and encode them as a base64 string.
1. Set the attachment's `contentUrl` to a data URI in the `data:<content-type>;base64,<encoded-image-data>` format.
1. Send the attachment with the message.

The following example shows an image attachment that is sent as base64 encoded data:

```json
{
  "name": "image.png",
  "contentType": "image/png",
  "contentUrl": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA..."
}
```

The following code shows how an agent sends a base64 encoded image:

```csharp
using Microsoft.Teams.Api;

// Encode the image as base64 data.
var imageData = Convert.ToBase64String(File.ReadAllBytes(filePath));

var reply = new MessageActivity("Here's the image you requested.");
reply.AddAttachment(new Attachment
{
    Name = "image.png",
    ContentType = "image/png",
    ContentUrl = $"data:image/png;base64,{imageData}",
});

await context.SendAsync(reply, cancellationToken);
```

To display an image in an Adaptive Card, set the `url` property of the `Image` element to the same data URI:

```json
{
  "type": "Image",
  "url": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUA..."
}
```

> [!NOTE]
> The 100-KB agent message size limit doesn't include base64 encoded images. For more information, see [format your agent messages](~/bots/how-to/format-your-bot-messages.md).

### Plan to deploy Teams in government clouds

To deploy Teams in GCC, GCC High, or DoD, you must purchase a suitable [Microsoft 365 Government plan](https://products.office.com/government/compare-office-365-government-plans). US federal, state, local or tribal government entity, or other entities that handle data subject to government regulations can opt for a government cloud service license. For more information, see [Teams for Government](/microsoftteams/expand-teams-across-your-org/teams-for-government-landing-page).

### Compliance with third-party services

Here are few pointers to consider while connecting with third-party services from government clouds:

> [!div class="checklist"]
>
> * Understand that by enabling third-party communication, the communication is processed through the third party and not Microsoft.
> * Understand that enabling bots extend your system boundary beyond this tenant, and is based on the bot you choose to use. You must ensure that the bot meets the compliance requirements including FedRAMP, DFARS, and ITAR.
> * Understand that Microsoft doesn't endorse and makes no warranties concerning the security of third parties that the customer allows to connect with their service.
> * Mitigate risks associated with connecting to third-party bots in their services.
> * Evaluate the risk and compliance of any endpoint and URL that the users connect to.

## See also

* [Format your agent messages](~/bots/how-to/format-your-bot-messages.md)
* [Deployment overview](/microsoftteams/deploy-overview)
* [Plan for governance in Teams](/microsoftteams/plan-teams-governance)
* [Customer eligibility](/office365/servicedescriptions/office-365-platform-service-description/office-365-us-government/office-365-us-government)
* [Microsoft 365 Government how to buy](/office365/servicedescriptions/office-365-platform-service-description/office-365-us-government/microsoft-365-government-how-to-buy)
* [Manage org-wide app settings](/microsoftteams/manage-apps)
