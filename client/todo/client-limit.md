Limit the number of clients per channel.

## Channel size

FreeSWITCH
is configured to run Lua hook scripts
for verto client login and disconnect events.

The scripts
use FreeSWITCH's `curl` application
to send requests to the `fsapi` endpoint
for every event,
and the `channels` app
provides request handlers.

The Django `channels` app
somehow limits the number of clients
allowed to log in to the channel,
somehow tracks the number of clients
currently logged in to the channel,
incrementing and decrementing the count
when it receives requests
from the scripts.

Since `mod_verto`
requests module configuration
from the `channels` app
when it starts,
and since no clients are logged in
when the verto module starts,
the `channels` app's
`verto.conf` request handler
should somehow reset
channel client count.

Per-channel client count
should probably be implemented
by adding a `connected` field
to the Client model
and finding the total number of connected clients
for a channel
by querying a count of connected clients
from the connecting client's Channel object
before setting the `connected` state
on the connecting client.

Since Django request handling
wraps database transactions,
changes to channel client counts
should be atomic
however it's implemented.

### Login events

The login script
sends `client_id` and `result=bad_sessid`
to the `fsapi` endpoint
and punts the client
when client ID and session ID don't match.

When client ID and session ID match,
the script sends
`client_id` and `result=success`
to the `fsapi` endpoint.

When it receives `result=success`,
the login request handler
checks if the client's channel is full
and increments client count
and sends an empty response
if the client is allowed to log in,
or responds with `punt`
if the channel is full.

When the script receives `punt` in response,
it punts the client.

### Disconnect events

Disconnect events
for clients that aren't logged in
don't have a client ID header.

The disconnect script
sends `client_id`
to the `fsapi` endpoint,
and if the `client_id` is not `nil`,
the `channels` app
decrements client count
of the client's channel.
