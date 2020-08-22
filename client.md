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
is the WebPack entry point
and simply imports `webrtc-adapter` 
and creates and connects a Peer object
on window load
and disconnects it
on window unload.


# View

The View class
provides an interface
for the client to interact
with the features of
the Django `channels` app
`channel_detail.html` template.

The view provides
channel ID, client ID and password variables
that controllers use
to connect the Client,
as well as
components defined by the
[Picnic CSS](https://picnicss.com/)
library.

- A navbar with three items.
  - A brand logo that links to the index page.
  - A status method pulled left next to the brand logo
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
(like the Peer)
to the Picnic CSS library,
but it's not too painful,
and I don't see a way to avoid it.


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
  that handle offer and accept use input.
- Manages a single Connection object
  based on Client and View input.

The Peer class
implements a simple protocol
to establish a peer connection.

One peer offers to a connection
and another accepts.

## Offer

To start a new peer connection,
peers create a Connection object,
initialize the object's local media stream
(by enumerating and getting local media),
and send an `offer` message
directly to another peer.

If the offering peer receives `close` in response,
the offering peer closes the Connection object.

If the offering peer receives `accept` in response,
the offering peer fully initializes the peer connection
by creating an RTCPeerConnection object,
attaching the peers's handlers to the object's events
and adding local media stream tracks
to the RTCPeerConnection object.

The Connection's
implementation of perfect negotiation
handles ICE and SDP negotiation.
The offering peer
is the polite peer.

Once the offering peer
has created and initialized
a Connection object
it rescinds its offer
and/or closes an existing peer connection
by closing its Connection object
and sending `close` to the other peer.

Closing a Connection object
removes the local media stream from the view's srcObject,
closes local media stream tracks,
and closes the RTCPeerConnection.

## Accept

When a peer receives an `offer` message,
it can ignore the offer
or send `close`
to refuse the offer.

The peer accepts an offer
by creating a Connection object,
initializing the object's local media,
sending `accept` to the offering peer
once local media is ready,
and immediately initializing the peer connection.

The Connection's
implementation of perfect negotiation
handles ICE and SDP negotiation.
The accepting peer
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
as per the spec example at
https://w3c.github.io/webrtc-pc/#perfect-negotiation-example
and modified as per the blog post at
https://blog.mozilla.org/webrtc/perfect-negotiation-in-webrtc/
and as per the Stack Overflow answer at
https://stackoverflow.com/questions/61956693/webrtc-perfect-negotiation-issues.

In perfect negotiation,
an SDP offer collision occurs
when a peer receives an SDP offer
while it's generating and sending
an offer based on its own local stream,
or when the RTCPeerConnection signaling state
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
by the adapter.


# Client and MyWebSocket

The Client and MyWebSocket classes
implement a subset of
the FreeSWITCH verto endpoint
JSON-RPC protocol.

TODO

## Signal channel

The peer-to-peer signal channel
is implemented using the verto endpoint
`verto.info` `msg` function.

Clients base64-encode message bodies before sending
and decode upon receipt
so that the FreeSWITCH chat command
doesn't change them en route.

The `msg` function
generates a `MESSAGE` event
with `proto` field `verto`
and `dest_proto` field `GLOBAL`.

FreeSWITCH is configured
to detect `MESSAGE` events
(via a simple Lua script)
and delivers the messages
to the correct peer
by executing an API command
that sends the `MESSAGE` body
directly to the target verto user.

## Channel security

Clients must know a valid client ID and password
for a specific channel
before logging in to the channel.

The verto module
requires user configuration
to specify allowed event channels,
and since the auth request handler
adds only the channel UUID
to allowed event channels,
logged-in clients
are able to send messages
only to clients in a single channel.

The verto module
disconnects logged-in clients
when other clients
attempt to re-use the same auth credentials.

The verto module
provides no mechanism to disconnect clients,
so it's not currently possible
to "kick" clients from a channel.

Once a WebRTC connection is established,
all sensitive information
is transmitted directly peer-to-peer,
and the call server
has no knowledge of it at all.
