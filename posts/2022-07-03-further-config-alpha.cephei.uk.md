# 2022-07-03: Further config for alpha.cephei.uk

This post documents a couple of configuration changes made to my gateway router
after the initial setup was complete.

## Hostname correction

After running pi-hole for a short time I noticed that the hostname
`alpha.cephei.uk` was not resolving correctly when performing a lookup from
another host on the network. The DNS response for this hostname was `127.0.0.1`
instead of `10.0.0.1`. This was a little confusing and led me into an
investigation of how pi-hole resolves host names and how the `/etc/hosts` file
is handled in podman containers.

I learned that the default behaviour of podman is to mount the `/etc/hosts` file
from the host into each container so that any customisations for a particular
site only need to be made in one place. Podman does support a `--no-hosts`
command line argument but when used with the pi-hole container image this
resulted in a completely absent `/etc/hosts` file in the container, preventing
the lookup of names like `localhost` which are expected to be valid in a Linux
environment. So we do want to preserve the default behaviour here.

The pi-hole DNS service will use entries in the local `/etc/hosts` file when
responding to requests, and since this file is mounted in from the host, it
will see the following entry which we previously added:

```
127.0.0.1       alpha.cephei.uk alpha localhost
```

So now we can see why the wrong response was being given for the hostname
`alpha.cephei.uk`!

We can resolve this issue fairly simply as our host has a static IP address
on the LAN network interface. At the same time we can add an entry for
`pihole.cephei.uk` so that the pi-hole container has a clearly defined
FQDN. The new contents of the `/etc/hosts` file for this machine is
stored in the configuration repository.

Configuration file:
[`/etc/hosts`](https://github.com/cephei-uk/config/blob/0bb9f28f925471063239ae5ab2f07354838be224/hosts/alpha/hosts)

## Automatic updates

By default, MicroOS will automatically apply updates and reboot the system once
a day. Since this host is providing key network services we want to ensure
that automatic reboots happen at an appropriate time and schedule automatic
updates to occur just before this reboot window.

### Automatic reboot window

On MicroOS, automatic reboots can configured using `rebootmgrctl` or by editing
`/etc/rebootmgr.conf` and restarting the service. So that the config file can be
kept in our config repo we will take the second approach. The reboot window is
set to 05:30-06:00 on Mondays, a time when I will most definitely be asleep and
will not be bothered by a short interruption to my internet connectivity.

Configuration file:
[`/etc/rebootmgr.conf`](https://github.com/cephei-uk/config/blob/4c9e649265729272786cfdc470bb89cb298f055d/hosts/alpha/rebootmgr.conf)

After changing this configuration file, the rebootmgr service must be restarted:

```
systemctl restart rebootmgr.service
```

### Automatic updates

Since MicroOS uses transactional updates and requires a reboot after applying
updates to switch to the new rootfs snapshot, it makes sense to run the
automatic update just before the reboot window each week. To change the
transactional update scheduling, the configuration of the
`transactional-update.timer` systemd unit needs to be overridden by
creating a config file in `/etc/systemd/system/transactional-update.timer.d`
and reloading the systemd daemon. In the override config file, we
schedule automatic updates to occur at 04:45 on Mondays so that there is
plenty of time for this to complete before the start of the reboot window.

Configuration file:
[`/etc/systemd/system/transactional-update.timer.d/override.conf`](https://github.com/cephei-uk/config/blob/4c9e649265729272786cfdc470bb89cb298f055d/hosts/alpha/systemd/transactional-update.timer.d/override.conf)
