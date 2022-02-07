## Synopsis
Basic configuration for creating multiple `openvpn` connections on the same machine
without making them the default connection and automatically starting a tinyproxy instance
to use that connection like a proxy server on localhost.

## Disclaimer

**This is not a step-by-step setup that will apply for any operating system!
The included files may or may not work out of the box for your system,
use them as template for your own setup!
I would only recommend this setup with an average knowledge of networking, routing and scripting!**

**Please refer to your vpn provider how many simultaneous connections are allowed.
Some only allow five, others up to ten. Using this setup may violate their TOU!**

## Preparation

### openvpn
You should have already setup `openvpn` so you can successfully fire up a connection from the command line.

```
/path/to/openvpn /path/to/your/config.ovpn
```

### tinyproxy
You should have already setup `tinyproxy` so you can successfully start up an instance. Only refering to `tinyproxy` here, but the setup cat easily be used and changed for any proxy you use.

```
/path/to/tinyproxy -c /path/to/your/tinyproxy.conf
```

## openvpn setup
Usually `openvpn` will create default routes on your system to direct every network access through the vpn interface. This prevents multiple instances as a second connection would be routed through the first and no vpn provider seems to allow that. 

Configuring `openvpn` to not create any routes on connect by adding this option to your `openvpn` configuration file:
```
route-nopull
```

To automatically start `tinyproxy` when `openvpn` establishes a connection we use the `up` and `down` scripts in the `openvpn` configuration file:
```
up /path/to/vpnroutes
down /path/to/vpnroutes
```
*Note: you can change the location and the name of the script to your liking*

To allow the execution of the script `vpnroutes` the `openvpn` script security setting has to be adjusted accordingly:
```
script-security 2
```

## tinyproxy setup
Several options in the `tinyproxy` configuration need to be adjusted based on the number of `openvpn` instances and the dynamic address of the vpn gateway. As these options change every time `openvpn` establishes a connection, we use a base template of your default `tinyproxy` configuration file:
```
cp /path/to/your/tinyproxy.conf /path/to/your/tinyproxy.conf.base
```

Edit your `tinyproxy.conf.base` like this:
- change the `Port` option to look like this (uppercase letters!):
```
Port OVPNPORT
```
- change the `Bind` option to look like this (uppercase letters!):
```
Bind OVPNIP
```
- change the `PidFile` option to look like this (uppercase letters!):
```
PidFile "/var/run/tinyproxy/tinyproxy_OVPNDEV.pid"
```
*Note: You can adjust the location of your pid file, just make sure to include the string OVPNDEV in the path!*

## routing tables
To create and refer to additional routing tables that have to be created for the `openvpn` network interfaces one can add additional entries to the `/etc/iproute2/rt_tables` file. There can be up to 255 routing tables, each one refered to by number or name. For easy access and scripting add a few more to the end of your `/etc/iproute2/rt_tables`.
```
12 vpn0
13 vpn1
14 vpn2
15 vpn3
16 vpn4
```
*Note: Make sure that the number is unique in your file, the number itself does not matter as the scripts will refer to the tables by the name*

## `vpnroutes` script
The script will be executed by `openvpn` when the connection gets established and disconnected.
```
#!/bin/bash                                                                                                                                 
                                                                                                                                            
# customize here!                                                                                                                           
                                                                                                                                            
# the log id, useful if you want to grep for messages in the syslog                                                                         
LOGID="vpnroutes-script"                                                                                                                    
                                                                                                                                            
# your default gateway without any vpns, usually your router                                                                                
DEFGATEWAY="192.168.1.200"                                                                                                                  
                                                                                                                                            
# we have to manually create routing tables for the new openvpn interfaces                                                                  
# those need to be known to the kernel in advance before we can use it                                                                      
# for that edit the file /etc/iproute2/rt_tables                                                                                            
# there add something like this at the end:                                                                                                 
# 12 vpn0                                                                                                                                   
# 13 vpn1                                                                                                                                   
# 14 vpn2                                                                                                                                   
# 15 vpn3                                                                                                                                   
# 16 vpn4                                                                                                                                   
# this will allow up to 5 additional routing tables, make sure the first number is unique in the file                                       
# if you change the names here to something else than vpnX then you also have to change it in the routes below                              
                                                                                                                                            
# log that this script has been run                                                                                                         
/usr/bin/logger -i -t $LOGID called with type $script_type for interface $dev on $ifconfig_local
```
