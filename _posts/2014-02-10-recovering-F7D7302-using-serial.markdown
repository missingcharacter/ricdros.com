---
layout: post
title:  "Recovering F7D7302 using serial"
date:   2014-02-09 00:00:00 +0000
tags:
  - Raspberry Pi
  - Serial
  - F7D7302
  - Belkin
---
Last year, I was trying to help a friend change his router's firmware (Belkin
Share N300 Wireless N+ Router MiMo 3D & USB Port) to DD-WRT, he tried first and
something didn't work like it should and asked me for help, because I've been
succesful to make DD-WRT work in other 2 routers (WRT54GL & F7D7301).

After a couple hours I suggested my friend we used
[tomato firmware](http://tomatousb.org/) instead of DD-WRT becuase tomato has a
friendlier interface.

### Yeah, right

He accepted and I flashed tomato, I swear I selected to "Reset to default
settings"

![dd-wrt Firmware upgrade](/assets/images/recovering/01___upgrade_firmware.png)

The problem was that the router stopped working, I have just made a beautiful
plastic brick :(

Tried several times the
[30/30/30 reset](http://www.dd-wrt.com/wiki/index.php/Hard_reset_or_30/30/30)
and it didn't work. We finally gave up on the router and decided to work on it
later.

### Gettin' Jiggy wit It

Well, until the past weekend I had the oportunity to look further into this
issue, thanks to
[Scott Gibson](https://www.blogger.com/profile/06759040624540828619) I was able
to confirm the router's serial port:

![F7D7302 Serial port](http://3.bp.blogspot.com/-nIDbLdqE8lc/Tj0lqUDacbI/AAAAAAAACFg/Y3Vg35mrcnU/s320/image+%25281%2529.jpeg)

Pin 1: Vcc (3.3V)
Pin 2: RX
Pin 3: TX
Pin 4: Gnd

Thanks to [Gtoniser](http://tweakers.net/gallery/247680) and their
[post](http://appventures.tweakblogs.net/blog/8736/unbricking-your-router-with-a-raspberry-pi.html)
I had the information on how to connect the Raspberry Pi and the router's serial.

### Setup the Pi

First, You have to comment the following line in `/etc/inittab`

```shell
T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100
```

it should look like this:

```shell
T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100
```

Reboot the RPi and now we can use the serial port.

### How to connect the RPi

Thanks to
[lavalink](http://lavalink.com/2012/03/raspberry-pi-serial-interfacing/) for the
image

![RPi GND TX RX](http://lavalink.com/wp-content/uploads/2012/04/raspberry-pi-serial_sm-241x300.jpg).

Now we connect these ports to the router (Router should be off)
RPi - Router
GND <-> GND
TX  <-> RX
RX  <-> TX

![RPi](/assets/images/recovering/2014_02_09_00_53_27.jpg)

![F7D7302 serial](/assets/images/recovering/2014_02_09_00_53_33.jpg)

### Listen to the serial

Install minicom in the RPi

```shell
sudo apt-get install minicom
```

Then run minicom on the serial port of the RPi

```shell
sudo minicom -b 115200 -o -D /dev/ttyAMA0
```

Now power on the router while holding `Ctrl` + `C` and now the terminal should
capture whatever is going in the router, sorry for not providing better looking
images.

![minicom output](/assets/images/recovering/2014_02_09_00_54_18.jpg)

In this case, the router was stuck verifying something and since it was wrong it
rebooted every time it got to that point, the good thing is that now I knew the
bootloader was OK and I didn't need to use jtag.
I pressed the `space` bar to stop the verification and finally got to the CFE
prompt.
Cleared nvram and reboot

```shell
CFE> nvram erase
CFE> reboot
```

### Success

I finally saw the device assigning and IP address to port, I connected a
computer to Port 1 and openned 192.168.1.1

![web console is back](/assets/images/recovering/2014_02_09_00_54_02.jpg)

TOMATO was up and running, for some reason the hard reset was not doing its
thing.

Now F7D7302 is working again :)

P.S. This is how the mess looked like

![dirty setup](/assets/images/recovering/2014_02_09_00_54_30.jpg)

If you think I might be able to help you out contact me on
[Twitter](http://twitter.com/ricdros)

Sources:

- [Unbricking your router with a Raspberry Pi](http://appventures.tweakblogs.net/blog/8736/unbricking-your-router-with-a-raspberry-pi.html)
- [Belkin F7D3302 Hacking](http://thegreatgeekery.blogspot.com/2011/08/belkin-f7d3302-hacking.html)
- [Serial Recovery](http://dd-wrt.com/wiki/index.php/Serial_Recovery)
