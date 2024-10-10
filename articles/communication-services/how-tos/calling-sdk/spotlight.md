---
title: Spotlight states
titleSuffix: An Azure Communication Services how-to guide
description: Use Azure Communication Services SDKs to send spotlight state.
author: cnwankwo
ms.author: cnwankwo
ms.service: azure-communication-services
ms.subservice: teams-interop
ms.topic: how-to 
ms.date: 03/01/2023
ms.custom: template-how-to
zone_pivot_groups: acs-plat-web-ios-android-windows
---

# Spotlight states

This article describes how to implement Microsoft Teams spotlight capability with Azure Communication Services Calling SDKs. This capability enables users in the call or meeting to pin and unpin videos for everyone. The maximum limit of pinned videos is seven.

Since the video stream resolution of a participant is increased when spotlighted, it should be noted that the settings done on [Video Constraints](../../concepts/voice-video-calling/video-constraints.md) also apply to spotlight.

## Overview

Spotlighting a video is like pinning it for everyone in the meeting. The organizer, co-organizer, or presenter can choose up to seven people's video feeds (including their own) to highlight for everyone else.

The two different ways to spotlight are your own video and someone else's video (up to seven people).

When a user spotlights or pins someone in the meeting, this increases the height or width of the participant grid depending on the orientation.

Remind participants that they can't spotlight a video if their view is set to Large gallery or Together mode.

## Prerequisites

- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F). 
- A deployed Communication Services resource. [Create a Communication Services resource](../../quickstarts/create-communication-resource.md).
- A user access token to enable the calling client. For more information, see [Create and manage access tokens](../../quickstarts/identity/access-tokens.md).
- Optional: Complete the quickstart to [add voice calling to your application](../../quickstarts/voice-video-calling/getting-started-with-calling.md)


## Support

The following tables define support for spotlight in Azure Communication Services.

## Identities and call types

The following table shows support for call and identity types. 

| Identities | Teams meeting | Room | 1:1 call | Group call | 1:1 Teams interop call | Group Teams interop call |
| --- | --- | --- | --- | --- | --- | --- |
|Communication Services user	| ✔️ |   ✔️   |    |  ✔️  |	   |  ✔️    |
|Microsoft 365 user | ✔️  |   ✔️  |         |   ✔️    |     |  ✔️    |

## Operations

Azure Communication Services or Microsoft 365 users can call spotlight operations based on role type and conversation type.

The following table shows support for individual operations in Calling SDK to individual identity types.

**In one-to-one calls, group calls, and meeting scenarios the following operations are supported for both Communication Services and Microsoft 365 users**

| Operations | Communication Services user | Microsoft 365 user |
| --- | --- | --- |
| `startSpotlight` | ✔️ [1] | ✔️ [1] |
| `stopSpotlight` | ✔️ | ✔️ |
| `stopAllSpotlight` |  ✔️ [1] | ✔️ [1] | 
| `getSpotlightedParticipants` |  ✔️ | ✔️ | 
| `StartSpotlightAsync` |  ✔️ [1] | ✔️ [1] | 
| `StopSpotlightAsync` |  ✔️ [1] | ✔️ [1] | 
| `StopAllSpotlightAsync` |  ✔️ [1] | ✔️ [1] | 
| `SpotlightedParticipants` |  ✔️ [1] | ✔️ [1] | 
| `MaxSupported` |  ✔️ [1] | ✔️ [1] | 

[1] In Teams meeting scenarios, these operations are only available to users with role organizer, co-organizer, or presenter.

## SDKs

The following table shows support for spotlight feature in individual Azure Communication Services SDKs.

| Platforms | Web | Web UI | iOS | iOS UI | Android | Android UI | Windows |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Is Supported | ✔️  |  ✔️  |  ✔️  |     |  ✔️  |    |  ✔️  |

::: zone pivot="platform-web"
[!INCLUDE [Spotlight Client-side JavaScript](./includes/spotlight/spotlight-web.md)]
::: zone-end

::: zone pivot="platform-android"
[!INCLUDE [Spotlight Client-side Android](./includes/spotlight/spotlight-android.md)]
::: zone-end

::: zone pivot="platform-ios"
[!INCLUDE [Spotlight Client-side IOS](./includes/spotlight/spotlight-ios.md)]
::: zone-end

::: zone pivot="platform-windows"
[!INCLUDE [Spotlight Client-side Windows](./includes/spotlight/spotlight-windows.md)]
::: zone-end

## Troubleshooting

| Code | Subcode | Result Category | Reason | Resolution |
| --- | --- | --- | --- | --- |
| 400	| 45900 | ExpectedError  | All provided participant IDs are already spotlighted. | Only participants who aren't currently spotlighted can be spotlighted. |
| 400 | 45902	| ExpectedError | The maximum number of participants are already spotlighted. | Only seven participants can be in the spotlight state at any given time. |
| 403 | 45903	| ExpectedError | Only participants with the roles of organizer, co-organizer, or presenter can initiate a spotlight. | Ensure the participant calling the `startSpotlight` operation has the role of organizer, co-organizer, or presenter. |

## Next steps

- [Learn how to manage calls](./manage-calls.md)
- [Learn how to manage video](./manage-video.md)
- [Learn how to record calls](./record-calls.md)
- [Learn how to transcribe calls](./call-transcription.md)
