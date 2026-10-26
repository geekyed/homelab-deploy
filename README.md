# homelab-deploy

The deployment platform for the mac mini. One repo holds the compose files that
say what runs on the machine, one script rolls them out safely, and one reusable
GitHub Actions workflow builds the images those compose files point at.

Service repos stay pure code. They contain a Dockerfile, a dev-only compose file
for running locally, and a four-line workflow that builds and pushes an image.
They do not contain anything about how the mini is wired up.

```
  agile-ed (code)                    homelab-deploy (this repo)
  ───────────────                    ─────────────────────────
  push to main
      │
      └─► GitHub Actions ──uses───►  .github/workflows/publish.yml
            tsc gate                     QEMU + buildx, linux/arm64
            build + push                 pushes :main and :<sha>
                    │
                    ▼
             ghcr.io/geekyed/agile-ed
                    │
                    │  polled every 300s by launchd
                    ▼
  mac mini:  ~/Projects/homelab-deploy
              services/agile-ed/compose.yml   ◄── the only compose that matters
              services/agile-ed.conf
              bin/deploy agile-ed
                    │
                    ├─ pull        (fail here = missed deploy, not an outage)
                    ├─ up --wait   (blocks on the healthcheck)
                    └─ rollback    (if it never goes healthy)
```

## Layout

| Path | What it is |
|---|---|
| `bin/deploy <service>` | Pull, roll out, health gate, roll back. Safe to run repeatedly. |
| `bin/register <service> <action>` | Manage the launchd agent: `install`, `run`, `status`, `remove`. |
| `services/<name>.conf` | Policy only: project name, paths, timeouts, interval. |
| `services/<name>/compose.yml` | The deployed contract: image tag, ports, volumes, env. |
| `.secrets/<name>.env` | Secrets, on the mini only. Gitignored, never committed. |
| `logs/<name>.log` | launchd stdout. `.err` for stderr. Gitignored. |
| `.github/workflows/publish.yml` | Reusable build+push, called by service repos. |

The split between `.conf` and `compose.yml` is deliberate. The compose file is
the service's own contract and holds what the service *is*. The conf holds what
the *platform* does about it: where files are, how long to wait, how often to
poll. Keeping the image tag next to the port bindings means a rollback is one
reviewed edit rather than two files agreeing on a variable.

## Services

| Service | Image | Loopback | Notes |
|---|---|---|---|
| `agile-ed` | `ghcr.io/geekyed/agile-ed` | `127.0.0.1:3000` | App plus Redis; needs `.secrets/agile-ed.env`. |
| `edward-weston-cv` | `ghcr.io/geekyed/edward-weston-cv` | `127.0.0.1:8081` | Static CV; no secrets, but the env file must exist. |
| `notion-discord` | `ghcr.io/geekyed/notion-discord` | `127.0.0.1:8080` | .NET app plus Redis; stateful Redis volume; needs `.secrets/notion-discord.env`. |

The CV is published at `https://cv.edweston.uk` through the same cloudflared
tunnel, with a dashboard rule mapping that hostname to
`http://127.0.0.1:8081`. Tunnel rules are configured in Cloudflare, not in this
repo.

`edward-weston-cv` is the first service with nothing to keep secret. `bin/deploy`
still requires its env file to exist, so on the mini:

```bash
mkdir -p .secrets
touch .secrets/edward-weston-cv.env
```

`notion-discord` is published at `https://nd.edweston.uk` through the same
tunnel, mapping to `http://127.0.0.1:8080`. If the existing rule says
`localhost:8080`, change it to `127.0.0.1:8080` so it matches the IPv4 loopback
binding.

`notion-discord` is the stateful one: its Redis volume holds Notion verification
tokens and queued events. The deployed compose pins the volume name to
`notion-discord_redis-data`, the volume the original manual deployment created.
Do not change that name or run `docker compose down -v`. The first migration
from the manual setup is documented in the notion-discord repo's `DEPLOY.md`.
Follow that manual cutover before running `./bin/register notion-discord install`;
`install` runs `bin/deploy` immediately.

## Why each service gets its own compose project

`bin/deploy` passes `-p "$PROJECT"` explicitly. That gives every service its own
Docker network, which is not tidiness — it is load-bearing. `agile-ed`'s login
rate limiter trusts the `cf-connecting-ip` header on the grounds that nothing but
the cloudflared tunnel can reach the app. If services shared one network, a
sibling container could POST straight to `agile-ed:3000` with a forged header and
mint a fresh rate-limit bucket per request, silently defeating the brute-force
protection. The separate network is what keeps that assumption true.

