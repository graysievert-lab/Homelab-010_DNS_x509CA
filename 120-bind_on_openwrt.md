# Installing BIND with dynamic DNS updates on OpenWRT
This guide has translation to [Russian language](https://habr.com/ru/articles/826826/).

Plan:
1. Install BIND
2. Disable dnsmasq's DNS forwarder leaving the DHCP server operational
3. Perform basic configuration 
4. Enable and test dynamic zones updates
5. Automate automatic updates of records for hosts receiving leases from DHCP

Assumptions:
- `.lan` is the suffix for hostnames on the local network, 
- local network CIDR is `192.168.1.0/24` 

Versions:
- OpenWrt 23.05.3 arm64
- BIND 9.18.24


Disclaimer

By following the steps below, you may break name resolution in your network, degrade your router's flash, or even disconnect yourself from the internet. I highly encourage you to try first on the [emulated device](https://github.com/graysievert/Homelab-010_DNS_x509CA/blob/master/110-openwrt_on_proxmox.md).


## Installation

Before start breaking our router let's check what we already have
```bash
$ netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address    Foreign Address  State    PID/Program name
tcp        0      0 0.0.0.0:22       0.0.0.0:*        LISTEN   1637/dropbear
tcp        0      0 0.0.0.0:80       0.0.0.0:*        LISTEN   2187/uhttpd
tcp        0      0 192.168.1.1:53   0.0.0.0:*        LISTEN   2510/dnsmasq
tcp        0      0 10.1.2.144:53    0.0.0.0:*        LISTEN   2510/dnsmasq
tcp        0      0 127.0.0.1:53     0.0.0.0:*        LISTEN   2510/dnsmasq
tcp        0      0 0.0.0.0:443      0.0.0.0:*        LISTEN   2187/uhttpd
udp        0      0 127.0.0.1:53     0.0.0.0:*                 2510/dnsmasq
udp        0      0 10.1.2.144:53    0.0.0.0:*                 2510/dnsmasq
udp        0      0 192.168.1.1:53   0.0.0.0:*                 2510/dnsmasq
udp        0      0 0.0.0.0:67       0.0.0.0:*                 2510/dnsmasq
```
As one can see both `53` (DNS) and `67` (DHCP) ports are being served with dnsmasq 

Installation is pretty straightforward:
```bash
$ opkg update
$ opkg install bind-server bind-tools bind-client
```
Let's disable dnsmasq's DNS, and also we need to instruct dnsmasq to announce `192.168.1.1` as domain server via DHCP
```bash
$ uci set dhcp.@dnsmasq[0].port=0
$ uci set dhcp.@dnsmasq[0].domain='lan'
$ uci add_list dhcp.@dnsmasq[0].dhcp_option='6,192.168.1.1'
$ uci commit dhcp
$ /etc/init.d/dnsmasq restart
$ /etc/init.d/named restart
```
The active services now should look like
```bash
$ netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address   Foreign Address   State   PID/Program name
tcp        0      0 0.0.0.0:22      0.0.0.0:*         LISTEN  1637/dropbear
tcp        0      0 0.0.0.0:80      0.0.0.0:*         LISTEN  2187/uhttpd
tcp        0      0 127.0.0.1:953   0.0.0.0:*         LISTEN  8511/named
tcp        0      0 127.0.0.1:953   0.0.0.0:*         LISTEN  8511/named
tcp        0      0 0.0.0.0:443     0.0.0.0:*         LISTEN  2187/uhttpd
udp        0      0 192.168.1.1:53  0.0.0.0:*                 8511/named
udp        0      0 192.168.1.1:53  0.0.0.0:*                 8511/named
udp        0      0 10.1.2.144:53   0.0.0.0:*                 8511/named
udp        0      0 10.1.2.144:53   0.0.0.0:*                 8511/named
udp        0      0 127.0.0.1:53    0.0.0.0:*                 8511/named
udp        0      0 127.0.0.1:53    0.0.0.0:*                 8511/named
udp        0      0 0.0.0.0:67      0.0.0.0:*                 8767/dnsmasq
```
dnsmasq listens on DHCP port and named (BIND) listens on 53/953 ports. 

Let's check that we have access to DNS 
```bash
$ dig +dnssec +multi . DNSKEY
; <<>> DiG 9.18.24 <<>> +dnssec +multi . DNSKEY
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19567
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags: do; udp: 1232
; COOKIE: a398f023571fde5701000000667c7bbbd959a05085315b53 (good)
;; QUESTION SECTION:
;.                      IN DNSKEY
;; ANSWER SECTION:
.                       171554 IN DNSKEY 256 3 8 (
                                AwEAAZBALoOFImwcJJg9Iu7Vy7ZyLjhtXfvO1c9k4vHj
                                Opf9i7U1kKtrBvhnwsOni1sb50gkUayRtMDTUQqvljMM
                                f4bpkyEtcE5evCzhHbFLq1coL5QOix3mfJm++FvIMaAt
                                52nOvAdqR/luuI11bA1AmSCIJKAUx147DcfOHYKg3as+
                                dznn3Iah4cWBMVzDe7PPsFS1AO6gU8EpmiRJ9VMNA09f
                                OyDuq9+d6sw8UUnJRMAFAuPLhUFjUAOuWOw74BC9lOtM
                                QpbLMz8pX0CDKdOXDHjyj61nxSSWxPdUjeoxI17lQTpS
                                PRtqRHFn5Fgj2e+9BVwhhWGDQN8kUVSJHZtQiI0=
                                ) ; ZSK; alg = RSASHA256 ; key id = 5613
.                       171554 IN DNSKEY 256 3 8 (
                                AwEAAdSiy6sslYrcZSGcuMEK4DtE8DZZY1A08kAsviAD
                                49tocYO5m37AvIOyzeiKBWuPuJ4m9u5HonCM/ntxklZK
                                YFyMftv8XoRwbiXdpSjfdpNHiMYTTV2oDUNMjdLFnF6H
                                YSY48xrPbevQOYbAFGHpxqcXAQT0+BaBiAx3Ls6lXBQ3
                                /hSVOprvDWJCQiI2OT+9+saKLddSIX6DwTVy0S5T4YY4
                                EGg5R3c/eKUb2/8XgKWUzlOIZsVAZZUSTKW0tX54ccAA
                                LO7Grvsx/NW62jc1xv6wWAXocOEVgB7+4Lzb7q9p5o30
                                +sYoGpOsKgFvMSy4oCZTQMQx2Sjd/NG2bMMw6nM=
                                ) ; ZSK; alg = RSASHA256 ; key id = 20038
.                       171554 IN DNSKEY 257 3 8 (
                                AwEAAaz/tAm8yTn4Mfeh5eyI96WSVexTBAvkMgJzkKTO
                                iW1vkIbzxeF3+/4RgWOq7HrxRixHlFlExOLAJr5emLvN
                                7SWXgnLh4+B5xQlNVz8Og8kvArMtNROxVQuCaSnIDdD5
                                LKyWbRd2n9WGe2R8PzgCmr3EgVLrjyBxWezF0jLHwVN8
                                efS3rCj/EWgvIWgb9tarpVUDK/b58Da+sqqls3eNbuv7
                                pr+eoZG+SrDK6nWeL3c6H5Apxz7LjVc1uTIdsIXxuOLY
                                A4/ilBmSVIzuDWfdRUfhHdY6+cn8HFRm+2hM8AnXGXws
                                9555KrUB5qihylGa8subX2Nn6UwNR1AkUTV74bU=
                                ) ; KSK; alg = RSASHA256 ; key id = 20326
.                       171554 IN RRSIG DNSKEY 8 0 172800 (
                                20240711000000 20240620000000 20326 .
                                k7Tz3FFlPySd/LF69we2WyDwnqf+JTTpJ3sriFGLkq26
                                MGBD/fioXO4xqcCrnWVF50nKs8CaEQpdI9N0N2rW3fZh
                                9sVryGEvPiNnxfv8JC9MiMlt5pnVWYyOzDWpt9OAznmv
                                JVvqhZIi19MvmkEj+S/WQCuJwZUx+0r1Nv8mBrN0dbms
                                LpH3sjgs8pw8SSL4QCLFlJzmqomt1ncM5ocoWqvOU7Hb
                                Xgt40Gg0ZiZFqs9IebA62pbu5GAVzJEMoANUqxIo3lAg
                                2JIEWTpo/+hF3QpaB/SFJ0obrJMi4OULOfY2DCx1jjlq
                                C4qaiS7c/IaGux2bMwQV1zfRDpu4AA5eSw== )
;; Query time: 9 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Wed Jun 26 20:36:11 UTC 2024
;; MSG SIZE  rcvd: 1169
```
Looks good and we are ready for configuring our DNS server.

## Default configuration

Right out of the box configuration folder /etc/bind looks like
```bash
$ ls -lah /etc/bind 
drwxr-xr-x  2 root root  3.4K Jun 26 20:15 .
drwxr-xr-x  1 root root  3.4K Jun 26 20:15 ..
-rw-r--r--  1 root root  3.8K Feb 16 18:24 bind.keys
-rw-r--r--  1 root root   237 Feb 16 18:24 db.0
-rw-r--r--  1 root root   271 Feb 16 18:24 db.127
-rw-r--r--  1 root root   237 Feb 16 18:24 db.255
-rw-r--r--  1 root root   237 Feb 16 18:24 db.empty
-rw-r--r--  1 root root   256 Feb 16 18:24 db.local
-rw-r--r--  1 root root  3.1K Feb 16 18:24 db.root
-rw-r--r--  1 root root   281 Jun 26 20:15 named-rndc.conf
-rw-r--r--  1 root root   982 Feb 16 18:24 named.conf
-rw-r--r--  1 root root   225 Jun 26 20:15 rndc.conf
```
Where
- `bind.keys` - trust anchors overrides for the DNS root zone (".") Curious readers may compare its contents with the answer we got from dig earlier
- `db.root` - information on root name servers needed to initialize our cache
- `db.0`, `db.255`, `db.empty` - reverse lookup zones for broadcast
- `db.local` - forward lookup zone for localhost
- `db.127` - reverse lookup zone for the loopback 
- `named-rndc.conf` - contains key and controls permitting rndc utility to manage bind
- `rndc.conf` - key and settings for rndc utility
- `named.conf` - main configuration

Out of the box `/etc/bind/named.conf` should look similar to this:
```JSON
// base named.conf file
// Recommended that you always maintain a change log in this file as shown here
// options clause defining the server-wide properties
options {
        // all relative paths use this directory as a base
        directory "/tmp";

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.
        // forwarders {
        //      0.0.0.0;
        // };

        auth-nxdomain no;    # conform to RFC1035

        // this ensures that any reverse map for private IPs
        // not defined in a zone file will *not* be passed 
        // to the public network
        empty-zones-enable yes;
};

include "/etc/bind/named-rndc.conf";
include "/tmp/bind/named.conf.local";

// prime the server with knowledge of the root servers
zone "." {
        type hint;
        file "/etc/bind/db.root";
};

// Provide forward mapping zone for localhost (optional)
zone "localhost" {
        type primary;
        file "/etc/bind/db.local";
};

// Provide reverse mapping zone for the loopback address 127.0.0.1
// zone "0.0.127.in-addr.arpa" 
zone "127.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.127";
};

zone "0.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.0";
};

zone "255.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.255";
};
```


## Adding zones

First, let's add a forward lookup zone file for our `lan.` local domain.
`/etc/bind/db.lan`: 
```
; forward zone file for lan.
$ORIGIN .
$TTL 0  ; 0 seconds
lan                     IN SOA  ns1.lan. root.lan. (
                                1719490275 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN lan.
ns1                     A       192.168.1.1
openwrt                 A       192.168.1.1
router                  CNAME   openwrt
acme                    CNAME   openwrt
```
and reverse lookup zone file for our `192.168.1.0/24` subnet
`/etc/bind/db.1.168.192`: 
```
; reverse zone file for lan.
$ORIGIN .
$TTL 0  ; 0 seconds
1.168.192.in-addr.arpa  IN SOA  ns1.lan. root.lan. (
                                1719490269 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN 1.168.192.in-addr.arpa.
1                       PTR     ns1.lan.
                        PTR     openwrt.lan.
```
Note:  For a quick recap on syntax [visit](https://datatracker.ietf.org/doc/html/rfc1035#section-5), but here is a summary applicable to the files above:
- `;` starts a comment. The rest of a line after it is ignored.
- `()` are used just to cut long records into several lines of text.
- Any domain name that does not end with the dot is appended with the current value in `$ORIGIN`. (e.g. CNAME record for `acme` is expanded to `acme.lan.` as the current value of `$ORIGIN` is `lan.`)
- You can omit to mention `IN` class in records after it is mentioned in `SOA` record.
- You can omit to mention TTL in records  as it will be set to the value currently defined in `$TTL` variable
- In `SOA` record `ns1.lan.` is the DNS server name that serves this zone, `root.lan.` translates to `root@lan` - the email address of an administrator of this zone (you may use anything you want - the DNS police avoid private zones).
- If in the record the very first field is empty (it is called `OWNER`) then it is assumed to be equal to the one on the previous line. 
- Names in `in-addr.arpa.` domain, corresponding to IP addresses are written in reverse order, e.g. ip `10.1.2.3` corresponds to `3.2.1.10.in-addr.arpa.`

Expansion rules above allow to really shorten records from `1.1.168.192.in-addr.arpa. 900 IN PTR openwrt.lan.` to just `PTR openwrt.lan.` 

Let's check that zone files have proper syntax
```bash
$ named-checkzone lan /etc/bind/db.lan 
zone lan/IN: loaded serial 1719490275
OK

$ named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.1.168.192 
zone 1.168.192.in-addr.arpa/IN: loaded serial 1719490269
OK
```

With the zone files verified we can add corresponding zones to the bottom of /etc/bind/named.conf
```bash
cat <<EOF>> /etc/bind/named.conf
zone "lan" {
        type primary;
        file "/etc/bind/db.lan";
};
zone "1.168.192.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.1.168.192";
};
EOF
```
Let's check that our config is properly formatted
```bash
$ named-checkconf -pzx
```

Let's reload the configuration and double-check our zones
```bash
$ rndc reload

$ rndc zonestatus lan 
name: lan
type: primary
files: /etc/bind/db.lan
serial: 1719490275
nodes: 5
last loaded: Thu, 27 Jun 2024 13:01:34 GMT
secure: no
dynamic: no
reconfigurable via modzone: no

$ rndc zonestatus 1.168.192.in-addr.arpa 
name: 1.168.192.in-addr.arpa
type: primary
files: /etc/bind/db.1.168.192
serial: 1719490269
nodes: 2
last loaded: Thu, 27 Jun 2024 13:13:03 GMT
secure: no
dynamic: no
reconfigurable via modzone: no
```
Looks fine. Let's test name resolution itself
```bash
$ nslookup acme.lan && nslookup 192.168.1.1 
Server:         127.0.0.1
Address:        127.0.0.1#53
acme.lan        canonical name = openwrt.lan.
Name:   openwrt.lan
Address: 192.168.1.1
1.1.168.192.in-addr.arpa        name = ns1.lan.
1.1.168.192.in-addr.arpa        name = openwrt.lan.
```

## Forwarders

Now you have a direct connection to DNS, which means anyone who can tap into your traffic would know what sites you're visiting. Let's try to fix that with forwarders.
Disclaimer: Concealing DNS traffic with forwarders is actually a selection between being watched by your ISP with regional Bigbrother, or by Dr.Evil with Illuminati. Of course, if forwarded queries are leaving your network over port 53 unencrypted, the ISP still knows :)
Starting from BIND 9.19.10 one may configure the use of TLS  for forwarded queries, alas only 9.18.24 in the OpenWRT package database at the moment.
To achieve DNS-over-TLS (DOT) we may use  `stubby` as a DNS-over-TLS proxy:
```bash
$ opkg install ca-certificates
$ opkg install stubby
```
To find out what port `stubby` is listening run
```bash
$ uci get stubby.global.listen_address 
127.0.0.1@5453 0::1@5453
```

Let's add `stubby` as a forwarder into `/etc/bind/named.conf`. This will redirect BIND's DNS traffic to `stubby` which by default redirects everything to CloudFlare.
```JSON
options {
...
   forward only;
   forwarders {
      127.0.0.1 port 5453; 
   }; 
...
};
```
check and reload
```bash
$ named-checkconf
$ rndc reload
$ rndc flush
```

If after enabling forwarders name resolution does not seem working or you are unhappy with CloudFlare - here are a few hints that helped me.
- `opkg install ca-certificates` helped me with error `TLS - *Failure* - (20) "unable to get local issuer certificate"` 
- Check config for stubby `/etc/config/stubby`. The configuration of DoT resolvers is pretty self-explanatory.
- Uncomment `option log_level '7'`  in `/etc/config/stubby` will enable debug output in syslog. Just run `logread -f` in another ssh session and reload service `/etc/init.d/stubby reload `



## Dynamic zone updates

At this moment our bind server is fully static and new records in zone files won't appear by themselves. We can manually edit our forward and backward zone files and reload configuration, but there is a better way - to use nsupdate utility or any other tool for cryptographically signed updates to DNS.

### Creating TSIG key

Note: Many examples on the internet use `dnssec-keygen` for TSIG key generation. It won't work with the current bind version as the feature to generate HMAC algorithms for use as TSIG keys via `dnssec-keygen` was removed in BIND 9.13.0.  We'll use `tsig-keygen` instead.

As the `tsig-keygen` output is already formatted for inclusion into a config file we just need to:
```bash
$ tsig-keygen | tee /etc/bind/keys.conf
key "tsig-key" {
        algorithm hmac-sha256;
        secret "HAyLN66//YxVF2lrZ6kSZK4TZEpV7WMvzYnNUQ0BvEo=";
};
```

Then append `/etc/bind/named.conf` with the `include "/etc/bind/keys.conf"`:
```bash
cat<<EOF>> /etc/bind/named.conf
include "/etc/bind/keys.conf";
EOF
```


### Adjusting zone configuration

Now add `allow-update` statement to the zone configs, which would allow any changes to the zone.
```JSON
zone "lan" {
        type primary;
        file "/etc/bind/db.lan";
        allow-update {
          key tsig-key;
        };
};
zone "1.168.192.in-addr.arpa" {
        type primary;
        file "/etc/bind/db.1.168.192";
        allow-update {
          key tsig-key;
        };
};
```
Note: if that is too broad permissions, then one may use proper `update-policy { update_policy_rule [...] };` stanza instead.

Check and reload configuration:
```bash
$ named-checkconf
$ rndc reload
```


### File access permissions

At the moment all files in `/etc/bind` directory have `-rw-r--r--  root:root` permissions. This means `named` launched as `bind:bind` wont be able to create or modify files. 
Let's change the directory owner to `bind` and protect files there from prying eyes of other system users:
```bash
$ chown -R bind:bind /etc/bind
$ chmod 600 -R /etc/bind/*
```




## Changing zone file records via nsupdate

Now let's add a few records into our zones.
Create `nsupdate.cmd` file with the following contents
```
server 127.0.0.1 53
zone lan.
update delete host2.lan.
update add host2.lan. 900 A 192.168.1.2
show
send
zone 1.168.192.in-addr.arpa.
update delete 2.1.168.192.in-addr.arpa.
update add 2.1.168.192.in-addr.arpa. 900 PTR host2.lan.
show
send
```
Here we telling the nameserver to delete all records for `host2.lan` in the zone `lan.` if such records are present, and then add `A` record with `TTL 900`  for `host2.lan.` pointing to `192.168.1.2`. Then repeat the flow for PTR record in `1.168.192.in-addr.arpa.` reverse lookup zone.

Run `nsupdate` pointing it to the file with the key and the file with the contents above:
```bash
$ nsupdate -k /etc/bind/keys.conf nsupdate.cmd
Outgoing update query:
;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
;; ZONE SECTION:
;lan.                           IN      SOA
;; UPDATE SECTION:
host2.lan.              0       ANY     ANY
host2.lan.              900     IN      A       192.168.1.2
Outgoing update query:
;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
;; ZONE SECTION:
;1.168.192.in-addr.arpa.                IN      SOA
;; UPDATE SECTION:
2.1.168.192.in-addr.arpa. 0     ANY     ANY
2.1.168.192.in-addr.arpa. 900   IN      PTR     host2.lan.
```
Check
```bash
$ host 192.168.1.2 && host host2.lan 
2.1.168.192.in-addr.arpa domain name pointer host2.lan.
host2.lan has address 192.168.1.2
```
After this two new journal files should have appeared in `/etc/bind`:
```bash
$ ls /etc/bind/*jnl 
/etc/bind/db.1.168.192.jnl  /etc/bind/db.lan.jnl
```
DISCLAIMER: These writes will speed up the degradation of your router's flash.  Please read the [docs](https://bind9.readthedocs.io/en/v9.18.24/chapter6.html#the-journal-file) to assess how bad that may be in your particular case. I am just accepting dumps once per 15 min as a lesser evil until I find a way to store them in memory on `/tmp` at the price of the possibility of losing some data on power loss.

For now, let's just sync zones from memory to zone files and check what changed.
```bash
$ rndc sync

$ cat /etc/bind/db.lan 
$ORIGIN .
$TTL 0  ; 0 seconds
lan                     IN SOA  ns1.lan. root.lan. (
                                1719490277 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN lan.
acme                    CNAME   openwrt
host2                   A       192.168.1.2
ns1                     A       192.168.1.1
openwrt                 A       192.168.1.1
router                  CNAME   openwrt

$ cat /etc/bind/db.1.168.192 
$ORIGIN .
$TTL 0  ; 0 seconds
1.168.192.in-addr.arpa  IN SOA  ns1.lan. root.lan. (
                                1719490271 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN 1.168.192.in-addr.arpa.
1                       PTR     ns1.lan.
                        PTR     openwrt.lan.
2                       PTR     host2.lan.
```
As one may see `host2.lan.` records were added in both zones.


## Changing records from a host on lan

Use your favorite secrets transfer method to copy contents of `/etc/bind/keys.conf` to the chosen host.
Also, we need to generate `nsupdate.cmd` file with server pointing to `192.168.1.1`:
```
server 192.168.1.1 53
zone lan.
update delete host2.lan.
show
send
zone 1.168.192.in-addr.arpa.
update delete 2.1.168.192.in-addr.arpa.
show
send
```
Run `nsupdate`:
```bash
[rocky@test ~]$ nsupdate -k keys.conf nsupdate.cmd 
Outgoing update query:
;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
;; ZONE SECTION:
;lan.                           IN      SOA

;; UPDATE SECTION:
host2.lan.              0       ANY     ANY

Outgoing update query:
;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
;; ZONE SECTION:
;1.168.192.in-addr.arpa.                IN      SOA

;; UPDATE SECTION:
2.1.168.192.in-addr.arpa. 0     ANY     ANY
```
No errors, which means it should have worked.


## Changing records via Terraform

Overall, it should not matter what tool is used for zone updates - you just need a TSIG key name and key material. Let's try to update our zones via terraform:
Create `main.tf`:
```JSON
# Disclaimer: Storing secrets in plain text within Terraform's configuration
# and state files is strongly discouraged due to the inevitable security risks.
# It is crucial to familiarize yourself with techniques to avoid those.
terraform {
  required_providers {
    dns = {
      source = "hashicorp/dns"
      version = "3.4.1"
    }
  }
}
provider "dns" {
  update {
    server        = "192.168.1.1"
    key_name      = "tsig-key."
    key_algorithm = "hmac-sha256"
    key_secret    = "HAyLN66//YxVF2lrZ6kSZK4TZEpV7WMvzYnNUQ0BvEo="
  }
}
resource "dns_a_record_set" "host100" {
  zone = "lan."
  name = "host100"
  addresses = [
    "192.168.1.100"
  ]
  ttl = 900
}
resource "dns_ptr_record" "ptr_192_168_1_100" {
  zone = "1.168.192.in-addr.arpa."
  name = "100"
  ptr  = "host100.lan."
  ttl = 900
}
```
Please note the trailing dot `.` in the `key_name` value. The `hashicorp/dns` provider is unhappy when it is not formatted as FQDN. At the same time, the value of `name` in resource declarations must NOT be fully qualified.

Install `terraform`/`tofu`  and run
```bash
[rocky@test ~]$ terraform init
[rocky@test ~]$ terraform plan
[rocky@test ~]$ terraform apply
Terraform will perform the following actions:
  # dns_a_record_set.host100 will be created
  + resource "dns_a_record_set" "host100" {
      + addresses = [
          + "192.168.1.100",
        ]
      + id        = (known after apply)
      + name      = "host100"
      + ttl       = 900
      + zone      = "lan."
    }
  # dns_ptr_record.ptr_192_168_1_100 will be created
  + resource "dns_ptr_record" "ptr_192_168_1_100" {
      + id   = (known after apply)
      + name = "100"
      + ptr  = "host100.lan."
      + ttl  = 900
      + zone = "1.168.192.in-addr.arpa."
    }
Plan: 2 to add, 0 to change, 0 to destroy.
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
  Enter a value: yes
dns_ptr_record.ptr_192_168_1_100: Creating...
dns_a_record_set.host100: Creating...
dns_a_record_set.host100: Creation complete after 0s [id=host100.lan.]
dns_ptr_record.ptr_192_168_1_100: Creation complete after 0s [id=100.1.168.192.in-addr.arpa.]
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```
Let's go back to OpenWRT shell and check what we have in the zone files
```bash
$ rndc sync 
$ cat /etc/bind/db.lan 
$ORIGIN .
$TTL 0  ; 0 seconds
lan                     IN SOA  ns1.lan. root.lan. (
                                1719490287 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN lan.
acme                    CNAME   openwrt
host100                 A       192.168.1.100
ns1                     A       192.168.1.1
openwrt                 A       192.168.1.1
router                  CNAME   openwrt


$ cat /etc/bind/db.1.168.192 
$ORIGIN .
$TTL 0  ; 0 seconds
1.168.192.in-addr.arpa  IN SOA  ns1.lan. root.lan. (
                                1719490281 ; serial
                                43200      ; refresh (12 hours)
                                900        ; retry (15 minutes)
                                1814400    ; expire (3 weeks)
                                7200       ; minimum (2 hours)
                                )
$TTL 900        ; 15 minutes
                        NS      ns1.lan.
$ORIGIN 1.168.192.in-addr.arpa.
1                       PTR     ns1.lan.
                        PTR     openwrt.lan.
100                     PTR     host100.lan.
```
Looks legit.  Now let's remove records we've created
```bash
[rocky@test ~]$ terraform destroy
Terraform will perform the following actions:
  # dns_a_record_set.host100 will be destroyed
  - resource "dns_a_record_set" "host100" {
      - addresses = [
          - "192.168.1.100",
        ] -> null
      - id        = "host100.lan." -> null
      - name      = "host100" -> null
      - ttl       = 900 -> null
      - zone      = "lan." -> null
    }
  # dns_ptr_record.ptr_192_168_1_100 will be destroyed
  - resource "dns_ptr_record" "ptr_192_168_1_100" {
      - id   = "100.1.168.192.in-addr.arpa." -> null
      - name = "100" -> null
      - ptr  = "host100.lan." -> null
      - ttl  = 900 -> null
      - zone = "1.168.192.in-addr.arpa." -> null
    }
Plan: 0 to add, 0 to change, 2 to destroy.
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.
  Enter a value: yes
dns_ptr_record.ptr_192_168_1_100: Destroying... [id=100.1.168.192.in-addr.arpa.]
dns_a_record_set.host100: Destroying... [id=host100.lan.]
dns_a_record_set.host100: Destruction complete after 0s
dns_ptr_record.ptr_192_168_1_100: Destruction complete after 0s
Destroy complete! Resources: 2 destroyed.
```


## Automatic registration of resource records for hosts requesting dhcp leases

For now, we utilized remote but human-involved management of zones' contents. Let's add some automation for DHCP clients.
Openwrt has a mechanism called [hotplug](https://openwrt.org/docs/guide-user/base-system/hotplug), which allows running user scripts as a reaction to events in various services. We are interested in `dhcp` events coming from `dnsmasq`, and  in order to run our script we need to just put it into `/etc/hotplug.d/dhcp/` directory. The script placed there would be called from `/usr/lib/dnsmasq/dhcp-script.sh` which in its turn is being called by `dnsmasq`. 
Our script will be provided with DHCP lease-related data in variables: `MACADDR`, `IPADDR`, `HOSTNAME`, and `ACTION="add|remove|update"`  

CAVEAT: There is an ambiguity with the `HOSTNAME` variable in the case when a joining host does not provide its name to the DHCP server.

#### Ambiguity details for curious:

Let's create a test hotplug script `/etc/hotplug.d/dhcp/00-hello` 
```bash
logger "======================="
logger "I've been supplied with this number of arguments: ${#}"
logger "There they are: ${@}"
logger "Here're variables available to me:"
for var_passed in $(set); do
    logger "${var_passed}"
done
logger "======================="
```
Let's trigger a DHCP event from a host on lan without providing DHCP server with hostname
```bash
[rocky@test ~]$ hostname
test

[rocky@test ~]$ sudo dhclient -r eth0 -v
Listening on LPF/eth0/bc:24:11:2a:92:e3
Sending on   LPF/eth0/bc:24:11:2a:92:e3
Sending on   Socket/fallback
DHCPRELEASE of 192.168.1.122 on eth0 to 192.168.1.1 port 67 (xid=0x303d9b4c)
```
Let's check output of `logread` command
```
user.notice root: =======================
user.notice root: I've been supplied with this number of arguments: 0
user.notice root: There they are:
user.notice root: Here're variables available to me:
user.notice root: ACTION='remove'
...
user.notice root: HOSTNAME='OpenWrt'
...
user.notice root: IPADDR='192.168.1.122'
...
user.notice root: MACADDR='bc:24:11:2a:92:e3'
...
user.notice root: =======================
```
As one can see our script received `'OpenWrt'` instead of an empty string. 

Although when host sends its hostname it is being passed correctly
```bash
$ hostname
test

$ sudo dhclient -v -H $(hostname) eth0
Listening on LPF/eth0/bc:24:11:2a:92:e3
Sending on   LPF/eth0/bc:24:11:2a:92:e3
Sending on   Socket/fallback
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 6 (xid=0x4c02d471)
DHCPOFFER of 192.168.1.122 from 192.168.1.1
DHCPREQUEST for 192.168.1.122 on eth0 to 255.255.255.255 port 67 (xid=0x4c02d471)
DHCPACK of 192.168.1.122 from 192.168.1.1 (xid=0x4c02d471)
bound to 192.168.1.122 -- renewal in 17215 seconds.
```
Here's the output of `logread` command
```
user.notice root: =======================
user.notice root: I've been supplied with this number of arguments: 0
user.notice root: There they are:
user.notice root: Here're variables available to me:
user.notice root: ACTION='add'
...
user.notice root: HOSTNAME='test'
...
user.notice root: IPADDR='192.168.1.122'
...
user.notice root: MACADDR='bc:24:11:2a:92:e3' 
user.notice root: =======================
```


### Hotplug dhcp script
Just create a file `/etc/hotplug.d/dhcp/00-hello` with the contents below. 
```bash
#!/bin/sh
#set -x 
if [ "${HOSTNAME}" = "$(uci get system.@system[0].hostname)" ] || [ "${HOSTNAME}" = "" ] || [ "${IPADDR}" = "" ]; then
    exit 0
fi
if [ "${ACTION}" != 'remove' ] && [ "${ACTION}" != 'add' ]; then
    exit 0
fi
br_lan_ip=$(ip addr show dev br-lan | awk '/inet / {print $2}' | cut -d'/' -f1)
br_lan_ptr_zone=$(dig -x ${br_lan_ip} SOA | awk '/AUTHORITY SECTION:/ {getline; print $1}')
domain=$(uci get dhcp.@dnsmasq[0].domain)
REVERSE_IP=$(echo "${IPADDR}" | awk -F. '{print $4"."$3"."$2"."$1}')
TTL=900
case "${ACTION}" in
        add)
                logger -p daemon.info -t hotplug.dhcp "Processing \"DHCPAC ${IPADDR} ${MACADDR} ${HOSTNAME}\": Adding RRs for \"${HOSTNAME}.${domain}.\" and \"${REVERSE_IP}.in-addr.arpa.\""
                nsupdate -k /etc/bind/keys.conf <<EOL | logger -p daemon.info -t hotplug.dhcp
                server 127.0.0.1 53
                zone ${domain}.
                update delete ${HOSTNAME}.${domain}.
                update add ${HOSTNAME}.${domain}. ${TTL} A ${IPADDR}
                show
                send
                zone ${br_lan_ptr_zone}
                update delete ${REVERSE_IP}.in-addr.arpa.
                update add ${REVERSE_IP}.in-addr.arpa. ${TTL} PTR ${HOSTNAME}.${domain}.
                show
                send
EOL
#                rndc sync
        ;;
        remove)
                logger -p daemon.info -t hotplug.dhcp "Processing \"DHCPRELEASE ${IPADDR} ${MACADDR}\": Removing RRs for \"${HOSTNAME}.${domain}.\" and \"${REVERSE_IP}.in-addr.arpa.\""
                nsupdate -k /etc/bind/keys.conf <<EOL | logger -p daemon.info -t hotplug.dhcp
                server 127.0.0.1 53
                zone ${domain}.
                update delete ${HOSTNAME}.${domain}.
                show
                send
                zone ${br_lan_ptr_zone}
                update delete ${REVERSE_IP}.in-addr.arpa.
                show
                send
EOL
#                rndc sync
        ;;
esac
exit 0
```
Then restart `dnsmasq`  
```bash
$ /etc/init.d/dnsmasq restart
```
One may test the script's operation by issuing these commands on a host on the lan side. 
```bash
[rocky@test ~]$ hostname
test

[rocky@test ~]$ sudo dhclient -v -H $(hostname) eth0 -r
Listening on LPF/eth0/bc:24:11:2a:92:e3
Sending on   LPF/eth0/bc:24:11:2a:92:e3
Sending on   Socket/fallback
DHCPRELEASE of 192.168.1.122 on eth0 to 192.168.1.1 port 67 (xid=0x6f71f069)

[rocky@test ~]$ sudo dhclient -v -H $(hostname) eth0 
Listening on LPF/eth0/bc:24:11:2a:92:e3
Sending on   LPF/eth0/bc:24:11:2a:92:e3
Sending on   Socket/fallback
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 7 (xid=0x46faa10b)
DHCPOFFER of 192.168.1.122 from 192.168.1.1
DHCPREQUEST for 192.168.1.122 on eth0 to 255.255.255.255 port 67 (xid=0x46faa10b)
DHCPACK of 192.168.1.122 from 192.168.1.1 (xid=0x46faa10b)
bound to 192.168.1.122 -- renewal in 20468 seconds.
```
with simultaneously watching system logs in OpenWRT
```bash
$ logread -f 
...
daemon.info dnsmasq-dhcp[1]: DHCPRELEASE(br-lan) 192.168.1.122 bc:24:11:2a:92:e3
daemon.info hotplug.dhcp: Processing "DHCPRELEASE 192.168.1.122 bc:24:11:2a:92:e3": Adding RRs for "test.lan." and "122.1.168.192.in-addr.arpa."
daemon.info hotplug.dhcp: Outgoing update query:
daemon.info hotplug.dhcp: ;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
daemon.info hotplug.dhcp: ;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
daemon.info hotplug.dhcp: ;; ZONE SECTION:
daemon.info hotplug.dhcp: ;lan.                                IN      SOA
daemon.info hotplug.dhcp: ;; UPDATE SECTION:
daemon.info hotplug.dhcp: test.lan.            0       ANY     ANY
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#44123/key tsig-key: signer "tsig-key" approved
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#44123/key tsig-key: updating zone 'lan/IN': delete all rrsets from name 'test.lan'
daemon.info hotplug.dhcp: Outgoing update query:
daemon.info hotplug.dhcp: ;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
daemon.info hotplug.dhcp: ;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
daemon.info hotplug.dhcp: ;; ZONE SECTION:
daemon.info hotplug.dhcp: ;1.168.192.in-addr.arpa.             IN      SOA
daemon.info hotplug.dhcp: ;; UPDATE SECTION:
daemon.info hotplug.dhcp: 122.1.168.192.in-addr.arpa. 0        ANY     ANY
daemon.info named[1681]: client @0xffff95de81d0 127.0.0.1#58206/key tsig-key: signer "tsig-key" approved
daemon.info named[1681]: client @0xffff95de81d0 127.0.0.1#58206/key tsig-key: updating zone '1.168.192.in-addr.arpa/IN': delete all rrsets from name '122.1.168.192.in-addr.arpa'
...
daemon.info dnsmasq-dhcp[1]: DHCPDISCOVER(br-lan) 192.168.1.122 bc:24:11:2a:92:e3
daemon.info dnsmasq-dhcp[1]: DHCPOFFER(br-lan) 192.168.1.122 bc:24:11:2a:92:e3
daemon.info dnsmasq-dhcp[1]: DHCPREQUEST(br-lan) 192.168.1.122 bc:24:11:2a:92:e3
daemon.info dnsmasq-dhcp[1]: DHCPACK(br-lan) 192.168.1.122 bc:24:11:2a:92:e3 test
daemon.info hotplug.dhcp: Processing "DHCPAC 192.168.1.122 bc:24:11:2a:92:e3 test": Adding RRs for "test.lan." and "122.1.168.192.in-addr.arpa."
daemon.info hotplug.dhcp: Outgoing update query:
daemon.info hotplug.dhcp: ;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
daemon.info hotplug.dhcp: ;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
daemon.info hotplug.dhcp: ;; ZONE SECTION:
daemon.info hotplug.dhcp: ;lan.                                IN      SOA
daemon.info hotplug.dhcp: ;; UPDATE SECTION:
daemon.info hotplug.dhcp: test.lan.            0       ANY     ANY
daemon.info hotplug.dhcp: test.lan.            900     IN      A       192.168.1.122
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#46151/key tsig-key: signer "tsig-key" approved
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#46151/key tsig-key: updating zone 'lan/IN': delete all rrsets from name 'test.lan'
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#46151/key tsig-key: updating zone 'lan/IN': adding an RR at 'test.lan' A 192.168.1.122
daemon.info hotplug.dhcp: Outgoing update query:
daemon.info hotplug.dhcp: ;; ->>HEADER<<- opcode: UPDATE, status: NOERROR, id:      0
daemon.info hotplug.dhcp: ;; flags:; ZONE: 0, PREREQ: 0, UPDATE: 0, ADDITIONAL: 0
daemon.info hotplug.dhcp: ;; ZONE SECTION:
daemon.info hotplug.dhcp: ;1.168.192.in-addr.arpa.             IN      SOA
daemon.info hotplug.dhcp: ;; UPDATE SECTION:
daemon.info hotplug.dhcp: 122.1.168.192.in-addr.arpa. 0        ANY     ANY
daemon.info hotplug.dhcp: 122.1.168.192.in-addr.arpa. 900      IN      PTR     test.lan.
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#39719/key tsig-key: signer "tsig-key" approved
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#39719/key tsig-key: updating zone '1.168.192.in-addr.arpa/IN': delete all rrsets from name '122.1.168.192.in-addr.arpa'
daemon.info named[1681]: client @0xffff95f64280 127.0.0.1#39719/key tsig-key: updating zone '1.168.192.in-addr.arpa/IN': adding an RR at '122.1.168.192.in-addr.arpa' PTR test.lan.
...
```
Test
```bash
[rocky@test ~]$ nslookup test.lan
Server:         192.168.1.1
Address:        192.168.1.1#53
Name:   test.lan
Address: 192.168.1.122
```
Configuration complete.
