# sensu-trapd
* * *

Sensu Trap Daemon

sensu-trapd is a SNMP trap receiver that translates SNMP traps to Sensu events.
It is designed to listen for SNMP traps and dispatch Sensu events based on a
preconfigured set of mappings (see Configuration).

[![Build Status](https://magnum.travis-ci.com/cloudant/sensu-trapd.png?token=HebgXxm76zdWHjBLhZpe)](https://magnum.travis-ci.com/cloudant/sensu-trapd)

# Requirements
* * *

pysnmp >= 4.2.4

# Installation
* * *

Sensu-trapd can be installed from source using its Makefile:

```
make install
```

Sensu-trapd can also be packaged to deb's (and hopefully RPMs at some point):

```
make deb
```

For more installation information see the Makefile.

# Configuration
* * *

### Converting MIBs for PySNMP

For sensu-trapd to translate a trap, it must have a corresponding MIB defining
that trap loaded. PySNMP (which sensu-trapd uses as a trap receiver), requires
that MIBs be converted into a special format before being loaded into
sensu-trapd. Fortunately, PySNMP provides a utility to do this:

Note: Make you have PySNMP installed, and the "build-pysnmp-mib" is in your path.

```
build-pysnmp-mib -o /some/destination/path/CLOUDANT-CLOUSEAU-MIB.py /some/source/path/CLOUDANT-CLOUSEAU-MIB.txt
```
Once all the relevant MIBs are converted, store them in a directory, say /opt/sensu-trapd/conf/mibs. Specify this path in the sensu-trapd config file under the mibs/paths section. And mention the name of each MIB file (without extension) in the mibs/mibs section (See Example Configuration).

### Configuring Daemon

Sensu-trapd is configured using the conf/config.json file. Additionally, some
configuration can be specified on the command line. See the help for more info. An example for conf/config.json looks like: 

```
{
    "daemon": {
        "log_file":     "/var/log/sensu/sensu-trapd.log",
        "log_level":    "DEBUG",
        "user":         "sensu",
        "group":        "sensu",
        "trap_file":    "/opt/sensu-trapd/conf/traps.json"
    },
    "dispatcher": {
        "host":             "localhost",
        "port":             3030,
        "timeout":          5,
        "events_log":       "/var/log/sensu/sensu-trapd-events.log"
    },
    "mibs": {
        "paths": ["/opt/sensu-trapd/conf/mibs"],
        "mibs": ["CLOUDANT-REG-MIB",
                 "CLOUDANT-PLATFORM-MIB",
                 "CLOUDANT-DBCORE-MIB",
                 "CLOUDANT-LOADBALANCER-MIB"]
    },
    "snmp": {
        "transport": {
            "listen_address":   "127.0.0.1",
            "listen_port":      1161,
            "udp": {
                "enabled": true
            },
            "tcp": {
                "enabled": true
            }
        },
        "auth": {
            "version2": {
                "enabled":      true,
                "community":    "supersecretcommunity"
            },
            "version3": {
                "enabled":      true,
                "users": {
                    "test-user": {
                        "authentication": {
                            "protocol": "MD5",
                            "password": "myAuthSecret"
                        },
                        "privacy": {
                            "protocol": "DES",
                            "password": "myPrivSecret"
                        }
                    }
                }
            }
        }
    }
}
```

- **daemons section**: These are the arguments for the sensu-trapd daemon. When sensu is installed it creates the user:group as sensu:sensu. The same is used here. 
- **dispatcher section**: The sensu server has a socket opened by default to listen to external events. sensu-trapd dispatches the sensu events to this socket. Refer [1] and [2]. The events are sent in json format and logged in the file specified by events_log. You can see some additional options in src/sensu/snmp/config.py
- **mibs section**: Here the path to the converted MIB files and their names are provided.
- **snmp section**: sensu-trapd creates an SNMP engine using pySNMP. The attributes of this engine are provided here. The SNMP trap must be send with the same attributes.Change listen_address to 0.0.0.0 if you want it to listen to traps from other agents. 

### Configuring Traps

Traps are configured using the conf/traps.json (unless another file is specified
in conf/config.json). Basic configuration looks like: 

```
"unique-name-for-sensu-trapd-to-identify-it": {
    "trap": {
        "type": ["MIB-name", "name-of-the-notification"],
        "args": {
            "first": ["MIB-name", "first-argument-the-notification-sends"]
            "second": ["MIB-name", "second-argument-the-notification-sends-if-any"]
        }
    },
    "event": {
        "name": "{hostname}-a-check-name",
        "output": "{first}-this-is-the-message-sensu-gets",
        "handlers": ["name-of-the-sensu-handler", "another-handler"],
        "severity": "CRITICAL"
    }
}
```
- **traps section**: Here you have to tell sensu-trapd which is the notification (trap) that it should look out for. For example, if the SNMP trap is like : 
    ```
    snmptrap -v 2c -c public host "" \
              NET-SNMP-EXAMPLES-MIB::netSnmpExampleHeartbeatNotification \ 
              netSnmpExampleHeartbeatRate i 1234
    ```
    
    Then, MIB-name = NET-SNMP-EXAMPLES-MIB,  
    name-of-the-notification = netSnmpExampleHeartbeatNotification,  
    first-argument-the-notification-sends = netSnmpExampleHeartbeatRate

- **events section**: Here you specify a check name.This will be used by Sensu as the name of the check, so make it meaningful. Additionally, you can specify the output of the check, handlers, and severity of the event. 

Note: If a trap as optional arguments, you must specify a trap handler for
each trap both with and without arguments.

### Example for Basic conf/traps.json Configuration
```
"cloudant-generic-trap-handler": {
    "trap": {
        "type": ["CLOUDANT-PLATFORM-MIB", "cloudantGenericTrap"],
        "args": {
            "message": ["CLOUDANT-PLATFORM-MIB","cloudantTrapMessage"]
        }
    },
    "event": {
        "name": "{hostname} Cloudant Generic Event",
        "output": "{message}",
        "handlers": ["debug"],
        "severity": "CRITICAL"
    }
}
```

### Example Advanced conf/traps.json Configuration

In this example, I've mapped two traps to the same event. This allows
events to recover/resolve when an UP trap is received after a corresponding DOWN
trap is received.

```
"cloudant-loadbalancer-server-up": {
    "trap": {
        "type": ["CLOUDANT-LOADBALANCER-MIB", "cloudantLoadBalancerServerUp"],
        "args": {
            "message": ["CLOUDANT-PLATFORM-MIB","cloudantTrapMessage"]
        }
    },
    "event": {
        "name": "Load Balancer Server Down",
        "output": "{hostname} reports {message}",
        "handlers": ["default"],
        "severity": "OK"
    }
},
"cloudant-loadbalancer-server-down": {
    "trap": {
        "type": ["CLOUDANT-LOADBALANCER-MIB", "cloudantLoadBalancerServerDown"],
        "args": {
            "message": ["CLOUDANT-PLATFORM-MIB","cloudantTrapMessage"]
        }
    },
    "event": {
        "name": "Load Balancer Server Down",
        "output": "{hostname} reports {message}",
        "handlers": ["default"],
        "severity": "WARNING"
    }
}
```

### References 

[1]https://sensuapp.org/docs/latest/clients#socket-attributes  
[2]https://sensuapp.org/docs/latest/clients#client-socket-input

