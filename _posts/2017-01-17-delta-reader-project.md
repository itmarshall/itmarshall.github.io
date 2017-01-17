---
layout: post
title: "Delta Reader - ESP8266 project"
date: 2017-01-17
---

After the success of the [DoT project] [dot] project, which used an ESP8266 to notify me when someone rings my doorbell, wherever I am, I have finally finished work on my second ESP8266 project. This project is a simple device that reads data from a solar power inverter and sends them to a local web server for logging.

[dot]: {{ site.github.url }}/blog/2016/07/02/dot-first-project

## Going Solar

Like many people, I have taken the first steps to not being an environmental vandal by installing some solar panels on my roof. The solar panels are connected to a Delta Solivia 3.3TR inverter, which converts the DC input from the solar panels to 240V AC to power my house or feed power to the grid if we're generating more than we're using.

The inverter has a 16x2 display on it, on which it displays the current AC power that is being generated (i.e. after the losses inside the inverter). This is very interesting information to know, and I thought it would be interesting to *log* it, too, so that I could see how it changes over time. At this point, the Delta Reader project was born.

## RS-485

Fortunately, the Delta inverter itself has two RS-485 ports available for communications. RS-485 is a serial interface that is used in many devices and industrial applications, and allows for multiple (> 2) devices to communicate over a single pair of wires. In this application, I am only using my device to talk to the inverter. 

Both of the physical RS-485 connections are via RJ-45 connectors - the same that is used in standard Ethernet cables. Only pins 2 and 3 are used for the connections. RS-485's wire pairs use a differential signal method, instead of a ground and logic level signal like TTL. This means that only the *difference* in voltage potential between the two wires is used when determining if a value is 0 or 1. Differential signals are excellent for rejecting induced noise, and allow greater communication speeds and distances with a higher reliability.

While RS-485 allows for high speed communications (up to 35 Mbit/s, according to [Wikipedia] [wikipedia]), the communications for the inverter is fixed at 19,200 bps. While that speed may be slow by modern communications standards, even the longest packet response for this application at 13 bytes would only take 5.42ms.

[wikipedia]: https://en.wikipedia.org/wiki/RS-485

## Getting an ESP8266 to Talk RS-485

As I wrote in my post [Quiet, UART!] [quietuart], the ESP8266 has one and a half UARTs, which can send and perhaps receive serial data. These UARTs only send a logical signal (0V = 0, 3.3V = 1), rather than conforming to either RS-232 (used in pre-USB PC serial communications) or RS-485. Just like how a chip like the FTDI FT232RL is required to allow the ESP8266 to talk to a PC via USB, a [MAX485] [max485] chip is used to talk over RS-485. I used a pre-built breakout that included the MAX485, as well as required resistors/capactors that I bought off AliExpress - similar to [these] [max485adapter].

[quietuart]: {{ site.github.url }}/blog/2016/11/13/esp8266-quiet-uart
[max485]: https://www.maximintegrated.com/en/products/interface/transceivers/MAX485.html
[max485adapter]: https://www.aliexpress.com/item/MAX485-Module-RS-485-TTL-to-RS485-MAX485CSA-Converter-Module-For-Arduino-Integrated-Circuits-Products/32667981058.html?spm=2114.01010208.3.20.MoAm7x&ws_ab_test=searchweb0_0,searchweb201602_2_10065_10068_10000009_10084_10000025_10083_10000029_10080_10082_10081_10000028_10110_10111_10060_10112_10113_10062_10114_10056_10055_10054_10059_10032_10099_10078_10079_10000022_10077_10000012_10103_10073_10102_10000015_10096_10000018_10000019_10052_10053_10107_10050_10106_10051,searchweb201603_9,afswitch_2,single_sort_1_default&btsid=450bafe3-525e-41ae-a379-b0f4fdc0e6e1

The MAX485 chip is a *half-duplex* communications chip, which means that it can be either sending or receiving data, but not both at the same time. Another pin from the ESP8266 is used to instruct the MAX485 as to the direction of communications. If this is set wrong, data will be lost.

## Communications Format

What RS-485 does not specify is what data should be transmitted over the serial link. I managed to find (and subsequently lose) some documentation on the Internet as to what the communications packet structure looks like, and what commands are accepted by the inverter. All communications with the inverter are based on a simple request/response pattern, with whatever device (ESP8266, PC, or something else) sending a *request* packet to the inverter, with the inverter then sending a *response* packet back in return. The inverter will never send an unsolicited message.

