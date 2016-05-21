---
layout: post
title: "ESP8266 - Networking Basics"
date: 2016-05-21`
---

This is the third in a series of posts regarding my experiences with the ESP8266 microcontroller. Previous posts on the ESP8266 can be found at the bottom of this page.

The key feature of the ESP8266, which makes it so desirable, is that it is a cheap and capable microcontroller with built-in WiFi networking. Holding off on talking about networking doesn't really seem like a good idea, so here we go with the basics of networking on the ESP8266!

## WiFi Connection

Before you can send or receive data over the network, you must first connect to the network. Typically, you will simply be connecting to your pre-existing home wireless network, though you may want to think about setting up a separate network for your ESP8266 and other "Internet of Things (IoT)" devices, to minimise the impact of any security problems that may arise (especially with 3rd party hardware). Here, I am assuming that a network has already been set up, and that it will be using DHCP to assign IP addresses.

There are two key things that you will need to know before you can connect to a wireless network: the *name* of the network (called an SSID), and the password for the network. These days, most, if not all networks are using "WPA2" encryption, which is supported on the ESP8266, and that is what I'm targetting here.

To connect to your WiFi network, there are some steps that need to be taken, which should be either in your "user_main" function, or in a function called by it. The following code will set up a wireless network:

``` c
// Set station mode - we will talk to a WiFi router.
wifi_set_opmode_current(STATION_MODE);

// Set up the network name and password.
struct station_config sc;
strncpy(sc.ssid, SSID, 32);
strncpy(sc.password, PASSWD, 64);
wifi_station_set_config(&sc);

