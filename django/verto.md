The verto app
registers for verto.conf configuration requests
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
