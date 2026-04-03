# Dendrite on Synology NAS

A self-hosted [Matrix](https://matrix.org) homeserver running on a Synology NAS using Docker Compose (Container Manager), with Synology's built-in nginx acting as the reverse proxy.

> **Note:** Synology's nginx permanently owns ports 80 and 443 and cannot be displaced. This guide uses Synology's built-in Application Portal reverse proxy to route traffic to Dendrite, and Synology's built-in Let's Encrypt integration for TLS certificates.

---

## Architecture

```
Internet → Router (port 443) → Synology nginx (Application Portal)
                                      ↓
                               localhost:8008
                                      ↓
                          Docker: dendrite-monolith
                                      ↓
                          Docker: dendrite-postgres
```

---

## Prerequisites

- A Synology NAS with **Container Manager** installed
- A domain name with DNS managed in Cloudflare (this guide uses `matrix.yourdomain.com` as an example)
- A DNS **A record** for `matrix.yourdomain.com` pointing to your Synology's public IP
  - Cloudflare **Proxy status must be set to DNS only** (grey cloud) — the orange proxy cloud will break TLS certificate issuance
- Ports **80** and **443** forwarded to your Synology's local IP on your router

---

## Step 1 — Enable SSH on your Synology

DSM → **Control Panel** → **Terminal & SNMP** → enable SSH.

---

## Step 2 — SSH in and create the project folder

```bash
ssh you@your-synology-ip
cd /volume1/docker
mkdir dendrite && cd dendrite
```

---

## Step 3 — Generate the private key

This only needs to be run **once**. Re-running it will overwrite the existing key and break your server.

```bash
mkdir -p ./config

docker run --rm --entrypoint="/usr/bin/generate-keys" \
  -v $(pwd)/config:/mnt \
  ghcr.io/element-hq/dendrite-monolith:latest \
  -private-key /mnt/matrix_key.pem
```

---

## Step 4 — Generate the config

Replace `matrix.yourdomain.com` with your actual domain. The Postgres credentials must match those in `docker-compose.yaml`.

```bash
docker run --rm --entrypoint="/bin/sh" \
  -v $(pwd)/config:/mnt \
  ghcr.io/element-hq/dendrite-monolith:latest \
  -c "/usr/bin/generate-config \
    -dir /var/dendrite/ \
    -db postgres://dendrite:YOUR_POSTGRES_PASSWORD@postgres/dendrite?sslmode=disable \
    -server matrix.yourdomain.com > /mnt/dendrite.yaml"
```

---

## Step 5 — Edit dendrite.yaml

Open the generated config:

```bash
nano ./config/dendrite.yaml
```

Make the following changes:

**1. Set real IP forwarding** so Dendrite sees actual client IPs through the reverse proxy. Find the `sync_api` section — it will already exist with an empty value:

```yaml
sync_api:
  real_ip_header: X-Forwarded-For
```

**2. Set a shared secret** to enable admin account creation. Find `registration_shared_secret` under `client_api` and set it to a long random string. Generate one with:

```bash
openssl rand -hex 32
```

```yaml
client_api:
  registration_shared_secret: "your-generated-secret-here"
```

---

## Step 6 — Place project files

Copy `docker-compose.yaml` into the project folder using File Station or `scp`. Your directory should look like this:

```
/volume1/docker/dendrite/
├── docker-compose.yaml
└── config/
    ├── dendrite.yaml
    └── matrix_key.pem
```

---

## Step 7 — Load the project in Container Manager

1. Open DSM → **Container Manager** → **Project**
2. Click **Create**
3. Set the project path to `/volume1/docker/dendrite`
4. Container Manager will detect `docker-compose.yaml` automatically
5. Click **Next** and then **Done** to start the stack

Confirm all three containers are running (postgres, monolith).

---

## Step 8 — Configure the reverse proxy in Application Portal

DSM → **Control Panel** → **Login Portal** → **Advanced** tab → **Reverse Proxy** → **Create**

| Field | Value |
|---|---|
| Reverse Proxy Name | `Matrix` |
| Source Protocol | `HTTPS` |
| Source Hostname | `matrix.yourdomain.com` |
| Source Port | `443` |
| Destination Protocol | `HTTP` |
| Destination Hostname | `localhost` |
| Destination Port | `8008` |

Click **Save**.

---

## Step 9 — Obtain a TLS certificate

DSM → **Control Panel** → **Security** → **Certificate** → **Add**

1. Select **Get a certificate from Let's Encrypt**
2. Enter `matrix.yourdomain.com` as the domain name
3. Enter your email address
4. Click **Done** — Synology will handle the ACME challenge automatically

Once issued, click **Configure** and assign the new Let's Encrypt certificate to `matrix.yourdomain.com` in the dropdown, then click **OK**.

---

## Step 10 — Create your first admin user

SSH into the Synology and run:

```bash
docker exec -it dendrite-monolith-1 /usr/bin/create-account \
  -config /etc/dendrite/dendrite.yaml \
  -username youruser \
  -admin \
  -url http://localhost:8008
```

You will be prompted to set a password.

---

## Step 11 — Connect a Matrix client

Download [Element](https://element.io/download) (web, desktop, or mobile). On the sign-in screen:

1. Click **Edit** next to the homeserver
2. Select **Other homeserver**
3. Enter `https://matrix.yourdomain.com`
4. Log in with the username and password you just created

---

## Verifying federation

Use the [Matrix Federation Tester](https://federationtester.matrix.org/) and enter `matrix.yourdomain.com` to confirm your server is reachable and federating correctly.

---

## Updating Dendrite

In Container Manager, select the dendrite project and click **Update**, or via SSH:

```bash
cd /volume1/docker/dendrite
docker compose pull
docker compose up -d
```

---

## Troubleshooting

### `create-account` fails with "Shared secret registration is not enabled"

The `registration_shared_secret` field in `dendrite.yaml` is empty. Set it to a long random string (see Step 5), then restart the dendrite project in Container Manager and try again.

### `create-account` fails with HTTP 404

The tool needs to reach Dendrite's internal API directly. Always pass `-url http://localhost:8008` to bypass the reverse proxy:

```bash
docker exec -it dendrite-monolith-1 /usr/bin/create-account \
  -config /etc/dendrite/dendrite.yaml \
  -username youruser \
  -admin \
  -url http://localhost:8008
```

Confirm Dendrite is healthy first:

```bash
curl http://localhost:8008/_matrix/client/versions
```

### Element reports "Homeserver URL does not appear to be a valid Matrix homeserver"

The TLS certificate is invalid or the reverse proxy rule is not routing correctly. Test with:

```bash
curl -sk https://matrix.yourdomain.com/_matrix/client/versions
```

If this returns a JSON response but without `-k` it fails, the certificate is self-signed — complete Step 9 to obtain a Let's Encrypt certificate and assign it to the domain.

If it times out entirely, the reverse proxy rule in Application Portal is not correctly configured or port 443 is not forwarded on your router.

### Port 443 conflict — "address already in use"

Synology's nginx permanently owns ports 80 and 443 and cannot be stopped without breaking DSM. Do not attempt to run a separate reverse proxy (Caddy, nginx container, etc.) on these ports — use Synology's built-in Application Portal instead, as described in Step 8.

### Verifying Dendrite is running correctly

```bash
# Check all containers are up
docker ps | grep dendrite

# Check Dendrite API is responding
curl http://localhost:8008/_matrix/client/versions

# Check Dendrite logs
docker logs dendrite-monolith-1 --tail 50
```
