# Channel security

Since username/password are easily discovered,
nefarious Internet denizens
can easily re-use
the pub/sub/peer-to-peer signal channel
for whatever purpose they devise,
but hopefully
I've minimized the possibility
of bad-faith actors
interfering with
the channel layer's intended purpose
of providing a medium
to discover other peers
and negotiate peer-to-peer
WebRTC voice and video calls
with them.

I'm sure someone
with a more malevolent imagination
will eventually prove me wrong.


## General overview

Clients must know a valid
per-channel login username (client ID) and password
before logging in to a channel.

Channel ID, client ID and password
are random UUIDs
generated by the Django `channels` app.

Channels are either public or private.

Public channels
are listed on the channels index page
and private channels are not.

Joining a channel
requires nothing more
than knowledge of the channel ID.

Since UUID address space is so large,
and since I assume that Django
generates sufficiently random UUIDs,
I assume that private channels
are essentially inaccessible
without specific knowledge of their UUID.

The Django `channels` app
generates new Client objects
and stores Client IDs
in per-channel session variables
when a browser requests a channel page
for the first time,
when sessions expire,
or when the Django session cookie
is deleted from the browser.

The Django `channels` app
delivers the server-generated Client object
login username (client ID) and password
to the JavaScript client
as values in DOM input elements.

The verto module
requires user configuration
to specify allowed event channels per user,
and the `channels` app
auth request handler
adds only the channel UUID
to the list of allowed event channels,
so logged-in clients
are able to broadcast to a single channel only.


## Session/client ID identity

The verto module
exposes a peer's session ID
to other peers
in broadcast messages.

For peer-to-peer messages,
the verto module
inserts login username (client ID)
into the `MESSAGE` event `from_user` field,
and the target peer receives this value
in the received event's `msg.from` field.

Since both session ID and client ID
are known to other peers,
in this system,
session ID and client ID
are the same value.

The system
also leverages session and client ID identity
to disable the default verto module behaviour
of allowing multiple registration
of clients with the same client ID
but different session IDs.


## Client ID spoofing and XSS injection

I've been as careful as I can be
to ensure that Peer objects
check expected peer ID
against the `from` field of incoming messages
so third parties
can't interfere in peer-to-peer messaging dialogues.

I've also been careful
to avoid using message data
from other peers
directly in peer view components,
so the system should offer
no potential for XSS injection.


## Cross-channel peer-to-peer messages

Peer-to-peer messages
are relayed by the info msg script
via the verto chat protocol,
and since the verto chat protocol
sends messages to any logged-in client
without validating
channel membership,
it's possible for clients
to send messages
to clients in other channels.

Since the address space of UUIDs is so large
and presumably random,
I assume that channels
are effectively isolated from one another
unless a user
has knowledge of
more than one channel ID.

And since the channel system is essentially
a means for clients to discover
each other's client ID,
and since peers can accept, decline or ignore
connection offers,
I assume that if a user knows a channel ID,
they're welcome to offer a connection
to any peer in any known channel
from another channel if they like.

It's certainly possible
for the info msg relay script
to validate channel membership
via the config server's `/fsapi` endpoint,
but this would add significant overhead,
and since I don't see the point,
I haven't done so.


## Disable multiple registration

By default,
the verto module
disconnects (`verto.punt`) a logged-in client
when another client
re-uses the same auth credentials
(client ID username and password)
*and session ID*,
but the module allows new connections
if username and password are correct
but session ID is different.

This allows nefarious actors
to connect any number of clients
once a single client ID and password are known.

Also by default,
the verto module
provides no mechanism to disconnect
logged-in clients.

To prevent misuse
of the default verto module
"multiple registration" scheme,
I've added a custom `verto_punt` API command
to the verto module
and a Lua hook script
that listens for verto logins
and punts clients whose
session ID is not the same as
their client ID.

This at least allows hackers
to connect only a single client
at a time
per login username and password.


## Message relay vulnerability

Peers base-64 encode
all messages relayed
by FreeSWITCH verto endpoints
to avoid corruption
by the FreeSWITCH verto chat app,
but base-64 encoding
is not encryption.

This means that
while peer-to-peer media streams
are "end-to-end encrypted",
the message channel
is encrypted only between peers
and the service host,
and ICE and SDP messages
could be compromised
to insert a "man-in-the-middle"
by anyone with privileged access
to the service host.

Securing messages
relayed by the service host
with end-to-end encryption
is possible,
but would rely on
a third party key-sharing service
trusted by peers,
but which is not available
to the service host.


## Media stream independence

Once a WebRTC connection is established,
media streams are encrypted
and transmitted directly peer-to-peer,
and the service host
has no knowledge of them at all.

Unless message relay
by the service host is compromised,
I don't see how media streams
can be compromised
except at the peer endpoints themselves.