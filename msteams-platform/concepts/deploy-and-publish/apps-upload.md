---
title: Upload your custom app
description: Learn how to upload your custom app in Microsoft Teams. Custom app upload is common when testing and debugging an app during development.
ms.topic: how-to
ms.localizationpriority: high
ms.date: 10/19/2026
---

# Upload your app in Teams

You can upload Microsoft Teams apps without having to publish to your organization or the Microsoft Teams Store in the following scenarios:

* You want to test and debug an app locally yourself or with other developers.
* You built an app for yourself to automate a workflow.
* You built an app for a small set of users, such as, your work group.

> [!IMPORTANT]
>
> * Custom app upload is possible only in Government Community Cloud (GCC) and isn't possible in GCC High, Department of Defense (DoD), and Teams operated by 21Vianet.
> * App upload and installation are supported in Teams for desktop and web (availability depends on your organization’s app setup policies and cloud environment)
> * We recommend you to avoid multiple uploads of your message extension app to Teams, as it results in multiple instances of the message extension.

Users can't chat with bots that aren't connected to a Microsoft Teams app such as Azure bots created through the Bot Framework or Power Virtual Agents. We recommend you to create a new Teams app connected to the Azure Bot using the [Developer Portal](../build-and-test/manage-your-apps-in-developer-portal.md#configure), publish it to Teams, and allow users to install the app using the [app permission policies](/microsoftteams/teams-app-permission-policies#edit-an-app-permission-policy).

If you want to test the app, you can assign users an [app setup policy](/microsoftteams/teams-app-setup-policies#manage-app-setup-policies) that allows uploading custom apps and chat with the Azure bot through a direct link to a chat.

## Prerequisites

> [!IMPORTANT]
> Upload your custom app in the same organization where the app was created. If the app manifest specifies `webApplicationInfo.id`, the app registration referenced by that ID must also belong to the organization where you install the app. Check the organization you're signed in to in Teams before uploading. If you receive a same-organization error, see [Resolve a same-organization installation error](#resolve-a-same-organization-installation-error).

* Ensure to create your [app package](~/concepts/build-and-test/apps-package.md), import it and [validate it](https://dev.teams.microsoft.com/tools/store-validation) for errors.
* [Enable custom app uploading](~/concepts/build-and-test/prepare-your-o365-tenant.md#enable-custom-teams-apps-and-configure-custom-app-upload-settings) in Teams.
* Ensure that your app is running and accessible using HTTPs.

## Upload your app

You can upload your app to a team, chat, meeting, or for personal use depending on how you configured your app's scope.

1. Log in to the Teams client with your [Microsoft 365 development account](https://developer.microsoft.com/microsoft-365/dev-program).

1. Select **Apps** > **Manage your apps** and **Upload an app**.

    :::image type="content" source="~/assets/images/publish-app/upload-an-app.png" alt-text="Screenshot shows the option to upload an app highlighted in red.":::

1. Select **Upload a custom app**.

   :::image type="content" source="~/assets/images/publish-app/upload-custom-app.png" alt-text="Screenshot shows the option to upload a custom app.":::

1. Select your app package .zip file.
1. Select **Add**.

    :::image type="content" source="~/assets/images/publish-app/teams-app-detail.png" alt-text="Screenshot of the custom app dialog with the Add option highlighted.":::

1. Select **Open** to open the app in personal scope.

    Alternatively, you can either search and select the required scope or select a channel or chat from the list, and move through the dialog to select **Go**.

    :::image type="content" source="~/assets/images/publish-app/teams-app-scope.png" alt-text="Screenshot of the scope selection dialog with the list of shared scopes.":::

> [!div class="nextstepaction"]
> [I ran into an issue](https://github.com/MicrosoftDocs/msteams-docs/issues/new?template=Doc-Feedback.yaml&title=%5BI+ran+into+an+issue%5D+Upload+your+app&&author=%40surbhigupta&pageUrl=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fmicrosoftteams%2Fplatform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload%23upload-your-app&contentSourceUrl=https%3A%2F%2Fgithub.com%2FMicrosoftDocs%2Fmsteams-docs%2Fblob%2Fmain%2Fmsteams-platform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload.md&documentVersionIndependentId=5f89865b-a9fd-ada5-72d2-6375357bba8c&platformId=0e3954be-8892-442b-b769-b6284670b9db&metadata=*%2BID%253A%2Be473e1f3-69f5-bcfa-bcab-54b098b59c80%2B%250A*%2BService%253A%2B%2A%2Amsteams%2A%2A)

## Troubleshoot

If your app fails to upload or encounters any issues to upload, check the following options:

1. Ensure that you follow all the instructions for [creating your app package](../../concepts/build-and-test/apps-package.md).
1. Upload your app package and then [validate it](https://dev.teams.microsoft.com/tools/store-validation).
1. Ensure your app manifest matches with the latest [schema](/microsoft-365/extensibility/schema).

### Resolve a same-organization installation error

If your custom app or its associated app registration belongs to a different organization from the installation organization, Teams displays:

> Custom apps must come from the same organization that they are being installed in, and the webApplicationInfo.id must be registered in the same organization.

You might also encounter the error identifier `WebApplicationInfoIdOfSideloadedAppMustBeInTheSameTenantAsUser`.

To resolve the error:

1. Check the account and organization you're using in Teams. Switch to the organization where you intend to upload and install the app.
1. In [Developer Portal](../build-and-test/manage-your-apps-in-developer-portal.md#configure), open the app in that organization. Under **Configure** > **Basic information**, check its **Application (client) ID**. Compare it with `webApplicationInfo.id` in the `manifest.json` file in the package you're uploading. Don't use the top-level manifest `id` or an app registration's object ID in place of the application (client) ID.
1. Check that the referenced app registration belongs to the installation organization. In the Microsoft Entra admin center, select the appropriate directory and find the existing app under **App registrations**. Verify its application (client) ID and directory (tenant) ID on the **Overview** page. For app registration and directory-selection guidance, see [Register an application with the Microsoft identity platform](/graph/auth-register-app-v2).
1. If the app or registration belongs to another organization, use an app configured for the target organization through the [Teams agent quickstart](../../agents-in-teams/quickstart-create-agent-teams-sdk.md) or your existing setup process. Ensure the app configuration and package reference its corresponding application (client) ID, then create an updated [app package](../build-and-test/apps-package.md) and try uploading again.

This requirement applies to custom app upload. It doesn't mean that apps distributed through every other method must be registered in each installation organization. For an app that requests resource-specific consent (RSC), see [Declare RSC permission requirements for an agent or app](../../graph-api/rsc/grant-resource-specific-consent.md).

## Manage your apps

Manage your apps allows users to have a dedicated place to manage, update and remove their apps, permissions, and subscriptions on the Teams client. The users can install the apps from **Manage your apps**.

### Access your app

To access apps through **Manage your apps**, follow the steps:

1. Go to **Apps** and select **Manage your apps** in Teams to view the installed apps across all your channels or for personal use in a list format.

    :::image type="content" source="~/assets/images/publish-app/manage-apps-list.png" alt-text="Screenshot shows the list of installed apps that appears when you select manage your apps.":::

1. Select the app dropdown to view the list of scopes where the app is installed.

    :::image type="content" source="~/assets/images/publish-app/app-scopes.png" alt-text="Screenshot shows the list of scopes where the app is installed.":::

1. Select the scope of app to go to the app in the channel or personal view. The list of scopes consists of personal scope and teams scope only. Apps installed in group chat scope aren't displayed in this view.

Teams provides several ways to open apps. For more information, see [access your apps in Teams](https://support.microsoft.com/office/access-your-apps-in-teams-0758cb09-9e85-40e7-a974-51df7734646a).

> [!div class="nextstepaction"]
> [I ran into an issue](https://github.com/MicrosoftDocs/msteams-docs/issues/new?template=Doc-Feedback.yaml&title=%5BI+ran+into+an+issue%5D+Access+your+app&&author=%40surbhigupta&pageUrl=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fmicrosoftteams%2Fplatform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload%23access-your-app&contentSourceUrl=https%3A%2F%2Fgithub.com%2FMicrosoftDocs%2Fmsteams-docs%2Fblob%2Fmain%2Fmsteams-platform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload.md&documentVersionIndependentId=5f89865b-a9fd-ada5-72d2-6375357bba8c&platformId=0e3954be-8892-442b-b769-b6284670b9db&metadata=*%2BID%253A%2Be473e1f3-69f5-bcfa-bcab-54b098b59c80%2B%250A*%2BService%253A%2B%2A%2Amsteams%2A%2A)

### Update your app

You don't have to upload your custom app again if you make code changes (these are reflected in Teams in real-time). However, you must reinstall if you change any app configurations. For more information, see [Role of an admin in upgrading Teams apps](/MicrosoftTeams/apps-update-experience).

When a Teams app receives an update, the user must consent to update the app in any one context, and the app is automatically updated in all the related contexts (chats, channels, and meetings) where the app is installed. The user can update the app from any context.

If an update is available for your app, follow these steps to update your app from the Teams Store:

1. Go to **Microsoft Teams**.
1. Select **Apps** > **Manage your apps**.

    :::image type="content" source="../../assets/images/publish-app/update-available-manage-apps.png" alt-text="Screenshot shows the updates available for apps installed in the manage your apps option.":::

1. Under the **Apps** tab, check for the **Update** option next to the app.

1. Select **Update**.

   :::image type="content" source="../../assets/images/publish-app/update-available.png" alt-text="Screenshot shows the option to select update for the Teams app.":::

1. Select **Update now**.

   :::image type="content" source="../../assets/images/publish-app/update-window.png" alt-text="Screenshot shows the option to update the app installed in Teams.":::

   The app is automatically updated in all the related contexts (chats, channels, and meetings) where the app is installed.

> [!div class="nextstepaction"]
> [I ran into an issue](https://github.com/MicrosoftDocs/msteams-docs/issues/new?template=Doc-Feedback.yaml&title=%5BI+ran+into+an+issue%5D+Update+your+app&&author=%40surbhigupta&pageUrl=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fmicrosoftteams%2Fplatform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload%23update-your-app&contentSourceUrl=https%3A%2F%2Fgithub.com%2FMicrosoftDocs%2Fmsteams-docs%2Fblob%2Fmain%2Fmsteams-platform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload.md&documentVersionIndependentId=5f89865b-a9fd-ada5-72d2-6375357bba8c&platformId=0e3954be-8892-442b-b769-b6284670b9db&metadata=*%2BID%253A%2Be473e1f3-69f5-bcfa-bcab-54b098b59c80%2B%250A*%2BService%253A%2B%2A%2Amsteams%2A%2A)

### Remove your app

To remove app from Teams, follow the steps:

1. Find the app in **Manage your apps**.

1. Select &nbsp;:::image type="content" source="../../assets/images/publish-app/bin-icon.png" alt-text="Screenshot shows the delete icon to remove app in Teams.":::&nbsp; at the scope of the installed app.

    :::image type="content" source="../../assets/images/publish-app/uninstall-from-channel.png" alt-text="Screenshot shows the option to remove app in a channel highlighted in red.":::

1. Select **Remove** to remove your app.

    :::image type="content" source="../../assets/images/publish-app/remove-app-teams.png" alt-text="Screenshot shows the option to select remove option in teams.":::

> [!NOTE]
>
> * You can't remove personal bot activity entirely. If you remove the app and add it again, new communication with the bot appends to the previous conversation with it.
> * You can't migrate your custom app to the Teams Store. If you want to list your app to the Teams Store, see [how to publish your app to the Teams Store](appsource/publish.md).

> [!div class="nextstepaction"]
> [I ran into an issue](https://github.com/MicrosoftDocs/msteams-docs/issues/new?template=Doc-Feedback.yaml&title=%5BI+ran+into+an+issue%5D+Remove+your+app&&author=%40surbhigupta&pageUrl=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fmicrosoftteams%2Fplatform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload%23remove-your-app&contentSourceUrl=https%3A%2F%2Fgithub.com%2FMicrosoftDocs%2Fmsteams-docs%2Fblob%2Fmain%2Fmsteams-platform%2Fconcepts%2Fdeploy-and-publish%2Fapps-upload.md&documentVersionIndependentId=5f89865b-a9fd-ada5-72d2-6375357bba8c&platformId=0e3954be-8892-442b-b769-b6284670b9db&metadata=*%2BID%253A%2Be473e1f3-69f5-bcfa-bcab-54b098b59c80%2B%250A*%2BService%253A%2B%2A%2Amsteams%2A%2A)

## Next step

> [!div class="nextstepaction"]
>[Create apps for Teams meetings](../../apps-in-teams-meetings/teams-apps-in-meetings.md)

## See also

* [Configure default install options](~/concepts/deploy-and-publish/add-default-install-scope.md)
* [Maintain your published Microsoft Teams app](~/concepts/deploy-and-publish/appsource/post-publish/overview.md)
* [Add app to chat](/graph/api/chat-post-installedapps)
* [Manage custom apps](/microsoftteams/teams-custom-app-policies-and-settings)
