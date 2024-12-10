High-Level Overview
===================

Raknet's overall design is set up to provide a reliable channel for packets over UDP. Unlike TCP, Raknet doesn’t provide a continuous stream but instead ensures reliability for discrete packets with lower overhead.

The protocol is split between "unconnected" and "connected" phases. In the unconnected phase, no reliability is present; this is used for transmission of the server MOTD, unconnected pings/pongs, and the initial handshake. In the connected phase, the reliability layer comes into play and packet sequencing, ordering, and fragmentation are all available; the second handshake, connected pings/pongs, and application-level packets occur on top of this.

Unconnected Phase
-----------------

Connected Phase
---------------
