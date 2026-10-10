# Hanasand Stalwart

Standalone Stalwart mail server and SMTP relay deployment. The webmail client remains in the [Hanasand Mail repository](https://github.com/hanasandcom/mail).

`compose.stalwart.yml` defines the primary Stalwart server and the host-specific relay services. The primary server and each relay run in separate containers and Compose projects. The Hanasand application stack connects to Stalwart through the existing shared Docker network.

## State and credentials

Mailbox data lives in the ignored `data/` directory, and relay state and credentials live in the ignored `mail-relay-state/` directory in this repository. Neither is committed. Set host-specific values in an untracked `.env`, using `.env.example` as a reference.

## Deploy

Deploy the primary Stalwart container from this checkout with `scripts/deploy-stalwart.sh`. Configure or update a host relay with:

```sh
bun mail-relay/setup.ts ovh --image hanasand-mail-relay-health:<revision>
bun mail-relay/setup.ts inspur --image hanasand-mail-relay-health:<revision> --api-container <active-api-container>
```

The setup tool keeps the existing relay Compose project names and network aliases. See [relay operations](mail-relay/README.md) for gateway, TLS, health and sender-authentication procedures.

Run the unit checks with `bun test mail-relay`.
