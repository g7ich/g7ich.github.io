---
title: Remote ZigBee network with unused MikroTik router
categories: [Smart Home, HomeLab]
tags: [ZigBee, MikroTik, WireGuard]

---

## Intro

I was wondering what to do with spare Zigbee dongle and MikroTik router, of course I could just flash dongle with router firmware and stick it into power plug somewhere in house, but it's not fun at all and also no sense in doing such thing once you have good network coverage.
As you may know or not most Zigbee dongles with coordinator firmware utilizes serial interface to communicate with the host device. If you had chance to configure Zigbee2MQTT solution you've seen something like this: `port: /dev/ttyUSB0` or other tty port, but as well Zigbee2MQTT supports IP based solutions.

## Idea

After thinking about that there is spare MikroTik device that could be used in one of remote locations where from time to time I need to switch power to PC for short sessions I could go with one of two options. Microcontroller that would act as power switch to it, or as relay. Unfortunately my spare router is Hex S with no WiFi option built in.
Other option is to somehow hook-up Zigbee network remotely.

## Initial Set-up time

To follow below instructions it's required to have Mikrotik router and working IP connection to that device and of course Zigbee dongle in my case Sonoff ZBDongle-E, and Proxmox VM server where we will spawn Zigbee2MQTT container, which requires MQTT server.

### Configure router

Step one of course as always is to login to the router device, then if you have MikroTik router we need to check if USB device is seen in the system. 

```terminal
> /system/resource/usb/print        
# Columns: DEVICE, VENDOR, NAME, SPEED
# # DEVICE  VENDOR                NAME                                  SPEED
# 0 1-0     Linux 5.6.3 xhci-hcd  xHCI Host Controller                    480
# 1 2-0     Linux 5.6.3 xhci-hcd  xHCI Host Controller                   5000
# 2 1-1     ITEAD                 SONOFF Zigbee 3.0 USB Dongle Plus V2     12
```

So we can see our Zigbee device connected to the system, great!
Now let's see if it's discoverd as serial device.
```terminal
/port/print 
# Columns: DEVICE, NAME, CHANNELS, BAUD-RATE
# # DEVICE  NAME  CHANNELS  BAUD-RATE
# 0 1-1     usb1         1     115200
```

It's possible you will see different baudrate, than this presented above double check with the configuration for given module what's the right value and change it, with:

```terminal
/port/set usb1 baud-rate=<BAUDRATE>
```

After that double check if now correct value is presented.
As the last step use, let's configure port forwarding to TCP port:

```terminal
/port/remote-access/add allowed-addresses=0.0.0.0/0 tcp-port=<port> channel=0 port=usb1 protocol=raw
```
And we can verify if it works with simple nc command from host with installed nc tool:

```terminal
nc <router_ip> <port> -v
# Connection to <router_ip> port <port> [tcp/italk] succeeded!
```

If connection is instantly blocked double check if your configuration is correct and if you have firewall rule that allows such connection. ( I will focus on securing such solution in the next post ).

Once we got succeed connection it's time to go back to Mikrotik shell and run following command:

```terminal
/port/remote-access/print detail 
# Flags: X - disabled, I - inactive, A - active, B - busy; L - logging-active 
#  0 A  port=usb1 channel=0 allowed-addresses=0.0.0.0/0 tcp-port=12345 
#       protocol=raw log-file="" remote-address=<remote_client_ip:port>
```

Good so we have working router configuration. 

**NOTE: This isn't production ready configuration, use it only for testing purposes, you shall configure strict firewall rules !**

### Install&configure Zigbee2MQTT

Now second step in terms of initial setup, Zigbee2MQTT service. 
First we need to install it, to do so in this case I will use the script created by @tteck. 
We will open Proxmox VM shell and then paste the following command.

```terminal
bash -c "$(wget -qO - https://github.com/tteck/Proxmox/raw/main/ct/alpine-zigbee2mqtt.sh)"
```
As result we will see following screen:

