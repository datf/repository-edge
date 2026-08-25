# Home Assistant Community Add-on: Dante to WireGuard

Welcome to the Dante to WireGuard add-on for Home Assistant!

This add-on combines Dante, a robust and flexible SOCKS proxy server (supporting SOCKS4 and SOCKS5), with a WireGuard VPN client. This allows you to run a SOCKS proxy directly on your Home Assistant instance that automatically routes all its outgoing traffic through a WireGuard VPN connection. You can easily route traffic from other devices or networks securely through the VPN without needing to install VPN clients on every device.

> [!WARNING]
> By default, the Dante configuration allows all traffic through the local `1080` port without authentication.
> **DO NOT** expose your Home Assistant instance publicly, or at least do not expose port `1080` to the internet unless you are absolutely sure of what you are doing. Leaving an open proxy exposed can lead to abuse and compromise your network.

## Installation

This add-on is currently a custom add-on. You will need to add a custom repository to install it:

1. Navigate in your Home Assistant UI to **Settings** -> **Add-ons** -> **Add-on Store**.
2. Click on the three dots (⋮) in the top right corner and select **Repositories**.
3. Add the following repository URL: `https://github.com/datf/hassio-addons`
4. Click **Add** and wait for the repository to load.
5. Close the repositories dialog and reload the page if needed.
6. Scroll down to find the new repository and click on the **Dante to WireGuard** add-on.
7. Click **Install**.
8. Place your WireGuard configuration file(s) (e.g., `wg0.conf`) in the add-on's `/config` directory.
9. Start the add-on.
10. Check the logs of the add-on to ensure it started properly and there are no errors.

That's it! The add-on should work right out of the box, provided port `1080` is not already in use by another service on your host and you have provided a valid WireGuard configuration.

## Configuration

**Note**: _Remember to restart the add-on whenever you change the configuration for the changes to take effect._

Here is an example add-on configuration:

```yaml
log_level: info
require_username_auth: false
credentials:
  username: "admin"
  password: "your_secure_password"
mtu: 1280
```

### Option: `log_level`

The `log_level` option controls the level of log output by the add-on. You can change this to be more or less verbose, which is especially helpful when you are troubleshooting an issue.

Possible values are:

- `trace`: Show every detail, like all called internal functions.
- `debug`: Shows detailed debug information.
- `info`: Normal (usually) interesting events.
- `notice`: Normal but significant events.
- `warning`: Exceptional occurrences that are not errors.
- `error`: Runtime errors that do not require immediate action.
- `fatal`: Something went terribly wrong. Add-on becomes unusable.

By default, this is set to `info`, which is the recommended setting for daily use.

### Option: `require_username_auth`

Set this to `true` if you want to require SOCKS clients to authenticate with a username and password before they can use the proxy.

When enabled, the add-on will automatically create a user on the system level (based on your `credentials` settings below) so that Dante can authenticate against it.

> [!WARNING]
> Standard SOCKS5 username/password authentication is **sent in cleartext** (unencrypted). Anyone intercepting your traffic on your local network can see the username and password.

### Option: `credentials`

This section defines the user credentials that will be created if `require_username_auth` is set to `true`.

- **`username`**: The username for your SOCKS proxy.
- **`password`**: The password for your SOCKS proxy. (Make sure to pick a strong one!)

### Option: `mtu`

The MTU (Maximum Transmission Unit) value for the WireGuard interface. WireGuard adds encryption overhead to your traffic. If this is set too high (e.g., 1500), packets will exceed physical network limits and get silently dropped, causing connection stalls.

Lowering the MTU prevents this issue. 1280 is the safest default, but you can try 1400 for better performance. This explicitly overrides any MTU specified in your WireGuard configuration files to ensure stable connections.

## WireGuard Configuration

To connect to a VPN, you must provide a valid WireGuard configuration file. Place your configuration file (e.g., `wg0.conf`) in the add-on's `/config` directory.

### Multiple Configurations (Rotation)

If you have multiple WireGuard configurations (for example, to connect to different servers for load balancing or to evade bans), you can simply place them all in your `/config` directory (e.g. `wg0.conf`, `wg1.conf`, `wg2.conf`).

On every start/restart of the add-on, it will automatically select the next configuration file in alphabetical sequence. This makes it easy to rotate servers by just restarting the add-on.

## Advanced Configuration (`sockd.conf`)

By default, the add-on provides a configuration that allows all local traffic and routes it through the `wg0` interface. However, if you want to customize how Dante behaves—for example, enforcing username authentication or setting specific routing rules—you need to provide your own `sockd.conf` file.

You can do this by creating a `sockd.conf` file in the **`addon_config` directory for Dante to WireGuard**. (Note: the exact folder name inside `addon_config` might have a unique repository ID prefix).

> [!IMPORTANT]
> **Network Interfaces (`external`):** In the examples below, the external interface is set to `wg0`. Because this add-on forces all proxy traffic through WireGuard, **your external interface MUST be `wg0`**. If you change this to `eth0` or `end0`, your proxy traffic will bypass the VPN and leak your real IP address!

### Basic Example (`sockd.conf`)

Here is a basic example of a `sockd.conf` file that allows all traffic through the VPN (similar to the default out-of-the-box behavior):

```text
logoutput: stderr
internal: 0.0.0.0 port = 1080
external: wg0

clientmethod: none
socksmethod: none

user.privileged: root
user.unprivileged: nobody

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error
}
```

### Example with Authentication

If you set `require_username_auth: true` in the add-on configuration, you **must** also configure Dante to require the `username` method in your `sockd.conf`.

Modify the `socksmethod` line like this:

```text
logoutput: stderr
internal: 0.0.0.0 port = 1080
external: wg0

# Require clients to provide a valid username and password
clientmethod: none
socksmethod: username

user.privileged: root
user.unprivileged: nobody

client pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error
}

socks pass {
    from: 0.0.0.0/0 to: 0.0.0.0/0
    log: error
}
```

For more advanced configurations and rule sets, you can always refer to the [official Dante documentation](https://www.inet.no/dante/doc/latest/sockd.conf.5.html). Happy proxying!