The other reason: `docker compose down -v` in a shared project would destroy
every service's volumes, not just the one you meant. For `notion-discord`, that
volume holds the Notion verification tokens; losing it means re-verifying every
webhook.

## Bootstrap the mini (once)

Do this on the Mac mini, not on your laptop.

**1. OrbStack is running, and starts by itself after a reboot.**

```bash
docker version && docker compose version
```

OrbStack runs inside a logged-in user session, so the mini needs an auto-login
user. Without one, Docker never comes up after a reboot and nothing else in this
document works either — including cloudflared, which is already a launch daemon
there.

**2. Get a registry token.**

You need read access to the packages, nothing more. In GitHub: Settings →
Developer settings → Personal access tokens. Create a classic token with only
the `read:packages` scope — fine-grained tokens do not authenticate to the
Container registry, and the `repo` scope is broader than needed.

**3. Clone this repo and log in.**

```bash
git clone git@github.com:geekyed/homelab-deploy.git ~/Projects/homelab-deploy
cd ~/Projects/homelab-deploy
echo "$CR_PAT" | docker login ghcr.io -u geekyed --password-stdin
```

`~/.docker/config.json` now holds that token. It is the only credential this
platform needs on the machine: there are no service git checkouts and no deploy
keys, because the mini only ever pulls images.

**4. Create the secrets file for each service.**

```bash
mkdir -p .secrets
$EDITOR .secrets/agile-ed.env
```

```sh
# .secrets/agile-ed.env
ADMIN_PASSWORD='the-real-one'
# Optional, these are the defaults anyway:
# SESSION_TTL_DAYS=30
# LOGIN_MAX_ATTEMPTS=5
# LOGIN_WINDOW_SECONDS=900
# AGILE_ED_TAG=          # leave unset to follow :main; see Rollback
```

For `notion-discord`, copy the existing manual env file rather than retyping it:

```bash
install -m 600 ~/Projects/notion-discord/.env .secrets/notion-discord.env
```

The file uses the same variable names the manual compose used: `NOTION_API_KEY`,
`DISCORD_WEBHOOK_URL`, optional `TENANT0_*`, and optional
`BATCH_INTERVAL_SECONDS`. See the notion-discord repo's `DEPLOY.md` before the
first cutover.

`.secrets/` is gitignored. This file is the only thing you must never commit.

## Install a service

**1. Make sure the image exists in the registry first.** A service cannot be
deployed before CI has published it once. The first `install` will run before
`:main` exists, the pull will fail, and `deploy` will exit without doing
anything — that is the intended behaviour, not a bug. Publish, then install.

**2. Load the agent.**

```bash
cd ~/Projects/homelab-deploy
./bin/register agile-ed install
```

That writes `~/Library/LaunchAgents/com.geekyed.agile-ed.deploy.plist`, loads it,
and deploys immediately because `RunAtLoad` is set. Watch it happen:

```bash
./bin/register agile-ed status
```

**3. Confirm the app is up.**

```bash
curl -fsS http://127.0.0.1:3000/health
```

For `notion-discord`, the health endpoint is `/`:

```bash
curl -fsS http://127.0.0.1:8080/
```

Note the plist embeds this repo's absolute path. If you move or rename the
checkout, re-run `install`.

## How a deploy happens

Every `INTERVAL` seconds (300 by default) launchd runs `bin/deploy <service>`.
There is no webhook and nothing inbound reaches the machine.

launchd does not pull this repo. After changing a compose file or conf, run
`git pull` in `~/Projects/homelab-deploy` on the mini; the next tick uses the new
contract.

- `docker compose pull` — a no-op if the tag has not moved.
- `docker compose up -d --wait` — recreates only the containers whose image or
  config actually changed, then blocks until they are healthy. Unchanged services
  are left running.
- A pull failure exits before any `up`, so an expired token costs you a missed
  deploy rather than an outage.
- A container that never becomes healthy is rolled back to the image the previous
  container was running.

To deploy without waiting for the tick:

```bash
./bin/register agile-ed run
```

To do it right now, by hand, bypassing launchd entirely:

```bash
./bin/deploy agile-ed
```

