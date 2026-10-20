# Incoming mail gateway and outbound relay

Outgoing mail follows this route:

```text
Inspur Stalwart → SSH connector → OVH Stalwart → recipient MX
```

Every mailbox uses the same relay; there is no per-address allowlist. Local mailbox
delivery stays on Inspur. Since Inspur blocks internet SMTP, incoming mail follows
`OVH port 25 → HAProxy → encrypted reverse SSH tunnel → Inspur Stalwart`. The PROXY
protocol preserves the sender IP for spam checks and rate limits.

## Services and health

| Host | Container | Local health | Public readiness API |
| --- | --- | --- | --- |
| Inspur | `hanasand-mail-relay-inspur` | `127.0.0.1:19261/health` | `https://api.hanasand.com/api/mail-relay/inspur/health` |
| OVH | `hanasand-mail-relay-ovh`, `hanasand-mail-relay-ovh-health` | `127.0.0.1:19262/health` | `https://api.hanasand.com/api/mail-relay/ovh/health` |

Public readiness routes are served by the gateways. They do not depend on the Hanasand
API, its database, or its recovery mode. Each gateway reads its local relay and checks
the peer gateway over certificate-verified HTTPS. The exact GET/HEAD routes are
rate-limited, send no client credentials, and do not cache health samples.

Checks run every 30 seconds:

- HTTP 200 means ready. HTTP 503 means a dependency failed, delivery is more than five
  minutes overdue, the queue exceeds 100 messages, or the sample is stale.
- A closed SMTP connection ends the current probe promptly; the checker records the
  failed sample and continues its next 30-second poll.
- Inspur checks SMTP authentication, its queue, the tunnel, and OVH authentication.
- OVH checks SMTP authentication with the Sales sender, its queue, an external
  Microsoft SMTP greeting, and STARTTLS through the incoming gateway and tunnel.
- Probes never send DATA or issue authenticated RCPT commands, so they do not use
  recipient quotas.

For recipients awaiting their first scheduled attempt, queue age starts at the
scheduled time because Stalwart delays aggregate reports by up to three hours.
Failed and retried recipients use message creation time; completed recipients do not
count as backlog. Readiness cannot establish inbox placement. Outbound logs show
recipient-server acceptance.

The Hanasand API's `api/scripts/setup-mail-relay-monitoring.ts` idempotently creates exactly two named one-minute health jobs using its existing monitor owner and notification destinations. Existing job preferences are preserved.

## Security and persistence

- Only OVH’s incoming SMTP gateway binds publicly on port 25. Submission, management and the reverse tunnel stay private. Inspur still validates recipients and rejects unauthenticated relaying.
- Dedicated SSH key, pinned host key, no shell, and two permitted forward destinations plus one loopback-only reverse listener on port 2625. SSH reconnects automatically with bounded keepalive failure detection.
- STARTTLS and certificate validation are required on the relay route. Inspur authenticates each mailbox and enforces sender ownership. OVH accepts the Hanasand sender domain (and empty bounce senders) only from the private `inspur-relay` account over TLS. Other identities retain sender matching, and unauthenticated relaying remains denied. Queue health principals have read-only queue permissions.
- Mail relay and health containers run as UID 1000 without capabilities. The gateway starts with only bind/setuid/setgid capabilities, binds port 25, then drops its worker to the haproxy user. All use read-only root filesystems, restart policies, resource limits and rotating logs. Connector replacement preserves its Inspur address because API pairs capture network aliases.
- Relay configuration and secrets remain in the existing host-side state directory at `$HOME/hanasand/mail/mail-relay`, mode 0700; secret files remain mode 0600. Compose mounts that directory and does not remove it. OVH SMTP storage remains durable in its existing data directory.
- OVH uses SPF authorization for `192.99.32.185` and RSA DKIM selector `ovhrelay202609`. Its private signing key stays in the relay datastore; publish only the public TXT record. The existing MX points to `mail.hanasand.com`, whose explicit A record must point to OVH `192.99.32.185`; do not rely on the Inspur wildcard. This also matches OVH’s reverse DNS.
- Hourly certificate refresh uses the existing OVH certificate, atomically writes private files, and reloads Stalwart on renewal. Logs use the `hanasand-mail-relay-tls` syslog tag.

## Deploy from the mail-server repository

Fast-forward the `mail-server` checkout on each host, then run setup from that
repository. Compose owns the relay, connector, gateway, and Stalwart containers. Tag
the health image with the full pushed revision so both hosts run the same code:

```sh
revision=$(git rev-parse HEAD)
"$HOME/.local/bin/bun" relay/setup.ts ovh --image "hanasand-mail-relay-health:$revision"
"$HOME/.local/bin/bun" relay/setup.ts inspur --image "hanasand-mail-relay-health:$revision" --api-container <active-api-container>
```

Inspur setup also needs the active API container to read the live sender credential,
including configured password overrides. Compose reuses existing bind mounts and keeps
the connector's shared-network address. The Stalwart image is pinned; setup refuses an
unexpected relay image or persistent mount.

Provision the restricted SSH key and remote SMTP credential locally before initial Inspur setup. Do not print credentials, DKIM generation responses, or message contents. Initial OVH DNS setup must authorize the outbound IP and publish the generated DKIM public key before activation; configure `auth.dkim.sign` to `['ovh-relay-rsa']` and reload `/api/reload`.

Run `bun relay/setup.ts ovh --configure-gateway` on OVH, then `bun relay/setup.ts inspur --configure-gateway` on Inspur to trust only its connector IP and restart the SMTP listener. Run `bun relay/setup.ts inspur --activate` only once relay authentication passes. This preserves a protected `route-before.json` snapshot and routes every external recipient through OVH with mandatory TLS. Install certificate refresh on OVH with `bun relay/setup.ts ovh --renewal-revision <full-pushed-sha>`; `--refresh-tls` also runs it immediately.

After deployment check both public APIs, both named jobs, and actual delivery logs. A queue is preserved across restarts and failures. If the relay fails, leave mail queued, repair the failing dependency, then verify recipient acceptance. Do not silently fall back to blocked direct outbound delivery or weaken TLS. Restore prior route settings from the protected snapshot only as an explicit rollback.

### Deploy gateway health routes

After fetching the pushed revision on both hosts, run `"$HOME/.local/bin/bun" relay/install-health-routes.ts <site> <nginx-config-root> <revision>`, with site `inspur` or `ovh`. Existing gateway configuration is backed up, syntax-checked and gracefully reloaded; validation failure restores the previous files. Mail servers and queues are not restarted.

Verify both health URLs against **both gateway IPs** using `curl --resolve api.hanasand.com:443:<gateway-IP>`. Check the `X-Mail-Relay-Health-Release` response header and fresh relay checks. The OVH application API may still reject nonessential routes during recovery; relay health must work independently of it. Authentication, queue, outbound-connectivity and stale-sample failures must continue returning 503.

### Incoming sender validation

After fetching the pushed revision on Inspur, run `"$HOME/.local/bin/bun" relay/enforce-sender-auth.ts`. This reloads sender authentication without restarting mail or changing routing. Unauthenticated SMTP enforces published DMARC reject policies, including `hanasand.com`; authenticated submission keeps its existing sender ownership checks. Configuration failures restore the previous DMARC settings.
