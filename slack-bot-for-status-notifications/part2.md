# Sending threaded Slack messages from a bot

Using out Slack bot to send messages from a browser app. Thread grouped messages together really easily TK

## TL;DR

In part one TK (link to original) I wrote about why and how we built a Slack bot. Our bot is used as middleware to send notifications from a browser app to any Slack channel in the companies workspace. The bot exposes endpoint mapped to methods in Slack web-api, in particular `chat.postMessage`. The endpoint is  a POST request that uses block-kit-builder to send richly formatted messages. Each message to Slack returns a `ts` or message id which can be used to send additional messages as a thread of the parent message. We leverage this to group similar notifications and avoid spamming the channels with updates.

Our app is used by drone pilots to upload imagery to our services. We want to be notified when an upload is started; progress updates every 25%; if the upload is paused, resumes or cancelled; and if any images failed.

In this post I'll walk though how we implemented the notification flow. If you're following along I assum that you already have a Slack bot setup and it exposes a `/post-message` endpoint.

---

## Setting up

To send messages to Slack I'll need:  

1. Channel Id
2. Url to the GCP function hosting the bot
3. A way to cache message ids that can survive page refreshes.

You can get a channel Id by right licking the name of you channel, then copy link. Paste it anywhere, the ID are the characters on the end. Something like this: `C21S4MN6C4`.

The url for my Cloud Function is found in the GCP console under the `TRIGGER` tab. It'll look something like: `https://europe-west1-[project-name].cloudfunctions.net/slack-bot`. The name on the end is the same as the name we used when deploying the bot in part 1.

To cache the `ts` key I'm using local storage. Nothing fancy here. Usually there are one or two keys live at anyone point. I use an upload Id to scope the `ts` keys.

## Sending the first message

I've structured my notifications as a series of async function. I use a primary `notifySlack` function which is called by notification type specific functions like, `notifySlackUploadInit` & `notifySlackUploadComplete`. The `notifySlack` uses fetch to POST to the bot with the channel Id and message body. It returns a `ts` key which I'll use later to thread messages.

```javascript
const SLACK_CHANNEL = process.env.UPLOADER_CHANNEL_ID

async function notifySlack(message) {
  try {
    const res = await fetch(`${config.common.adminSlackBotUrl}/post-message`, {
      method: 'POST',
      headers: {
        'Content-type': 'application/json',
        // The bot validates that the client user is logged in.
        Authorization: `Bearer ${getToken()}`,
      },
      body: JSON.stringify({
        channel: SLACK_CHANNEL,
        ...message,
      }),
    });

    const body = await res.json();

    // The `ts` key used to thread later messages
    return body.ts;
  } catch (error) {
    // Handle error, we use Sentry...
  }
}



export async function notifySlackUploadInit(uploadId, .../** args with info about the upload */) {
  try {
    const ts = await notifySlack({
      attachments: [
        {
          color: '#EDE275',
          blocks: // JSON from block-kit-builder
        },
      ],
    });

    // Notifications for the same upload are threaded into the same message with this ts key
    storeUploadNotificationTs(uploadId, ts);
  } catch (error) {
    console.error('SLACK NOTIFY ERROR', error);
  }
}
```

These two functions will send a message to slack that looks like this:

TK (Image of slack message)

[Block Kit Builder](https://api.slack.com/block-kit) is a great tool for designing messages. Copy the JSON and use it directly in the request function.

My `storeUploadNotificationTs` function adds the `ts` key to local storage using the upload Id to scope it in case a user triggers multiple uploads from a different tab.

## Threading messages

To send a message as a thread of another message we'll add the `ts` key we stored in local storage to the `thread_ts` property of the request. This function sends an "Upload Completed" message:

```javascript
export async function notifySlackUploadCompleted(uploadId, size, totalFiles) {
  const ts = getUploadNotificationTs(uploadId);
  try {
    await notifySlack({
      thread_ts: ts,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `${totalFiles} files uploaded successfully`,
          },
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `SIZE: ${size}`,
          },
        },
      ],
    });
    // clean up local storage after upload is complete
    removeUploadNotificationTs(uploadId);
  } catch (error) {
    console.error('SLACK NOTIFY ERROR', error);
  }
}
```

I retrieve the `ts` key from local storage. Send a message to Slack with `thread_ts` property. This will post the complete message in the thread of the initial message looking like this:

TK (Image of thread)

Finally, this is the end of the line for this upload I clean up local storage if the request was successful.

That's it. The hard part is getting a Slack bot setup and working. Then with this endpoint I can send messages to any channel in out workspace from any client app.
