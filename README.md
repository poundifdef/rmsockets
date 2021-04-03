# Websockets + LiveView on the Remarkable2

This document describes how notifications, streaming, and LiveView works via
the Remarkale2 API.

The API has 2 websocket URLs for clients to get realtime data about the tablet. The first has events related to document updates, and also notifies the
subscriber when LiveView has been enabled. The second streams raw "lines file" 
data during a LiveView session.

## Prequisite

This guide assumes that you have a current `Authorization: Bearer` token.
(Described [here](https://github.com/splitbrain/ReMarkableAPI/wiki/Authentication#refreshing-a-token).) That token will look someting like `eyJhbGciOi...`

## Notifications

Here's how to subscribe to the RM's notifications websocket:

1. The `Bearer` token is a [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token). From here, you want to parse the `auth0-profile.UserId` value.

The entire JWT looks something like this:

```
    "UserID": "auth0|5a68dc51cb30df3877a1d7c4",
    "IsSocial": false,
    "Connection": "Username-Password-Authentication",
    "Name": "foo@example.com",

    ...(etc)...
```

If you paste that token into jwt.io, then you can see the full JSON.

The `UserId` should look something like `auth0|5a68dc51cb30df3877a1d7c4`, an we will need this later.

2. Make a GET request to the following URL:

```
https://service-manager-production-dot-remarkable-production.appspot.com/service/json/1/notifications?environment=production&group=auth0%7C5fd432bc2bfc8f006e923bee&apiVer=1
```

In this case, substitute the `group` query param with your own `UserId` This will return a host specific to you, which is the websocket URL for notifications! It will look like:

```
{
    "Status":"OK",
    "Host":"abcd-notifications-production.cloud.remarkable.engineering"
}
```

3. Now you can make a websocket connection to the above Host. You must also add your `Authorization: Bearer ...` header. This is the URL:

**(make sure to substitute the URL from the previous step)**

```
wss://abcd-notifications-production.cloud.remarkable.engineering/notifications/ws/json/1
```

If you install the [websocat](https://github.com/vi/websocat) command line utility, you can view websocket events like so:

```
$ websocat wss://abcd-notifications-production.cloud.remarkable.engineering/notifications/ws/json/1 -H 'Authorization: Bearer eyJhbGciO...'
```

From here, I've seen two types of events come through:

### DocAdded

This event occurs every time a document is synced with the RM cloud.
It tells you the document ID, the name of the file, and some other
data. You could use let to let your client do live updates.

```
{
  "message": {
    "attributes": {
      "auth0UserID": "auth0|5a68dc51cb30df3877a1d7c4",
      "bookmarked": "false",
      "event": "DocAdded",
      "id": "df0acd10-1c1f-4068-ba88-ffe63f16ee8c",
      "parent": "",
      "sourceDeviceDesc": "remarkable",
      "sourceDeviceID": "RM000-000-00000",
      "type": "DocumentType",
      "version": "5",
      "vissibleName": "Quick sheets"
    },
    "messageId": "1234567899012345",
    "message_id": "1234567899012345",
    "publishTime": "2021-01-01T12:52:49.368Z",
    "publish_time": "2021-01-01T12:52:49.368Z"
  },
  "subscription": "projects/remarkable-production/subscriptions/sub-abcd-notifications-production"
}
```

### LivesyncStarted

This event occurs when you turn on LiveView (which they seem to call "LiveSync"
internally) from your RM device! This is how the desktop client knows that
there is an incoming LiveView session.

It looks like a livesync assumes that you are streaming a specific page
as of a specific layer of a specific document.

```
{
  "message": {
    "attributes": {
      "auth0UserID": "auth0|5a68dc51cb30df3877a1d7c4",
      "docLayer": "0",
      "docPage": "8",
      "event": "LivesyncStarted",
      "id": "df0acd10-1c1f-4068-ba88-ffe63f16ee8c"
    },
    "messageId": "1234567899012345",
    "message_id": "1234567899012345",
    "publishTime": "2021-01-01T12:53:23.525Z",
    "publish_time": "2021-01-01T12:53:23.525Z"
  },
  "subscription": "projects/remarkable-production/subscriptions/sub-abcd-notifications-production"
}
```