---
title: "Openbsd unbound nsd"
date: 2023-01-11T11:42:19+09:00
draft: false
---

## Introduction

The default installation of OpenBSD comes with both unbound(8) and nsd(8); unbound is a validating, recursive, and caching DNS resolver that provides DNSSEC validation, while nsd is an authoritative name server that holds DNS records. The combination of the two running locally, means that name server lookups (i.e., requests to resolve domain names into IP addresses and vice versa) can be handled locally without being sent upstream to your ISP or another public name server such as Google. This almost completely prevents snooping or tampering such as DNS cache poisoning or spoofing attacks. Both programs have a small memory footprint, offer a secure environment to provide lightning quick retrieval of both forward and reverse DNS requests, and are exceedingly simple to setup. This article will detail the steps to configure both unbound and nsd on your OpenBSD box.

## Start Unbound

First, use rc to enable and start unbound:

~~~
# rcctl enable unbound
# rcctl start unbound
unbound(ok)
~~~

Before testing that it's working, we need to assign it as our primary nameserver in /etc/resolv.conf by either commenting out or removing the existing nameserver, and adding unbound, which is running on localhost:

~~~
nameserver 127.0.0.1
~~~

Test that it's working with a dig(1) DNS lookup request:

~~~
% dig bsdbox.org

; <<>> DiG 9.4.2-P2 <<>> bsdbox.org
;; global options:  printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57451
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;bsdbox.org.                    IN      A

;; ANSWER SECTION:
bsdbox.org.             3600    IN      A       101.161.32.217

;; Query time: 119 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Apr  4 23:16:09 2018
;; MSG SIZE  rcvd: 44
~~~

The SERVER: 127.0.0.1#53 shows that requests are being served locally by unbound on port 53. Given that unbound is a caching resolver, another dig request should show a faster query time:

~~~
% dig bsdbox.org

; <<>> DiG 9.4.2-P2 <<>> bsdbox.org
;; global options:  printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20855
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;bsdbox.org.                    IN      A

;; ANSWER SECTION:
bsdbox.org.             3595    IN      A       101.161.32.217

;; Query time: 2 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Apr  4 23:16:14 2018
;; MSG SIZE  rcvd: 44
~~~

From 119 down to 2 msec, so all appears to be operating as expected.

## Configure DNSSEC

DNSSEC validation is enabled by obtaining an initial trust anchor. This is achieved by downloading the KSK (Root Key Signing Key) with:

~~~
# unbound-anchor -a "/var/unbound/db/root.key"
~~~

And adding it under the server: directive in unbound's configuration file /var/unbound/etc/unbound.conf; while there, activate QNAME minimisation to improve DNS privacy by uncommenting the corresponding line shown below:

~~~
server:
        root-hints: "/var/unbound/db/root.hints"
        auto-trust-anchor-file: "/var/unbound/db/root.key"
        qname-minimisation: yes
~~~

The KSK initiates a chain of trust akin to the Public Key Infrastructure by essentially co-signing DNS entries to verify identities (i.e., confirming that our URI destinations are who they say they are). Therefore, it is essential that the key is authenticated at the time of anchoring and, thereafter, kept current. Apart from the key, we also need to know all the primary root DNS servers; we can achieve this by either downloading from Internic the root-hints file containing the definitive list of all primary root DNS servers, or by using the hardcoded list stored within unbound. The former ensures we're using the most up-to-date servers, but the latter is perfectly viable. If you choose the latter, remove or comment out the line above that begins with root-hints, but should you want the former, download the file with:

~~~
ftp -S do -o /var/unbound/db/root.hints https://www.internic.net/domain/named.root
~~~

Once complete, check configuration is syntactically correct before restarting, then confirm DNSSEC validation is operational.

~~~
# unbound-checkconf
unbound-checkconf: no errors in /var/unbound/etc/unbound.conf
# rcctl restart unbound
# dig com. SOA +dnssec

; <<>> DiG 9.4.2-P2 <<>> com. SOA +dnssec
;; global options: printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31120
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 14, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 4096
;; QUESTION SECTION:
;com. IN SOA

;; ANSWER SECTION:
com. 900 IN SOA a.gtld-servers.net. nstld.verisign-grs.com. 1523108041 1800
900 604800 86400
com. 900 IN RRSIG SOA 8 1 900 20180414133401 20180407122401 46967 com.
EPhpNR1CLIzYifJ6f5y3F8Lkg48tBndHptGkeBQOuJyMQUQgRwOjq5yl
V8XFu8kTIhj1c0tH+ZbeP29iclRBfbV2F3eFUXlkjHQqI0hrFgL3dBod
7bLe6HCwn0acdDbqlkLlGRJe58peR3lNMr8NpaJ+Y2KDU754tzYSnmZz AV4=

