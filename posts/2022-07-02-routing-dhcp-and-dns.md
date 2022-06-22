# 2022-07-02: Routing, DHCP and DNS

Following on from my [last post](/posts/2022-06-05-alpha.cephei.uk.md),
in this post we'll cover the setup of routing,
DHCP and DNS services on alpha.cephei.uk.
But first,
we'll outline the network configuration changes I made
in preparation for the setup of these services.

Rather than including the full text of all configuration files in this post,
we'll be referencing the [config repository](https://github.com/cephei-uk/config)
where appropriate.

## Network configuration

Before we can turn our newly set up machine into a router,
we need to configure the network connections on the system.
The default network configuration service in MicroOS is NetworkManager,
however this is not my preferred choice.
Instead we'll be switching to systemd-networkd.
So first, we need to install it:

```
transactional-update pkg in systemd-networkd
reboot
```

To make network configuration and management easier,
we're going to assign meaningful network interface names using udev rules.
These rules identify the network interfaces by their MAC addresses
(which are, at least in this context, fixed and unique)
and replace the kernel generated names like `enp2s0` and `enp3s0` with
the more meaningful names `wan` (for the externally-facing interface)
and `lan-port0` (for the internally-facing interface).

Configuration file:
[`/etc/udev/rules.d/70-net-names.rules`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/udev-rules/70-net-names.rules)

Now we can write our systemd-networkd configuration in terms of the new network
interface names.

Initially, our router will be deployed behind another router
which was provided by my current ISP.
This is not the ideal deployment as it results in two stages
of routing and network address translation (NAT)
before packets have even left my house.
However, it's a good place to start and
it means that our `wan` configuration is currently very simple.
We tell systemd-networkd to acquire an IPv4 address using DHCP
and to enable packet forwarding
(effectively packet routing) for this interface.
We also prevent use of the DNS server advertised in the DHCP response
since we'll be setting up our own DNS server shortly.

Configuration file:
[`/etc/systemd/network/wan.network`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/network/wan.network)

In the future we'll be replacing the ISP-provided router
(and possibly also the ISP!)
and revisiting the `wan` interface.

On the LAN side things are a little more complicated.
In order to run a DHCP server in a container,
we need to have a layer 2 network bridge which can pass DHCP packets.
A layer 3 bridge is not suitable here as
a device on the network sending a DHCP request
will use a source address of `0.0.0.0`
and a destination address of `255.255.255.255`
since it does not have a valid IPv4 address (i.e. layer 3 address) or netmask
until it gets a DHCP response.
In Linux, we can use a macvlan bridge
to provide the functionality we need here.

To define a macvlan bridge named `lan`,
we create a `lan.netdev` file specifying that this is a macvlan bridge device
and a `lan.network` file to assign an IP address, netmask, DNS server & domain name
and to enable packet forwarding for this interface.
Our LAN network will use the `10.0.0.0/24` subnet
and we will be using the static IP addresses `10.0.0.1` for this host
and `10.0.0.2` for our DHCP & DNS container
(the reason why this container has its own IP address will be covered later).
We also create a `lan-port0.network` file
to assign the underlying network interface to our macvlan bridge.

Configuration files:
* [`/etc/systemd/network/lan.netdev`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/network/lan.netdev)
* [`/etc/systemd/network/lan.network`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/network/lan.network)
* [`/etc/systemd/network/lan-port0.network`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/network/lan-port0.network)

Once all these configuration files are in place we can disable NetworkManager,
enable systemd-networkd and reboot our machine.
At this point we'll also enable systemd-resolved to prove DNS resolution
(via our local DNS server due to the DNS setting in the `lan.network` file)
and modify the `/etc/resolv.conf` symlink to point at
the dynamic `resolv.conf` file managed by systemd-networkd.
If all goes well, after rebooting the system will come back up
with the new network interface names and configurations.

```
systemctl disable NetworkManager
systemctl enable systemd-networkd
systemctl enable systemd-resolved
ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
reboot
```

## Routing and firewall

There are multiple options available on Linux
to configure routing and firewall capabilities
including `iptables`, `nftables` and `firewalld`.
My preference is to use `nftables` due to the straightforward syntax and efficiency.
So firstly we need to install nftables and reboot:

```
transactional-update pkg in nftables
reboot
```

