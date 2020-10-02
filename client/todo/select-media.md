The client should provide
some simple means
for people to select
audio/video input/output media options.

- Select audio input source
  from the set of all audio input devices.
- Select video input source
  from the set of all video input devices
  and display media sources.
- Select audio output sink
  from the set of all audio output devices,
  if possible.

Add a media selection dialog
that pops up
before initializing local media
so that people can select
initial media sources/sinks.

Since device change listeners
don't seem to work on all browsers,
the dialog should probably
regenerate its option lists
every time it's displayed,
which means keeping state
and presenting
some kind of loading indicator
while it's generating options.

The nav menu should include
a menu item that pops up the media dialog
and lets people change media sources/sinks
after the connection is established.
