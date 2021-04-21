# Conference

Add pbx-web conference app.

Conference app
provides
verto auth realm
and conference module config
handlers.

Conference admin
creates a Conference.

A Conference has a Channel.
with `conference` auth realm.

Conference module config
returns config
for the simplest possible conference,
maybe a default 3x2 grid
with an optional floor member,
and when the floor is taken,
display only the floor feed,
but leave mute alone.

All members
have the same moderator commands.
Everyone can mute others,
kick others,
take the floor, etc.

Clients join `conf/<channelId>`
and client gets login credentials
from `conf/<channelId>/sessions`
and logs in.

Clients have no pub/sub permissions
to the channel,
and can only send `verto.invite`.

Clients send `verto.invite` immediately
when they log in successfully,
and FreeSWITCH requests
dialplan config from the conference app,
that validates the Channel
and routes the call
to the correct conference extension.

The conference module
mixes media
for all participants
and the client
simply displays the conference media.

Everyone sees exactly the same grid/floor.

The conference client
uses the same media controls dialog
as the peer client
to set their media source/sink
preferences.

How does the client
interact with the grid?
