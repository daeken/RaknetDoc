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

General Packet Format
---------------------

Packets all begin with a single-byte type then the data detailed below. This byte isn't included in the structure as documented.

Unconnected Packets
-------------------

`0x01` - Unconnected Ping (Fetch MOTD)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Client->Server**

Structure:
""""""""""

1. `uint64 timestamp_in_milliseconds` -- From Unix Epoch (UTC)
2. `magic`
3. `uint64 client_id` -- Seems to be unused; 0 is used by some clients, random by others and servers don't seem to do anything with it.

Purpose:
""""""""

In addition to checking for the server's responsiveness, this packet gets you the server MOTD. It is not strictly necessary, but will typically be the first packet sent by a client.

`0x1C` - Unconnected Pong (MOTD)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Server->Client**

Structure:
""""""""""

1. `uint64 timestamp_in_milliseconds` -- From Unix Epoch (UTC)
2. `uint64 server_id` -- Seems to be unused; 0 is used by some servers, random by others, and a consistent value by even others. Don't rely on this to be consistent.
3. `magic`
4. `string motd`

MOTD Format:
""""""""""""

The MOTD is a semicolon(`;`)-delimited string containing the following elements:

1. "MCPE" (Minecraft Pocket Edition -- standard Bedrock) or "MCEE" (Minecraft Educational Edition)
2. Server name
3. Protocol version (decimal)
4. Server version
5. Number of players online (decimal)
6. Maximum number of players allowed (decimal)
7. Game mode (decimal)
    - 0: Survival
    - 1: Creative
    - 2: Adventure
    - 3: Spectator
8. Server GUID (as decimal)
9. Server "sub name"
10. Game mode (string -- identical to the enum above)
11. [Optional] "Nintendo limited" -- 1 indicates it's accessible from Switch, 0 indicates that it should not be, AFAICT. (Lifeboat sends 1.)
12. [Optional] IPv4 port
13. [Optional] IPv6 port

It is currently unknown to me what makes these optional fields occur and they probably shouldn't be relied on by clients. (If you have any clue, drop an issue on the Github repo!)

Purpose:
""""""""

Provides key details about the server, e.g. whether the protocol version is one that a client is compatible with. It also can be used as a simple pong, but this use appears to be rare.

`0x05` - Open Connect Request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Client->Server**

Structure:
""""""""""

1. `magic`
2. `uint8 protocol_version` -- This is the **Raknet** protocol version, currently 11. Not to be confused with the Minecraft protocol version sent in the MOTD.
3. `uint8[mtu_size] padding` -- MTU size will be discussed in the Purpose section

Purpose:
""""""""

This packet is primarily for figuring out the maximum single datagram size that the client and server can send each other. Clients should start with a reasonably high value (e.g. 2000) and continue to decrease it until it receives a reply. Technically this isn't the "true" MTU size, as it doesn't factor in the IP header, UDP header, or the size of the packet itself.

`0x06` - Open Connect Reply
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Server->Client**

Structure:
""""""""""

1. `magic`
2. `uint64 server_id` -- Unlike the `server_id` used in the Unconnected Pong, this *does* appear to be consistent
3. `bool has_security`
4. [Only when has_security is true] `uint32 cookie`
5. `uint16 mtu_size` -- The actual accepted MTU size, minus overhead

Purpose:
""""""""

This packet serves to let the client know 1) that their MTU was accepted (and remember, this is UDP so you may receive a second or even third reply to this depending on how quickly you're checking them; only accept the first so you are maximizing bandwidth and reducing latency as much as possible), and 2) the cookie value, if needed.

`0x07` - Session Info Request
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Client->Server**

Structure:
""""""""""

1. `magic`
2. [Only when has_security was true in the Open Connect Reply] `uint32 cookie` -- The cookie sent by the server
3. [Only when has_security was true in the Open Connect Reply] `uint8 unk` -- Always 0; unknown purpose. Maybe a bool indicating something else is present?
4. `endpoint server` -- The server's IP and port
5. `uint16 mtu_size` -- As accepted by the server
6. `uint64 client_id` -- Needs to be consistent with the later Connection Request packet, but can be chosen randomly

Purpose:
""""""""

Requests information about the session and completes the client side of the unconnected portion of the handshake.

`0x08` - Session Info Reply
^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Server->Client**

Structure:
""""""""""

1. `magic`
2. `uint64 server_id` -- The same ID sent in Open Connect Reply
3. `endpoint client` -- The client's IP and port
4. `uint16 mtu_size` -- Again, the same MTU size from the previous packets
5. `bool has_encryption` -- Always false in Minecraft, AFAICT. This enables Raknet-level encryption, but this appears to be unused (and possibly unsupported) by Minecraft as they have their own app-level encryption.

Purpose:
""""""""

Completes the server side of the unconnected portion of the connection handshake.

`0x19` - Incompatible Protocol Version
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

**Server->Client**

Structure:
""""""""""

1. `uint8 protocol_version` -- The protocol version requested by the client
2. `magic`
3. `uint64 server_id` -- Appears to be the same server ID used in Open Connect Reply and Session Info Reply

Purpose:
""""""""

Sent in response to an Open Connect Request with an incompatible Raknet version. (Which, again, should be 11.)

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
