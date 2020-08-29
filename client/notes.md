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

The Index class
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
channel ID, client ID and password variables
that controllers can use
to connect and authorize the Client,
as well as
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

The methods that populate
the navbar and modal dialog
with arbitrary HTML elements
couple controllers
to the Picnic CSS library.


# Peer

The Peer class
is a controller
between View and Client classes.

- Handles presence events
  and peer-to-peer messages
  received by the Client class.
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

The Connection's
implementation of perfect negotiation
handles ICE and SDP negotiation.
The offering peer
is the polite peer.

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
the accepting peer
is the impolite peer.

Once the accepting peer
has accepted an offer
and initialized the Connection,
it closes the connection
at any time
by closing the Connection object
and sending `close` to the offering peer.


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
while it's generating and sending
an offer based on its own local stream,
or while the RTCPeerConnection signaling state
is not `stable`.

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

The Client provides methods to
connect and disconnect MyWebSocket
to and from the verto endpoint.

The Client logs in to the verto endpoint on connect,
sends `verto.subscribe` to the channel
on successful login,
and sends its availability to the channel
via `verto.broadcast`
on successful subscription
and on disconnection.

The Client
provides methods for controllers
to publish its available/unavailable/absent presence state
to the channel,
and provides a method
for controllers
to send messages directly to other Clients.

The verto module
disconnects idle WebSockets
after one minute,
so the Client sends `echo` messages periodically
to keep the connection alive.

The Client
manages WebSocket connection state
by reconnecting immediately on WebSocket disconnection,
but backing off five seconds on every retry
until it retries every 30 seconds.


# Client ID, session ID and peer ID

They're all the same thing.

The web app models Client objects
as a server-generated,
per-channel client ID and password.
Both values are UUID fields.

The web app channel template
and the JavaScript client View class
expose client ID as a View object variable,
and it's used by Client objects
as the "login" parameter
in verto login messages.

The verto module
doesn't provide
a peer-to-peer messaging channel,
but it's possible to send messages between peers
addressed by client ID "login" username.

(A FreeSWITCH Lua script
intercepts the `MESSAGE` events
and forwards them to the target peer
using the FreeSWITCH `chat` application.)

The verto module
also requires a UUID `sessid`
in JSON-RPC messages.
A client's `sessid`
is exposed to other clients
in channel broadcast messages.

(The FreeSWITCH verto JavaScript library
generates `sessid` and stores it in `localStorage`
so the library can re-attach channels
when users reload the browser page,
for example.)

Since client ID as login username
must be exposed
so that FreeSWITCH
can route peer-to-peer messages
between clients,
and since `sessid` is exposed to clients
in channel broadcast messages,
I think it makes sense
to use client ID for both.

Peer objects display
and send/receive client IDs
as variable `peerId`
only as a reminder to myself
that Peer objects provide
single simple peer-to-peer
WebRTC connections.


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