;; AUTHORITY SECTION:
com. 172800 IN NS j.gtld-servers.net.
com. 172800 IN NS m.gtld-servers.net.
com. 172800 IN NS h.gtld-servers.net.
com. 172800 IN NS l.gtld-servers.net.
com. 172800 IN NS g.gtld-servers.net.
com. 172800 IN NS d.gtld-servers.net.
com. 172800 IN NS i.gtld-servers.net.
com. 172800 IN NS f.gtld-servers.net.
com. 172800 IN NS e.gtld-servers.net.
com. 172800 IN NS k.gtld-servers.net.
com. 172800 IN NS b.gtld-servers.net.
com. 172800 IN NS a.gtld-servers.net.
com. 172800 IN NS c.gtld-servers.net.
com. 172800 IN RRSIG NS 8 1 172800 20180414044601 20180407033601 46967 com.
gURGiJt0j/b19Zi27P4QwYpxzswL0q2EAG7L9EzYgEVOJm6qZbCyR6RQ
61OiOkPCCVhpZyD/rbxKuEGHACJaXZ5TaWUbrL/AUDNC1UtiXAVgNyG1
iqNHf1+qAS/f5EQcKDvEJA1ISeKYExENG9E3GWbQ3pRcgMWLK9WmTdgc y3o=

;; Query time: 87 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sat Apr 7 23:34:18 2018
;; MSG SIZE rcvd: 637
~~~

From the ad flag, we can see that DNSSEC is operational but not that responses are valid; to verify that correct responses are being returned, we can use the DNSSEC validation test from https://verteiltesysteme.net by checking the following signed domain, which should return an A record:

~~~
# dig sigok.verteiltesysteme.net @127.0.0.1

; <<>> DiG 9.4.2-P2 <<>> sigok.verteiltesysteme.net @127.0.0.1
;; global options: printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49205
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 4

;; QUESTION SECTION:
;sigok.verteiltesysteme.net. IN A

;; ANSWER SECTION:
sigok.verteiltesysteme.net. 60 IN A 134.91.78.139

;; AUTHORITY SECTION:
verteiltesysteme.net. 1340 IN NS ns2.verteiltesysteme.net.
verteiltesysteme.net. 1340 IN NS ns1.verteiltesysteme.net.

;; ADDITIONAL SECTION:
ns1.verteiltesysteme.net. 1340 IN A 134.91.78.139
ns1.verteiltesysteme.net. 1340 IN AAAA 2001:638:501:8efc::139
ns2.verteiltesysteme.net. 1340 IN A 134.91.78.141
ns2.verteiltesysteme.net. 1340 IN AAAA 2001:638:501:8efc::141
Confirmed. And the following domain should return a SERVFAIL:

# dig sigfail.verteiltesysteme.net @127.0.0.1

; <<>> DiG 9.4.2-P2 <<>> sigfail.verteiltesysteme.net @127.0.0.1
;; global options: printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 62744
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;sigfail.verteiltesysteme.net. IN A

;; Query time: 148 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sat Apr 7 21:26:28 2018
;; MSG SIZE rcvd: 46
~~~

Confirmed! Now that DNSSEC validation has been activated, restart the daemon with rcctl restart unbound, and check the log for any errors:

~~~
# tail -20 /var/log/daemon
Apr 3 23:31:55 nsx unbound: [93843:0] info: service stopped (unbound
1.6.6).
Apr 3 23:31:55 nsx unbound: [93843:0] info: server stats for thread 0: 20
queries, 5 answers from cache, 15 recursions, 0 prefetch, 0 rejected by ip
ratelimiting
Apr 3 23:31:55 nsx unbound: [93843:0] info: server stats for thread 0:
requestlist max 0 avg 0 exceeded 0 jostled 0
Apr 3 23:31:55 nsx unbound: [93843:0] info: average recursion processing
time 0.529802 sec
Apr 3 23:31:55 nsx unbound: [93843:0] info: histogram of recursion
processing times
Apr 3 23:31:55 nsx unbound: [93843:0] info: [25%]=0.013824
median[50%]=0.057344 [75%]=0.821608
Apr 3 23:31:55 nsx unbound: [93843:0] info: lower(secs) upper(secs)
recursions
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.004096 0.008192 1
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.008192 0.016384 4
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.016384 0.032768 1
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.032768 0.065536 2
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.065536 0.131072 1
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.131072 0.262144 1
Apr 3 23:31:55 nsx unbound: [93843:0] info: 0.524288 1.000000 2
Apr 3 23:31:55 nsx unbound: [93843:0] info: 1.000000 2.000000 1
Apr 3 23:31:55 nsx unbound: [93843:0] info: 2.000000 4.000000 2
Apr 3 23:31:56 nsx unbound: [93111:0] notice: init module 0: validator
Apr 3 23:31:56 nsx unbound: [93111:0] notice: init module 1: iterator
Apr 3 23:31:56 nsx unbound: [93111:0] info: start of service (unbound
1.6.6).
~~~

Everything is operating as expected. We can check a domain's DNSSEC signature like so:

