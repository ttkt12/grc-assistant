# Deployment — Zalopay Agent Base

GRC Assistant deploys to **Zalopay Agent Base** (a Coolify instance at
`dev-coolify.zalopay.xyz`). The previous GreenNode AgentBase path has been removed.

Deployment is driven by the `zlp-agentbase-*` Claude Code skills, not by scripts in this
repository. Install them per the Agent Base user guide; `zlp-agentbase-init` must report
`API_CHECK=PASS` before anything else.

## Prerequisites

| Requirement | Notes |
|---|---|
| `zlp-agentbase-*` skills in `~/.claude/skills/` | 5 skills: init, deploy, status, updatedomain, delete-app |
| `~/.claude/zlpagentbase.env`, `chmod 600` | Issued by SRE. Holds `AGENTBASE_*` only — platform credentials, never app config |
| Node.js LTS | The Agent Base MCP server runs under `npx` |
| `vector_db/` present locally | See **The index problem** below |
| `.env` with a real `AI_PLATFORM_API_KEY` | The app calls an OpenAI-compatible LLM; Agent Base does not provide one |

`~/.claude/zlpagentbase.env` and this repository's `.env` share **no variables**. The first
authenticates the deploy tooling; the second configures the running app. Adding app config
to the SRE file has no effect — `config.py` and `teams_bot.py` call `load_dotenv()`, which
reads the project `.env`.

## The index problem — read before deploying

`Dockerfile` uses `COPY . .`, so `vector_db/` is baked into the image. But `vector_db/` is
gitignored (it holds embeddings derived from internal ISMS documents).

That matters because of how the deploy skill picks a build pack:

| Local Docker | Build pack | Where the image is built | Does `vector_db/` reach it? |
|---|---|---|---|
| available | `dockerimage` | your machine, pushed to the registry | **Yes** — the local working tree is the build context |
| **not** available | `dockerfile` | the Agent Base server, from the git repository | **No** — gitignored files are not in the repo |

Without local Docker the deployed image has no index, and
`validate_startup_config()` in `teams_bot.py` raises `RuntimeError: Vector database not
found`. The container exits immediately and the domain returns 502/503 even though the
build step reported success.

Run `python predeploy_check.py` first; it fails on a missing index before anything ships.

Three ways out, none free:

1. **Install Docker (or colima/podman) locally.** Restores the `dockerimage` path. Smallest
   change.
2. **Track `vector_db/` in git.** Works immediately, but publishes embeddings of internal
   documents — only defensible if the repository is private, and it contradicts
   `SHAREPOINT_AUTO_UPDATE_PIPELINE.md`. Never do this while the repo is public.
3. **Load the index at startup** from object storage or a mounted volume instead of baking
   it into the image, with an admin endpoint to reload it. Correct architecture, and the
   only option that also enables scheduled knowledge refreshes, but it needs code changes
   and storage that Agent Base dev may not offer. Confirm with SRE.

## App naming

The app name comes from the **directory name**, automatically. Agent Base prefixes it with
the account name, so this project deploys as:

```
aiweek_grc-assistant
```

Never rename the `grc-assistant` directory. A renamed directory is treated as a brand new
application rather than an update, leaving a second copy behind.

`AGENTBASE_USER_NAME` is `aiweek`, a **shared** account. The token can read, modify and
delete every application under it, including other teams'. The platform will not stop a
mistaken delete, so verify the full resolved app name before any destructive action.

## Environment variables pushed to the container

The deploy skill uploads a `.env` file as the application's environment variables via
`scripts/load-env.sh --env-file=<path>`. Because the path is explicit, point it at a
deploy-safe file rather than the local `.env`.

**`PORT` and `TEAMS_BOT_PORT` must not be uploaded.** The container must listen on 8080.
`teams_bot.py` resolves its port as:

```python
port = int(os.getenv("PORT") or os.getenv("TEAMS_BOT_PORT", "3978"))
```

`Dockerfile` sets `ENV PORT=8080`, but a platform environment variable overrides it. A
local `.env` carrying `PORT=3978` therefore makes the app listen on the wrong port, and the
proxy returns 502/503.

Exclude from the uploaded file:

```text
PORT
TEAMS_BOT_PORT
TEAMS_BOT_HOST
```

`EXPOSE 8080` in the Dockerfile is detected automatically (`EXPOSE_PORTS=8080`), so the
skill will not ask which port to use.

## Basic auth breaks the Teams bot

The deploy skill offers to put HTTP basic auth in front of the app. Accepting it blocks
`POST /api/messages`: the Bot Framework authenticates with its own JWT and cannot send
basic-auth credentials, so Teams messages are rejected at the proxy and the bot silently
stops replying.

Choose one:

- **Keep Teams** → decline basic auth. Protect the web API with the app's own check:
  `REQUIRE_APP_ACCESS_TOKEN=true` plus `APP_ACCESS_TOKEN`, which deliberately exempts
  `/api/messages`, `/`, `/health` and static assets.
- **Web only** → accept basic auth and accept that Teams will not work.

## Teams bot credentials

`MICROSOFT_APP_TYPE` must be `SingleTenant`, matching the Azure Bot registration, and
`MICROSOFT_APP_TENANT_ID` must be set. `channel_auth_tenant()` scopes incoming-token
validation to the tenant only for single-tenant apps; a multi-tenant bot must leave it
unset. Getting this pair wrong makes every incoming request fail validation with 401 and
the bot never replies — with no error in the app logs.

`MS_CLIENT_ID` / `MS_CLIENT_SECRET` belong to the **SharePoint Graph** app registration,
not the bot. They are unrelated to `MICROSOFT_APP_*` and must not be swapped.

## Resource limits

Agent Base dev pins every application to **2 CPU / 4 GB RAM**, standalone, with no
rollback. The app loads a `sentence-transformers` embedding model plus the FAISS index at
startup and takes roughly 60–90 seconds before it serves traffic. Give the health check a
grace period long enough to cover that, or the platform will treat a healthy start as a
failure.

## Platform resource names keep the old brand

The repository was renamed SecureMind RAG → GRC Assistant, but registered resources keep
their original names **on purpose**. Renaming them breaks things:

- Azure Bot: `SecureMind_RAG`
- Teams manifest `id` and `botId` (GUIDs): changing either breaks the app already
  installed in Teams for existing users

## Teams manifest domain

`teams_app/manifest.json` still carries the retired GreenNode runtime hostname in
`websiteUrl`, `privacyUrl`, `termsOfUseUrl` and `validDomains`. Agent Base assigns a fresh
domain (`https://<generated>.ai.zalopay.xyz`) at first deploy, so the manifest can only be
corrected afterwards:

```bash
python scripts/package_teams_app.py --domain "<generated>.ai.zalopay.xyz"
```

Then re-upload the package in Teams. Until that is done the bot may work while the app's
links point at a dead host.

## Order of operations

1. `python predeploy_check.py` — must pass, especially the `vector_db` check
2. Run the app locally (`python teams_bot.py`) and confirm it answers a content question
3. Prepare the deploy-safe env file (no `PORT` / `TEAMS_BOT_PORT`)
4. Deploy with the `zlp-agentbase-deploy` skill; decline basic auth if Teams is needed
5. Check with the `zlp-agentbase-status` skill; `running` with HTTP 200/401/404 is healthy,
   while `running` with 502/503 usually means the wrong port or a startup crash
6. Update the Teams manifest domain and re-upload
7. Verify the live endpoint: `python scripts/production_verify.py --base-url <domain>`

Deploying before step 2 passes means debugging the app and the platform at the same time.
