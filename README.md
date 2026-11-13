# ns8-mautic

[![Build Status](https://github.com/geniusdynamics/ns8-mautic/workflows/test-module/badge.svg)](https://github.com/geniusdynamics/ns8-mautic/actions)

Mautic module for NethServer 8 (NS8). This repository packages Mautic, an open-source marketing automation platform, with a MariaDB database and exposes it through the NS8 gateway (Traefik) with optional Let's Encrypt TLS.

- **Image**: `ghcr.io/geniusdynamics/mautic:latest`
- **Orchestrator**: NS8 modules (podman-based)
- **Status**: Stable for evaluation and lab use

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
- [Update the Module](#update-the-module)
- [Uninstall](#uninstall)
- [Smarthost Discovery (SMTP)](#smarthost-discovery-smtp)
- [Debugging](#debugging)
- [Testing](#testing)
- [UI Development and Translation](#ui-development-and-translation)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Overview

This module integrates Mautic into NethServer 8, providing a complete marketing automation solution. It includes:

- Mautic application container
- MariaDB database for data persistence
- Traefik reverse proxy for secure access
- Optional Let's Encrypt SSL certificates
- UI components for NS8 management interface

The module supports dynamic SMTP configuration via smarthost discovery and includes comprehensive testing with Robot Framework.

---

## Prerequisites

Before installing the ns8-mautic module, ensure you have:

- A running NethServer 8 (NS8) node with admin access
- A public DNS record pointing to your NS8 gateway (e.g., `mautic.example.com`)
- Ports 80 and 443 reachable from the internet if using Let's Encrypt for automatic HTTPS
- Basic familiarity with NS8 module management and Podman containers

## Quick Start

Follow these steps to install, configure, and access Mautic on NS8:

### 1. Install the Module

Install the module image and create an instance:

```bash
add-module ghcr.io/geniusdynamics/mautic:latest 1
```

This command returns a JSON object. Note the `module_id`, for example:

```json
{
  "module_id": "mautic1",
  "image_name": "mautic",
  "image_url": "ghcr.io/geniusdynamics/mautic:latest"
}
```

### 2. Configure the Instance

Replace `mautic1` with your instance ID and adjust the domain and options:

```bash
api-cli run configure-module --agent module/mautic1 --data - <<'EOF'
{
  "host": "mautic.example.com",
  "http2https": true,
  "lets_encrypt": false
}
EOF
```

This configuration:

- Starts and configures the Mautic instance
- Sets up a virtual host for Traefik to route HTTP/HTTPS traffic to Mautic
- Enables HTTP to HTTPS redirection if `http2https` is true

### 3. Verify Configuration

Get the current configuration to ensure everything is set up correctly:

```bash
api-cli run get-configuration --agent module/mautic1
```

### 4. Access the Application

- **URL**: `http(s)://mautic.example.com`
- If `lets_encrypt` is `true`, a certificate will be requested automatically
- If `lets_encrypt` is `false`, provide your own TLS certificate or use HTTP for internal testing
- Set `http2https` to `true` if you plan to terminate TLS at the NS8 gateway

Once configured, you can log in to Mautic using the default credentials (check Mautic documentation for initial setup).

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

For detailed UI development instructions, see the [NS8 UI Developer Manual](https://nethserver.github.io/ns8-core/ui/modules/#module-ui-development).

### Translation Management

Translations are managed on Weblate: https://hosted.weblate.org/projects/ns8/

To set up translation syncing:

1. Install the GitHub Weblate app: https://docs.weblate.org/en/latest/admin/continuous.html#github-setup
2. Add the repository on https://hosted.weblate.org (or ask a NethServer developer to include it in the NS8 Weblate project)

Translation files are located in `ui/public/i18n/` and support multiple languages including English, German, Spanish, and more.

---

## Troubleshooting tips

- DNS and certificates: ensure the host value has a valid public DNS A/AAAA record pointing to your NS8 gateway before enabling Let's Encrypt
- HTTP to HTTPS redirection: set http2https to true when a certificate is available at the gateway
- Connectivity: verify that port mappings and the Traefik route are active; use podman ps and journal logs

---

## Contributing

Issues and pull requests are welcome! When submitting changes, please include:

- A short description of the change and the motivation
- Test notes (how you verified the change)
- Any related documentation updates

### Development Workflow

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Make your changes and test thoroughly
4. Run tests: `./test-module.sh <NODE_ADDR> ghcr.io/geniusdynamics/mautic:latest`
5. Submit a pull request with a clear description

## License

This project is licensed under the GPL-3.0 License. See the [LICENSE](LICENSE) file for details.
