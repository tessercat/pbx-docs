# Features

- Reconnects on WebSocket disconnection with backoff on retry.
- Logs in on WebSocket connection.
- Subscribes to receive
  the channel's presence events
  on client ready.
- Publishes presence status
  to the event channel.
- Negotiates peer-to-peer media connections
  with other clients 
  via the verto endpoint.


# Index

The index
is the webpack entry point
and simply imports
[`webrtc-adapter`](https://www.npmjs.com/package/webrtc-adapter "WebRTC adapter on npm")
and creates and connects a Peer object
on window load
and disconnects it
on window unload.


# View

The View class
provides an interface
for controllers to interact
with the features of
the Django `channels` app
`channel_detail.html` template.

View provides
components defined by the
[Picnic CSS](https://picnicss.com/)
library.

- A navbar with three items.
  - A brand logo that links to the index page.
  - A status menu item pulled left next to the brand logo
    and a method to set its text content.
  - A menu item pulled right
    and a method to fill the menu item
    with arbitrary HTML elements.
- A video element
  that fills the document window
  when no connection is active
  (using the "cover" object fit style),
  but is scaled horizontally and vertically
  with a black background
  (using the "contain" object fit style)
  when a connection is active.
- An activity watcher on the document
  that hides the navbar when a connection is active
  but no mouse or touch activity is detected,
  and unhides the navbar when
  the mouse is moved or the screen touched.
- A modal dialog
  and methods
  to control its visibility
  and to fill the dialog
  with arbitrary HTML elements.
- A modal alert dialog
  that's dismissable only by its close button
  and a method to set the alert message
  and show the alert dialog.
  The show method
  stores the content of the current nav menu item
  and the close button restores its previous content.

The methods that populate
the navbar and modal dialog
with arbitrary HTML elements
couple controllers
to the Picnic CSS library.


# Peer

The Peer class
is a controller
between View and Client classes.

- Handles websocket connect/disconnect events,
  channel presence events,
  and peer-to-peer messages
  received by the Client class
  via the websocket.
- Interacts with the view
  based on input from the Client
  and sets method callbacks on the View
  that handle "offer" and "accept" user input.
- Manages a single Connection object
  based on Client and View input.

The Peer class
implements a simple protocol
to establish a peer connection.
One peer offers a connection
and another accepts.

## Offer

To start a new peer connection,
peers create a Connection object,
initialize local user media
(by enumerating and getting local media),
and send an `offer` message
directly to another peer.

If the offering peer receives `close` in response,
the offering peer closes the Connection object.

The offering peer
can rescind its offer
by closing its Connection object
and sending `close` to the other peer.

If the offering peer receives `accept` in response,
the offering peer fully opens the peer connection
by creating an RTCPeerConnection object,
attaching Peer handlers to Connection object events
and adding local media stream tracks
to the RTCPeerConnection object.

In this implementation of perfect negotiation,
the offering peer is the polite peer.

## Accept

When a peer receives an `offer` message,
it can ignore the offer
or send `close`
to refuse the offer.

The peer accepts an offer
by creating a Connection object,
initializing local media,
sending `accept` to the offering peer
once local media is ready,
and opening the connection.

In perfect negotiation,
the accepting peer is the impolite peer.

Once the accepting peer
has accepted an offer
and initialized the Connection,
either peer can close the connection
at any time
by closing the Connection object
and sending `close` to the other peer.


# Connection

The Connection class
provides an interface for controllers to
manage user media,
set `RTCPeerConnection` handlers
and handle ICE candidate and SDP events
to implement so-called "perfect negotiation".

The Connection class
implements perfect negotiation
as per the
[W3C spec example](https://w3c.github.io/webrtc-pc/#perfect-negotiation-example)
and modified as per this
[blog post](https://blog.mozilla.org/webrtc/perfect-negotiation-in-webrtc/)
and this
[Stack Overflow answer](https://stackoverflow.com/questions/61956693/webrtc-perfect-negotiation-issues)
from Jan-Ivar Bruaroey.

In perfect negotiation,
an SDP offer collision occurs
when a peer receives an SDP offer
while the local RTCPeerConnection signaling state
is not `stable`
or while the peer is generating and sending
an SDP offer based on the local media stream,

When collisions occur,
the impolite peer
ignores the incoming offer
and the polite peer
rolls back the offer it's
currently generating,
accepts the incoming offer,
and generates and sends a new offer.

As of August, 2020,
rollback in spec and browsers
is not aligned,
and not polyfilled
by the WebRTC adapter library.


# Client/MyWebSocket

The Client and MyWebSocket classes
implement a subset of
the FreeSWITCH verto endpoint
JSON-RPC protocol.

The client
provides methods
to connect and disconnect MyWebSocket
to and from the verto endpoint.

On connect,
the client generates a per-channel UUID session ID,
sends the session ID to the service host
in a separate fetch request
and receives client ID and password
login credentials in reply.

On success,
the client stores the session ID
in browser `localStorage`
and sets a two week expiry timestamp
on the session ID.

The service host
replies to the auth credential request
with 404 to indicate session expiry,
and the client clears its current session ID
and requests a new one
in response.

he service host
replies with 403 on any other error
and the client stops connecting.

Once session ID, client ID and password are known,
the client logs in to the verto endpoint,
sends `verto.subscribe` to the channel
on successful login,
and sends its availability to the channel
via `verto.broadcast`
on successful subscription
(and on disconnection).

The client
provides methods for controllers
to publish its available/unavailable/absent presence state
to the channel,
and provides a method
for controllers
to send messages directly to other Clients.

Since they're peer-specific,
all pub/sub methods
should probably move to the Peer class.

Some browsers disconnect idle WebSockets
after one minute,
so the Client sends `echo` messages periodically
to keep the connection alive.

The Client
manages WebSocket connection state
by reconnecting immediately on WebSocket disconnection,
but backing off five seconds on every retry
until it retries every 30 seconds.


# Peer ID

It's the client ID.

I've used variable name `peerId` in the Peer class
only as a reminder to myself
that Peer objects support
only one peer-to-peer WebRTC connection
at a time.


# Signal channel

The peer-to-peer signal channel
is implemented using the verto endpoint
`verto.info` `msg` function.

Clients base64-encode message bodies before sending
and decode upon receipt
so that the FreeSWITCH `chat` application
doesn't change them en route.

The verto `msg` function
generates a `MESSAGE` event
with `proto` field `verto`
and `dest_proto` field `GLOBAL`
and sets the `from_user` field
to the sender's login username client ID.

FreeSWITCH is configured
to detect `MESSAGE` events
(via a simple Lua script)
and delivers the messages
to the correct peer
by executing a `chat` application
API command
in the verto protocol
that sends the `MESSAGE` body
directly to the target verto user.
