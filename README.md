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

*Example files have been added to this repo*

### openvpn
You should have already setup `openvpn` so you can successfully fire up a connection from the command line.

```
/path/to/openvpn /path/to/your/config.ovpn
```

### tinyproxy
You should have already setup `tinyproxy` so you can successfully start up an instance. Only refering to `tinyproxy` here, but the setup can easily be used and changed for any proxy you use.

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
*Note: Make sure that the number is unique in your file, the number itself does not matter as the scripts will refer to the tables by the name. The name and location of the file may vary depending on your operating system!*

## `vpnroutes` script
The script will be executed by `openvpn` when the connection gets established and disconnected.
```
#!/bin/bash

# customize here!

# the log id, useful if you want to grep for messages in the syslog
LOGID="vpnroutes-script"

# your default gateway without any vpns, usually your router
DEFGATEWAY="192.168.2.1"

# customize above!

# log that this script has been run                                                                                                         
/usr/bin/logger -i -t $LOGID called with type $script_type for interface $dev on $ifconfig_local

case $script_type in
up)
        # run by openvpn when the interface was created
        echo -e "\n\tConnected through "$common_name" with public ip "$trusted_ip

        # get a unique number for the instance, simply using it from tunX or tapX
        # we don't have to worry about it because openvpn will handle that for us
        # ie takes one that is available                              
        devnum=$(echo $dev | sed -e 's/[^0-9]//g')

        # set a route to the vpn gateway through the default system gateway
        # in case we already have a vpn running hogging the default route
        # vpn hosters won't allow you to run a vpn through a vpn lol
        /usr/sbin/route add $ifconfig_local gw $DEFGATEWAY

        # connect the new network device with a dedicated routing table (must exist! see above!)
        /sbin/ip route add $ifconfig_local/32 dev $dev src $ifconfig_local table vpn$devnum

        # set a route in the global table that the vpn gateway is handled by the vpnX table
        /sbin/ip route add default via $ifconfig_local dev $dev table vpn$devnum

        # direct all traffic from the vpn gateway through the vpnX table 
        /sbin/ip rule add from $ifconfig_local/32 table vpn$devnum
        /sbin/ip rule add to $ifconfig_local/32 table vpn$devnum

        # get a proxy port by adding an offset to the unique device number
        # when adding 8888 then dev0 will be available on port 8888, dev1 will be port 8889 etc
        proxyport=$(($devnum + 8888))
               
        # create a customized config file for tinyproxy using the base template
        cat /path/to/your/tinyproxy.conf.base | sed -e "s/OVPNIP/$ifconfig_local/" -e "s/OVPNDEV/$dev/" -e "s/OVPNPORT/$proxyport/" > /path/to/your/tinyproxy_$dev.conf
        echo -e "\tStarting tinyproxy with conf /path/to/your/tinyproxy_"$dev".conf dev "$dev" on port "$proxyport" public ip "$trusted_ip" listening on port "$proxyport
 
        # start tinyproxy with that config file
        /path/to/tinyproxy -c /path/to/your/tinyproxy_$dev.conf &
        
        # wait a short time until tinyproxy has spawned and setup its pid file
        sleep 3
        
         # get the tinyproxy pid - just for the status output
        TINYPID=$(grep -E "^[0-9]+$" /var/run/tinyproxy/tinyproxy_$dev.pid)
        echo -e "\tTinyproxy for dev "$dev" running with pid "$TINYPID" for openvpn pid "$daemon_pid" listening on port "$proxyport
        /usr/bin/logger -i -t $LOGID Starting tinyproxy for $dev on $ifconfig_local from tinyproxy_$dev.conf pubip $trusted_ip pid $TINYPID ovpnpid $daemon_pid proxyport $proxyport
        
        # create a dat file so we can always see which tinyproxy pid is running for which openvpn pid
        echo -e $(date)" Tinyproxy for dev "$dev" running with pid "$TINYPID" for openvpn pid "$daemon_pid" listening on port "$proxyport" pubip "$trusted_ip > /var/run/tinyproxy/$dev.dat
        
        # for debugging, log all environment variables too
        env > /var/run/tinyproxy/$dev.env
        ;;
down)
        # run by openvpn when then interface gets removed

        # get a unique number for the instance, simply using it from tunX or tapX
        # we don't have to worry about it because openvpn will handle that for us
        devnum=$(echo $dev | sed -e 's/[^0-9]//g')

        # get the tinyproxy pid that has been spawned for this device and kill it
        # this will also kill all child processes that tinypid spawned
        TINYPID=$(grep -E "^[0-9]+$" /var/run/tinyproxy/tinyproxy_$dev.pid)
        echo -e "\tKilling tinyproxy for dev "$dev" with pid "$TINYPID 
        kill $TINYPID

        /usr/bin/logger -i -t $LOGID Stopping tinyproxy for $dev with pid $TINYPID and removing routes for $dev

        echo -e "\tRemoving routes through $ifconfig_local for $dev"

        # remove routes that have been created
        # routes in the vpnX tables get dropped automatically when the device vanishes
        /usr/sbin/route del $ifconfig_local gw $DEFGATEWAY
        /sbin/ip rule del from $ifconfig_local/32 table vpn$devnum
        /sbin/ip rule del to $ifconfig_local/32 table vpn$devnum

        # delete our status files
        test -e /var/run/tinyproxy/$dev.dat && rm /var/run/tinyproxy/$dev.dat
        test -e /var/run/tinyproxy/$dev.env && rm /var/run/tinyproxy/$dev.env

        # optionally remove our custom tinyproxy config, if not deleted, it gets overwritten on the next run
        test -e /etc/tinyproxy/tinyproxy_$dev.conf && rm /etc/tinyproxy/tinyproxy_$dev.conf
        ;;
esac

exit 0
```

