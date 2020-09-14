
# wakeupbr-docker

[![Docker Cloud Build Status](https://img.shields.io/docker/cloud/build/aecampos/wakeupbr.svg?label=build)](https://hub.docker.com/r/aecampos/wakeupbr)

A containerized version of [`wakeupbr`]. Acts as bridge for Wake-on-LAN packets. The program
listens for Wake-on-LAN packets on the incoming interface and forwards any
received packets to the outgoing interface. Useful for allowing other containers to send WOL packets without running them in host networking mode.




## Quickstart
Start `wakeupbr` with:
```
docker  run --net=host --name wakeupbr aecampos/wakeupbr -o <interface address>
```
For example:
```
docker  run --net=host --name wakeupbr aecampos/wakeupbr -o 192.168.1.255
```

Other command-line arguments (`-h`, `-l`) can also be specified.

Note: `--net=host` is usually required for magic packets to make it onto your lan.

### Docker Compose
Example [`docker-compose`](https://github.com/docker/compose) file that runs wakeupbr, listening on `0.0.0.0:9` (default, all interfaces) and forwarding WOL packets to `192.168.1.255`.
```
version: '3.6'
services:
  wakeupbr:
    image: aecampos/wakeupbr
    network_mode: host
    command: -l 0.0.0.0:9 -o 192.168.1.255
    restart: always
```
Alternatively, you can use the sample [`docker-compose.yml`](https://github.com/adriancampos/wakeupbr-docker/blob/master/docker-compose.yml) file to start the container with `docker compose up`.

## `wakeupbr` usage

```
$ wakeupbr -h
Usage:
  wakeupbr [OPTIONS]

Application Options:
  -l, --listen=IP     Listen address to use when listening for WOL packets (default: 0.0.0.0:9)
  -o, --forward=IP    Address of interface where received WOL packets should be forwarded

Help Options:
  -h, --help          Show this help message
```

## `wakeupbr` Details

The `wakeupbr` program acts as bridge for Wake-on-LAN packets. The program
listens for Wake-on-LAN packets on the incoming interface and forwards any
received packets to the outgoing interface.

Example:

A device has two interfaces, one wired (`eth0`) with the address `172.16.0.10`
and one wireless (`wlan0`) with the address `10.0.0.10`. The device we want to
wake is on the wired network. We want to pick up Wake-on-LAN packets that are
received on the wireless network and send them out on the wired network. This
can be accomplished with the following command:

```
$ wakeupbr -l 10.0.0.10 -o 172.16.0.10
```

Any Wake-on-LAN packet that is broadcast on the wireless network will then be
forwarded. When a packet is received and forwarded, a message will be logged:

```
2017/07/28 19:34:54 Forwarded magic packet for AA:BB:CC:12:34:56 to 172.16.0.10
```

The command above listens on UDP port 9 for Wake-on-LAN packets. As port 9 is a
privileged port, `wakeupbr` must be run as root. This is less than ideal, but
Wake-on-LAN packets are always broadcast to port 9. To avoid binding to a
privileged port we can use a `iptables` rule:

```
$ iptables -t nat -A PREROUTING -i wlan0 -p udp --dport 9 -j REDIRECT --to-port 9000
```

You should also ensure that traffic to UDP port 9000 is accepted by the `INPUT`
chain:

```
$ iptables -A INPUT -p udp --dport 9000 -j ACCEPT
```

This will redirect all packets on UDP port 9 to port 9000. `wakeupbr` can then
listen on port 9000 and run as a regular user:

```
$ wakeupbr -l 10.0.0.10:9000 -o 172.16.0.10
```

## Example: HomeAssistant
This bridge is useful, for example, for allowing a [Home Assistant](http://home-assistant.io/) docker container to send WOL packets without using `net=host` on the container.

You can use the [wake on lan integration](https://www.home-assistant.io/integrations/wake_on_lan/) normally:
```
switch:
  - platform: wake_on_lan
    mac: "ab:cd:ef:gh:ij:jk"  
```
and rely on `wakeupbr` to forward the WOL packets to the rest of your lan.
