# The Falsetto Protocol

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
> NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
> "OPTIONAL" in this document are to be interpreted as described in
> [RFC2119](https://tools.ietf.org/html/rfc2119).

## Introduction

The Falsetto Protocol is a chat protocol, inspired by Matrix and Discord. It's fully free and can be implemented by anybody. Servers using the protocol are designed to federate, which allows for multiple self-hosted instances to connect with each other.

The main goal of this project is to create a chat platform that is transparent and easy to implement, while retaining many functions that other software does not have (such as conferences - groups with channels in them).

### Recent changes

- cleared up information about which IDs should be used when making remote requests
- made IDs strings

## Keywords

- User - User of an instance.
- Account - An account on an instance. Only used when mentioning the object type.
- Instance - The instance (running a Falsetto Project compatible server).
- Conference - A group that contains channels.
- Channel - A channel of communication. DMs and group chats are also channels.
- Message - A single message, placed in a text channel.

Additional note: all JSON fields that use "..." as the name are intended to be comments and should be ignored.

### Example keywords

Note: if only one user/instance/conference/action is mentioned in the example, use the full words. Only use these keywords for examples in which multiple objects exist.

- Ux - User. The x is a number assigned to each user in the example.
- Ix - Instance. The x is a number assigned to each instance in the example.
- Cx - Conference. The x is a number assigned to each conference in the example.
- Ox - Object. The x is a number assigned to each object in the example.

## Dates

All dates MUST be stored in the ISO 8601 format. Any dates provided in any values MUST match this format.

## Authentication

TODO

## Objects and IDs

Users, conferences, channels and messages are objects.

Every object MUST have a unique ID, which allows it to be quickly located and accessed. These IDs can be strings or numbers, depending on the implementation. Object IDs MUST NOT overlap. You **cannot** change an object's ID once it's been assigned.

Information about IDs can be accessed through

```url
/api/v1/id/<ID>
```

In the Falsetto Project, the main hierarchy of objects is as follows (from largest/most important to smallest/least important):

- Instance (running a Falsetto Project compatible server) - this MUST have the ID of ``"0"``, even if the server uses non-numerical IDs
- User (somebody who uses the service)
- Role (groups together certain people in a conference)
- Conference (a group containing channels, compare to Discord's servers/guilds)
- Channel (which contains messages, compare to IRC/Matrix channels or channels in a Discord server/guild)
- Attachment (as in, a quote, poll or other kinds of embeds)
- Message

Each object, when retrieved, MUST contain the ``id`` value (set as a string) containing the object's ID, and a ``type`` value set to ``"object"``, for example:

```json
{
 "id": "1",
 "type": "object",
 "...": "...insert any additional values..."
}
```

Since IDs can be either numbers or strings, all IDs MUST be stored as strings.

### Example

An application wants to get information about a user. To do so, it must first find its object ID. Once located, this ID can be used to access information about the user, even if they change their name.

### Retrieving information about IDs

To get information about an object/action based on its ID, you can query the API:

```url
/api/v1/id/<ID>
```

where ``<ID>`` is the ID you want to get information on. This will return a request that will look as follows:

```json
{
 "id": "1",
 "...": "...insert any required values..."
}
```

where 1 is the ID you requested.

In order to prevent abuse, ALL IDS except for 0 MUST NOT be accessible without prior authentication.

## Stashes

In order to make requesting large pools of IDs less request-intensive, multiple IDs can be recieved at once using stashes.

In order to improve general speeds and reduce bandwidth waste, stashed IDs use their own format, which allows them to be sent to the remote server in one request. This format is NOT an object; instead, it uses its own type, named ``stash``. Here is an example stash layout:

```json
{
 "type": "stash",
 "id_list": [ "5", "6" ],
 "5": {
 "id": "5",
 "type": "object",
 "...": "...insert any required information..."
 },
 "6": {
 "id": "6",
 "type": "object",
 "...": "...insert any required information..."
 }
}
```

Stashes are returned as regular JSON objects.

> :warning: Don't send IDs to the remote server that aren't supposed to go to it! It's a waste of bandwidth and a potential security threat, as bad actors may attempt to intercept information that way.

## Federation

Each instance MUST have a public inbox (which can be written to by other instances) and SHOULD have a private outbox (where requests that will be sent to other instances are stashed).

Federation works on the following principle:

Every channel and conference can be subscribed to, which means that the remote instance knows that it has to send data to another instance. To subscribe to a channel or conference, the endpoint

```url
/api/v1/federation/subscribe/<id>
```

can be used. Instances stash content that has to be sent to other instances' inboxes in a private outbox.

The inbox is located at

```url
/api/v1/federation/inbox
```

and, upon being queried with a GET request, it MUST return information about the instance (ID 0).

To send an object to another server, the object's content is simply POSTed to the remote instance's inbox.

When a remote object is saved on the local instance, it is assigned a local ID and turns into a regular local object. The remote object's ID and domain are kept in the ``remote_domain`` and ``remote_id`` values. These values can be used to determine whether the object came from a remote instance.

An example of an ID that was recieved from another instance:

```json
{
 "id": "1",
 "remote_domain": "remote.domain",
 "remote_id": "4",
 "...": "...insert any required values..."
}
```

### Example

U1 wants to join a conference. U1 is on I1, while the conference is on I2. I1 isn't subscribed to the conference yet, so it sends a request to I2 asking to be subscribed to the server. If I2 doesn't block I1 and I2 is up, I1 will subscribe to the conference on I2, thus allowing U1 to communicate with people in the conference.

### Stashing

When the remote server can't be contacted, the local server creates a stash list, which contains the ID ranges that have to be sent to the remote server. It pings the remote server every X minutes (recommended default: 10 minutes) and if it gets a connection, it sends it the stashed requests as a stash. If a server is down for more than 7 days, it is considered closed, until any request is received back.

To ease this process, servers can send a request to another letting them know that they're back online. This will mark the server as "back online" and tell the remote server to send information to the previously-closed server. It is recommended that server software checks for connection dropouts and, once identified, waits for the dropout to end, then send the resync request once the connection is available.

As such, this concept covers two cases:

- one of the servers temporarily drops the connection but is still running
- one of the servers is shut down

#### Server status requests

To check if an instance is available, servers can send a GET request to the instance's inbox. This should return information about the instance. If any information is received, the instance is considered up; otherwise, it's considered down.

### Requesting resources from another instance

Servers must request access to information through the following endpoint on the remote server:

```url
/api/v1/federation/request/...
```

where ``...`` is the same as any API call you would usually make, without the ``/api/v1/`` prefix.

For example, to request an account by name, you would use:

```url
/api/v1/federation/request/account/by-name/U1
```

The remote server will then proceed to send the requested information to the local server's inbox, instead of returning it directly.

### Accessing information about remote instances

When an instance begins to federate with another instance, its ID 0 is copied to

```url
/api/v1/federation/$remotedomain
```

where ``$remotedomain`` is the remote instance's domain.

When queried, this will return the equivalent of the remote server's ID 0 (which contains information about the instance).

``GET $domain/api/v1/federation/$remotedomain``

```json
{
 "id": "0",
 "type": "object",
 "object_type": "instance",
 "...": "...insert instance-specific information here..."
}
```

Assuming an object/action has federated to the instance, it is possible to retrieve its ID on the server it has federated to using

```url
/api/v1/federation/$remotedomain/ID
```

Where ``ID`` is the ID from the remote domain.

This will automatically locate the right ID and print the required information.

``GET $domain/api/v1/federation/$remotedomain/$ID``

```json
{
 "remote_domain": "Domain from which the ID was recieved",
 "remote_id": "ID on remote server (number)",
 "id": "ID on local server",
 "...": "...insert other object/action specific information..."
}
```

When querying objects by ID from the remote server, the remote ID must be used. However, if you do not have the remote ID but only the local ID, you can use the local ID prefixed with ``OURS_``, for example:

```url
/api/v1/federation/$remotedomain/remote-id
/api/v1/federation/$remotedomain/OURS_local-id
```

Where ``remote-id`` is the remote ID and ``local-id`` is the local ID.

## Users/Accounts

Users are stored as Account objects. An account can own messages, direct message channels and conferences. An account can have a custom status, which is a string displayed on their profile when they're online, a bio which is a string displayed on their profile at all times and a short status, which quickly shows the person's status (offline, online, away, do not disturb)

### Blocking

> TODO: Expand on this.

Users can block other users.

### Bots

Bot accounts can be marked as such by setting the ``bot`` bool to ``true`` and the ``bot_owner`` to the bot owner's account ID.

Unlike normal users, bots can be invited to conferences (through the ``/api/v1/accounts/$ID/invite`` endpoint) and cannot be logged into. Servers SHOULD implement a way to remotely modify the bot's account information.

## Conferences

A conference is a group comprising of any amount of text and voice channels. Users can join a conference, and roles can be given to them. These roles can have certain permissions assigned to them.

### Invites

Users can join a conference through an invite. Invites can be created through the ``POST /api/v1/conference/<ID>/invites`` API method.

Invites can be listed through the ``GET /api/v1/conference/<ID>/invites`` API method.

Invites are objects. The invite name SHOULD be changeable. There MUST NOT be multiple invites with the same link on one instance.

### Members

When a user joins a conference and they aren't banned, their ID is added to the conference's member list and they have their conference permission set to 1. Each user can then have additional information assigned to their account:

- their conference-specific nickname;
- the roles they're part of;
- the permissions they have been assigned;
- their ban state (banned or not).

This information can be found by querying ``/api/v1/conference/<ID1>/members/<ID2>``, where ``<ID1>`` is the conference's ID and ``<ID2>`` is the user's ID.

This information is stored in an object with the type "conference_user".

See more information about this object in the List of objects with properties.

#### Banning

A user can be banned from a conference. This means they cannot join or access the conference.

If a user was banned, their ID can still be queried through the API endpoint, but only contains the following information:

- "banned" - bool, true
- "permissions" with the conference bit set to 0

These are set by the server at ban time.

## Channels

Channels are channels of communication that can be nested within conferences or stay standalone as group DMs. There are three types of channels: text channels, media channels (voice and video chat) and direct message channels.

Channels have the ``object_type`` of ``channel``.

### Channel types

#### Text channels

Text channels have the ``channel_type`` of ``text``. They are capable of storing messages.

Text channels MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

#### Media channels

Media channels have the ``channel_type`` of ``media``. They are used for the transport of voice and video. They MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

> TODO: How are audio and video going to be transported? Do some research on this.

#### Direct message channels

Direct message channels can transport both text and media. They have the same API calls as both text and media channels. They have the ``channel_type`` of ``direct_message``.

Direct message channels MUST NOT be attached to a conference. The ``parent_conference`` MUST be ignored when paired with direct message channels.

#### Categories

Categories have the ``channel_type`` of ``category``. They can group text and media channels in a conference.

Categories MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

### Messages

Text channels and direct message channels can store messages. These messages can then be retrieved in two ways: by post date and by ID.

```url
/api/v1/channels/messages/{by-date/by-id}/{date/ID}
```

## Attachments

Attachments are objects that can be attached to a message for additional information. The following attachment types are available:

### Quote

```json
{
 "attachment_type": "quote",
 "quoted_message": "id"
}
```

Adds a quote to a message. Should be displayed before the message. Can be used as a reply function.

### Media

```json
{
 "attachment_type": "media",
 "media_link": "https://link-to-media/image.png"
}
```

Adds media to the message. Should be displayed after the message.

Media is one of the following:

- an image (in which case, it SHOULD be embedded as such)
- a video (in which case, it SHOULD be embedded as such)
- a sound file (in which case, it SHOULD be embedded as such)
- any other file

Any image, video or sound links SHOULD be interpreted as an attachment and embedded accordingly.

## Embeds

Embeds are small boxes which contain text, alongside a side-color and an image. Two types of embeds exist: the side-image embed and the main-image embed.

```txt
__________________________      _________________
| Title           -------|      | Title         |
| Description     --IMG--|      | Description   |
|_________________-------|      | ------------- |
                                | -----img----- |
         TYPE 1                 | ------------- |
                                |_______________|

                                     TYPE 2
```

## Permissions

Permissions are stored in a single string value using numbers, where each number represents a permission set. This value is called a permission map and has the following layout:

``"permissions": "12345"``

- 1 - message
- 2 - channel
- 3 - conference
- 4 - roles
- 5 - user

Permissions can be assigned to a conference, a channel, a role or a user in a conference. The order in which permissions are read and overwritten is as follows (from the least to the most important):

- conference
- channel
- role
- user in a conference

The RECOMMENDED initial permission map for a user in a conference is ``21101``.

Permission maps MUST be stored in strings, to prevent trailing ``0``s from being cut.

### List of permission sets

#### Message

- 0 - cannot read or write
- 1 - can read
- 2 - can read and write
- 3 - can pin or delete other user's messages

#### Channel

- 0 - cannot see the channel
- 1 - can see the channel, but can't read/write from/to it
- 2 - can read and write in the channel
- 3 - can modify channels

#### Conference

- 0 - cannot access the conference (user banned or conference password-locked and no/wrong password provided)
- 1 - can access the conference
- 2 - can modify the conference

#### Role

- 0 - cannot modify or assign roles
- 1 - can modify or assign roles

#### User

- 0 - cannot modify nicknames of, kick or ban users
- 1 - can change their nickname but can't modify other users
- 2 - can modify other users' and their nicknames
- 3 - can modify other users' and their nicknames and kick
- 4 - can modify other users' and their nicknames, kick and ban

## Messages

Messages are single messages in a text channel. They MUST be attached to a text channel, the ID of which is stored in the ``parent_channel`` value.

___

## List of objects with properties

This section contains every object with its required values.

> Note: Value type "bool" means a string that has to be either "true" or "false".

### Instance

``"object_type": "instance"``

| Key             | Value type | Required? | Require authentication?         | Read/write | Federate? | Notes                                                                                 |
|-----------------|------------|-----------|---------------------------------|------------|-----------|---------------------------------------------------------------------------------------|
| address         | string     | yes       | r: no; w: no                    | r          | yes       | Contains the domain name for the instance. Required for federation.  MUST NOT CHANGE. |
| server_software | string     | yes       | r: no; w: no                    | r          | yes       | Contains the name and version of the used server software.                            |
| name            | string     | yes       | r: no; w: yes [instance:modify] | r[w]       | yes       | Contains the name of the server. This can be changed by an user.                      |
| description     | string     | no        | r: no; w: yes [instance:modify] | r[w]       | yes       | Contains the description of the server. This can be changed by an user.               |

### Account

``"object_type": "account"``

| Key             | Value type | Required?    | Require authentication?         | Read/write | Federate? | Notes                                                                       |
|-----------------|------------|--------------|---------------------------------|------------|-----------|-----------------------------------------------------------------------------|
| username        | string     | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Instance-wide username. There MUST NOT be two users with the same username. |
| short_status    | number     | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Short status. 0 - offline, 1 - online, 2 - away, 3 - do not disturb         |
| status          | string     | no           | r: no; w: yes [account:modify]  | rw         | yes       | User status.                                                                |
| bio             | string     | no           | r: no; w: yes [account:modify]  | rw         | yes       | User bio. Hashtags can be taken as profile tags and used in search engines. |
| index           | bool       | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Can the user be indexed in search results? MUST be ``false`` by default.    |
| email           | string     | yes          | r: yes; w: yes [account:modify] | rw         | no        | User email. Used for logging in.                                            |
| bot             | bool       | no           | r: no; w: yes [account:modify]  | rw         | yes       | Is the user a bot? See the Accounts > Bots section.                         |
| bot_owner       | string     | if bot=true  | r: no; w: yes [account:modify]  | rw         | yes       | The bot's owner.                                                            |

#### Currently authenticated account

| Key       | Value type      | Required? | Require authentication?                         | Read/write | Federate? | Notes                                                                                 |
|-----------|-----------------|-----------|-------------------------------------------------|------------|-----------|---------------------------------------------------------------------------------------|
| friends   | list of IDs | no        | r: yes [user needs to be authenticated]         | r          | no        | Contains IDs of the users the account is friends with. This MUST be handled through friend requests, to prevent users from adding people to their friend list without their prior permission. |
| blocklist | list of IDs | no        | r: yes, w: yes [user needs to be authenticated] | rw         | no        | Contains IDs of blocked users. |

### Channel

``"object_type": "channel"``

| Key             | Value type | Required? | Require authentication?             | Read/write | Federate? | Notes                                                                                 |
|-----------------|------------|-----------|-------------------------------------|------------|-----------|---------------------------------------------------------------------------------------|
| channel_type      | string     | yes       | r: yes*; w: yes**                 | r          | yes       | Contains the type of the channel. Can be ``text``, ``media`` or ``direct_message``. |
| parent_conference | string     | if channel-type is ``text`` or ``media`` | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the ID of the conference the channel is in. |
| name              | string     | yes       | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the name of the channel. If the channel is a direct message channel with one participant, the name is set to the IDs of the users in the conversations, separated by a space.|
| description       | string     | no        | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the name and version of the used server software.                            |
| permissions       | string     | yes       | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the permissions for the channel. This is a permission map.                   |

``* Must require prior per-user authentication. Direct message channels require the user to be a part of the direct message. Channels in conferences require the user to join the conference.

** Only writeable when the object is created, not re-writable.``

#### Direct message channels

Beside the regular channel values, direct message channels have the following additional values:

| Key             | Value type  | Required? | Require authentication?                              | Read/write | Federate? | Notes                                                                                       |
|-----------------|-------------|-----------|------------------------------------------------------|------------|-----------|---------------------------------------------------------------------------------------------|
| members         | list of IDs | yes       | r: yes*; w: yes [x3xxx permissions]                  | rw         | yes       | Contains the IDs of the members of the direct message.                                      |
| icon            | string      | yes       | r: yes*; w: yes [x3xxx permissions]                  | rw         | yes       | Contains the icon of the direct message. This is a link. Servers MUST provide placeholders. |

``* Must require prior per-user authentication.``

### Message

``"object_type": "message"``

| Key             | Value type | Required? | Require authentication?                                                 | Read/write | Federate? | Notes                                                                              |
|-----------------|------------|-----------|-------------------------------------------------------------------------|------------|-----------|------------------------------------------------------------------------------------|
| content         | string     | yes       | r: no; w: yes [must be authenticated as the user who wrote the message] | rw         | yes       | Message content. Any further writes are counted as edits.                          |
| attachment      | string     | no        | r: no; w: yes [must be authenticated as the user who wrote the message] | rw         | yes       | ID of the attachment. Any further writes are counted as edits.                     |
| parent_channel  | string     | yes       | r: no                                                                   | r          | yes       | ID of the channel in which the message has been posted. Assigned by the server at message creation. |
| author          | string     | yes       | r: no                                                                   | r          | yes       | ID of the message author. Assigned by the server at message creation.              |
| post_date       | string     | yes       | r: no                                                                   | r          | yes       | Date of message creation. Assigned by the server at message creation.              |
| edit_date       | string     | no        | r: no                                                                   | r          | yes       | Date of last message edit. Assigned by the server at message edit.                 |
| edited          | bool       | yes       | r: no                                                                   | r          | yes       | Is the message edited? Defaults to ``false``. Set by the server at message edit.   |
| reactions       | list of strings | no   | r: no; w: yes [must be able to read the message]                        | rw         | yes       | List of emoji shortcodes.                                                          |

### Conference

``"object_type": "conference"``

| Key           | Value type  | Required? | Require authentication?           | Read/write | Federate? | Notes                                                                                          |
|---------------|-------------|-----------|-----------------------------------|------------|-----------|------------------------------------------------------------------------------------------------|
| name          | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | Name of the conference.                                                                        |
| description   | string      | no        | r: no; w: yes [xx3xx permissions] | rw         | yes       | Description of the conference.                                                                 |
| icon          | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | URL of the conference's icon. Servers MUST provide a placeholder.                              |
| owner         | string      | yes       | r: no; w: yes [user needs to be authenticated and be the owner of the conference] | rw | yes | ID of the conference's owner. MUST be an account. Initially assigned at conference creation by the server. |
| index         | bool        | yes       | r: no; w: yes [user needs to be owner] | rw    | yes       | Should the conference be indexed in search results? SHOULD default to ``false``.               |
| permissions   | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | Conference-wide permission set, stored as a permission map.                                    |
| creation_date | string      | yes       | r: no                             | r          | yes       | Date of the conference's creation. Assigned by the server.                                     |
| channels      | list of IDs | yes       | r: no                             | r          | yes       | List of IDs of channels present in the conference. Assigned by the server at channel creation. |
| members       | list of IDs | yes       | r: yes [user needs to be authenticated and in the conference] | r | yes | List of conference_member objects (people who have joined the conference). Modified by the server when a user joins. |
| roles         | list of IDs | no        | r: yes [xxx1x permissions]        | r          | yes       | List of IDs of roles present in the conference. Modified by the server when a role is added.   |

#### Conference member

``"object_type": "conference_member"``

| Key           | Value type  | Required? | Require authentication?                                           | Read/write | Federate? | Notes                                  |
|---------------|-------------|-----------|-------------------------------------------------------------------|------------|-----------|----------------------------------------|
| user_id       | string      | yes       | r: ues [must be a part of the conference]; w: yes [xxxx2 and up]; | r          | yes       | The user's ID on the local server (when federating, set this to the ID that the user has on your server). |
| nickname      | string      | no        | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | The user's nickname on the conference. |
| roles         | list of IDs | no        | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | Contains the user's roles' ID.         |
| permissions   | string      | yes       | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | The user's permissions, in a permission map. |
| banned        | bool        | no        | r: yes [must be a part of the conference];                        | r          | yes       | Is the user banned? Modified by the server at ban/unban time. |

#### Invite

``"object_type": "invite"``

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                  |
|---------------|------------|-----------|-----------------------------------|------------|-----------|----------------------------------------|
| name          | string     | yes       | r: no, w: yes [xx3xx permissions] | rw         | yes       | Contains the invite's name.            |
| conference_id | string     | yes       | r: no                             | r          | yes       | Contains the ID of the conference the invite leads to. MUST NOT be changeable. SHOULD be verified through the ``/api/v1/conference/invites/$INVITEID`` endpoint. |
| creator       | string     | yes       | r: no                             | r          | yes       | Contains the ID of the account that created the invite. Assigned by the server at invite creation. |

### Role

``"object_type": "role"``

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                                                                           |
|---------------|------------|-----------|-----------------------------------|------------|-----------|-------------------------------------------------------------------------------------------------|
| name          | string     | yes       | r: no; w: yes [xxx1x permissions] | rw         | yes       | Name of the role.                                                                               |
| description   | string     | no        | r: no; w: yes [xxx1x permissions] | rw         | yes       | Short description of the role.                                                                  |
| color         | string     | yes       | r: no; w: yes [xxx1x permissions] | rw         | yes       | Color of the role, in RGB ("R, G, B" (does not support alpha)). Servers MUST provide a default. |
| permissions   | string     | no        | r: no; w: yes [xxx1x permissions] | rw         | yes       | Permissions for the role, as a permission map.                                                  |

### Attachment

``"object_type": "attachment"``

| Key             | Value type      | Required? | Require authentication?                        | Read/write          | Federate? | Notes                                         |
|-----------------|-----------------|-----------|------------------------------------------------|---------------------|-----------|-----------------------------------------------|
| attachment_type | string          | yes       | r: no; w: yes [user needs to be authenticated] | rw (not rewritable) | yes       | Attachment type. See the Attachments section. |

#### Quote

``"attachment_type": "quote"``

| Key            | Value type | Required? | Require authentication?                        | Read/write | Federate? | Notes                     |
|----------------|------------|-----------|------------------------------------------------|------------|-----------|---------------------------|
| quoted_message | string     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | ID of the quoted message. |

#### Media

``"attachment_type": "media"``

| Key        | Value type | Required? | Require authentication?                        | Read/write | Federate? | Notes                      |
|------------|------------|-----------|------------------------------------------------|------------|-----------|----------------------------|
| media_link | string     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | URL of the attached media. |

#### Embed

``"attachment_type": "embed"``

| Key         | Value type | Required? | Require authentication?                        | Read/write | Federate? | Notes                           |
|-------------|------------|-----------|------------------------------------------------|------------|-----------|---------------------------------|
| title       | string     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The title.                      |
| description | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The description.                |
| color       | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The color, in RGB ("R, G, B")   |
| image       | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The link to the embedded image  |
| embed_type  | number     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | 1 or 2 (1 - type 1, 2 - type 2) |

___

## List of REST API methods

This section contains required API methods, alongside a description.

All POST and PATCH methods MUST ignore the ``id`` value if provided with one.

Every method MUST require authentication (user must be logged in), unless specified otherwise.

### GET /api/v1/instance

Returns ID 0 (information about the instance).

See details about the Instance object in the List of objects with properties for more information.

MUST NOT require authentication.

### POST /api/v1/id

Takes an object and adds it to the server. Returns the object. MUST return 400 if the ID exists. Server MUST make sure that no extra variables are being added, and the ``id`` and ``type`` are not being overwritten.

### GET /api/v1/id/$ID

Returns information about the object associated with the ID. If the ID does not exist, it MUST return the 404 status code.

For most IDs, prior authentication is required. See the List of objects with properties for more information about required authentication methods.

Please note that ``/api/v1/id/$ID`` MUST NOT be pushed to or patched.

Replace ``$ID`` with the ID.

MUST NOT require authentication IF ID 0 is being queried.

### PATCH /api/v1/id/$ID

Takes an object's values and applies them to the object with the target ID. Returns the object. MUST return 400 if the ID exists. Server MUST make sure that no extra variables are being added, and the ``id``. ``type`` and ``object_typr`` are not being overwritten.

### GET /api/v1/id/$ID/type

Returns the ``id``, ``type``, ``object_type`` and any other applicable type variables of an object with a certain ID. If the ID does not exist, it MUST return the 404 status code.

This request MUST require authentication unless ID 0 is being queried.

Replace ``$ID`` with the ID.

### GET /api/v1/stash/request

Requires the ``id_list`` value, which is a list of IDs that should be requested. For security reasons, this should be limited to about 100 IDs per request, and be ratelimited. The server MUST respect the usual authentication required to access the IDs.

Data MUST be sent in JSON format and have the ``Content-Type`` of ``application/json``.

```json
{
  "id_list": [ "1", "2", "3" ]
}
```

Returns a stash.

### GET /api/v1/accounts/$ID

Returns information about an account, by ID. If the ID is not an account, it MUST return the 400 status code.

If the ID does not exist, it MUST return the 404 status code.

Replace ``$ID`` with the account's ID.

This returns an Account object. See details about the Account object in the List of objects with properties for more information.

#### Example output

```json
{
 "id": "1",
 "type": "object",
 "object_type": "account",
 "nickname": "example",
 "bio": "Test account",
 "status": "Testing :)",
 "email": "tester@example.com"
}
```

### PATCH /api/v1/accounts/$ID

Modifies information about an account. Replace ``$ID`` with the account's ID.

### POST /api/v1/accounts/$ID/invite

If the account is a bot, add it to a conference. Takes, and requires, a "conference_id" value. The requester must have xx3xx permissions on the conference that they want to add the bot to. Replace ``$ID`` with the bot's ID.

### GET /api/v1/accounts/by-name/$NAME

Returns information about an account, by name. ``$NAME`` MUST be the name of a local user, without the trailing ``@`` and without the ``@domain`` suffix.

This can only be used with accounts registered on the server.

If there is no account with the name, it MUST return the 404 status code.

This returns an Account object. See details about the Account object in the List of objects with properties for more information.

### PATCH /api/v1/accounts/by-name/$NAME

Modifies information about an account, by name. Replace ``$NAME`` with the account's username, without the trailing ``@`` and the ``@domain`` suffix.

### POST /api/v1/accounts

Creates a new Account object. This is not equivalent to creating a new account; see the Authentication section for more information.

### GET /api/v1/accounts

Gets information about the currently authenticated account.

### POST /api/v1/accounts/$ID/block

Blocks the account as the currently authenticated user.

### GET /api/v1/messages/$ID

Returns information about a message, by ID. If the ID does not belong to a message, it MUST return the 400 status code.

If the ID does not exist, it MUST return the 404 status code.

Returns a Message object.

### POST /api/v1/messages

Takes a Message object and posts it to the specified channel (specified in the ``parent_channel`` value) as the 
currently authenticated user. Returns the ``id`` of the resulting message.

### PATCH /api/v1/messages/$ID

Modifies a message. MUST ignore the ``id`` value if a different one is provided. Server-side, this should change the ``edit_date`` variable to the time of edition and set the ``edited`` bool to true.

Replace ``$ID`` with the ID you want to query.

### POST /api/v1/federation/inbox

Federation inbox. See the Federation section.

### GET /api/v1/federation/$remotedomain

Returns information about a remote domain. Replace ``$remotedomain`` with the remote domain you want to get information about. See the Federation section for more information.

```json
{
 "id": "0",
 "type": "object",
 "object_type": "instance",
 "...": "...insert instance-specific information here..."
}
```

### GET /api/v1/federation/$remotedomain/$ID

Returns the contents of the local server's equivalent of the remote server's "$ID". Replace ``$remotedomain`` with the remote server's domain and ``$ID`` with the ID from the remote server. See the Federation section for more information.

```json
{
 "remote_domain": "Domain from which the ID was recieved",
 "remote_id": "ID on remote server (number)",
 "id": "ID on local server",
 "...": "...insert other object/action specific information..."
}
```

### *** /api/v1/federation/request/...

Requests information from a remote domain. MUST be done from a server. Replace ``...`` with the API request you want to make, and ``***`` with GET/POST/PATCH/etc. See the Federation section for more information.

### POST /api/v1/federation/subscribe/$ID

Subscribes to a conference or channel with the given ID. See the Federation section.

### GET /api/v1/conferences/$ID

Returns a Conference object, by ID. If the ID does not belong to a conference, it MUST return the 400 status code.

If the ID does not exist, it MUST return the 404 status code.

Replace ``$ID`` with the conference's ID.

This MUST NOT require authentication.

### POST /api/v1/conferences

Takes a Conference object and creates it. Returns a Conference object.

### GET /api/v1/conferences/$CONFERENCE_ID/members/$ACCOUNT_ID

Returns the conference_member object for the user with the given ID. This method can also take the ID of the conference_member object, as long as it belongs to the specified conference. See the Conferences > Members section for more information.

If the ID does not belong to an account, it MUST return the 404 status code.

If the user does not belong to the conference, it MUST return the 404 status code.

If the ID does not exist, it MUST return the 404 status code.

Replace ``$ID`` with the user's ID.

### PATCH /api/v1/conferences/$ID/members/$ID

Modifies information about a user in a conference.

### POST /api/v1/conferences/$CONFERENCE_ID/members/$ACCOUNT_ID/ban

Bans an user.

### POST /api/v1/conferences/$CONFERENCE_ID/members/$ACCOUNT_ID/kick

Kicks an user.

### POST /api/v1/conferences/$CONFERENCE_ID/invites

Creates an invite using provided data. Returns the invite link.

### GET /api/v1/conferences/$CONFERENCE_ID/invites

Returns a list of dicts named "invites" containing dicts of all available invites.

### POST /api/v1/conferences/$CONFERENCE_ID/channels

Creates a channel. Returns the created Channel object.

### GET /api/v1/conferences/$CONFERENCE_ID/channels

Returns a list of dicts named "channels" containing dicts of all available channels.

### GET /api/v1/channels/$ID

Gets information about a channel, by ID. Returns a Channel object. If the ID does not belong to a channel, this MUST return the 400 status code.

Replace ``$ID`` with the channel's ID.

### PATCH /api/v1/channels/$ID

Modifies information about a channel. Replace ``$ID`` with the channel's ID.

### POST /api/v1/channels/$ID/messages/post

Takes a message object and posts it in the given channel, regardless of what the ``parent_channel`` variable contains.

### GET /api/v1/channels/$ID/messages/by-time/$MINUTES

Get all messages in the specified channel from X minutes ago. Returns Message objects. Replace ``$ID`` with the channel's ID and ``$MINUTES`` with the number of minutes.

#### Example output

```json
{
  "id1": {
	"author_id": "id",
    "content": "content",
	"post_date": "x",
	"edited": "false"
  },
  "id2": {
    "...": "..."
  }
}
```

### GET /api/v1/channels/$ID/messages/by-date/<ISO8601-compliant-date>

Get all messages posted in the specified channel beginning at a certain date. Returns a list of IDs, similarily to ``/api/v1/channel/ID/messages/by-time``. Replace ``<ISO8601-compliant-date>`` with the date. Replace ``$ID`` with the channel's ID.

### GET /api/v1/attachments/$ID

Gets information about an attachment. Replace ``$ID`` with the attachment's ID.

### POST /api/v1/attachments

Creates an attachment. Returns the newly created attachment.

### GET /api/v1/invites/$ID

Returns information about the invite object with the given ID. MUST return a 400 status code if the given ID does not belong to an invite.

### POST /api/v1/invites/$ID/join

Joins the conference specified in the invite as the currently authenticated user.
