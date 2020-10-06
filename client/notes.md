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
    and a method to set its content.
  - A menu item pulled right
    and a method to set its content.
- A video element
  scaled horizontally and vertically
  (with the "contain" object fit style)
  with a black background
  and methods to show and hide it.
- An activity watcher on the document
  that hides the navbar when a connection is active
  but no mouse or touch activity is detected,
  and unhides the navbar when
  the mouse is moved or the screen touched.
- Methods to show and hide modal dialogs.
- A modal alert dialog and methods to show and hide it.


# Peer

The Peer class
is a controller
between View,
Client
and Connection
classes.

- Creates a Client object
  to interact with the channel.
- Creates a Connection object
  to manage connection negotiation.
- Creates NavStatus and NavMenu objects
  to manage and display navbar content.
- Creates a NameDialog object
  to manage peer display name.
- Creates a HelpDialog object
  to display basic troubleshooting information.
- Creates a PeersPanel object
  to manage and display known peers.
- Creates an OfferDialog object
  to manage and display a single Connection offer.
- Creates an OffersDialog object
  to manage and display offers received from other peers.

## Presence protocol

The Peer class
implements a simple protocol
to share presence information
between peers.

### Ready

When their client is ready,
peers subscribe to the client's event channel
and publish `ready` to the channel.

### Available

Peers that receive `ready`
add the other peer
to their list of known peers
and send `available`
directly to the newly subscribed peer
(as opposed to publishing to the channel).

Peers register a callback
on client ping events
and,
when they are not connected to another peer,
publish `available`
to the channel
with every ping.

Peers that receive `available`
update their list of known peers
to track the time the peer was last seen.

Peers detect the absence of other peers
by removing peers
from their list of known peers
(in the ping callback)
that have not sent `ready` or `available`
within the past 60 seconds.

### Unavailable

Peers broadcast `unavailable`
to the channel
when they successfully
open (on offer or accept) a connection
to another peer.

Peers that receive `unavailable`
remove the peer from their list of known peers.

### Gone

Peers broadcast `gone`
before they disconnect
from the channel.

Peers that receive `gone`
remove the peer from their list of known peers.

## Connection protocol

The Peer class
implements a simple protocol
to establish peer connections.
One peer offers a connection
and another accepts.

### Offer

To connect to another peer,
a peer initializes local user media
and sends an `offer` message
directly to another peer.

The offering peer
registers a callback on its client
that re-sends `offer` with every ping.

When the offering peer
receives `accept` in response,
it opens its side of the connection.

The offering peer is
perfect negotiation's
polite peer.

### Accept

A peer accepts an offer
by initializing local media,
sending `accept` to the offering peer,
and once local media is ready,
it opens its side of the connection.

The accepting peer is
perfect negotiation's
impolite peer.

### Close

Either peer can close the connection
at any time
by sending `close` to the other peer
and closing its own
initialized or open connection.

### Error

Either peer sends `error`
when it encounters an error condition
while initializing local media
or when ICE fails
when opening the connection.


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

## Sessions

On WebSocket connect,
the client generates a per-channel UUID session ID,
sends the session ID to the service host
in a separate fetch request
and receives client ID and password
login credentials in reply.

On success,
the client stores the session ID
in browser `localStorage`.

The service host
indicates session expiry
by sending 404 in reply
to the auth credential request.

The client clears its current session ID
and tries again once
when it receives 404 in reply.

Except for the first session expiry 404,
the client leaves the channel
if it receives any non-ok status code.

Once session ID, client ID and password are known,
the client logs in to the verto endpoint.
If login is successful,
the client receives a ready event
from the server.

FreeSWITCH notes
describe the details
of session authentication.

## Pub/sub channels

The Client class
provides methods
that allow controllers
to subscibe to and publish events
to a client's channel.

To subscribe to the client's channel,
controllers provide a callback
that subscribes
when the client is ready.

Note that
FreeSWITCH configuration
must authorize a client
to subscribe and publish
to the client's channel.

Disconnecting from the WebSocket
automatically cleans up subscriptions
on the server,
so I haven't bothered to implement
an unsubscribe method.

## Peer-to-peer signal channel

The Client class
provides a method
that allows controllers
to send messages
directly to other peers
based on client ID.

FreeSWITCH notes
describe how this is implemented
on the server side.

## Automatic WebSocket reconnection

The Client
manages WebSocket connection state
by reconnecting immediately
on WebSocket disconnection,
but backing off five seconds,
+/- five seconds,
on every retry
until it retries every 30 seconds,
also +/- five seconds.

