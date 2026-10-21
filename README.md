# Hanasand Mail Server

This repository runs the Stalwart mail services and the host-specific SMTP relay
and incoming-mail gateways. It does not contain the webmail interface or the
browser-facing Mail API; those are in `mail-web`.

## Service boundaries

| Service | Responsibility | Configuration here |
| --- | --- | --- |
| Stalwart | Mailboxes, JMAP, SMTP, IMAP, and ManageSieve | `compose.stalwart.yml`, `stalwart/` |
| Relay and gateways | Route outbound mail through OVH and deliver inbound mail to Inspur | `relay/`, `compose.stalwart.yml` |
| Webmail | Browser UI at `mail.hanasand.com`; the Hanasand dashboard at `/mail` uses the same UI package | [mail-web](https://github.com/hanasandcom/mail-web) |
| Mail API | Dedicated browser API for both mail UIs; validates sessions with Identity and talks to Stalwart directly | [mail-web API](https://github.com/hanasandcom/mail-web/tree/main/api) |

The mail flow is:

```text
Webmail (standalone or dashboard) → Mail API → Identity, PostgreSQL, and Inspur Stalwart
Outbound: Inspur Stalwart → SSH connector → OVH relay → recipient MX
Inbound: recipient MX → OVH gateway → SSH tunnel → Inspur Stalwart
```

The relay uses separate Compose services and persistent state from the mailbox
server. Relay setup, health checks, gateway configuration, and rollback steps are in
the [relay runbook](relay/README.md).

## Requirements

- Docker Engine and the Docker Compose plugin on each host.
- Bun for relay setup, checks, and health-route installation.
- The shared Hanasand Docker network and the host-specific relay network.
- The existing host state directories and TLS certificates. Deployment preserves
  these directories; do not remove them when replacing containers.

## Configuration and persistent data

Copy `.env.example` to an untracked `.env` and set values for the host. The example
defines:

- `HANASAND_STALWART_STATE_DIR`: Stalwart mailbox and server state.
- `HANASAND_MAIL_RELAY_ROOT`: relay data, credentials, health configuration, and
  gateway configuration.
- `HANASAND_SHARED_NETWORK` and `HANASAND_MAIL_RELAY_NETWORK`: existing Docker
  networks used by the services.
- `HANASAND_MAIL_RELAY_INSPUR_IP`: stable Inspur relay address on its private
  network.

Keep `.env`, private keys, relay credentials, and state files out of Git. Existing
container, image, network, API, and persistent-state names still include `mail-relay`
because deployed services and integrations use those names.

Check Compose interpolation before deploying:

```sh
docker compose --project-name hanasand-mail-stalwart -f compose.stalwart.yml config -q
```

## Deploy Stalwart

Run the deployment from a clean `main` checkout on the host that owns the Stalwart
state directory. The script builds before stopping an older container, verifies that
the persistent mount matches, then waits for the new container's health check and
`/healthz/live` endpoint.

```sh
scripts/deploy-stalwart.sh
```

The script defaults to `/home/hanasand/stalwart/data` for user `hanasand` and
`/home/ubuntu/stalwart/data` for user `ubuntu`. Set
`HANASAND_STALWART_STATE_DIR` only when the existing data is stored elsewhere. The
Compose project is `hanasand-mail-stalwart`; the primary service is `stalwart` and
the existing container name is `hanasand_mail`.

## Deploy and operate relays

Use the [relay runbook](relay/README.md) for the supported sequence. It covers the
OVH and Inspur setup commands, sender authentication, gateway configuration, health
routes, monitoring, certificate refresh, and rollback. Configure and verify relay
authentication before activating outbound routing. The setup tools preserve the
existing runtime names and state paths.

The relay checks run with:

```sh
bun test relay
```

## Verify after changes

Check the Stalwart container and local health endpoint on the host:

```sh
docker compose --project-name hanasand-mail-stalwart -f compose.stalwart.yml ps
curl --fail --show-error http://127.0.0.1:8081/healthz/live
```

For relay changes, verify both public relay health routes against both gateway IPs,
confirm the `X-Mail-Relay-Health-Release` value, and check recipient-server acceptance
in delivery logs. A healthy container or relay probe does not by itself prove inbox
placement. Keep mail queued while repairing a failed relay; do not enable direct
outbound delivery or weaken TLS as a fallback.

## Repository layout

- `stalwart/`: pinned Stalwart image definition.
- `relay/`: health service, sender checks, setup tools, and operational runbook.
- `scripts/`: Stalwart deployment and TLS synchronization.
- `compose.stalwart.yml`: Stalwart and relay service definitions.
