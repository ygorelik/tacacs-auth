# NSO TACACS Authentication
This project focuses on enabling TACACS based external authentication for Cisco NSO.  

## Background 
External Authentication for NSO is accomplished by creating some executable script that NSO can call when user tries
to login. NSO simply passes user provided credentials to the script via standard input in the format:
>[user;password;]\n

and expects an output such as
>accept ncsadmin ncsoper 1004 1004 /tmp

In this example the user will be mapped into the NSO groups _ncsadmin_ and _ncsoper_.
What is done in the external authentication script is irrelevant to NSO, all that matters is the output.
For example, this script would work just fine, but will result in any provided credentials passing authentication and
being authorized for both admin and operator roles.
```python
#! /usr/bin/env python3 

accept = "accept ncsadmin ncsoper 1004 1004 /tmp"
print(accept)
```

> For more details on how external authentication works, see the documentation at
`$NCS_DIR/doc/html/nso_admin_guide/ug.aaa.External_authentication.html` from your NSO installation.
Also, a presentation on access control with NSO is included in this repo:
[NSO Access Control Role based and Resource based Access](resources/NSODevDays2020-NSO-Access-Control-Role-based-and-Resource-based-Access.pdf)

### Guides
The following detailed guides have been written on how to use this service.
* [Preparing TACACS Server - Cisco ISE](resources/README-setup-tacacs-ise.md).
How to setup Cisco ISE to receive and responde to NSO authentication requests from this service.
* [How the External Authentication Works](resources/README-deepdive-tacacs-auth.md).
A technical deep dive into this service works.
* [How to Deploy and Use the tacacs-auth Service](resources/README-tacacs-auth-installation.md).
A walk through on how to install the tacacs-auth service to NSO.
* [Troubleshooting External Authentication with Logs](resources/README-troubleshooting-logs.md).
A review of how to troubleshoot the external authentication with logs.

## Basic External Authentication with TACACS

### Enabling External Authentication
To enable external authentication the NSO base configuration defined in _ncs.conf_ must include an _<external-authentication>_
definition under the _<aaa>_ configuration and provide path to the authentication executable. It also should provide
order of authentication methods in case multiple methods are enabled, because the default order is different:

```xml
<?xml version="1.0"?>
<ncs-config xmlns="http://tail-f.com/yang/tailf-ncs-config">
  <aaa>
    <external-authentication>
      <enabled>true</enabled>
      <executable>./scripts/python/tacacs_auth.py</executable>
    </external-authentication>
    <local-authentication>
      <enabled>true</enabled>
    </local-authentication>
    <auth-order>external-authentication local-authentication pam</auth-order>
  </aaa>
</ncs-config>  
```

> Note. In the example `local-authentication` is also enabled.
This could be disabled once the external-authentication is configured, tested, and trusted.
Or you can leave `local-authentication` enabled as a secondary access method.

### Configuring the TACACS Hosts in NSO
Before NSO can perform external authentication to the TACACS server, it needs to know its IP address and communication port
as well as the secret or key, which is used for data encryption. Rather than backing these data into the script, or requiring
that they are set on the underlying Linux host running NSO, these values are stored in the NSO CDB. The data model
defined in the YANG module _tacacs_config.yang_, which has this tree view:
```
$ pyang -f tree tacacs-config.yang 
module: tacacs-config
  +--rw tacacs* [host]
     +--rw host      inet:ipv4-address
     +--rw port?     uint16
     +--rw secret?   tailf:aes-256-cfb-128-encrypted-string
```
where TACACS server's credentials are stored in a list _tacacs_ with the element key _host_. The list element contains:
* _host_ - IP address of the server
* _port_ - communication port
* _secret_ - data encryption key

Note that more than one TACACS server can be configured. The external authentication script will attempt
to authenticate with the hosts in the order they are stored in the CDB. If a successful authentication
is achieved on a host, that host will be used for authorizing the user. If an attempt to authenticate
to a host fails for any reason, the next host in the list will be tried.
If all hosts fail authentication, authorization is skipped, and the overall external authentication will fail.

### Logging into NSO with External Authentication
With the `tacacs-auth` configured, and `ncs.conf` configured to enable external authentication,
we can now try to log into NSO with the external credentials. First, need try login with a user,
who has both `ncsadmin` and `ncsoper` rights. 

```
ssh jdoe@nso
jdoe@nso's password: 

# Try basic "show" command
jdoe@ncs# show packages package oper-status 
packages package tacacs-auth
 oper-status up

# Let's see what configuration options exist
jdoe@ncs# config terminal 
Entering configuration mode terminal
jdoe@ncs(config)# ?
Possible completions:
  aaa                           AAA management
  alarms                        Alarm management
  alias                         Create command alias.
  cluster                       Cluster configuration
  compliance                    Compliance reporting
  customers                     Customers using services
  devices                       The managed devices and device
                                communication settings
  high-availability             Configuration, status and actions
                                concerning NSO Built-in HA
  java-vm                       Control of the NCS Java VM
  nacm                          Access control
  ncs-state                     NCS status information
  packages                      Installed packages
```

So `jdoe` looks to have full access for read and config actions.  

What about a user with only `ncsoper` (read-only) rights.  

```
ssh bsmith@nso
bsmith@nso's password: 

# Try a basic "show" command
bsmith@ncs# show packages package oper-status 
packages package tacacs-auth
 oper-status up

# Let's see what configuration options exist
bsmith@ncs# config terminal 
Entering configuration mode terminal
bsmith@ncs(config)# ?
Possible completions:
  user              User specific command aliases and default CLI
                    session parameters
  webui             Web UI specific configuration
  ---               
  abort             Abort configuration session
  activate          Activate a statement
  annotate          Add a comment to a statement
  clear             Remove all configuration changes
  commit            Commit current set of changes
```

User `bsmith` only has `user` and `webui` listed.  These show up in the list due to how their models are setup,
but attempts to actually configure something under them result in `access-denied`.

```
bsmith@ncs(config)# user test
Error: access denied
```