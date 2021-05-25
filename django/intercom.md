The intercom app models
Extensions as a number and a sofia Intercom.
Numbers are unique per Intercom.

Extensions have a "web-enabled" boolean field,
and the app registers signal handlers
to add/remove an optional, hidden
verto Channel field to/from Extensions on save.

An extension Action
is an XML dialplan template
and an Extension.

Action is a concrete class,
meaning that the Django ORM
creates a table for it,
but it's meant to be subclassed.

Extensions have `action` and `<actionsubclass>` attributes
through an Action's Extension reference.

The intercom app
enumerates Action subclass field names
on app ready,
and the Extension model
provides a `get_action` method
that returns the Extension's
subclassed Action object.

Dialplan handlers
process an action object's dialplan template
with data from subclasssed object's fields.

The intercom app implements
only one Action subclass, the Bridge.

(Eventually I'd like to add
apps/Actions to handle
Conference,
Record,
Playback
and VoiceMail
(and maybe IVR)
diaplan actions/applications,
as well.)

GatewayExtensions are
a combined extension/action
(but subclass neither).

GatewayExtensions combine
a diaplan template,
a regular expression,
and a sofia Gateway.

Intercom Lines are an intercom profile
registration username and password.

Lines reference Bridges
and GatewayExtensions.

OutsideLines have a desination phone number,
and like lines,
they reference Bridges,
but also reference a Gateway.

When the dialplan
receives a call
to an Extension number
with a Bridge action,
the dialplan sim-rings all Lines
and OutsideLines (via their Gateway)
that reference the Bridge.

When the diaplan recieves a call
that doesn't exactly match
a particular Extension,
it matches the destination number
against the calling Line's GatewayExtension expressions,
and if one matches,
calls the expression's `$1` group match
through the GatewayExtension's gateway.

Lines without GatewayExtensions
an call OutsideLines
through Bridge Extensions,
but can't call
through Gateways otherwise.

Lines
GatewayExtensions,
and OutsideLines
all have an OutboundId field
that specifies outbound
caller ID name and number
to send to a Gateway's ITSP.

Line OutboundId is optional
and if present,
applies to calls through
any of a Line's
Bridges or GatewayExtensions.

When a Line without an OutboundId
calls through a GatewayExtension
or to OutsideLines through a Bridge,
the diaplan presents the
GatewayExtension/OutsideLine caller ID
to the ITSP.

When a Line with an OutboundId
calls out through a GatewayExtension,
or when it calls OutsideLines through a Bridge Extension,
the dialplan presents the Line's caller ID
to the ITSP,
overriding the GatewayExtension/OutsideLine's caller ID.

TODO
Implement and explain
intercom and gateway
registration and auth event
Lua scripts.

TODO
Remove Gateway fields
from GatewayExtensions
and OutsideLines
and add a priority field.
Modify the diaplan
so that outbound calls through
GatewayExtensions and OutsideLines (via Bridges)
go through Gateways in priority order.

TODO
Explain verto Client/Channel
auth and dialplan handling.
