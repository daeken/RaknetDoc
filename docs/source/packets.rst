Packets
=======

Below you'll find detailed descriptions of each packet used in Raknet.

Data Types
----------

Unless otherwise specified, all types are in **big-endian format**, or "network order".

.. list-table:: Data Types
   :widths: 15 25 25 35
   :header-rows: 1

   * - Name
     - Size (bytes)
     - Signedness
     - Notes
   * - uint8
     - 1
     - unsigned
     -
   * - bool
     - 1
     - unsigned
     - non-zero == true
   * - uint16
     - 2
     - unsigned
     -
   * - uint32
     - 4
     - unsigned
     -
   * - uint64
     - 8
     - unsigned
     -
   * - uint24
     - 3
     - unsigned
     - **LITTLE ENDIAN**
   * - endpoint
     - 7 or 19
     - unsigned
     - see below
   * - magic
     - 16
     - n/a
     - see below
   * - string
     - 2+
     - unsigned
     - see below

`endpoint`
^^^^^^^^^^

Endpoints are a compact representation of an IP address and port. Because it needs to support IPv4 and IPv6, they contain a type tag to differentiate between the two. In order, the structure is:

1. `uint8 type` -- either 4 or 6
2. The IP address
    - For IPv4 addresses, the bits are all inverted and the bytes are written in big-endian (`uint32`). E.g. 101.5.4.2 becomes `9A FA FB FD`
    - For IPv6 addresses, the bits are left unchanged and the bytes are written out in big-endian format.
3. `uint16 port` -- the port number for the endpoint

`magic`
^^^^^^^

This is a special sequence used in several Raknet packets to, presumably, prevent errant packets from being detected as potential connections to servers. It is the following 16 bytes:

`00 FF FF 00 FE FE FE FE FD FD FD FD 12 34 56 78`

`string`
^^^^^^^^

Strings in Raknet consist of a `uint16` length in bytes, then that number of UTF-8-encoded bytes. No null-termination is present.

Unconnected Packets
-------------------

`0x01` - Unconnected Ping (Fetch MOTD)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x1C` - Unconnected Pong (MOTD)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x05` - Open Connect Request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x06` - Open Connect Reply
^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x07` - Session Info Request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x08` - Session Info Reply
^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x19` - Incompatible Protocol Version
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Connected Packets
-----------------

`0x00` - Connected Ping
^^^^^^^^^^^^^^^^^^^^^^^

`0x03` - Connected Pong
^^^^^^^^^^^^^^^^^^^^^^^

`0x04` - Lost Connection
^^^^^^^^^^^^^^^^^^^^^^^^

`0x09` - Connection Request
^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x10` - Connection Accept
^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x13` - New Connection
^^^^^^^^^^^^^^^^^^^^^^^

`0x15` - Disconnect
^^^^^^^^^^^^^^^^^^^^^^^^^^^

`0x80-0x8D` - Frame Set
^^^^^^^^^^^^^^^^^^^^^^^

`0xA0` - NAck
^^^^^^^^^^^^^

`0xC0` - Ack
^^^^^^^^^^^^

`0xFE` - Game Data
^^^^^^^^^^^^^^^^^^
