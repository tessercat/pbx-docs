# Peer state

NavMenu items vary
by peer and connection state.

## Offline

The peer's WebSocket client
isn't connected
and the peer isn't trying
to connect it.

Peers enter this state
by failing to log in to the verto endpoint,
by failing to subscribe to channel events,
by failing to publish ready to the channel,
by leaving the channel
via the NavMenu "Leave" item,
or by WebSocket or ping timeout.

Peers move from offline
to offline trying
via the NavMenu "Join" item.

NavMenu options are:
- Join
- Help

## Offline trying

The peer is in the process
of attempting to
connect its WebSocket client,
log in to the verto endpoint,
subscribe to channel events
and publish its ready state.

Peers enter this state
when the index connects them
or via the offline NavMenu "Join" item.

Peers leave this state
and move to the offline state
by connecting the WebSocket,
but failing to log in,
subscribe
or publish ready.

Peers leave this state
and move to the online state
by connecting the WebSocket,
logging in,
subscribing
and publishing ready.

NavMenu options are:
- Leave
- Help

## Online

The peer has logged in,
subscribed to channel events
and published ready,
but its connection isn't active.

The peer moves to this state
from offline-trying
when the peer
connects its WebSocket,
logs in,
subscribes to channel events
and publishes ready.

NavMenu options are:

- Leave
- Help

## Connected offline

The peer's connection is active
but the WebSocket isn't connected.

Peers enter this state
from connected online
when the WebSocket disconnects
for some reason.

Peers leave this state
and move to connected-online
when the WebSocket reconnects
and the peer logs in,
subscribes to channel events
and publishes ready.

Peers move to offline
when they close the connection
before it's able to re-establish
the signal channel

NavMenu options are:
- Close

## Connected online

The peer's connection is active,
the WebSocket is connected,
the client has logged in
subscribed to channel events,
and published ready.

NavMenu options are:
- Close
- Select media
- Share file
- Help

Media selection
is a dialog box
of options lists
to select input/output devices/sources
available to the browser.

File sharing
is peer-to-peer
via data channel.
