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

The `channels` app
Channel model `max_size` field
limits the number of clients
allowed to log in to the channel,
and the `client_count` field
tracks the number of clients
currently logged in to the channel.

The `channels` app
increments and decrements client count
when it receives requests
from the scripts.

Since Django request handling
wraps database transactions,
changes to Channel object `client count` values
should be atomic.

Since `mod_verto`
requests module configuration
from the `channels` app
when it starts,
and since no clients are logged in
when the verto module starts,
the `channels` app's
`verto.conf` request handler
resets client count
for all Channel objects.

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
