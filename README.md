# acme-solid quickstart

Guide to run a W3C Solid protocol server called. `acme-solid`. 

`acme-solid` is a lean server written in Rust using S3 storage. You can run it locally with `podman compose`, using your own WebID. It pulls pre-built container images and runs them.

## 1. Get access

The container images are private. Message the maintainer with your GitHub username to request
access to `ghcr.io/jeremycaine/acme-solid` and `ghcr.io/jeremycaine/acme-idp`.

Once granted, log in locally with a [GitHub personal access token](https://github.com/settings/tokens)
that has the `read:packages` scope:

```bash
podman login ghcr.io -u <your-github-username> --password-stdin
# paste your PAT, then Ctrl-D
```

## 2. Prerequisites

| Tool | Purpose |
|---|---|
| [Podman](https://podman.io/) v5.8+ | Runs the compose stack |
| A GitHub account with image access (step 1) | Pulls the images |

## 3. One-time host setup

The stack uses the hostname `server` internally and externally, so add this to `/etc/hosts`:

```bash
echo "127.0.0.1 server" | sudo tee -a /etc/hosts
```

## 4. Start the stack

If you are using a WebID you from another Solid IDP then skip to section 6.

```bash
git clone https://github.com/jeremycaine/acme-solid-quickstart.git
cd acme-solid-quickstart
cp .env.example .env
```

Generate an OIDC signing key and paste it into `.env` as `OIDC_JWKS`:

```bash
podman run --rm ghcr.io/jeremycaine/acme-idp:latest node dist/keygen.js
```

Pull and start everything:

```bash
podman compose pull
podman compose up -d
```

Check health:
```bash
curl http://server:3000/health/live   # → ok
curl http://server:4000/health        # → ok
```

## 5. Create your own account/WebID

```bash
podman compose exec acme-idp node dist/create-account.js yourname yourname@example.com my-secret-password
```

This prints your new WebID, e.g. `http://server:4000/users/yourname#me`.

Make it the pod owner — edit `.env`:
```
SERVER_OWNER_WEBID=http://server:4000/users/yourname#me
```

Then recreate `acme-solid` so it picks up the new value and re-syncs the root access grant (`podman compose restart` does NOT pick up an `.env` change — it restarts the existing container with its already-baked-in environment; `up -d` recreates the container when its config differs):
```bash
podman compose up -d acme-solid
```

You (and only you, via that WebID) now have full Read/Write/Control access to this pod. Point
any Solid-compatible client app at `http://server:3000` as storage, and authenticate against
`http://server:4000` as the issuer.

## 6. Alternative: point at an external IDP instead

If you already have a WebID from another Solid identity provider (e.g. a pod on
solidcommunity.net or Inrupt's PodSpaces), you can skip `acme-idp` entirely and use it instead:

```bash
podman compose up -d seaweedfs valkey acme-solid
```

Set in `.env`:
```
TRUSTED_ISSUERS=https://your-external-idp.example
SERVER_OWNER_WEBID=https://your-external-idp.example/profile/card#me
```

Then:
```bash
podman compose up -d acme-solid
```

## 7. Optional: add HTTPS

See `docker-compose.local-https.yml` for a Caddy-based local-HTTPS overlay (no mkcert required,
but a local [Caddy](https://caddyserver.com/docs/install) install — not just Podman — is needed
for the `caddy trust` step below).
Note: switching to HTTPS changes the issuer/base URL, so redo step 5 under the new URLs if you
already created an account over HTTP.

**Note:** this overlay hasn't been validated for authenticated requests — health checks and
unauthenticated reads should work, but the Solid client authentication flow may not complete
over this local-HTTPS setup without further network/TLS-trust configuration. Treat this as
experimental.

One-time setup:

```bash
echo "127.0.0.1 solid.localhost idp.localhost" | sudo tee -a /etc/hosts
```

Install Caddy locally and trust its local CA:

```bash
caddy trust
```

Run:

```bash
podman compose --profile https -f docker-compose.yml -f docker-compose.local-https.yml up -d
```

## Demo accounts (optional, for a fast smoke test)

Set `SEED_TEST_ACCOUNTS=true` in `.env` before first `podman compose up -d` to get pre-seeded
`alice`/`bob` accounts. These are for a quick look around only — they don't own the pod unless
you also set `SERVER_OWNER_WEBID` to one of their WebIDs.

## Stopping the stack

```bash
podman compose down -v
```


