# Login username and session ID constraint

A Lua hook script
listens for login events,
and punts sessions
when login username and session ID
are not identical.

This prevents hackers
from connecting multiple clients
with the same username and password,
but different session IDs.


# Custom FreeSWITCH API verto_punt command

```
SWITCH_ADD_API(api_interface, "verto_punt", "Punt a verto session", verto_punt_function, "sessid");

SWITCH_STANDARD_API(verto_punt_function)
{
	char *sessid = (char *) cmd;
	jsock_t *jsock;

	switch_mutex_lock(verto_globals.jsock_mutex);

	if ((jsock = switch_core_hash_find(verto_globals.jsock_hash, sessid))) {
		cJSON *params = NULL;
		cJSON *msg = NULL;
		msg = jrpc_new_req("verto.punt", NULL, &params);
		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_WARNING, "%s dropping connection.\n", jsock->uuid_str);
		switch_core_hash_delete(verto_globals.jsock_hash, jsock->uuid_str);
		ws_write_json(jsock, &msg, SWITCH_TRUE);
		cJSON_Delete(msg);
		jsock->nodelete = 1;
		jsock->drop = 1;
		stream->write_function(stream, jsock->uuid_str);
	}

	switch_mutex_unlock(verto_globals.jsock_mutex);

	return SWITCH_STATUS_SUCCESS;
}
```
