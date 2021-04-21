A visibility listener pings the server
when the client's document becomes visible,
but it's not reliable.
The ping timer halts or slows
on sleeping devices,
so when the device wakes up
and the visibility listener doesn't ping,
the client sometimes disconnects
when the timer finally runs
because it's been too long
since its last ping.

I don't know what to do about this.
