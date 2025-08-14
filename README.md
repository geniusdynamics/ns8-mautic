# ns8-mautic

Mautic module for NethServer 8 (NS8). This repository packages Mautic with a MariaDB database and exposes it through the NS8 gateway (Traefik) with optional Let's Encrypt TLS.

- Image: ghcr.io/geniusdynamics/mautic:latest
- Orchestrator: NS8 modules (podman-based)
- Status: stable for evaluation and lab use

## Contents

- Quick start (install, configure, access)
- Configuration reference and examples
- Update, get configuration, and uninstall
- Smarthost discovery (SMTP)
- Debugging tips (runagent, podman)
- Testing (Robot Framework)
- UI development and translation

---

## Quick start

Prerequisites:

- A running NS8 node with admin access
- A public DNS record pointing to your NS8 gateway (for example: mautic.example.com)
- Ports 80 and 443 reachable if you want automatic HTTPS via Let's Encrypt

1) Install the module image and create an instance

```bash
add-module ghcr.io/geniusdynamics/mautic:latest 1
```

This command returns a JSON object. Note the module_id, for example:

```json
{"module_id": "mautic1", "image_name": "mautic", "image_url": "ghcr.io/geniusdynamics/mautic:latest"}
```

2) Configure the instance

Replace mautic1 with your instance id and adjust the domain and options.

```bash
api-cli run configure-module --agent module/mautic1 --data - <<'EOF'
{
  "host": "mautic.example.com",
  "http2https": true,
  "lets_encrypt": false
}
EOF
```

What this does:

- Starts and configures the Mautic instance
- Configures a virtual host for Traefik to route HTTP/HTTPS traffic to Mautic

3) Get the current configuration

```bash
api-cli run get-configuration --agent module/mautic1
```

4) Access the application

- URL: http(s)://mautic.example.com
- If lets_encrypt is true, a certificate request will be performed automatically
- If lets_encrypt is false, provide your own TLS or keep HTTP for internal testing; set http2https to true if you plan to terminate TLS at the NS8 gateway

---

## Configuration reference

These inputs are accepted by configure-module:

- host (string, required): Fully qualified domain name for public access
- http2https (boolean, default: true): Redirect plain HTTP to HTTPS at the gateway
- lets_encrypt (boolean, default: false): Issue a Let's Encrypt certificate for host

Example without Let's Encrypt (internal lab):

```bash
api-cli run configure-module --agent module/mautic1 --data - <<'EOF'
{
  "host": "mautic.lab.local",
  "http2https": false,
  "lets_encrypt": false
}
EOF
```

Example with Let's Encrypt:

```bash
api-cli run configure-module --agent module/mautic1 --data - <<'EOF'
{
  "host": "mautic.example.com",
  "http2https": true,
  "lets_encrypt": true
}
EOF
```

---

## Update the module

To update the instance to the latest image:

```bash
api-cli run update-module --data '{"module_url":"ghcr.io/geniusdynamics/mautic:latest","instances":["mautic1"],"force":true}'
```

---

## Uninstall

To remove an instance and its data:

```bash
remove-module --no-preserve mautic1
```

Caution: --no-preserve deletes the instance data.

---

## Smarthost discovery (SMTP)

Some configuration, like outbound SMTP (smarthost), is not passed directly to configure-module. Instead, it is discovered dynamically through Redis keys.

On every Mautic start, the command bin/discover-smarthost refreshes state/smarthost.env with values from Redis to keep the module aligned with the centralized smarthost setup:

- Docs: https://geniusdynamics.github.io/ns8-core/core/smarthost/
- If the smarthost is changed while Mautic is running, the handler events/smarthost-changed/10reload_services restarts the main module service
- See also systemd/user/mautic.service

This mechanism is an example implementation and can be adapted or replaced.

---

## Debugging

Useful commands when working with an instance named mautic1.

- Inspect environment from the root terminal (loads agent env from /home/mautic1/.config/state):

```bash
runagent -m mautic1 env
```

- Enter the agent environment for testing scripts and inheriting module variables:

```bash
runagent -m mautic1
```

Your PATH will include:

```
/home/mautic1/.config/bin:/usr/local/agent/pyenv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/usr/
```

- Inspect containers:

```bash
podman ps
# Example output
# CONTAINER ID  IMAGE                                      COMMAND               STATUS        NAMES
# ...           docker.io/library/mariadb:10.11.5          --character-set-s...  Up           mariadb-app
# ...           docker.io/library/nginx:stable-alpine3.17  nginx -g daemon o...  Up           mautic-app
```

- Print environment variables inside the app container:

```bash
podman exec mautic-app env
```

- Open a shell in the app container:

```bash
podman exec -ti mautic-app sh
```

---

## Testing

Run the end-to-end tests with Robot Framework using the helper script:

```bash
./test-module.sh <NODE_ADDR> ghcr.io/geniusdynamics/mautic:latest
```

See: https://robotframework.org/

---

## UI development and translation

- UI development: see the Developer manual
  https://nethserver.github.io/ns8-core/ui/modules/#module-ui-development
- Translations are managed on Weblate: https://hosted.weblate.org/projects/ns8/

To set up translation syncing:

- Install the GitHub Weblate app: https://docs.weblate.org/en/latest/admin/continuous.html#github-setup
- Add the repository on https://hosted.weblate.org (or ask a NethServer developer to include it in the NS8 Weblate project)

---

## Troubleshooting tips

- DNS and certificates: ensure the host value has a valid public DNS A/AAAA record pointing to your NS8 gateway before enabling Let's Encrypt
- HTTP to HTTPS redirection: set http2https to true when a certificate is available at the gateway
- Connectivity: verify that port mappings and the Traefik route are active; use podman ps and journal logs

---

## Contributing

Issues and PRs are welcome. When submitting changes, please include:

- A short description of the change and the motivation
- Test notes (how you verified the change)
- Any related documentation updates
