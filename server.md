# Channels app

The `channels` app models
channels as
a randomly generated UUID ID
and a set of clients.
Channels can be private or public.

Public channels
are listed on the index page.

A client is modelled
as a randomly generated UUID client ID,
a randomly generated UUID password
and a channel reference.

Per-channel client IDs
are associated with browsers
using Django session cookies.

The app creates Client objects
when it receives channel view requests
that don't include
a (channel-specific) client ID
session variable key.

The channel view
places client ID and password
in responses,
and the JavaScript client reads them
and sends them to FreeSWITCH
via WebSocket
in JSON-RPC login commands.

When it receives the login command,
FreeSWITCH is configured to send auth requests
to the Django project's `/fsapi` endpoint.
The `fsapi` app hands the request
to the `channels` app auth request handler,
and the auth request handler
retrieves the Client object
and returns a FreeSWITCH directory fragment
that contains the client's correct ID and password.
FreeSWITCH uses this fragment
to authenticate the login.

The auth fragment
includes configuration
that allows clients to send messages
only to a single channel.

Clients `verto.subscribe` to the channel UUID on login
and send available presence status
to the channel with `verto.broadcast`.


# Channel/client ID and Django sessions

Django session `session_key` changes
when an anonymous session logs in,
but channel/client ID in session data
is preserved.

Django sessions are destroyed
when a logged-in session logs out,
so channel/client ID changes as well.

Channel/client ID also change
when sessions expire
and a new session is created.


# FreeSWITCH/Django integration

FreeSWITCH
sends requests to the project's `/fsapi` endpoint
via `mod_xml_curl`.
All requests contain POST data
as URL-encoded dicts of key/value pairs.

The `/fsapi` endpoint view
matches POST data
to a request handler object
registered in project settings,
and if it finds one,
responds to the request
using the template and context
returned by the handler's `process` method.

The requests handler registry
is a simple array,
so the first matching handler
processes the request.

The `fsapi` app provides
an abstract request handler class,
a function to add custom request handler objects
to the handler registry,
and auto-discovery of `fsapi` modules
in other apps
using Django's `autodiscover_modules` utility
on app ready.

Request handler classes
initialize a list of keys and values
that must be present in the request's POST data
and implement the `process` method
that returns a template and context
that the `/fsapi` view uses
to generate a response.

The `/fsapi` endpoint view
raises 404 when no request handler
matches the request's POST data.

The app provides
a custom 404 method
that returns
a FreeSWITCH XML fragment.

The `common` app
detects the custom method
and returns its result
instead of the default HTML 404 response.


# Verto peer-to-peer message relay

Clients send messages directly to other clients
with `verto.info` messages
that include the `msg` param.
This sends a verto protocol `MESSAGE` event
that a Lua script detects
and sends directly to the target client
using the FreeSWITCH `chat` API command.

Clients base-64 encode these relayed info messages
before sending and decode upon receipt
to avoid corruption by the `chat` command.
It possible for clients to also encrypt
these info messages
using the Web Crypto API,
but since messages contain only
peer presence and call control data
(unless misused),
they offer no information
beyond what the call server already knows
by hosting the web client.
