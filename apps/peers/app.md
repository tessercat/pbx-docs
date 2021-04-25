# Peers client/app

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

The peer client
generates a per-channel UUID session ID,
stores it locally
and fetches client ID and password
from the Django `peers` app
in a separate request.

The Django `peers` app
provides a per-channel `/peers/<channelId>/sessions` endpoint
that receives the client-generated session ID
as a request argument
and retrieves an existing Client object
based on session ID
(if one already exists)
or creates a new one
(if it doesn't),
and returns the Client object's
session-specific login credentials
to the requesting client.

The `peers` app
expires Client objects
(and their sessions)
after 14 days.

Since the `peers` app
places no restriction
on the creation of new login credentials,
pub/sub and peer-to-peer message exchange
could easily be misused
for purposes other than discovering peers
and negotiating WebRTC connections between them.

Verto module user configuration
specifies allowed event channels
and allowed verto protocol message types
per directory user.

Peer client directory users
are configured to allow logged-in clients
to send only the subset of verto protocol messages
(echo, subscribe, publish and info)
that support peer discovery
and WebRTC connection negotiation
with other peers in the same channel.


# Client ID spoofing and XSS injection

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


# Peer-to-peer info messages

FreeSWITCH is configured
to run a hook script
to relay `verto.info` `msg` protocol messages
to any logged-in verto client
without validating channel membership.

Directory config
must allow clients to send info messages.

To send a message to another peer,
the client sends a Base64-encoded JSON object
to the login ID of a logged-in verto user.

The hook script
receives messages as `MESSAGE` events
and sends verto protocol chat messages
to the addressed user,
but from the login ID of the sender
extracted from event headers.

The verto chat app
filters messages to bad recipients
and logged-out clients,
and the hook script
affords sending clients
no opportunity to forge
the `from` address,
so I'm pretty sure it's safe
for peers to trust
that messages are actually from
the peer they say they're from. 


# Cross-channel peer-to-peer messages

The fact that the info msg hook script
sends messages to any logged-in verto client
(without considering channel membership)
means it's possible for clients
(with permission to send info messages)
to send peer-to-peer messages
to clients in other channels.

(It's not possible,
however,
for clients to subscribe to events
broadcast from other channels
nor to broadcast messages
to clients in other channels.)

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


# Disable multiple registration

By default,
the verto module
disconnects (`verto.punt`) a logged-in client
when another client
re-uses the same auth credentials
(client ID username and password)
*and session ID*,
but the module allows new connections
if username and password are correct
but session ID is different or missing.

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
that listens for successful verto logins
and POSTs client ID and session ID
to the `channels` app
(via the Django project's `/fsapi` endpoint)
for inspection.

When the `channels` app
login event request handler
detects that the client's session ID
is not the expected value,
it sends `punt` to the hook script in reply,
and the hook script punts the client.

Since the verto module
allows clients to log in
without supplying a session ID,
I've also modified the verto module
to deny login
to clients that have not supplied a session ID.

This at least allows hackers
to connect only a single client
at a time
per login username and password.

I'm not sure how effective
such a restriction is in practice,
though,
since,
for the `peers` app in any case,
once a channel ID is known,
there's nothing stopping anyone
from requesting any number of
client IDs and passwords
in different session contexts.

One way to limit
resources consumed
by such misuse
would be to restrict
the maximum number of connected clients
per channel,
though filling channels
would also count as a form of misuse,
so I'm not sure if it's worth it.


# Message relay vulnerability

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
and WebRTC ICE and SDP messages
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


# Media stream independence

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


# Protected endpoints

The Django app
exposes `/fsapi` and `/metrics` endpoints
only to requests from `localhost` processes.
All other endpoints are available
to non-localhost clients.

A custom Django middleware
returns the `common` app's 404 page
when it processes requests
for these protected endpoints
from non-localhost clients.