# lxc-devuan template

LXC container template for creating Devuan LXC containers.

Supported releases: jessie, ceres, ascii

---

## Prerequisite

LXC itself must first be installed and working, obviously.

You can either install LXC from your OS distro repository, or get  
the latest version from https://linuxcontainers.org/lxc/downloads/.

If your distro version is old consider downloading the latest.

---

## Quick Setup

1. Clone this repo:

    ```shell
    $ cd ~
    $ git clone git@git.devuan.org:gregolsen/lxc-devuan.git
    ```

2. Copy config and template files:

    Choose A or B depending on how LXC was installed.

    A. LXC installed from OS distro: /usr/share/lxc
  
    ```shell
       $ cp ~/lxc-devuan/config/*     /usr/share/lxc/config/
       $ cp ~/lxc-devuan/templates/*  /usr/share/lxc/templates/
    ```

    B. LXC installed locally: /usr/local/share/lxc
  
    ```shell
       $ cp ~/lxc-devuan/config/*    /usr/local/share/lxc/config/
       $ cp ~/lxc-devuan/templates/* /usr/local/share/lxc/templates/
    ```

That's it.

---
  
## Install Devuan Containers

<br>Ex 1. Install Devuan Jessie (amd64)
  
   ```shell
   $ sudo lxc-create -t devuan -n devuan-jessie-box1
   ```

<br>Ex 2. Install Devuan Jessie with ZFS backingstore (amd64)
   
   ```shell
   $ sudo lxc-create -t devuan -n devuan-jessie-box2 -B zfs
   ```
   - To use *backingstore* `-B zfs` the ZFS root device is specified in `/etc/lxc/lxc.conf`:  
     `lxc.bdev.zfs.root = zfs/srv/lxc`     <== Example ZFS root device

<br>Ex 3. Install Devuan Jessie (i386)
  
   ```shell
   $ sudo lxc-create -t devuan -n devuan-jessie32-box3 -- -a i386
   ```

   >**Hint:**
   Don't forget the double dash '--' before the --arch parameter!

<br>Enjoy your new Devuan containers!

---

## General Features

- Automatically downloads devuan-keyring for GPG package verification
- Timezone is set correctly from host system
- `/etc/apt/apt.conf.d/01lean` to help keep containers minimal
- gcc-4.8-base purged in favor of gcc-4.9-base
- Commonly used console-based tools pre-installed

## Network Features

- Useful network tools pre-installed
- Correctly updated `/etc/hosts`
- Resolvconf configured to auto-update `/etc/resolv.conf`
- Random MAC generation supports overriding the OUI prefix. Default `OUI='00:16:3e'`
- Supports pre-assigned MAC address (*network.hwaddr*) at creation time (see *Advanced Networking*)

---

## LXC Documentation

A lot of good LXC documentation is available so there's little 
point in attempting to reproduce it here.

This is an excellent LXC blog series: 
https://www.stgraber.org/2013/12/20/lxc-1-0-blog-post-series/

Also see lxc manpage: $ man lxc

NB. Some basic commands to get things going:

|  Action            |  Command                             |
| ------------------ | ------------------------------------ |
|  List containers   |  lxc-ls -f                           |
|  Start container   |  lxc-start   -n container            |
|  Container info    |  lxc-info    -n container            |
|  Launch console    |  lxc-console -n container            |
|  Shell directly in |  lxc-attach  -n container            |
|  Execute command   |  lxc-attach  -n container -- command |
|  Stop container    |  lxc-stop    -n container            |
|  Destroy container |  lxc-destroy -n container            |

---

## Advanced Networking

This section describes how to use lxc-devuan to *pre-assign* a container *MAC* address, and 
how this can be used for automatic Dnsmasq static IP assignment and DNS name resolution.

>If you have no interest in doing any manual *Host network* configuration, please skip this section.  
>  
>**NB.** This is not intended as a Network or Dnsmasq howto. Fundamental network 
>knowledge is needed to fill in the gaps, and **_examples must be adapted_** to your 
>specific configuration.

<br>
On my Workstation I run a *routed* Xen VM/LXC configuration with a 
multi-zone Shorewall firewall, and use Dnsmasq for both *static* 
and *dynamic* IP assignments as well as DNS name resolution.

Assigning dynamic IPs from an IP-range is basic to any DHCP setup,
however there's more than one way to handle static IP assignment.

