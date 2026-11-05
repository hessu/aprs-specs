
Specification for DNS-SD discovery of DX cluster telnet services
===================================================================

*localhost:7300? 192.168.1.23:7373? Was it 7550 for the skimmer?*

PROPOSAL

2026-11-05 - first version

Heikki Hannikainen, OH7LZB


Introduction
---------------

A lot of software serves DX spots over the same plain TCP "telnet"
protocol: DX cluster nodes (DXSpider, AR-Cluster, CC Cluster), CW
Skimmer and Skimmer Server, the RBN Aggregator, and CW decoders such as
CW Whole Milk.  Logging and contest programs connect to it to fill their
band maps.  The protocol has been the same since the AK1A PacketCluster:
lines of text, with each spot looking like this:

    DX de N0CALL-#:  14028.7  Q0ABC        CW   21 dB  38 WPM  CQ     1401Z

Every logger speaks it, and every one of them asks the user for an IP
address and a port number.  When the spot source is a skimmer running on
another computer in the shack, with an address handed out by DHCP, that
is exactly the guessing game DNS-SD makes unnecessary.

Since all of these servers speak the same protocol, they should all be
announced with the **same** DNS-SD service type, so that a logger finds
every spot source on the network with a single browse - the cluster node
on the Raspberry Pi, the skimmer on the SDR computer and the decoder on
the laptop - and one written today will also find the ones written
later.  This document specifies that service type.


Specification
----------------

A server providing DX spots over the DX cluster telnet protocol on a TCP
port SHALL announce its presence, and the port being listened on, using
mDNS DNS-SD with the **"_dx-cluster._tcp" service** in the **"local."
domain**.  The same service type MAY also be used for wide-area (unicast)
DNS-SD, for example to announce a public cluster node in the operator's
own DNS zone.

The announcement SHOULD be registered while the server is accepting
connections, and withdrawn when it stops.  A server which can be switched
on and off by the user should register and deregister accordingly.

The service type means "this port speaks the DX cluster line protocol":
after connecting, the client receives text lines, terminated by CR LF,
in which DX spots begin with `DX de `.  The server MAY first send a
banner and a `login:` prompt and expect the client to send a callsign
followed by a line ending; clients SHOULD be prepared to answer such a
prompt, as they already do for existing cluster nodes.  A server which
accepts cluster commands (`sh/dx`, `set/skimmer` and so on) may of course
continue to do so; the announcement does not promise any particular
command set beyond the spot lines.

A service using a **different** protocol for spots - JSON, a WebSocket,
an HTTP API - MUST NOT use `_dx-cluster._tcp`, so that loggers expecting
spot lines will not connect to it.  It should pick a service type of its
own, as described in
[hamradio-zeroconfig.md](https://github.com/hessu/aprs-specs/blob/master/hamradio-zeroconfig.md).

### Instance name

As per RFC 6763, the **instance name** is a user-friendly UTF-8 encoded
Unicode name.  It SHOULD name the software, and the computer or the
cluster node, so that the user can tell the sources apart.  Servers
with a configuration interface SHOULD allow the user to change the name
and permit non-ASCII characters.  Client applications MUST handle names
containing non-ASCII Unicode characters.

A few examples of good names:

* CW Skimmer on SHACK-PC
* DXSpider N0CALL-2
* CW Whole Milk on Hessu's Mac
* Kuuntelija mökillä

As with [KISS TNCs](TCP-KISS-DNS-SD.md), the name SHOULD NOT contain
serial numbers, MAC addresses or other hexadecimal gibberish to make it
unique.  On a name conflict the server should add or increment a number
at the end of the name, giving "CW Skimmer on SHACK-PC (2)", as described
in RFC 6763 Appendix D.  Most DNS-SD libraries do this automatically.

### TXT record

No TXT record keys are defined in this version.  A server MAY publish an
empty TXT record.  Clients MUST ignore TXT keys they do not recognize,
so that keys can be added to this document later without breaking them.

If a server needs to tell clients something machine-readable - a second
port, a login requirement - please propose a key here rather than
putting it in the instance name.


Client behaviour
-------------------

A logger supporting this specification browses for `_dx-cluster._tcp`
and presents the instance names in a list.  When the user picks one, the
client resolves it to a host and a port and connects as it would to a
manually configured cluster.  The client SHOULD store the instance name
rather than (or in addition to) the resolved address, and resolve it
again when reconnecting: the address of a computer on a DHCP network
changes, and the name does not.


Announcing an existing server
--------------------------------

A cluster node which does not yet announce itself can be announced on its
behalf.  On Linux with Avahi, place a file like this in
`/etc/avahi/services/dx-cluster.service`:

    <?xml version="1.0" standalone='no'?>
    <!DOCTYPE service-group SYSTEM "avahi-service.dtd">
    <service-group>
      <name replace-wildcards="yes">DXSpider on %h</name>
      <service>
        <type>_dx-cluster._tcp</type>
        <port>7300</port>
      </service>
    </service-group>

Or from the command line:

    # Linux, Avahi
    avahi-publish -s "DXSpider on raspberrypi" _dx-cluster._tcp 7300
    # macOS
    dns-sd -R "DXSpider on raspberrypi" _dx-cluster._tcp local 7300
    # macOS, proxy announcement for a server running on another host
    dns-sd -P "CW Skimmer on SHACK-PC" _dx-cluster._tcp local 7300 \
        shack-pc.local. 192.168.1.23


Debugging
------------

On a Mac, browse for the servers and resolve one of them:

    $ dns-sd -B _dx-cluster._tcp
    Browsing for _dx-cluster._tcp
    Timestamp     A/R    Flags  if Domain   Service Type        Instance Name
    16:49:51.187  Add        3   1 local.   _dx-cluster._tcp.   CW Whole Milk on Hessu mac

    $ dns-sd -L "CW Whole Milk on Hessu mac" _dx-cluster._tcp
    Lookup CW Whole Milk on Hessu mac._dx-cluster._tcp.local
    16:49:57.203  CW\032Whole\032Milk\032on\032Hessu\032mac._dx-cluster._tcp.local. can be reached at Hessu-mac-2.local.:7300 (interface 14)

On Linux with avahi-utils:

    $ avahi-browse -r _dx-cluster._tcp


Implementations
------------------

* CW Whole Milk (macOS CW decoder), announces its DX cluster port while
  it is switched on.


References
--------------

* RFC 6762, mDNS / Multicast DNS: https://tools.ietf.org/html/rfc6762
* RFC 6763, DNS-SD: https://tools.ietf.org/html/rfc6763
* [Amateur radio DNS-SD Zero-configuration services](hamradio-zeroconfig.md)
* [Specification for user-friendly DNS-SD discovery of KISS over TCP](TCP-KISS-DNS-SD.md)
