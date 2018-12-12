# eap_proxy

Proxy EAP packets between interfaces on Ubiquiti Networks EdgeRouter™ products. Also works with UniFi® Security Gateway.

Inspired by 1x_prox as posted here:

<http://www.dslreports.com/forum/r30693618->

AT&T Residential Gateway Bypass - True bridge mode!

## Instructions (USG)

Please see <https://blog.taylorsmith.xyz/att-uverse-modem-bypass-unifi-usg/>
for detailed instructions.

Instructions summary:

1. Connect AT&T gateway into WAN2 on USG, ONT into WAN on USG
1. Use UniFi UI to configure USG:
  1. Configure WAN
     1. Enable DHCPv6 (prefix 60) and VLAN 0 on WAN
     1. New IPv6 firewall rule to accept ICMPv6 on WAN IN
  1. Configure WAN2
     1. Disable VOIP as WAN2
     1. Create a corp network 192.168.1.254/30 on WAN2
  1. Configure LAN
     1. Enable IPv6 on LAN (enable DHCPv6 ?)
     1. Configure IPv6 DNS?
1. Setup magical shell/python scripts:
   1. Copy them to USG
      ```sh
      cat eap_proxy.py | ssh gateway sudo tee /config/scripts/eap_proxy.py
      cat eap_proxy.sh | ssh gateway sudo tee /config/scripts/post-config.d/eap_proxy.sh
      ssh gateway sudo chmod +x /config/scripts/post-config.d/eap_proxy.sh
      ```
   1. Verify that everything works:
      ```sh
      ssh gateway sudo python /config/scripts/eap_proxy.py \
        --restart-dhcp --ignore-when-wan-up --ignore-logoff \
        --ping-gateway --set-mac eth0 eth2
      ```
      And power cycle AT&T gateway, wait 5 min and see many `EAP packet` in logs
      and finally `restarting dhclient`.
      You should be able to access internet now.
   1. Restart USG and verify that you connect back to the internet.

> Debugging when something goes wrong:
> ```sh
> # Check logs
> tail -n 500 -f /var/log/messages
> # Verify service is running
> pa sux | grep eap
> ```

> Note: After factory reset EAP proxy scripts would be lost.

## Usage

```
usage: eap_proxy [-h] [--ping-gateway] [--ignore-when-wan-up] [--ignore-start]
                 [--ignore-logoff] [--restart-dhcp] [--set-mac] [--daemon]
                 [--pidfile PIDFILE] [--syslog] [--promiscuous] [--debug]
                 [--debug-packets]
                 IF_WAN IF_ROUTER

positional arguments:
  IF_WAN                interface of the AT&T ONT/WAN
  IF_ROUTER             interface of the AT&T router

optional arguments:
  -h, --help            show this help message and exit

checking whether WAN is up:
  --ping-gateway        normally the WAN is considered up if IF_WAN.0 has an
                        IP address; this option additionally requires that
                        there is a default route gateway that responds to a
                        ping

ignoring router packets:
  --ignore-when-wan-up  ignore router packets when WAN is up (see --ping-
                        gateway)
  --ignore-start        always ignore EAPOL-Start from router
  --ignore-logoff       always ignore EAPOL-Logoff from router

configuring IF_WAN.0 VLAN:
  --restart-dhcp        check whether WAN is up after receiving EAP-Success on
                        IF_WAN (see --ping-gateway); if not, restart dhclient
                        on IF_WAN.0
  --set-mac             set IF_WAN.0's MAC (ether) address to router's MAC
                        address

daemonization:
  --daemon              become a daemon; implies --syslog
  --pidfile PIDFILE     record pid to PIDFILE
  --syslog              log to syslog instead of stderr

debugging:
  --promiscuous         place interfaces into promiscuous mode instead of
                        multicast
  --debug               enable debug-level logging
  --debug-packets       print packets in hex format to assist with debugging;
                        implies --debug
```
