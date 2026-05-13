# Scopes-SOC Overview

Scopes-SOC is a Docker Compose SOC lab for local use. It runs Wazuh Indexer, Wazuh Manager, Wazuh Dashboard, Suricata, and an optional traffic generator with persistent Docker volumes and isolated lab networking.

This project is safe to view on GitHub because it does not include real credentials. It is still a lab configuration, not a production deployment.

## Security Defaults

- `.env` is ignored by Git and should contain your local-only secrets.
- `.env.example` intentionally leaves password fields blank.
- Published service ports bind to `127.0.0.1` by default.
- Wazuh Indexer security and dashboard TLS are disabled for lab simplicity, so do not expose this stack directly to a LAN or the internet.

If this project was ever committed with real passwords, rotate those values before reusing the lab.

## Requirements

- Docker Engine or Docker Desktop
- Docker Compose plugin
- At least 4 GB free RAM for the lab
- `sudo` access on Pop!_OS or another Linux host

Set the required virtual memory value for Wazuh Indexer:

```bash
sudo sysctl -w vm.max_map_count=262144
```

To make that setting survive reboots:

```bash
echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system
```

## Configure

From this project directory:

```bash
cp .env.example .env
```

Generate strong local passwords and add them to `.env`:

```bash
openssl rand -base64 32
```

Required values:

```text
INDEXER_PASSWORD=
WAZUH_API_PASSWORD=
DASHBOARD_PASSWORD=
```

Keep the default `127.0.0.1` bind addresses unless you have a firewall and a specific reason to accept connections from other hosts.

## Launch

Start the lab:

```bash
sudo docker compose up -d
```

Check that everything is running:

```bash
sudo docker compose ps
```

Expected services:

- `wazuh-indexer`
- `wazuh-manager`
- `wazuh-dashboard`
- `suricata`

## Open The Dashboard

Open:

```text
http://localhost:8443
```

Use the dashboard username and password you set in `.env`:

```text
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=<your-local-password>
```

The Wazuh API is available only on localhost by default:

```text
https://localhost:55000
```

Use the API username and password you set in `.env`:

```text
WAZUH_API_USERNAME=wazuh-wui
WAZUH_API_PASSWORD=<your-local-password>
```

## Generate Test Traffic

Start a temporary traffic generator on the isolated sensor network:

```bash
sudo docker compose --profile tools run --rm traffic-generator bash
```

Inside that shell, ping Suricata:

```bash
ping -c 2 suricata
```

Exit the shell:

```bash
exit
```

## View Suricata Alerts

Suricata writes logs to the persistent `suricata_logs` volume. Check alerts with:

```bash
sudo docker exec soc-lab-suricata tail -n 20 /var/log/suricata/fast.log
```

Check JSON events with:

```bash
sudo docker exec soc-lab-suricata tail -n 5 /var/log/suricata/eve.json
```

The included rule `SOC-LAB ICMP ping detected` should fire when you ping Suricata from the traffic generator.

## Enable Suricata In Wazuh

Suricata logs are mounted read-only into the Wazuh Manager at:

```text
/var/log/suricata/eve.json
```

To have Wazuh ingest Suricata alerts, add this block to `/var/ossec/etc/ossec.conf` inside the manager:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

You can copy the snippet from `wazuh/suricata-localfile.conf`.

Edit the manager config:

```bash
sudo docker exec -it soc-lab-wazuh-manager bash
vi /var/ossec/etc/ossec.conf
/var/ossec/bin/wazuh-control restart
exit
```

The manager config is stored in a persistent Docker volume, so this change remains after container restarts.

## Networks

- `soc_backend`: internal Wazuh service network.
- `sensor_net`: internal Suricata lab traffic network.
- `soc_frontend`: host-facing network for localhost-published Wazuh Dashboard and Manager/API ports.

Published host ports bind to `127.0.0.1` by default:

- Dashboard: `8443/tcp`
- Wazuh API: `55000/tcp`
- Agent events: `1514/tcp` and `1514/udp`
- Agent enrollment: `1515/tcp`
- Syslog input: `5514/udp`

To allow another host to connect, override the matching `*_BIND` value in `.env`. Only do this on a trusted network with firewall controls.

## Stop

Stop containers while keeping data:

```bash
sudo docker compose down
```

Stop containers and delete persistent lab volumes:

```bash
sudo docker compose down -v
```

## Troubleshooting

View all logs:

```bash
sudo docker compose logs --tail=100
```

View one service:

```bash
sudo docker compose logs --tail=100 wazuh-indexer
sudo docker compose logs --tail=100 wazuh-manager
sudo docker compose logs --tail=100 wazuh-dashboard
sudo docker compose logs --tail=100 suricata
```

If the indexer fails to start, verify:

```bash
sysctl vm.max_map_count
```

It should be at least:

```text
262144
```

If the dashboard is unreachable, confirm the published port:

```bash
sudo docker port soc-lab-wazuh-dashboard
```
