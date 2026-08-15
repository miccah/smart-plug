# Kasa Smart Plug

I have a few [TP-Link Kasa Smart Plugs](https://www.kasasmart.com/us/products/smart-plugs)
that I want to setup and control without an account.

## Connect to WiFi

Kasa exposes an open WiFi hotspot for initial setup (indicated by alternating
blinking red/orange LEDs, initiated by holding the power button for 5s), and I
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

# Below are some example queries for controlling the smart plug and setting a #
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
# Delete all scheduling rules.
pprint(await protocol.query('{"schedule":{"delete_all_rules":{}}}'))
# Globally enable using schedules.
pprint(await protocol.query('{"schedule":{"set_overall_enable":{"enable":1}}}'))


await protocol.close()
```
