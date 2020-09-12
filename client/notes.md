# Features

- Reconnects on WebSocket disconnection
  with randomized backoff on retry.
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
  scaled horizontally and vertically
  with a black background
  (using the "contain" object fit style)
  and methods to show and hide it.
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
  stores current modal content
  and optionally restores it
  when the alert is dismissed.

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

When the offering peer receives `close` in response,
the offering peer closes its Connection object.

The offering peer
can rescind its offer
before the other peer accepts
by closing its Connection object
and sending `close` to the other peer.

If the offering peer receives `accept` in response,
the offering peer fully opens the peer connection
by creating an RTCPeerConnection object,
attaching Peer handlers to Connection object events
and adding local media stream tracks
to the RTCPeerConnection object.

The offering peer is
perfect negotiation's
polite peer.

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

The accepting peer
perfect negotiation's
impolite peer.

Either peer can close the connection
at any time
by closing its Connection object
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
by the WebRTC adapter library,
so I'm guessing that
the rollback tweaks
suggested by Jan-Ivar Bruaroey
will be required
for quite a while longer.


# Client/MyWebSocket

The Client and MyWebSocket classes
implement a subset of
the FreeSWITCH verto endpoint
JSON-RPC protocol
and manage its WebSocket transport.

## Session ID

On connect,
the client generates a per-channel UUID session ID,
sends the session ID to the service host
in a separate fetch request
and receives client ID and password
login credentials in reply.

On success,
the client stores the session ID
in browser `localStorage`.

The service host
indicates session expiry.
by sending 404 in reply
to the auth credential request.

When it receives 404 in reply
to its first request,
the client requests new login credentials
by clearing its current session ID
and generating and sending a new one.

The client halts
after the first session request
if it receices any non-ok status code
except 404,
and on any non-ok status code
on a session-renewal request.

## Login and pub/sub

Once session ID, client ID and password are known,
the client logs in to the verto endpoint,
subscribe to channel presence events
on client-ready,
and publishes its presence status
to the channel
on successful subscription
(and on disconnection).

Since they're peer-specific,
all pub/sub methods
should probably move to the Peer class.

## Peer-to-peer signal channel

FreeSWITCH notes
describe how this is implemented
on the server side.

## Automatic WebSocket reconnection

The Client
manages WebSocket connection state
by reconnecting immediately
on WebSocket disconnection,
but backing off five seconds,
+/- two seconds,
on every retry
until it retries every 30 seconds,
also +/- two seconds.

Randomness in the backoff period
is intended
to help stagger the work of handling
connection requests and ping replies
when FreeSWITCH or the verto endpoint
restarts with many connected clients.

## WebSocket timeout

All browsers I've tested
disconnect idle WebSockets
after one minute,
so the Client sends `echo` messages periodically
to keep the connection alive.

Some browsers slow or halt timers
when a page is in a background tab,
when the browser runs in the background,
or when the device sleeps,
so the client might fail to send an echo message
before the browser times out
and closes its WebSocket.

I've noticed this only
in Chromium-based browsers on Android,
but I haven't noticed it otherwise,
so I assume it's a feature
intended to preserve battery life.
Clients could echo
in Web Workers,
which don't slow/halt timers,
but since I assume
that browsers do what they do
for a reason,
I haven't done so.

In fact,
since the client
reconnects WebSockets on disconnect,
but timer slowdown stops the echoes
that keep it connected,
I've encouraged this behaviour
by halting WebSocket reconnect
in situations where the client
has failed to send an echo message
before the browser
disconnects its WebSocket.

The only mitigation I've added
is to send echo immediately 
when the window gains focus
to avoid timeouts
in situations where
the phone has gone to sleep
for a little while
but the person using it
wakes it up
before the WebSocket disconnects
because they're expecting
or about to make
a call.