```

Line 2 sets the ESP8266 into *station mode*, which is where the ESP8266 acts like any other client device on a wireless network. The alternative is to run in *soft AP mode*, where the ESP8266 acts like the router of the wireless network. This is good for simple setup tasks, but it is too limited to be used as a "real" WiFi network in general use.

A *station_config* structure is needed, which will hold the configuration of the network to be set up. For most usages, you will only need to fill in the SSID and password fields. This structure is passed in to the *wifi_station_set_config* function. At this point, the ESP8266 can *start* to establish a connection with your WiFi network.

To tell when things are changing in the status of the wireless network, you can register a call-back function. When you are using station mode, your call-back function will be passed event will contain one of the following values:

* EVENT_STAMODE_CONNECTED - The ESP8266 has connected to the WiFi network as a station
* EVENT_STAMODE_DISCONNECTED - The ESP8266 has been disconnected from the WiFi network
* EVENT_STAMODE_AUTHMODE_CHANGE - The ESP8266 has a change in the authorisation mode
* EVENT_STAMODE_GOT_IP - The ESP8266 has an assigned IP address
* EVENT_STAMODE_DHCP_TIMEOUT - There was a timeout while trying to get an IP address via DHCP

The event passed in to your call-back contains an *event_info* field, which is a union, with different structures within it depending on the event type. This information can be useful to your program, if you choose to use it. An example of a call-back function, which shows the event_info information available is:

``` c
LOCAL void ICACHE_FLASH_ATTR wifi_event_cb(System_Event_t *event) {
    struct ip_info info;

    // To determine what actually happened, we need to look at the event.
    switch (event->event) {
        case EVENT_STAMODE_CONNECTED: {
            // We are connected as a station, but we don't have an IP address yet.
            char ssid[33];
            uint8_t len = event->event_info.connected.ssid_len;
            if (len > 32) {
                len = 32;
            }
            strncpy(ssid, event->event_info.connected.ssid, len + 1);
            os_printf("Received EVENT_STAMODE_CONNECTED. "
                      "SSID = %s, BSSID = "MACSTR", channel = %d.\n",
                      ssid, MAC2STR(event->event_info.connected.bssid), event->event_info.connected.channel);
            break;
        }
        case EVENT_STAMODE_DISCONNECTED: {
            // We have been disconnected as a station.
            char ssid[33];
            uint8_t len = event->event_info.connected.ssid_len;
            if (len > 32) {
                len = 32;
            }
            strncpy(ssid, event->event_info.connected.ssid, len + 1);
            os_printf("Received EVENT_STAMODE_DISCONNECTED. "
                      "SSID = %s, BSSID = "MACSTR", channel = %d.\n",
                      ssid, MAC2STR(event->event_info.disconnected.bssid), event->event_info.disconnected.reason);
            last_addr = 0;
            break;
        }
        case EVENT_STAMODE_GOT_IP:
            // We have an IP address, ready to run. Return the IP address, too.
            os_printf("Received EVENT_STAMODE_GOT_IP. IP = "IPSTR", mask = "IPSTR", gateway = "IPSTR"\n", 
                      IP2STR(&event->event_info.got_ip.ip.addr), 
                      IP2STR(&event->event_info.got_ip.mask.addr),
                      IP2STR(&event->event_info.got_ip.gw));
            break;
        case EVENT_STAMODE_DHCP_TIMEOUT:
            // We couldn't get an IP address via DHCP, so we'll have to try re-connecting.
            os_printf("Received EVENT_STAMODE_DHCP_TIMEOUT.\n");
            wifi_station_disconnect();
            wifi_station_connect();
            break;
    }
}
```

## Talking over the network

Once you have your WiFi network connection set up, you can start talking and listening to other systems over it, which I find to be a rather good use of a computer network! There are two major types of communications that can be performed by the ESP8266, TCP and UDP, which are explained in the following sections:

### TCP - Transmission Control Protocol

TCP is probably the most widely used communications protocol in the world today. When you're browsing the web, you're using TCP. Email? TCP.

What TCP provides is an error free, bi-directional, asynchronous connection stream between two systems. What do these terms mean? "Error free" refers to the fact that TCP will perform basic checksumming on the data, and ensure that any data that doesn't get sent is retransmitted, or that which arrives out of order is correctly ordered. "Bi-directional" means that the systems at each end of the connection can send data to the other system. "Asynchronous" means that each end can send whenever they are able, without waiting for a transmission from the other system.

An important thing to note about a TCP connection is that it is *stream oriented*, so while all of the data by one system will arrive at the other one in the right order, there are no guarantees as to the grouping of the data. You may send 100 bytes at one end, and they might be received with all 100 in one go, 100 separate receptions of 1 byte each, or anywwhere in between. You cannot assume that the grouping of data in messages you send will be the same for the receiver.

To establish a connection, one system (called the *server*) must listen for a connection on a specified *port* number, while the other system must initiate a connection to the server. An example of this is when browsing the web - your PC is the client, which establishes a connection to the server, typically on another computer somewhere on the internet. Once the connection is established, there is no difference between the client and the server, each can send data to the other equally.

To set up the ESP8266 as a server, you set up some basic information, and call the *espconn_accept* function:

``` c
// Structure holding the TCP connection information.
LOCAL struct espconn tcp_conn;

// TCP specific protocol structure.
LOCAL esp_tcp tcp_proto;

...

