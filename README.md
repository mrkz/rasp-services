# rasp-services

Homelab service stack for Raspberry Pi and x86 hosts.

Manages containerized services via Podman Quadlets deployed by Ansible.
No Docker Compose. No Portainer. Services are managed with `systemctl` and
observed with `journalctl`.

```
Hosts:  Debian 13 / Ubuntu 24.04 / Fedora 40+
Arch:   arm64 (Raspberry Pi 4/5) / x86_64
Engine: Podman (rootless-compatible Quadlets, user or system)
CM:     Ansible (ansible.builtin.* only -- no galaxy dependencies)
```

---

## Repository layout

```
ansible/
  inventory/
    hosts.yml                  -- static inventory
    group_vars/all.yml         -- image versions, network definitions
  host_vars/
    rpi4.yml.example           -- per-host secrets template (committed)
    rpi4.yml                   -- real secrets (gitignored)
  roles/
    common/                    -- directories, homelab.target
    podman_network/            -- Podman network units
    pihole/                    -- Pi-hole DNS + ad-blocker
    mosquitto/                 -- Eclipse Mosquitto MQTT broker
    zigbee2mqtt/               -- Zigbee2MQTT bridge
    homeassistant/             -- Home Assistant
    nextcloud_db/              -- MariaDB for Nextcloud
    nextcloud_redis/           -- Redis for Nextcloud
    nextcloud/                 -- Nextcloud
    nextcloud_cron/            -- Nextcloud background cron
    esphome/                   -- ESPHome dashboard
  site.yml                     -- master playbook
host/
  99-zigbee.rules              -- udev rule for Zigbee USB dongle
  homelab.target               -- static reference copy of systemd target
```

---

## Prerequisites

### On the control machine

- Ansible >= 2.14
- Python >= 3.10

No `ansible-galaxy` collections are required.

### On each target host

- Podman >= 4.6 (Quadlet support built-in)
- `systemd --user` running for the deploy user, or system-level Podman
- `dialout` group exists (for Zigbee dongle access)

Install Podman:

```
# Debian/Ubuntu
sudo apt install podman

# Fedora
sudo dnf install podman
```

---

## Service-specific prerequisites

### Mosquitto

Mosquitto is configured entirely through a `mosquitto.conf` volume mount.
Ansible does NOT manage this file. Before the first run, place a valid
`mosquitto.conf` at:

```
/opt/homelab/data/mosquitto/config/mosquitto.conf
```

Minimal example (no auth, local-only):

```
listener 1883
allow_anonymous true

persistence true
persistence_location /mosquitto/data/

log_dest file /mosquitto/log/mosquitto.log
```

For WebSocket support add:

```
listener 9001
protocol websockets
```

### Zigbee2MQTT

Configuration is managed through `/opt/homelab/data/zigbee2mqtt/configuration.yaml`.
Ansible does NOT create this file. Before the first run, create it with at
minimum:

```yaml
homeassistant: true
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
serial:
  port: /dev/ttyUSB0
frontend:
  port: 8080
```

The Zigbee USB dongle symlink `/dev/zigbee` is created by:

```
sudo cp host/99-zigbee.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Verify the symlink exists before deploying: `ls -la /dev/zigbee`

---

## Deployment

### 1. Create host vars

```
cp ansible/host_vars/rpi4.yml.example ansible/host_vars/<hostname>.yml
chmod 600 ansible/host_vars/<hostname>.yml
# Edit the file and fill in all values
```

### 2. Full deploy

```
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml
```

### 3. Deploy a single service

The `common` and `podman_network` roles are tagged `always` and run on
every invocation regardless of `--tags`.

```
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags pihole
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags mosquitto
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags zigbee2mqtt
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags homeassistant
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags nextcloud
ansible-playbook -i ansible/inventory/hosts.yml ansible/site.yml --tags esphome
```

---

## Service map

```
Service           Network(s)                    Default port
-----------       ---------------------------   ----------------
pihole            frontend                      8080 (HTTP)
                                                53   (DNS, host)
mosquitto         iot                           1883 (MQTT, loopback)
                                                9001 (WS, loopback)
zigbee2mqtt       iot, frontend                 8085
homeassistant     host                          8123
nextcloud_db      nextcloud_internal            (internal only)
nextcloud_redis   nextcloud_internal            (internal only)
nextcloud         frontend, nextcloud_internal  8081
nextcloud_cron    frontend, nextcloud_internal  (no port)
esphome           host                          6052
```

---

## Podman networks

| Name                | Subnet          |
|---------------------|-----------------|
| frontend            | 172.20.0.0/24   |
| iot                 | 172.21.0.0/24   |
| nextcloud_internal  | 172.22.0.0/24   |

---

## Image update policy

All images are pinned to explicit tags in `group_vars/all.yml`.
Exception: `home-assistant:stable` -- HA does not publish semver tags.

To update an image, bump the tag in `group_vars/all.yml` and re-run the
playbook. The role pulls the new image, compares digests, and restarts the
service only if the digest changed.

---

## Day-2 operations

Check service status:

```
systemctl --user status homelab.target
systemctl --user status pihole.service
```

Follow logs:

```
journalctl --user -fu pihole.service
journalctl --user -fu homeassistant.service
```

Restart a service:

```
systemctl --user restart mosquitto.service
```

---

## Notes

- `nextcloud_cron` logs errors until the Nextcloud first-run wizard
  completes. `Restart=on-failure` handles retries automatically.
- Home Assistant and ESPHome use `Network=host` for mDNS/SSDP/Matter
  discovery. `PublishPort` must not be used with host networking.
- Home Assistant runs `--privileged` for full hardware access (Bluetooth,
  USB, GPIO). It is the only service in this stack with that flag.
- The `:Z` SELinux volume label is safe on Debian/Ubuntu (AppArmor);
  Podman silently ignores it when SELinux is not active.
