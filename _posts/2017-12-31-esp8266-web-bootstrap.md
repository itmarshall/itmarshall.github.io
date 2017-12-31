---
layout: post
title: "ESP8266 - Web Bootstrap"
date: 2017-12-31
---

After a hiatus for most of the year, I am back for a last post of 2017.

## The problem

One thing that has always bothered me is the way that I have been putting my network ESSID and password information into the source code for my ESP8266 projects. This is the method you see used on most demonstration projects, and is indeed the easiest way to get started in ESP8266 programming. If you're sharing your projects, however, you have to make sure that you don't accidentally share your sensitive network details with the whole world. I committed a code sample to a public Git repository once that contained my network password. Fortunately, I managed to spot it before syncing the repository to GitHub, and deleted the revision before the world learnt of my password.

Another problem with this is that I once pushed a minor code update via an [over-the-air (OTA)] [otaupgrades] upgrade to an ESP8266 installed in a hard to reach location, whereupon it promptly stopped working. This was due to my having scrubbed the ESSID and password in my code for Git, and forgot to reinstate it for my own use. I had to then get the ESP8266 out from its hiding spot and reprogram it via serial before re-installing it.

[otaupgrades]: {{ site.github.url }}/blog/2016/05/29/esp8266-pushing-ota-upgrades

## The solution

There is a solution to this problem, which comes from the realisation that the ESP8266 stores its network configuration in the flash storage on the microcontroller. Upon startup, the ESP8266 will automatically read this information and connect to the configured WiFi network.

My plan, therefore, was to create a one-off bootstrap program that configured the network, and then allowed itself to be re-programmed either by serial or an OTA update. I worked out a way that the ESSID and password parameters could be queried by a make file, and then set to #define variables for the code to use at compile time. This was small and simple, so of course I couldn't use it!

My final solution was to embed a web (HTTP) server into the ESP8266, and have it serve a  HTML page to allow the user to choose a wireless network to connect to, and to supply the password for. This information would then be saved in the flash memory. From this point, the user can then upload the "real" firmware, and it will be able to connect to the network without needing any knowledge of the ESSID or password.

## libesphttpd

To serve the web pages, I turned to a project called [lbesphttpd] [libesphttpd]. This is a small and simple web server that is easy to integrate into your own projects. It's written by a user called "Sprite_tm", real name of Jeroen Domburg. He has some inspired hacks under his belt - check them out on [Hackaday] [spritetmhackaday].

[libesphttpd]: https://github.com/Spritetm/libesphttpd
[spritetmhackaday]: https://hackaday.com/tag/sprite_tm/

There are several ways in which libesphttpd can serve information to HTTP clients:

* Files (e.g. HTML, CSS, PNG) can be stored in the ESP8266's flash memory and served by name upon a client request.
* Templates can be stored in the flash memory. These files are standard files e.g. HTML), but with variables in percentage marks (e.g. "%total%"). A call-back function is then called for each variable, and the results are then inserted into the template file and served to the client.
* A call-back function is invoked, which can directly write the HTML response. This is called CGI, after the common gateway interface used in web servers. This isn' strictly "CGI", as it doesn't invoke a separate program, but hey, it's a microcontroller!
* Redirects, where an address can be set to redirect to another URL.

The web server needs to have a configuration structure passed into it that specifies the addresses that it can serve, often with a fall-back to looking for pages to serve from the flash RAM. An example from the web bootstrap system is:

``` c
HttpdBuiltInUrl builtInUrls[]={
    {"/", cgiRedirect, "/net/networks.html"},
    {"/net", cgiRedirect, "/net/networks.html"},
    {"/net/", cgiRedirect, "/net/networks.html"},
    {"/net/scan.cgi", cgiWiFiScan, NULL},
    {"/net/status.cgi", cgi_wifi_status, NULL},
    {"/net/connect.cgi", cgi_connect_network, NULL},
    {"*", cgiEspFsHook, NULL}, //Catch-all cgi function for the filesystem
    {NULL, NULL, NULL}
};
```

The first three entries are simple redirects to get to the proper URL.
The entry for "/net/scan.cgi" uses a "CGI" function that comes with libesphttpd to scan for WiFi networks, and I use it in preference to writing my own. The next two "CGI" functions I did write, and they are used to get the current network status as JSON data, and to actually connect to the chosen networks in the chosen mode.

The second last entry is used to serve files from the flash storage. This should always be at the end, so as to only serve a file if there are no other specific matching entries. The final entry is all NULLs, which merely serves as a marker for the end of the built in URLs.

## Using libesphttpd

To use the libesphttpd in your own project, you need to first get the code, which is easy in a Git project by adding libesphttpd as a *sub-module*. A sub-module allows Git to track the changes in libesphttpd separately from your code, without having to make a copy and lose that association.

To create a sub-module for libesphttpd in your own Git project, simply run the following command in the top level of your project:

``` sh
git submodule add http://git.spritesserver.nl/libesphttpd.git/
```

You then need to create a "html" directory, which will store the static files and templates that are used by your web site. Sub-directories under the "html" directory will work just as you expect.

You then need to invoke the make file of libesphttpd from your own make file. I do this with the following snippet:

``` make
libesphttpd/Makefile:
    $(Q) echo "No libesphttpd submodule found. Using git to fetch it..."
    $(Q) git submodule init
    $(Q) git submodule update

libesphttpd: libesphttpd/Makefile
    $(Q) make -C libesphttpd USE_OPENSDK=yes
```

This will not only build the library, but create the "file system" for your project. This will create two libraries, one for libraries and the other containing the files. Both of these libraries are then included in your application.

## Web bootstrap

The web bootstrap system uses libesphttpd to serve a single HTML file. This file uses Javascript to read (via XMLHTTPRequest objects) the current status of the connection, and to scan for visible wireless networks to which the ESP8266 can be connected. Two separate user functions are used to retrieve this information, which is returned in JSON form. JSON is a great format for this, as it is easy to write for the ESP8266, and trivially easy to parse for Javascript.

When the user chooses a network and supplies a password, another function is called, which configures the network and stores the result in the ESP8266's flash memory for future use. The network is always configured (but not stored in flash) to allow both station and "soft AP" modes. Station mode is used to connect to a wireless network as a normal client. Soft AP mode has the ESP8266 acting as an access point. This is used to allow clients to connect to the ESP's network to configure the correct wireless network.

The code for the web bootstrap system is available on [GitHub] [github].

[github]: https://github.com/itmarshall/esp8266-projects/tree/master/web-bootstrap