The packets that are sent utilise the following structure:

<table class="maintable">
  <tr>
    <th>Byte(s)</th>
    <th>Symbol</th>
    <th>Hexadecimal</th>
    <th>Description</th>
  </tr>
  <tr>
    <td class="center">1</td>
    <td class="center">STX</td>
    <td class="center">0x02</td>
    <td class="center">Start of TeXt</td>
  </tr>
  <tr>
    <td class="center">2</td>
    <td class="center">ADDR</td>
    <td class="center">0x05 / 0x06</td>
    <td class="center">Address of recipient (0x05 = inverter, 0x06 = reader)</td>
  </tr>
  <tr>
    <td class="center">3</td>
    <td class="center">ID</td>
    <td class="center">0x01</td>
    <td class="center">Chain ID of the inverter</td>
  </tr>
  <tr>
    <td class="center">4</td>
    <td class="center">LEN</td>
    <td class="center">N/A</td>
    <td class="center">Length of the command (including any response data)</td>
  </tr>
  <tr>
    <td class="center">5..6</td>
    <td class="center">Command</td>
    <td class="center">N/A</td>
    <td class="center">The command that is being requested or responded to</td>
  </tr>
  <tr>
    <td class="center">7..(4 + LEN)</td>
    <td class="center">Data</td>
    <td class="center">N/A</td>
    <td class="center">The data for the command response (if any)</td>
  </tr>
  <tr>
    <td class="center">(5 + LEN)..(6 + LEN)</td>
    <td class="center">CRC</td>
    <td class="center">N/A</td>
    <td class="center">CRC-16 of packet contents</td>
  </tr>
  <tr>
    <td class="center">(7 + LEN)</td>
    <td class="center">ETX</td>
    <td class="center">0x03</td>
    <td class="center">End of TeXt - the last byte in the packet</td>
  </tr>
</table>

The key data for each request/response is the *command*. This is a series of two bytes, representing a single piece of information for the inverter. The response for numeric values will always be of the same length for a given command, making parsing the response very simple.

Note that while there is an ETX character to mark the end of the packet, there is no byte stuffing to ensure that no other ETX character appears without an escape prefix, so simply searching for the first 0x03 byte will not necessarily find the end of the packet.

## Chosen Commands

With this information, I was able to plan what to get from the inverter. In the end I chose the following data points:

