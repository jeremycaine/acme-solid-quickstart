# acme-solid quickstart

Run [acme-solid](https://github.com/jeremycaine/acme-solid) — a Solid protocol pod server —
locally with `podman compose`, using your own WebID.

This repo has no source code. It pulls pre-built container images and runs them.

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
podman compose exec acme-idp node dist/create-account.js alice alice@example.com my-secret-password
```

This prints your new WebID, e.g. `http://server:4000/users/alice#me`.

Make it the pod owner — edit `.env`:
```
SERVER_OWNER_WEBID=http://server:4000/users/alice#me
```

Then restart `acme-solid` so it re-syncs the root access grant:
```bash
podman compose restart acme-solid
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
podman compose restart acme-solid
```

## 7. Optional: add HTTPS

See `docker-compose.local-https.yml` for a Caddy-based local-HTTPS overlay (no mkcert required).
Note: switching to HTTPS changes the issuer/base URL, so redo step 5 under the new URLs if you
already created an account over HTTP.

## Demo accounts (optional, for a fast smoke test)

Set `SEED_TEST_ACCOUNTS=true` in `.env` before first `podman compose up -d` to get pre-seeded
`alice`/`bob` accounts. These are for a quick look around only — they don't own the pod unless
you also set `SERVER_OWNER_WEBID` to one of their WebIDs.

## What this is

`acme-solid` is a lean, S3-backed [W3C Solid](https://solidproject.org/) server. See the
[main project](https://github.com/jeremycaine/acme-solid) for architecture and protocol details.
