---
layout: post
title: "freeradius and google authenticator"
date: 2014-03-06 18:56:20 -0500
comments: false
categories: [freeradius, google, two-factor]
---

Caveat
---
freeradius is a bit baffling to get a full grasp on and I don't pretend to be an expert. My goals were two-fold - radius users authenticate against pam (`rlm_pam`) with two-factor google authenticator and ensure freeradius doesn't have to run as root. 

Logistically, The two-factor code will come in AFTER (no space, enter, etc) the user's password like:

```
    user: <username>
    password: <password><googlecode>
```

Radius Files
---
By default, freeradius config files will be found in `/etc/raddb`. From the start you will be concerned with:


* `radiusd.conf` - lists all the relevant paths of radius files. sets radiusd permissions 
* `sites-enabled/default` - default site configuration parameters specified. Lists enabled modules for aaa 
* `clients.conf` - This file defines radius clients by IP or subnet that are allowed to connect, client specific parameters, and where you will indicate the secret shared key.
* `users` - You can list users in this file if you want and define parameters of groups. This is an optional file technically which is enabled by default thanks to the 'files' parameter listed in the 'default' sites file.
* `/etc/pam.d/radiusd` - pam parameters to run for the radiusd rlm_pam module

In `radiusd.conf`, modified the following under the authenticate section (used to be pap):

```
    Auth-Type PAP {
          pam
    }
```

So, I wanted to allow users on the local system that are in the `switchuser` group and also a user named rancid. Everyone else should fail out. My rancid user is used for scraping configs and doesn't have full command rights. Part of this setup involves configuring the privilege level commands on the IOS device itself. I set some system specific Appended the following to `users`:

```
# 
# Allow Unix users that are in the switchuser group
#
DEFAULT Group == 'switchuser' 
    foundry-privilege-level := 0,
    foundry-command-string := "*",
    cisco-avpair = "shell:priv-lvl=4",

#
# Rancid user is only allowed show run commands
#
rancid
    foundry-command-string := "show running-config ;show conf*; show vlan",
    foundry-privilege-level := 5,
    foundry-command-exception-flag := 0,
    cisco-avpair = "shell:priv-lvl=4",

# On no match, the user is denied access.
DEFAULT Auth-Type := REJECT

```




Google Authenticator Pam "Hack"
---
I use the term "hack" very very loosely. I merely just commented out a line really. Okay, 4 lines. *ahem* Anyways...

If you run radiusd without root permissions, you'll have problems trying to use the google auth pam module. This is because it wants to change uid/gid to the incoming user by default. I did three things to get around this:

1. I set the `secret=/var/run/user/${USER}/.google_authenticator` parameter. Will need all users to copy their files here and chown this folder to the same user that will be running radiusd 
2. I forced the user with the `user=radiusd`
3. I re-compiled the source for the google pam module and commented out a fail value when it tries to change group id. (see below)

```
*** pam_google_authenticator.c  2012-05-14 21:32:27.000000000 -0400
--- /tmp/libpam-google-authenticator-1.0/pam_google_authenticator.c     2014-03-05 13:17:33.777588058 -0500
***************
*** 311,320 ****
        // Inform the caller that we were unsuccessful in resetting the uid.
        *old_uid = uid_o;
      }
!     log_message(LOG_ERR, pamh,
!                 "Failed to change group id for user \"%s\" to %d", username,
!                 (int)gid);
!     return -1;
    }
  
    *old_uid = uid_o;
--- 311,325 ----
        // Inform the caller that we were unsuccessful in resetting the uid.
        *old_uid = uid_o;
      }
!     //      In order to run RADIUS without root permissions, I commented
!     //      out the failure on setgroup and logging code below. This also
!     //      required me to feed the 'user=' option in the pam radius file.
!     //
!     //log_message(LOG_ERR, pamh,
!     //            "Failed to change group id for user \"%s\" to %d", username,
!     //            (int)gid);
!     // return -1;
    }

    *old_uid = uid_o;
```


My `/etc/pam.d/radiusd` now looks like:

```
#%PAM-1.0
auth     required       pam_google_authenticator.so forward_pass secret=/var/run/user/${USER}/.google_authenticator user=radiusd
auth     requisite      pam_nologin.so
auth     include        common-auth
account  include        common-account
password include        common-password
session  include        common-session

```

`forward_pass` produces the functionality explained at the top (google code follows password).

Fire!
---

Start up the server in debug mode with:

```bash
[msheiny:~] $ radiusd -X
```

Start firing packets at your instance with:
```bash
[msheiny:~] $ radtest msheiny {password}{google_code} localhost {port} {shared_key}
```
* `shared_key` is a value found in the clients file