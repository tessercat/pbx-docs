Limit the number of clients per channel.

Add a size field
to the Channel model.

Add a check to the `channels` app
login event handler
that queries the count of connected clients
for the newly logged-in client's channel
and punts the logged-in client
if the channel is already full.
