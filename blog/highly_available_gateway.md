## Overview

With my variety of distribute VPS', I run a Virtual Private Network (VPN) that allows them all to connect to central services - such as mail, OSSEC, DNS, rsync to the backups server, etc. It also allows me to firewall SSH off from everywhere except the VPN, providing extra protection against attacks. However, when I'm at home, I want to always be connected to the VPN - that allows me to access those servers, and route traffic via the VPN gateway in order to access services that have firewalling restricting access to "me". I don't have a static IP address at home, so tend to firewall things off to only allow access from my VPN server's external IP address - that allows me to access things from home, or while I'm out and about, as long as I'm connected to the VPN.

When I'm at home, I used to have network manager configured to connect to my VPN automatically when I connected to my wireless network - however these days I've got multiple servers at home, which need to access those shared services I mentioned. You *can* have multiple PPTP clients connected behind a single IP address, but it relies on the NAT router understanding how to differentiate between them, and this is flaky at the best of time. My solution to this is to have a VPN gateway for these services, which is permanently connected to the VPN - it can then also relay DNS & mail on behalf of the local network, and allow my monitoring server (which lives on the VPN) to check services at home. However, this is a single point of failure.

## The solution

It's easy to bring up or take down a PPTP connection with a single command, providing you've done all the config before hand to make it work. In this case I had, so I just copied the config over to the new secondary gateway, and made sure that both could connect individually. Then I decided to look into options - most people say you should use keepalived with custom startup and tear-down scripts. That's fine, but I thought I'd take the opportunity to look into a lesser-known tool - UCARP. UCARP is the Linux implementation of the Common Address Routing Protocol - designed to ensure that a given address always exists somewhere on the network. Once installed, the configuration for it is all handled in /etc/network/interfaces - which is arguably messy in some ways, but I feel provides an heir of simplicity - you don't need to know any extra config syntax, and bringing the device up and down is as simple as taking the interface up and down.

# Primary config

The config on the servers should be identical, with the exception of marking one as primary - lots of people say that for UCARP to work effectively you have to set both servers to be the same priority, and let them fight it out. That means that you'll always have a gateway, but don't know which is which - I want a designated primary and secondary, so haven't done that.

You'll need to install the UCARP package, and then add config like this to /etc/network/interfaces (assuming eth0 is your real interface);

    auto eth0
    iface eth0 inet static
    	address 192.168.0.254
    	netmask 255.255.255.0
    	network 192.168.0.0
    	gateway 192.168.0.1
    	broadcast 192.168.0.255
    	dns-nameservers 127.0.0.1
    	dns-search home
    	post-up ip route add 192.168.2.0/24 via 192.168.0.252
    	pre-down ip route del 192.168.2.0/24 via 192.168.0.252
    	ucarp-vid      1
    	ucarp-vip      192.168.0.252
    	ucarp-password foobar-fish
    	ucarp-advskew  1
    	ucarp-advbase  1
    	ucarp-master   yes

    # ucarp VIP
    iface eth0:ucarp inet static
    	address 192.168.0.252
    	netmask 255.255.255.0
    	network 192.168.0.0
    	broadcast 192.168.0.255
    	pre-up ip route del 192.168.2.0/24 via 192.168.0.252
    	post-up ip route add 1.2.3.4 via 192.168.0.1 dev eth0:ucarp src 192.168.0.252
    	post-up pon vpn
    	pre-down poff vpn
    	pre-down ip route del 1.2.3.4 via 192.168.0.1 dev eth0:ucarp src 192.168.0.252
    	post-down ip route add 192.168.2.0/24 via 192.168.0.252

This declares a virtual interface (: interfaces are deprecated, so in the future I might look to move this onto a dummy or bridge interface instead) on top of eth0 - currently UCARP seems to expect to find an interface with :ucarp on it in interfaces, so I've done that. Then you declare the address details as normal - for your floating virtual IP address, not the real one. My pre-up, post-up, pre-down, and post-down commands handle the failover steps for the VPN - adjusting the gateway and routing config to behave as necessary. This will obviously vary for each environment. The normal settings for eth0 are up to you, and the ucarp-* settings on the real interface (eth0) are as follows;

* ucarp-vid - a virtual ID for the peering. This must be unique across a UCARP environment - it uses multicast, so ideally keep it unique across an entire subnet. This must match on each of the peers.
* ucarp-vip - the virtual, floating IP address of the adapter.
* ucarp-password - a shared secret to use for communication between the multicast peers.
* ucarp-advskew - the host with the lowest value here will be the master in a master-slave set-up.
* ucarp-advbase - the frequency (in seconds) of communication between peers.
* ucarp-master - declares that there is a master in this relationship - as against letting them fight it out.

## Secondary config

The config should be nearly identical, with the exception of the advskew value - the master should be 1, the slave should be 2. You can have multiple slaves, just remember that a lower number is a higher priority. As UCARP uses multicast, there's no value to declare your peers - authentication is done solely on a peer having the shared secret.

Once configured, you should test your set-up - this can be done by taking the interface down on the primary, or rebooting the primary. If all goes well, you'll see the following in syslog on the slave:

failover;

    ucarp[289]: [WARNING] Switching to state: MASTER
    ucarp[289]: [WARNING] Spawning [/usr/share/ucarp/vip-up eth0 192.168.0.252]

recovery;

    ucarp[289]: [WARNING] Switching to state: BACKUP
    ucarp[289]: [WARNING] Spawning [/usr/share/ucarp/vip-down eth0 192.168.0.252]
    ucarp[289]: [WARNING] Preferred master advertised: going back to BACKUP state
