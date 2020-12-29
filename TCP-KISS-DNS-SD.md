
Specification for user-friendly DNS-SD discovery of KISS over TCP
=====================================================================

*COM1? COM2? COM13? /dev/ttyUSB4? /dev/cu.USBtoSerial14?*

*USB Sound Device? USB Audio CODEC? Default Sound Output?*

*127.0.0.1:8001 or 127.0.0.1:8042? TCP or UDP?*


Introduction
---------------

Most hams who have tried digital modes are quite familiar with the
difficulties of getting modem software attached to the correct sound card,
finding the correct virtual serial port device that's going to the radio,
and connecting logging software to the modem software. It gets even more
complicated when connecting to a device on the local network with dynamic
DHCP-assigned IP addresses.

In 2021 it is time to make one of these things a bit easier, and start
using DNS Service Discovery (RFC 6763 DNS-SD) over Multicast DNS (RFC 6762)
for locating KISS TNCs attached to the local network.

DNS-SD libraries exist for all relevant operating systems.  Finding DNS-SD
capable TNCs is very easy on Android and iOS platforms, as well as desktop
systems.  To the end user, it feels like magic, when compared to the
traditional hunting of dynamic IP addresses and weird port numbers.  Easier
than setting up a new printer, anyway.

User selects the correct TNC from a list, application connects to the TNC,
and the user does not need to know which IPs or ports are being used.

Example code for the TNC side can be found in the references below.


Specification
----------------

KISS TNCs providing a TCP socket server on a LAN shall announce their
presence, and the port being listened on, using mDNS DNS-SD with the
"_kiss-tnc._tcp" service in the "local." domain.


User-friendly names
----------------------

The **instance name** of each KISS TNC SHOULD be user-friendly, containing the
name of the software or the model and manufacturer of the device.  A few
examples of good names:

* ACME SuperTNC
* WinDSPTnc on NameOfServer
* Dire Fox on raspberrypi

As per [RFC 6763 Appendix C](https://tools.ietf.org/html/rfc6763#appendix-C),
the name of the TNC SHOULD NOT, by default, contain random hexadecimal
gibberish, such as serial numbers or MAC addresses, to make the name unique.
RFC 6763 describes, in [Appendix D](https://tools.ietf.org/html/rfc6763#appendix-D),
that when the device detects a naming conflict, it should add or increment
a number in the end of the name, resulting an user-friendly name like
"ACME SuperTNC (2)".

If possible, the adjusted name SHOULD be stored persistently, so that it
will remain assigned to the correct device in case both devices remain on
the same network, even if the device is turned off temporarily.

If the device has an user interface, it is also a good idea to make the name
configurable, so that the user can have meaningful names for multiple
TNCs attached to different radios on different frequencies.

If the device has a [BLE KISS interface](https://github.com/hessu/aprs-specs/blob/master/BLE-KISS-API.md),
the same name MUST be used for both BLE and DNS-SD.


Debugging
------------

Note that some APIs require you to pass the TCP port in *network byte order*
when registering the DNS-SD announcement.  On Mac dnssd, netwoork byte
order; on Linux Avahi, host byte order.  Use htons() to do the conversion so
that your code will work on both Big Endian and Little Endian computers.

On a Linux system with Avahi and avahi-utils installed, you can use
avahi-browse to fetch currently discoverable TNCs from the cache. It'll also
continue to print updates of newly added and removed TNCs.

    $ avahi-browse -v _kiss-tnc._tcp
    Server version: avahi 0.7; Host name: raspberrypi.local
    E Ifce Prot Name                           Type                 Domain
    +   eth0 IPv6 Dire Wolf on Hessu-mac-2     _kiss-tnc._tcp       local
    +   eth0 IPv4 Dire Wolf on raspberrypi     _kiss-tnc._tcp       local
    +   eth0 IPv4 Dire Wolf on Hessu-mac-2     _kiss-tnc._tcp       local
    : Cache exhausted
    : All for now

To resolve the IP addresses and ports on Linux:

    $ avahi-browse -v -r _kiss-tnc._tcp
    Server version: avahi 0.7; Host name: raspberrypi.local
    E Ifce Prot Name                                          Type                 Domain
    +   eth0 IPv4 Dire Wolf on Hessu-mac-2                      _kiss-tnc._tcp       local
    =   eth0 IPv4 Dire Wolf on Hessu-mac-2                      _kiss-tnc._tcp       local
    hostname = [Hessu-mac-2.local]
    address = [10.11.0.106]
    port = [8001]
    txt = []

On a Mac, the "dns-sd" tool will do the same.

    $ dns-sd -B _kiss-tnc._tcp
    Browsing for _kiss-tnc._tcp
    DATE: ---Tue 29 Dec 2020---
     9:31:26.959  ...STARTING...
    Timestamp     A/R    Flags  if Domain   Service Type     Instance Name
     9:31:26.960  Add        3   6 local.   _kiss-tnc._tcp.  Dire Wolf on raspberrypi iGate 144.800
     9:31:26.960  Add        3   1 local.   _kiss-tnc._tcp.  Dire Wolf on Hessu-mac-2
     9:31:26.960  Add        2   6 local.   _kiss-tnc._tcp.  Dire Wolf on Hessu-mac-2

To resolve the hostname and TCP port (8001):

    $ dns-sd -L "Dire Wolf on Hessu-mac-2" _kiss-tnc._tcp
    Lookup Dire Wolf on Hessu-mac-2._kiss-tnc._tcp.local
    DATE: ---Tue 29 Dec 2020---
     9:32:49.275  ...STARTING...
     9:32:49.275  Dire\032Wolf\032on\032Hessu-mac-2._kiss-tnc._tcp.local. can be reached at Hessu-mac-2.local.:8001 (interface 1) Flags: 1
     9:32:49.275  Dire\032Wolf\032on\032Hessu-mac-2._kiss-tnc._tcp.local. can be reached at Hessu-mac-2.local.:8001 (interface 6)

And finally the IP address:

    $ dns-sd -G v4 Hessu-mac-2.local.
    DATE: ---Tue 29 Dec 2020---
     9:33:13.662  ...STARTING...
    Timestamp     A/R    Flags if Hostname            Address       TTL
     9:33:13.662  Add 40000002  6 Hessu-mac-2.local.  10.0.0.42     120


References
--------------

* RFC 6762, mDNS / Multicast DNS: https://tools.ietf.org/html/rfc6762
* RFC 6763, DNS-SD: https://tools.ietf.org/html/rfc6763
* Pull Request to add DNS-SD support to Dire Wolf: https://github.com/wb2osz/direwolf/pull/307

