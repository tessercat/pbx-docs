# PBX extensions

Add a new extensions app
to the Django project.

The app models Extension objects
as Channel and User foreign keys.

The app serves the extension page
only to logged in Users
when it can find an Extension
that references the logged-in User
and the requested Channel.

Extension channel auth realm
is `extensions`.

Add a new client to the node project
served by the extensions app.

The extension client logs in
by being served an Extension channel view,
POSTing a randomly-generated session ID
to the Extension app's sessions endpoint at
`/extensions/<channel-id>/sessions`.

The extensions app
checks CSRF,
so Django settings must make
session cookie storage
available to JavaScript.
Is this a problem?

The session endpoint
gets Extension by Channel ID or raises 404,
checks that the Django session User
matches the Channel's User or raises 404,
returns Session-specific client ID and password.

The client connects
and uses the session's client ID and password
to log in to the verto endpoint.

FreeSWITCH POSTs the verto login event
to the Django `fsapi` config endpoint,
and the app looks for a Channel
matching the auth realm and session ID,
and the extensions app
auth request handler
sends an auth fragment
that lets the client
use call-specific verto protocol commands.

The extensions app's channel page
shows a paginated table
of all extensions available
for routing on the PBX.

How can this work?
Filter by online?
How to update the table
as extensions go on and offline?
Use the verto channel?

What does it mean for an extension to be online?
Probably an extension is online
when it has at least one verto endpoint login
or sofia endpoint SIP registration,
so how is that tracked?
By a field in the app's Extension model?

In any case,
people who log in to the web app
see a searchable table
of available extensions
and they can click a button
to initialize user media,
send `verto.invite` to the switch,
and negotiate a WebRTC leg
with the switch.

The switch POSTs the invitation
to the `fsapi` endpoint.

The extensions app has registered
a handler on verto dialplan events
and the handler returns an XML document
that FreeSWITCH uses
to route the call to some destination.

The lowest hanging fruit
would be to route the call
to the called extension's
registered sofia users
and logged-in verto users,
or to voicemail
if the extension is offline
or doesn't answer.

Add a `sofia` Django app.

The sofia app models Line
as User and Extension foreign keys
and Line-specific registration credentials.

To connect a phone to the PBX,
a User with at least one Extension
logs in and creates a Line,
sets username and password for the Line
and configures the phone to use the credentials.

The phone registers.

Do sofia profiles
allow multiple registration
so that all phones registered
with the same Line credentials
ring when the extension receives a call,
or should this be done with groups?

Does the extensions auth handler
allow multiple logins
from different browsers
for the same verto user,
or does logging in from one browser
log other users out?

To truly make this a PBX,
the sofia app models Gateway,
and adds a request handler
that returns dialplan routing
to send calls from Extensions
that are allowed to make outbound calls
to a Gateway.

How are Extensions
allowed to make outbound calls,
and which gateway is chosen
and how is caller ID set
for outbound calls?

To receive calls from a PSTN gateway,
the extensions app models DID
as a many-to-many field
that associates an inbound caller ID
to a group of Extensions.

When a call comes in for a DID,
the app returns dialplan config
that routes the call so that all
logged-in verto clients
and registered SIP clients
for all Extensions for a DID
ring when the switch receives
a call for that DID.

How to secure
sofia profile
listening ports
for Lines and Gateways?