For my own test beds, I dislike administering static IPs locally 
in each VM/Container that needs one. It's much better, and easier, 
to let Dnsmasq handle things more automatically (once you learn how).

The key here is configuring Dnsmasq to use:
- `/etc/ethers` to map the *MAC address* to a *static* IP [Ex. 4]
- `/etc/hosts` for resolving *hostname* to *static* IP [Ex. 5]

To enable both behaviors, `/etc/dnsmasq.conf` must have **read-ethers**, and must
**not** have **no-hosts**.

File `/etc/ethers.used` is where you define/track all of the *pre-assigned* MACs, IPs, 
and Hostnames [Ex. 6].  

*lxc-devuan* uses `/etc/ethers.used` to pre-assign the MAC address when creating the 
Devuan container.

<br>Ex 4. Host `/etc/ethers` -- for *static IP* assignment by Dnsmasq

```
# Ethernet address to IP number database
#
# LXC
00:16:3e:00:91:01 10.0.145.1
00:16:3e:00:91:02 10.0.145.2
00:16:3e:00:91:03 10.0.145.3
```

<br>Ex 5. Host `/etc/hosts` -- for container *hostname* resolution by Dnsmasq

```
# IP addr       Hostname                  Alias
127.0.0.1       localhost
127.0.1.1       hostname.mylandomain.com  hostname
# The following lines are desirable for IPv6 capable hosts.
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
#
### VM/LXC SECTION - DO NOT CHANGE THIS LINE ###
# IP addr       Hostname                                Alias
10.0.145.1      devuan-jessie-box1.mylandomain.com      devuan-jessie-box1
10.0.145.2      devuan-jessie-box2.mylandomain.com      devuan-jessie-box2
10.0.145.3      devuan-jessie32-box3.mylandomain.com    devuan-jessie32-box3
```

<br>Ex 6. Host `/etc/ethers.used` -- for MACs assigned by lxc-devuan (and tracking MAC/IP/LXC assignments)

```
# -*- mode: conf-space -*-
#
#  MAC Address    Static IP    Container-hostname    Comment (distro-version)
00:16:3e:00:91:01 10.0.145.1   devuan-jessie-box1    devuan-jessie64
00:16:3e:00:91:02 10.0.145.2   devuan-jessie-box2    devuan-jessie64
00:16:3e:00:91:03 10.0.145.3   devuan-jessie32-box3  devuan-jessie32
```

>**Note:**
lxc-devuan uses Container *name* to locate the MAC address.

<br>
Example 4 shows the simple MAC to IP mapping used by Dnsmasq. The DHCP client with a MAC 
that matches an entry in `/etc/ethers` will be assigned/offered the corresponding IP.

Example 5 is a standard `/etc/hosts` file with entries for the Devuan 
containers. Dnsmasq will provide DNS name resolution for `/etc/hosts` 
entries on all bound/listening interfaces.

The new network feature provided here by lxc-devuan, is to pre-assign 
the container *MAC* from file `/etc/ethers.used` (during *lxc-create*).
The container *name* is used to locate the MAC [Ex. 6 above].

The resulting *"Network configuration"* section in the container `./config` file, 
is generated with the MAC from `/etc/ethers.used` [Ex. 7, see *lxc.network.hwaddr*].

<br>Ex 7. Network config for 'devuan-jessie-box1' (as generated by lxc-devuan)

```
    [snip]
# Network configuration
lxc.network.type   = veth
lxc.network.flags  = up
lxc.network.link   = lxcbr0
lxc.network.hwaddr = 00:16:3e:00:91:01  # 10.0.145.1 devuan-jessie-box1
lxc.network.ipv4   = 0.0.0.0
```

>The trailing comment with IP and LXC name is also pulled from `/etc/ethers.used`.  
>**NB.** The `lxc.network.ipv4 = 0.0.0.0` address is used when IP is DHCP assigned.

<br>
With this setup, your nice shiny Devuan containers will have pre-assigned 
static IPs, and DNS name resolution working properly right from the 
initial *lxc-create* (no container admin required).

Another benefit of centrally managing the IP addressing this way is,
you can easily change your addressing scheme without having to locally 
administer the net config in each container.

#### Pertinent Network Configuration Details  