```
Proxmox VE Helper Scripts







                       ┌───────────────┤ Alpine-Zigbee2MQTT LXC ├───────────────┐
                       │                                                        │ 
                       │ This will create a New Alpine-Zigbee2MQTT LXC.         │ 
                       │ Proceed?                                               │ 
                       │                                                        │ 
                       │                                                        │ 
                       │                                                        │ 
                       │              <Yes>                 <No>                │ 
                       │                                                        │ 
                       └────────────────────────────────────────────────────────┘ 
                                                                                  
```

As we want to install service we select "Yes", then we will be asked if we want to use defualt setting, I recommend using "Advanced" to have better control over installed container, but all settings can be configured later. 

Final configuration for me:

```
/__  /  (_)___ _/ /_  ___  ___ |__ \ /  |/  / __ \/_  __/_  __/
  / /  / / __  / __ \/ _ \/ _ \__/ // /|_/ / / / / / /   / /   
 / /__/ / /_/ / /_/ /  __/  __/ __// /  / / /_/ / / /   / /    
/____/_/\__, /_.___/\___/\___/____/_/  /_/\___\_\/_/   /_/     
       /____/ Alpine
 
Using Advanced Settings
Using Container Type: 1
Using Root Password: ********
Container ID: 104
Using Hostname: alpine-zigbee2mqtt
⚠ DISK SIZE MUST BE AN INTEGER NUMBER!
Using Container Type: 1
Using Root Password: ********
Container ID: 104
Using Hostname: alpine-zigbee2mqtt
Using Disk Size: 1
Allocated Cores: 2
Allocated RAM: 1024
Using Bridge: vmbr0
Using IP Address: dhcp
Using Gateway IP Address: Default
Disable IPv6: yes
Using Interface MTU Size: Default
Using DNS Search Domain: Host
Using DNS Server IP Address: Host
Using Vlan: 100
Enable Root SSH Access: no
Enable Verbose Mode: no
Creating a Alpine-Zigbee2MQTT LXC using the above advanced settings
```

Once container is created we can access it with: `lxc-attach <container_id>`
And then edit the configuration file, in my case basic configuration like that:

```/etc/zigbee2mqtt/configuration.yaml
permit_join: true # For testing purposes
mqtt:
  base_topic: zigbee2mqtt_remote
  server: mqtt://<mqtt_server>
  user: <user>
  password: <password>
serial:
  port: tcp://<router_ip>:<port>
  baudrate: 115200
  adapter: ezsp
frontend: # Enable frontend
  port: 8080
  host: 0.0.0.0
homeassistant: true # Enable ha topics
advanced:
  log_level: info
  log_output:
    - console
    - syslog
  log_directory: /var/log/zigbee2mqtt
  log_syslog:
    facility: daemon
    path: /dev/log
    protocol: unix
  network_key: '!secret.yaml network_key'
  pan_id: '!secret.yaml pan_id'
  homeassistant_legacy_entity_attributes: false
  legacy_api: false
  legacy_availability_payload: false
devices: devices.yaml
groups: groups.yaml
device_options:
  legacy: false
```

## It's time to verify if everything working

Restart container and open http://<container_ip>:8080 to check the dashboard.
As permit join is enabled try to pair new Zigbee device to the network, everything shall work correct.

![Zigbee Home](assets/img/posts/2024-07-18/zigbee_home.png)

From Dashboard interface we have now access to sensors data and control over devices.

![Zigbee Dashboard](assets/img/posts/2024-07-18/zigbee_dashboard.png)

## Debrief

Now we have functional remote Zigbee coordinator, we of course could just use one off-the-shelf, but it's always fun to use spare tech and what's more important learn new stuff. 
For the next steps I will try to make it a little bit more secure than it is in current form, but for now it allows to control and collect data from the remote Zigbee sensors, goal achieved!
