# Kasa Smart Plug

I have a few [TP-Link Kasa Smart Plugs](https://www.kasasmart.com/us/products/smart-plugs)
that I want to setup and control without an account.

## Connect to WiFi

Kasa exposes an open WiFi hotspot for initial setup (indicated by alternating
blinking blue/orange LEDs, initiated by holding the power button for 5s), and I
used [python-kasa](https://github.com/python-kasa/python-kasa) to tell it which
address to join. Note that `keytype` is `3`, based on
[this kasa reverse engineering research](https://miccah.io/assets/kasa.pdf) (page 67).

* `0` = Unencrypted
* `1` = WEP
* `2` = WPA
* `3` = WPA2

First, setup `python-kasa`:

```bash
> python3 -m venv /tmp/kasa
> source /tmp/kasa/bin/activate
(kasa) > python3 -m pip install python-kasa
```

Then, connect to the smart-plug hotspot and run `kasa wifi join`:

```bash
(kasa) > kasa --host 192.168.0.1 wifi join "$SSID"
Discovering device 192.168.0.1 for 10 seconds
Keytype: 3
Password:
Asking the device to connect to "$SSID"..
Response: {} - if the device is not able to join the network, it will revert back to its previous state.
```

I confirmed it joined my network by checking my MikroTik router DHCP leases:

```bash
> ssh admin@10.1 '/ip/dhcp-server/lease/ print'
Flags: D - DYNAMIC
Columns: ADDRESS, MAC-ADDRESS, HOST-NAME, SERVER, STATUS, LAST-SEEN
 #   ADDRESS     MAC-ADDRESS        HOST-NAME    SERVER   STATUS   LAST-SEEN
17 D 10.0.0.74   E4:C3:2A:C4:90:F3  HS105        defconf  bound    33s
```


## Send raw JSON requests

Run `python -m asyncio`:

```python
from kasa.deviceconfig import DeviceConfig
from kasa.transports.xortransport import XorTransport
from kasa.protocols.iotprotocol import IotProtocol
from pprint import pprint

config = DeviceConfig(host="10.74")
protocol = IotProtocol(transport=XorTransport(config=config))

# Below are some example queries for controlling the smart plug and setting a
# device schedule. The query parameter can be a dictionary or raw string.

# Get system info.
pprint(await protocol.query({"system": {"get_sysinfo": {}}}))
# Turn on the smart plug.
pprint(await protocol.query('{"system":{"set_relay_state":{"state":1}}}'))
# Turn off the smart plug.
pprint(await protocol.query('{"system":{"set_relay_state":{"state":0}}}'))
# Schedule a rule to turn on/off the device at 6pm / 11pm respectively.
pprint(await protocol.query({"schedule":{"add_rule":{
  "name": "6pm to 11pm",
  "smin": 18 * 60,  # 18:00 (6:00 pm)
  "emin": 23 * 60,  # 23:00 (11:00 pm)
  "wday": [ 1, 1, 1, 1, 1, 1, 1 ],  # every day
  "enable": 1,
  "repeat": 1,
  "stime_opt": 0,
  "etime_opt": 0,
  "sact": 1,        # turn on
  "eact": 0,        # turn off
  "force": 1
}}}))
# Print all scheduling rules.
pprint(await protocol.query('{"schedule":{"get_rules":{}}}'))
# Delete all scheduling rules.
pprint(await protocol.query('{"schedule":{"delete_all_rules":{}}}'))
# Globally enable using schedules.
pprint(await protocol.query('{"schedule":{"set_overall_enable":{"enable":1}}}'))


await protocol.close()
```


## Add a firewall to MikroTik

I generally don't want my IoT devices to be able to talk to the internet.

```routeros
# Show the current DHCP leases (to get the MAC address).
/ip/dhcp-server/lease print

# Drop all packets originating from the plug's MAC address going to the WAN
# (internet).
/ip/firewall/filter add chain=forward src-mac-address=A8:6E:84:FB:4D:B9 out-interface-list=WAN \
    action=drop comment="drop all smart plug packets to the internet"

# Show the firewall filters.
/ip/firewall/filter print
```

Alternatively, we can use an address list, which requires giving the plug a
static IP address.

```routeros
# Show the current DHCP leases (to get the hostname).
/ip/dhcp-server/lease print

# Make the plug have a static IP.
/ip/dhcp-server/lease make-static [find host-name=HS105]

# Add the address assigned to the hostname to the no-internet list.
/ip/firewall/address-list add list=no-internet \
    address=[/ip/dhcp-server/lease get [find host-name=HS105] address]

# Drop all packets originating from any addresses in the no-internet list to
# the WAN (internet).
/ip/firewall/filter add chain=forward src-address-list=no-internet out-interface-list=WAN \
    action=drop comment="drop all packets to the internet"
```

### Synchronizing Time

Because of the firewall, the smart plug cannot synchronize time via the
internet, so sometimes we have to manually set it. `index` is the timezone
index (see [this reference](https://github.com/whitslack/kasa/blob/master/API.md#request-63) for
the full enumerated values).

* `5` = Pacific Standard Time
* `6` = Pacific Daylight Time

```python
import time

# Set time.
t = time.localtime()
pprint(await protocol.query({"time":{"set_timezone":{
    "index": 6 if t.tm_isdst else 5,
    "year": t.tm_year,
    "month": t.tm_mon,
    "mday": t.tm_mday,
    "hour": t.tm_hour,
    "min": t.tm_min,
    "sec": t.tm_sec,
}}}))

# Print time.
pprint(await protocol.query('{"time":{"get_time":{}}}'))
```
