
Specification for KISS over BLE (Bluetooth Low Energy)
================================================================

Introduction
---------------

This is a specification for transmitting frames using the KISS
protocol over BLE (Bluetooth Low Energy).  Typically KISS is used for
exchanging AX.25 packet radio frames with a KISS TNC but other link layers
than AX.25 can also be used.

BLE is important because Apple iOS (iPhones, iPads) does not allow apps to
talk to USB devices or classic Bluetooth devices (Bluetooth Serial profile). 
Apps can, however, talk to BLE devices just fine.

BLE is available to Android and other devices just as well. It's just that
on iOS there's no other option.

BLE allows the apps to discover nearby BLE devices which support a specific
profile.  An APRS app can neatly show TNCs (and TNCs only) and let the user
pick one from a list.

This protocol is currently implemented by the aprs.fi iOS iPhone & iPad
application, and the Mobilinkd TNC3.  Authors of both products sincerely
hope that other future BLE-capable TNCs and APRS radios would implement the
same protocol, so that the ecosystem could grow faster without a lot of
reimplementation work.

If you produce a radio or TNC which supports this protocol, it will *just
work* with the aprs.fi iOS app and any others with BLE KISS support.  Make
it, test it, ship it.


Magic UUID constants
------------------------

BLE KISS service UUID:

    KTS_SERVICE_UUID 00000001-ba2a-46c9-ae49-01b0961f68bb

Notify, read KISS data:
    
    KTS_RX_CHAR_UUID 00000003-ba2a-46c9-ae49-01b0961f68bb

Write KISS data, response:

    KTS_TX_CHAR_UUID 00000002-ba2a-46c9-ae49-01b0961f68bb


Discovering BLE KISS devices
-------------------------------

An application looking for BLE KISS TNCs does a scan for BLE peripherals
which advertise the service UUID of KTS_SERVICE_UUID.

Each TNC device MUST have an unique BLE peripheral identifier UUID, so that
multiple TNCs of the same model can be present at the same time and selected
separately.

Each TNC MUST provide a BLE peripheral name string which clearly identifies
the manufacturer and model of the TNC, such as "ACME SuperTNC 7".  If the
TNC has a configuration tool or interface, the name may also be configurable
by the user, but the default name MUST still identify the manufacturer and
model of the TNC.  Since most TNCs will not have such a feature, the
application doing the discovery may also display the identifier UUID.

After the user has selected one of the discovered BLE TNCs, and a connection
has been established successfully, the application may store the unique BLE
peripheral identifier UUID in persistent configuration, so that it can later
automatically reconnect to the same TNC at startup.


Discovery of characteristics and subscribing to notifications
----------------------------------------------------------------

After establishing a connection to the BLE peripheral an application shall
discover the services on that peripheral having the service UUID of
KTS_SERVICE_UUID.  The TNC MUST provide one such service.

The application shall then discover the characteristics for that service. 

This document specifies two characteristics with UUIDs KTS_RX_CHAR_UUID and
KTS_TX_CHAR_UUID, but more may be defined later.  Applications must ignore
other characteristics which may be implemented by TNCs later and happily
use these two characteristics if they are present.

When a characteristic for KTS_RX_CHAR_UUID is found, the application shall
subscribe to notifications for that characteristic.  This will enable the
receive path for KISS frames.

When a characteristic for KTS_TX_CHAR_UUID is found, the application shall
keep a reference to that characteristic, as it will be used for transmitting
KISS frames later on.


Receiving KISS frames from a BLE TNC
---------------------------------------

Received frames are transferred KISS encoded raw byte arrays on the
KTS_RX_CHAR_UUID peripheral.

Since the length of a BLE transfer (MTU) is limited, a KISS frame may span
multiple BLE transfer units.  Likewise, a single BLE transfer may contain
more than one small KISS frame, or the tail and beginning of two larger KISS
frames.

A BLE TNC shall wrap received binary frames in a KISS frame and escape
any KISS FEND packets appearing in the frame as specified by the KISS
protocol.  The KISS frames shall then be sent on the KTS_RX_CHAR_UUID
peripheral, back to back.

A TNC MUST be capable of splitting a larger frame to multiple BLE
transfer units, should the frame be larger than a BLE transfer unit.  The
TNC MAY optionally be capable of inserting another KISS frame, or the
beginning of a frame, if there's space in the BLE transfer unit after the
previous KISS frame is filled in, and if there is another KISS frame waiting
to be transferred.

An application receiving data from a BLE TNC MUST concatenate the received
BLE transfer units in a buffer, and scan for complete KISS packets according
to the KISS protocol by looking for a KISS FEND byte in the beginning and
end of each KISS frame, just as if the data was coming from a serial line. 
The application must also handle KISS escaping according to the KISS
protocol, as the data contents may well contain the FEND byte.

The receive buffer must be large enough to fit a complete KISS frame.  The
maximum length an AX.25 frame (ignoring AX.25 v2.2 connected mode for a
while) is 329 bytes.  In the very unlikely case, where all bytes would be
FEND or FESC and they would be escaped, the maximum KISS frame size in a
buffer would be 2*329+2 = 660 bytes.  In 2020 a few kilobytes for a buffer
should be easy to find.

* AX.25 header length: 17 bytes
* AX.25 address length: 7 bytes
* AX.25 maximum digipeaters: 8
* AX.25 default MTU: 256 bytes
* AX.25 maximum length: 17+7*8+256 = 329

The MTU may be negotiated between two connected stations to a larger value
in AX.25 2.2.  This can not be done for UI frames which are used in APRS.


Transmitting KISS frames via a BLE TNC
-----------------------------------------

