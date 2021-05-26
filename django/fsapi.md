Django CSRF protection
is disabled on the `/fsapi` endpoint.

It's probably possible to configure `mod_xml_curl`
to use cookies
and enable CSRF protection on the endpoint,
but since the endpoint is accessible
only on localhost,
I haven't bothered to do so.

The project handles xml-curl requests
from FreeSWITCH
by processing them using
app-specific
auto-discovered
registries of request handler objects.

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
raises 404 if a matching handler raises 404,
or raises 404 if no handler matches.

The `configuration`, `directory` and `dialplan` apps
add FsapiHandlers that match
POST data `section` fields.

All three apps provide
their own registries
that other apps use
to register handlers for
endpoint and application specific requests.

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
and add them to the respective registry
in `configuration.py`,
`directory.py`
and `dialplan.py` modules.

The configuration/directory/dialplan apps
auto-discover app modules
and populate their respective registry on app ready.
