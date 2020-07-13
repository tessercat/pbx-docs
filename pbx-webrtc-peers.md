# The PBX WebRTC peers WebSocket protocol

The following
describes the `pbx-webrtc-peers` WebSocket protocol
as implemented by the
[pbx-web](https://github.com/tessercat/pbx-web)
Django project's `extensions.PeerConsumer` class
and the
[pbx-clients](https://github.com/tessercat/pbx-clients)
`peer.js` app.

The protocol consists of
peer, extension and session management requests and responses,
extension and session status event messages
and WebRTC signals.

## Peers

Peers send `init` requests
and the server sends a consumer's
unique peer ID in response
(and current extension state
to avoid an extension state request).
All subsequent requests from the peer
must include the correct peer ID.

To register,
peers send a `register` peer request
with the extension's password,
and the server sends
`registered` or `unregistered` in response.

To de-register,
peers send a `deregister` peer request
and the server sends
`unregistered` in response.

## Extensions

Once initialized,
peers can send extension status requests
and the server sends `ready` or `offline`
in repsonse.

The server sends extension events
asynchronously to unregistered peers
when the number of registered peers
changes from 0 to 1 (`ready`)
or from 1 to 0 (`offline`).

## Sessions

When an un-registered extension
sends a `call` session request,
the server sends `ringing` in response
and sends `ringing` session events
to all registered peers.

Registered peers answer calls
by sending an `answer` session request
that contains the call's unique call ID.
The server sends `connected` or `unavailable` in response,
and it sends a `connected` session event to the caller,
and `claimed` session events to all registered peers.
Answering peers should ignore `claimed` session events
after receiving `connected` in response
to an `answer` request.

When the calling peer
sends an `end` session request
before a registered peers
answers the call,
the server sends `disconnected` in response
and `abandoned` session events
to all registered peers.

Either peer can end an answered session
by sending an `end` session request.
The server sends `disconnected` in response
and a `disconnected` session event
to the other connected peer.

## Signals

The server simply forwards signal messages
between a call's A and B channels.
