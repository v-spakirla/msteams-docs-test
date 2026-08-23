> [!IMPORTANT]
>
> In GCC High and DoD environments, an agent must embed images directly in the message as base64 encoded data. Images that are referenced by a URL aren't supported in these environments due to compliance requirements. Encode the image and set the attachment's `contentUrl` to a data URI, such as `data:image/png;base64,<encoded-image-data>`. For more information, see [images in agent messages and cards](~/concepts/cloud-overview.md#images-in-agent-messages-and-cards).
