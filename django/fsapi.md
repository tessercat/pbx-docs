# FreeSWITCH configuration

The Django project
provides the `/fsapi` endpoint
as a target for
FreeSWITCH's `mod_xml_curl`.


## Deployment

Systemd runs the project
in daphne
on a localhost port
(in a project-specific venv).

An nginx server named `{{ hostname }}`
listens on the public TLS port
and sends requests
to the project
via the daphne port.

An nginx server named `localhost`
listens on a loopback port
and sends requests
to localhost-only endpoints
to the project
via the daphne port.

Project settings allow requests
to both `{{ hostname }}` and `localhost`.


## Protected paths

To protect localhost-only endpoints
from access via the public port,
the `common` app
provides a registry and middleware
to reject requests to registered `url_name` strings
unless the request was made
to the `localhost` nginx server.

Other apps
implement a `protected_paths.py` module
that registers `url_name` strings.

The `common` app
loads the registry on app ready.

Prometheus is configured
to scrape `/metrics`
from the `localhost` nginx server.

The `common` app
registers the `/metrics` endpoint's `url_name`
in the protected paths registry.

FreeSWITCH `mod_xml_curl` is configured
to send requests to `/fsapi`
on the `localhost` nginx server,

The `fsapi`  app
registers the `/fsapi` endpoint's `url_name`
in the protected paths registry.

## CSRF

Django CSRF protection
is disabled on the `/fsapi` endpoint.

It's probably possible to configure `mod_xml_curl`
to use cookies
and enable CSRF protection on the endpoint,
but since the endpoint is accessible
only on localhost,
I haven't bothered to do so.


## Request handlers

The project handles requests
by processing them using
app-specific
auto-discovered
registries of request handler objects.

### 404

The `/fsapi` view
returns a FreeSWITCH XML fragment
when handlers raise 404
instead of the default
by adding a custom 404 handler method
to every request
that returns a FreeSWITCH XML fragment.

The default `common` app 404 handler
detects the custom attribute
and returns its result
instead of the default HTML 404 response.

### Fsapi base

The `fsapi` app
declares an FsapiHandler abstract class
and a registry of FsapiHandler objects.

Other apps
implement and register FsapiHandler subclasses
in app-specific `fsapi.py` modules.

The `fsapi` app auto-discovers these modules
and populates its registry on app ready.

The `fsapi` registry is an array,
and apps add handlers to the registry
in Django module load order.

FsapiHandler subclasses
define a set of required POST fields
and their expected content
and implement FsapiHandler's abstract `process` method.

The `fsapi` view
matches a request's POST data
against each handler's required fields,
and runs the `process` method
of the first handler with matching fields,
or raises 404 if no handler matches.

### Configuration, directory and dialplan apps

The `configuration`, `directory` and `dialplan` apps
add FsapiHandlers that match
POST data `section` fields.

All three apps provide
their own registries
that other apps use
to register handlers for
endpoint- and application-specific requests.

Configuration/directory/diaplan registries
are dicts that map a single POST data field
to a registered handler.

The `configuration` app
provides a dict registry
that maps configuration section requests
to registry handlers by `key_value` POST data,
the `directory` app
registers handlers by `domain`,
and the `dialplan` app
registers handlers by `Caller-Context`.

Other apps
implement ConfigurationHandler,
DirectoryHandler
and DialplanHandler classes
(defined in app-specific `registries.py` modules)
and add them to the respective registry
in `configuration.py`,
`directory.py`
and `dialplan.py` modules.

The configuration/directory/dialplan apps
auto-discover app modules
and populate their respective registry on app ready.


## Verto punt API command

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
using the same username/password,
and could possibly be used
to limit the number of connected clients
per channel.


## Verto channel login/disconnect

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
