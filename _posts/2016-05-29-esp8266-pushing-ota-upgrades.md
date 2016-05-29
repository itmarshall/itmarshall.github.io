---
layout: post
title: "ESP8266 - Pushing OTA Upgrades"
date: 2016-05-29`
---

This is the fourth in a series of posts regarding my experiences with the ESP8266 microcontroller. Previous posts on the ESP8266 can be found at the bottom of this page.

Now that you have your ESP8266 microcontroller on the network, you can take advantage of the fact to upgrade it's code without having to connect a serial cable. Updating the firmware of a microcontroller without a direct (wired) connection is called Over The Air (OTA) updates. This is especially helpful for when you have an ESP8266 in place in a box, or perhaps in a hard to reach location.

A common setup is to have an [HTTP server] [esphttpd] running on the ESP8266, with one of the URL's PUSHing the firmware data to the ESP8266. This works, and would be my preferred solution if I were also wanting HTTP server support in my project. A great example of a project that includes a HTTP server, with flashing ability is the JeeLab's [esp-link] [esplink] project. I used this project as an inspiration for my code.

[esphttpd]: https://github.com/Spritetm/libesphttpd
[esplink]: https://github.com/jeelabs/esp-link

## Push vs Pull Updates

There are two ways that the firmware could be upgraded over the air. The first way is to have the ESP poll a remote system at set intervals (e.g. nightly) and retrieve the latest flash code and upgrade to it if it's newer. This method of upgrading the firmware is called "Pull" upgrade, as the ESP8266 is "pulling" the latest code from a server. Pull upgrades are a good method for when you have a device in production, especially if you're integrating this into something to be sold. All you have to do is put code updates in a known location, and the devices will update themselves automatically!

When you're in development, however, or when you have only a small number of ESP8266s in and around your own home, waiting for a scheduled upgrade window is quite limiting. The answer for this is to have the ESP8266 listen for a connection, on which new firmware can be sent at any time. This is called "Push" updates, as your PC is "pushing" the newest firmware to the ESP8266 on demand. As everything I'm doing with ESP8266s at the moment is small-scale learning and development, I'm only currently looking at push style updates here.

## How The ESP8266 Flashes Itself

The ESP8266 has built-in support for flashing itself on the fly. To enable this, the ESP8266 is set up to break its flash memory up into two halves. These two halves contain *two* copies of the firmware. These are typically called "user1.bin" and "user2.bin". At any one time, the microcontroller is running on only one of the two firmware copies, so the other one can be rewritten with a new firmware. Once the new firmware has been written, The ESP8266 is told to use the new firmware on the next restart, and is then restarted.

The below table (from [the ESP8266 wiki] [wikimap]) shows the flash mapping when set up for OTA upgrades:

[wikimap]: http://www.esp8266.com/wiki/doku.php?id=esp8266_memory_map

<table class="maintable">
<tr> <th>Address</th> <th>Size</th> <th>Name</th> <th>Description</th> </tr>
<tr> <td>00000h</td> <td>4k</td> <td>boot.bin</td> <td>Bootloader</td> </tr>
<tr> <td>01000h</td> <td>64k</td> <td>app.v6.flash1.bin</td> <td>User application, slot 1</td> </tr>
<tr> <td>11000h</td> <td>180k</td> <td>app.v6.irom0text1.bin</td> <td>SDK libraries, slot 1</td> </tr>
<tr> <td>3E000h</td> <td>8k</td> <td>master_device_key.bin</td> <td>OTA device key</td> </tr>
<tr> <td>40000h</td> <td>4k</td> <td> </td> <td>Unused</td> </tr>
<tr> <td>41000h</td> <td>64k</td> <td>app.v6.flash1.bin</td> <td>User application, slot 2</td> </tr>
<tr> <td>51000h</td> <td>180k</td> <td>app.v6.irom0text1.bin</td> <td>SDK libraries, slot 2</td> </tr>
<tr> <td>7C000h</td> <td>8k</td> <td>esp_init_data_default.bin</td> <td>Default configuration, see note below</td> </tr>
<tr> <td>7E000h</td> <td>8k</td> <td>blank.bin</td> <td>Filled with FFh. May be WiFi configuration.</td> </tr>
</table>

## How To Flash With Your Own Code

If you are writing your own code to have the flash the microcontroller itself, it's actually quite easy. The steps involved are:

1. Find out whether you need to flash the user1.bin or user2.bin firmware - this is performed via the *system_upgrade_userbin_check* function call. Note that this call returns the *current* flash block in use, not the one that needs to be flashed!
1. Get the sender to send through the contents of the appropriate flash firmware binary
1. Erase the flash blocks to be written to via the *spi_flash_erase_sector* function call. Note that the flash sectors are 4096 bytes long
1. Store the new firmware in the flash memory via the *spi_flash_write* function call, one sector at a time
1. Get the ESP8266 to use the new version of the flash on the next reboot with the *system_upgrade_flag_set* function
1. Reboot the ESP8266 into the new firmware using *system_upgrade_reboot*. This call should not be made in the call-back for receiving TCP data, so a good idea is to run it from a timer in a couple of second's time.

## TCP/IP Flashing

For those times where I have WiFi enabled in your ESP8266 microcontroller, but don't want to have a full HTTP server, I have developed a relatively simple TCP/IP based solution, which I called TCP-OTA. This is similar to the HTTP method, in that the new firmware is sent over WiFi to the ESP8266 for flashing. Even more confusingly, HTTP is a protocol that runs *on top* of TCP/IP. So what exactly is the difference? The TCP-OTA code is:

* Standalone - There is only one C file, and one header file
* Easy to use - Simply call the *ota_init* method after setting up WiFi
* Simple - The protocol's headers are HTTP-like in some ways, but everything is pared back as far as possible

It listens for a connection on TCP port 65056, which is close to the highest numbered port you can have in TCP. I chose it because it is a palindrome - there is no technical reason that it couldn't be any other unused port.

Unfortunately, handling TCP data is *messy*, as you never know how many bytes you'll be receiving in a single request. A single request may come in one packet, or in a dozen of them! This means that I had to write code to buffer the incoming data until we had enough to process, and then ensuring that any extra data was subsequently stored in the buffer once the first batch's processing was complete. As there isn't enough RAM in the ESP8266 to store the entire firmware, we have to erase and rewrite the flash blocks as we get each 4096 bytes of data.

The protocol that I created will use the following exchanges:

<table class="maintable">
<tr> <th>Direction</th> <th>Message</th> </tr>
<tr> <td>to ESP8266</td> <td>"OTA\r\n"</td> </tr>
<tr> <td>to ESP8266</td> <td>"GetNextFlash\r\n"</td> </tr>
<tr> <td><i>from ESP8266</i></td> <td>"user1.bin\r\n" or "user2.bin\r\n", depending on which binary is the next one to be flashed</td> </tr>
<tr> <td>to ESP8266</td> <td>"FirmwareLength: &lt;len&gt;\r\n", where "&lt;len&gt;" is the number of bytes (in ASCII) to be sent in the firmware</td> </tr>
<tr> <td><i>from ESP8266</i></td> <td>"Ready\r\n"</td> </tr>
<tr> <td>to ESP8266</td> <td>&lt;Firmware&gt;, for "&lt;len&gt;" bytes</td> </tr>
<tr> <td><i>from ESP8266</i></td> <td>"Flashing\r\n" or "Invalid\r\n"</td> </tr>
<tr> <td><i>from ESP8266</i></td> <td>"Flash upgrade success. Rebooting in 2s.\r\n"</td> </tr>
</table>

## OTA Flashing From Linux

To run the above sequence from my PC's end, I wrote a very simple Python program called "tcp_flash.py" that will communicate with the ESP8266 and forward the firmware data to it.

``` python
#!/usr/bin/env python
#
# tcp_flash.py - flashes an ESP8266 microcontroller via 'raw' TCP/IP (not HTTP).
#
# Usage:
#   tcp_flash.py <host|IP> <user1.bin> <user2.bin>
#
# Where:
#   <host|IP>    the hostname or IP address of the ESP8266 to be flashed.
#   <user1.bin>  the file holding the first flash format file. Used when the currently used flash is user2.bin
#   <user2.bin>  the file holding the second flash format file. Used when the currently used flash is user1.bin
#
# Author: Ian Marshall
# Date: 27/05/2016
#

