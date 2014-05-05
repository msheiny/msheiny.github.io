---
layout: post
title: "Mikrotik and Me"
date: 2014-03-17 17:11:24 -0400
comments: false
categories: [mikrotik, networking, router, ipv6]
---

{% img left /images/mikrotik_rtrb_750.jpg 400 %}

So I stumbled across a company out of Latvia that produces a line of hardware routers
and manages their own Linux-based network operating system. The hardware is called
`Routerboard` and the operating system is `RouterOS`. I needed a cheap but feature-filled home router and ended up picking up the RB750GL off Amazon. I can’t speak to performance or stability yet but so far I’m very happy with the extensibility and feature-set of RouterOS. I’ve always run dd-wrt in the past but have been turned off by the lack of documentation and the hoops you have to jump through to do more advanced tasks outside the web interface.
ssh logins

## SSH key logins ###
You must utilize DSA ssh keys if you’d like to use ssh key authentication. From a linux host use `ssh-keygen` and the `-t` argument:

```
ssh-keygen -t dsa -i .ssh/mikrotik
scp .ssh/mikrotik.pub $mikrotik_ip:
```

on the mikrotik:

    
```
[msheiny@mikrotik] > /user ssh-keys import public-key-file=mikrotik.pub user=msheiny
```

## IPv6 with comcast ##

Comcast uses [DHCP-PD](http://tools.ietf.org/html/rfc3633) to pass along our /64 network prefix. Add a dhcpv6 client to our external interface:

```
/ipv6 dhcp-client
add add-default-route=yes interface=ether1-gateway pool-name=comcast
```

and then add firewall rules:

```
/ipv6 firewall filter
add chain=input connection-state=established
add chain=input connection-state=related
add chain=input dst-port=546 in-interface=ether1-gateway protocol=udp
add chain=input in-interface=ether1-gateway protocol=icmpv6
add action=drop chain=input in-interface=ether1-gateway
add chain=forward connection-state=established
add chain=forward connection-state=related
add action=drop chain=forward connection-state=invalid
```

advertise that route to our internal client ports:

```
/ipv6 address
add eui-64=yes from-pool=comcast interface=ether4    
```


## Finishing up ##

Let’s turn off services we arent using:

```
/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set irc disabled=yes
set h323 disabled=yes
set sip disabled=yes
set pptp disabled=yes
/ip hotspot service-port
set ftp disabled=yes
/ip service
set telnet disabled=yes
set ftp disabled=yes
set api disabled=yes
set winbox disabled=yes
set api-ssl disabled=yes
```

Configure ntp:

```
/system clock
set time-zone-name=America/New_York

/system ntp client
set enabled=yes mode=unicast primary-ntp=x.x.x.x secondary-ntp=y.y.y.y
```

Hostname:

```
/system identity
set name=homerouter
```

ToDo

* Get a cron job going updating a dynamic dns entry. So far I’ve had success using the following code that is specific for no-ip.com that I found on the Mikrotik script repository:

```
##############Script Settings##################

:local NOIPUser "sheiny"
:local NOIPPass "asdasdzxdzsdfsfdsfxf"
:local WANInter "ether1-gateway"
:local NOIPDomain "myhost"

###############################################


:local IpCurrent [/ip address get [find interface=$WANInter] address];
:for i from=( [:len $IpCurrent] - 1) to=0 do={
    :if ( [:pick $IpCurrent $i] = "/") do={
     :local NewIP [:pick $IpCurrent 0 $i];
     :if ([:resolve $NOIPDomain] != $NewIP) do={
       /tool fetch mode=https user=$NOIPUser password=$NOIPPass url="https://dynupdate.no-ip.com/nic/update\3Fhostname=$NOIPDomain&myip=$NewIP" keep-result=no
       :log info "NO-IP Update: $NOIPDomain - $NewIP"
      }
    }
}

    Get a VPN up and running (or maybe just utilize SSH with keys + forwarding)
    Script to email daily digest of network activity and router status changes

```