In OpenSuse there is no nftables service provided by default
so we need to import this
[from Debian](https://salsa.debian.org/pkg-netfilter-team/pkg-nftables/-/blob/master/debian/nftables.service).

The nftables configuration will allow all outgoing traffic from this host
but will restrict incoming and forwarding (i.e. routed) traffic.

Incoming traffic will be allowed if it meets any of the following criteria:
* Is in response to earlier outgoing traffic.
* Originated from the localhost interface.
* Is ICMP traffic required for correct network operation and does not exceed a rate of 10 packets/second.
* Is directed to the SSH port (this may be restricted further in the future).

Forwarding traffic will be allowed if it meets any of the following criteria:
* Originates on the LAN side.
* Originates on the WAN side and is in response to earlier outgoing traffic.

Forwarding traffic which leaves via the WAN interface will be modified
to use the source IP address of the WAN interface itself (this is called masquerading).

Configuration files:
* [`/etc/systemd/system/nftables.service`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/common/systemd/nftables.service)
* [`/etc/nftables.conf`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/nftables/nftables.conf)

Once these configuration files are in place,
we can reload systemd (to parse the new service file)
and start the nftables service.

```
systemctl daemon-reload
systemctl enable --now nftables.service
```

The nftables ruleset can now be checked using the command `nft list ruleset`.

## Podman & podman-compose

When we installed OpenSuse MicroOS, podman was setup by default
but a little additional configuration is neeeded.
First, we need to add an additional CNI configuration for our macvlan bridge
so that we can attach our pihole container to this network.

Configuration file:
[`/etc/cni/net.d/lan.conflist`](https://github.com/cephei-uk/config/blob/cfd4692d0172aa2f85c7763b4510f17b12ce17e5/hosts/alpha/cni/lan.conflist)

We will be using [podman-compose](https://github.com/containers/podman-compose)
to manage containerised services on our network. This gives us two main
benefits: firstly, we can write tidy yaml configuration files instead of long
lists of podman command line arguments, and secondly, we can manage a service
as a single unit even it if is composed of multiple container instances.

The latest podman-compose release at the time of writing is v1.0.3 which
does not support the network configuration that we'll be using to attach
a container to our macvlan bridge. Therefore, until a new release is made
which includes this support, we need to install the latest development
snapshot of podman-compose. We will install this package and its
dependencies into `/usr/local` since it is being managed with pip instead of our
distribution's package manager.

```
pip3 install --prefix=/usr/local https://github.com/containers/podman-compose/archive/refs/heads/devel.zip
```

We will use systemd to provide a consistent interface for managing our
containerised services and to allow these services to be started at boot.
Each containerised service will be defined by a `podman-compose.yaml` file
placed in a directory under `/etc/containers/compose`.
To support this we will need a service template which will use podman-compose
to start or stop an individual service.

Configuration file:
[`/etc/systemd/system/podman-compose@.service`](https://github.com/cephei-uk/config/blob/85ad2418b6f90ae1e20edf807b8fc0b805c15d17/common/systemd/podman-compose@.service)

The first containerised service which we will set up is pi-hole, which will also
serve as an example of how to use this systemd service template.

## Pi-hole

[Pi-hole](https://pi-hole.net/) will be used to provide DHCP & DNS services for
our network with network-wide ad blocking.

This service will run inside a container connected to the macvlan bridge defined
earlier so that DHCP requests and responses can get to and from the service.
When using the macvlan bridge, the will need to have its own IP address on our
network in order to be reachable from the hose and from other machines on the
network. We will use the address 10.0.0.2 for this purpose. The container will
require the NET_ADMIN capability so that it can respond to DHCP requests and it
will require ports exposing for DNS, DHCP and the web interface. The DNS server
for the container itself will be set to 127.0.0.1 so that local queries are
resolved via the pi-hole service.

The administrator password for the pi-hole web interface is set via the
environment variable `WEBPASSWORD`. This variable will not be stored directly
in the podman-compose yaml file as we do not wish to commit the password into
our public config repository. Instead a secrets file will be used, stored
under `/etc/secrets` (a directory with 0700 permissions), to store this
variable. We also need to set two other environment variables for pi-hole:
`FTLCONF_REPLY_ADDR4` & `VIRTUAL_HOST`, and since these variables are not
secret they can be set in the podman-compose config file.

We need to mount two data directories into the pi-hole container, one for
the pihole state and one for the dnsmasq state. Each of these directories
will be placed under `/var/opt/pihole` on the host and will be included in
our backup (when we set up backups at a later date).

Configuration file:
[`/etc/containers/compose/pihole/podman-compose.yaml`](https://github.com/cephei-uk/config/blob/4c9e649265729272786cfdc470bb89cb298f055d/services/pihole/podman-compose.yaml)

Once the podman-compose configuration file is in place, we can reload systemd
and start our pi-hole instance:

```
systemctl daemon-reload
systemctl enable --now podman-compose@pihole.service
```

After the service started I did some testing to confirm that the DHCP & DNS
services were working as expected, and that the pi-hole web interface could be
accessed by visiting <http://pi.hole> from inside the network. With this testing
complete, I was happy that the initial setup of this firewall and gateway router
was a success.
