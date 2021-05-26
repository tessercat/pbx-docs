The intercom app models
Extensions as a number and a sofia Intercom.

Extension numbers are unique per Intercom.

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
enumerates Action subclass names
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
diaplan actions/applications
as well.)

OutboundExtensions are
a combined extension/action
(but subclass neither).

OutboundExtensions combine
a diaplan template
and a regular expression.

Intercom Lines are an intercom profile
registration username and password.

Lines reference Bridges
and OutboundExtensions.

OutboundCallerIds are
a name and a phone number,
that OutboundExtensions and OutsideLines
send 

Like Lines,
OutsideLines reference Bridges.

When the dialplan
receives a call
to an Extension
with a Bridge action,
the dialplan sim-rings
all Lines and OutsideLines
that reference the Bridge.

When the diaplan recieves a call
that doesn't exactly match
a particular Extension number,
it matches the destination number
against the calling Line's OutboundExtension expressions,
and if one matches,
calls the expression's `$1` group match.

Lines without OutboundExtensions
can call OutsideLines
through Bridge Extensions,
but can't call
through Gateways otherwise.

Lines
OutboundExtensions,
and OutsideLines
all have an OutboundCallerId field.

OutboundCallerIds are
a caller ID name and phone number.

Line OutboundCallerId is optional
and if present,
applies to calls through
any of a Line's
Bridges or OutboundExtensions.

When a Line without an OutboundCallerId
calls through an OutboundExtension
or to OutsideLines through a Bridge,
the diaplan presents the
OutboundExtension/OutsideLine caller ID
to the ITSP.

When a Line with an OutboundCallerId
calls out through an OutboundExtension,
or when it calls OutsideLines through a Bridge Extension,
the dialplan presents the Line's caller ID
to the ITSP,
overriding the OutboundExtension/OutsideLine's
default caller ID.

Failover for 
OutboundExtension and OutsideLine calls
is accomplished by
adding one dialstring per Gateway
(joined by the bridge application's `|` operator)
to each outbound leg
in Gateway `priority` field order.

TODO
Implement and explain
intercom and gateway
registration and auth event
Lua scripts.

TODO
Explain verto Client/Channel
auth and dialplan handling.
