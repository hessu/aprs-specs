
Specifications for amateur radio protocols
==============================================

Here are a few specifications for ham radio protocols and their extensions.

* [Amateur radio DNS-SD Zero-configuration services](hamradio-zeroconfig.md)
  provides an index of known DNS-SD services published by amateur radio
  software and devices, and provides guidelines and ideas for new
  implementations.
* [Specification for user-friendly DNS-SD discovery of KISS over TCP](TCP-KISS-DNS-SD.md)
  describes how DNS-SD is used to publish packet radio KISS TNCs
  on a LAN/WiFi, so that users of packet radio and APRS applications
  no longer need to figure out and type IP addresses and port numbers.
  Implemented in multiple applications. The document is also useful
  for implementing other similar DNS-SD services.
* [Specification for KISS over BLE (Bluetooth Low Energy)](BLE-KISS-API.md)
  describes how KISS frames are sent over Bluetooth 4.0 Low Energy
  (BLE), with unique service identifiers allowing packet radio
  and APRS applications to easily discover KISS devices within
  range, without confusing them with other BLE devices. Implemented
  in multiple applications and devices.
* [APRS Base91 comment telemetry](aprs-base91-comment-telemetry.txt)
  describes an APRS extension for sending telemetry in the comment
  field of a position packet, with higher resolution and fairly
  tight packing of the data. The sequence number is also used
  by aprs.fi to deduplicate position packets, and the sequence number
  can be sent alone without sending any telemetry values.

