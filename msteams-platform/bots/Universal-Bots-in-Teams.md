---
title: Universal Bots in Microsoft Teams
PM:  Ojasvi Choudhary (transitioning from Bill Bliss)
description: Understand Universal Bots in Teams
ms.topic: conceptual
---

# Universal Bots in Microsoft Teams

## Overview

Currently, Adaptive Cards are available across multiple platforms in multiple products (“hosts” in Adaptive Card parlance), but action buttons are product-specific. Except for `Action.OpenUrl`, a button action for one product won’t work in another. Outlook uses `Action.Http` but relies on a product-specific validation process for trusting specific https:// domains. Microsoft Teams uses `Action.Submit` but depends on a Teams-specific `data.msteams:` object schema. Cortana does something different too.

To solve this problem, in summer 2019, David Claux led the launch of a cross-product group (spec [here](https://microsoft.sharepoint-df.com/:w:/r/teams/AdaptiveAppFrameworkv-team/_layouts/15/guestaccess.aspx?e=897HmS&wdLOR=c01AB79F5-B0C8-4B57-82B9-036E574DAB42&share=EVkBNIFHGzdOndgBQvPvGAYBaTMEMGd8vAA8vPBu6cKICA)), and the Bot Framework/SMBA API/protocol/schema, app registration, and service infrastructure was used, specifically focused on using the `invoke` activity.

Here’s how it will work:

1. There is a new Adaptive Card Action, `Action.Invoke`.
2. There is a new invoke type `(name), adaptiveCard/action`.
3. When the user presses an `Action.Invoke` button, the Adaptive Card host client sends a message to whatever subsystem handles `Action.Invoke` requests for the host.  See [Client-Service Protocol](#host-service-integration-conventions).
4. That subsystem generates a Bot Framework `invoke` activity message appropriate for the Bot Framework channel/Direct Line plugin  and does an HTTP POST of that message to the bot.
5. In the body of the response to the HTTP POST, the bot returns an Adaptive Card with a specific schema.
6. The host updates the Adaptive Card JSON of the card body and also updates the message.

## Adaptive Action Schema

The [spec]( https://microsoft.sharepoint-df.com/:w:/t/AdaptiveAppFrameworkv-team/EVkBNIFHGzdOndgBQvPvGAYBaTMEMGd8vAA8vPBu6cKICA?e=897HmS) describes the schema for the new action as such:

```
{ 
  "type": "Action.Execute", 
  "title": "Click me",
  "id": : "<id>",
  "verb": "<name of the method to invoke>", 
  "data": { … } 
}
```

## Invoke Schema

In the [spec]( https://microsoft.sharepoint-df.com/:w:/t/AdaptiveAppFrameworkv-team/EVkBNIFHGzdOndgBQvPvGAYBaTMEMGd8vAA8vPBu6cKICA?e=897HmS), what gets sent over the wire to the bot is specified in the "Channel-to-Bot wire protocol" section. In this section, I'll describe the salient points for what it will look like in Teams since we are heavy users of the `invoke` activity message. 

In [Appendix A – Full Invoke Activity Schema Example](#appendix-a-full-invoke-activity-schema-example), there's a fully-fleshed-out version from Teams captured recently, but here's an excerpt:

```
{
    "name": "task/fetch",
    "type": "invoke",
    "timestamp": "2020-01-11T04:16:01.288Z",
    "localTimestamp": "2020-01-10T20:16:01.288-08:00",
    "id": "f:7258034409303026457",
    "channelId": "msteams",
    "serviceUrl": "https://canary.botapi.skype.com/amer/",
    "from": { … },
    "conversation": { … },
    "recipient": { … },
    "entities": [ … ],
    "channelData": { … },
    "value": { … },
    "locale": "en-US"
}
```

Of the top-level attributes of the invoke message, the only ones that matter are `name` (which, for invoke, really means "type") and value. All the others are channel-specific. This is the schema:

|Attribute | Specification |
|:--------------------|:----------------|
|name | adaptiveCard/action |
|value | A copy of the Action payload is shown below: <br /> ``` <br /> "value": <br /> { <br />   "action": { <br />     "type": "Action.Execute", <br />     "id": "<id>", <br />     "verb": "<verb>", <br />     "data": { … } <br /> } <br /> ``` |

## Host/Service Integration Conventions

We need to standardize how a host implements this behavior if a bot wants to work with multiple hosts. As you can see in [Client-Service Sequence Diagram](#client-service-sequence-diagram), there are timeout and retry semantics to define in how long a host waits for a universal bot to reply and how many times it attempts to contact a universal bot. We also need to define error handling behavior.

1. How long does a host wait for a universal bot to reply before retrying the request?
   It waits to get the value Teams uses.
1. How many times does a host retry the request before giving up and reporting an error?
  Teams retries five times, verifying that as well.
1. How are errors handled? In particular, is there a way to propagate errors from the bot to the client?
    1. A 400 error can occur if the Adaptive Card Action.Invoke payload is malformed. Note that the bot never sees this though, or shouldn't – this should be enforced by the client before it's sent to the "runtime" layer (which in Teams is SMBA).
    1. 500 errors in Teams today are only generated by the "runtime" layer when the bot doesn't respond to HTTP messages. This is then passed back to Teams via SMBA.
    1. Currently, Teams doesn’t support a way to propagate 4xx/5xx errors from the bot back to the client. For example, if a bot wants to say it's too busy and return a 429 error, there's no way to do that today.
    1. There is currently no way for the host to return errors to a bot if what the bot returns are causing problems in the host. This is especially important to solve for this case because it hasn't been as important – in this case, the bot has no way of knowing whether its update has succeeded, failed, and if it failed, why.
1. There are conventions for what the host should show while the invoke is in progress. For example, Teams shows a spinner inside the button associated with the action. (Note that this is complementary to what would be shown if `"displayCurrentCardWhileRefreshing": true` in the autoRefresh section of the card.)
1. The current spec does not call for a dependency on Adaptive Card data binding. This reduces scope, and a different invoke method can be used. If not, something like where a host must look at the returned payload to see if it's an adaptive card or just a new data packet would not be wanted.

## Client-Service Sequence Diagram

<img width="530px" alt="Client-Service Sequence Diagram" src="~/assets/images/bots/ClientServiceSequence.png"/>

> **Note:** This sequence diagram does not currently illustrate `autoRefresh` behavior.
[Link](https://www.lucidchart.com/invitations/accept/c4b6a614-53dc-4681-bd29-6900d17a1c4c)

## Card Auto-Refresh and Card Localization

In the [spec]( https://microsoft.sharepoint-df.com/:w:/t/AdaptiveAppFrameworkv-team/EVkBNIFHGzdOndgBQvPvGAYBaTMEMGd8vAA8vPBu6cKICA?e=897HmS), there's a section called "Automatic card refresh" that proposes a new top-level property in the Adaptive Card schema that looks like this:

```
“autoRefresh”: {
  "frequency": 60,
  "displayCurrentCardWhileRefreshing": true,
  "fallbackOnFailure": {
    "type": "TextBlock",
    "text": "Text shown if auto-refresh fails."
  },
  "action": {
    "type": "Action.Execute",
    "verb": "refreshCard",
    "data": { ... }
    }
},
[…]
```

There are several issues to consider in supporting this in Teams (many of which are mentioned in the spec).

1. Do we support `frequency`? When does the timer start/stop? Is there a limit on the number of refreshes?
2. What about scrolling a card out of view and back again, is refresh triggered?
3. What about channel/chat switch, does that trigger refresh?
4. Is the refreshed card persisted? Can that lead to too many writes?
5. Do we support `displayCurrentCardWhileRefreshing` and `fallbackOnFailure` or is that too expensive?
6. Retry behavior is noted as an open issue in the spec.
7. The current spec calls for `autoRefresh` updates to use the same invoke type (`adaptiveCards/invoke`). Is that wise? Might we want to use a second one, say `adaptiveCards/refresh`? 
8. Should we set a time limit after which refresh does not happen? Is there value in refreshing a year-old card? (Viewing one in Teams is not uncommon because of thread refresh behavior.) If so should this be under programmer control via a schema value?

## Role-Based Views (Kaizala Requirement)

The assumption is that refresh will solve this problem, but this needs to be validated.

## Localization

While one might imagine using something like our manifest-based localization scheme if we assumed support for Adaptive Card templating, that's a fair amount of work. The spec contemplates refresh for showing the initial card in the right language (see the "Localization" section), but there is nothing about this feature that's meaningfully different than how localization is done.

## Appendix A – Full Invoke Activity Schema Example

For reference purposes, here is a full example of what an invoke activity message looks like – this one was received by a Teams bot from SMBA (the Office/Chat Service implementation of Bot Framework):

```
{
    "name": "task/fetch",
    "type": "invoke",
    "timestamp": "2020-01-11T04:16:01.288Z",
    "localTimestamp": "2020-01-10T20:16:01.288-08:00",
    "id": "f:7258034409303026457",
    "channelId": "msteams",
    "serviceUrl": "https://canary.botapi.skype.com/amer/",
    "from": {
        "id": "29:15Kwb6tfWnDYEyIEkAi62kD2j3JKY5DQIRw9h1n5TSBKu16sWri7kV3ooE0xpAKsysqlaAEzt1oAMobGUG8zPahQGcLs-9ayRYOukSHU125Q",
        "name": "Bill Bliss",
        "aadObjectId": "7faf8ab2-3d56-4244-b585-20c8a42ed2b8"
    },
    "conversation": {
        "isGroup": true,
        "conversationType": "channel",
        "tenantId": "72f988bf-86f1-41af-91ab-2d7cd011db47",
        "id": "19:40925f967e714f6f8b2cb01087a6cb55@thread.skype;messageid=1578716147404"
    },
    "recipient": {
        "id": "28:e89fb6c4-38db-4d49-84ba-b09fee71c158",
        "name": "Task Module Debug"
    },
    "entities": [
        {
            "locale": "en-US",
            "country": "US",
            "platform": "Windows",
            "timezone": "America/Los_Angeles",
            "type": "clientInfo"
        }
    ],
    "channelData": {
        "channel": {
            "id": "19:40925f967e714f6f8b2cb01087a6cb55@thread.skype"
        },
        "team": {
            "id": "19:40925f967e714f6f8b2cb01087a6cb55@thread.skype"
        },
        "tenant": {
            "id": "72f988bf-86f1-41af-91ab-2d7cd011db47"
        },
        "source": {
            "name": "compose"
        }
    },
    "value": {
        "data": {
            "taskModule": "youtube",
            "type": "task/fetch"
        },
        "context": {
            "theme": "dark"
        }
    },
    "locale": "en-US"
}
```