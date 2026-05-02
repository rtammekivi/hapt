# HAPT

Home Assistant Presence Tracker (HAPT) is an event-driven device presence tracker for [Home Assistant][homeassistant] on
an OpenWRT router or access point.

## Description

HAPT listens on association and disassociation events to wireless networks, using the hostapd control interface. It
keeps track of which device is connected to which networks. When a device connects to its first network, or disconnects
from its last network, a service call to Home Assistant is performed to mark the device as home or away. By tracking
active device connections, HAPT ensures that a device switching between different networks (e.g. the 2.4 GHz and 5 GHz
bands) is not marked as away.

## Install

### Via LuCi (web UI)

1. Drop the feed's public key into `/etc/apk/keys/`. LuCi has no UI for trusted keys, so do this once from a shell
   (SSH, or **System > Custom Commands**):

   ```sh
   wget -O /etc/apk/keys/hapt.pub https://rtammekivi.github.io/hapt/hapt.pub
   ```

2. In LuCi, open **System > Software** and click **Configure APK**. The dialog exposes the contents of
   `/etc/apk/repositories.d/distfeeds.list` (the official OpenWrt feeds) and `/etc/apk/repositories.d/customfeeds.list`
   (your own feeds). Add this line under `customfeeds.list`:

   ```none
   https://rtammekivi.github.io/hapt/packages.adb
   ```

   Save & Apply.

3. Back on the Software page, click **Update lists**, search for `hapt`, then **Install**.

`apk upgrade` (or LuCi's **Upgrade...** button) will pull future versions from the same feed.

### Via shell

```sh
wget -O /etc/apk/keys/hapt.pub https://rtammekivi.github.io/hapt/hapt.pub
echo 'https://rtammekivi.github.io/hapt/packages.adb' >> /etc/apk/repositories.d/customfeeds.list
apk update
apk add hapt
```

### One-shot install (no feed)

Download an `.apk` from `https://rtammekivi.github.io/hapt/` and install it via LuCi's **System > Software > Upload
Package**, or from a shell with `apk add --allow-untrusted <file>`.

## Usage

### Configuration

The package installs but does not start the service — it must be configured first. Required: the Home Assistant URL and
a long-lived access token from your [Home Assistant profile][ha-profile] ([more info][token]). Strongly recommended: a
`track_mac_address` whitelist, otherwise every device that connects to the wifi (guests, IoT, etc.) shows up in Home
Assistant's entity registry.

```sh
uci set hapt.global.host='http://homeassistant:8123'
uci set hapt.global.token='YOUR_LONG_LIVED_TOKEN'
uci add_list hapt.global.track_mac_address='AA:BB:CC:DD:EE:FF'
uci commit hapt
service hapt restart
```

Repeat the `uci add_list` line for each device you want to track. To find MAC addresses, list the currently associated
wireless clients (replace `phy1-ap0` with your interface — `iw dev` shows them all):

```sh
iwinfo phy1-ap0 assoclist
```

Devices that have a DHCP lease are also visible alongside their hostname:

```sh
cat /tmp/dhcp.leases
```

In LuCi the same information is under **Status > Overview** (associated wireless stations) and **Status > Overview >
Active DHCP Leases**.

`service hapt restart` works for both the initial start and for picking up later config changes. Tail hapt's log
messages with:

```sh
logread -f -e hapt
```

#### Example `/etc/config/hapt`

```
config hapt 'global'
    # Home Assistant URL (including scheme).
    option host                     'http://homeassistant:8123'

    # Long-lived access token from Home Assistant (user profile > Security).
    option token                    'eyJ0eXAiOiJKV1QiLCJhbGciOi...'

    # Seconds a device stays "home" after its first association event.
    option consider_home_connect    '86400'

    # Seconds a device stays "home" after its last disassociation event.
    # Home Assistant has no "away" call, so this short window lets HA expire the entity.
    option consider_home_disconnect '60'

    # Prefix prepended to the Home Assistant device_tracker entity name. Leave empty for none.
    option device_id_prefix         ''

    # Wireless interfaces to monitor. Omit the list to monitor every interface
    # under /var/run/hostapd (e.g. to skip a guest network, list only the others).
    list   wifi_interfaces          'phy1-ap0'

    # MAC address whitelist. Omit entirely to track every device that joins
    # the wifi (usually not what you want — clutters Home Assistant).
    list   track_mac_address        'aa:bb:cc:dd:ee:ff'
    list   track_mac_address        '11:22:33:44:55:66'
```

### In Home Assistant

hapt calls Home Assistant's `device_tracker.see` service on every connect and disconnect. Each tracked device appears
as a `device_tracker.<id>` entity with state `home` / `not_home`, where `<id>` is the device's DHCP hostname — or the
MAC with `:` replaced by `_` if no lease is found — optionally prefixed with `device_id_prefix`. New entities are
persisted in Home Assistant's `known_devices.yaml` and can be used in automations, presence detection, and dashboards
like any other device tracker.

### One-shot sync

Synchronize just the currently connected devices and print any errors to the terminal — useful for debugging the Home
Assistant connection:

```sh
hapt
```

## Development

You can build a custom package by running the `makepkg.sh` script, which will run the package build in a Docker
container and place the compiled package in the `build/bin` directory.

### Publishing the feed (one-time setup for forks)

The CI workflow signs the feed index with an RSA key supplied via the `APK_SIGN_KEY` repository secret, and deploys
the signed index, the `.apk` files, and the matching public key to GitHub Pages.

To set this up on a fork:

1. Generate a signing keypair locally (do not commit the private key):

   ```sh
   openssl genrsa -out key-hapt.rsa 2048
   ```

2. Add the contents of `key-hapt.rsa` as a repository secret named `APK_SIGN_KEY` (Settings > Secrets and variables >
   Actions).
3. Enable Pages in repository settings (Settings > Pages > Source: GitHub Actions).
4. Push to `master`. The workflow will publish the feed at `https://<owner>.github.io/hapt/`.

## Acknowledgments

This project has been inspired by the [openwrt_hass_devicetracker][hasstracker] package.

[homeassistant]: https://www.home-assistant.io/
[hasstracker]: https://github.com/mueslo/openwrt_hass_devicetracker
[releases]: https://github.com/oxan/hapt/releases
[token]: https://developers.home-assistant.io/docs/auth_api/#long-lived-access-token
[ha-profile]: https://my.home-assistant.io/redirect/profile/
