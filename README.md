# ietf-dprive-poc
Proof of Concept stuff for IETF standards work

There is a PDF with the explanation, for presentions:  "DoT POC overview.pdf"
(This explains the intended architecture of same-resolver discovery/upgrade, rather than explaining the PoC itself.)

Very high level elevator pitch:
This is stuff to enable a client host, to upgrade its DNS resolver connection from ordinary DNS (unencrypted) to DNS-over-TLS.
There is no standard for any of the needed bits, so I have proposed ways to do this.
The proof-of-concept is to show it can be done and works.

The PoC is primarily intended to create a local "stub", which is actually a listener on 127.0.0.2, that is a forwarder
The "stub" is configured to use whatever resolver is discovered.
It is designed to do the discovery of an upgraded resolver, which is publishing the necessary information, and configure itself to do DoT to the resolver.

The PoC scripts are also meant to be run on the resolver(s) (plus optionally forwarders), to enable this discovery.

Example code is in perl, to implement things via config changes on "knot-resolver" and "bind".
Long term expectation is various implementations will build the logic/functionality into their respective components
(Not stuff I'll be working on directly, and lots of implementations hopefully will do this.)

Requirements:
 - perl
 - (probably some perl libraries)
 - bind (recent build) and bind-tools (especially delv)
 - knot-resolver
 - gnu tls
 - openssl tools

NB: OS notes:
- Packages for centos7 don't have the necessary version of gnu tls, so I used centos8 (we use centos).
- knot-resolver was challenging to build (at all), and some plug-ins wouldn't work after building on centos8.
- installation and execution for knot-resolver was also very problematic; I had to use LD_LIBRARY_PATH stuff. Sorry.
- As a result of these limitations, all of the DoH stuff is commented out or not created.
- DoT works just fine. Using tcpdump confirms that no unencrypted traffic travels between the stub and resolver.
- The logic currently bypasses any upgraded forwarder(s); this may or may not be a desired result, or could be a selectable option


The main part of the PoC, "upgrade-proxy", is a big perl script

The upgrade-proxy script is doing stuff that involves:
- creating self-signed TLS certs (using the openssl toolkit)
- doing DNSSEC key generation
- writing DNS zone files
- creating/modifying configs for open source DNS packages "bind" and "knot-resolver".
- obtaining and using DNSSEC trust anchors
- obtaining and using TLSA records
- current knot-resolver limitation:
  - does not do TLSA directly
  - must take snapshot by using "delv" to get TLSA, convert to a pin-sha256, install and use that
  - TLS certificate is validated with installed pin-sha256 value

The overall proof-of-concept design is meant to be run as a bunch of instances of bind and knot-resolver,
running on a number of loopback sub-interfaces which have been set up specifically for this purpose.

The main loopback interface needs to have its netmask changed to 255.255.255.255, so it will be 127.0.0.1/32.
All of the other loopback subinterfaces have IP addresses assigned sequentially, e.g. 127.0.0.2/32, etc.
(IPv6, if it is used, would get similar treatment using /128 sub-interface addressing.)

The other scripts here do the following things:

- setup-loopbacks
  - create sub-interfaces with expected names and addresses, requires the netmask change to be done first
- build-scenarios
  - uses "scenarios.txt" to create one out of a collection of scenarios
  - each scenario is a set of servers
    - these servers are assigned single loopback interfaces and addresses on which they listen
    - each server is a single stub, forwarder, or resolver
    - stubs and forwarders have upstream server(s) configured
    - each server config has "knot-resolver" config file, and possibly a "bind" config file (i.e. resolvers and forwarders)
    - these config files are generated, and form the base config of the respective servers
    - some of the servers are upgraded as part of the scenario
    - each scenario is intended to run those servers to demonstrate how it all works within that scenario
  - each scenario has a topology relating the servers, rooted at a single "stub"
  - each scenario has rules for which servers get upgraded
  - upgrading needs to be done in the correct order, basically right-to-left; this is also specified in the "scenarios.txt" file
  - each upgrade needs to be done while the other servers are running, and then the upgraded server is started.
    - doing this right-to-left avoids having to re-start servers
  - build-scenarios will upgrade and start the servers in the correct order, for the specified scenario number.
    - the upgrading is done via the "upgrade-proxy" script
    - running the servers is done either by calling named (with the required parameters), or the poc-run-kresd script (with parameters)
- poc-run-kresd
  - script for "daemon"-type launch of knot-resolver daemon, handles PID stuff and reaping logic.
  - knot-resolver doesn't do this itself, while bind does do this correctly out-of-the-box.
  - supports "start" and "stop" as well as selection of scenario/server/config-file