It was never my intent to get into this level of detail. However certain 
Network/Dnsmasq configuration options are necessary to make all this work 
as intended.

Dnsmasq should be permitted to read/poll the *host* `/etc/resolv.conf`.

To ensure this happens, the Dnsmasq options **no-resolv** and **no-poll** 
must be **removed** or commented from `/etc/dnsmasq.conf`. This is 
probably already the default on your system, but you might want to check it.

<br>Ex 8. Host `/etc/resolv.conf` - generated by Host *resolvconf*

```
   nameserver 127.0.0.1
   nameserver 192.168.0.1
   search mylandomain.com
   # tail
   nameserver 208.67.222.222
   nameserver 208.67.220.220
```

>- First nameserver queries the local *Host* DNS cache (cached by Dnsmasq)
>- Second nameserver is the *upstream* in-network DNS (can be combined Router/DNS)
>- Search domain specifies same *domain* name as the LAN domain
>- Tail nameservers are OpenDNS servers (**not** from IP provider)
>
>**NB.** If using a Consumer-grade Router for your LAN, substitute "Router/DNS" 
>for "DNS" (unless of course you have a separate DNS).

<br>
Devuan containers are pre-configured by lxc-devuan with *resolvconf*. Therefore 
once Dnsmasq is properly configured, the *Container* `/etc/resolv.conf` 
will be similar to the one above, excluding local DNS cache (see *Example 10*).

In addition to the **_IP address_**, **_Subnet mask_**, and **_Gateway IP_**, it's 
important the **_Host DNS Server_** *(Dnsmasq)* and **_Domain name_** are also DHCP 
assigned to the Container (by Dnsmasq).

Following is a good example of the *bridge-specific* Dnsmasq config, 
with the necessary DHCP options.

<br>Ex 9. Dnsmasq config specific to *bridge lxcbr0*

```
   # listen on lxcbr0 bridge address
   listen-address=10.0.0.1

   # LXC dynamic range 10.0.143.1 <--> 143.254
   dhcp-range=set:lxc0,10.0.143.1,10.0.143.254,255.255.0.0,24h

   # IP subnet 10.0/16 for STATIC /etc/ethers MAC:IP assignment
   dhcp-range=set:lxc0,10.0.0.1,static,255.255.0.0,infinite

   # Subnet Mask                 (1  = option:netmask)
   dhcp-option=tag:lxc0,1,255.255.0.0

   # Gateway IP                  (3  = option:router)
   dhcp-option=tag:lxc0,3,10.0.0.1

   # DNS Servers                 (6  = option:dns-server)
   dhcp-option=tag:lxc0,6,10.0.0.1

   # Domain Name                 (15 = option:domain-name)
   dhcp-option=tag:lxc0,15,mylandomain.com

   # IP Forward                  (19 = option:ip-forwarding  0=disable 1=enable)
   dhcp-option=tag:lxc0,19,0

   # Source Routing              (20 = option:?  0=disable 1=enable)
   dhcp-option=tag:lxc0,20,0

   # Broadcast                   (28 = option:broadcast  0.0.0.0 references local machine)
   dhcp-option=tag:lxc0,28,10.0.255.255
```
>**NB.**
The example above is not the entire Dnsmasq configuration.

<br>
With Dnsmasq properly configured for DHCP on bridge lxcbr0, 
*resolvconf* will then correctly generate the *Container* `/etc/resolv.conf`.

<br>Ex 10. Container `/etc/resolv.conf` - generated by Container *resolvconf*

```
   # Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
   #     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
   nameserver 10.0.0.1
   search mylandomain.com
   # tail
   nameserver 208.67.222.222
   nameserver 208.67.220.220
```

>- First nameserver (DHCP assigned) is the *Host* Dnsmasq (listening on lxcbr0 IP)
>- Search domain (DHCP assigned) is the same *domain* name as the LAN domain
>- Tail nameservers are OpenDNS servers (**not** from IP provider)

<br>
The final pieces of the Dnsmasq puzzle are the following options in `/etc/dnsmasq.conf`:
  * **strict-order**
  * **domain=**mylandomain.com
  * **expand-hosts**

>**Warn:** Unless you know what you're doing, please do **not** put the 
>Host Dnsmasq server into **authoritative** mode.  Leave **dhcp-authoritative** 
>commented or **remove it** completely.

