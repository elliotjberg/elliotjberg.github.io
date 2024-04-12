# Overview
At home, I run a local DHCP and DNS server, using isc-dhcp-server and bind9, respectively - this is for multiple reasons, such as;

* Commodity routers' DHCP implementations don't allow custom options, and I need to push custom routes to clients, in order to utilise resources on my VPN, which are accessed via a gateway other than the router.
* Commodity routers' DNS implementations often don't provide local zones which auto-update with DHCP leases, and I've yet to find one that allows adding custom records on top of that.

Previously this has been running on a single server, which has been sufficient for a couple of years - but recently I've been doing a lot of tinkering with said server, which has periodically broken DHCP and DNS. So, I thought this was a good opportunity to experiment with high availability DHCP and DNS.

For the purposes of this, my addresses are the following:

* 192.168.0.254 - primary DNS + DHCP server
* 192.168.0.253 - secondary DNS + DHCP server

## DHCP
There are a couple of ways of doing this - if you don't care about the addresses that are given to devices, you could just have a second DHCP server, and have some kind of keepalived set-up that started the second one when the first disappeared. But, it turns out that isc-dhcp-server supports a proper fail-over capability. To configure it, you add the following to the primary server's /etc/dhcp/dhcpd.conf (adjusting the names and addresses as appropriate):

    failover peer "dhcp-failover" {
      primary;
      address 192.168.0.254;
      port 647;
      peer address 192.168.0.253;
      peer port 647;
      max-response-delay 30;
      max-unacked-updates 10;
      load balance max seconds 3;
      mclt 1800;
      split 128;
    }

the important parts of that are;

* primary = indicates that this is the primary peer
* dhcp-failover = a custom name, it doesn't matter what it is, but must match between the servers
* address = the local address of the primary server
* port = the local TCP port for the primary server to use for communications (must be open on the firewall to receive traffic from the secondary server)
* peer address = the address of the secondary server
* peer port = the TCP port for the secondary server to use for communications (must be open on the firewall to receive traffic from the primary server)

You'll also need to adjust your subnet definition to include a pool and fail-over section, like this;

    subnet 192.168.0.0 netmask 255.255.255.0 {
      option routers router.home;
      pool {
        range 192.168.0.15 192.168.0.245;
        failover peer "dhcp-failover";
      }
    }

on the secondary server you'll need to add (also to /etc/dhcp/dhcpd.conf):

    failover peer "dhcp-failover" {
      secondary;
      address 192.168.0.253;
      port 647;
      peer address 192.168.0.254;
      peer port 647;
      max-response-delay 30;
      max-unacked-updates 10;
      load balance max seconds 3;
    }

the important parts of that are;

* secondary = indicates that this is the secondary peer
* dhcp-failover = a custom name, it doesn't matter what it is, but must match between the servers
* address = the local address of the secondary server
* port = the local TCP port for the secondary server to use for communications (must be open on the firewall to receive traffic from the primary server)
* peer address = the address of the primary server
* peer port = the TCP port for the primary server to use for communications (must be open on the firewall to receive traffic from the secondary server)

You'll also need to adjust your subnet definition to include a pool and fail-over section, to match the primary server. Other than this, the two servers should have identical config - this should include things like reserved addresses, and they'll need to be issuing the same range of addresses.

If all goes well, you'll see entries like this in syslog for normal use:

dhcpd: DHCPDISCOVER from 00:24:e9:05:bb:a1 via eth0: load balance to peer dhcp-failover

and entries like this when failover takes place and recovers:

failover:

    dhcpd: peer dhcp-failover: disconnected
    dhcpd: failover peer dhcp-failover: I move from normal to communications-interrupted

recovery:

    dhcpd: failover peer dhcp-failover: peer moves from normal to normal
    dhcpd: failover peer dhcp-failover: I move from communications-interrupted to normal
    dhcpd: failover peer dhcp-failover: Both servers normal

## DNS

DNS in principle is easy, because you can specify multiple DNS servers when pushing them out via DHCP. However, my local DNS zone is automatically updated to include forward and reverse maps for anything that gets an address via DHCP - that needs to be replicated between the servers. Making the primary server a master, and the secondary server a slave, will achieve this - but means that when the primary server (for both DHCP and DNS) fails, the updates won't take place. This is easily solved by adding some config to bind9 to instruct it to pass these onto the primary server if received, and telling the DHCP server to pass on the mapping requests to multiple servers.

First let's do the bind configuration. On the primary server, you'll simply need to add allow-transfer options to the zones you'll be replicating in /etc/bind9/named.conf.local , like this (adjusting the allow-transfer as appropriate);

    zone "home" {
        type master;
        file "home.zone";
        allow-transfer { 192.168.0.253; };
        allow-update { key DHCP_UPDATER; };
    };
    
    zone "0.168.192.in-addr.arpa"  {
        type master;
        file "db.0.168.192.in-addr.arpa";
        allow-transfer { 192.168.0.253; };
        allow-update { key DHCP_UPDATER; };
    };

The allow-update section is from my existing auto-update from DHCP configuration, and rather than allowing certain addresses, allows it from anyone with the correct key. I'm sharing the same key between the DNS and DHCP servers, for ease, and as they're running on the same two servers.

On the secondary server, you'll need to add config like the following to /etc/bind9/named.conf.local (adjusting the master as appropriate);

    zone "home" {
        type slave;
        file "home.zone";
        masters { 192.168.0.254; };
        allow-update-forwarding { key DHCP_UPDATER; };
    };

    zone "0.168.192.in-addr.arpa"  {
        type slave;
        file "db.0.168.192.in-addr.arpa";
        masters { 192.168.0.254; };
        allow-update-forwarding { key DHCP_UPDATER; };
    };

The allow-update-forwarding section means that when this server receives requests for updates from the DHCP server, it should store them up and pass them onto the primary server when it's back up, providing it was signed by the DHCP_UPDATER key (the same key used for the normal updates).

You then also need to adjust the DHCP server configuration (in /etc/dhcp/dhcpd.conf) to say the following under the subnets you'll push updates for (this should be identical on the primary and secondary servers);

    zone home. {
      primary 192.168.0.254;
      secondary 192.168.0.253;
      key DHCP_UPDATER;
    }

    zone 0.168.192.in-addr.arpa. {
      primary 192.168.0.254;
      secondary 192.168.0.253;
      key DHCP_UPDATER;
    }

Now that they're both running, edit the DNS server section of /etc/dhcp/dhcpd.conf to include all of your DNS servers, in whatever order's appropriate, something like this;

option domain-name-servers 192.168.0.254, 192.168.0.253;

If all's gone well, you should now have a high-availability DHCP and DNS configuration, where any combination of the DNS and DHCP servers can be used, as long as there's at least one DHCP server running and at least one DNS server running.
