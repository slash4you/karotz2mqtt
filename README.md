# karotz2mqtt
Homeassistant gateway for Karotz device through MQTT protocol.

- Karotz card when switched-on 

![homeassistant_karotz](doc/homeassistant_karotz.png)

- Karotz card when switched-off

![homeassistant_karotz_off](doc/homeassistant_karotz_off.png)

## Pre-requisites

You obviously have to take control of your favorite rabbit to be able to insert the gateway into the processes startup sequence. There are several way to do this, that I won't explain here. In my case I previously installed FreeRabbits OS and customized it a little to fit my needs. Feel free to do this as you want.

## Startup process

Ears and Led states/controls are registered into the broker as soon as the MQTT gateway is started.  Then Karotz device is automatically detected by homeassistant if discovery has been enabled. The gateway shall be setted up by the following valid command line arguments :

```
                        _           _                    
                       / \         / \                   
                       \  \       /  /                   
                        \  \     /  /                    
                         \  \___/  /                     
                         /         \                     
  ______               _|__ O   O  _|    _     _ _       
 |  ____|             |  __ \     | |   | |   (_) |      
 | |__ _ __ ___  ___  | |__) |____| |__ | |__  _| |_ ___ 
 |  __|  __/ _ \/ _ \ |  _  // _  |  _ \|  _ \| | __/ __|
 | |  | | |  __/  __/ | | \ \ (_| | |_) | |_) | | |_\__ \
 |_|  |_|  \___|\___| |_|  \_\____|____/|____/|_|\__|___/
                                                         

-bash-4.1# /usr/karotz/bin/karotz2mqtt -h

karotz2mqtt - Proxy mqtt for KAROTZ system
 usage: /usr/karotz/bin/karotz2mqtt [options]
 where options are:
  -h, --help              display this message and exit.
  -a, --about             output version information and exit.
  -d, --discovery-prefix  prefix for mqtt discovery.
  -m, --mqtt-prefix       prefix for mqtt topics.
  -b, --broker-address    mqtt broker address.
  -p, --broker-port       mqtt broker port.
  -u, --broker-username   mqtt broker username.
  -P, --broker-password   mqtt broker password.
  -l, --location          location setup.
  -v, --verbose           verbosity increase.
  -f,--flash              flashing period in ms
  
-bash-4.1# 
```

More command line options may come later as soon as this tool will be stable enough. Keep in mind that it's a beta release without warranty of being bug-free.

The process has a few dependencies as displayed hereafter :

```
-bash-4.1# ldd /usr/karotz/bin/karotz2mqtt 
	libuuid.so.1 => /usr/lib/libuuid.so.1 (0x4000e000)
	libzmq.so.1 => /usr/lib/libzmq.so.1 (0x4001a000)
	libdbus-1.so.3 => /usr/lib/libdbus-1.so.3 (0x4004f000)
	libmosquittopp.so.1 => /usr/lib/libmosquittopp.so.1 (0x4008b000)
	libpthread.so.0 => /lib/libpthread.so.0 (0x40099000)
	libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x400b3000)
	libm.so.0 => /lib/libm.so.0 (0x4015e000)
	libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x40176000)
	libc.so.0 => /lib/libc.so.0 (0x40189000)
	libmosquitto.so.1 => /usr/lib/libmosquitto.so.1 (0x401e8000)
	librt.so.0 => /lib/librt.so.0 (0x40201000)
	ld-uClibc.so.0 => /lib/ld-uClibc.so.0 (0x40000000)
-bash-4.1# 
```

New ones are libzmq.so.1, libmosquittopp.so.1 and libmosquitto.so.1. I used the former for my internal software design but it may be removed in a future release. I obviously needed the latter to manage MQTT protocol and the second library is only a C++ wrapper to main mosquitto library provided as C library code.

The process shall be started after all dbus daemons ( ears-daemon, led-daemon, voice-daemon,... ) as it connects itself to the system bus and controls the device through those proxies. As I found no way to be notified to daemons startup, I insert a short delay before starting the gateway by this way : 

```
-bash-4.1# cat /usr/scripts/startup.sh 
#!/bin/bash
...
function start_bricks {
    logger -s "[STARTUP] Start Immortaldog bricks"
    /usr/karotz/bin/immortaldog /var/run/karotz/led.pid /usr/karotz/bin/led-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/rfid.pid /usr/karotz/bin/rfid-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/webcam.pid /usr/karotz/bin/webcam-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/button.pid /usr/karotz/bin/button-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/ears.pid /usr/karotz/bin/ears-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/voice.pid /usr/karotz/bin/voice-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/multimedia.pid /usr/karotz/bin/multimedia-daemon >/dev/null 2>/dev/null
    /usr/karotz/bin/immortaldog /var/run/karotz/dbus_watcher.pid /usr/scripts/dbus_watcher >/dev/null 2>/dev/null
}

function start_mqtt_gateway {
    logger -s "[STARTUP] Start mqtt gateway"
   [ -f /usr/karotz/bin/karotz2mqtt ] && /usr/karotz/bin/karotz2mqtt -b @@IP@@ -u @@USERNAME@@ -P @@PASSWORD@@ & >/dev/null 2>/dev/null      
}
logger -s "[STARTUP] Starting Startup script"

led_cyan_pulse
/usr/scripts/waitfornetwork.sh >/dev/null

if [ $? -eq 0 ]; then
    logger "[STARTUP] Karotz connected to internet"
    led_green
    dbus_start
    start_cron
    /bin/killall led >/dev/null
    start_bricks
    sleep 3
    dbus_led_green_pulse
    sync_time
    play_ready
    sleep 3
    start_mqtt_gateway
else
    logger "[STARTUP] Karotz not connected to internet"
    led_red    
fi

logger "[STARTUP] Startup script finished"
...

```

Logs can be found into system logs as they are managed by syslog tool.

```
-bash-4.1# tail -n 100 /var/log/messages
...
May 19 18:55:13 karotz user.crit karotz2mqtt[20436]: karotz2mqtt startup - software release '3.8.0-r1013' built on '2020-05-18 09:20 +0200'
...
```

## MQTT broker connection

During startup the gateway tries to connect to the MQTT broker according the command line settings every 60 seconds. On successful connection the gateway publish discovery data into the following topics : 

- <discovery prefix>/cover/<location>-karotz_<id>/left_ear_position
- <discovery prefix>/cover/<location>-karotz_<id>/right_ear_position
- <discovery prefix>/light/<location>-karotz_<id>/dimmer
- <discovery prefix>/binary_sensor/<location>-karotz_<id>/status

![mqtt_explorer](doc/mqtt_explorer.png)

Then current ears position and led-relative data are published into the following topics :

- <mqtt prefix>/karotz_<id>/status

- <mqtt prefix>/karotz_<id>/ear/left

- <mqtt prefix>/karotz_<id>/ear/right

- <mqtt prefix>/karotz_<id>/led

![mqtt_karotz_data](doc/mqtt_karotz_data.png)

## Homeassistant card

TODO