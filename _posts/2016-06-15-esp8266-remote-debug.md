---
layout: post
title: "ESP8266 - Remote Debug"
date: 2016-06-15`
---

This is the fifth in a series of posts regarding my experiences with the ESP8266 microcontroller. Previous posts on the ESP8266 can be found at the bottom of this page.

The [previous] [otaupgrades] blog post covered how to push upgraded firmware to an ESP8266 over the network, which is wonderful for updating firmware when the ESP8266 is in a hard to reach location, or in place in a project. Unfortunately, while you can *update* the code on the microcontroller remotely, you will still need to connect your serial cable to see what's going on inside it.

Unless you make some changes...

I do most to all of my debug output via the *os_printf* function. It works like the standard C library's *printf* method (hence the name), and provides for nice and easy conversion of variables to the text being debugged. I tend to use only the "%d" (decimal number, i.e. integer), "%x" (hexadecimal number, great with pointers) and "%s" (string) converstion types, as they work, and cover pretty much every occasion I've needed debug output so far. By default, *os_printf* will simply send the output to the output of UART 0, but this can be diverted through a call to the *os_install_putc1* function. The only parameter in this function takes a pointer to a function that is to be called whenever a single character is to be written by *os_printf*. This is described in the documentation as being used to divert the output to UART 1 like this:

``` c
os_install_putc1((void *)uart1_write_char);
```

Luckily, you can provide any function to be invoked, so I wrote my own that sends the debug information over the WiFi network. When planning this feature, I had several ideas as to how this could be done:

1 Sending the packets via UDP to a pre-defined IP address
1 Sending the information via TCP to a pre-defined IP address
1 Sending the information via TCP to whoever has connected to the microcontroller, or else via the default UART 0

While TCP/IP provides a closer analog to a serial line (as it is a stream oriented protocol), I decided to go with the first option, sending via UDP. I did this because, firstly, UDP is much simpler than TCP - there are no connections to worry about, and you don't have to ensure that one packet has finished sending before another can be sent. The third option was an attractive one, as it allows us to choose where the debug output goes, it means that after a reboot for any reason, you won't get any debug information until you reconnect. Fortunately, I have a "server" PC that is always on, with a fixed IP address, so I can direct my debug packets at it.

## Sending the packets via UDP

To send the packets via UDP, I had to create a local buffer. We receive the characters to be sent one at a time, and we don't want to send each character in a separate packet, as the overheads (28 bytes for UDP + IPv4) would be ruinous, as well as the fact that the ESP8266 can have a maximum of 8 UDP packets being transmitted at once. The problem is that we are never told when a call to *os_printf* has finished, just each individual character created, so it's hard to know when to stop buffering, and send the resulting data. The decision that I made was to use a new-line ('\n') character as a marker for sending a packet, as that's how I and many (most?) others end debug lines. I also have a trigger for when the buffer is full, just in case a long line is being printed. From a transmission perspective, it would be best to always send only when the buffer is full (remember those 28 bytes of overhead in every packet), but for debug output, I want to be able to see them as soon as possible, so the newpline character was the best compromise. There is one other minor check - I won't send a packet if it contains nothing but a single new-line character.

I covered sending UDP packets in an [earlier blog post] [netbasics], which is code that I re-used in a modified form here. The code that does all the work is reproduced below:

``` c
// The UDP destination port for the debug packets.  #define DBG_PORT 65432

// The number of bytes to use for the debug message buffer.
#define DBG_BUFFER_LEN 128

// Stores the address to which debug packets are sent in an ip_addr structure.
#define DBG_ADDR(ip) (ip)[0] = 10; (ip)[1] = 0; (ip)[2] = 1; (ip)[3] = 253;

// Structure holding the TCP connection information for the debug communications.
LOCAL struct espconn dbg_conn;

// UDP specific protocol structure for the debug communications.
LOCAL esp_udp dbg_proto;

// Buffer used for storing debug message bytes until a new line (\n) character is received, or it fills up.
LOCAL char dbg_buffer[DBG_BUFFER_LEN];

// The number of bytes currently used in the debug buffer.
LOCAL uint8_t dbg_buffer_len = 0;

/*
 * Receives a single character of output for debugging. This is used to send through the data via UDP.
 */
