After using UCARP for a while, I started to notice that it would sometimes fail to...uh...failover properly. My other big problem with it was very difficult to force a failover - you could kill the UCARP process and then try and start it by hand again with the right parameters, but it's not as clean as something that uses, say, an init script.

In order to rectify this I thought I'd play with Keepalived. I've now got a fairly basic configuration with one designated master and one designated backup, with a single IP address on a single virtual router instance. My only special configuration is to handle failover (using notify scripts), and to handle stopping Keepalived - that's particularly important, as it allows you to force a failover when performing maintenance on things or generally tinkering.

For my config, the following addresses are used:

* 192.168.0.252 - floating services VIP.
* 192.168.0.253 - primary services host.
* 192.168.0.254 - secondary services host.

When everything's fine, I want the primary host to own the services VIP, and the secondary to do nothing. My services include an always-on VPN connection to bridge my local network and global network of VPS', to give me access to things like mail, monitoring, file services/backups, etc. As a result of this, the primary and secondary services hosts must route the VPN traffic via the services address, not relying on them being connected to it themselves.

So, config is as follows (both in /etc/keepalived/keepalived.conf):

On the master:

    vrrp_instance VI_1 {
        state MASTER
        interface ens3
        virtual_router_id 51
        priority 150
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass $ SOMEPASSWORDHERE
        }
    
        virtual_ipaddress {
            # floating VIP.
            192.168.0.252/24
        }
    
        # send email alerts, start VPN, etc.
        notify /usr/local/sbin/keepalived.sh
        notify_stop "/usr/local/sbin/keepalived.sh 0 VI_1 STOP"
    }

Reasonably self-explanatory, I'll go into a couple of interesting bits:

* virtual_router_id - should be unique on any given network, gets used to identify the multicast group for peering, I'm informed that these should start at 51.
* priority - higher number is higher priority, I want the primary to take priority so I've set that to 150, and the secondary to 100.
notify - a script that expects a pre-defined set of arguments, can be used to notify the admin and handle failover events. I'll go into this in more detail later.
* notify_stop - a script to be called on service stop. I want the service being stopped to force the server into a secondary state, so I'm calling the same script but with hard-coded arguments to indicate a stop. Again, I'll cover this in the script later.

On the secondary the config's much the same, the only change is to the priority and the initial state:

    vrrp_instance VI_1 {
        state BACKUP
        interface eth0
        virtual_router_id 51
        priority 100
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass $ SOMEPASSWORDHERE
        }
        virtual_ipaddress {
            192.168.0.252/24
        }
    
        # send email alerts, start VPN, etc.
        notify /usr/local/sbin/keepalived.sh
        notify_stop "/usr/local/sbin/keepalived.sh 0 VI_1 STOP"
    }

Obviously it's imperative that the router ID and password match - it's advisable for things like the advert_int to match, otherwise you can get odd behaviour. Setting the state to BACKUP means that it will default to a secondary state before elections have occurred - this should stop both of the servers trying to take over if, say, they're both restarted at once. In practice I'm not sure this makes much difference (I have run it with both in initial state of MASTER and it's worked fine), but this is probably the best way to configure it.

That's about it for this config. My network interfaces are configured to route the VPN traffic over the services address with a metric of 10 - that way I don't need to mess around with the routing in my failover scripts, I can let the VPN client handle it for me on UP/DOWN events.

So, the script (same on both servers, in /usr/local/sbin/keepalived.sh):

    #!/bin/bash
    
    TYPE=$1
    NAME=$2
    STATE=$3
    
    function disableMaster(){
      echo 0 > /tmp/master_status
    }
    
    function enableMaster(){
      echo 1 > /tmp/master_status
    }
    
    function stopPPTP(){
      PPTPDACTIVE=$(pgrep pptp >/dev/null; echo $?)
      while [[ $PPTPDACTIVE -eq 0 ]]; do
        /usr/bin/poff personal
        sleep 5
        PPTPDACTIVE=$(pgrep pptp >/dev/null; echo $?)
      done
    }
    
    case $STATE in
            "MASTER") # route VPN directly via me, start VPN.
                      enableMaster
                      echo "Switching to master state" | mail -s "Keepalived failover - $(hostname)/$NAME" root
                      # start VPN
                      /usr/local/sbin/vpn.sh
                      exit 0
                      ;;
            "BACKUP") # stop VPN, route via VPN gateway.
                      disableMaster
                      echo "Switching to backup state" | mail -s "Keepalived failover - $(hostname)/$NAME" root
                      # stop VPN
                      stopPPTP
                      exit 0
                      ;;
            "STOP")  stop VPN.
                      disableMaster
                      echo "Stop detected!" | mail -s "Keepalived stopping - $(hostname)/$NAME" root
                      # stop VPN
                      stopPPTP
                      exit 0
                      ;;
            "FAULT")  # not sure what to do here.
                      disableMaster
                      echo "Fault detected! Parameters: $*" | mail -s "Keepalived fault - $(hostname)/$NAME" root
                      exit 1
                      ;;
            *)        # any other state
                      disableMaster
                      echo "Unkown state ($STATE) detected! Parameters: $*" | mail -s "Keepalived unknown state - $(hostname)/$NAME" root
                      exit 1
                      ;;
    esac

Again probably mostly self-explanatory with the comments, but essentially the key states to handle are MASTER, BACKUP, and FAULT. I've added in STOP for my notify_stop handler, so that I can force a failover while keeping the server itself running. I've also handled FAULT and UNKNOWN slightly differently, just to aid debugging later down the line - I've never actually seen the UNKNOWN trigger, but I've seen fault when experiencing local networking issues (e.g. during a power failure). To be safe, all I do is mark the server as no longer being a master, but don't try and mess with the state - that's enough to prevent any of my other cron jobs from running (as they check the master_state file before doing so), without potentially breaking things.
