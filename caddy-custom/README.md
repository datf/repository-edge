# Home Assistant App: Caddy Custom

Caddy is a modern, cross-platform open-source web server written in Go with HTTPS certificate renewal built-in.

## About

Add the Caddy plugins you want straight from the configuration! No need to compile caddy yourself or use their downloads page.

### Plugins Configuration

Standard plugins, specific versions, module replacements, and subpackages are supported:

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

## WARNING! THIS IS AN EDGE VERSION!

This Home Assistant Apps repository contains edge builds of apps.
Edge builds apps are based upon the latest development version.

- They may not work at all.
- They might stop working at any time.
- They could have a negative impact on your system.

This repository was created for:

- Anybody willing to test.
- Anybody interested in trying out upcoming apps or app features.
- Developers.

If you are more interested in stable releases of these add-ons:

<https://github.com/datf/hassio-addons>

