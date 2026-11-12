# Home Assistant App: Caddy Custom

Welcome to the Caddy Custom app for Home Assistant!

Caddy is a modern, cross-platform open-source web server and reverse proxy written in Go with automatic HTTPS (Let's Encrypt / ZeroSSL) certificate issuance and renewal built-in.

Unlike standard Caddy distributions, **Caddy Custom** includes `xcaddy` and Go directly inside the container so you can declare any Caddy plugins you need right in the Home Assistant UI configuration—no manual compilation, custom Dockerfiles, or external builds required.

> [!IMPORTANT]
> **Host Networking (`host_network: true`):** This app runs on the host network so Caddy can bind directly to ports `80` and `443` (or any port defined in your `Caddyfile`). Ensure no other app or service on your host is already using the ports you configure.

---

## Installation

The Caddy Custom app is available from the [`datf/hassio-addons`](https://github.com/datf/hassio-addons) custom repository:

1. Navigate in your Home Assistant UI to **Settings** -> **Apps** -> **Install app**.
2. Open the context menu (**⋮**) in the top right corner and select **Repositories**.
3. Add the repository URL:
   ```text
   https://github.com/datf/hassio-addons
   ```
4. Click **Add** and wait for the repository to load.
5. Close the repositories dialog and reload the page if needed.
6. Scroll down to find the **HASS Apps by @datf** repository and click on **Caddy Custom**.
7. Click **Install**.
8. Start the app and check the logs to verify that Caddy started properly.

Out of the box (when no custom `Caddyfile` is provided), Caddy Custom starts with a built-in test `Caddyfile` listening on port `8901` (`auto_https off`) and responds with `Hello, world!`.

---

## Configuration

_Remember to restart the app whenever you change the configuration for the changes to take effect._

Here is an example app configuration:

```yaml
plugins:
  - github.com/caddy-dns/cloudflare
args:
  - "--watch"
env_vars:
  - name: CLOUDFLARE_API_TOKEN
    value: "your_cloudflare_api_token"
log_level: info
```

### Option: `plugins`

A list of Go module paths for Caddy plugins to compile into a custom Caddy binary using `xcaddy`.

Standard plugins, pinned versions, module replacements, and subpackages are all supported:

```yaml
plugins:
  # Standard plugin (defaults to @latest)
  - github.com/caddyserver/replace-response
  # Pinned version
  - github.com/caddyserver/ntlm-transport@v0.1.1
  # Module replacement syntax (forks, patches) and subpackages
  - github.com/hslatman/caddy-crowdsec-bouncer=github.com/datf/caddy-crowdsec-bouncer@v0.0.1
  - github.com/hslatman/caddy-crowdsec-bouncer/http
  - github.com/hslatman/caddy-crowdsec-bouncer/appsec
  - github.com/hslatman/caddy-crowdsec-bouncer/layer4
```

- **Vanilla mode (`plugins: []`):** When no plugins are configured, the app skips compilation and runs the pre-installed vanilla Caddy binary immediately. If you later clear `plugins`, the cached custom binary is automatically removed and Caddy reverts to vanilla.
- **Smart caching & automatic updates:** When plugins are configured, the compiled binary is saved in `/config/caddy-custom` (`addon_config`). Subsequent restarts reuse the cached binary without recompiling unless your `plugins` list changes, a newer upstream Caddy version is released, or the container's Go toolchain updates.
- **Pre-deploy validation:** Before activating a newly compiled binary, the app exports your `env_vars` and runs `caddy validate` against your `Caddyfile`. If validation fails, deployment is aborted so you can inspect the error in the logs.
- **Offline resilience:** If upstream version resolution fails (for example, when temporarily offline) and your plugin configuration has not changed, the app skips rebuilding and continues using your cached binary.

> [!NOTE]
> Compiling plugins on the first startup (or after an update) downloads Go modules and builds Caddy from source, which may take a minute or two depending on your hardware (especially on `aarch64` devices like a Raspberry Pi). Check the app logs to follow build progress.

### Option: `args`

A list of additional CLI arguments passed directly to `caddy run`.

For example, adding `--watch` tells Caddy to automatically reload your `Caddyfile` whenever you save changes to it on disk, without needing to restart the app:

```yaml
args:
  - "--watch"
```

> [!TIP]
> The app automatically passes `--config` pointing to `/config/Caddyfile` (or the built-in `/etc/caddy/Caddyfile` if none exists), so you do not need to specify `--config` in `args`.

### Option: `env_vars`

A list of environment variables (`name` and `value` pairs) exported to **both** the `xcaddy` validation step and the running `caddy` process.

This is the recommended way to keep secrets such as DNS provider API tokens out of your `Caddyfile`. Inside your `Caddyfile`, reference variables using `{env.VAR_NAME}` (runtime) or `{$VAR_NAME}` (parse time):

```yaml
env_vars:
  - name: CLOUDFLARE_API_TOKEN
    value: "your_cloudflare_api_token"
```

### Option: `log_level`

The `log_level` option controls the verbosity of the app's startup and supervision script logs, which is especially helpful when troubleshooting plugin resolution or `args` passing.

Possible values are:

- `trace`: Show every detail, including internal helper calls.
- `debug`: Show detailed debug information (such as resolved paths, `args`, and subpackage detection).
- `info`: Normal interesting events (default and recommended for daily use).
- `notice`: Normal but significant events.
- `warning`: Exceptional occurrences that are not errors.
- `error`: Runtime errors that do not require immediate action.
- `fatal`: Something went terribly wrong; the app becomes unusable.

---

## Storage and File Locations

Caddy Custom maps three Home Assistant directories so your configuration, compiled binary, and TLS certificates persist safely across restarts and updates:

| Container Path                | Home Assistant Mapping      | Purpose                                                                                                                                   |
| :---------------------------- | :-------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| `/config` (`XDG_CONFIG_HOME`) | `addon_config` (read/write) | Place your custom `Caddyfile` here (`/config/Caddyfile`). Also persists the compiled `/config/caddy-custom` binary and build state files. |
| `/ssl` (`XDG_DATA_HOME`)      | `ssl` (read/write)          | Caddy's data directory, storing Let's Encrypt / ZeroSSL certificates, private keys, and local PKI authorities.                            |
| `/share`                      | `share` (read-only)         | Shared Home Assistant directory (for example, to serve static files or read shared certificates).                                         |

To provide your own configuration, create a file named `Caddyfile` inside the **`addon_config` directory for Caddy Custom** (on your Home Assistant host under `/addon_configs/<repo_prefix>_caddy-custom/Caddyfile`, for example `/addon_configs/91f8a72d_caddy-custom/Caddyfile` when installed from `datf/hassio-addons`).

If `/config/Caddyfile` does not exist, Caddy Custom falls back to the built-in `/etc/caddy/Caddyfile`:

```caddy
{
    auto_https off
}

:8901 {
    respond "Hello, world!"
}
```

---

## `Caddyfile` Examples

### 1. Reverse Proxying Home Assistant (Automatic HTTPS)

When running Home Assistant behind a reverse proxy, you must configure Home Assistant's `http` integration to trust forwarded headers (`X-Forwarded-For`) from Caddy.

In your Home Assistant `configuration.yaml`, add:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
    - 172.30.32.0/23
```

Restart Home Assistant for this change to take effect.

Then create `/config/Caddyfile` in Caddy Custom's `addon_config` directory:

```caddy
https://homeassistant.example.com {
    reverse_proxy 127.0.0.1:8123
}
```

> [!NOTE]
> Because Caddy Custom uses `host_network: true`, proxying to `127.0.0.1:8123` reaches Home Assistant directly over loopback (matching `127.0.0.1` in `trusted_proxies`). Make sure ports `80` and `443` are forwarded to your Home Assistant host so Caddy can obtain and renew certificates automatically.

### 2. Automatic HTTPS with a DNS-01 Plugin (e.g., Cloudflare)

If port `80` is blocked by your ISP or you do not want to expose it publicly, you can use a Caddy DNS plugin to obtain certificates via DNS-01 challenges.

1. Add the DNS plugin and token in the app **Configuration** tab:

   ```yaml
   plugins:
     - github.com/caddy-dns/cloudflare
   env_vars:
     - name: CLOUDFLARE_API_TOKEN
       value: "your_cloudflare_api_token"
   ```

2. Reference the token in your `Caddyfile`:

   ```caddy
   https://homeassistant.example.com {
       tls {
           dns cloudflare {env.CLOUDFLARE_API_TOKEN}
       }
       reverse_proxy 127.0.0.1:8123
   }
   ```

3. Restart the app. Caddy Custom will compile the Cloudflare DNS module, validate your `Caddyfile`, and obtain your TLS certificate via DNS-01.

### 3. Serving Static Files from `/share`

Because `/share` is mounted read-only into the container, you can serve static files directly from your Home Assistant `/share` folder:

```caddy
:8080 {
    root * /share/www
    file_server
}
```

---

## Hardening Caddy with CrowdSec

Exposing Home Assistant or other self-hosted services behind a reverse proxy makes them a target for automated vulnerability scanners, brute-force logins, and exploit bots.

If you want to harden your Caddy setup, you can pair **Caddy Custom** with the companion **[CrowdSec Home Assistant App](https://github.com/datf/app-crowdsec)** from the same [`datf/hassio-addons`](https://github.com/datf/hassio-addons) repository.

Combining Caddy Custom with CrowdSec gives you multi-layered protection:

- **Automated IP Blocking (CrowdSec Bouncer):** Analyzes Caddy access logs and Home Assistant logs to ban malicious IPs, while proactively blocking known attackers from the global CrowdSec community blocklist.
- **In-line WAF (`AppSec`):** Inspects incoming HTTP requests in real time against OWASP Top 10 rules and virtual patches—blocking SQL injection, XSS, path traversal, and exploit probes on the very first request.
- **Layer 4 Protection (Optional):** Supports `caddy-l4` bouncer enforcement if you proxy raw TCP/UDP streams.

### Quick Overview

1. **Enable the CrowdSec bouncer modules** in your **Caddy Custom** configuration:

   ```yaml
   plugins:
     - github.com/hslatman/caddy-crowdsec-bouncer/http
     - github.com/hslatman/caddy-crowdsec-bouncer/appsec
   ```

2. **Install and configure CrowdSec** following the complete step-by-step walkthrough in **[`datf/app-crowdsec`](https://github.com/datf/app-crowdsec)** (covering journald log acquisition for `app_91f8a72d_caddy-custom`, generating a bouncer API key with `cscli`, enabling AppSec WAF collections, and testing blocks).

3. **Add `crowdsec` and `appsec` directives** to your `Caddyfile`:

   ```caddy
   {
       crowdsec {
           api_url http://91f8a72d-crowdsec:8080
           api_key <YOUR_GENERATED_API_KEY>
           ticker_interval 15s
           lapi_timeout 10s
           appsec_url http://91f8a72d-crowdsec:7422
       }
   }

   https://homeassistant.example.com {
       route {
           crowdsec
           appsec
           reverse_proxy 127.0.0.1:8123
       }
   }
   ```

For full instructions, troubleshooting, and `cscli` commands, see **[https://github.com/datf/app-crowdsec](https://github.com/datf/app-crowdsec)**.

---

## Troubleshooting & Useful Commands

### Inspecting Compiled Modules & Caddy Version

If you have the **Advanced SSH & Web Terminal** app installed (with **Protection Mode** disabled), you can inspect your running Caddy binary directly:

```bash
# Check compiled custom modules
docker exec -it app_91f8a72d_caddy-custom /config/caddy-custom list-modules

# Check Caddy version (vanilla binary)
docker exec -it app_91f8a72d_caddy-custom /app/caddy version

# Manually validate your Caddyfile
docker exec -it app_91f8a72d_caddy-custom /config/caddy-custom validate --config /config/Caddyfile
```

> [!TIP]
> If you installed Caddy Custom from a forked or local repository, your container name may have a different prefix than `app_91f8a72d_caddy-custom`. Run `docker ps` to find the exact container name.

### Common Issues

- **Build fails during startup:** Set `log_level: debug` in the app configuration and restart to see detailed Go module resolution and `xcaddy` output in the logs. Verify the plugin module path matches a valid Go module and satisfies the schema format.
- **Validation error after changing plugins:** Caddy Custom validates your `/config/Caddyfile` before deploying the newly built binary. Check the app logs for the exact `Caddyfile` syntax or directive error.
- **Port already in use:** Because `host_network: true` is active, stop any other service listening on ports `80` or `443` (or change the listening port in your `Caddyfile`).
- **Additional resources:** For full `Caddyfile` directive syntax and module documentation, refer to the [official Caddy documentation](https://caddyserver.com/docs/).