import socket
import sys

PORT=65056

# Verify the parameters.
if len(sys.argv) < 3:
	print 'Usage: '
	print '   Usage:'
	print '     tcp_flash.py <host|IP> <user1.bin> <user2.bin>'
	print ''
	print '   Where:'
	print '     <host|IP>    the hostname or IP address of the ESP8266 to be flashed.'
	print '     <user1.bin>  the file holding the first flash format file.'
	print '                  Used when the currently used flash is user2.bin'
	print '     <user2.bin>  the file holding the second flash format file.'
	print '                  Used when the currently used flash is user1.bin'
	sys.exit(1)

# Copy the parameters to more descriptive variables.
host = sys.argv[1]
user1bin = sys.argv[2]
user2bin = sys.argv[3]
print 'Flashing to "{}"'.format(host)

# Open the connection to the ESP8266.
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(5);
s.connect((host, PORT))

# Send a request for the correct user bin to be flashed.
s.send('OTA\r\nGetNextFlash\r\n')

# Wait for the reply.
f = None
response = s.recv(128)
if response == "user1.bin\r\n":
	print 'Flashing \"{}\"...'.format(user1bin)
	f = open(user1bin, "rb")
elif response == "user2.bin\r\n":
	print 'Flashing \"{}\"...'.format(user2bin)
	f = open(user2bin, "rb")
