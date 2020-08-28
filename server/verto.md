# Login username and sessid constraint

Constrain login username and sessid
to be identical
as a simple method
to prevent multiple client connections
using the same username/password
in the same channel.
Add the following to `attach_jsock`.

```
 else if (jsock->uuid_str != jsock->id) {
            proceed = 0;
            cJSON *params = NULL;
            cJSON *msg = NULL;
            msg = jrpc_new_req("verto.punt", NULL, &params);
            switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_WARNING, "Constraint failure %s dropping new connection.\n", jsock->uuid_str);
            switch_core_hash_delete(verto_globals.jsock_hash, jsock->uuid_str);
            ws_write_json(jsock, &msg, SWITCH_TRUE);
            cJSON_Delete(msg);
            jsock->nodelete = 1;
            jsock->drop = 1;
        }
```

Could this also be done
with the `verto_punt` API command
and a Lua script
that listens for login
(or possibly client connect)
events?
I think so.
The script could check
that login username and sessid are identical
and call `verto_punt`
on the connection if they are not.
If login username and sessid are identical,
the script could check the output of `verto status`,
and `verto_punt` the new session
if a client with login username already exists.

    <hook event="CUSTOM" subclass="verto::login" script="login.lua"/>


# Punt API command

Add `verto_punt` API command.

```
SWITCH_ADD_API(api_interface, "verto_punt", "Punt a verto session", verto_punt_function, "sessid");

SWITCH_STANDARD_API(verto_punt_function)
{
        jsock_t *jsock;
        char *sessid = (char *) cmd;
        switch_status_t status;

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
            status = SWITCH_STATUS_SUCCESS;
        } else {
            status = SWITCH_STATUS_FALSE;
        }

        switch_mutex_unlock(verto_globals.jsock_mutex);

        return status;
}
```
