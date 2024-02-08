
Amateur radio DNS-SD Zero-configuration services
====================================================

Most hams who have tried digital modes and configured integrations between
logging software, modem software and radios on a network, are quite familiar
with the challenges of finding the correct dynamic IP address and port of
each application and device on the network.  Many have been seriously
frustrated about it being unnecessarily difficult, much more difficult
than it should be in the 2020s.

By implementing DNS-SD zeroconf we can make service and device discovery
very easy - just like finding a printer on a modern computer.  With broad
DNS-SD support in modern operating systems it is not difficult for
programmers to implement.  For the end user it all feels like magic.

This document lists known DNS-SD zeroconf services in the field of ham
radio.

* TCP KISS TNCs for packet radio and APRS
  * Service: `_kiss-tnc._tcp`
  * Instance name: User-friendly UTF-8 encoded Unicode name of the TNC
  * Implementations: Direwolf, aprs.fi for iOS, RadioMail for iOS
  * Specification: [TCP-KISS_DNS-SD.md](https://github.com/hessu/aprs-specs/blob/master/TCP-KISS-DNS-SD.md)
  * Example implementation: [Direwolf](https://github.com/wb2osz/direwolf/tree/dev/src)
    is open source, implementations included for MacOS and Linux (Avahi)
* VARA HF modem launched with Varanny
  * Service: `_varahf-modem._tcp`, `_varafm-modem._tcp`
  * Implementations: varanny, RadioMail for iOS
  * Specification: [varanny](https://github.com/islandmagic/varanny)
  * Example implementations: [varanny](https://github.com/islandmagic/varanny)
    is open source, multiplatform implementation in Go


Example of an user interface
--------------------------------

The aprs.fi iOS app supports KISS TNCs via WiFi, discovered using DNS-SD.
BLE and TCP KISS TNCs can be connected by simply selecting them from a list.
The user does not need to be concerned by what IP address or TCP port the
Direwolf TNC is using. This view shows two Dire Wolf DSP modems running
on two different computers (Raspberry Pi, Macbook). Each Dire Wolf has been
configured with a user-friendly name for easy identification.

![DNS-SD on iPhone](images/tcp-kiss-dns-sd-iphone.png?raw=true)


Ideas for future implementations
-----------------------------------

* Logging software

  A logging application running could announce its TCP or UDP port using
  DNS-SD, so that client applications like digital mode modem software
  could find the correct IP and port.

  This can be done on a local computer too - the modem software could
  let the user pick a logging software from a list of automatically
  found logging apps, whether they are on the local computer or
  somewhere on the wifi/LAN.

  Many logging applications support a common, shared QSO submission
  protocol, which should be announced with a single, common DNS-SD
  service name, so that all clients can find all loggers easily.

* Contest logging software

  When the same contest logging application is run on multiple computers
  on the same LAN, each instance could well announce itself using DNS-SD,
  allowing other instances to find the IPs and ports easily.
	
* Radios with built-in Ethernet interfaces

  Many modern SDR-based radios have an Ethernet or Wifi interface. They
  could well announce themselves using DNS-SD, so that clients could
  locate their IPs and ports, and users wouldn't have to resort to
  configuring static IPs, or trying to guess which dynamic IP the
  DHCP server has given this time.

* Rig control

  TCP services for doing CAT control of rigs could be announced using
  DNS-SD, such as `_rigctl._tcp`.

* Antenna switches

  Antenna switches with APIs should announce their services with a new
  DNS-SD service, such as `_antenna-switch-rest-api._tcp`.


Best practices for implementation
------------------------------------

### Support UTF-8 Unicode encoding for user-friendly names

When users are allowed to configure multiple instances of their software
or devices (the same contest logging software on multiple computers,
the same modem software attached to different radios on different
frequencies), it is a good practice to let the user give meaningful
names to each instance.

UTF-8 should be supported for these names, so that users can use their
native language in the naming.

### Use a single DNS-SD service name for all services using the same protocol

When providing multiple instances of the same service, and all of those
services use the same protocol when communicating with a client, announce
all of those services using the same DNS-SD service name.

This allows a client to find all instances with a single search, and
additional instances and variants can be added without changes to the
clients, as long as they support the same protocol.

If additional machine-readable parameters need to be passed to clients,
such as additional TCP ports, communication parameters or other properties,
they can be added to each announcement as TXT fields, without resorting
to to additional service names or polluting the user-visible friendly name
with machine-readable gibberish.

For example, a modem software using the KISS TCP protocol may support
running many instances of the software on the same computer, on different
TCP ports.  Each instance can be easily announced with the `_kiss-tnc._tcp`
service, allowing clients to find all of them, with each announcement
specifying a different user-visible name and TCP port.

If they would have been announced separately as
`_vhf-1200baud-kiss-tnc._tcp` and `_hf-300baud-kiss-tnc._tcp`, a client
would have to be specifically programmend to query for the two.  That client
would then not know about a future `_uhf-9600baud-kiss-tnc._tcp` service,
limiting compatibility.

### Use a new and different DNS-SD service for a new and different protocol

When implementing a new and different control protocol, pick a new DNS-SD
service name.  If there is an existing antenna switch control protocol using
`_antenna-control-http-rest._tcp` and a new, different, improved but
incompatible control protocol is developed, do not announce it as
`_antenna-control-http-rest._tcp`, so that clients using the old protocol
will not try to connect to it.  Pick a new service name.

### Document your protocols

Provide a document for each protocol, and the corresponding DNS-SD
announcement.  This helps others when implementing clients for your service.

It will also help others to create new services for the existing clients,
further growing the ecosystem and making digital applications for amateur
radio more accessible.

