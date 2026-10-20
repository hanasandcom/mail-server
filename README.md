# Hanasand Mail Server

This repository deploys Stalwart and the host-specific SMTP relay and incoming-mail
gateway. The [mail-web repository](https://github.com/hanasandcom/mail-web) contains
the webmail UI. Mailbox data stays in Stalwart; the Hanasand API provides the app's
mailbox actions.

## Deployment units

| Unit | What it runs | How to deploy |
| --- | --- | --- |
| Mail server | Primary Stalwart service on the application network | `scripts/deploy-stalwart.sh` |
| OVH relay | Stalwart relay and public SMTP gateway | `bun relay/setup.ts ovh --image hanasand-mail-relay-health:<revision>` |
| Inspur relay | Private connector and relay health service | `bun relay/setup.ts inspur --image hanasand-mail-relay-health:<revision> --api-container <active-api-container>` |

The primary server and each host relay use separate Compose projects. The relay's
source and setup tools live in `relay/`. Existing container, network, API, image, and
persistent-state names retain `mail-relay` because deployed systems already use them.

## State and configuration

Copy `.env.example` to an untracked `.env` and set host-specific values. Mailbox data
uses `HANASAND_STALWART_STATE_DIR`; relay state and credentials use
`HANASAND_MAIL_RELAY_ROOT`. Both directories are ignored by Git and must be preserved
across deployments. Do not commit `.env` or state files.

The Compose definition is `compose.stalwart.yml`. It connects the primary Stalwart
service to the existing Hanasand Docker network and keeps relay services behind the
`relay-ovh` and `relay-inspur` profiles.

## Deploy

On a mail host, run `scripts/deploy-stalwart.sh` from a clean `main` checkout. It
fast-forwards `origin/main`, builds the image, replaces only the primary Stalwart
container, and checks health before returning.

For relay setup and operations, follow the [relay runbook](relay/README.md). It covers
host setup, gateway configuration, certificate refresh, sender checks, health routes,
and rollback. Configure and verify relay authentication before activating outbound
routing. Setup commands preserve existing runtime names and state paths.

Run the relay checks locally with:

```sh
bun test relay
```
