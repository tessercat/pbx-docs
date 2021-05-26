Systemd runs the project
in daphne
on a localhost port
in a project-specific venv.

An nginx server named `{{ hostname }}`
listens on the public TLS port
and sends requests
to the project
via the daphne port.

An nginx server named `localhost`
listens on a loopback port
and sends requests
to localhost-only endpoints
to the project
via the daphne port.

Project settings allow requests
only to `{{ hostname }}` and `localhost`.
