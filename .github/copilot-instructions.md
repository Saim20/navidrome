<!-- .github/copilot-instructions.md - guidance for AI coding agents working in this repo -->
# Copilot instructions for the Navidrome Docker deployment

Short, focused guidance to help an AI coding agent be productive editing this repository which contains a small Docker-based Navidrome deployment.

Key points
- Purpose: this repository runs Navidrome (a lightweight music server) inside Docker via `docker-compose.yml`, fronted by an Nginx reverse proxy with optional TLS (`nginx-ssl.conf`). The `data/` directory holds Navidrome persistent data, `music/` holds the music library. A filebrowser service provides remote upload access to the music folder on port 8080.
- Important files:
  - `docker-compose.yml` — declares the `navidrome` service using image `deluan/navidrome:latest`, binds port 4533, mounts `./data` and `./music`, and exposes a handful of ND_ environment variables. Also includes `filebrowser` service for file uploads.
  - `nginx-ssl.conf` — reverse-proxy configuration that forwards HTTPS traffic to `http://navidrome:4533` and expects certificates under `/etc/nginx/certs`.
  - `data/README.md`, `music/README.md` — describe the purpose of those mounts.

What the agent should assume
- This is an infra/deployment repo, not an application source tree. Changes should avoid breaking existing Docker or Nginx behavior.
- The compose file is authoritative for runtime behavior (ports, mounts, env vars). Update it when changing container-level config.
- Nginx config is minimal and is intended to be used inside another container that has the `navidrome` service available on the Docker network.

Common tasks and exact patterns
- Add new environment variables to Navidrome: update `docker-compose.yml` under `environment:` for the `navidrome` service. Example: to change log level add `- ND_LOGLEVEL=debug`.
- Change the exposed port: edit the `ports:` mapping (`HOST:CONTAINER`) in `docker-compose.yml`. Default container port is `4533`.
- Persist additional host directories: add new `volumes:` under the service. Use relative paths (e.g., `./data:/data`) to keep behavior consistent.
- TLS certs for Nginx: `nginx-ssl.conf` references `/etc/nginx/certs/fullchain.pem` and `privkey.pem`. Ensure the container running Nginx mounts a host folder or secret there.
- Configure file server authentication: edit `docker-compose.yml` under `environment:` for the `filebrowser` service. Set `FB_NOAUTH=false` and add `FB_USERNAME`/`FB_PASSWORD` for basic auth.
- Change file server port: edit the `ports:` mapping in `docker-compose.yml` for the `filebrowser` service (default host port is `8080`).

Patterns to follow when editing
- Minimal, incremental edits. This repo is configuration-first; avoid large refactors.
- Preserve existing YAML/NGINX formatting and comments. When adding keys to `docker-compose.yml`, keep lists aligned and use the existing style (short `- KEY=VALUE` environment entries).
- When adding features, prefer adding environment flags or new volume mounts rather than replacing the base image.

Testing and validation steps the agent should add or run after changes
- After edits, recommend these manual checks (document them in PRs):
  1. Lint YAML (basic): ensure `docker-compose.yml` is valid YAML.
  2. Dry-run compose: `docker compose config` to validate merged config.
  3. Start services locally: `docker compose up -d` and check container logs `docker compose logs -f navidrome`.
  4. If Nginx is changed, validate config inside the Nginx container: `nginx -t` and check that cert files are mounted.
  5. If filebrowser is changed, access http://localhost:8080 to verify the web interface loads and allows file uploads to the music folder.

Examples from the repo
- To add an env var (change log level), modify:

  services:
    navidrome:
      environment:
        - ND_LOGLEVEL=info

  -> change to `- ND_LOGLEVEL=debug` to increase verbosity.

- To proxy a different upstream path, edit `nginx-ssl.conf`'s `proxy_pass http://navidrome:4533;` line. Keep the four `proxy_set_header` lines; they are required for correct header forwarding.

Risks and guardrails
- Don't modify `music/` or `data/` content programmatically; these are user data folders. Changes that would move or delete files there are out-of-scope for routine infra edits.
- When bumping images, prefer explicit image tags in future PRs rather than `latest` to avoid unexpected runtime changes.

When to ask the human
- If a config change requires new secrets or certificate provisioning, ask for the intended secret management approach (host mount, Docker secrets, or an external vault).
- If a change affects user data layout under `data/` or `music/`, confirm backup and migration plans before editing.

Files of highest interest for reviewers
- `docker-compose.yml` — runtime config and the single place to change container behavior.
- `nginx-ssl.conf` — TLS and proxy settings. Keep proxied headers as-is.

If unclear or incomplete, request these specifics from the maintainer:
- Which environment variables do you want persisted and why?
- Do you want pinned image tags instead of `latest`?
- How should TLS certs be provisioned (host path, Docker secret, Let's Encrypt automation)?

End of file.
