# Dograh on Coolify from source

Use this setup for the DROW-CPU/dograh fork on the existing Contabo VPS.
It builds the API and UI from the selected Git commit. Coolify terminates
HTTPS and routes to a private gateway on container port 8080.

The earlier Railpack deployment built no application start command. Its
single container restarts, so Traefik has no healthy backend and returns
"No available server". Do not reuse Railpack for this multi-service stack.

## Coolify settings

In the existing Git application, stop the failed Railpack resource, then set:

| Setting | Value |
| --- | --- |
| Repository | https://github.com/DROW-CPU/dograh |
| Branch | main (after uploading these files to the fork) |
| Build strategy | Docker Compose / Compose |
| Base Directory | / |
| Docker Compose Location | /docker-compose.coolify.yaml |
| Git submodules | Enabled (the API Dockerfile needs pipecat) |
| Gateway service domain | https://voice.bluskyvas.com:8080 |
| Domains on API, UI, storage, database, Redis, or TURN | None |

The :8080 suffix selects the internal gateway port. Visitors use
https://voice.bluskyvas.com on ordinary HTTPS port 443.

If Coolify cannot convert the existing resource to Compose, create another
Git-based application with these settings. Keep the failed resource stopped
and remove its domain assignment so only the Compose gateway owns the hostname.
Do not delete database volumes to troubleshoot deployment.

## Environment variables and secrets

Load/reload the Compose definition in Coolify before deploying.

| Variable | Value |
| --- | --- |
| PUBLIC_HOST | voice.bluskyvas.com |
| PUBLIC_BASE_URL | https://voice.bluskyvas.com |
| SERVER_IP | 89.116.24.99 |
| ENABLE_SIGNUP | true for initial account creation |
| ENABLE_TELEMETRY | false by default |
| FORCE_TURN_RELAY | false initially; true can test TURN-only media |

Coolify's SERVICE_PASSWORD_64 variables generate independent 64-character
alphanumeric secrets. Confirm these have non-empty values:

- SERVICE_PASSWORD_64_POSTGRES
- SERVICE_PASSWORD_64_REDIS
- SERVICE_PASSWORD_64_MINIO
- SERVICE_PASSWORD_64_JWT
- SERVICE_PASSWORD_64_TURN

If your Coolify version does not generate one, generate a value privately with
`openssl rand -hex 32` on the VPS and enter it in that variable. Never commit
the generated values, paste them into logs, or send them in chat. The same
variable is reused by the services that must share that credential.

Retain these values across redeployments. Changing the PostgreSQL environment
secret does not rotate the password already stored in its persistent volume.

## DNS, HTTPS, and voice ports

The A record for voice.bluskyvas.com must point to 89.116.24.99. Only add an
AAAA record if IPv6 has been verified for this server. Coolify needs TCP
80 and 443 for the proxy and certificate issuance; these are already allowed
in the existing UFW setup.

Run these additional commands in the VPS for browser audio:

```bash
sudo ufw allow 3478/tcp
sudo ufw allow 3478/udp
sudo ufw allow 49152:49200/udp
```

If a Contabo/provider firewall is enabled, permit the same ports there.
TURN uses the raw VPS IP, so it does not depend on a Cloudflare-proxied
application hostname. This stack offers authenticated TURN over UDP/TCP
3478. TLS TURN on 5349 is disabled on both the API and coturn because no
TURN certificate is provisioned here. Networks requiring TURN over TLS need
an independently configured TLS TURN service.

No database, Redis, API, UI, MinIO, or gateway ports are published on the
host. In particular the API's container port 8000 does not conflict with
Coolify's host port 8000. Only TURN's listener and relay ports are published.
Docker's published ports require Docker-aware firewall controls; UFW alone
does not reliably restrict bridge-network port publishing.

## Storage

The stack runs the bundled MinIO image with a persistent volume, but uses
Dograh's existing S3-compatible backend. No AWS account is required.
The one-shot storage-init service creates the voice-audio bucket and removes
anonymous access. Object downloads and direct uploads use signed S3 URLs.

S3_ENDPOINT_URL is the public HTTPS origin. The gateway preserves the
original Host and URI while proxying /voice-audio/ to MinIO, so the SDK and
browser use the same signature endpoint. DNS and HTTPS must also work from
inside the VPS/API container. There is no public MinIO administration console.

This assumes a new deployment. Do not attach existing anonymous-storage
volumes without reviewing their policies and regenerating any old unsigned
links. Back up all three data volumes before migrations or major upgrades.

## First deployment and verification

1. Confirm the generated secrets, Git submodules, and gateway domain above.
2. Deploy. Initial Python and Next.js source builds can take several minutes.
3. Wait for postgres, redis, minio, api, ui, and gateway to be healthy.
   storage-init should exit successfully with code 0; it is not a daemon.
4. Check https://voice.bluskyvas.com/api/v1/health for JSON with status "ok",
   deployment_mode "oss", auth_provider "local", and the public API endpoint.
5. Open https://voice.bluskyvas.com and create your account.
6. For a private team install, set ENABLE_SIGNUP=false and redeploy.
7. Create a test agent and verify both Test Chat and Test Audio.
8. Upload a test file and verify signed upload/download URLs work; an unsigned
   /voice-audio/object request must not return private object contents.
9. Confirm inbound telephony webhooks and WSS media separately after adding
   provider credentials. A healthy UI does not verify a live phone call.

If the site still shows "No available server", inspect runtime logs and the
health status of gateway, ui, and api. Deployment success alone only confirms
that Docker accepted the containers.

## Validation limits

The configuration is checked against the Compose JSON schema with substituted
test values, the service dependency graph, required secret references, source
Dockerfile paths, private port bindings, and the Python bucket-init syntax.
It has not been built or run against the user's VPS during preparation.
Live source builds, Nginx startup, signed storage operations, and WebRTC must
be verified by the first Coolify deployment.