## Rollback

Two paths. The fast one is for a bad release you have already noticed; the slow
one is for anything you want on the record.

**Fast — pin the tag.** Set this in `.secrets/agile-ed.env` and deploy:

```sh
AGILE_ED_TAG=8598b8b...    # a :<sha> tag from a previous build
```

The previous sha tag is always published alongside `:main`, so there is always
something to go back to. Unset the variable to follow `:main` again. The CV uses
the same pattern with `EDWARD_WESTON_CV_TAG`; `notion-discord` uses
`NOTION_DISCORD_TAG`.

**Reviewed — revert the compose.** Edit the `image:` line in
`services/agile-ed/compose.yml` back to the sha tag, commit, and let launchd pick
it up. This is the one to use for anything you did not catch in the moment.

Rollback depends on the previous image still being on disk, which means:

> **Never run `docker system prune -a` on the mini.** It deletes untagged images,
> which is exactly the layer a rollback needs. Prune individual dangling images
> if you must, and never with `-a`.

## Add a service

1. Create the image pipeline in the service repo — a workflow whose only job is
   to call this repo's reusable workflow:

   ```yaml
   name: publish image
   on:
     push: { branches: [main] }
     workflow_dispatch:
   jobs:
     publish:
       permissions:              # a called workflow cannot widen this
         contents: read
         packages: write
       uses: geekyed/homelab-deploy/.github/workflows/publish.yml
       with:
         image: ghcr.io/geekyed/<service>
         setup: <install command>
         validate: <gate command>
   ```

   If the service builds static files in an earlier job, add
   `artifact: <artifact-name>` so the reusable workflow unpacks that artifact
   into the build context before `docker build`. The CV workflow is the example.

   A compiled service can run its own compile gate before the publish job and
   still call the reusable workflow with only `image:`. The notion-discord
   workflow is the example.

2. Write `services/<name>/compose.yml` in this repo. Image-only, no `build:`.
   Publish its ports on loopback if a tunnel fronts it.

3. Write `services/<name>.conf` with a distinct `PROJECT` so it gets its own
   network.

4. On the mini: `mkdir -p .secrets`, create `.secrets/<name>.env`, then
   `./bin/register <name> install`.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `deploy: pull failed, leaving the running image in place` | Token expired or package access revoked. `docker login ghcr.io` again on the mini. Nothing was restarted. |
| `deploy: not healthy within the configured timeout` then a rollback | The new image is bad. Check `logs/<name>.err` and `docker compose -p <project> logs <service>`. The old image is already back up. |
| `deploy: nothing to roll back to` | First ever deploy failed. Nothing was running before it, so there is nothing to restore. Fix and re-run. |
| `deploy: missing secrets file` | `.secrets/<name>.env` does not exist on the machine running `deploy`. |
| `deploy: docker not on PATH` | OrbStack is not running, or the mini rebooted without an auto-login. |
| Nothing happens at all | `launchctl list gui/$(id -u)/com.geekyed.<service>.deploy` — if absent, re-run `install`. |
| Deploys stop arriving | The mini was asleep or logged out. Deploys resume on wake; nothing is lost, the tick just runs late. |
| GitHub: `403` on push to the registry | The calling job is missing `permissions: packages: write`. A called workflow can only narrow what the caller grants. |
| GitHub: `uses:` fails to resolve | The reusable workflow must be on the default branch of this repo, and the calling repo must be able to read it. Check this repo's visibility. |

## Known limits

- **macOS and OrbStack only.** `bin/register` writes a LaunchAgent, and the
  service binds `127.0.0.1` for a tunnel on the same host.
- **One registry, one machine.** The credential and the polling model both assume
  GHCR and a single mini.
- **arm64 default.** The reusable workflow builds `linux/arm64` by default, which
  suits an Apple Silicon mini. An Intel machine wants `platform: linux/amd64` at
  the call site.
- **Not a general CI framework.** `publish.yml` can optionally set up Bun to run a
  gate, or unpack a prebuilt artifact from an earlier job. A service in another
  language still needs its own build job and a Dockerfile that packages the
  result. `notion-discord` is the .NET example: a compile job in the service repo,
  then the reusable image build.
- **Config changes span two repos.** A new environment variable is a change in
  the service repo and a change here. `up --wait` fails loudly if a required one
  is missing, so it cannot fail silently.
