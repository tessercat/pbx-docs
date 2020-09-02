# FreeSWITCH

I'm a cautious FreeSWITCH fanboy.
FreeSWITCH is messy and complicated,
it has a steep learning curve,
and it's not well documented outside of code,
but Anthony Minessale is a genius,
and he's built the most
flexible and robust
open source
telecom platform
I know of.

FreeSWITCH isn't perfect,
but it's like any tech that lasts.
Once you understand
a few basic systems,
you can do practically anything with it,
and in ten different ways.


# Deployment

As in many old-school projects,
the FreeSWITCH project
used to expect developers
to compile binaries
from source code themselves,
and the default make configuration
installed binaries and configuration
in `/usr/local/freeswitch`.

The FreeSWITCH project
now hosts a Debian apt repo
and recommends its use in production,
and I'm sure it's great,
but I've been compiling by own instead
for no better reason than I like doing it.

I also like the flexibility of
keeping a really minimal module configuration
in the same git repo as binaries,
and relying on git and its many transports
to sync the repo to the file system
reliably.

I also like putting things
in self-contained subdirectories
of `/opt`.

The minimal configuration
relies heavily on the FreeSWITCH XML curl module
to fetch dynamic configuration
from the call control server.
There's no local directory or dialplan config,
or any config of endpoint modules
like `mod_sofia` for SIP
or `mod_verto` for FreeSWITCH's JSON-RPC protocol.
All such configuration
is delivered to FreeSWITCH
by the Django
[pbx-web](https://github.com/tessercat/pbx-web)
project `fsapi` app
and registered request handlers.

I tried Dockerizing FreeSWITCH
a few years ago,
but some of what I discovered about Docker networking
while doing so
warned me against
running a real time telecom service
in a Docker container.

I also prefer
to think of single hosts
as units of deployment.
Most managed resources,
from CPU to certs,
are host-based,
and adding a layer of abstraction
like Docker
to deployment
seems like a lot of overhead
for a small-scale,
single-developer,
single-host
project like this.

Since the FreeSWITCH project
has chosen Debian 10 for me,
it seems simpler
to use Debian tools
like apt and systemd
to manage host resources
instead.

Note that
to provide many of its essential services,
the FreeSWITCH module system
wraps shared libraries,
and this ties compiled binaries
to the operating system
it's compiled on.


# Verto

I looked at the verto module
in early 2019,
but the JavaScript client
didn't look well maintained,
and I wasn't yet willing
to wade very far into
the complexity of the modern front-end
web development experience.

So in the meantime,
I've been playing with other
open source,
self-hostable signal layers
for the project
(SIP.js with `mod_sofia` and `mod_valet_parking`,
Pushpin, nginx nchan and Django channels with Django
for peer-to-peer WebRTC),
but I ended up
wishing that I'd looked more closely
at verto from the start.

The JavaScript might be ugly,
but I've spent some time in the module itself,
and it's a lot more like the rest of FreeSWITCH.
It's messy and complicated,
but the module
essentially exposes
the FreeSWITCH event and channel systems
via a sensible,
secure,
reliable
and configurable
JSON-RPC protocol.

The protocol is tailored
for clients to negotiate
WebRTC media streams
with FreeSWITCH itself,
but the protocol also
proves very capable
as a pub/sub signal layer
to relay presence
and WebRTC call control signals
between peers.

For now,
that's how I've used it here,
but eventually I'd like to
integrate the FreeSWITCH conference module
into the existing
peer-to-peer only
system.


# Event subscriptions

Verto clients can subscribe to FreeSWITCH events
by setting `enable-fs-events` to `true`
in verto module config (default is `false`)
and adding `FSevent` (to receive all events),
or a specific eventChannel string
such as `FSevent.custom::verto::login` (for example)
to `jsonrpc-allowed-event-channels`
in a user's directory entry.

Once this is in place,
subscribing to login events (for example)
on the client side
is as simple as:

    client.subscribe('FSevent.custom::verto::login');


# Module config

At boot,
modules in `modules/modules.conf.xml`
are loaded in the order listed,
with the exceptions noted below.

Once `mod_xml_curl` loads,
the switch starts making requests
to the curl gateway URL
to configure modules that load after `mod_xml_curl`.

Once other listed modules have loaded,
the switch loads configuration for
`post_load_modules.conf`,
`acl.conf`,
`event_socket.conf`
and `post_load_switch.conf`,
and in that order.

Since they load last
and after `mod_xml_curl`,
the switch always makes curl requests
for these files at boot.

Once the initial at-boot configuration process is complete,
since `mod_xml_curl` is loaded,
the switch requests configuration files
for all reloaded modules.

Note that `mod_event_socket`
must be listed in `modules/modules.conf.xml`
or it doesn't load at all,
but `mod_acl` loads without being listed.

According to `mod_xml_curl` docs,
the content of `post_load_switch.conf.xml`
should be an exact copy of `switch.conf.xml`
except for the configuration section name,
which should be `post_load_switch.conf`.
I can't figure out why the switch does this.