Similar to the receive path, transmitted frames are sent by the application,
KISS encoded, as binary byte arrays, on the KTS_TX_CHAR_UUID characteristic.

Longer frames are split into multiple BLE transfer units.  Each unit may
contain multiple KISS frames.  The beginning of the next KISS frame may be
used to fill the BLE data transfer unit right after the end of the previous
frame, so the TNC must assemble received BLE messages in a buffer and parse
out KISS encoded frames as if the buffer was being filled from a serial
line.

Likewise, the buffer length requirements described above apply for the
transmit buffer in the TNC.  As the radio interface is usually quite slow
(1200 bit/s or 9600 bit/s), and the channel may be busy for extended periods
of time, the TNC must be able to buffer multiple packets waiting to be
transmitted.


KISS Command codes
---------------------

KISS command codes other than "Data frame" 0x00 may be used by the
application to configure channel access parameters of the TNC.  These
command frames are small and may be easily sent in a single BLE
transfer.

The KISS specification specifies that only the "Data frame" code of 0x00
should be sent from the TNC to the host.  In practice some TNCs may well
send other codes.  For example, the TNC3 uses 0x06 SetHardware codes to
communicate with the configuration utility.  Those frames will at times
accidentally be delivered to other applicatins, and they MUST ignore them
silently.


Closing BLE
--------------

When disconnecting from a TNC, the application shall unsubscribe from
notifications for KTS_RX_CHAR_UUID, and then disconnect from the peripheral.


Expanding the protocol, vendor extensions
--------------------------------------------

If the TNC or radio implementing BLE needs to have features such as rig
control and configuration, such protocols can be implemented by adding
custom characteristics and services with a vendor-specific UUID.  This gives
you a free playground without fear of collisions which would be more likely
if extensions would be developed on the KISS layer.

As a BLE device may announce supporting multiple services, a BLE KISS device
may list both KTS_SERVICE_UUID and a vendor-specific UUID to indicate
support for an extension. This is preferred instead of string matching with
the device or manufacturer name.

If you wish to provide an open protocol for extended features like this,
please submit a pull request to document it here.  This will help other
apps support your hardware.

For example, if SMACK, TNC2 command mode, or other alternate protocol needs
to be implemented, separate characteristic UUIDs should be used.

The BLE specification would allow selecting any random UUID, but if a Nordic
Semiconductor BLE module is used, only the third and fourth octet of the
UUID may vary.  If you're using a Nordic chip or another chip with a
similar restriction, please select a random 10-bit integer (in the range
1...1023) to use as your vendor code, and request for it to be allocated to
you in this document by making a github ticket. Vendor code 0 is allocated
for services and characteristics specified in this document.

To pick vendor UUIDs for services and characteristics:

* Generate a 16-bit integer where the high 10 bits are your vendor code, and
  low 6 bits identify your own characteristic. The 16-bit integer goes to
  the 3rd and 4th octet of the UUID, marked with V in this example UUID:
  0000VVVV-ba2a-46c9-ae49-01b0961f68bb
* Use 0 for the low 6 bits for the first characteristic or service, 6-bit
  integer value of 1 for the second UUID and so on. You'll get 64 UUIDs
  for the vendor code, each of which can be used for either a characteristic
  or a service.  If you run out, you can allocate a second vendor code.
* For example, for a vendor code of 595, the 16-bit integer for service 0
  would be (513 << 6) == 0x94c0, resulting in the first UUID of
  000094c0-ba2a-46c9-ae49-01b0961f68bb.
* The second and third UUIDs for vendor code of 595 would have octets of
  (513 << 6) | 1 == 0x94c1 and (513 << 6) | 2 == 0x94c2, resulting in
  UUIDs of 000094c1-ba2a-46c9-ae49-01b0961f68bb and
  000094c2-ba2a-46c9-ae49-01b0961f68bb.


Implementation examples
--------------------------

Rob Riggs of Mobilinkd has kindly published the TNC3 firmware as open
source.  Note that it is licensed under the GPLv3, so if you incorporate his
source code in your product and distribute the product, you need to give the
source code of your firmware out too.  This is great, since others can then
make your hardware better and more valuable by adding features and fixing
bugs.  Using existing source code can make your product development faster
and since the TNC3 BLE/KISS firmware is already well tested you'll have less
bugs when you ship!

https://github.com/mobilinkd/tnc3-firmware

The Mobilinkd TNC3 iOS Config App source code is also available under the Apache
license. It implements the TNC discovery and connection phases, and enough
KISS to configure TNC hardware parameters.

https://github.com/mobilinkd/iosTncConfig


Known implementations
------------------------

Client applications:

* aprs.fi APRS application (iPhone & iPad)
* RadioMail WinLink messaging app (iPhone & iPad)
* PocketPacket APRS app (iPhone & iPad)

Radios and TNCs:

* Mobilinkd TNC3
* Mobilinkd TNC4
* PicoAPRS v4 transceiver
* RPC Electronics ESP32-APRS Tracker
* Richonguzman / CA2RXU LoRa APRS Tracker on TGO T-Beam and LoRa32v2.1 boards

By supporting the BLE KISS API your device or application will be compatible
with these existing implementations without any changes in those.


References
-------------

* Adafruit Introduction to Bluetooth Low Energy: GATT:
  https://learn.adafruit.com/introduction-to-bluetooth-low-energy/gatt
* Getting Started with Bluetooth Low Energy:
  https://www.oreilly.com/library/view/getting-started-with/9781491900550/ch04.html
* KISS: https://en.wikipedia.org/wiki/KISS_(TNC)
* KISS specification: http://www.ax25.net/kiss.aspx
* AX.25 v2.2: https://www.tapr.org/pdf/AX25.2.2.pdf