<br>
The above *common* Dnsmasq options ensure the following behaviors:
  * The first *upstream server* contacted is your *in-network* DNS (**strict-order**)
  * Host Dnsmasq has the same *domain* as your *in-network* DNS (**domain**)
  * The *domain* name is appended to simple names for DNS lookup (**expand-hosts**)
  * Containers can connect to local VMs/Containers by *simple hostname*
  * Containers can connect to LAN servers/workstations by *simple hostname*  
    (provided *IP Masquerading* is configured on the Host *firewall*, or a *route* is available)

<br>
**NB.** To go in the other direction and connect from LAN to LXC container by its name, 
requires admin on both the Router and DNS.  Minimally you'll need a **route** to the *LXC host* 
for the *LXC subnet*, and a DNS **A record** for each *Container hostname* you want resolved.
Plus, the traffic must also be allowed through the Host firewall.

If your Router runs a Dnsmasq daemon (Ex. Consumer-grade Routers), and the router firmware 
supports it (Ex. DD-WRT), DNS **A records** can be added using the **host-record** option.

The Dnsmasq **host-record** option comes with the benefit of also adding the **PTR record**.

Router/DNS admin is beyond the scope here. However if your Router runs Dnsmasq, and you're up to 
the task, below is an example Dnsmasq **host-record** for an LXC container.

<br>Ex 11. Add DNS **A** and **PTR records** to Dnsmasq daemon on Router/DNS

```shell
   host-record=devuan-jessie-box2.mylandomain.com,10.0.145.2
```

<br>
>**Advice:**
If you're setting up the Host Dnsmasq for the first time, get a *dynamic* IP **dhcp-range** working first. 
Then add a *static* IP **dhcp-range**, assigning MACs/IPs using `/etc/ethers` and `/etc/ethers.used`. 
Finally, remember to add the LXC IP/Hostname/Alias to `/etc/hosts`.

---

## Test Machine Specs

This was tested using LXC 1.1.5 on my Workstation/Laptop.
However it'll probably also work with earlier LXC versions.

#### Hardware  
**CPU:** 8 x x86_64 Intel(R) Core(TM) i7-2960XM CPU @ 2.70GHz  
**RAM:** 32 GB Low-latency  
**Storage:**  2TB -- **_Boot:_** RAID 1, **_OS:_** RAID 1 + LVM, **_Storage:_** ZFS mirror (3 GiB ARC) w/Dedup  
**Ethernet:** Intel Corporation 82579LM Gigabit Network Connection (rev 04)  

#### Software
**Hypervisor:** Xen version 4.3.0  
**Kernel:**     Upstream linux-stable 4.4.1, custom-patched with ZFS 0.6.5.4, Aufs 4.4, & Xen security  
**OS:**         Debian-based system, originally LMDE 1 (UP8), plus some Jessie, Stretch and Sid  
**LXC:**        Source: git://github.com/lxc/lxc, compiled version 1.1.5

---

## Known Issues

### Issue #1 - DHCP not working - Transmit (Tx) checksum offload problem

#### Synopsis: Container can't get DHCP assigned IP address

The failed sequence looks like this:  
- Client DHCPDISCOVER(lxcbr0), followed by Server DHCPOFFER(lxcbr0)
- Client does **not** follow with DHCPREQUEST as required per protocol
- Consequently neither will DHCPACK be sent from server to acknowledge

Don't blame Devuan for this.

It's an old issue related to Linux transmit (Tx) checksum offload 
handling for virtual devices such as:  
- bridges, veth devices, etc.  

For a more in-depth explanation see **_Root Cause_** below.

#### Issue #1 - Solution

Here's the solution I've used reliably for many years with my Xen VMs, 
and more recently with LXC:

1. Install ethtool:

    ```shell
    $ sudo apt-get install ethtool
    ```

