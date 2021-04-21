# Django project notes


## Deployment

Systemd runs the project
in daphne
on a localhost port
in a project-specific venv.

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
from both `{{ hostname }}` and `localhost`.


## Protected paths

To protect localhost-only endpoints
from access via the public port,
the `common` app
provides a registry and middleware
to reject requests to registered `url_name` strings
unless the request was made
via the `localhost` nginx server.

Other apps
implement a `protected_paths.py` module
that registers `url_name` strings.

The `common` app
loads the registry on app ready.

Prometheus is configured
to scrape `/metrics`
from the `localhost` nginx server.

The `common` app
registers the '/metrics' endpoint's `url_name`
in the protected paths registry.

FreeSWITCH `mod_xml_curl` is configured
to send requests to `/fsapi`
on the `localhost` nginx server,

The `fsapi`  app
registers the '/fsapi' endpoint's `url_name`
in the protected paths registry.


## FreeSWITCH request handling

The `fsapi` app provides
the localhost-only `/fsapi` endpoint
to handle requests
sent by FreeSWITCH's `mod_xml_curl` module.

Django CSRF protection
is disabled on the `/fsapi` endpoint.

It's probably possible to configure `mod_xml_curl`
to use cookies
and enable CSRF protection for the endpoint,
but since the endpoint is accessible
only on localhost,
I haven't bothered to do so.

The project handles FreeSWITCH `mod_conf_xml` requests
by processing requests using
app-specific
auto-discovered
registries of request handler objects
defined in project settings.

### 404

The `fsapi` app provides
a custom 404 method
that returns
a FreeSWITCH XML fragment.

The `common` app default 404 handler
detects the custom 404 method
and returns its result
instead of the default HTML 404 response.

The `/fsapi` view
returns this 404 fragment
when a request handler
raises (an unhandled) Django `Http404`.

### Fsapi base

The `fsapi` app
populates a project settings registry
of FsapiHandler objects.

Other apps
implement and register FsapiHandler subclasses
in app-specific `fsapi.py` modules.

The `fsapi` app auto-discovers these modules
and populates its registry on app ready.

The `fsapi` registry is an array,
and handlers are added to the registry
in Django module load order.

FsapiHandler subclasses
define a set of required POST fields
and their expected content
and implement FsapiHandler's abstract `process` method.

The `fsapi` view
matches a request's POST data
against each handler's required field,
and runs the `process` method
of the first handler with matching fields,
or raises 404 if no handler matches.

The `fsapi` view
doesn't catch exceptions.

### Directory and dialplan

The `directory` and `dialplan` apps
add FsapiHandlers that match
*all* directory/diaplan requests
and pass these requests on to
DirectoryHandler/DialplanHandler objects
that other apps register
(in `directory.py` and `dialplan.py` modules.

The `directory` and `dialplan` apps auto-discover app modules
and populate their respective registry on app ready.

Directory/diaplan registries are arrays,
and handlers are added to registries
in Django module load order.

The directory/dialplan FsapiHandlers
return template/context
from the first sub-handler that doesn't raise 404,
but continue to process other sub-handlers
when they encounter 404.

### Verto directory and diaplan

The `verto` app
implements and registers directory/diaplan handler subclasses
(in `directory.py` and `dialplan.py`).

Verto directory/dialplan handlers
retrieve a verto Client object
based on POST data
or raise 404
when no Client object is found.

The `verto` app
provides registries and base classes
for other apps
to implement and register
verto-specific directory/diaplan handlers
(in an app's `verto.py` module).

Verto directory/diaplan registries are dicts,
and `verto` app request handlers
retrieve a sub-handler
(from their directory/dialplan registry)
based on the `realm` field
of the verto Client's Channel object.

Other apps
register their verto directory/diaplan handlers
in the `verto` app's registry dicts
with the Channel-specific realm key.

### Conference directory and dialplan

The `conference` app
implements and registers verto directory/diaplan subclasses
(in `verto.py`)
that return verto-specific
templates and contexts.


## Verto punt API command

I've slightly modified
the verto module
in the FreeSWITCH binaries
used in the project
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
registers handlers for the events.

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