LOCAL void dbg_putc(char c) {
    // Add the character to the buffer.
    dbg_buffer[dbg_buffer_len++] = c;

    // See if we're ready to send the buffer through - a new-line (with other data) or full buffer will trigger a transmission.
    if (((c == '\n') && (dbg_buffer_len > 1)) || (dbg_buffer_len == DBG_BUFFER_LEN)) {
        // Set the destination IP address and port.
        DBG_ADDR(dbg_proto.remote_ip);
        dbg_proto.remote_port = DBG_PORT;

        // Prepare the UDP "connection" structure.
        dbg_conn.type = ESPCONN_UDP;
        dbg_conn.state = ESPCONN_NONE;
        dbg_conn.proto.udp = &dbg_proto;

        // Send the debug message via a UDP packet.
        espconn_create(&dbg_conn);
        espconn_send(&dbg_conn, dbg_buffer, dbg_buffer_len);
        espconn_delete(&dbg_conn);

        // Reset the buffer.
        dbg_buffer_len = 0;
    }
}
```

## Receiving UDP debug packets

Now you have your ESP8266 transmitting debug packets, how do you actually *receive* them? If you're running on Linux like me, then you're in luck. A software package called *netcat* exists that can send and receive network packets. The following command:

``` bash
netcat -l -u 65432
```

This will then print out the contents of every packet received on port 65432 (which is the one that I randomly decided was a good port to use). The down-side to this command, though, is that it does not show any information as to the time that a message was received, nor which IP address it was sent from. While timestamps are not a big issue (you don't get them on a normal serial connection, either), if you end up using the remote debugging code on multiple ESP8266's, it would be nice to be able to filter out the results based on which microcontroller you're interested at any one time.

To help with this, I've created a quick little Python program that prints out the contents of UDP packets, as well as the IP address they came from (with included filtering by IP address if you want it). The script is very simple:

``` python
#!/usr/bin/env python
#
# udp_debug_rx.py - receives UDP debug packets, and prints them out
#
# Usage:
#   udp_debug_rx.py <IP>
#
# Where:
#   <IP> the IP address from which debug packets are printed, or all addresses, if not supplied
#
# Author: Ian Marshall
# Date: 13/06/2016
#

from __future__ import print_function
from datetime import datetime

import socket
import sys

PORT=65432

# Get the filtering details from the parameter, if any.
match_addr = ''
if len(sys.argv) > 1:
	match_addr = sys.argv[1]

# Prepare the UDP socket.
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
s.bind(('', PORT))

# Variables that are used to decide if we need to "inject" a new-line character in the output.
last_addr = ''
last_nl = False

# Repeat forever, to receive multiple packets.
while True:
	# Wait for a packet to arrive.
	message, (addr, port) = s.recvfrom(1024);

	# See if we're filtering for this address.
	if len(match_addr) != 0 and match_addr != addr:
		continue

	if not last_nl and addr != last_addr:
		# The last message did not end in a new-line, but was from a different IP, so we want to put a new-line in.
		print('', end='\n')
	
	# Get the current time.
	dt = datetime.now().strftime('%Y-%m-%d %H-%M-%S.%f')[:-3]

	# Print the received message.
	if not last_nl and addr == last_addr:
		# This is a message continuation, just print the message contents.
		print(message, end='')
	else:
		# Print the address, date/time and the message.
		print(addr, dt, message, sep=': ', end='')

	# Remember the status of this message, ready for the next one.
	last_addr = addr
	last_nl = message[-1:] == '\n'
```

## Sample Program

To demonstrate the concepts involved with remote debugging, I have built a new demonstration from [last week's] [otaupgrades] project, changing nothing but the addition of the UDP debug packet transmission. All of the new code is held in two files, *udp_debug.h* and *udp_debug.c*. If you wish to appropriate this into your own code, don't forget to change the IP address in *udp_debug.h*!

The full source code for the "udp-debug" demonstration project is available in GitHub [here] [github].

[github]: https://github.com/itmarshall/esp8266-projects/tree/master/udp-debug

## Previous ESP8266 blogs

* [First Steps] [firststeps]
* [UART Fun] [uartfun]
* [Networking Basics] [netbasics]
* [Pushing OTA Upgrades] [otaupgrades]

[firststeps]: {{ site.github.url }}/blog/2016/05/07/esp8266-first-steps
[uartfun]: {{ site.github.url }}/blog/2016/05/14/esp8266-uart-fun
[netbasics]: {{ site.github.url }}/blog/2016/05/21/esp8266-networking-basics
[otaupgrades]: {{ site.github.url }}/blog/2016/05/29/esp8266-pushing-ota-upgrades
