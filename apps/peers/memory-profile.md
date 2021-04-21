I should spend more time
with the memory profiler
because I'm no expert.

I'm fairly sure
there are no memory leaks.

I'm mostly concerned about
hanging references to requests,
request callbacks
and incoming events and messages.

I've searched heap snapshots
for VertoRequest and RequestCallbacks,
and there don't appear to be any.

The Client class
parses WebSocket message data
into objects,
so they're difficult to see
in heap snapshots.

To test,
I wrapped message data objects
in a class
to make them more visible
in heap snapshots,
and I don't see any.

I'm also not totally sure that
the recursive ping setTimout
doesn't keep growing the stack
because I've missed a reference somewhere,
but I'm not sure how to test it.
