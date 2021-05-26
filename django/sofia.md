Sofia profiles
are defined in
dict-literal `var/sofia.py`.

```
{
    'INTERCOMS': {
        '<slug>': <port>,
    },
    'GATEWAYS': {
        '<slug>': {
            'port': <port>,
            'proxy': '<primary-server>.voip.ms',
            'realm': 'voip.ms',
            'username': '<username>',
            'password': '<password>',
            'priority': 1,
            'allow_list': [
                '<primary-server-ip>',
            ]
        },
        '<slug>': {
            'port': <port>,
            'proxy': '<backup-server>.voip.ms',
            'realm': 'voip.ms',
            'username': '<username>',
            'password': '<password>',
            'priority': 2,
            'allow_list': [
                '<backup-server-ip>',
            ]
        }
    },
    },
}
```

Intercoms and gateways
are managed as fixtures
by the `importprofiles` management command.

The command logs to console
when a related service
(web app,
iptables
or mod_sofia)
is affected.

The command reads the `sofia.py` dict-literal configuration file
and creates, updates and deletes
`sofia.Intercom` and `sofia.Gateway`
model objects
based on the data.

The sofia app's admin module
sets intercom and gateway objects
as read-only to all users
and hides profile passwords,
but not other profile data.

The sofia app
registers for sofia module configuration requests.

Intercom profiles
authenticate inbound SIP
using Line credentials,
but otherwise,
they're open to any connection.

If attacked,
add ACL addresses to Lines
or implement flooding protection
that hooks in to
FreeSWITCH auth failure events
via Lua script
and reports bad IPs
to the firewall API
via xml_curl.

ACL addresses on Lines
sounds simpler,
more secure,
and unregistered lines
could alert admin by email.

Intercom profiles force
registration domain
and dialplan context
to the same value as intercom slug/domain.

Gateway profiles
don't authenticate inbound SIP
but firewall configuration
allows access to gateway profiles
only from a set of
configured source addresses.

Gateway dialplan context
is the same value as gateway slug/domain.

The sofia app
opens ACL firewall ports
by making use of the firewall API
on app ready.

Gateways are registration-only,
and they're designed to work with
[voip.ms](https://voip.ms) only.

Intercom and gateway profiles
are configured to require
TLS and inbound SRTP.

Channel variables 
set in the dialplan
require outbound SRTP.
