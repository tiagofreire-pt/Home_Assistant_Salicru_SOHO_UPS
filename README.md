This procedure depicts all the steps made on an attempt to fully integrate this cheap UPS on Proxmox VE and Home Assistant Core running on top of it.


# Proxmox VE integration

## Connecting the UPS to the Proxmox hardware

Use the USB port and the provided USB cable to connect the UPS straight into a free USB port, after powering up the UPS on the front power button.

## Detecting the new USB device from the UPS

Run on the Proxmox shell, throught the its web GUI:

```
# lsbusb
```

You should see a similar line within the response:

```
Bus 001 Device 028: ID 06da:ffff Phoenixtec Power Co., Ltd 
```

If not, check the USB connection or use a different USB port on your hardware.


## Installing the NUT server and client

Run:

```
# apt update && apt install nut -y
```

If does not exist, create the following file and its contents:

```
# nano /etc/nut/ups.conf
```

```
maxretry = 3
[salicru]
driver = usbhid-ups
port = auto
desc = "Salicru"
```

Create the following rule:

```
nano /etc/udev/rules.d/90-nut-ups.rules
```

```
# Rule for a Salicru UPS
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="06da", ATTR{idProduct}=="ffff", MODE="0660", GROUP="nut"
```

Restart the udev to apply the above rule that allows the `nut` user to access the UPS driver later on:

```
# service udev restart
```

Next, replug the USB cable into the hardware USB port, waiting a couple of seconds between.


Then, paste the following contents inside:

```
# nano /etc/nut/nut.conf
```

```
MODE=standalone
```

Edit the file and adjust accordingly:

```
# nano /etc/nut/upsd.conf
```

```
# LISTEN <address> [<port>]
LISTEN 127.0.0.1 3493
# OR USE 0.0.0.0 IF NEEDED FOR ACCESSING ACROSS THE LAN
```

Set the monitor username and its password. Use a strong one!

```
# nano /etc/nut/upsd.users
```

```
[upsmonitor]
password = YOUR_STRONG_PASSWORD
upsmon master

```

As the last file to edit:

```
# nano /etc/nut/upsmon.conf
```

```
# Commands for shutdown on power loss
MONITOR salicru@THE_PROXMOX_LAN_IP_OR_LOCALHOST 1 upsmonitor YOUR_STRONG_PASSWORD_HERE master
POWERDOWNFLAG /etc/killpower
SHUTDOWNCMD "/sbin/shutdown -h now"
```


Set permissions correctly:

```
# chown root:nut /etc/nut/*
```
and 
```
# chmod 640 /etc/nut/*
```

Enable both NUT server and client services:

```
# systemctl enable nut-server.service
```
and:
```
# systemctl enable nut-client.service
```

Finally, start both:


```
# service nut-server start
# service nut-client start
```

If you are confronted with errors, check them through:

```
# journalctl -xe
```


# Home Assistant Core integration

Go to your Home Assistant instance and open the following path:

```
Configuration -> Integrations -> + sign (at the buttom-right side) -> search for NUT
```

Fill the correct parameters:

![home_assistant_nut_client_config_options](./img/home_assistant_nut_client_config_options.png)
>


After all the steps, you should find a proper configured integration like this:

![home_assistant_nut_client_configured](./img/home_assistant_nut_client_configured.png)
>


Finally, create a manual `lovelace` card with this proposal:

```
entities:
  - sensor.salicru_battery_chemistry
  - sensor.salicru_battery_runtime
  - sensor.salicru_battery_voltage
  - sensor.salicru_load
  - sensor.salicru_status
  - sensor.salicru_status_data
type: entities
title: Salicru SPS SOHO+ 500 VA
```

Have fun! ;)


# References

* https://diyblindguy.com/howto-configure-ups-on-proxmox/
* https://books.google.pt/books?id=Zls0TZcopvEC&pg=PA138&lpg=PA138&dq=chown+root:nut+/etc/nut/*&source=bl&ots=ARi3BvwKK-&sig=ACfU3U1XzOLtuie6jTxayyrfJC07Cxkq7A&hl=en&sa=X&ved=2ahUKEwi94_WDosHqAhXUDmMBHcr8DC8Q6AEwAHoECAoQAQ#v=onepage&q=chown%20root%3Anut%20%2Fetc%2Fnut%2F*&f=false
