# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Role

Apply the role (or specific tags) against an inventory:

```bash
# Full role
ansible-playbook site.yml --tags nginx

# Only prepare base config
ansible-playbook site.yml --tags prepare

# Only sites/upstreams
ansible-playbook site.yml --tags sites

# Only streams
ansible-playbook site.yml --tags streams

# Only auth (htpasswd)
ansible-playbook site.yml --tags auth

# Validate nginx config without reloading
ansible -m command -a '/usr/sbin/nginx -t' <host>
```

## Role Architecture

This is a standard Ansible role for `nginx-extras` (Debian/Ubuntu). The role is structured around four independent blocks in `tasks/main.yml`, each with its own tag:

- **`prepare`** — deploys `nginx.conf` from template, copies `conf.d/` addons (json log formats, etc.), copies the `ffdhe4096.pem` DH param file, and disables the default site.
- **`auth`** — creates htpasswd files from `nginx_auth` list (requires `python-passlib`).
- **`sites`** — renders HTTP/HTTPS virtual host configs and upstream definitions into `sites-available/`, then symlinks them into `sites-enabled/`.
- **`streams`** — same pattern as sites but for TCP/UDP `stream {}` blocks; creates `streams-available/` and `streams-enabled/` directories on demand.

### Template data model

Sites and streams use a list-of-dicts variable structure. Key fields:

**`nginx_sites`** / **`nginx_streams`** (list):
- `name` — filename stem (e.g. `myapp` → `myapp.site`)
- `options` — list of raw nginx directives placed inside `server {}`
- `locations` — list of `{name, options[], ifs[]}` blocks (sites only)
- `extras` — list of directives placed *outside* `server {}` (e.g. `map` blocks)

**`nginx_sites_upstreams`** / **`nginx_streams_upstreams`** (list):
- `name` — upstream block name
- `servers` — list of explicit `server` addresses
- `groups` — list of Ansible inventory group names (servers resolved at template time via `groups[group]`)
- `ports` — port appended to each group-resolved server
- `options` — per-server options (e.g. `weight=5`)

**`nginx_auth`** (list):
- `file` — htpasswd filename under `/etc/nginx/htpasswd/`
- `credentials` — list of `{user, pass}`

### Handlers

Two-stage: `check and restart/reload nginx` runs `nginx -t` first and only triggers the actual `restart nginx` / `reload nginx` handler if the config test passes.

### Dependencies (`meta/main.yml`)

- Role `apt` is called to install `nginx-common`, `nginx-extras`, and `libssl1.0.0` (backports on Jessie).
- Role `apt` installs `python-passlib` when `nginx_auth` is defined.
- Role `certs` is invoked when `nginx_ssl_cert` or `nginx_ssl_pkey` is defined, setting cert ownership to `nginx_user`.
- Role `shorewall` is invoked when `nginx_shorewall` is defined (firewall rules).

### Static files (`files/`)

- `etc/nginx/conf.d/json_log.conf` — defines `log_format json` with extensive nginx variables.
- `etc/nginx/conf.d/json_with_request_body_log.conf` — same format plus `$request_body`.
- `etc/nginx/ffdhe4096.pem` — pre-generated RFC 7919 DH group for `ssl_dhparam`.

### Key defaults to know

- `nginx_version: 1.10.*` — pinned; change to upgrade.
- `nginx_ssl_protocols` defaults to `TLSv1 TLSv1.1 TLSv1.2` (outdated; consider restricting to TLSv1.2/1.3).
- HSTS is enabled by default with `max-age=31536000`, `includeSubDomains`, and `preload`.
- `stream {}` block is only emitted in `nginx.conf` when `nginx_streams` is defined.
