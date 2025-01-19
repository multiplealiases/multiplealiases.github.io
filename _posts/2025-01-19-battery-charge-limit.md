---
layout: post
title: Battery charge limit systemd service (adjustable)
date: 2025-01-19
permalink: /battery-charge-limit.html
---

Just a slight expansion on
[a Stack Overflow answer](https://unix.stackexchange.com/a/716800).

Save as `/etc/systemd/system/battery-charge-limit@.service`.

```ini
[Unit]
Description=Set battery charge limit to %I percent
After=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/sh -c 'echo %I > /sys/class/power_supply/BAT0/charge_control_end_threshold'
ExecStop=/bin/sh -c 'echo 100 > /sys/class/power_supply/BAT0/charge_control_end_threshold'

[Install]
WantedBy=multi-user.target
```

(replace `/sys/class/power_supply/BAT0/charge_control_end_threshold` with
the correct sysfs path for your machine, if applicable)

# Usage

Limit the battery to 80%.

```console
# systemctl start  battery-charge-limit@80.service
# systemctl enable battery-charge-limit@80.service
```

Keep in mind that this service has no way of knowing
if 2 instances of itself exist,
that's to say, the following situation isn't prevented by systemd:

```
# systemctl enable battery-charge-limit@80.service
# systemctl enable battery-charge-limit@50.service
```
