# Home Assistant Community App: Dante

Welcome to the Dante app for Home Assistant!

Dante is a robust, flexible, and highly reliable SOCKS proxy server (supporting both SOCKS4 and SOCKS5). This app allows you to run a SOCKS proxy directly on your Home Assistant. You can easily route traffic from other devices or networks securely through your Home Assistant host.

> [!WARNING]
> By default, the Dante configuration allows all traffic through the local `1080` port without authentication.
> **DO NOT** expose your Home Assistant instance publicly, or at least do not expose port `1080` to the internet unless you are absolutely sure of what you are doing. Leaving an open proxy exposed can lead to abuse and compromise your network.

## Installation

This app is currently a custom add-on. You will need to add a custom repository to install it:

1. Follow [this link](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fdatf%2Fhassio-addons) to add the repository and click Add.
2. Scroll down to find the new repository and click on the **Dante** app.
3. Click **Install**.
4. Start the "Dante" app.
5. Check the logs of the app to ensure it started properly and there are no errors.

That's it! The app should work right out of the box, provided port `1080` is not already in use by another service on your host.

## Configuration

**Note**: _Remember to restart the app whenever you change the configuration for the changes to take effect._

Here is an example app configuration:

```yaml
log_level: info
require_username_auth: false
credentials:
  username: "admin"
  password: "your_secure_password"
```

### Option: `log_level`

The `log_level` option controls the level of log output by the app. You can change this to be more or less verbose, which is especially helpful when you are troubleshooting an issue.

Possible values are:

- `trace`: Show every detail, like all called internal functions.
- `debug`: Shows detailed debug information.
- `info`: Normal (usually) interesting events.
- `notice`: Normal but significant events.
- `warning`: Exceptional occurrences that are not errors.
- `error`: Runtime errors that do not require immediate action.
- `fatal`: Something went terribly wrong. App becomes unusable.

By default, this is set to `info`, which is the recommended setting for daily use.

### Option: `require_username_auth`

Set this to `true` if you want to require SOCKS clients to authenticate with a username and password before they can use the proxy.

When enabled, the app will automatically create a user on the system level (based on your `credentials` settings below) so that Dante can authenticate against it.

> [!WARNING]
> Standard SOCKS5 username/password authentication is **sent in cleartext** (unencrypted). Anyone intercepting your traffic on the network can see the username and password. Do not use this over untrusted networks (like public Wi-Fi) unless the connection is wrapped in a secure tunnel, such as a VPN or an SSH tunnel.

### Option: `credentials`

This section defines the user credentials that will be created if `require_username_auth` is set to `true`.

- **`username`**: The username for your SOCKS proxy.
- **`password`**: The password for your SOCKS proxy. (Make sure to pick a strong one!)

## Advanced Configuration (`sockd.conf`)

By default, the app provides a very basic configuration to get you started quickly. However, if you want to customize how Dante behaves—for example, enforcing the username authentication or setting specific routing rules—you need to provide your own `sockd.conf` file.

You can do this by creating a `sockd.conf` file in the **`addon_config` directory for Dante**. (Note: the exact folder name inside `addon_config` might have a unique repository ID prefix, so just look for the folder belonging to Dante).

> [!TIP]
> **Network Interfaces (`external`):** In the examples below, the external interface is set to `eth0`. Because Home Assistant can run on many different types of hardware, **your interface name might be different**:
>
> - `eth0` is common for standard x86/64 systems and virtual machines.
> - `end0` is common for ARM devices (like the Raspberry Pi).
> - `wlan0` is common if you are using Wi-Fi as your main network adapter.
>
> If the app fails to start, check the logs—you might just need to update `eth0` in your configuration to match your system's actual network interface!

### Basic Example (`sockd.conf`)

Here is a basic example of a `sockd.conf` file that allows all traffic (similar to the default out-of-the-box behavior):

```text
logoutput: stderr
internal: 0.0.0.0 port = 1080
external: eth0

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

If you set `require_username_auth: true` in the app configuration, you **must** also configure Dante to require the `username` method in your `sockd.conf`.

Modify the `socksmethod` line like this:

```text
logoutput: stderr
internal: 0.0.0.0 port = 1080
external: eth0

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
