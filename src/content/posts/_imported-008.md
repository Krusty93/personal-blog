+++
date = '2023-02-01T22:54:57Z'
title = "[Imported] Sending an alert in Microsoft Teams from Azure DevOps Pipelines"
description = 'Imported post about how to send an alert in Microsoft Teams from Azure DevOps Pipelines using Teams webhooks'
url = 'azure-devops-teams-alert'
tags = [
    "imported-devto",
    "cloud",
    "tooling",
    "tutorial",
    "devops"
]
+++

**This post was originally published on [Dev.to](https://dev.to/krusty93/sending-an-alert-in-microsoft-teams-from-azure-devops-pipelines-419k).**

Sometimes there is the need to warn your team about an on going deployment to the cloud. And if you are using Microsoft Teams as primary communication platform, you are likely to have a private channel with your team. This is my case, where anyone is going to run a release pipeline warns the rest of the team about it.
Since manually sending messages is annoying, we automated this operation by using Teams webhooks - without the help of Graph APIs 🐱‍👤.

## Setup a channel webhook

In order to start, you need to set up a webhook for the channel of where you would like to send messages. So select your channel and then click the 3 dots in the upper right corner; open the `Connectors` menu and search for `Incoming Webhook`. Once found, click on `Configure`:

{{< cloudinary src="v1773855083/tyinbah96btx872ht8ug_dq4rx1.png" alt="Connectors list on Teams" caption="Connectors list on Teams" >}}

Here, set the name, optionally upload an image (it will be the avatar next to the channel posts) and finally click on `Create`.

An URL should be generated: copy and note it, you will need it later on.

## Setup the pipeline

Select your pipeline and a task to run a bash script. Here write down the following code:

```bash
set -x
umask 0002

cat > ./post.json <<endmsg
      {
        "summary": "$System.DefinitionName is being deployed in $Release.EnvironmentName",
        "text": "[$System.DefinitionName]($Release.ReleaseWebUrl) ($BRANCH_NAME) is being deployed in $Build.SourceBranchName by $Release.RequestedFor"
      }
endmsg

curl -X POST -H "Content-Type: application/json" -d @post.json <url-you-copied-from-teams> -v
```

This script is simply creating a json file called `post.json` with some [content supported by Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL). Then, a POST request is sent to the endpoint of the webhook we previously configured. You should now see the message on Teams:

{{< cloudinary src="v1773855118/i6zzgeuft7r3bz5smlj2_yk0g1u.png" alt="The sent message on Teams channel" caption="The sent message on Teams channel" >}}

In the example above, the variables `System.DefinitionName`, `Release.EnvironmentName`, `Release.ReleaseWebUrl`, `Build.SourceBranchName` and `Release.RequestedFor` are automatically provided by Azure DevOps in a release pipeline and they stand for:

- the pipeline name
- the pipeline stage name
- a permanent link to the current pipeline run
- the artifact branch name
- the user who triggered the pipeline

## Conclusions

As shown, sending a message to a Teams channel is really easy and you don't need to integrate Graph APIs unless you care about some advanced features. However, implementing Graph APIs in a pipeline is not a 5 minute job like this one, so find your tradeoff between utility and effort.

This tutorial sends a simple message as an example, but messages are customizable and also interactive as described in [Microsoft documentation](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL).

Even if this procedure is quite straightforward, for me it took few hours since at the time, it wasn't documented at all.

For this reason, I hope this has been helpful.

Happy coding and happy releases! 🐱‍👤