else:
	print 'Unknown binary version requested by ESP8266: "{}"'.format(response)
	sys.exit(2)

# Read the firmware file.
contents = f.read()
f.close()

# Send through the firmware length 
s.send('FirmwareLength: {}\r\n'.format(len(contents)))

# Wait until we get the go-ahead.
response = s.recv(128)
if response != "Ready\r\n":
	print 'Received response: {}'.format(response)
	sys.exit(3)

# Send the firmware.
print 'Sending {} bytes of firmware'.format(len(contents))
s.sendall(contents)
response = s.recv(128)
if len(response) > 0:
	print 'Received response: {}'.format(response)

# Close the connection, as we're now done.
s.close()
sys.exit(0)
```

The above script isn't very pretty, as the error handling can best be described as "crash, and let Python's error printing code give the user a stack trace that they can hopefully use." With not too much work in the future, I hope to be able to improve it significantly.

## Sample Program

I have created a simple demonstration program, which is built once again on the network blinking program from the last blog post. The only change that I've made to the code itself is to call out to the OTA code to initialise it ready for remote flashing. Unfortunately, things got rather complicated rather quickly when I did this, as my previous make files were all generating a single firmware image, instead of the user1.bin/user2.bin combinations.

Not being a make file guru (most of my coding life is spent in Java these days, which uses tools that are *rather more advanced* than the venerable "make" program), I've been forced to use the [esp-link] [esplink] project's make file. This make file is a work of art, which I can't honestly say that I completely understand even now, but I did get it to work. To do this, I had to put my new code into a *src* subdirectory, which does make things a little more complicated, but it does work. And that's really all I'm going for here. (The code I'm writing for the ESP8266 is typically *not* up to my normal professional standards of documentation or build quality!)

The build and flash for this is now performed via the following command:

``` bash
make tcpflash ESP_HOSTNAME=10.0.1.4
```
Replace *10.0.1.4* with whatever IP address your ESP8266 is running on. Note that the first time, you'll have to flash it via serial (using *make flash*), to put the code that will receive new code on the ESP8266!

The full source code for the "ota-tcp" demonstration project is available in GitHub [here] [github].

[github]: https://github.com/itmarshall/esp8266-projects/tree/master/ota-tcp

## Previous ESP8266 blogs

* [First Steps] [firststeps]
* [UART Fun] [uartfun]
* [Networking Basics] [netbasics]

[firststeps]: {{ site.github.url }}/blog/2016/05/07/esp8266-first-steps
[uartfun]: {{ site.github.url }}/blog/2016/05/14/esp8266-uart-fun
[netbasics]: {{ site.github.url }}/blog/2016/05/21/esp8266-networking-basics
