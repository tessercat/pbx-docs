Sofia profiles
are defined in
dict-literal `var/sofia.py`.

The dict-literal
defines intercom
slugs and ports
and gateway slugs,
ports,
priorities
and registration data.

Intercoms and gateways
are Django ORM objects,
but they're not dynamic.

The sofia app's
`importprofiles` management command
reads the dict-literal configuration files
and creates `sofia.Intercom` and `sofia.Gateway`
model objects
based on the data.

The sofia app's admin module
sets intercom and gateway objects
as read-only to all users.

The sofia app
registers for sofia module configuration requests.

Sofia intercom profiles
require authentication
to process INVITEs
and force the registration domain
and dialplan context
to the same value as intercom slug/domain.

Gateway profiles
don't require authentication
to process INVITES,
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
