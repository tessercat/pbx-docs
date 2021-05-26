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
