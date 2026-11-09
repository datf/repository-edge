# Home Assistant App: CrowdSec

CrowdSec is a free, open-source, and collaborative Intrusion Prevention System (IPS) and Security Engine. It inspects incoming events from logs (Home Assistant logins, systemd journal logs, and reverse proxies like Caddy), detects aggressive behaviors and attack patterns (such as brute force, port scans, web crawlers, and vulnerability exploits), and triggers automated remediation (IP bans, captcha challenges, or request blocking).

CrowdSec also connects to a decentralized global threat intelligence network. When an attack is detected locally, anonymized signals can be shared with the CrowdSec Central API, while your instance automatically receives the curated global community blocklist to proactively shield your Home Assistant setup from known bad actors worldwide.

---

## Installation

The CrowdSec App is available as a custom add-on from the [`datf/hassio-addons`](https://github.com/datf/hassio-addons) repository:

1. Navigate in your Home Assistant UI to **Settings** -> **Add-ons** -> **Add-on Store**.
2. Click on the three dots (**⋮**) in the top right corner and select **Repositories**.
3. Add the repository URL:
   ```text
   https://github.com/datf/hassio-addons
   ```
4. Click **Add** and wait for the repository to load.
5. Close the repositories dialog and reload the page if needed.
6. Scroll down to find the **HASS Apps by @datf** repository and click on **CrowdSec**.
7. Click **Install**.
8. Review the configuration options, then proceed to the Quick Start below to connect your log sources and bouncers.
9. Start the app and check the logs to verify that the CrowdSec service started properly.

---

## Quick Start: IP Blocking with Caddy Reverse Proxy

CrowdSec acts as the detection and decision engine (Local API). To actively block malicious traffic before it reaches Home Assistant, you pair CrowdSec with a **Remediation Component (Bouncer)**.

Caddy is a great HTTP(S) server and reverse proxy with automatic HTTPS certificate renewal that works great with CrowdSec when paired with the **Caddy 2 Custom** add-on and its CrowdSec HTTP bouncer module.

---

### Step 1: Install Caddy Custom Add-on

From the same [`datf/hassio-addons`](https://github.com/datf/hassio-addons) repository, install the **Caddy 2 Custom** add-on ([`datf/hassio-caddy-custom`](https://github.com/datf/hassio-caddy-custom)).

Unlike standard reverse proxy add-ons, Caddy Custom can dynamically compile Go plugins on the fly using `xcaddy`.

In the **Caddy Custom** add-on configuration tab, declare the CrowdSec bouncer plugin under `plugins`:

```yaml
plugins:
  - github.com/hslatman/caddy-crowdsec-bouncer/http
```

Save your configuration. Caddy will compile this plugin on its next startup.

---

### Step 2: Install Advanced SSH & Web Terminal

To manage CrowdSec with its command-line tool (`cscli`) and create configuration files, we use Home Assistant's **Advanced SSH & Web Terminal** add-on.

> [!NOTE]
> **Why not an embedded terminal like the official add-on?**
> The official CrowdSec add-on bundles `ttyd` to run a web terminal inside the container itself. However, running a persistent web terminal daemon 24/7 consumes extra memory and system resources. In practice, you only need console access during initial configuration or occasional maintenance. Keeping this add-on lightweight and using the community **Advanced SSH & Web Terminal** gives you full CLI control without the unnecessary background overhead.

#### Setup:

1. In the Home Assistant **Add-on Store**, find **Advanced SSH & Web Terminal** (available by default from the official Community Add-ons repository).
2. Click **Install**.
3. In its configuration/info tab, disable **Protection Mode** (required so you can access Docker containers via `docker exec`).
4. Configure your credentials/password and start the add-on.
5. Open its **Web UI** (or connect via SSH).

#### Shell Prompt Legend:

Throughout this guide, commands are prefixed to show where they should be executed:

- `homeassistant$` — Run in the Home Assistant terminal (Advanced SSH).
- `crowdsec#` — Run inside the CrowdSec container shell.
- `pc$` — Run on your local computer or external test machine.

To enter the CrowdSec container shell from Home Assistant:

```bash
homeassistant$ docker exec -it app_91f8a72d_crowdsec bash
crowdsec#
```

_(To return to Home Assistant, simply type `exit`.)_

---

### Step 3: Configure Caddy Log Acquisition in CrowdSec

To enable CrowdSec to read Caddy's access logs directly from `systemd-journald`:

1. From your Home Assistant terminal, create the acquisition directory and configuration file:

```bash
homeassistant$ mkdir -p /config/.storage/etc/crowdsec/acquis.d
homeassistant$ nano /config/.storage/etc/crowdsec/acquis.d/caddy.yaml
```

2. Paste the following configuration:

```yaml
---
source: journalctl
journalctl_filter:
  - "--directory=/var/log/journal/"
  - "--output=cat"
  - "CONTAINER_NAME=app_91f8a72d_caddy-custom"
labels:
  type: caddy
```

---

### Step 4: Add Hub Collections & Parsers

In the Home Assistant UI, go to **CrowdSec** -> **Configuration** tab:

1. Add the `crowdsecurity/caddy` and `crowdsecurity/linux` collections to `collections`:

```yaml
collections:
  - crowdsecurity/home-assistant
  - crowdsecurity/linux
  - crowdsecurity/caddy
parsers:
  - crowdsecurity/geoip-enrich
```

2. (Optional) Turn on **Enable metrics** if you wish to query stats via `cscli metrics`.
3. Click **Save** and restart the CrowdSec add-on. On startup, CrowdSec will automatically download and enable the new collections and GeoIP data files.

---

### Step 5: Generate a Bouncer API Key in CrowdSec

Inside the CrowdSec container shell, register a new bouncer with CrowdSec's Local API:

```bash
crowdsec# cscli bouncers add caddy-bouncer
```

CrowdSec will output:

```text
Api key for 'caddy-bouncer':

   a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6

Please keep this key since you will not be able to retrieve it!
```

Copy the generated API key.

---

### Step 6: Configure Caddyfile

Open your Caddyfile (typically located in `/share/caddy/Caddyfile` or configured in the Caddy add-on) and add the CrowdSec global block and route handler with a minimal test endpoint:

```caddy
{
    crowdsec {
        api_url http://91f8a72d-crowdsec:8080
        api_key a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
        ticker_interval 15s
        lapi_timeout 10s
    }
}

https:// {
    route {
        crowdsec
        respond "Allowed by CrowdSec!"
    }
}
```

Restart the **Caddy Custom** add-on.

---

### Step 7: Verify Bouncer Registration

Inside the CrowdSec container shell, verify that Caddy has connected and is pulling decisions:

```bash
crowdsec# cscli bouncers list
```

You should see your bouncer registered with a green checkmark:

```text
────────────────────────────────────────────────────────────────────────────────────────────────────
 Name           IP Address   Valid  Last API pull              Type              Version  Auth Type
────────────────────────────────────────────────────────────────────────────────────────────────────
 caddy-bouncer  172.30.32.1  ✔️      2026-11-07T12:34:59+01:00  caddy-cs-bouncer  v0.15.0  api-key
────────────────────────────────────────────────────────────────────────────────────────────────────
```

---

### Step 8: Test IP Blocking

From your personal computer or an external machine on your network, simulate a suspicious scan (using a known bad User-Agent like Nikto):

```bash
pc$ curl -I -A "Mozilla/5.00 (Nikto/2.1.5)" --insecure https://homeassistant/
```

Inside the CrowdSec container shell, check the alerts and active decisions:

```bash
crowdsec# cscli alerts list
crowdsec# cscli decisions list
```

You will see an active ban decision against your test IP:

```text
╭────┬──────────────┬───────────────────────────────────┬─────────┬────┬───────────┬───────────────────────────┬──────────╮
│ ID │    value     │               reason              │ country │ as │ decisions │         created_at        │   kind   │
├────┼──────────────┼───────────────────────────────────┼─────────┼────┼───────────┼───────────────────────────┼──────────┤
│ 1  │ Ip:10.3.0.50 │ crowdsecurity/http-bad-user-agent │         │    │ ban:4h    │ 2026-11-07T12:44:26+01:00 │ crowdsec │
╰────┴──────────────┴───────────────────────────────────┴─────────┴────┴───────────┴───────────────────────────┴──────────╯
```

If you try accessing your Caddy server in your browser from that test IP, Caddy will return:

```text
HTTP 403 Forbidden
```

To unban your IP after testing:

```bash
crowdsec# cscli decisions delete --all
```

---

## Community Blocklists & CrowdSec Console

Protecting against local attacks is only part of CrowdSec. You can also block threats _before_ they ever reach your network using global intelligence:

### 1. Community Blocklist (Central API)

When **Enable Central API** (`enable_online_api: true`) is toggled on in the add-on configuration:

- CrowdSec registers with the Central API (CAPI).
- It automatically pulls down the community consensus blocklist, representing thousands of active, verified malicious IPs reported by the global CrowdSec community.
- Your connected bouncers (e.g. Caddy) will deny traffic from those IPs automatically before they can reach your server.

### 2. CrowdSec Console (Cloud Dashboard & Custom Blocklists)

You can enroll your Home Assistant CrowdSec instance into the free [CrowdSec Console](https://app.crowdsec.net):

1. Register an account at [app.crowdsec.net](https://app.crowdsec.net).
2. Go to **Security Engines** -> **Enroll Engine** and copy your enrollment key.
3. In the CrowdSec container shell, run:

```bash
crowdsec# cscli console enroll <YOUR_ENROLLMENT_KEY>
```

4. Restart the CrowdSec add-on, then approve the enrollment request in the web Console.
5. In the Console under **Blocklists**, subscribe to specialized blocklists (e.g., Tor exit nodes, open proxies, VPNs, malware droppers, VoIP scanners). These lists automatically synchronize to your Local API and are enforced by Caddy.

---

## Setup AppSec (Web Application Firewall - WAF)

IP blocking blocks repeat offenders and known bad IPs. CrowdSec's **AppSec Component** acts as an in-line Web Application Firewall (WAF) implementing the **OWASP Top 10** rules, protecting your web services from SQL injections, Cross-Site Scripting (XSS), Path Traversal, and virtual patching for 0-days on the very first request.

---

### Step 1: Configure AppSec Acquisition in CrowdSec

From your Home Assistant terminal, create the AppSec acquisition file:

```bash
homeassistant$ mkdir -p /config/.storage/etc/crowdsec/acquis.d
homeassistant$ nano /config/.storage/etc/crowdsec/acquis.d/appsec.yaml
```

Add the following configuration:

```yaml
---
appsec_config: crowdsecurity/appsec-default
labels:
  type: appsec
listen_addr: 0.0.0.0:7422
source: appsec
name: myAppSecComponent
```

---

### Step 2: Install AppSec Collections

In the CrowdSec add-on **Configuration** tab, add the generic rules and virtual patching collections:

```yaml
collections:
  - crowdsecurity/home-assistant
  - crowdsecurity/linux
  - crowdsecurity/caddy
  - crowdsecurity/appsec-generic-rules
  - crowdsecurity/appsec-virtual-patching
```

Click **Save** and restart the CrowdSec add-on.

---

### Step 3: Add the AppSec Module to Caddy Custom

In your **Caddy Custom** add-on configuration, add `github.com/hslatman/caddy-crowdsec-bouncer/appsec` to `plugins`:

```yaml
plugins:
  - github.com/hslatman/caddy-crowdsec-bouncer/http
  - github.com/hslatman/caddy-crowdsec-bouncer/appsec
```

Save and restart Caddy Custom to trigger compilation of both modules.

---

### Step 4: Update Caddyfile with AppSec Directives

In your `Caddyfile`, add `appsec_url` to the global `crowdsec` block and add `appsec` to your site route:

```caddy
{
    crowdsec {
        api_url http://91f8a72d-crowdsec:8080
        api_key a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
        ticker_interval 15s
        lapi_timeout 10s
        appsec_url http://91f8a72d-crowdsec:7422
    }
}

https:// {
    route {
        crowdsec
        appsec
        respond "Allowed by Bouncer and AppSec!"
    }
}
```

Restart the Caddy Custom add-on.

---

### Step 5: Test AppSec WAF Protection

From your computer, test that AppSec blocks an attempt to access sensitive environment files:

```bash
pc$ curl -I --insecure https://homeassistant/.env
```

AppSec intercepts the request and Caddy responds with:

```text
HTTP/2 403 Forbidden
```

Inside the CrowdSec container shell, check your alerts:

```bash
crowdsec# cscli alerts list
```

You will see an alert triggered under `crowdsecurity/appsec-generic-rules` confirming that the WAF detected and blocked the exploit attempt!

---

## Full Example: Reverse Proxying Home Assistant with Caddy, CrowdSec & AppSec

Once you have tested the bouncer and AppSec with the minimal test snippets above, you can configure Caddy to reverse proxy your actual Home Assistant instance.

### 1. Home Assistant Configuration

When running Home Assistant behind a reverse proxy, you must instruct Home Assistant's HTTP integration to trust forwarded headers from the proxy.

In your Home Assistant `configuration.yaml`, add:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.0.0/24 # Change this to your home network where your router/firewall will let incoming traffic through
    - 172.30.32.0/23 # Default Home Assistant internal Docker subnet
    - 127.0.0.1
    - ::1
```

Restart Home Assistant for these settings to take effect.

### 2. Complete Caddyfile

In your `Caddyfile`, configure `homeassistant` to forward traffic to `homeassistant:8123` with both `crowdsec` (IP bouncer) and `appsec` (WAF) active:

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
        reverse_proxy homeassistant:8123
    }
}
```

Restart the Caddy Custom add-on. Your Home Assistant instance is now fully protected by:

- **Local IPS**: Detecting failed logins and suspicious scanning patterns.
- **Community Blocklist**: Blocking verified malicious IPs globally.
- **AppSec WAF**: Blocking OWASP Top 10 web vulnerabilities and exploit attempts in real time.

---

## Helpful `cscli` Cheat Sheet

Enter the CrowdSec container shell from your Home Assistant terminal:

```bash
homeassistant$ docker exec -it app_91f8a72d_crowdsec bash
crowdsec#
```

Once inside (`crowdsec#`), you can run any `cscli` command directly:

### Decisions & Alerts (Bans & Threat Detection)

| Task                             | Command                                                              |
| :------------------------------- | :------------------------------------------------------------------- |
| **View active bans (decisions)** | `cscli decisions list`                                               |
| **Manually ban an IP**           | `cscli decisions add --ip <IP> --duration 24h --reason "manual ban"` |
| **Unban an IP**                  | `cscli decisions delete --ip <IP>`                                   |
| **Delete all active bans**       | `cscli decisions delete --all`                                       |
| **View triggered alerts**        | `cscli alerts list`                                                  |
| **Inspect a specific alert**     | `cscli alerts inspect <ALERT_ID>`                                    |

### Bouncers & Machines (Connected Components & Agents)

| Task                                | Command                        |
| :---------------------------------- | :----------------------------- |
| **List registered bouncers**        | `cscli bouncers list`          |
| **Register a new bouncer**          | `cscli bouncers add <NAME>`    |
| **Remove a bouncer**                | `cscli bouncers delete <NAME>` |
| **List registered machines/agents** | `cscli machines list`          |

### Hub Management (Collections, Parsers & Scenarios)

| Task                            | Command                  |
| :------------------------------ | :----------------------- |
| **List installed Hub items**    | `cscli hub list`         |
| **Update Hub index**            | `cscli hub update`       |
| **Upgrade installed Hub items** | `cscli hub upgrade`      |
| **List installed collections**  | `cscli collections list` |
| **List installed scenarios**    | `cscli scenarios list`   |

### Diagnostics & Metrics

| Task                             | Command             |
| :------------------------------- | :------------------ |
| **View engine metrics**          | `cscli metrics`     |
| **Check configuration validity** | `cscli config show` |
| **Show Local API status**        | `cscli lapi status` |

> [!TIP]
> You can also run single commands directly from the Home Assistant terminal without opening the container shell by prefixing:
>
> ```bash
> homeassistant$ docker exec -it app_91f8a72d_crowdsec cscli <COMMAND>
> ```