2. Turn off Tx checksum offloading on the LXC bridge:

    There's more than one way to do this, but the cleanest way
    I've found is to simply add an "up" command to the bridge
    interface definition.

    Example 'up' command ($IFACE = bridge interface):  

    ```shell
    up /sbin/ethtool -K $IFACE tx off  # <== TURN OFF TX CHECKSUM OFFLOAD
    ```

    I manually define all my bridges.

    Example bridge definition in `/etc/network/interfaces`:

    ```
    auto lxcbr0
    iface lxcbr0 inet static
            pre-up    brctl addbr $IFACE
            address   10.0.0.1
            netmask   255.255.0.0
            network   10.0.0.0
            broadcast 10.0.255.255
            bridge_stp off                    # disable Spanning Tree Protocol
            bridge_waitport 0                 # no delay before a port becomes available
            bridge_fd 0                       # no forwarding delay
            up        ip link set $IFACE up
            up        /sbin/ethtool -K $IFACE tx off  # <== TURN OFF TX CHECKSUM OFFLOAD
            down      ip link set $IFACE down
            post-down brctl delbr $IFACE
    ```

    <br>
    *CentOS/RHEL/SUSE* example (**untested**):

    Add to interface config `/etc/sysconfig/network/ifcfg-lxcbr0`:

    ```shell
    ETHTOOL_OPTIONS='-K iface tx off'
    ```

<br>
Other solutions include, setting an iptables rule on either the 
POSTROUTING or OUTPUT chains of the *mangle* table. At this time 
I don't do this, therefore I don't have an example to provide.

IMHO, the iptables rule solution might be prone to breakage as 
subsequent rule changes can lose the checksum rule.

However if maximum performance on a prod server is important, 
I suggest using the **OUTPUT chain** of the *mangle* table to 
calculate checksums for DHCP.

#### Issue #1 - Root Cause

The root cause of the issue is a missing or incorrect checksum for packets 
transmitted from virtual devices, and a *DHCLIENT* that rejects packets with 
bad checksums.

There's a lot of confusion about the cause.

First and foremost, know this is **not** a problem with Devuan, 
and nor is it a problem with LXC.

It's an old issue related to Linux transmit (Tx) checksum *offload* handling 
for virtual devices:  
- bridge's, veth's, etc.  

The kernel Linux defers calculating checksums until packet *egress* on 
**physical devices** only.

Therefore this is not a kernel bug, but a design decision.

As I understand it, it's because there's no need to calculate checksum's 
when packets transmit via in-memory copy, as is the case for virtual devices. 
Makes sense. Why consume CPU performing the calculation when there's no 
physical network?

There's also the *partial checksum offload* feature which was introduced 
as a performance optimization. I don't fully understand it. Apparently it 
has resulted in incorrect checksums that can cause DHCP to reject packets.

Disabling Tx checksum *offload* circumvents the issue.

On the surface it may seem like there's a problem with the *DHCLIENT*, especially 
if other VMs/Containers work fine on the same bridge. However all this 
implies is some patching has been done to accept packets with bad checksums.
Depending on ones perspective this is either good, or it's bad. IMHO, both 
perspectives have some merit.

#### Issue #1 - Related links
[Fix Debian base boxes networking issue #153](https://github.com/fgrehm/vagrant-lxc/issues/153)  
[Bug #1244589 VM can't get DHCP lease due packet has no checksum](https://bugs.launchpad.net/neutron/+bug/1244589)  
[Don't rely on setting `ethtool tx off` on guest interfaces #1255](https://github.com/weaveworks/weave/issues/1255)  
[DHCP not working due to bad udp checksum - TX offloading problem #40](https://github.com/projectcalico/calico/issues/40)  
[dhcp client not working properly in Xen domU due to partial checksum offload](https://bugzilla.novell.com/show_bug.cgi?id=668194)  
[Always fill UDP checksums in DHCP replies](https://review.openstack.org/#/c/148718/)  
[Add an iptables mangle rule per-bridge for DHCP](https://review.openstack.org/#/c/18336/4)  
[KVM guests networking issues with no virbr0 and with vhost_net kernel modules loaded](https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/1029430)  
[dhcp3-server reports many bad udp checksums to syslog using virtio NIC](https://bugs.launchpad.net/ubuntu/+source/isc-dhcp/+bug/930962)  

---

## Release History

* 1.0.0
    * Initial release

---

## Meta

<p>Greg Olsen – <a href="https://www.linkedin.com/pub/greg-olsen/1/179/a4a"><img src="https://static.licdn.com/scds/common/u/img/webpromo/btn_viewmy_160x33.png" width="160" height="33" border="0" alt="View Greg Olsen's profile on LinkedIn"></a> – gregolsen@computer.org</p>

Distributed under the LGPL license. See [LICENSE.md](LICENSE.md) for more information.
  
---