<table class="maintable">
  <tr>
    <th>Byte 1</th>
    <th>Byte 2</th>
    <th>Tag Name</th>
    <th>Data</th>
  </tr>
  <tr><td>0x10</td> <td>0x01</td> <td>instant-current-i1</td>   <td>Instantaneous current - Input 1</td></tr>
  <tr><td>0x10</td> <td>0x02</td> <td>instant-voltage-i1</td>   <td>Instantaneous voltage - Input 1</td></tr>
  <tr><td>0x10</td> <td>0x03</td> <td>instant-power-i1</td>     <td>Instantaneous power - Input 1</td></tr>
  <tr><td>0x11</td> <td>0x01</td> <td>average-current-i1</td>   <td>Average current - Input 1</td></tr>
  <tr><td>0x11</td> <td>0x02</td> <td>average-voltage-i1</td>   <td>Average voltage - Input 1</td></tr>
  <tr><td>0x11</td> <td>0x03</td> <td>average-power-i1</td>     <td>Average power - Input 1</td></tr>
  <tr><td>0x20</td> <td>0x05</td> <td>internal-temp-ac</td>     <td>Internal temperature - AC assembly</td></tr>
  <tr><td>0x21</td> <td>0x08</td> <td>internal-temp-dc</td>     <td>Internal temperature - DC assembly</td></tr>
  <tr><td>0x10</td> <td>0x07</td> <td>instant-current-ac</td>   <td>Instantaneous current - AC output</td></tr>
  <tr><td>0x10</td> <td>0x08</td> <td>instant-voltage-ac</td>   <td>Instantaneous voltage - AC output</td></tr>
  <tr><td>0x10</td> <td>0x09</td> <td>instant-power-ac</td>     <td>Instantaneous power - AC output</td></tr>
  <tr><td>0x10</td> <td>0x0A</td> <td>instant-frequency-ac</td> <td>Instantaneous frequency - AC output</td></tr>
  <tr><td>0x11</td> <td>0x07</td> <td>average-current-ac</td>   <td>Average current - AC output</td></tr>
  <tr><td>0x11</td> <td>0x08</td> <td>average-voltage-ac</td>   <td>Average voltage - AC output</td></tr>
  <tr><td>0x11</td> <td>0x09</td> <td>average-power-ac</td>     <td>Average power - AC output</td></tr>
  <tr><td>0x11</td> <td>0x0A</td> <td>average-frequency-ac</td> <td>Average frequency - AC output</td></tr>
  <tr><td>0x13</td> <td>0x03</td> <td>day-energy</td>           <td>Day energy</td></tr>
  <tr><td>0x13</td> <td>0x04</td> <td>day-run-time</td>         <td>Day running time</td></tr>
  <tr><td>0x14</td> <td>0x03</td> <td>week-energy</td>          <td>Week energy</td></tr>
  <tr><td>0x14</td> <td>0x04</td> <td>week-run-time</td>        <td>Week running time</td></tr>
  <tr><td>0x15</td> <td>0x03</td> <td>month-energy</td>         <td>Month energy</td></tr>
  <tr><td>0x15</td> <td>0x04</td> <td>month-run-time</td>       <td>Month running time</td></tr>
  <tr><td>0x16</td> <td>0x03</td> <td>year-energy</td>          <td>Year energy</td></tr>
  <tr><td>0x16</td> <td>0x04</td> <td>year-run-time</td>        <td>Year running time</td></tr>
  <tr><td>0x17</td> <td>0x03</td> <td>total-energy</td>         <td>Total energy</td></tr>
  <tr><td>0x17</td> <td>0x04</td> <td>total-run-time</td>       <td>Total running time</td></tr>
  <tr><td>0x12</td> <td>0x01</td> <td>solar-current-limit</td>  <td>Solar current limit - Input 1</td></tr>
  <tr><td>0x12</td> <td>0x02</td> <td>solar-voltage-limit</td>  <td>Solar voltage limit - Input 1</td></tr>
  <tr><td>0x12</td> <td>0x03</td> <td>solar-power-limit</td>    <td>Solar power limit - Input 1</td></tr>
  <tr><td>0x12</td> <td>0x07</td> <td>current-max-ac</td>       <td>AC current max</td></tr>
  <tr><td>0x12</td> <td>0x08</td> <td>voltage-min-ac</td>       <td>AC voltage min</td></tr>
  <tr><td>0x12</td> <td>0x09</td> <td>voltage-max-ac</td>       <td>AC voltage max</td></tr>
  <tr><td>0x12</td> <td>0x0A</td> <td>power-ac</td>             <td>AC power</td></tr>
</table>

The descriptions are the best I have been able to come up with, as the original documentation that I could find was an Excel spreadsheet in German, which is not a language I speak - so liberal use of Google Translate and educated guessing was used to come up with what I hope are relatively accurate values.

## Tag Names?

Those who looked closely at the above table will have noticed a column called "Tag Name". This relates to how I am processing the data on the server side. The values for each command are stored in a real-time tag database, where only the latest value for that tag are kept. Other code on the server then reads these tag values from the database and stores them into an SQL database for future analysis. These two databases are shared among other programs and projects communicating with the server, this work of mine dates back almost ten years now.

The tags themselves must be unique across the entire tag database to avoid overwriting values from either within the inverter's fields, or between the inverter and other logged values.

## Sending the Data to the Server

The ESP8266's program reads each of the above values from the inverter, one at a time, then sends a single HTTP POST request to the server, with the tag names and their values encoded in a JSON structure. JSON is a nice format for this, as it is easily parsed on the web server, and relatively light weight (compare it to XML), so it's also easy to create on the ESP8266.

Unfortunately, JSON is a text format, which means that a large amount of data needs to be pieced together for transmission. While it is possible to stream this data out via TCP/IP as it is being assembled (e.g. one tag at a time), there is enough RAM in the ESP8266 to be able to piece the entire structure together and send it in one go.

The problem with text is that while we know how many numbers we want to send, and how many bytes they hold in a binary representation, the number of bytes they will require when converted to text is unknown. For an example, the number 0x00 is one character of text ("0"), while 0x0A is two ("10") and 0xFF is three ("255"). So a single 8-bit number can take anywhere from 1-3 characters. The problem gets worse for 16-bit (1-5) and 32-bit (1-10) numbers.

