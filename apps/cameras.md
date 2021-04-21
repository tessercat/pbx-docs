# Django cameras app and client

A Camera has
a camera ID,
a Channel with auth realm cameras.
a Client
and a User.

Channel and Camera admin pages
let people
create Channels and Cameras,
assign Cameras to Channels,

To log in,
camera clients must know
the camera ID of a camera
configured on the server
and that has a Channel associated with it.
Cameras that aren't in Channels
can't log in to the verto endpoint.

Camera gets channel ID
by sending its camera ID
to the camera app's cameras endpoint.
The app gets Camera by camera ID,
gets or generates a Client
and sends back the camera's channel ID.

The camera client then logs in
by generating and sending session ID
to the camera app's sessions endpoint
for the channel specified
by the camera ID request.

The app gets Client by session ID
and Camera by client ID,
validates Camera's Channel,
and sends the Client's auth credentials back.

Camera client
joins the channel,
but has no access to the channel.

The camera's owner
must have side-channel access
to the camera
to configure its host
and camera ID.

# Channel op view

Channel view is a list of
camera channel members
and login status.

Camera client joins the channel,
subscribes to channel events,
queries current members
and shows live login status
of camera channel members.


