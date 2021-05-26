The verto app
registers for verto module configuration requests
and creates
non-TLS
websocket-only
IPv4 and IPv6 mod_verto profiles
on loopback interfaces
on the project settings verto port.

Nginx is configured to upgrade and forward
all requests to the `verto` endpoint
to one of the mod_verto profile ports.

Websockets are encrypted
via Let's Encrypt TLS
between browser and nginx,
but unencrypted between nginx and profile ports.

Verto profile domain and dialplan context
are both `verto`.

TODO
Explain relationships between
Channel and Client
including intercom Client
auth and session management.

I've slightly modified
the verto module
in the FreeSWITCH binaries
to require a non-empty session ID on login
and to provide a `verto_punt` API command
that can be used
to punt logged-in connections
by session ID.

I raised a
[GitHub issue](https://github.com/signalwire/freeswitch/issues/832)
to discuss and include the changes
in the official release,
and it seemed welcome,
but since I'm compiling anyhow,
I don't mind adding the changes
by hand when I build.

The project
uses the `verto_punt` command
to deny login of multiple clients
using the same username/password.

The FreeSWITCH repo's Lua module 
is configured to run hook scripts
on verto login and disconnect events.

The hook scripts
use FreeSWITCH's `curl` API command
to POST requests to the Django app's `/fsapi` endpoint
on successful client login events
and on all client disconnect events.

The `channels` app
registers FsapiHandler objects for the events.

The client login handler
queries a Client object
based on POSTed login username
and sends `punt` to the hook script
when the login event's session ID
is not the expected value,
or updates the Client's connected timestamp
when it is.

When the hook script receives `punt` in response,
it calls the `verto_punt` API command
to terminate the login.

The client disconnect handler
queries a Client object
based on POSTed event data login username
and removes the object's connected timestamp
when it exists.

(Disconnect events
may or may not
be associated with a logged-in client,
and the request handler sees `nil`
when a client disconnects
before successful login.)