~~~
# unbound-host -C /var/unbound/etc/unbound.conf -v sigok.verteiltesysteme.net
sigok.verteiltesysteme.net has address 134.91.78.139 (secure)
sigok.verteiltesysteme.net has IPv6 address 2001:638:501:8efc::139 (secure)
sigok.verteiltesysteme.net has no mail handler record (secure)
For comparison, an unsigned domain without DNSSEC:

# unbound-host -C /var/unbound/etc/unbound.conf -v sigfail.verteiltesysteme.net
sigfail.verteiltesysteme.net has address 134.91.78.139 (BOGUS (security
failure))
validation failure <sigfail.verteiltesysteme.net. A IN>: signature crypto
failed from 2001:638:501:8efc::139
sigfail.verteiltesysteme.net has IPv6 address 2001:638:501:8efc::139 (BOGUS
(security failure))
validation failure <sigfail.verteiltesysteme.net. AAAA IN>: signature crypto
failed from 2001:638:501:8efc::139
~~~

Now that unbound is up and running, serving our DNS requests locally, we can move onto nsd.

## NSD Configuration

Unlike unbound, which resolves our outgoing queries for domain name resolution, nsd is an authoritative nameserver, which holds our own DNS records, and provides responses to incoming queries for names in our own zone. Because of this, it's highly recommended that you configure both a primary and a secondary nameserver so that in the event one is unreachable, requests of your zone are still received; this article only covers setting up the primary (master) server—configuring the slave will be left as an exercise to the reader.

First, edit the configuration file at /var/nsd/etc/nsd.conf to add the master zone record as shown (substituting with your server's particulars):

~~~
server:
        hide-version: yes
        ip-address: 199.22.33.44    # ipaddr of server

remote-control:
        control-enable: yes

## master zone example
zone:
        name: "nsx.bsdbox.org"
        zonefile: "nsx.bsdbox.org.zone"
~~~

Next, edit the specified zone file at /var/nsd/zones/nsx.bsdbox.org.zone:

~~~
$ORIGIN nsx.bsdbox.org.
$TTL 4h

@        IN  SOA  a.ns.nsx.bsdbox.org. hostmaster.nsx.bsdbox.org. (
                  2018040502      ; serial
                  1h              ; refresh
                  30m             ; retry
                  7d              ; expire
                  1h )            ; negative
         IN  NS   a.ns.nsx.bsdbox.org.
         IN  NS   b.ns.nsx.bsdbox.org.

         IN  MX   0 mail.nsx.bsdbox.org.

a.ns     IN  A    199.222.33.44
b.ns     IN  A    199.222.33.44
mail     IN  A    199.222.33.44
~~~

Now check the configuration before starting the service:

~~~
# nsd-checkconf /var/nsd/etc/nsd.conf
# nsd-checkzone nsx.bsdbox.org /var/nsd/zones/nsx.bsdbox.org.zone
zone nsx.bsdbox.org is ok
# rcctl enable nsd && rcctl start nsd
nsd(ok)
nsd(ok)
~~~

Check the log for errors:

~~~
# tail -5 /var/log/daemon
Apr 4 00:33:43 nsx last message repeated 2 times
Apr 4 00:34:20 nsx nsd[59379]: signal received, shutting down...
Apr 4 00:34:21 nsx nsd[24345]: nsd starting (NSD 4.1.17)
Apr 4 00:34:22 nsx nsd[8356]: zone nsx.bsdbox.org read with success
Apr 4 00:34:22 nsx nsd[8356]: nsd started (NSD 4.1.17), pid 79289
~~~

The log indicates a successful, error-free start, which means we have both a local DNS server to resolve our outgoing queries—freeing us from the prying eyes of Google—and an authoritative nameserver serving requests for our own records.

## DNSCrypt

If you have DNSCrypt setup, and want to forward outgoing DNS queries to a DNScrypt nameserver rather than resolving them locally with unbound, replace the existing unbound.conf with the following:

~~~
server:
        interface: 127.0.0.1
        interface: ::1
        access-control: 0.0.0.0/0 refuse
        access-control: 127.0.0.0/8 allow
        access-control: ::0/0 refuse
        access-control: ::1 allow
        do-not-query-localhost: no
        hide-identity: yes
        hide-version: yes
        root-hints: "/var/unbound/db/root.hints"
        auto-trust-anchor-file: "/var/unbound/db/root.key"
        qname-minimisation: yes

remote-control:
        control-enable: yes
        control-use-cert: no
        control-interface: /var/run/unbound.sock

forward-zone:
        name: "."
        forward-addr: 127.0.0.1@40 # primary DNScrypt server
        forward-addr: 127.0.0.1@41 # failover DNScrypt server
~~~

Check unbound configuration and, if no complaints, restart the service:

~~~
# unbound-checkconf
unbound-checkconf: no errors in /var/unbound/etc/unbound.conf
# rcctl restart unbound
unbound(ok)
unbound(ok)
~~~