## Ping

Browsers time out WebSocket connections
after 60 seconds of inactivity,
so the client pings the server
with a verto protocol echo message
every 45 seconds +/- five seconds.

Controllers can register a callback
on client ping
to execute various operations
periodically.

## Random retry/ping

Randomness in retry and ping periods
is intended to help stagger
the work of handling connection requests
and ping replies
when FreeSWITCH or the verto endpoint
restarts with many connected clients.

## Ping timeout

The client schedules pings with `setTimeout`,
but some browsers
(e.g. mobile browsers on battery power)
slow or halt timers
when the browser or document
doesn't have focus,
or when the phone sleeps.
To preserve this battery-saving behaviour,
the client disconnects and/or stops
the WebSocket from reconnecting
when it detects a slowed timer.

When the WebSocket disconnects,
the client assumes
the browser initiated the disconnection
and stops reconnecting
if the WebSocket connected
or last pinged the WebSocket server
more than 60 seconds ago.

Since receiving messages
prevents the browser
from disconnecting the WebSocket,
to prevent clients on sleeping devices
from draining the battery,
and to prevent sleeping clients
from continually appearing and disappearing
from the channel
(from the point of view of other clients),
the client detects a slowed timer
on every ping
and leaves the channel
if it last pinged the server
more than 60 seconds ago.

Since the ping timer halts or slows
on sleeping devices,
when a device wakes up
before the slowed timer expires,
the client disconnects
when the timer finally runs
because it's been too long
since its last ping.
This can happen some time after
the device wakes up.

To avoid this,
the client registers a visibility listener
on its document
and pings when the document becomes visible.
This is not reliable,
however,
so the client sometimes disconnects
when the timer runs
some time after the device wakes up.

I'm not quite sure
what to do about this.
Maybe timing out on ping
isn't such a good idea.


# Interdependencies between classes

## Client/MyWebSocket dependency injection

Client creates a MyWebSocket object
and registers this-bound callbacks on socket events.

## Peer as controller

Peer creates a Client object,
registers this-bound callbacks on the object's events
and calls the Client object's interface methods
to interact with the channel.

Peer creates a View object.

Peer creates menus, dialogs and panels
and calls View object methods
to add and remove their content from the View.

Peer provides `getFullName` and `getDisplayName` methods
and makes them available
to view objects
that render peer names dynamically.

Peer registers this-bound callbacks on
its view objects
so the Peer can react to various input actions.

Peer registers callbacks on its Connection object
before opening connections.

## NameDialog as peer name validator

NameDialog provides
state methods that Peer uses
to initialize its own name
and to validate the names 
it receives from the channel.

Once initialized,
a NameDialog object's `peerName` property
contains validated data,
and Peer refers to the property
when it needs to display or send
its own name.

## NavStatus and NavMenu state

NavStatus and NavMenu
both provide a `menu` property,
a list of HTML elements to display
in their respective View element
depending on state.

Peer sets nav object state as state changes,
and places current nav object `menu` elements
in the view.

## PeersPanel and OffersDialog 

PeersPanel and OffersDialog
both keep dicts of HTML elements,
indexed by client ID,
that represent other peers in the channel,
and that Peer uses
to populate View elements.

Peer initializes,
updates
and removes
PeersPanel and OffersDialog
peer elements and properties
as it receives messages
from the channel,
and when it detects
peer activity timeout.

## OfferDialog identity and state

Peer initializes its OfferDialog object
with the other peer's client ID and name
when it makes an offer.

Peer sets OfferDialog state
as connection/client state changes.

Peer uses OfferDialog methods
to validate peer names
in messages it receives
from the channel.

## View modal properties and methods

View provides a `modalHeader` method
that returns a view-comptible modal header.

Peer passes headers to dialogs
as constructor args.

Peer sets callbacks on properties and methods
that View looks for in the objects it displays
in the modal dialog area.

- View uses a modal object's `hasModalContent` method
  to determine whether it should restore the modal object
  after dismissing an active alert.

- View checks `hasModalContent` on objects passed to `showModal`
  and shows the modal if the method doesn't exist,
  but if it does, only if it returns true.

- View fills the modal area with the content of
  a modal object's `modalContent` property.
    
- View registers a listener on the document's keydown event
  and calls a displayed modal object's `onModalEscape` method
  when the method exists and the escape key is pressed.

- If one exists,
  View runs a modal object's `onModalVisible` method
  immediately after showing the dialog.