## usage
- With this config in place you can simply start openvpn in the foreground like this:
```
/path/to/openvpn /path/to/your/config.ovpn
```
To stop the process, press <kbd>ctrl-c</kbd>. Openvpn will kill the spawned tinyproxy instance.

- or in the background like this:
```
/path/to/openvpn /path/to/your/config.ovpn &
```

Sample output:
```
        Connected through **VPNINFO** with public ip XX.YY.ZZ.WW
        Starting tinyproxy with conf /etc/tinyproxy/tinyproxy_tun2.conf dev tun2 on port 8890 public ip XX.YY.ZZ.WW listening on port 8890
        Tinyproxy for dev tun2 running with pid 1496464 for openvpn pid 1496413 listening on port 8890
```

## status
As the script creates `.dat` files with status messages you can checkout your running instances at any time like this:
```
cat /var/run/tinyproxy/tun*.dat
```

Sample output:
```
Tue Feb 8 00:14:15 CET 2022 Tinyproxy for dev tun1 running with pid 1495945 for openvpn pid 1495885 listening on port 8889 pubip XX.YY.ZZ.WW
Tue Feb 8 00:18:42 CET 2022 Tinyproxy for dev tun2 running with pid 1496464 for openvpn pid 1496413 listening on port 8890 pubip XX.ZZ.WW.YY
```

Stopping an instance can be done by killing the `openvpn` instance:
```
kill 1495885
```

Sample output:
```
        Killing tinyproxy for dev tun1 with pid 1495945
        Removing routes through XX.YY.ZZ.WW for tun1
```

## appendix
The `openvpn` instance can only be started and stopped by the `root` user. A reconnect (to get a new ip address) can be initiated by sending the `SIGUSR1` signal to the `openvpn` process - which is also limited to the `root` user. However it is possible for any user to send any signal to `openvpn` if it gets started with the management console:
```
/path/to/openvpn /path/to/your/config.ovpn --management 127.0.0.1 3333
```

Any user can then connect to the `openvpn` interface:
```
telnet 127.0.0.1 3333
```

Sample output triggering a reconnect:
```
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
>INFO:OpenVPN Management Interface Version 1 -- type 'help' for more info
signal SIGUSR1
SUCCESS: signal SIGUSR1 thrown
        Killing tinyproxy for dev tun1 with pid 1498915
        Removing routes through XX.YY.ZZ.WW for tun1

        Connected through **VPNINFO** with public ip XX.YY.ZZ.WW
        Starting tinyproxy with conf /etc/tinyproxy/tinyproxy_tun1.conf dev tun1 on port 8889 public ip XX.YY.ZZ.WW listening on port 8889
        Tinyproxy for dev tun1 running with pid 1499037 for openvpn pid 1498859 listening on port 8889
```

Sample output shutting down openvpn:
```
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
>INFO:OpenVPN Management Interface Version 1 -- type 'help' for more info
signal SIGTERM
SUCCESS: signal SIGTERM thrown
        Killing tinyproxy for dev tun1 with pid 1499037
        Removing routes through XX.YY.ZZ.WW for tun1
Connection closed by foreign host.
```

*Note: These signals can be sent by simple scripts to automate a reconnect or shutdown of openvpn. The output of the `vpnroutes` script may not be shown when you started `openvpn` on a different console. You can still see the messages in your syslog.*

## Additional status information
Once the `openvpn` instance is running, you can see all available environment variables that can be used in the `vpnroutes` script in the `/var/run/tinyproxy/$dev.env` file. By default this file will be removed when the connection is dropped.
