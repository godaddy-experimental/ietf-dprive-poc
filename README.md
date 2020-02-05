# ietf-dprive-poc
Proof of Concept stuff for IETF standards work

Very high level elevator pitch:
This is stuff to enable a client host, to upgrade its DNS resolver connection from ordinary DNS (unencrypted) to DNS-over-TLS.
There is no standard for any of the needed bits, so I have proposed ways to do this.
The proof-of-concept is to show it can be done and works.

Example code is in perl, to implement things via config changes
Long term expectation is various implementations will build the logic/functionality into their respective components
(Not stuff I'll be working on directly, and lots of implementations hopefully will do this.)

(The PoC itself is a big perl script)

It is doing stuff that involves:
- creating self-signed TLS certs (using the openssl toolkit)
- doing DNSSEC key generation
- writing DNS zone files
- creating/modifying configs for open source DNS packages "bind" and "knot-resolver".

There will also be a PDF with the explanation for presenting.
