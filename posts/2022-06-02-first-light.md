# 2022-06-02: First Light

![Erakis/Herschel's Garnet Star/μ Cephei](/img/mu-cephei.jpg)

*Image of μ Cephei, aka "Erakis" or "Herschel's Garnet Star".
From [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Erakis_(Garnet_Sidus).jpg),
public domain.*

## Goal

**Refresh my home network, improving the available services and
connectivity.**

Over the past few months I've simplified my home network all the way down to the
absolute minimum. That is, a vendor supplied router and a network switch with no
special configuration. I've done this due to various distractions and due to a
desire to save electricity costs. However, it's boring. Profoundly boring.
Unfulfilling. Choosing not to nerd out, choosing not to build cool systems,
makes me sad.

So, I'm reversing that decision. Part of embracing my autistic, nerdy self is to
build cool shit when I want to build cool shit. I revel in putting together
connected systems, choosing every detail of my setup and going way deeper than
is necessary into the technical minutiae.

The trick is to define the scope well. That scope can change over time, it
doesn't need to be overly precise, but it's necessary to provide some context
and avoid staying stuck in the analysis stage. I want to actually build this
network, not just research and dwell in indecision.

## Scope

* Use equipment I already have where possible. Buy new equipment only where
  needed.

* Use low power solutions where possible. Not everything needs to be always-on,
  make use of things like Wake-on-LAN.

* Publish as much information, configuration and code as possible without
  compromising network security. This will enable others to learn & reuse.

* Have network recovery plans which can be executed rapidly. This will help me
  avoid getting stuck debugging a network issue when I need to be doing my job
  or relaxing.

* Have fun!

## Domain name and host names

![Gandi order completion page showing the purchase of the domain cephei.uk.](/img/2022-06-02_001.png)

The refreshed network needs a domain name. I always like to use a registered
domain name so that there is no possibility of confusion or URL hijacking. After
trying several ideas I found the domain cephei.uk was available and registered
it via my preferred provider for domain names, [Gandi](https://www.gandi.net/).
I really like this name, it's associated with space and stars as well as with my
[favourite Deadmau5 track](https://www.youtube.com/watch?v=ARNa-TxqZvQ).

The initial set of hostnames on this network will be based on the names of the
letters in the Greek alphabet. This gives me 24 hostnames to play with, from
alpha.cephei.uk to omega.cephei.uk. I doubt I'll need more than 24 hostnames for
a while and I can expand the naming convention in the future where necessary.

![Greek alphabet](/img/2022-06-02_002.png)

*Greek alphabet screenshot from
[Wikipedia](https://en.wikipedia.org/wiki/Greek_alphabet),
used under
[CC-BY-SA 3.0 license](https://en.wikipedia.org/wiki/Wikipedia:Text_of_Creative_Commons_Attribution-ShareAlike_3.0_Unported_License).*

Single function devices (e.g. smart network switches, access points) will not
use hostnames from the above list. Instead they will have simple functional
names with a digit appended (e.g. sw0 for a switch, ap0 for an access point).

General purpose devices will also have aliases where appropriate which identify
a function or service provided. For example, the first host I'll be configuring
will be alpha.cephei.uk and this will also act as a firewall/gateway router and
so will have an alias r0.cephei.uk.

## Initial technology selections

I intend to make use of the following technologies, services and applications in
building my network:

* [GitHub](https://github.com/) for hosting configurations, scripts and this
  netlog. I've created the [cephei-uk](https://github.com/cephei-uk)
  organization to hold the git repositories I'll create.

* [GitPod](https://www.gitpod.io/) and
  [Visual Studio Code](https://code.visualstudio.com/) to provide a
  straightforward remote development platform.

* [Python](https://www.python.org/) for most scripting and automation.

* [Ansible](https://www.ansible.com/) to deploy consistent configration across
  multiple hosts.

* [Markdown](https://www.markdownguide.org/) for notes and these netlog posts so
  I can focus on content and not syntax.

* [OpenSUSE MicroOS](https://microos.opensuse.org/) and 
  [Podman](https://podman.io/) as a small, atomic host environment for running
  containers.

* [Wireguard](https://www.wireguard.com/) for remote access to my network.
