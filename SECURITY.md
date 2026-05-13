# Security Notes

This repository is a local SOC lab only. It is not hardened for production, internet exposure, or shared network deployment.

## Safe GitHub Publishing

- Do not commit `.env`; it is ignored by Git.
- Keep `.env.example` free of real passwords, tokens, private keys, hostnames, and internal IP addresses.
- Generate fresh local passwords before starting the lab.
- Treat any value that has already been committed to a public repository as compromised and rotate it.

## Local Exposure Defaults

Published Compose ports bind to `127.0.0.1` by default. This keeps the dashboard, API, enrollment, agent, and syslog ports reachable only from the host running Docker.

If you change any `*_BIND` value to `0.0.0.0` or a LAN address, place the lab behind a firewall and use strong unique credentials. The included Compose configuration disables indexer security and TLS for lab simplicity, so it should remain on a trusted local machine.