To make matters worse, we need to put a line in the HTTP header (i.e. *before* we start sending the values) containing the number of bytes in the message body. To cope with the varying lengths, we have to either create the whole structure in memory (what I'm doing) or make two passes over the values, converting them to decimal and throwing it away the first time, just to count how many bytes will be required, and then converting them a second time when actually sending the values.

Another language that I program in (not in ESP8266s) is Java, and it has a class in it's base library called a StringBuilder. This class holds an internal character array which is resized as needed whenever it overflows. I created a similar set-up in C, without the fancy OO encapsulation, which allows me to build the HTTP request string and only send it when I'm ready, as well as to query the created string as to it's length when constructing the HTTP header.

The actual sending of the HTTP packet is very easy. Set up a TCP/IP connection in the normal way for an ESP8266, and then make a single call to *espconn_send*.

## Serial Problems

While working through the code, I had a lot of problems reading the serial responses from the inverter. After a lot of investigations, I worked out that the ESP8266 was lying to me. My serial transmissions were made using the *uart_tx_one_char* function, which I thought was synchronous, as there is also a *uart_tx_one_char_no_wait* function. Unfortunately, it is not synchronous, so the direction pin in the MAX485 was being set to receive while the ESP8266 was still transmitting, which truncated the messages. Adding a 5ms delay after the transmission request before clearing the output fixed the issue. Later testing with an oscilloscope (that I bought after already solving this particular problem) showed that this gave me a 600 microsecond buffer between the transmission of the last character and the setting of the direction pin.

While trying to work out why things weren't working correctly, I tried a new approach with the serial connections. I decided to *not* use a call-back for when serial data arrives. Instead, I set up a 5ms timer, and check at each expiry if we have received enough data yet. This timer was also then used for a time-out - if we try too many times, we simply assume that the inverter is not going to reply at all. This occurs every night - once there is insufficient power coming from the solar panels to drive the inverter, it switches itself off.

## Program Flow

The basic program flow is as follows:

1. Initialise the ESP8266, and start a 1 minute timer
1. At the expiry of the timer send the first command request to the inverter
1. Poll every 5ms for a reply from the inverter. Once we have response, validate the packet, store the value and transmit the next command request.
1. Once all command requests have been correctly replied to, form the HTTP request.
1. Connect to the server.
1. Once the connection is established, send the HTTP POST request.
1. Parse any HTTP reply for success/failure.

If you look closely at the program, you'll see that there isn't a single, overarching loop that is run through for each group of transmissions. This is due to the call-back nature of ESP8266 programming (even when using a timer instead of a serial call-back), and because it could take too long to do it all in one big go. The recommendation is that no task should take longer than 2ms, and the watchdog will reset the ESP8266 if control isn't returned to the underlying system in 500ms.

## The Delta Reader Circuit

The Delta Reader circuit is another simple one, electrically speaking. There is a 5V power supply (the MAX485 requires 5V) from an old USB phone charger whose connector got smashed, a 5V->3.3V converter for the ESP8266 and the MAX485 and ESP8266 devices themselves. Throw in a few resistors to keep the ESP8266 happy, and three wires between the ESP8266 and the MAX485 and that's it!

After much debate around the internet, it was finally confirmed that the GPIO inputs on the ESP8266 are 5V tolerant, so I didn't bother to add any level shifting for the MAX485->ESP8266 connection.

![Schematic]({{ site.url }}/images/20170117-Delta-Reader-Schematic.png)

*Note* that like the schematic for DoT, I use a different step-down voltage regulator than in the above schematic. It's just what was available in the default KiCad component libraries, and I couldn't find a compatible one for the Pololu D24V5F3 unit I used.

## The Program

The program for the Delta Reader has been placed in my normal place in GitHub, alongside all of the demonstration projects. It builds on the make files, [remote debugging] [remotedebug] and [OTA upgrade] [otaupgrades] code that I have introduced in earlier blog posts. The full source code is available [here] [github].

[otaupgrades]: {{ site.github.url }}/blog/2016/05/29/esp8266-pushing-ota-upgrades
[remotedebug]: {{ site.github.url }}/blog/2016/06/15/esp8266-remote-debug
[github]: https://github.com/itmarshall/esp8266-projects/tree/master/delta_reader