// Set up the TCP server.
tcp_proto.local_port = 2345;
tcp_conn.type = ESPCONN_TCP;
tcp_conn.state = ESPCONN_NONE;
tcp_conn.proto.tcp = &tcp_proto;
espconn_regist_connectcb(&tcp_conn, tcp_connect_cb);
espconn_accept(&tcp_conn);
```

Note on line 14 that another call-back has been registered. When an incoming TCP connection is received by the ESP8266, the call-back function will be invoked. An example connection call-back function is shown below:

``` c
LOCAL void ICACHE_FLASH_ATTR tcp_connect_cb(void *arg) {
    struct espconn *conn = (struct espconn *)arg;
    os_printf("TCP connection received from "IPSTR":%d\n",
              IP2STR(conn->proto.tcp->remote_ip), conn->proto.tcp->remote_port);
    espconn_regist_recvcb(conn, recv_cb);
}
```

Now we have an established connection, we need to be able to receive data from it, or send data to it. To receive data, we set up another call-back function, which will be invoked whenever data is received. You can see this being set up via *espconn_regist_recvcb* in the above code sample. A sample data call-back function is: 

``` c
LOCAL void ICACHE_FLASH_ATTR recv_cb(void *arg, char *data, uint16_t len) {
    // Store the IP address from the sender of this data.
    struct espconn *conn = (struct espconn *)arg;
    os_printf("Received %d bytes from "IPSTR" \n", len, IP2STR(conn->proto.tcp->remote_ip));

    // Send out the received data via serial in hexadecimal, 16 characters to a line.
    for (uint16_t ii = 0; ii < len; ii++) {
        os_printf("%02x ", data[ii]);
        if ((ii % 16) == 15) {
            // We have reached the end of a line
            os_printf("\n");
        }
    }
}
```

To send data over an established TCP connection, the *espconn_send* function is used, passing in the *espconn* structure for the connection, the byte array to be sent, and the number of bytes in that array to be sent. A simple example is below:

``` c
uint8_t message[] = {0x00, 0x01, 0x02, 0x03};
int8_t res = espconn_send(con, message, 4);
```

### UDP - User Datagram Protocol

UDP is the other main communications protocol used on the internet, but it differs from TCP in that it does not provide for error-correction, nor does it involve a connection. Instead, packets are simply sent from one host to another, and each packet has to fend for itself. Packets could be lost or arrive out of order, and *nothing will be done to fix it!* While this sounds bad, it's actually good for communications where it's OK to lose data some times. A common example is streaming a video and/or voice conversation - if a packet is lost, it'll glitch for a fraction of a second, but continue. With TCP, the loss of a packet will stop everything until it can be resent and acknowledged, a much more jarring experience.

As there is no connection, all you need to do is decide if you want to send or receive UDP packets. To receive, set the ESP8266 to accept incoming packets and register a call-back for when they arrive. A short example for accepting incoming packets is below:

``` c
udp_proto.local_port = 2345;
udp_conn.type = ESPCONN_UDP;
udp_conn.state = ESPCONN_NONE;
udp_conn.proto.udp = &udp_proto;
espconn_create(&udp_conn);
espconn_regist_recvcb(&udp_conn, recv_cb);
```
Note that the port can safely be the same or different to a TCP port, as they don't interact with each other. With UDP, you can register the receiver call-back immediately, as we don't have to wait for a connection.

One problem with UDP reception on the ESP8266 I've noticed is the remote IP address in the call-back function is always set to zero, rather than the IP address of the sender. This seems to be a bug to me, so hopefully it will be fixed in the near future. So for now, you can receive data via UDP, but you can't find out *who* sent it to you, unless you get the sender to embed their IP address in the message body - rather inelegent, and totally impractical if you're implementing someone else's protocol.

To send data via UDP, you need to set the destination IP address and port, then call *espconn_send*, an example is below:

``` c
LOCAL void ICACHE_FLASH_ATTR udp_tx_data(uint8_t *data, uint16_t len, uint32_t ip_addr) {
    // Set the destination IP address and port.
    os_memcpy(udp_proto_tx.remote_ip, &ip_addr, 4);
    udp_proto_tx.remote_port = 1234;

    // Prepare the UDP "connection" structure.
    udp_tx.type = ESPCONN_UDP;
    udp_tx.state = ESPCONN_NONE;
    udp_tx.proto.udp = &udp_proto_tx;

    // Send the UDP packet.
    espconn_create(&udp_tx);
    espconn_send(&udp_tx, data, len);
    espconn_delete(&udp_tx);
}
```

Note the *espconn_create* and *espconn_delete* calls around the send, this is because you must use a different *espconn* structure for sending UDP packets than for receiving them, though they can use the same port numbers.

## Sample Program

I have created a simple demonstration program, which is built once again on the basic blinking program. This time, I've modified it to receive the blinking interval via TCP or UDP, and to send a UDP packet with the new blink state to the last IP address that sent data to us. Unfortunately, as we can't get the IP address of senders of UDP packets, the call-back will only send to systems that send us data via TCP (a TCP connection alone is not enough, there must actually be data sent).

The full source code for the "net-blink" demonstration project is available in GitHub [here] [github].

[github]: https://github.com/itmarshall/esp8266-projects/tree/master/net-blink

## Previous ESP8266 blogs

* [First Steps] [firststeps]
* [UART Fun] [uartfun]

[firststeps]: {{ site.github.url }}/blog/2016/05/07/esp8266-first-steps
[uartfun]: {{ site.github.url }}/blog/2016/05/14/esp8266-uart-fun
