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
